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

- **RELAXED 判定**：电池电压 `dV/dt < 1 µV/s`；在传统 OCV 路径下，停止充电后通常最多约 **2 小时**、停止放电后通常最多约 **5 小时**才能满足该条件；若 5 小时后仍未满足，TI 算法也会取得一次读数。注意这里说的是"停止充电/放电后"的静置等待时间，不要求充到 100% 或放到 0%。
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

### 5.1 可写节点

`/sys/class/xm_power/fg_master/` 下有两个**可写**节点：

| 节点 | 权限 | 作用 |
|---|---|---|
| `start_learning`（及 `_b`） | `rw-rw-r-- system system` | 向 `NVT_FG_REG_START_LEARNING` 写 `0x01`；用于厂商自定义 learning 机制，具体状态机语义未公开 |
| `stop_learning`（及 `_b`） | `rw-rw-r-- system system` | 向 `NVT_FG_REG_STOP_LEARNING` 写 `0x01`；具体结束行为由厂商 FG 固件定义 |

### 5.2 内核侧：寄存器结构证据

源码确认 `fg_set_start_learning()` 会写 `NVT_FG_REG_START_LEARNING` 寄存器。该寄存器周围的寄存器组包括：

```
START_LEARNING / STOP_LEARNING
EST_POWER / ACT_POWER / POWER_DEV / TIME_DEV
CONST_POWER / REF_POWER / REF_CURRENT
START_LEARNING_B ...
```

从整个寄存器组的结构看，这更像厂商自己的**功率/续航预测学习机制**。它跟 TI QMax 更新所需的 `GaugingStatus` / `QEN` / `VOK` / `QMAX` / `OCV` / `Ra` 链路在现有开源驱动中看不到直接关联。

### 5.3 内核侧：SM8550 属性枚举结构证据

在较早的 Xiaomi/Qualcomm 开源内核（SM8550 `qti_battery_charger.c`）中，这些属性明确被归在同一功能块：

```c
/*********nvt fuelgauge feature*********/
XM_PROP_START_LEARNING,
XM_PROP_STOP_LEARNING,
XM_PROP_SET_LEARNING_POWER,
XM_PROP_GET_LEARNING_POWER,
XM_PROP_GET_LEARNING_POWER_DEV,
XM_PROP_GET_LEARNING_TIME_DEV,
XM_PROP_SET_CONSTANT_POWER,
XM_PROP_GET_REMAINING_TIME,
XM_PROP_SET_REFERANCE_POWER,
XM_PROP_GET_REFERANCE_CURRENT,
XM_PROP_GET_REFERANCE_POWER,
XM_PROP_START_LEARNING_B,
XM_PROP_STOP_LEARNING_B,
XM_PROP_SET_LEARNING_POWER_B,
XM_PROP_GET_LEARNING_POWER_B,
XM_PROP_GET_LEARNING_POWER_DEV_B,
```

而 QMax/FCC/SOH 是紧接着的**另一组独立属性**：

```c
XM_PROP_FG1_QMAX,
XM_PROP_FG1_RM,
XM_PROP_FG1_FCC,
XM_PROP_FG1_SOH,
XM_PROP_FG1_FCC_SOH,
...
```

这已经是较强的结构证据：`START_LEARNING` 这一组高度像 NVT/MPC 电量计的功率/续航预测学习，而非 TI QMax learning。

> 源码依据：LineageOS SM8550 `qti_battery_charger.c`

### 5.4 HAL 侧：micharge-service 反汇编确证（2026-09-15 追加）

对本机 `/vendor/bin/hw/vendor.xiaomi.hardware.micharge-service`（119472 字节，AARCH64 PIE，剥离符号）做了完整反汇编分析：

