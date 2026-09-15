# 电池容量校准与 SOH（K80 Pro / miro）

> 本文是**电池级（Fuel Gauge）**参数分析，与有线/无线充电路径无关。

> 数据来源（本机实时读取，2026-09-15 实测）：
> - `/sys/class/xm_power/fg_master/*`：电量计原始量（`qmax`、`qmax_cyclecount`、`cyclecount`、`charger_full`、`soh`、`ui_soh`、`design_capacity` 等），由 `bq27z561.ko` 驱动导出
> - `/sys/class/power_supply/battery/uevent`：Android 上报值（`CHARGE_FULL`、`CYCLE_COUNT` 等）
> - 内核源码：`drivers/power/supply/mca/`（`mca_business/business_battery/mca_battery_psy.c`、`mca_strategy/strategy_fg/mca_strategy_fg.c`、`mca_hardware_ic/fuelgauge_ic/bq27z561/bq27z561.c`）
> - 算法条件参考：TI **BQ27Z561-R2 / BQ27Z558 Technical Reference Manual (SLUUC54C Rev. C) §4.7.2 QMax Update Conditions**

> **芯片说明**：本机实测 `vendor=3`，按 Xiaomi 开源驱动 `bq27z561.c` 中的 `enum fg_vendor` 枚举对应 **MPC8011B**，并非 TI BQ27Z561。驱动文件名为 `bq27z561.c` 是因为它是多厂商兼容驱动。下文涉及 TI TRM 的算法条件**仅作为兼容架构参考**，MPC8011B 是否完整复用 TI 的全部 QMax 更新条件尚未通过其固件/资料确认。

---

## 一、先分清几个容量数字（本机实测）

| 字段 | 本机值 | 含义 |
|---|---:|---|
| `design_capacity` | 6000 mAh | 设计（标称）容量，来自设备树/电池包 |
| `qmax` | 6057 mAh | 电量计学习得到的 **QMax（化学容量 chemical capacity 估计）**，是 FCC/SOC 计算的重要基础参数，但不等同于当前的 FullChargeCapacity |
| `charger_full` | 5797 mAh | 电量计当前上报的 **FullChargeCapacity (FCC)**，Android 的 `charge_full` 就是它 |
| `tfullchgq` | 5781 mAh | 另一路满充量记录（随工况小幅浮动） |
| `soh` | 100 | FG 标准 SOH 寄存器值；TI BQ27Z561-R2 的对应算法基于 `FCC_SOH`（固定 25℃、SOH Load Rate），但本机 MPC8011B 是否采用相同算法未确认 |
| `ui_soh` | 多字段 | 厂商 MAC `0x007B` 的 UI SOH 数据块（11 字节），sysfs 输出 `data[0]`~`data[10]`；本机实测首字节为 97 |
| `soh_new` | 97 | 系统使用的标量 UI SOH，即驱动缓存 `info->ui_soh`（源自 `FG_IC_PROP_SOH_NEW`） |
| `cyclecount` | 390 | 总循环次数 |
| `qmax_cyclecount` | 280 | 若该厂商字段遵循 TI QMaxCycles 语义，则为最近一次 QMax 更新时的循环数 |

### QMax 与 FCC 的关系

TI 对两者的定义非常明确：

- **QMax** 是通过 OCV（开路电压）和电荷积分等确定的 **chemical capacity**（化学容量），反映的是电池本身的化学活性总量。
- **FullChargeCapacity (FCC)** 是根据 QMax + 阻抗模型 (Ra) + 温度 + 电压 + 电流/负载条件 **实时计算**出来的当前可用容量。

$$ FCC = f(QMax,\ Ra,\ Temperature,\ Load,\ Voltage,\ \ldots) $$

因此 QMax = 6057 并不代表"当前能充进 6057 mAh"，而 charger_full = 5797 才是当前工况下 FG 上报的 FCC。

Xiaomi 源码也确认两者是独立字段：

```c
case FG_IC_PROP_QMAX:
    fg_read_qmax(info, &val);   // 读 QMax

case FG_IC_PROP_FCC:
    fg_read_fcc(info, &val);    // 读 FullChargeCapacity
```

> 源码依据：`bq27z561.c`（`drivers/power/supply/mca/mca_hardware_ic/fuelgauge_ic/bq27z561/`）

### SOH 与 charge_full 的关系

`5797 ÷ 6000 ≈ 96.6%`，与 `ui_soh = 97` 高度吻合，说明 Xiaomi 的 UI SOH 很可能与当前容量衰减程度高度相关。

但需要注意，这里至少存在**三套不同的 SOH 概念**：

