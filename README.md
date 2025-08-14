地暖智能控制系统（Node‑RED）

Smart Radiant Floor Heating Controller (Node‑RED)

版本 / Version: V24.0（主控） + 附加混水泵站 PID 控制
日期 / Date: 2023‑10‑05（主控）
运行环境 / Runtime: Node‑RED（建议开启 file‑based context）

本项目包含两部分：

主控逻辑（根据房间需求、室外温度、阳光强度、水箱模型计算混合水目标温度）。

混水泵站/阀门开度 PID 控制（将温度误差转换为 12‑bit 设定点，适配 40–50 s 行程慢阀门）。

目录 / Table of Contents

简介 / Overview

主要特性 / Features

系统架构 / System Architecture

快速开始 / Quick Start

配置与上下文键 / Configuration & Context Keys

算法说明 / Algorithms

混水泵站/阀门开度 PID 控制 / Mixed Station Valve PID

调参建议 / Tuning Tips

安全与隐私 / Safety & Privacy

故障排查 / Troubleshooting

许可证 / License

简介 / Overview

该 Node‑RED 方案实现了一个改进型地暖智能控制器。主控逻辑每 10 秒计算一次混合水目标温度，综合了房间温度误差、室外温度、日照强度以及缓冲水箱热力学模型；附加逻辑将“目标温度 vs 当前温度”的偏差通过 PID 转成混水泵站/阀门 12‑bit 开度，并包含斜率限制，适配约 40–50 s 全行程的电动阀门。

主要特性 / Features

PID + 保护：主控使用 P/I/D、积分限幅与单步最大调整，防止过冲与积分饱和。

外部扰动补偿：室外温度线性补偿、阳光强度负向补偿（至多 −2 °C）。

水箱热模型：考虑自然冷却（约 6 h 下降 30 °C）、供热负荷与流量估计。

未达标补偿：对长期未达标房间增加权重与温度补偿（上限可配）。

锅炉工况意识：区分 buffer_heat（供热）、cooling（余热）、domestic（生活热水）并采用不同策略。

慢执行器友好：开度控制带速率限制与反积分保护，适配 40–50 s 行程球阀。

状态持久化：使用 Node‑RED flow context 保存状态（建议启用文件持久化）。

系统架构 / System Architecture
传感/状态输入 (HA/Modbus/MQTT/HTTP)
  ├─ 室外温度、日照、回水/混水/出水温度
  ├─ 锅炉燃烧/地暖加热状态、生活热水状态
  └─ 房间启停/目标温度/当前温度
            │
            ▼
      主控函数 (V24.0)
   - 房间加权误差 → PID
   - 外部补偿 & 场景补偿
   - 水箱温度更新（热负荷/自然冷却）
            │
            ├─ 输出：混合水目标温度 (°C, 整数)
            ▼
   混水泵站/阀门 PID 控制
   - 目标 vs 当前温度偏差
   - 输出 12-bit 设定点 (1–4095)
            │
            ▼
      执行器 (阀门/泵站/混水站)

快速开始 / Quick Start

准备 Node‑RED

推荐 Node‑RED 3.x+。

settings.js 开启文件型上下文：

contextStorage: {
  default: { module: "localfilesystem" }
}


导入两个 Function 节点

主控函数（V24.0）：以 Inject 节点每 10 s 触发。

混水泵站/阀门 PID：接入上游气候实体/温控器消息。

将传感器写入 flow 上下文（键名见下节）。

对接执行器

主控输出：混合水目标温度（整数 °C）。

PID 输出：msg.payload = set_point（1–4095），用于 12‑bit DAC/PWM/0–10 V 模块。

配置与上下文键 / Configuration & Context Keys
主控核心参数（代码顶部）

安全限幅：MIN_TEMP=30、MAX_TEMP=61、HYSTERESIS=8、MAX_STEP=1.0、MAX_INTEGRAL=10.0

主控 PID：KP=1.0、KI=0.05、KD=0.5

外部补偿：室外基准 5 °C、斜率 0.3；阳光斜率 -1/50000，最小（负）补偿 -2 °C

水箱模型：容量 80 L、比热 4.186 J/(g·°C)、密度 1000 kg/m³、自然冷却 30 °C/6h、额定流量估计 13 L/min

以上物理参数请按你的系统实测校准。

flow 上下文键（示例，需上游节点填充）

开关/状态

floorheating_state — "on"/"true"/"1"/"heat" 之一启用系统

boiler_burning_status — 锅炉燃烧

boiler_floorheating_status — 锅炉是否在加热地暖

boiler_hot_water_status — 是否在制备生活热水（供附加 PID 使用）

环境与水温

outside_temperature_state — 室外温度 (°C)

floorheating_watertank — 水箱温度 (°C)

floorheating_sunlight — 阳光强度 (Lux)

floorheating_returnwater_temperature_state / floorheating_mixedwater_temperature_state / floorheating_outputwater_temperature_state

房间（每个房间一组）

floorheating_<roomId>_di_nuan_state — 房间地暖开关

floorheating_<roomId>_di_nuan_current_temp — 当前温度

floorheating_<roomId>_di_nuan_target_temp — 目标温度

输出

主控节点：msg.payload = 混合水目标温度（整数 °C）

PID 节点：msg.payload = 开度设定点 1–4095

算法说明 / Algorithms

房间加权误差：按生活区权重与温差大小加权；对超过阈值未达标的房间加权并叠加温度补偿（上限可配）。

主控 PID 与保护：积分限幅、单步限幅、微分抗扰；最终目标温度被钳制在 [MIN_TEMP, MAX_TEMP]。

外部补偿：室外越冷目标越高；阳光越强目标越低（最低至 −2 °C）。