- `.gnu_debugdata` 解出完整符号表（14728 字节），确认服务为 `aidl::vendor::xiaomi::hardware::micharge::MiCharge`，所有 AIDL 方法为 **get/set 型被动接口**（`getBatterySoh`、`getBatteryChargeFull`、`getBatteryCycleCount`、`getMiChargePath`、`setMiChargePath`、`setBatteryCommonInfo` 等）。
- **节点映射表**（`_GLOBAL__sub_I` 构建，`0x1ca50` 的 unordered_map）：109 个 sysfs 路径全部被装入 map，key 是节点名短串，value 是完整路径。包括：
  - 学习组：`start_learning`、`stop_learning`、`start_learning_b`、`stop_learning_b`、`learning_power`、`learning_power_b`、`learning_power_dev`、`learning_power_dev_b`、`learning_time_dev`、`action_power`、`constant_power`、`remaining_time`、`referance_power`、`referance_current`、`nvt_referance_power`
  - 容量/SOH 组：`qmax`、`qmax_cyclecount`、`soh`、`ui_soh`、`rel_soh`、`rel_soh_cyclecount`、`eis_soh`、`eis_soh_cyclecount`、`cyclecount`、`design_capacity`、`batt_sn`、`manufacturing_date` 等
  - 其余：`fast_charge`、`soc_decimal`、`reverse_chg_mode`、`wireless_ctrl_limit` 等
- **核心写路径 `setMiChargePath` @ 0xb788**：查 map → 找到路径 → 调 `writeToFile`（0x8d18，open/write 封装）。`setBatteryCommonInfo`/`setChargeCommonInfo` 等最终都走这条通用键值路径。
- **后台线程 `thread_func` @ 0x8fe4 + `do_work` @ 0x8e24**：只做 CPU 负载监测与节流（RUN_US 计算、定时器 `timer_hanlder` 置标志），**没有任何主动写学习节点的逻辑**。
- **`o1gUpgradeThread` @ 0xc064**：处理 o1g 固件升级（`PhoneBatteryChanged`/`screenStateChanged`/`startSendFirmFile` 等），全程无学习相关字符串引用。

**结论（反汇编确证）**：micharge-service 是**纯被动转发层**——它暴露了 `start_learning`/`stop_learning`/`enable_rollback` 等节点的读写能力，但**自身从不主动触发学习**。谁在何时写这些节点，取决于上层（PowerKeeper/系统服务）通过 AIDL 调用 `setMiChargePath(key, value)`。

### 5.5 HAL 侧：init rc 文件分组证据

`vendor.xiaomi.hardware.micharge-service.rc` 中 `chown` 块的节点排列同样呈现分组特征：

```
# 功率学习组（连续排列）
start_learning
stop_learning
learning_power
action_power
learning_power_dev
learning_time_dev
constant_power
remaining_time
referance_power
referance_current
nvt_referance_power
start_learning_b / stop_learning_b / learning_power_b / learning_power_dev_b

# 容量/SOH 组（另一段）
qmax
qmax_cyclecount
soh / ui_soh
rel_soh / eis_soh
rel_soh_cyclecount / eis_soh_cyclecount
```

### 5.6 HAL 侧：NDK 接口定义

HAL 的 NDK 接口库 `vendor.xiaomi.hardware.micharge-V2-ndk.so` 导出的 AIDL 方法列表中，部分容量指标有专用 convenience getter，而 `start_learning` 等节点通过通用键值接口访问：

- `getBatteryChargeFull()` / `getBatterySoh()` / `getBatteryCycleCount()`：FCC/SOH/CycleCount 的专用读取方法
- `getMiChargePath(path)` / `setMiChargePath(path, value)`：通用 sysfs 路径读写
- `getBatteryCommonInfo(key)` / `setBatteryCommonInfo(key, value)`：通用键值读写，`start_learning` 等功率学习节点通过此接口操作
- 注意：公开 AIDL 中**没有** `getBatteryQmax()` 之类的 QMax 专用方法，QMax 仍存在于通用节点映射中

HAL 接口结构证据：`start_learning` 主要作为通用 sysfs 键值项暴露；FCC、SOH、CycleCount 另有专用读取接口，而 QMax 仍存在于通用节点映射中。该结构进一步表明 HAL 没有实现一套显式的"QMax learning"控制流程，但**不能仅凭 HAL 接口分组排除 FG 固件内部 START_LEARNING 与 QMax 的潜在关联**。反汇编进一步确认：`getBatterySoh` 等专用方法内部仍是查同一张 109 节点 map 读路径，不存在独立的学习触发逻辑。

