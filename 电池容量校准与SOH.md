# 电池容量校准与 SOH（K80 Pro / miro）

> 本文同样是**电池级（Fuel Gauge）**参数，与有线/无线充电路径无关：`charge_full`（满充容量）不是厂商写死的标称值，而是电量计"学"出来的结果。

> 数据来源（本机实时读取，2026-09-15 实测）：
> - `/sys/class/xm_power/fg_master/*`：电量计原始量（`qmax`、`qmax_cyclecount`、`cyclecount`、`charger_full`、`soh`、`ui_soh`、`design_capacity` 等），由 `bq27z561.ko` 驱动导出
> - `/sys/class/power_supply/battery/uevent`：Android 上报值（`CHARGE_FULL`、`CYCLE_COUNT` 等）
> - 内核源码：`drivers/power/supply/mca/`（`mca_business/business_battery/mca_battery_psy.c`、`mca_strategy/strategy_fg/mca_strategy_fg.c`、`mca_hardware_ic/fuelgauge_ic/bq27z561/bq27z561.c`）
> - 算法条件原文：TI **BQ27Z561-R2 / BQ27Z558 Technical Reference Manual (SLUUBO7) §4.4.2 QMax Update Conditions**

## 一、先分清几个容量数字（本机实测）

| 字段 | 本机值 | 含义 |
|---|---:|---|
| `design_capacity` | 6000 mAh | 设计（标称）容量，来自设备树/电池包 |
| `qmax` | 6057 mAh | 电量计算出的**真实满充容量**，只有它会触发 `charge_full` 变化 |
| `charger_full` | 5797 mAh | 电量计当前上报的满充容量，Android 的 `charge_full` 就是它 |
| `tfullchgq` | 5781 mAh | 另一路满充量记录（随工况小幅浮动） |
| `soh` / `soh_new`（UI 显示） | 100 / 97 | 健康度；`soh_new` 即系统读的 `ui_soh` |
| `cyclecount` | 390 | 总循环次数 |
| `qmax_cyclecount` | 280 | 最近一次 Qmax 更新时的循环数（推断自驱动同块读取的字段顺序） |

观察：`5797 ÷ 6000 ≈ 96.6%`，与 `soh_new = 97` 基本一致——即"电池健康 97%"与 `charge_full` 是同一件事的两种表述。

## 二、谁决定 charge_full

- 内核**不做**容量计算：`mca_battery_psy.c` 只把电量计的 `POWER_SUPPLY_PROP_CHARGE_FULL` 透传给 Android；`strategy_fg_get_soh_new()` 直接读电量计的 `ui_soh`。
- 结论：**只有电量计自己完成一次 Qmax 更新，`charge_full` 才会变**。改 UI、改 HAL、改 MCA 都只会改显示，不会改这个值。
- 校准是"重学"，不是"修正偏差"：新值上限受 `Qmax Upper Bound` 约束、单次变化幅度受 `Qmax Delta` 约束（见下节）。

## 三、触发条件（TI TRM 原文口径）

Qmax 更新需要**两次 OCV（开路电压）读数**，分别在一次充/放电活动的前后、电池处于 RELAXED 状态时取得：

- **RELAXED 判定**：电池电压 `dV/dt < 1 µV/s`；充满态通常需要**最多 2 小时**、放空态**最多 5 小时**才满足；超过 5 小时即使未满足也会取读数。
- **必须有一次真实的充/放电活动**夹在两次静置之间——也就是一次完整的充放电循环。

以下任一条件成立会**取消**这次 Qmax 更新：

| 取消条件 | 门槛（TRM 默认值） |
|---|---|
| 温度出界 | `Temperature()` 必须在 **10 ℃ ~ 40 ℃** |
| 容量变化不足 | 两次静置之间容量变化 **< 37%** 即取消 |
| 电压落在平台区 | `GaugingStatus()[OCVFR]` 置位时取消 |
| 累计偏移误差超标 | 距上次 OCV 累计 offset error **> 1.5%** 设计容量（另有 11 小时最小时限保护） |

更新幅度的两道边界检查：

- `Qmax Delta`（默认 **5%** 设计容量）：单次更新最多变化这么多，超出会被截断。
- `Qmax Upper Bound`（默认 **130%** 设计容量）：Qmax 终生不超过该上限。

## 四、本机状态（为什么现在没在校准）

```bash
adb shell "su -c 'for n in qmax qmax_cyclecount cyclecount charger_full soh soh_new ui_soh learning_power learning_power_dev learning_time_dev design_capacity; do printf \"%-20s=\" \$n; cat /sys/class/xm_power/fg_master/\$n; done'"
```

实测：`qmax=6057`、`qmax_cyclecount=280`、`cyclecount=390`、`charger_full=5797000`、`soh_new=97`、`learning_power=0`、`learning_power_dev=0`、`learning_time_dev=0`。

- `learning_*` 全为 0 → **当前不在学习周期内**。
- `qmax_cyclecount(280) < cyclecount(390)` → 已有约 110 个循环没有更新过 Qmax（推断）。
- 设备芯片标识：`device_name=1XM31`、`vendor=3`、`eeprom_version=C`、`chip_ok=1`；驱动为 TI `bq27z561.ko`，但学习命令走定制寄存器（`NVT_FG_REG_START_LEARNING`），属 BQ27Z561 兼容的定制电量计。

## 五、系统侧还有一个"主动学习"通道

`/sys/class/xm_power/fg_master/` 下有两个**可写**节点：

| 节点 | 权限 | 作用 |
|---|---|---|
| `start_learning`（及 `_b`） | `rw-rw-r-- system system` | 向电量计写 `0x01`，开启一次学习窗口 |
| `stop_learning`（及 `_b`） | `rw-rw-r-- system system` | 结束学习窗口 |

证据：`/vendor/etc/init/vendor.xiaomi.hardware.micharge-service.rc` 明确 `chown system system .../start_learning`、`.../stop_learning`；`micharge-hal`（本机 pid 1932，running）二进制中含这些路径与 `qmax`、`soh`、`ui_soh` 字符串。

要点（**推断**）：这只是"开一个学习窗口"，**能不能真正更新 Qmax 仍取决于第三节的物理条件**；HAL 触发学习的内部门槛在闭源实现里，未验证。

## 六、实践结论

- 想让它重新校准：一次**深度放电 → 充满到终止**的循环，且两次静置各自留够时间（充满态 ~2 h、放空态 ~5 h），全程 **10~40 ℃**。
- **浅充浅放不会触发校准**：容量变化 < 37% 直接取消。把 SOC 限制在某个中间档（如 55%）长期使用，`charge_full` 基本不会更新。
- 高温充电（> 40 ℃）、边充边用、频繁插拔都会让 OCV/温度条件不合格。
- 校准是渐进的：即使成功，单次变化也被限制在 5% 设计容量以内，不会一次跳变。

