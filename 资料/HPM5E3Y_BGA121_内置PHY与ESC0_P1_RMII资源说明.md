# HPM5E3Y BGA121 内置 PHY 与 ESC0_P1 RMII 资源说明

本文总结 HPM5E3Y BGA121 同时使用一个内置 Ethernet PHY，以及通过 `ESC0_P1` 连接外部 100BASE-TX PHY（RMII）的时钟、引脚和复用关系。

## 1. 方案结论

该组合原则上可以实现：

```text
HPM5E3Y 内置 PHY  ── 芯片内部 ENET/PHY 连接
外部 PHY         ── ESC0_P1，RMII 接口
```

内置 PHY 和外部 ESC0_P1 PHY 使用不同的数据接口资源，不是把两颗 PHY 并接到同一组 MII 数据线上。

但两者必须遵守 ESC 的统一参考时钟要求，且所有引脚必须按照实际封装和 PINMUX 配置逐一核对。

## 2. 时钟关系

HPM5E 用户手册说明：

- ESC 使用 100 MHz 功能时钟；
- 100 MHz 经 4 分频生成 25 MHz；
- 25 MHz 从 `ESC0_REFCK` 输出，作为 PHY 的参考时钟；
- ESC 和 Ethernet PHY 必须使用同一个时钟源；
- 不支持 ESC 使用芯片内部时钟、PHY 另接独立 25 MHz 晶振。

因此，系统时钟关系为：

```text
ESC 100 MHz
    ↓ 4 分频
25 MHz ESC0_REFCK
    ├── 内置 PHY 参考时钟
    └── 外部 ESC0_P1 PHY 参考时钟
```

### 25 MHz 与 50 MHz 的区别

- **25 MHz**：PHY/ESC 的参考时钟，来自 `ESC0_REFCK`。
- **50 MHz**：RMII 数据接口时钟，用于 `TXD[1:0]`、`RXD[1:0]` 的数据传输。

外部 PHY 使用 RMII 时，仍然使用 50 MHz RMII 接口时钟；该时钟通常由 ESC 或 PHY 内部 PLL 从统一的 25 MHz 参考时钟产生。不能把 50 MHz 直接接到要求 25 MHz 的 `ESC0_REFCK` 或 PHY 参考时钟输入。

RMII 的 100 Mbps 传输关系为：

```text
2 bit × 50 MHz = 100 Mbps
```

## 3. 内置 PHY 引脚

### PHY0

启用内置 PHY0 时，需要保留：

```text
PA20  PHY0_RD_N
PA21  PHY0_RD_P
PA22  PHY0_TD_N
PA23  PHY0_TD_P
PA27  PHY0 RBIAS
PA28  PHY0 100M LED/配置脚
PA29  PHY0 10M LED/配置脚
PA30  PHY0 MDIO
PA31  PHY0 MDC
```

### PHY1

启用内置 PHY1 时，需要保留：

```text
PA16  PHY1_RD_N
PA17  PHY1_RD_P
PA18  PHY1_TD_N
PA19  PHY1_TD_P
PA24  PHY1 RBIAS
PA25  PHY1 100M LED/配置脚
PA26  PHY1 10M LED/配置脚
PA30  PHY1 MDIO
PA31  PHY1 MDC
```

`PA30/PA31` 是管理接口脚。若启用内置 PHY 的 MDIO/MDC 管理功能，这两个脚不能再分配给其他外设。

## 4. ESC0_P1 使用 RMII 时的引脚

RMII 只使用 2 bit 数据，不使用 MII 的第 2、3 位数据线。

### 必需引脚

| RMII 功能 | ESC0_P1 信号 | BGA121 管脚 |
|---|---|---|
| TXD0 | `ESC0_P1_TXD_0` | PF03 / J10 |
| TXD1 | `ESC0_P1_TXD_1` | PF04 / H9 |
| TXEN | `ESC0_P1_TXEN` | PF07 / H11 |
| RXDV | `ESC0_P1_RXDV` | PF08 / G11 |
| RXD0 | `ESC0_P1_RXD_0` | PF09 / H10 |
| RXD1 | `ESC0_P1_RXD_1` | PF10 / L9 |
| RMII 时钟 | `ESC0_P1_TXCK` 或 `ESC0_P1_RXCK` | PF02 / L10 或 PF13 / J11，按时钟方向选择 |