### 5.7 动态追踪：strace 初步结果

对本机 micharge-hal（PID 1932）进行 60 秒 strace 跟踪（`openat/read/write`），在手机空闲状态下：

- HAL 仅反复读取 `reverse_chg_mode`、`fast_charge`、`real_type`（轮询充电状态）
- **未观察到**对 `start_learning`、`learning_power`、`qmax`、`charger_full` 等节点的访问

这与 5.4 反汇编结论一致：HAL 后台线程只轮询充电状态，不主动触碰学习节点。

### 5.8 结论（更新为反汇编确证版）

综合内核寄存器结构、SM8550 属性枚举分组、HAL **反汇编**（非仅 strings）、rc 文件分组、NDK 接口分离和动态追踪，判断更新为：

**确证（反汇编级）**：
1. **micharge-service 是被动转发层**：`start_learning`/`stop_learning`/`enable_rollback` 等节点只是它 map 里的读写项，由上层 AIDL 调用触发，**HAL 自身不主动写**。
2. HAL 接口结构：`start_learning` 主要作为通用 sysfs 键值项暴露；FCC、SOH、CycleCount 另有专用读取接口，而 QMax 仍存在于通用节点映射中。该结构进一步表明 HAL 没有实现一套显式的"QMax learning"控制流程，但**不能仅凭 HAL 接口分组排除 FG 固件内部 START_LEARNING 与 QMax 的潜在关联**。
3. `batteryantiaging-service`（87KB，防老化 HAL）只含 `LowSohFvDown` / `BasedOnCC_VolDown` / `FreqChgFvDown` 策略类与 `UpdateChargeInfo`/`TriggerEvent`/`support_charger_mode` 字符串，**不含任何 start_learning/qmax/rollback 相关代码**，它管的是浮充电压（fv）下调，与 FG 学习无关。公开旁证：Xiaomi vendor tree 中存在 `libbaa_LowSohFvDown.so`、`libbaa_FreqChgFvDown.so` 等组件，属于 SOH/循环次数相关的充电电压降额策略，而非 FG QMax 学习模块。

### 5.9 上层触发者确证：PowerKeeper 反编译（2026-09-15 追加）

> 本节基于 `PowerKeeper.apk`（系统应用，负责电量/续航策略）DEX 反编译（dexdump 字节码级）。所有偏移均为 dex 内方法/指令偏移，方法索引为 method@X / field@X。源码类：`DynamicTurboPowerHandler.java`。

#### 5.9.1 完整调用链（确证）

```
PowerKeeper
  DynamicTurboPowerHandler$energyLearning.setStart_A()   [@1b3828]
    ├─ 若 mIsMiChargeOn（o()）为真（本机）：
    │     C("start_learning", "1")          → setMiChargeData("start_learning", "1")   [@1b3cd8]
    │       → mMiCharge.setMiChargePath(key, value)   // AIDL → HAL → sysfs 写入
    │         （setMiChargeData @1b3cf0：invoke-virtual IMiCharge.setMiChargePath / field@1f66）
    │     mIsFeatureFlip（m()）为真 → 再 C("", "1")  // Flip 场景额外写默认路径
    └─ 若 mIsMiChargeOn（o()）为假（qcom 旧方案）：
         H("sys/class/qcom-battery/start_learn", 1)
         mIsFeatureFlip（m()）为真 → H("sys/class/qcom-battery/fg2_start_learn", 1)
```

同样结构（对称，`o()`=mIsMiChargeOn 真走 HAL、假走 qcom）：

