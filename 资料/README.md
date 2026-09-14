# HPM5E 机器人 EtherCAT 主站仲裁与备份方案

## 1. 目标

系统包含三个 HPM5E 相关节点：

- **NUC11**：正常 EtherCAT 主站；
- **A、B**：机器人上的 EtherCAT 从站（A 为云台，B 为底盘）；
- **C（HPM5E3Y）**：备用 EtherCAT 主站，同时承担仲裁器功能。

假设 A、B 已经通过经过验证的物理层方案接入同一条 EtherCAT 网段。任意时刻只能有一个活动主站。

## 2. 总体架构

```text
NUC 100BASE-TX ──┐
                  ├── 100BASE-TX 高速差分切换器 ── A ── B
C   100BASE-TX ──┘

NUC USB ── USB CDC/USB转串口 ── C（HPM5E3Y）
```

C 的 HPM5E3Y 同时运行：

1. 仲裁任务：监测 NUC 心跳、控制切换器、执行互锁；
2. EtherCAT 主站任务：只有获得 `C_ACTIVE` 后才允许发送周期帧。

## 3. EtherCAT 数据线接法

仲裁器设置三个端口：

```text
J1：NUC 主站端口
J2：C 主站端口
J3：A 侧 EtherCAT 端口
```

100BASE-TX 只切换两对差分线，共 4 根：

| 信号  | 常见 RJ45 引脚 |
| --- | ----------:|
| TX+ | 1          |
| TX− | 2          |
| RX+ | 3          |
| RX− | 6          |

切换器的公共端接 J3，两个选择端分别接 J1 和 J2。A 的另一端继续连接 B：

```text
仲裁器 J3 ── A ── B
```

必须使用明确支持 **100BASE-TX、100 Ω 差分信号**的 Ethernet bypass relay 或高速差分切换器。普通低速信号继电器不建议使用。切换器必须支持 break-before-make，不能将 NUC、C 的 TX/RX 并联。

## 4. NUC 心跳与控制

NUC 没有 GPIO 时，使用 USB 传递状态：

```text
NUC USB ── USB CDC 或 USB转串口 ── C 的 USB/UART
```

NUC 软件周期发送心跳（例如每 10 ms 一次）。USB 只用于心跳和控制，不传 EtherCAT 实时 PDO。

建议消息：

- `HEARTBEAT`：NUC 正常运行；
- `NUC_RELEASE`：NUC 主动释放控制权；
- `C_READY`：C 已完成接管准备；
- `C_ACTIVE`：仲裁器授予 C 发帧权限。

## 5. 仲裁器硬件

C 的 GPIO 控制继电器或切换器驱动器，但线圈不能由 GPIO 直接驱动，应使用 MOSFET/驱动芯片和续流二极管。

建议信号：

```text
C_GPIO_RELAY_NUC ── NUC 通道驱动器
C_GPIO_RELAY_C   ── C 通道驱动器
RELAY_NUC_FB     ── C 输入
RELAY_C_FB       ── C 输入
```

必须有硬件互锁：

```text
NUC 触点已接通时，C 通道不能吸合；
C   触点已接通时，NUC 通道不能吸合。
```

上电、复位或 C 看门狗超时时，推荐默认两路全断（`ALL_OPEN`）。

## 6. 切换状态机

### 正常运行

```text
INIT → 接通 NUC → NUC_ACTIVE
```

C 保持不发送 EtherCAT 周期帧。

### NUC 故障切 C

```text
NUC_ACTIVE
  ↓ 心跳超时或 NUC Link Down
断开 NUC
  ↓ 确认触点断开
两路全断（短暂等待）
  ↓ C_READY 且 C Link Up
接通 C
  ↓
C_ACTIVE → C 扫描并初始化 A、B
```

### NUC 恢复

NUC 恢复后先完成心跳和主站初始化握手；C 释放控制权后，按同样的“断开—全断—接通”流程切回 NUC，禁止检测到 Link 恢复就立即切换。

## 7. C 如何确认 A、B 配置正确

C 应保存 NUC 使用的 ESI/XML、PDO 映射和 SDO 初始化配置。启动后在 `PRE-OP` 读取并比较：

| 信息            | 标准对象        |
| ------------- | ----------- |
| 设备名称          | `0x1008`    |
| Vendor ID     | `0x1018:01` |
| Product Code  | `0x1018:02` |
| Revision      | `0x1018:03` |
| Serial Number | `0x1018:04` |

随后读取实际 PDO 分配/映射（如 `0x1C12`、`0x1C13`、`0x1600`、`0x1A00` 等），与配置文件比较一致，再进入 `SAFE-OP` 和 `OP`。

如果 A/B 固件不支持 CoE/SDO，需要改为读取 SII EEPROM，或在从站固件中实现相应对象字典。

## 8. 运动与安全策略

切换期间通常会有 EtherCAT 通信中断，不能默认无缝保持运动。第一版建议：

1. NUC 故障后让 A、B 安全停机；
2. C 重新建立 `INIT → PRE-OP → SAFE-OP → OP`；
3. C 确认状态和 PDO 正确后，再重新使能伺服。

后续如需连续轨迹，必须额外同步目标位置/速度、模式、伺服使能、分布式时钟和轨迹状态。

## 9. 最低限度验证项目

- NUC 与 C 绝不会同时接通；
- 继电器确实 break-before-make；
- NUC USB 心跳超时能触发切换；
- C 未置 `C_READY` 时不会接通；
- C 能识别 A、B 并验证 PDO；
- NUC 进程崩溃、系统死机、网线断开均能进入安全流程；
- C 或仲裁器复位时不会同时接入两个主站；
- NUC 恢复后不会发生主站来回振荡。
