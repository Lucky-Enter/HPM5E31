# 通过监听 EtherCAT 确定上位机是否失效

## 1. 方案概述

C 的 HPM5E3Y 同时包含：

- **EtherCAT ESC**：正常运行时作为从站，接收 NUC 发来的心跳 PDO；
- **ENET MAC**：NUC 故障后切换为 EtherCAT 备用主站。

NUC 不需要额外 USB 或 GPIO。NUC 通过 EtherCAT 周期性向 C 的 ESC 写入专用心跳 PDO。C 检测心跳是否持续更新，超时后切换主站。

```text
NUC 主站 ── A ── B ── C 的 ESC（心跳从站）
  │
  └─ 主站端口通过仲裁切换器连接 A

C 的 ENET MAC ── 备用主站端口（切换后接入 A）
```

## 2. 心跳 PDO

建议在 C 的 ESC 从站对象字典中定义专用接收 PDO：

| 字段 | 建议类型 | 作用 |
|---|---|---|
| `HeartbeatCounter` | `uint32` | NUC 每周期递增 |
| `HeartbeatCode` | `uint16` | 可选校验值或固定标识 |
| `NUC_ACTIVE` | `uint8` | NUC 当前是否声明拥有控制权 |

示例：

```text
周期 1：HeartbeatCounter = 100
周期 2：HeartbeatCounter = 101
周期 3：HeartbeatCounter = 102
```

C 只要确认计数器持续变化，即可判断 NUC 的 EtherCAT 主站任务仍在运行。

## 3. C 的 ESC 从站配置

C 的 ESC 必须具备：


1. 对应的对象字典；
2. 接收 PDO 映射；
3. SyncManager 配置；
4. EtherCAT 看门狗或软件超时计时器；
5. 将心跳状态提供给 C 的仲裁程序。

NUC 的 EtherCAT 工程中要加入 C 这个从站，并配置相同的 PDO 映射。C 的 ESC 在 NUC 正常运行期间必须始终接入网络。

## 4. 故障判断逻辑

建议同时使用三项判断：

```text
HeartbeatCounter 在规定时间内发生变化
且 ESC 通信状态正常
且 NUC_ACTIVE = 1
→ NUC 正常
```

若满足以下任一条件，则进入故障计时：

- 心跳计数器在超时时间内没有变化；
- C 的 ESC 通信看门狗超时；
- NUC_ACTIVE 清零；
- C 连续多个周期没有收到有效 PDO。

建议初始参数：

| 参数 | 初始值建议 |
|---|---:|
| EtherCAT 周期 | 1 ms～10 ms |
| 心跳故障判定 | 连续 3～10 个周期 |
| 切换前全断时间 | 10 ms～50 ms |

最终值必须通过实际继电器、PHY 和伺服测试确定。

## 5. 主站切换时序

```text
NUC_ACTIVE
  ↓ 心跳 PDO 超时
撤销 NUC 授权
  ↓
断开 NUC 主站端口
  ↓ 确认触点断开
两路全断并等待
  ↓
接通 C 的 ENET 主站端口
  ↓
C 扫描 A、B
  ↓
验证身份、PDO 和 SDO 配置
  ↓
INIT → PRE-OP → SAFE-OP → OP
  ↓
恢复基本控制
```

C 只有在获得 `C_ACTIVE` 后才能发送 EtherCAT 周期帧。NUC 和 C 的主站端口必须通过硬件互锁，禁止同时接入。

## 6. 这种方式为什么比被动监听可靠

这里不是让 ESC 抓取所有经过的 EtherCAT 帧，而是让 NUC 明确发送一个专用心跳 PDO。因此 C 只需检查自己的从站 PDO 是否更新，不需要解析发往 A、B 的全部报文。

该心跳可以反映 NUC 的 EtherCAT 周期任务是否还在运行，比单独检测 Link 或载波更可靠。

但它仍不能保证 NUC 的控制算法一定正确。因此切换后应先执行安全停机或低速接管，不应默认无缝保持运动。

## 7. 主要限制

- C 的 ESC 是网络中的真实 EtherCAT 从站，不是透明旁路监听器；
- C 的 ESC 在 NUC 正常期间必须保持在线；
- NUC 的配置必须包含 C 的从站和心跳 PDO；
- C 必须保存 A、B 的完整主站配置；
- C 自身复位时，外部硬件看门狗和互锁必须保证两路主站不会同时接通。

## 8. 推荐结论

推荐采用以下结构：

```text
NUC 主站
  ├─ 周期控制 A、B
  └─ 周期发送心跳 PDO → C 的 ESC

C 的 ESC
  └─ 检测心跳超时 → 触发仲裁器

C 的 ENET MAC
  └─ 继电器切换后作为备用 EtherCAT 主站
```

这是不使用 USB 时较实用的方案，但必须完成 C ESC 从站 PDO 配置、主站继电器互锁以及切换后的 EtherCAT 重新初始化。