```
setReport_A()  [@1b36f0]：o()=真→ C("stop_learning","1")；o()=假→ H("sys/class/qcom-battery/stop_learn",1) / fg2_stop_learn
setReport_B()  [@1b378c]：o()=真→ C("stop_learn_b","1")；o()=假→ H(".../stop_learn_b",1) / fg2_stop_learn_b
setStart_B()   [@1b38b8]：o()=真→ C("start_learn_b","1")；o()=假→ H(".../start_learn_b",1) / fg2_set_learn_power_b
setStart_A()   [@1b3828]：o()=真→ C("start_learning","1")；o()=假→ H(".../start_learn",1) / fg2_start_learn
```

其中：

- `C(key, value)`：`setMiChargeData`（@1b3cd8）→ 静态字段 `mMiCharge`（field@1f66，类型 `miui/util/IMiCharge`）→ AIDL `setMiChargePath(key,value)`（method@6f3c）→ HAL（micharge-service）→ 查 map → `writeToFile("/sys/class/xm_power/fg_master/<key>", value)`。
- `H(path, int)`：直接写 qcom sysfs（旧机路径）。
- `K()`：`sget mIsFeatureOn`（总开关，@1b39dc）；`L()`：`sget mIsNVT`（@1b39f4）。
- **`o()` = `mIsMiChargeOn`**（@1b39a8）——注意这不是 `isSupportNVT`；`isSupportNVT` 是独立方法 @1b3bf0（见下）。`m()` = `mIsFeatureFlip`（@1b3978）——Flip 场景开关，不是总开关。
- **`isSupportNVT()`**（@1b3bf0，method@29bc）：`mIsMiChargeOn` 为真时读 `vendor`（getMiChargeDataToInt("vendor")），否则读 `sys/class/qcom-battery/fg_vendor`；**vendor 非零 → 设 `persist.sys.power.fuel.gauge=1` 返回 true**（本机走 HAL start_learning 分支）；否则 → 设 =0 返回 false（走 qcom 路径）。`K()`/`L()` 是 static bridge，`K()`=mIsFeatureOn（总开关），`L()`=mIsNVT。

**本机实测**：`/sys/class/xm_power/fg_master/vendor = 3`（MPC8011B，MPC8970 系列）→ `isSupportNVT()=true` → 本机确实走 **HAL 的 `start_learning`/`stop_learning` 分支**。

#### 5.9.2 触发条件（确证）

- `DynamicTurboPowerHandler.<init>`（@1b4ad8）创建 5 个 Runnable（`$3`/`$4`/`$5`/`$6`…）并注册 `EventsAggregator` 事件监听、`com.miui.powerkeeper_sleep_changed` 广播、休眠 `IntentReceiver`。
- 构造函数末尾 → `allScenePowerStatistics()`（@1b4efc）→ 若 **mOnBattery=false（充电中）**：`isSupportNVT()` 设置 `mIsNVT` 静态字段，然后 `postDelayed(mBScenesRunnable, 20000)` + `postDelayedAScenesRunnable()`。
- `handleMessage` 中 `MSG_CHARGING`（充电状态变化事件，@1b60b 附近）同样在 mOnBattery 翻转时调用 `allScenePowerStatistics()` → **每次充电/拔电事件都会重启整个 NVT 统计链**。

**三个 Runnable 的触发时机（确证）**：

| Runnable | 注册点 | 触发时机 | 动作 |
|---|---|---|---|
| `mSetStartRunnable`（`$3.run` @1b2974） | init | `postDelayed(10s)` 后 | 查 K()+L() → `setStart_A()` |
| `mSetSecondStartRunnable`（`$4.run` @1b29d0） | init | `postDelayed(12s)` 后 | 查 K()+L() → `setStart_A()` |
| `mAScenesRunnable`（`$5.run`） | init | `postDelayed(mAStartPostDelay≈200ms)` 后，每 20s 自调度 | 按 `mIsFeatureFlip`/`mIsInCall` 分派到 Flip/卫星/普通场景 |