| 概念 | 说明 |
|---|---|
| `soh` (FG 标准 SOH) | TI BQ27Z561-R2 的 `StateOfHealth()` 使用固定初始环境温度 (25℃) 和配置的 SOH Load Rate 专门计算一个 `FCC_SOH`，以减少普通 FCC 因实时负载/温度变化造成的波动。**但本机为 MPC8011B，是否采用 TI R2 的 FCC_SOH 算法未确认** |
| `fcc_soh` | FG 的 SOH 专用 FCC，是标准 SOH 的中间产物 |
| `ui_soh` (厂商 UI SOH) | 厂商自定义字段，Xiaomi 源码中 `fg_get_ui_soh()` 直接读取、`fg_store_ui_soh()` 可写入厂商 MAC `0x007B` |

本机 `soh = 100` 而 `ui_soh = 97`，本身就证明它们不是同一个东西。`ui_soh` 属于厂商自定义字段，其完整计算公式尚未从闭源 HAL/FG 固件中确认，因此不能直接断言 `ui_soh = FCC / DesignCapacity`。

---

## 二、谁决定 charge_full

### FCC 重新计算 vs FCC 长期基准校准

需要区分两件事：

| | FCC 重新计算 | FCC 长期基准校准
|---|---|---|
| **含义** | 电量计在正常使用中不断重算 FullChargeCapacity | 让 FCC 的长期基准变准，即让电量计重新认识电池实际容量 |
| **是否需要专门触发** | 不需要，自动发生 | 需要 QMax / Ra 阻抗模型发生学习更新 |
| **结果** | FCC 随工况小幅波动 | FCC 的基准值发生持久性变化 |

没有证据表明存在一个单独的 "FCC calibration" 命令，也不能用 `start_learning` 来触发 FCC 校准。所谓 "容量校准" 本质上是指 QMax + Ra 的学习更新，而非对 FCC 本身下达校准指令。

### Android/MCA 的角色

Android/MCA 层不自行估算 FCC。`mca_battery_psy.c` 只把电量计的 `POWER_SUPPLY_PROP_CHARGE_FULL` 透传给 Android；`strategy_fg_get_soh_new()` 直接读电量计的 `ui_soh`。

### FCC 何时重新计算

以 TI BQ27Z561-R2 的 Impedance Track 为参考，FCC 会在以下情况重新计算：

| 事件 | FCC 是否重算 |
|---|:---:|
| QMax 更新 | 是 |
| Ra 阻抗表更新 | 是 |
| 开始充电 | 是 |
| 开始放电 | 是 |
| 有效充电终止 | 是 |
| RELAX 后取得 OCV，每小时 | 是 |
| 温度变化 > 5℃ | 是 |
| 有电流、累计电荷变化 | 每秒都可能更新 |

因此即使 QMax = 6057 完全不动，FCC 也完全可能从 5797 → 5760 → 5810 之类地变化。一次正常充满到真正 termination，本身就会触发 FCC 重算，并不要求 QMax 更新。

### 长期准确性依赖什么

FCC 的长期准确性主要依赖 QMax、Ra 和 SOC/OCV 模型的正常学习。FCC 无独立校准流程；FCC 自动重算，其长期基准是否发生变化取决于底层模型是否发生了学习更新。

改 UI/HAL/MCA 无法改变 FG 芯片内部真实计算的 FCC；但修改上层或 MCA 驱动可以拦截、缩放或伪造 Android 最终看到的 `CHARGE_FULL` 上报值。

---

## 三、QMax 更新触发条件（TI TRM 参考）

> 以下条件来自 TI BQ27Z561/BQ27Z561-R2 Impedance Track 算法，仅作为兼容架构参考。本机 `vendor=3` 按 Xiaomi 开源驱动枚举对应 MPC8011B，是否完整复用 TI 的所有 QMax 更新条件尚未通过 MPC8011B 固件/资料确认。

QMax 更新需要**两次合格的 OCV（开路电压）读数**，分别在一次充/放电活动的前后、电池处于 RELAXED 状态时取得：

- **RELAXED 判定**：电池电压 `dV/dt < 1 µV/s`；充满态通常需要**最多 2 小时**、放空态**最多 5 小时**才满足；超过 5 小时即使未满足也会取读数。
- **两次静置之间必须有足够的充/放电活动**——TI 要求容量变化 **≥ 37%** 设计容量。

以下任一条件成立会**取消**这次 QMax 更新：

| 取消条件 | 门槛（TRM 默认值） |
|---|---|
| 温度出界 | `Temperature()` 必须在 **10 ℃ ~ 40 ℃** |
| 容量变化不足 | 两次静置之间容量变化 **< 37%** 即取消 |
| 电压落在平台区 | `GaugingStatus()[OCVFR]` 置位时取消 |
| 累计偏移误差超标 | 距上次 OCV 累计 offset error **> 1.5%** 设计容量（另有 11 小时最小时限保护） |

