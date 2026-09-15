# K80 Pro MPC8011B 电量计实机参数快照

> 采集时间：2026-09-15，设备 192.168.33.118:5555，root
> 采集方式：逐节点 `cat /sys/class/xm_power/fg_master/<node>`
> 节点总数：78 个（排除 power/ 子目录和 subsystem/uevent 符号链接）

---

## 0. 设备基本信息

| 项目 | 值 |
|---|---|
| 机型 | Xiaomi K80 Pro (onyx) |
| 电量计（Fuel Gauge，FG）芯片 | MPC8011B（vendor=3） |
| 电池型号 | 1XM31 |
| 电芯供应商 | lwn |
| 电池序列号 | SLBP5D4B18029163WMD1H00000000000 |
| 生产日期 | 2024-11-18 |
| 首次使用日期 | 2025-02-05 |
| Data Flash 版本 | C |
| seal 状态 | 3（sealed） |

---

## 1. 核心电量计参数

| 节点 | 实机值 | MPC8011B 含义 | 单位 | 备注 |
|---|---|---|---|---|
| `chip_ok` | 1 | FG 通信/芯片状态 | 0/1 | 1=正常 |
| `vendor` | 3 | FG 厂商枚举 | 枚举 | 3=MPC8011B（NVT 路径） |
| `device_name` | 1XM31 | FG/电池设备型号 | 字符串 | — |
| `vbatt` | 3857 | 电芯电压 | mV | 采集时电压 |
| `ibatt` | 289000 | 电池电流 | µA | 正值=充电方向 |
| `rsoc` | 44 | FG 原始 SOC | % | — |
| `cyclecount` | 390 | 循环次数 | cycle | RW 但写入是 fake override，勿写 |
| `qmax` | 6057 | QMax 化学容量估计 | mAh | **> 设计容量 6000** |
| `qmax_cyclecount` | 280 | 驱动命名的 QMax CycleCount 字段；精确更新语义未确认 | cycle | MPC 私有语义，与 cyclecount 差 110 |
| `rm` | 2521000 | 剩余容量 | µAh | = 2521 mAh |
| `charger_full` | 5797000 | FCC 满充容量 | µAh | = 5797 mAh |
| `design_capacity` | 6000000 | 设计容量 | µAh | = 6000 mAh |
| `soh` | 100 | 标准 SOH 寄存器返回值 | % | MPC8011B SOH 算法未公开 |
| `soh_new` | 97 | Xiaomi 缓存 UI SOH | % | 与 ui_soh byte0 一致 |
| `ui_soh` | 97 119 1 2 97 100 100 100 100 100 207 | UI SOH 11 字节块 | byte[] | byte0 确认被驱动用作 UI SOH；后 10 byte 含义未确认 |
| `fcc_soh` | 0 | TI FCC_SOH | — | 当前返回 0，可能未启用/未实现/无有效值 |
| `fast_charge` | 0 | FG 快充状态 | 0/1 | 采集时非快充 |
| `seal` | 3 | FG 密封状态 | 枚举 | 3=sealed |
| `eeprom_version` | C | Data Flash 版本 | 字符 | — |
| `df_check` | 33001 | Data Flash 校验值 | 私有 | — |
| `resistance_id` | 100000 | 电阻 ID | 私有 | 单位待确认 |

### 关键发现

**QMax=6057 > 设计容量 6000**：QMax 超过设计容量约 0.95%。qmax_cyclecount 字段值为 280，与 cyclecount(390) 差 110，但该字段的精确更新语义未确认，不能据此认定"第 280 次循环时更新了 QMax"。QMax 超标称值在 TI 算法中可出现于全新电池，但 MPC8011B 的 QMax 语义是否与 TI 完全一致尚待确认。

**FCC=5797 mAh，QMax=6057 mAh**：FCC < QMax 是正常的——FCC = f(QMax, Ra, T, I, V)，阻抗和温度/负载会降低可用容量。FCC/Design = 96.6%，与 SOH_new=97% 吻合。

**SOH=100 vs SOH_new=97**：soh=100 为 MPC8011B 标准 SOH 寄存器的当前返回值。虽然 QMax/DesignCapacity=100.95% 与 100% 数值上接近，但 MPC8011B 的 SOH 具体计算公式未公开，不能据此认定 SOH 由 QMax/DesignCapacity 截断得到。soh_new=97 与 FCC/Design=96.6% 数值高度吻合，但目前只是相关性，尚无证据证明 Xiaomi UI SOH 直接由 FCC/Design 计算。两者算法不同——Xiaomi UI SOH 可能综合了循环次数、温度历史、阻抗趋势等因素。