- `postDelayedAScenesRunnable`（@1b73fc）：`postDelayed(mSetStartRunnable, 10000)` + `postDelayed(mSetSecondStartRunnable, 12000)` + `postDelayed(mAScenesRunnable, mAStartPostDelay)`。
- `startAScenesSelfLearningFlip`（@1b8068，Flip 场景）：`mCount==6` 或 `mCount%15==0` 时先 `setReport_A()`，读 `get_learn_power`，再 `setStart_A()`。

**结论（反汇编确证）**：`start_learning` 是 PowerKeeper 在 **充电过程中**（`mOnBattery=false`）通过 **A 场景自学习循环** 定时调用的功率学习机制：充电开始后延迟 10s/12s/200ms 触发，之后每 20s 循环，累计到特定计数（6 或 15 的倍数）时上报/重启学习。**所有写入都通过 HAL 的 `setMiChargePath("start_learning","1")` 转发**，与该机制是"功率/续航学习"而非 TI QMax 校准的推断一致，且**明确由上层主动触发、每次充电都会发生**。

#### 5.9.3 证据等级更新

| 结论 | 证据等级（更新） |
|---|---|
| PowerKeeper 是 `start_learning` 的**主动调用方** | **确证**（反编译 AIDL 调用链） |
| 触发时机：充电中，10s/12s 定时 + A 场景循环 | **确证**（postDelayed 常量 + 循环计数） |
| 本机（vendor=3）走 HAL `start_learning` 分支 | **确证**（实测 vendor + isSupportNVT 逻辑） |
| `setReport`（stop_learning）由同一循环在 `mCount==6/15的倍数` 时触发 | **确证** |
| `start_learning` 是功率/续航学习而非 QMax 校准 | 由"90% 推断"升为**高置信确证**（上层行为 + 寄存器/接口结构） |
| `start_learning` 一定与 QMax 完全无关 | 仍待 MPC8011B 固件行为确认 |
| `start_learning` 不能间接影响 FCC | 仍待实测 |

### 5.10 当前证据等级总结

| 结论 | 当前证据等级 |
|---|---|
| micharge-service 自身不会主动决定什么时候学习 | 确证 |
| HAL 本质是把上层请求转成 sysfs 读写 | 确证 |
| `learning_*` 属于 Estimated/Actual/Reference Power、Time Deviation 这一套 | 确证 |
| `learning_power=0` 不是 QMax learning 状态 | 确证 |
| PowerKeeper 充电中主动写 `start_learning=1`（A 场景自学习） | 确证（反编译调用链） |
| `start_learning` 是功率/续航学习而非 QMax 校准 | 高置信（上层行为 + 寄存器结构） |
| `start_learning` 一定与 QMax 完全无关 | 尚不能确证 |
| `start_learning` 不能间接影响 FCC | 尚不能确证 |
| TI QMax 条件就是 MPC8011B 实际条件 | 尚不能确证 |

---

## 六、FCC 无独立校准流程

没有发现独立的 FCC calibration 命令。现有内核、HAL、动态跟踪和 **PowerKeeper 反编译**证据共同确认 `start_learning` 是 PowerKeeper 在充电中主动触发的功率/续航学习机制，而非 FCC/QMax 校准触发器。但由于 MPC8011B 内部固件未公开，仍不能完全排除该机制在 FG 固件内部间接影响容量模型的可能性。所谓 "让 FCC 变准" 的本质是：

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
100% / 正常高 SOC
  ↓
停止充电，静置约 2h → 取得 OCV1
  ↓
正常放电至 30~40%（容量变化 >37%）
  ↓
停止使用，静置最长按约 5h 考虑 → 取得 OCV2
  ↓
可能发生 QMax update → FCC 随新 QMax/Ra 模型重算
  ↓
