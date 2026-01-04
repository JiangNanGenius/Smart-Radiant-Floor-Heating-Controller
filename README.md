
---

# 地暖智能控制系统（Node-RED）

Smart Radiant Floor Heating Controller (Node-RED)

**版本 / Version**: **v25.5 (FINAL)**
**最后更新 / Last updated**: 2026-01-04
**运行环境 / Runtime**: Node-RED（强烈建议启用 file-based context / localfilesystem）
**输出 / Outputs**: 7（目标温度显示、阀门开度、WS 同步/恢复、锅炉命令、强制回弹、供水温控器模式、严重报警）
**Tick / Timer**: 5 秒

---

## 目录 / Table of Contents

* [1. 简介 / Overview](#1-简介--overview)
* [2. 主要特性 / Features](#2-主要特性--features)
* [3. 系统架构 / System Architecture](#3-系统架构--system-architecture)
* [4. 快速开始 / Quick Start](#4-快速开始--quick-start)
* [5. 消息契约（7 路输出）/ Message Contract (7 Outputs)](#5-消息契约7-路输出-message-contract-7-outputs)
* [6. 上下文键与传感器分辨率 / Context Keys & Sensor Resolution](#6-上下文键与传感器分辨率--context-keys--sensor-resolution)
* [7. 启停逻辑 / Enable-Disable Logic](#7-启停逻辑--enable-disable-logic)
* [8. 算法详解 / Algorithms Deep Dive](#8-算法详解--algorithms-deep-dive)
* [9. 阀门控制 V2（前馈+PID+限速）/ Valve Control V2](#9-阀门控制-v2前馈pid限速--valve-control-v2)
* [10. WS 同步与恢复协议 / WS Sync & Recover Protocol](#10-ws-同步与恢复协议--ws-sync--recover-protocol)
* [11. 健康监控与报警 / Health Monitoring & Alarms](#11-健康监控与报警--health-monitoring--alarms)
* [12. Node Status 指示含义 / Node Status Meaning](#12-node-status-指示含义--node-status-meaning)
* [13. 调参与校准流程 / Tuning & Calibration](#13-调参与校准流程--tuning--calibration)
* [14. 与 Home Assistant 对接建议 / Home Assistant Integration](#14-与-home-assistant-对接建议--home-assistant-integration)
* [15. 故障排查 / Troubleshooting](#15-故障排查--troubleshooting)
* [16. FAQ / 常见问题](#16-faq--常见问题)
* [17. 变更记录 / Changelog](#17-变更记录--changelog)
* [18. 安全与隐私 / Safety & Privacy](#18-安全与隐私--safety--privacy)
* [19. 许可证 / License](#19-许可证--license)

---

## 1. 简介 / Overview

这是一个 **工程化可长期无人值守运行** 的地暖综合控制器，核心为一个 Node-RED Function 节点（主控）。每 5 秒运行一次，完成：

* 读取房间状态（heat/off + 当前温度 + 目标温度）
* 读取系统传感器（混水/回水/出水/水箱/户外/日照/流量）
* 推断锅炉工况（buffer_heat / domestic / cooling）
* 计算热需求（目标功率 kW）
* 计算目标混水温度（含户外曲线、日照补偿、锅炉场景惩罚、水箱上限）
* 平滑目标温度（抑制跳变）
* 阀门控制（前馈比例 + PID 微调 + 设定点限速 + 防抖）
* 系统健康监控与严重报警（微信推送 + 去重）
* 状态持久化 + WS 同步快照 + 重启自动恢复（recover）

**English (short):**
A production-grade radiant floor heating controller implemented in Node-RED. Every 5 seconds it computes the target mixed-water temperature from room demand and sensors, drives a slow mixing valve using feedforward+PID with slew limits, syncs state snapshots via WS, supports restart recovery, and emits severe fault alarms with deduplication.

---

## 2. 主要特性 / Features

* ✅ **单 Function 一体化**：启停、目标温度、阀门、锅炉命令、供水模式、恢复、报警全部集成（7 Outputs）。
* ✅ **工程型阀门控制 V2**：

  * 前馈比例（基于水箱与回水解混合比例）
  * PID 微调（死区、积分限幅、不可达保护）
  * 设定点限速（点/秒）与最小步进（防抖）
  * 适配 **40–50 秒全行程**的慢阀
* ✅ **外部扰动补偿**：户外越冷目标越高；日照越强目标越低（最多 -2°C）。
* ✅ **锅炉工况意识**：domestic（生活热水）场景降温策略，避免抢热导致系统抖动。
* ✅ **健康监控只盯“系统本体异常”**：避免关机流量归零误报；含启动宽限。
* ✅ **严重报警去重**：同 code 3 小时内只推一次，防刷屏。
* ✅ **WS 快照同步/恢复**：重启后无本地状态自动 recover（最多 2 次），恢复完成后继续快照同步。
* ✅ **node.status 可读性强**：ON/OFF、锅炉短标记、目标/混水/阀门/流量/功率一眼可见。

---

## 3. 系统架构 / System Architecture

```text
[HA/Modbus/MQTT/HTTP] → 上游采集节点 → flow context 写入
    ├─ 户外温度 outside_temperature_state (0.1°C)
    ├─ 水箱温度 floorheating_watertank (0.1°C)
    ├─ 日照 floorheating_sunlight (Lux)
    ├─ 混水/回水/出水温度 (mixed 0.1°C / return 1°C / output 1°C)
    ├─ 地暖总流量 floorheating_waterflow_state (0.001 m³/h)
    ├─ 锅炉燃烧 boiler_burning_status
    └─ 房间：floorheating_<room>_di_nuan_state/current/target
                   │
                   ▼
      [Function] 地暖综合控制主代码 v25.5 (每 5 秒)
      - enable/disable 判定 + 强制回弹
      - 目标功率 kW 估算
      - 目标混水温度候选 + 平滑
      - 阀门控制 V2（前馈+PID+限速）
      - 健康监控 + 严重报警（去重）
      - state 持久化 + WS 快照同步 + recover
                   │
                   ├─ Output1 目标温度显示（整数）
                   ├─ Output2 阀门 1..4095
                   ├─ Output3 WS: snapshot JSON / "recover"
                   ├─ Output4 锅炉 heat/off
                   ├─ Output5 enforce 回弹 off
                   ├─ Output6 供水温控器模式 heat/off
                   └─ Output7 微信报警 title + 多行 payload
```

---

## 4. 快速开始 / Quick Start

### 4.1 Node-RED 必备设置（强烈推荐）

启用文件型上下文：**防重启丢状态**（非常重要）

在 `settings.js` 中：

```js
contextStorage: {
  default: { module: "localfilesystem" }
}
```

> 不启用 file-based context 也能跑，但重启后会更依赖 recover，并且 PID/EMA/欠温计时等会被重置。

### 4.2 最小可运行的 Flow 结构

你需要至少这三部分：

1. **Inject 定时器（5 秒）** → 触发主控 Function
2. **上游采集写 flow**：把传感器、房间状态写入 `flow.set(...)`
3. **下游执行**：消费 7 路输出（阀门、锅炉、供水模式、WS、报警等）

### 4.3 运行前检查清单

* [ ] `outside_temperature_state` 有值（0.1°C）
* [ ] `floorheating_watertank` 有值（0.1°C）
* [ ] `floorheating_mixedwater_temperature_state` 有值（0.1°C）
* [ ] `floorheating_returnwater_temperature_state` 有值（1°C）
* [ ] `floorheating_waterflow_state` 有值（0.001 m³/h）
* [ ] 至少一个房间 `<id>_di_nuan_state = heat`
* [ ] 锅炉状态（至少 burning/floorheating）有值（可空但影响判断）
* [ ] Output2 下游执行器可以接受 1..4095（并已做限幅/安全）

---

## 5. 消息契约（7 路输出）/ Message Contract (7 Outputs)

> 下面是“主控 Function 每次 tick 的输出数据格式”。你可以据此写 Switch/MQTT/HA action 节点，不会踩坑。

### Output1：目标温度显示（仅前端展示）

* **topic**: `fh/target_display`
* **payload**: `"44"`（字符串，整数 °C）

示例：

```json
{ "topic": "fh/target_display", "payload": "44" }
```

### Output2：阀门开度（12-bit）

* **topic**: `fh/valve_sp`
* **payload**: `1920`（number，范围 1..4095；OFF 时为 4095）

示例：

```json
{ "topic": "fh/valve_sp", "payload": 1920 }
```

> 注：你的策略是“系统关闭时输出 4095（快开+排气策略）”。

### Output3：WS 同步 / 恢复（复用口）

* **topic**: `fh_sync`
* **payload**:

  * 正常：`JSON.stringify(stateObj)`（字符串）
  * 恢复请求：`"recover"`（字符串，最多 2 次）

示例（正常快照）：

```json
{ "topic": "fh_sync", "payload": "{\"sys\":{\"enabled\":true,...}}" }
```

示例（请求恢复）：

```json
{ "topic": "fh_sync", "payload": "recover" }
```

### Output4：锅炉地暖命令（重复下发）

* **topic**: `boiler/floorheating`
* **payload**: `"heat"` 或 `"off"`

示例：

```json
{ "topic": "boiler/floorheating", "payload": "heat" }
```

### Output5：强制开关回弹（enforce reset）

* **topic**: `enforce_reset`
* **payload**: `"off"`（当检测到按下时才会输出一次）

示例：

```json
{ "topic": "enforce_reset", "payload": "off" }
```

### Output6：物理供水温控器模式（climate mode）

* **topic**: `climate/dinuan/gongshui/mode/set`
* **payload**: `"heat"` 或 `"off"`

示例：

```json
{ "topic": "climate/dinuan/gongshui/mode/set", "payload": "heat" }
```

### Output7：严重报警（微信推送）

* **title**: 简短标题（放在 `msg.title`）
* **payload**: 多行详细说明（放在 `msg.payload`）
* （内部带 code 去重逻辑，不必下游处理去重）

示例：

```json
{
  "title": "地暖流量异常",
  "payload": "现象：地暖已开启且需要升温，但流量持续接近 0（严重）\n当前：目标45.0℃ 混水38.2℃ 水箱55.1℃ 流量0.00m³/h 阀门3000\n建议：检查循环泵/过滤器/排气/阀门/流量计"
}
```

---

## 6. 上下文键与传感器分辨率 / Context Keys & Sensor Resolution

### 6.1 传感器分辨率（与你代码注释一致）

* `floorheating_mixedwater_temperature_state`: **0.1°C**
* `floorheating_watertank`: **0.1°C**
* `outside_temperature_state`: **0.1°C**
* `floorheating_outputwater_temperature_state`: **1°C**
* `floorheating_returnwater_temperature_state`: **1°C**
* `floorheating_waterflow_state`: **0.001 m³/h**
* 房间 current/target：**1°C**（整度），房间模式无 auto，仅 heat/off

### 6.2 主控读取的 flow keys（完整表）

#### 系统/锅炉（SYSTEM.KEY.*）

* `boiler_burning_status`：锅炉是否燃烧（on/true/1/heat 等）
* `boiler_floorheating_status`：锅炉是否在地暖/缓冲加热
* `boiler_hot_water_status`：生活热水状态（本版本主控不用于推断，但保留）

#### 环境/水温

* `outside_temperature_state`
* `floorheating_watertank`
* `floorheating_sunlight`
* `floorheating_mixedwater_temperature_state`
* `floorheating_returnwater_temperature_state`
* `floorheating_outputwater_temperature_state`
* `floorheating_waterflow_state`

#### 房间（默认 ids）

* `floorheating_can_ting_di_nuan_state/current_temp/target_temp`
* `floorheating_ke_ting_di_nuan_state/current_temp/target_temp`
* `floorheating_shu_fang_di_nuan_state/current_temp/target_temp`
* `floorheating_zhu_wo_di_nuan_state/current_temp/target_temp`
* `floorheating_wo_shi_1_di_nuan_state/current_temp/target_temp`
* `floorheating_wo_shi_2_di_nuan_state/current_temp/target_temp`
* `floorheating_wo_shi_3_di_nuan_state/current_temp/target_temp`

#### 启停/强制

* `floorheating_state_enforce`：强制按钮（按下触发一次并回弹）

#### 主控状态存储（RECOVER_CONFIG / SYSTEM.STORAGE_KEY）

* `fh_v24_final`：主控 state 对象（建议保留；改名需同步改代码）
* `fh_remote_state_ready`：恢复写回完成标志（由“恢复写回节点”置 true）
* `fh_recover_ctrl`：recover 内部控制对象（context）
* `fh_recover_applied`：避免重复应用恢复（context）
* `fh_alarm_last_by_code`：报警去重 map（context）
* `fh_health`：健康监控计时/基线（context）
* `integral` / `previous_error`：阀门 PID 记忆量（flow）

---

## 7. 启停逻辑 / Enable-Disable Logic

### 7.1 房间判定

* 任意房间 `state === "heat"`（或 on-like）→ `anyOpen=true`
* 所有房间都不 heat → `allClosed=true`

### 7.2 户外阈值与强制开关

默认阈值 `OUTDOOR_THRESHOLD_C = 16`：

* **开**：`anyOpen && outdoor < 16`
* **关**：`allClosed && outdoor > 16`
* **应急开**：`enforcePressed && anyOpen`（触发一次并自动回弹 off）

> enabled 状态会同步写入：`flow.set("floorheating_state", enabled)` 供其它 flow 使用。

---

## 8. 算法详解 / Algorithms Deep Dive

### 8.1 目标功率（kW）估算：从“房间舒适”到“热需求”

主控用两条线叠加：

1. **demandIndex（房间加权欠温指数）**

   * 只看开启（heat/on）的房间
   * 欠温 `diff = tgt - cur`
   * 客厅/餐厅权重更大
   * 长期欠温会额外加权（underheat duration + comp）

2. **comfort（最大欠温 emax + hold 策略）**

   * emax ≥ 1°C 持续 10min → 功率 +1kW
   * emax ≤ 0°C 持续 60min → 功率 -1kW
   * 目的：舒适优先 + 慢速节能收敛（避免频繁震荡）

最终：

* `P_target = KW_PER_INDEX*demandIndex + KW_PER_C*max(0,emax) ± STEP_KW`
* clamp 到 `0..TARGET_MAX_KW`（默认 20kW）

### 8.2 目标 ΔT：从“功率”到“需要的温差”

如果流量足够：
[
\Delta T \approx \frac{P_{target}}{1.163 \cdot Q}
]
否则用经验值（无流量时也要给出目标以推动阀门/系统进入可工作区）。

### 8.3 目标温度候选 T_candidate（叠加修正）

基本结构：

* `T_candidate = Tr + ΔT_need`
* 户外修正（越冷越高）
* 日照修正（越亮越低，最多 -2°C）
* 锅炉模式惩罚：domestic -2°C / cooling -1°C
* 最低不低于户外曲线最小值
* 最高不超过水箱温度（避免定一个热源根本达不到的目标）

### 8.4 目标温度平滑（强烈重要）

采用：

* 一阶惯性滤波（tau=20s）
* 每 tick 最大步进限制（默认 0.35°C/5s）

目的：**抑制目标跳变 → 减少阀门忙碌 → 系统更稳**。

---

## 9. 阀门控制 V2（前馈+PID+限速）/ Valve Control V2

阀门输出为 1..4095（12-bit），并且有**不可达保护**与**限速**，适配慢阀（40–50s 行程）。

### 9.1 温度反馈选择（优先级）

* mixedWaterTemp（主反馈）
* outputWaterTemp（兜底）
* returnWaterTemp（再兜底）

### 9.2 前馈比例（关键）

如果 `T_tank` 与 `T_return` 可用：
[
ratio = \frac{T_{target}-T_{return}}{T_{tank}-T_{return}}
]

* ratio 限制到 0..1
* 做 EMA 平滑（RATIO_TAU_S）
* 映射到 1..4095 得到 `sp_ff`

前馈的意义：**阀门大致一次到位**，PID 只负责小修正。

### 9.3 不可达（unreachable）保护

当水箱温度接近/低于目标：

* 进入条件：`T_tank <= T_target + 0.2°C`
* 退出条件：`T_tank >= T_target + 1.0°C`
  不可达时：
* 直接 `sp_ff = 4095`（全开）
* PID 积分停止增长（避免 windup）

### 9.4 PID 微调（死区 + 积分限幅）

* err = `T_target - T_feedback`
* DEAD_BAND_C 以内 err=0
* 积分限幅 INT_LIMIT
* 输出叠加到 `sp_ff` 上形成 `sp_target`

### 9.5 设定点限速 + 防抖

* `SP_SLEW_PER_SEC`：点/秒（越小越稳）
* `SP_MIN_STEP`：小于此阈值直接不动（防止抖动）
* 最终写回 `state.valveLast.set_point`

---

## 10. WS 同步与恢复协议 / WS Sync & Recover Protocol

> 主控本身只负责从 Output3 发消息；**恢复写回/远端存储**要靠你在 Node-RED 里另一个 flow 来实现（你之前的 A/B 实例通信体系正好用得上）。

### 10.1 Output3 行为

* 如果本地 `flow.get(STORAGE_KEY)` 没数据，且 `fh_remote_state_ready != true`：
  主控会在 timer tick 上发 `"recover"`，最多 `MAX_ATTEMPTS=2` 次。
* 若 ready=true：主控每 tick 都发 `JSON.stringify(state)` 快照。

### 10.2 推荐的“远端状态仓库”实现（示例逻辑）

**远端（B 节点）**：

* 收到 topic `fh_sync`：

  * payload 是 JSON：保存到 B 自己的持久化存储（file context / disk / kv）
  * payload 是 `"recover"`：把最新快照回发给 A（HTTP/WS/MQTT 任选）

**本地（A 节点）恢复写回节点**（必须做）：

* 当收到远端回传的快照（对象或 JSON）：

  * `flow.set("fh_v24_final", snapshotObj)`
  * `flow.set("fh_remote_state_ready", true)`

> 主控内部会检测 ready=true 且 applied=false，然后只应用一次（防重复）。

### 10.3 推荐你在仓库里附带的 Recover 写回 Function（模板）

你可以在 README 里给一个标准模板，方便复制：

```js
// Recover write-back node (runs on A)
// input: msg.payload = snapshot (object or json string)
const STORAGE_KEY = "fh_v24_final";
const READY_FLAG_KEY = "fh_remote_state_ready";

let s = msg.payload;
if (typeof s === "string") {
  try { s = JSON.parse(s); } catch(e) { return null; }
}
if (!s || typeof s !== "object") return null;

flow.set(STORAGE_KEY, s);
flow.set(READY_FLAG_KEY, true);

// optional: status
node.status({ fill:"green", shape:"dot", text:"recover applied" });

return null;
```

---

## 11. 健康监控与报警 / Health Monitoring & Alarms

健康监控只在：

* `enabled=true` 且 `anyOpen=true` 时启用
  否则立即 reset（避免关机误报）。
  并且有启动宽限：`STARTUP_GRACE_S=180` 秒。

### 11.1 报警去重

同一报警 code（如 FLOW0 / MIX_STALE / NO_RISE / CODE_ERR）
在 `SAME_CODE_DEDUP_MS`（默认 3 小时）内只发送一次。

### 11.2 报警类型与判定（简明）

* **FLOW0（地暖流量异常）**

  * 需要升温（目标-混水 ≥ 1.5°C）
  * 阀门已很大（≥ 2600）
  * 流量 < 0.05 m³/h 持续 ≥ 120s
  * 建议检查：循环泵/过滤器/排气/阀/流量计

* **MIX_STALE（混水温度采集异常）**

  * 需要升温
  * 流量 ≥ 0.10 m³/h
  * 混水温度 600s 没变化
  * 建议检查：mixed 传感器/采集链路/Modbus 节点

* **NO_RISE（地暖升温异常）**

  * 需要升温
  * 流量 ≥ 0.15 m³/h
  * 锅炉 buffer_heat 且燃烧，并且水箱比混水至少高 2°C（热源可用）
  * 600s 内混水上升 < 0.2°C
  * 建议检查：阀方向/旁通短路/锅炉是否真正给水箱加热

* **CODE_ERR（程序异常）**

  * try/catch 捕获到错误
  * 进入安全输出并推送报警

---

## 12. Node Status 指示含义 / Node Status Meaning

### 12.1 锅炉短标记（boilerTag）

* `--`：不燃烧/未知
* `B+FH`：燃烧且在地暖/缓冲加热（buffer_heat）
* `B+HW`：燃烧但非地暖（推断 domestic）
* `B+`：燃烧但模式未知（理论上少见）

### 12.2 OFF 显示

示例：

```
⛔OFF|B+HW|🧩4095|🔄1/2
```

含义：

* 系统关闭
* 当前锅炉在生活热水（或非地暖燃烧）
* 阀门输出 4095（快开/排气策略）
* recover 正在进行（第 1 次，共 2 次）

### 12.3 ON 显示

示例：

```
🔥ON|B+FH|🎯45.0|🌡️38.2|🧩1920|💧0.23|⚡6.1/7.0
```

含义：

* 系统开启
* 锅炉地暖加热
* 目标 45.0°C（控制用）
* 混水 38.2°C
* 阀门 1920
* 流量 0.23 m³/h
* 功率：EMA 6.1kW / 目标 7.0kW

### 12.4 有报警时

显示类似：

```
⚠️地暖流量异常|B+FH
```

---

## 13. 调参与校准流程 / Tuning & Calibration

> 强烈建议按顺序来：先稳定 → 再响应 → 再节能 → 最后报警阈值。

### 13.1 基础校准（必须）

1. **确认流量单位**：`floorheating_waterflow_state` 必须是 **m³/h**
2. **确认温度分辨率**：mixed/tank/outdoor 必须有小数 0.1
3. **确认房间状态**：`heat/off`（避免出现 auto/unknown）
4. **确认阀门方向**：输出增大时，混水温度应上升（否则阀方向/接线/映射可能反了）

### 13.2 目标温度平滑（决定“系统是否舒服”）

* 想更稳：增大 `TARGET_TAU_S` 或减小 `TARGET_MAX_STEP_PER_TICK_C`
* 想更快：反向调整，但阀门会更忙、更容易抖

建议初始策略：先稳后快。

### 13.3 阀门限速（最关键）

`SP_SLEW_PER_SEC` 是“阀门动作速度旋钮”：

* 太小：系统追不上目标，升温慢
* 太大：容易抖动，噪声与磨损增加

你阀门全行程 40–50 秒，粗略目标：

* 满量程 4094 点 / 50 秒 ≈ 82 点/秒
  所以如果你想“接近满速”，可以把 `SP_SLEW_PER_SEC` 调到 ~80 左右；
  如果你想“更稳”，就用 30~60。

> 推荐：先从 30~50 稳定跑一晚，再逐步加。

### 13.4 PID 微调（只做小修正）

* 震荡/来回抖：减小 `Kp_sp`、减小 `Ki_sp`、增大 DEAD_BAND
* 总差一点上不去：略增 `Ki_sp` 或放宽积分限幅
* 不要依赖 PID 做大动作：大动作应由前馈比例完成

### 13.5 健康报警阈值（最后调）

* FLOW 阈值要匹配你的流量计与泵
* NO_RISE 的 hold 时间要匹配你的系统热惯性（地暖惯性大可设更久）

---

## 14. 与 Home Assistant 对接建议 / Home Assistant Integration

### 14.1 房间 climate 实体（最常见）

你可以用 HA 的状态变化节点，把每个房间写入 flow：

* `flow.set("floorheating_<id>_di_nuan_state", msg.data.state)`
* `flow.set("floorheating_<id>_di_nuan_current_temp", msg.data.attributes.current_temperature)`
* `flow.set("floorheating_<id>_di_nuan_target_temp", msg.data.attributes.temperature)`

### 14.2 供水温控器模式（Output6）

Output6 topic 固定：`climate/dinuan/gongshui/mode/set`
下游可直接接 HA `call-service`（climate.set_hvac_mode）或 MQTT bridging。

### 14.3 锅炉命令（Output4）

你现在是每 tick 下发 `"heat"/"off"`，建议下游做一个“幂等/限频”层（可选）：

* 若 60 秒内命令相同则不重复调用 service（减少日志与调用压力）

### 14.4 微信报警（Output7）

只要下游节点支持：

* `msg.title` 当标题
* `msg.payload` 当内容（多行）
  即可直接接入。

---

## 15. 故障排查 / Troubleshooting

* **一直 OFF**

  * 检查是否任意房间是 `heat`
  * 检查户外温度是否 < 16°C（默认阈值）
  * 检查 `floorheating_<id>_di_nuan_state` 是否写入正确键名

* **阀门一直 4095**

  * OFF 分支：设计如此（快开/排气）
  * ON 分支：可能触发 unreachable（水箱达不到目标）或传感器缺失保护
  * 检查水箱温度是否异常偏低/偏高、mixed 是否为 0

* **混水温度不变化**（或报 MIX_STALE）

  * 检查 mixed 传感器是否冻结、Modbus 是否断
  * 检查采集节点是否卡住

* **有流量但不升温（NO_RISE）**

  * 检查阀门方向
  * 检查是否存在旁通短路
  * 检查锅炉是否真正给水箱加热（buffer_heat 状态是否可靠）

* **recover 不生效**

  * 检查 Output3 下游是否真的把快照写回 `fh_v24_final`
  * 是否置 `fh_remote_state_ready=true`
  * 检查网络/HTTP 同步链路

---

## 16. FAQ / 常见问题

**Q1：为什么系统关闭时阀门输出 4095？**
A：这是你的“快开+排气”策略。关闭时全开可帮助排气/减少局部滞留（按你的系统设计）。如果你想“关闭时关小”，改 OFF 分支输出即可，但要考虑你原本的目的。

**Q2：为什么锅炉命令每 5 秒都发一次？**
A：为了“反复下发确保可靠”。如果你不想这么频繁，可以在 Output4 下游加一层限频或仅状态变化时下发。

**Q3：为什么 state key 叫 fh_v24_final，但版本是 v25.5？**
A：这是历史兼容命名。可以改，但要同时改 `RECOVER_CONFIG.STORAGE_KEY` 与所有恢复写回节点。

**Q4：我房间多/少怎么办？**
A：改 `readRoomModes()` 里的 ids 列表，并确保上游写入同名 keys。

**Q5：我的流量单位不是 m³/h 怎么办？**
A：必须统一。若你是 L/min：
`m³/h = (L/min) * 0.06`。
建议在写入 `floorheating_waterflow_state` 前转换好。

---

## 17. 变更记录 / Changelog

### v25.5 (FINAL)

* 7 路输出一体化（目标显示/阀门/WS同步恢复/锅炉命令/enforce回弹/供水模式/严重报警）
* recover：无本地状态自动请求 `"recover"`（最多 2 次），支持远端写回后自动应用
* node.status 增加锅炉短标记（B+FH/B+HW/--）与 recover 进度
* 阀门控制升级为 V2：前馈比例 + PID + 限速 + 防抖 + 不可达保护
* 健康监控与严重报警：FLOW0 / MIX_STALE / NO_RISE + 3h 去重

### v24.x（历史）

* 主控与阀门 PID 分离或较弱耦合（README 旧版描述）
* 更少的输出通道与较弱的恢复机制

---

## 18. 安全与隐私 / Safety & Privacy

* **异常安全降级**：catch 分支输出阀门 4095、锅炉 off、供水 off，并发送 CODE_ERR 报警。
* **敏感信息剥离**：公开仓库建议隐藏真实房间命名（用 zone_1..n），真实映射放私有配置。
* **.gitignore 建议**：

```text
flows_*_cred.json
flows_*.json.backup
*.backup
*.bak
config.local.json
*.local.json
.env
.env.*
secrets.*
secrets.yaml
*.key
*.pem
```

---

## 19. 许可证 / License

本项目基于 **GNU General Public License v3.0（GPL-3.0-only）** 发布。
建议在源码文件头加入：

```text
SPDX-License-Identifier: GPL-3.0-only
```

> **免责声明 / Disclaimer**：本项目不提供任何明示或暗示的保证。请在理解风险的前提下使用，并做好硬件温控上限、手动应急、断电保护等安全措施。

---

## 附录 A：推荐的“上游写 flow”模板（可复制）

> 你可以在 Node-RED 里用多个 Function 节点接 HA state changed，把数据标准化后写入 flow。

```js
// Example: write outdoor temperature
const v = Number(msg.payload);
flow.set("outside_temperature_state", Number.isFinite(v) ? v : 0);
return null;
```

房间模板（id 替换成 can_ting/ke_ting/...）：

```js
const id = "can_ting";
const data = msg.data;

if (data && data.attributes) {
  flow.set(`floorheating_${id}_di_nuan_state`, data.state); // heat/off
  flow.set(`floorheating_${id}_di_nuan_current_temp`, data.attributes.current_temperature);
  flow.set(`floorheating_${id}_di_nuan_target_temp`, data.attributes.temperature);
}
return null;
```

---