---

## 2. IT / 容量模型内部参数

| 节点 | 实机值 | 含义 | 备注 |
|---|---|---|---|
| `tfullchgq` | 5835 | TFullChgQ 满充容量模型内部量 | 比 charger_full(5797) 略高，可能是 FG 内部模型值 |
| `tremq` | 2605 | TRemQ 内部剩余容量 | 比 rm(2521) 略高，同样可能是内部模型值 |
| `tsim` | 3052 | ITStatus1 内部温度/模拟字段 | 私有，具体定义未公开 |
| `tambinet` | 65232 | Ambient Temperature（源码拼写错误） | ITStatus1 厂商字段，私有 raw |
| `avercurrent` | 93 | FG AverageCurrent | mA（非 µA） |
| `average_current` | 93 | NVT/MPC 10 秒平均电流 | mA |
| `average_temperature` | 63096 | NVT/MPC 10 秒平均温度 | 私有 raw |
| `calc_rvalue` | 0 | 综合 R-value | 0 可能表示未计算/不支持 |
| `batt_use_environment` | 0 0 253 25 0 0 7 0 187 31 0 0 177 0 17 3 0 0 0 0 20 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 | 电池使用环境数据块 | 私有 38 字节 |

### tfullchgq vs charger_full

| 量 | 值 | 来源 |
|---|---|---|
| `charger_full` | 5797 mAh | 驱动从 FG 寄存器读取后 ×1000 输出（µAh） |
| `tfullchgq` | 5835 | 驱动从 ITStatus1 直接取的原始值 |
| `rm` | 2521 mAh | 驱动从 FG 寄存器读取后 ×1000 输出（µAh） |
| `tremq` | 2605 | 驱动从 ITStatus1 直接取的原始值 |

tfullchgq 和 tremq 与 charger_full/rm 之间分别有约 38 mAh 的差值。这可能对应 FG 内部不同计算路径（TI 标准 MAC vs NVT/MPC 私有 ITStatus 块），或者两条路径的更新时机不同。

---

## 3. 电池身份与生产信息

| 节点 | 实机值 | 含义 |
|---|---|---|
| `batt_sn` | SLBP5D4B18029163WMD1H00000000000 | 电池序列号 |
| `cell_vendor` | lwn | 电芯供应商 |
| `pack_vendor` | 2 | 电池 PACK 厂商 ID |
| `manufacturing_date` | 20241118 | 生产日期 2024-11-18 |
| `first_usage_date` | 20250205 | 首次使用日期 2025-02-05 |
| `fake_first_usage_date` | 0 | 未伪造 |
| `batt_use_environment` | 0 0 253 25 0 0 7 0 187 31 ... | 电池使用环境数据块（38 字节，私有） |

### 电池年龄

| 项目 | 值 |
|---|---|
| 生产到首用 | 79 天 |
| 首用到采集 | 587 天（约 19.3 个月） |
| 总循环 | 390 次 |
| 日均循环 | 约 0.66 次/天 |

0.66 次/天属于正常使用强度。循环次数不能直接等比为寿命消耗比例，故不据此估算剩余寿命。

---

## 4. 老化与 DOD 统计

| 节点 | 实机值 | 含义 | 备注 |
|---|---|---|---|
| `aged_flag` | 0 | 老化标志 | 未触发 |
| `dod_count` | 0 | DOD 记录刷新结果 | 0=操作成功（非实际次数） |
| `count_level1` | 1 | DOD/使用强度第1档累计 | 有 1 次触发 |
| `count_level2` | 0 | 第2档累计 | 未触发 |
| `count_level3` | 0 | 第3档累计 | 未触发 |
| `count_lt` | 0 | 低温使用累计 | 未触发 |
| `rel_soh` | 0 | Relative SOH | 当前返回 0，可能未启用/未实现/无有效值 |
| `rel_soh_cyclecount` | 0 | rel_soh 循环计数 | 同上 |
| `eis_soh` | 0 | EIS SOH | 当前返回 0，可能未启用/未实现/无有效值 |
| `eis_soh_cyclecount` | 0 | EIS SOH 循环计数 | 同上 |

**rel_soh / eis_soh / fcc_soh 全为 0**：这三个 TI 兼容 SOH 指标在 MPC8011B 上当前均返回 0，可能未启用、未实现或无有效值，尚不能确证 MPC8011B 不支持 TI 多 SOH 体系。