供热场景补偿：buffer_heat 维持、cooling −1 °C、domestic −2 °C。

水箱模型：加热期按 HEAT_RATE × Δt 升温；冷却期考虑自然冷却与 Q = m·c·ΔT 负荷。

混水泵站/阀门开度 PID 控制 / Mixed Station Valve PID

功能：将 目标温度 - 当前温度 偏差经 Kp/Ki/Kd 输出阀门/泵站开度（1–4095），并用斜率限制匹配慢阀门。
输入消息契约（来自上游气候/恒温器节点）：

msg.data.attributes.temperature           // 目标温度 (°C)
msg.data.attributes.current_temperature   // 当前温度 (°C)


代码中变量 traget_temperature 为拼写错误的历史兼容名，可保留或自行更正为 target_temperature。

依赖的 flow 上下文：

boiler_hot_water_status → true 时临时提升 Kp（加快响应）。

integral / previous_error / previous_level → PID 与斜率限制的内部记忆量。

核心逻辑（你的附加代码，原样收录）：

// 获取锅炉是否在制备热水
var boiler_hot_water_status = flow.get("boiler_hot_water_status") || false;

// 从 flow 中获取存储的水箱状态（温度）
var di_nuan_shui_xiang_state = flow.get("di_nuan_shui_xiang_state");

// 检查是否成功获取到状态
if (di_nuan_shui_xiang_state === undefined) {
    di_nuan_shui_xiang_state = 60;  // 默认值 60°C
}

// 目标温度与当前温度（来自上游消息）
var traget_temperature = msg.data.attributes.temperature;
var current_temperature = msg.data.attributes.current_temperature;

// 计算温差
var D_temperature = traget_temperature - current_temperature;
msg.D_temperature = D_temperature;

// PID 参数
var Kp = 8;
var Ki = 0.05;
var Kd = 0.5;

// 获取积分与上次误差
var integral = flow.get("integral") || 0;
var previous_error = flow.get("previous_error") || 0;

// 生活热水时提高比例增益
if (boiler_hot_water_status === true) {
    Kp = Kp * 1.5;
}

// 积分限制
var integral_limit = 10;
var min_threshold = 0.1;

// 小误差下抑制积分增长
if (Math.abs(D_temperature) > min_threshold) {
    integral += D_temperature;
} else {
    integral = Math.min(Math.max(integral, -integral_limit), integral_limit);
}

// 积分饱和保护
if (Math.abs(integral) >= integral_limit) {
    integral = integral > 0 ? integral_limit : -integral_limit;
}

var derivative = D_temperature - previous_error;
var level = Kp * D_temperature + Ki * integral + Kd * derivative;

// 记忆量更新
flow.set("integral", integral);
flow.set("previous_error", D_temperature);

// 动态调整 Kp
if (Math.abs(D_temperature) < 1) Kp = 5;
else if (Math.abs(D_temperature) > 3) Kp = 12;

// 将 level 限制到 [-4, 4]
level = Math.min(Math.max(level, -4), 4);

// 斜率限制（慢阀门）
var max_change = 0.1;
var previous_level = flow.get("previous_level") || 0;
level = previous_level + max_change * Math.sign(level - previous_level);
flow.set("previous_level", level);

// 映射到 1..4095
var set_point = Math.round(((level + 4) / 8) * 4094) + 1;

msg.payload = set_point;
return msg;


斜率/行程时间估算：若节点周期为 T 秒、每周期最大变化量为 max_change，则全行程时间 ≈ 8 / max_change × T。
例如：T=1 s、max_change=0.16 → 全行程约 50 s（可匹配你的 40–50 s 阀门）。

调参建议 / Tuning Tips

主控：震荡→减小 KP 或增大 KD；收敛慢→小幅提升 KP 或 KI（注意 MAX_INTEGRAL）。

外部补偿：采光强/外温波动大→可适当放大/缩小补偿斜率与上限。

PID（开度）：先以 Ki≈0 找到不过冲的 Kp，再逐步增加 Ki 消除稳态误差；用 max_change 校准阀门行程时间。

安全与隐私 / Safety & Privacy

安全限幅：目标温度被限制在 [30, 61] °C；致命异常进入安全模式输出最低温度。

最小暴露：公开仓库建议将房间 ID 抽象为 zone_1..n，真实映射放入未跟踪配置。

.gitignore 建议：

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


日志：默认 DEBUG_LEVEL=0，避免泄露运行模式与时间戳。

故障排查 / Troubleshooting

目标温度不变：检查 floorheating_state 是否启用，是否有激活房间。

输出总被夹住：查看是否触发安全限幅或补偿过强。

阀门过慢/过快：按行程时间调整 max_change 与节点周期。

积分发散/粘边：减小 Ki 或缩小 integral_limit，确认小误差门限生效。

缺少键告警：Debug 面板提示“无法获取…”，请补齐上游 flow 键。

许可证 / License

本项目基于 GNU General Public License v3.0（GPL‑3.0‑only） 发布。
请在分发该软件及其修改版本时遵循 GPLv3 的条款与条件（包括但不限于源代码开放、相同许可证传播等要求）。
仓库中应包含完整的 LICENSE 文件（GNU GPL v3 文本）。

为便于追溯，建议在源码文件头部加入 SPDX 标识（示例）：

SPDX-License-Identifier: GPL-3.0-only


免责声明 / Disclaimer：本项目不提供任何明示或暗示的保证。在适用法律允许的范围内，作者与贡献者不对任何损害承担责任。

贡献 / Contributing：欢迎通过 Issue/PR 提交问题与改进建议。
作者 / Author：JiangNanGenius