再充满 → Valid Charge Termination → FCC 再次重算
```

不需要放到 0%，也不需要把"再次充到 100%"当成第一次 QMax 更新的必要条件。步骤①②已构成一次完整的 QMax 更新机会；步骤③是额外收益。

这比 "0%→100% 完整循环" 更符合 Impedance Track 的实际工作方式：步骤①静置提供 OCV1；步骤②完成 ≥37% 放电后再次静置提供 OCV2，此时已满足普通 QMax 更新所需的两次 OCV 基本结构，同时放电过程也给 Ra grid 更新机会。步骤③重新充满并静置可继续提供下一次 OCV，同时有效充电终止也会触发 FCC 重算——但步骤③不是第一次 QMax 更新的必要条件。

### 什么情况不会触发 QMax 更新

- **长期窄区间浅循环**：当两次合格 OCV 之间的容量变化始终 < 37% 时，按 TI 普通 QMax 算法不会触发更新。例如长期在 55%~65% 之间浅充浅放，容量变化仅约 10%，远低于 37% 门槛。但若从 55% 放到 10%（容量变化约 45%），两端都有合格 OCV 且其他条件满足时，理论上仍可更新 QMax。
- 高温充电（> 40℃）、边充边用、频繁插拔都会让 OCV/温度条件不合格。
- QMax 更新是渐进的：即使成功，单次变化也被限制在 5% 设计容量以内，不会一次跳变。
- FCC 本身会因 QMax、Ra、温度、负载等实时变化，不需要等 QMax 更新就会波动。

---

## 七、后续研究方向

### 7.1 QMax 更新追踪

当前最值得追的线索是 `qmax_cyclecount=280` 与 `cyclecount=390` 的差异：若 MAC `0x0071` 在 MPC8011B 上确实表示 QMaxCycles，则该设备已约 110 个循环没有发生 QMax 更新。

后续可以研究：
- 为什么 K80 Pro 长期没有触发 QMax update——是使用习惯（长期浅充浅放）导致，还是 MPC8011B 的触发条件与 TI 不同？
- 能否从 `qmax` / `qmax_cyclecount` 的实时变化设计实验，证实 MPC8011B 的实际学习条件。

### 7.2 start_learning 机制确认

第五节已汇总了多层证据（内核寄存器结构、SM8550 属性枚举、HAL 反汇编、rc 文件分组、NDK 接口分析、strace、**PowerKeeper 反编译调用链**），当前判断已从"约 90%+ 推断"**升级为高置信确证**：

- **调用链确证**：PowerKeeper `DynamicTurboPowerHandler$energyLearning.setStart_A()` → `C("start_learning","1")` → `setMiChargeData()` → AIDL `mMiCharge.setMiChargePath("start_learning","1")` → micharge-service HAL → 写 `/sys/class/xm_power/fg_master/start_learning`。
- **触发时机确证**：充电中（`mOnBattery=false`）`allScenePowerStatistics()` 启动 A 场景自学习循环；`postDelayed` 10s/12s 定时 + 每 20s 循环，`mCount==6/15 倍数` 时 setReport/重启。
- **本机路径确证**：`vendor=3` → `isSupportNVT()=true` → 走 HAL `start_learning` 分支（非 qcom `start_learn`）。

剩余不确定性（需 MPC8011B 固件或实测）：
1. `start_learning` 是否在 FG 固件内部间接影响 QMax / FCC（无法从 APK 层确认）
2. 动态抓一次实际 `start_learning=1` → `stop_learning=1` 完整周期，同步记录 `qmax`、`qmax_cyclecount`、FCC、`learning_power`、`power_dev`、`remaining_time` 前后变化

这才是当前真正有价值的下一层——从"HAL 不主动触发"推进到"上层在什么条件下触发，触发后 FG 内部到底变了什么"。

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
  厂商自定义功率学习机制（PowerKeeper 充电中主动触发）
        ↓
  调用链确证（PowerKeeper 反编译）：充电中 10s/12s 定时 + A 场景循环
  触发后写 /sys/class/xm_power/fg_master/start_learning = 1
    ↓
  尚未发现与 QMax 更新链路的直接联系（需 MPC8011B 固件确认）


FCC 重新计算（自动发生，不等同于校准）：
  QMax 更新 / Ra 更新 / 充放电切换 / 充电终止
  RELAX 定期 / 温度变化 >5℃ / 电流与电荷变化
        ↓
  FCC 动态重算（每秒~每小时）
```