外部 PHY 的 RMII 时钟方向必须与 ESC 配置一致：由 PHY 输出，或由 ESC 输出。不能让两端同时输出。

### RMII 下不需要的 MII 数据脚

以下信号只用于 MII 4-bit 数据，在 RMII 模式下不参与数据传输：

```text
ESC0_P1_TXD_2  → PF05 / G9
ESC0_P1_TXD_3  → PF06 / K11
ESC0_P1_RXD_2  → PF11 / G10
ESC0_P1_RXD_3  → PF12 / K10
```

如果外部 PHY 和 ESC0_P1 都配置为 RMII，这四根线可以不连接，并可复用给其他外设。PHY 芯片侧应按照其数据手册将这些脚处理为 NC 或其他允许的功能，不能自行接地或接电源。

如果任意一端配置为 MII，则这四根线必须全部连接，不能再复用。

## 5. ESC 管理接口与参考时钟

外部 PHY 通常还需要 ESC 的 MDC/MDIO 管理接口。BGA121 可优先考虑：

```text
PA30  ESC0_MDIO
PA31  ESC0_MDC
```

`ESC0_REFCK` 需要选择一个实际引出的 BGA121 管脚，例如手册列出的 `PA09` 等复用位置。选定后，该脚不能再用于其他功能。

注意：内置 PHY 的 `PA30/PA31` 管理接口与 ESC 外部 PHY 的管理接口是否能够共享，必须依据芯片的 PHY/ESC 管理架构和 SDK 配置确认，不能仅凭同名信号直接并接。

## 6. 可复用与不可复用汇总

### 启用内置 PHY 后不可复用

对应所选内置 PHY 的：

- 差分数据脚（PHY0：`PA20~PA23`；PHY1：`PA16~PA19`）；
- RBIAS 脚；
- LED/配置脚；
- 使用中的 `PA30/PA31` MDIO/MDC。

### ESC0_P1 RMII 后不可复用

```text
PF03、PF04、PF07、PF08、PF09、PF10
以及实际选择的 RMII 时钟脚 PF02 或 PF13
```

若配置需要同时使用 TXCK 和 RXCK，则 `PF02`、`PF13` 都应保留。

### ESC0_P1 RMII 后可复用

```text
PF05  ESC0_P1_TXD_2
PF06  ESC0_P1_TXD_3
PF11  ESC0_P1_RXD_2
PF12  ESC0_P1_RXD_3
```

这四根脚可分配给 I2C、MCAN、PWM、EUI 等其他 ALT 功能，前提是软件将其配置为对应 ALT，而不是 ESC0_P1 的 MII 信号。

## 7. 投板前检查清单

- [ ] HPM5E3Y 的具体料号确实包含内置 PHY。
- [ ] 内置 PHY 选择 PHY0 或 PHY1，并保留相应 PA 引脚。
- [ ] 外部 PHY strap 配置为 RMII，而不是 MII。
- [ ] ESC0_P1 软件配置为 RMII。
- [ ] `ESC0_REFCK` 输出为 25 MHz。
- [ ] 内置 PHY 与外部 PHY使用同一个 25 MHz 参考时钟源。
- [ ] RMII 50 MHz 时钟方向唯一，不能两端同时输出。
- [ ] `TXD2/TXD3/RXD2/RXD3` 在 RMII 模式下按 PHY 手册处理为 NC 或其他允许功能。
- [ ] `PA30/PA31` 的 MDIO/MDC 管理归属已经在原理图和 SDK 中确认。
- [ ] 所有 BGA121 坐标已经依据最终封装引脚表复核。

## 8. 核心结论

对于“一个内置 PHY + 一个 ESC0_P1 外部 RMII PHY”的方案：

1. 内置 PHY使用 25 MHz 参考时钟和专用 PA 引脚；
2. 外部 PHY通过 ESC0_P1 使用 RMII，数据接口使用 50 MHz；
3. 两套 PHY必须共享 ESC 输出的 25 MHz 参考时钟源；
4. RMII 不需要 `TXD2/TXD3/RXD2/RXD3`，这四根脚可以复用；
5. 只有在 MII 模式下，四根扩展数据线才必须连接；
6. `PA30/PA31`、`ESC0_REFCK` 以及 RMII 时钟脚必须重点核对，不能直接视为普通 GPIO。