---

## 5. Lifetime 历史极值

| 节点 | 实机值 | 含义 | 备注 |
|---|---|---|---|
| `max_life_vol` | 4519 | 历史最高电压 | mV |
| `min_life_vol` | 2883 | 历史最低电压 | mV |
| `max_life_temp` | 44 | 历史最高温度 | 厂商 raw（可能 44°C） |
| `min_life_temp` | 6 | 历史最低温度 | 厂商 raw |
| `temp_max` | 44 | Lifetime 最大温度 | 同 max_life_temp |
| `max_temp_occur_time` | 2780431 | 最高温发生时刻/计数 | 私有时间单位 |
| `max_temp_time` | 899857 | 距最高温时间/计数 | 私有时间单位，单位链条未确认，不应换算 |
| `run_time` | 56771911 | FG 运行时间计数 | 私有 |
| `total_fw_runtime` | 0 | FG 固件累计运行时间 | 私有 |
| `time_ht` | 0 | 高温累计时间 | 当前为 0，可能是未触发或未支持/未记录 |
| `time_ot` | 0 | 过温累计时间 | 当前为 0，可能是未触发或未支持/未记录 |
| `over_vol_duration` | 0 | 高电压累计时间 | 当前为 0，可能是未触发或未支持/未记录 |

time_ht=0、time_ot=0、over_vol_duration=0，可能是未触发或未支持/未记录，不能仅凭 0 值判断温度管理状态。max_life_temp=44（厂商 raw，可能对应 44°C）在手机充电场景中属于正常偏高范围。

max_life_vol=4519 mV 和 min_life_vol=2883 mV 均在锂电池安全区间内（典型 4.5V 上限和 2.8V 下限）。

---

## 6. MPC/NVT 私有功率学习系统

| 节点 | 实机值 | 含义 |
|---|---|---|
| `start_learning` | 0 | 命令寄存器/状态字段，0 的状态语义未公开 |
| `stop_learning` | 0 | — |
| `learning_power` | 0 | Estimated Power（空闲） |
| `action_power` | 0 | Actual Power（空闲） |
| `learning_power_dev` | 0 | 功率偏差 |
| `learning_time_dev` | 0 | 时间偏差 |
| `constant_power` | 0 | 恒定功率参数 |
| `remaining_time` | 0 | 剩余时间预测 |
| `referance_power` | 0 | Reference Power（源码拼写 referance） |
| `referance_current` | 0 | 厂商推荐放电电流 |
| `nvt_referance_power` | 0 | NVT 推荐放电功率 |
| `start_learning_b` | 0 | B 组 start learning |
| `stop_learning_b` | 0 | B 组 stop learning |
| `learning_power_b` | 0 | B 组 Estimated Power |
| `action_power_b` | 0 | B 组 Actual Power |
| `learning_power_dev_b` | 0 | B 组功率偏差 |

采集时设备正在低电流充电（ibatt=289000 µA），但该组 learning 节点均为 0。说明正在充电并不等于自动触发 start_learning。单次静态快照不足以判断其触发条件，也不能据此认定其仅在充电或放电时工作。

详见《电池容量校准与SOH.md》第五节对 start_learning 机制的完整分析。

---

## 7. 安全/异常监控

| 节点 | 实机值 | 含义 |
|---|---|---|
| `peak_flag` | 0 | 无 Over Peak 峰值异常 |
| `isc` | 0 | 无内部短路告警（写操作是 fake override） |
| `soa` | 0 | 无 SOA / Safe Operating Area 告警 |
| `current_deviation` | 0 | 无电流偏差 |
| `power_deviation` | 0 | 无功率偏差 |
| `cutoff_vol` | 3050 | 截止电压 3050 mV |

安全状态全部正常，无异常告警。

---

## 8. 核心数据关系总览

```
设计容量:    6000 mAh
QMax:        6057 mAh  (QMax/Design = 100.95%，与 SOH=100 数值接近但因果关系未确认)
FCC:         5797 mAh  (FCC/Design = 96.6%，与 SOH_new=97 数值吻合但因果关系未确认)
RM:          2521 mAh  (RSOC=44%)
TFullChgQ:   5835      (FG 内部模型满充值，比 FCC 高 38)
TRemQ:       2605      (FG 内部模型剩余值，比 RM 高 84)
循环:        390 次
qmax_cyclecount: 280  (与 cyclecount 差 110，精确语义未确认)
生产日期:    2024-11-18
首用日期:    2025-02-05
使用时长:    约 19.3 个月
日均循环:    约 0.66 次/天
```

