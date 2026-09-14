# HPM5E3Y ESC 与 ENET 资源及冲突确认补充

## 1. 文档目的

本文是《通过监听 EtherCAT 确定上位机是否失效》的补充文档，用于确认 C（HPM5E3Y）同时承担 EtherCAT 从站 ESC、备用 EtherCAT 主站 ENET 和仲裁器时的资源、引脚、接线和切换条件。

本文不改变原文档的总体方案，也不把 C 的 ESC 当作透明监听器。C 通过 NUC 写入自己的心跳 PDO 判断 NUC 是否仍然正常。

## 2. 最终结论

### 2.1 可以确认的结论

| 项目 | 结论 |
|---|---|
| ESC 与 ENET 是否可以同时运行 | 可以。ESC 从站和 ENET MAC 是两套独立功能，协议和软件任务可以并行 |
| HPM5E3Y 的功能资源是否足够 | 足够：一套 ESC 用于 C 从站，一套 ENET MAC 用于备用主站，GPIO 另留出仲裁控制和反馈信号即可 |
| 是否需要两个 ENET MAC | 不需要。备用主站只需要一个 ENET MAC；其物理出口由高速差分切换器选择接入 A 侧 |
| C_ESC 是否可串接在 B 后面 | 可以。推荐 `NUC → A → B → C_ESC P0`，C_ESC 为末端时 P1 不接 |
| C_ESC 是否与 C_ENET 内部连接 | 不连接。两者必须保持物理隔离，禁止形成环路 |
| NUC 与 C 是否能同时成为活动主站 | 通过硬件互锁、break-before-make 和软件状态机禁止 |

### 2.2 必须满足的条件

| 条件 | 说明 |
|---|---|
| ENET 采用独立且可用的 PHY 资源 | 优先采用目标型号支持的片内 PHY；若使用外部 PHY，必须分配不与 ESC 冲突的 MII/RMII 管脚 |
| ESC 与 ENET 的信号不共用 GPIO | 已知 ESC0_P0/P1 和 ETH0 外部 MII 在 ALT11/ALT18 存在重叠，不能直接假定无冲突 |
| 切换器支持 100BASE-TX | 100 Ω 差分、100 Mbps、全双工、受控阻抗、break-before-make |
| C 的 ENET 主站待命期间不发帧 | 只有获得 `C_ACTIVE` 后才允许发送 EtherCAT 周期帧 |
| 使用具体封装的 IOMUX 表完成核验 | 芯片系列级手册不能代替具体料号、封装和原理图核查 |

因此，本方案可以作为硬件设计基线；“绝对不会有引脚冲突”只有在选定具体 HPM5E3Y 料号/封装并完成下表核验后才能签字确认。

## 3. 资源分配

| 资源 | C 的用途 | 是否与另一功能共享 |
|---|---|---|
| EtherCAT ESC | 正常模式从站，接收 NUC 心跳 PDO | 不与 ENET MAC 共享数据通道 |
| ENET MAC | 接管后备用 EtherCAT 主站 | 不与 ESC 共享 MAC/DMA |
| ESC PHY 端口 | `C_ESC P0` 接 B 后端；P1 末端不接 | 仅服务 ESC |
| ENET PHY/接口 | 经切换器连接 A 侧 | 仅服务备用主站 |
| ENET DMA/描述符 | 主站帧收发 | 与 ESC 内存区分开 |
| GPIO | 切换器控制、触点反馈、PHY 复位 | 必须避开 MII/RMII 信号 |
| 定时器/中断 | 心跳超时、切换死区、PHY 状态 | 可由实时定时器独占仲裁任务 |
| SRAM/共享内存 | ESC DPRAM、ENET buffer、主站协议栈 | 分区，禁止地址重叠 |

## 4. 引脚冲突确认

### 4.1 已知风险

资料显示 ESC0_P0/P1 的 MII 信号和 ETH0 外部 MII 信号在多组 GPIO 上分别复用 ALT11/ALT18，涉及 PB00、PB01、PB07、PB11、PF02、PF03 等。若 ESC 和 ENET 都采用外部 MII，存在 TX/RX、TX_EN、RX_DV、MDC、MDIO 或参考时钟复用冲突的可能。

### 4.2 推荐实现

| 实现 | 结论 |
|---|---|
| ESC 使用专用可用 MII 管脚，ENET 使用片内 PHY | 首选，外部 MII 冲突最少 |
| ESC 使用 MII，ENET 使用独立 RMII | 可选，但必须确认该型号和 SDK 支持 RMII |
| ESC、ENET 都使用外部 MII | 仅在逐脚核对后采用，风险最高 |
| 运行时切换同一组 GPIO | 不采用。不能满足 ESC 在线和 ENET 待命并行 |

### 4.3 投板前必须完成的逐脚表