更新幅度的两道边界检查：

- `Qmax Delta`（默认 **5%** 设计容量）：单次更新最多变化这么多，超出会被截断。
- `Qmax Upper Bound`（默认 **130%** 设计容量）：QMax 终生不超过该上限。

### Ra 阻抗表学习

FCC 不仅取决于 QMax，还取决于 Ra——电池在不同 SOC 下的内阻表。TI 的 Ra 表主要在**放电过程中学习**：电池从高 SOC 向低 SOC 放电时跨过不同 DOD grid point，电量计利用负载下的电压跌落来更新对应的阻抗。

因此一次从高 SOC 到中低 SOC 的正常放电，除了为 QMax 更新提供两次 OCV 之间的容量变化外，还有一个重要作用：**给 FG 机会更新多个 Ra grid**。这也是为什么 100% → 90% → 再充回 100% 这种窄区间循环即使天天重复，也很难充分重新学习容量模型。

### 不要求 0→100% 完整循环

TI 普通 QMax 更新的核心条件是**两次合格 OCV 之间容量变化 ≥ 37%**，并不要求从 0% 放到 100% 或完成一次完整循环。例如 90% → 40%（容量变化 50%），两端都有合格 OCV，一样可能完成 QMax 更新。

此外 BQ27Z561-R2 还支持：
- **Fast QMax**：可以只依赖一个 OCV，并在放电到 10% RSOC 以下时更新。
- **Cycle Count Based QMax Degradation**：用于长期无法满足普通 QMax 条件的场景。

> 以上均为 TI 算法描述，本机 MPC8011B 是否完全沿用尚待确认。

---

## 四、本机状态

```bash
adb shell "su -c 'for n in qmax qmax_cyclecount cyclecount charger_full soh soh_new ui_soh learning_power learning_power_dev learning_time_dev design_capacity; do printf \"%-20s=\" \$n; cat /sys/class/xm_power/fg_master/\$n; done'"
```

实测：`qmax=6057`、`qmax_cyclecount=280`、`cyclecount=390`、`charger_full=5797000`、`soh=100`、`soh_new=97`、`learning_power=0`、`learning_power_dev=0`、`learning_time_dev=0`。

- `qmax_cyclecount(280) < cyclecount(390)`：若该厂商字段遵循 TI QMaxCycles 语义，则大约从 cycle 280 到现在 390 没有发生新的 QMax 更新。但本机驱动实际读取的是厂商 MAC `0x0071`（`FG_MAC_CMD_QMAX_CYCLECOUNT`），而非 TI 标准 `0x3A/0x3B`，因此该推断仍需注明限定条件。
- 设备芯片标识：`device_name=1XM31`、`vendor=3`（MPC8011B）、`eeprom_version=C`、`chip_ok=1`；驱动文件名为 `bq27z561.ko`（多厂商兼容驱动），学习命令走定制寄存器。

> **关于 `learning_*` 节点**：`learning_power`、`learning_power_dev`、`learning_time_dev` 在驱动中的真实定义为 `NVT_FG_REG_EST_POWER`（估算功率输入）、`NVT_FG_REG_POWER_DEV`（估算功率与实际功率偏差）、`NVT_FG_REG_TIME_DEV`（估算功率时间与实际时间偏差）。它们属于厂商"功率学习/估算"相关寄存器，**不是 QMax 学习状态位**。其值为 0 只说明这些估算量当前为 0，不能推出"当前是否处于 QMax 学习周期"。

---

## 五、厂商自定义"功率学习"接口（与 QMax 的关系未确认）

`/sys/class/xm_power/fg_master/` 下有两个**可写**节点：

| 节点 | 权限 | 作用 |
|---|---|---|
| `start_learning`（及 `_b`） | `rw-rw-r-- system system` | 向电量计写 `0x01`，开启一次学习窗口 |
| `stop_learning`（及 `_b`） | `rw-rw-r-- system system` | 结束学习窗口 |

证据：`/vendor/etc/init/vendor.xiaomi.hardware.micharge-service.rc` 明确 `chown system system .../start_learning`、`.../stop_learning`；`micharge-hal`（本机 pid 1932，running）二进制中含这些路径与 `qmax`、`soh`、`ui_soh` 字符串。

源码确认 `fg_set_start_learning()` 会写 `NVT_FG_REG_START_LEARNING` 寄存器。但该寄存器周围的寄存器组包括：

```
START_LEARNING / STOP_LEARNING
EST_POWER / ACT_POWER / POWER_DEV / TIME_DEV
CONST_POWER / REF_POWER / REF_CURRENT
START_LEARNING_B ...
```