### SOH 体系对比

| SOH 指标 | 值 | 来源/算法 | MPC8011B 支持 |
|---|---|---|---|
| `soh` | 100 | MPC8011B 标准 SOH 寄存器返回值，算法未公开 | 是（标准寄存器） |
| `soh_new` | 97 | Xiaomi UI SOH 缓存 | 是（厂商私有） |
| `ui_soh` | 97 119 1 2 ... | MAC 0x007B 11 字节块 | 是（厂商私有） |
| `fcc_soh` | 0 | TI FCC_SOH | 当前返回 0，可能未启用/未实现 |
| `rel_soh` | 0 | TI Relative SOH | 当前返回 0，可能未启用/未实现 |
| `eis_soh` | 0 | TI EIS SOH | 当前返回 0，可能未启用/未实现 |

### 容量数据交叉验证

| 比值 | 计算值 | 对应指标 | 一致性 |
|---|---|---|---|
| QMax / Design | 6057/6000 = 100.95% | soh=100 | 数值接近，但因果关系未确认 |
| FCC / Design | 5797/6000 = 96.6% | soh_new=97 | 数值高度吻合，但因果关系未确认 |
| RM / FCC | 2521/5797 = 43.5% | rsoc=44 | 基本一致 |
| TRemQ / TFullChgQ | 2605/5835 = 44.6% | rsoc=44 | 一致 |

---

## 附：完整节点值原始记录

```
action_power = 0
action_power_b = 0
aged_flag = 0
average_current = 93
average_temperature = 63096
avercurrent = 93
batt_sn = SLBP5D4B18029163WMD1H00000000000
batt_use_environment = 0 0 253 25 0 0 7 0 187 31 0 0 177 0 17 3 0 0 0 0 20 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
calc_rvalue = 0
cell_vendor = lwn
charger_full = 5797000
chip_ok = 1
constant_power = 0
count_level1 = 1
count_level2 = 0
count_level3 = 0
count_lt = 0
current_deviation = 0
cutoff_vol = 3050
cyclecount = 390
design_capacity = 6000000
device_name = 1XM31
df_check = 33001
dod_count = 0
eeprom_version = C
eis_soh = 0
eis_soh_cyclecount = 0
fake_first_usage_date = 0
fast_charge = 0
fcc_soh = 0
first_usage_date = 20250205
ibatt = 289000
isc = 0
learning_power = 0
learning_power_b = 0
learning_power_dev = 0
learning_power_dev_b = 0
learning_time_dev = 0
manufacturing_date = 20241118
max_life_temp = 44
max_life_vol = 4519
max_temp_occur_time = 2780431
max_temp_time = 899857
min_life_temp = 6
min_life_vol = 2883
nvt_referance_power = 0
over_vol_duration = 0
pack_vendor = 2
peak_flag = 0
power_deviation = 0
qmax = 6057
qmax_cyclecount = 280
referance_current = 0
referance_power = 0
rel_soh = 0
rel_soh_cyclecount = 0
remaining_time = 0
resistance_id = 100000
rm = 2521000
rsoc = 44
run_time = 56771911
seal = 3
soa = 0
soh = 100
soh_new = 97
start_learning = 0
start_learning_b = 0
stop_learning = 0
stop_learning_b = 0
tambinet = 65232
temp = 291
temp_max = 44
tfullchgq = 5835
time_ht = 0
time_ot = 0
total_fw_runtime = 0
tremq = 2605
tsim = 3052
ui_soh = 97 119 1 2 97 100 100 100 100 100 207
vbatt = 3857
vendor = 3
```

---

## 采集说明

- 设备：Xiaomi K80 Pro (onyx)，Android 14，KernelSU root
- ADB 连接：192.168.33.118:5555（无线 ADB）
- 采集时状态：RSOC=44%，正在充电中（ibatt=289000 µA），非快充
- 驱动：Xiaomi MCA 多厂商共用框架 `bq27z561.c` / `strategy_fg`
- 节点路径：`/sys/class/xm_power/fg_master/`
- 本快照为静态采集，功率学习等动态参数在充电/放电场景下会变化
- 节点含义和 MPC8011B 私有语义解释基于 Xiaomi 开源驱动代码，部分含义为推断
- `fcc_soh`/`rel_soh`/`eis_soh` 当前返回 0，可能未启用/未实现/无有效值，尚不能确证 MPC8011B 不支持 TI 多 SOH 体系