| 检查项 | 通过标准 |
|---|---|
| ESC0_P0/P1 所有 MII 信号 | 每根信号都有唯一 GPIO 和正确 ALT 配置 |
| ETH0 所有 MII/RMII 信号 | 不占用 ESC GPIO |
| REF_CLK | ESC、ENET 时钟源和方向均正确，无输出争用 |
| MDC/MDIO | PHY 地址、复位脚和管理总线无重叠 |
| PHY_RESET/INT | 每个 PHY 有独立复位和必要的状态输入 |
| 仲裁 GPIO | 不占用 MII、调试口或启动配置脚 |
| 封装可引出性 | 目标封装实际存在对应管脚，不能只看芯片全引脚表 |

## 5. 物理接线

### 正常运行链路

```text
NUC 主站 ── A ── B ── C_ESC P0
                         C_ESC P1：末端时不接
```

### 接管运行链路

```text
C_ENET 主站 ── 100BASE-TX 高速差分切换器 ── A ── B
```

切换器的公共端接 A 侧，两个选择端分别接 NUC 主站端和 C_ENET 主站端。不能把 NUC、C_ENET 的 TX/RX 差分线直接并联，也不能使用普通以太网交换机代替切换器。

切换器至少需要：

1. 两对差分线同时切换（TX±、RX±）；
2. 100 Ω 差分阻抗和 100BASE-TX 速率验证；
3. break-before-make；
4. NUC 通道和 C 通道的触点/开关状态反馈；
5. 硬件互锁，禁止两路同时闭合。

## 6. 心跳 PDO 和主站软件

建议心跳 PDO 至少包含：

| 字段 | 建议类型 | 说明 |
|---|---|---|
| `HeartbeatCounter` | uint32 | NUC 每个周期递增，禁止重复值长期保持 |
| `NUC_ACTIVE` | uint8 | NUC 是否声明拥有控制权 |
| `MasterState` | uint8 | NUC 的主站状态 |
| `CycleCounter` | uint32 | 可选，用于检测周期停止或异常跳变 |
| `CRC/Code` | uint16/uint32 | 对高安全等级系统建议加入 |

C 判定 NUC 正常的条件为：计数器在超时时间内持续变化、ESC 通信状态正常、`NUC_ACTIVE=1`。单次丢包不能触发切换，建议连续 3~10 个 EtherCAT 周期超时后进入故障确认。

C 的 ENET 主站软件可在正常模式完成驱动和协议栈初始化，但必须保持“只初始化、不发 EtherCAT 周期帧”。接管后必须重新扫描 A、B，校验 Vendor ID、Product Code、Revision、PDO 映射和必要的 SDO 配置，再按 `INIT → PRE-OP → SAFE-OP → OP` 进入运行。

## 7. 切换时序

```text
心跳连续超时
  ↓
故障确认与接管锁存
  ↓
停止 C_ENET 发送队列
  ↓
断开 NUC 通道并读取反馈
  ↓
两路全断，等待切换器死区和触点稳定
  ↓
确认 C_READY，接通 C_ENET
  ↓
等待 Link Up
  ↓
C 扫描 A、B，配置 PDO/SDO/DC
  ↓
先 SAFE-OP，写入安全目标值
  ↓
确认条件满足后进入 OP
```

推荐初始参数为 1~10 ms EtherCAT 周期、连续 3~10 个周期故障判定、10~50 ms 全断等待时间；最终值必须通过实际切换器、PHY、A/B 从站和伺服联合测试确定。

## 8. 验证清单

| 验证 | 通过标准 |
|---|---|
| ESC 单独运行 | NUC 可稳定写入和读取 C 的心跳 PDO |
| ENET 单独运行 | C 可扫描并控制 A、B |
| ESC+ENET 并行 | 连续运行无丢帧、DMA 覆盖、IRQ 异常或 PHY 复位互扰 |
| IOMUX 逐脚检查 | 两套接口无 GPIO、时钟、MDIO、复位冲突 |
| 切换器测试 | NUC/C 永不同时接入，链路可重复恢复 |
| 故障测试 | NUC 进程崩溃、死机、断网均进入安全切换流程 |
| 接管测试 | C 能重新扫描 A/B 并稳定进入 OP |
| 回退测试 | 接管失败时不反复切换，不产生危险输出 |
| 上电/复位测试 | C 复位或看门狗触发时两路不会同时闭合 |

## 9. 结论签字条件

在以下条件全部满足时，可确认“资源够用且设计无冲突”：

1. 选定具体 HPM5E3Y 料号和封装；
2. 依据该封装 IOMUX 表完成 ESC、ENET、PHY、复位、反馈和仲裁 GPIO 逐脚分配；
3. ENET 采用片内 PHY 或已验证的独立外部 PHY 接法；
4. ESC、ENET 的 DMA、内存和中断资源完成分区；
5. 100BASE-TX 高速差分切换器通过链路和信号完整性测试；
6. 完成上述并行运行、切换、接管和安全回退测试。

在未完成第 1、2 项前，只能确认“架构资源足够”，不能声称具体 PCB 引脚已经百分之百无冲突。