从整个寄存器组的结构看，这明显更像厂商自己的**功率/续航预测学习机制**。它跟 TI QMax 更新所需的 `GaugingStatus` / `QEN` / `VOK` / `QMAX` / `OCV` / `Ra` 链路在现有开源驱动中看不到直接关联。

因此：`start_learning` / `stop_learning` 确实控制厂商自定义学习寄存器，同时伴随 Estimated Power、Actual Power、Power Deviation、Time Deviation 等参数。从现有开源驱动**无法证明该机制用于 TI Impedance Track 的 QMax 学习**，暂不把它与 QMax 校准等同。

---

## 六、FCC 无独立校准流程

FCC 没有独立的校准命令，也不能通过 `start_learning` 触发。所谓 "让 FCC 变准" 的本质是：

```
正常充放电
     ↓
 ┌── QMax 学习
 ├── Ra 学习
 ├── OCV 修正 SOC
 └── 温度/负载模型
     ↓
FG 更新内部模型
     ↓
重新计算 FCC
     ↓
charger_full 发生变化
```

### 基于 TI 算法的最大概率操作法（仅供参考）

> 以下基于 TI Impedance Track 算法推测。本机实际芯片为 MPC8011B，不能保证完全一样。

没必要放到自动关机，也不建议故意过度深放。可以做一次这样的周期：

```
① 正常充到 100%
   ↓ 继续等到真正停止充电/显示 FULL
   ↓ 拔电，屏幕关闭静置约 2 小时

② 正常使用放到约 30～40%
   （100→40 已远大于 37%）
   ↓ 屏幕关闭、不要充电，长时间静置
   ↓ 理想情况下约 5 小时

③ 再正常充到 100%
   ↓ 等待真正充电终止
   ↓ 拔电，再静置约 2 小时
```

整个过程中尽量保持 20～30℃，不要边充边打游戏，不要为了校准把手机放到高温环境。

这比 "0%→100% 完整循环" 更符合 Impedance Track 的实际工作方式：步骤①③提供合格 OCV，步骤②提供 ≥37% 的容量变化并同时给 Ra grid 更新机会。

### 什么情况不会触发 QMax 更新

- **长期窄区间浅循环**：当两次合格 OCV 之间的容量变化始终 < 37% 时，按 TI 普通 QMax 算法不会触发更新。例如长期在 55%~65% 之间浅充浅放，容量变化仅约 10%，远低于 37% 门槛。但若从 55% 放到 10%（容量变化约 45%），两端都有合格 OCV 且其他条件满足时，理论上仍可更新 QMax。
- 高温充电（> 40℃）、边充边用、频繁插拔都会让 OCV/温度条件不合格。
- QMax 更新是渐进的：即使成功，单次变化也被限制在 5% 设计容量以内，不会一次跳变。
- FCC 本身会因 QMax、Ra、温度、负载等实时变化，不需要等 QMax 更新就会波动。

---

## 七、后续研究方向

当前最值得追的线索是 `qmax_cyclecount=280` 与 `cyclecount=390` 的差异：若 MAC `0x0071` 在 MPC8011B 上确实表示 QMaxCycles，则该设备已约 110 个循环没有发生 QMax 更新。

后续可以研究：
- 为什么 K80 Pro 长期没有触发 QMax update——是使用习惯（长期浅充浅放）导致，还是 MPC8011B 的触发条件与 TI 不同？
- 能否从 `qmax` / `qmax_cyclecount` 的实时变化设计实验，证实 MPC8011B 的实际学习条件。

---

## 附：QMax / FCC / SOH / learning 关系总览

```
正常充放电
     ↓
 ┌── QMax 学习（需两次合格 OCV，中间 ΔCapacity ≥37%）
 ├── Ra 学习（放电过程中跨 DOD grid 更新阻抗）
 ├── OCV 修正 SOC
 └── 温度/负载模型
     ↓
FG 更新内部模型
     ↓
重新计算 FCC = f(QMax, Ra, T, I, V, ...)
     ↓
charger_full / Android charge_full


TI 标准 SOH：
  QMax + Ra + 固定 SOH 工况 (25℃, SOH Load Rate)
        ↓
      FCC_SOH
        ↓
       SOH


Xiaomi UI SOH：
  厂商自定义算法
        ↓
      ui_soh
        ↓
      soh_new


start_learning / learning_power：
  厂商自定义功率学习机制
        ↓
  目前无证据证明 = QMax Learning


FCC 重新计算（自动发生，不等同于校准）：
  QMax 更新 / Ra 更新 / 充放电切换 / 充电终止
  RELAX 定期 / 温度变化 >5℃ / 电流与电荷变化
        ↓
  FCC 动态重算（每秒~每小时）
```
