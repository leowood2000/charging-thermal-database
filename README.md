# 充电热控数据库（K80 Pro / miro）

Redmi K80 Pro（miro）充电热控完整数据库，全部数据来自设备本身（设备树 + 解密后的热控配置）。

## 内容

- `无线充电热控数据库.md`：无线充电热控完整数据库（温度→等级→电流/功率、充电场景对比、虚拟温度公式、场景清单）
- `有线充电热控数据库.md`：有线充电热控数据库（等级→电流表、快充协议矩阵、Quick Charge 电压/电流阶段曲线、SIC-BAT 温度→电流、MONITOR-BAT 电池等级、场景概览、实测验证）

## 数据链路（有线与无线共用）

```
物理传感器（电池/PA/充电IC/CPU/静音/WiFi）→ 虚拟皮肤温度（5 方向加权平均）
  → thermal-engine 按场景查阈值 → ctrl_limit 等级
    → 驱动查 wired/wireless_thermal 表 → 各路径 voter 投电流 → MCA 取最低票生效
```

## 数据来源

- 等级→电流：设备树 `mca_charger_thermal/wired_thermal` / `wireless_thermal`
- 有线 Quick Charge 曲线：设备树 `mca_strategy_quick_charge` / `mca_quick_charge_batt_para_gbl`（每点为电压、current_max、current_min）
- 电池抗老化叠加配置：`/odm/etc/charger/BAA_config_miro.json`；该配置的浮充/终止策略不等同于主热控限流表
- 温度→等级：`/vendor/etc/thermal-map.conf` + `/odm/etc/thermal-*.conf`
- 配置文件为 AES-128-CBC 加密，key/IV = `thermalopenssl.h`（工具：helloklf/vtools mi-thermal-config / hdzungx/ThermalMunch）
