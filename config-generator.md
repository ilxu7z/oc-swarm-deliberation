# 配置生成器 — Config Generator

## 用途

在 Layer 2 推演启动前，用 LLM 分析问题和背景，自动生成推演配置。
包括：维度选择、模型分配、角色温度、推演轮数、收敛条件等。

灵感来源：MiroFish 的 `SimulationConfigGenerator` —— 分步生成 + JSON 修复 + 规则回退。

---

## 分步生成协议

```
Phase 0: 智能配置生成
  │
  ├─ Step 1: 问题类型识别
  │   ├─ 输入：用户的问题描述 + 背景信息
  │   ├─ LLM 判断：决策类 / 创意类 / 预测类 / 混合类
  │   ├─ 输出：problem_type + 推荐维度组合
  │   └─ 规则回退：无法判断 → 默认 "decision" + 时间线+成本收益+极端场景
  │
  ├─ Step 2: 参与者配置
  │   ├─ 输入：问题 + 问题类型 + 利益相关方初步分析
  │   ├─ LLM 生成：
  │   │   ├─ 角色列表（含 model / temperature / stance）
  │   │   ├─ 维度匹配（从 12 维中选 3-4 个）
  │   │   └─ 推演轮数建议（2-5 轮）
  │   └─ 规则回退：见下方硬编码默认表
  │
  ├─ Step 3: 收敛条件设定
  │   ├─ convergence_threshold：0.7-0.9（默认 0.8）
  │   ├─ max_rounds：5（默认）
  │   ├─ min_rounds：2（即使已收敛也不在 1 轮停止）
  │   └─ stale_limit：连续 2 轮无变化 → 强制停止
  │
  └─ Step 4: 输出并保存
      ├─ 保存到 .openclaw-project/deliberation-config.json
      └─ 供 Layer 2 的 Phase 1-5 直接读取
```

---

## LLM 调用 Prompt

### 配置生成 Prompt（Steps 1-3 一步到位）

```
你是一个群体推演配置生成器。分析给定的推演问题，自动生成最优的推演参数配置。

## 输入
- 问题：【问题描述】
- 背景：【背景信息】

## 输出格式（纯 JSON，不要其他内容）
{
  "problem_type": "decision|creative|prediction",
  "problem_analysis": "一句话分析",
  "dimensions": ["维度1", "维度2", "维度3", "维度4"],
  "rounds": 3,
  "convergence_threshold": 0.8,
  "max_rounds": 5,
  "min_rounds": 2,
  "personas": [
    {
      "name": "具体角色名（不要用A/B/C，要有辨识度）",
      "stance": "supportive|opposing|neutral|observer",
      "model": "deepseek/deepseek-v4-flash",
      "temperature": 0.7
    }
  ],
  "report_model": "deepseek/deepseek-v4-pro",
  "report_temperature": 0.3,
  "reasoning": "配置理由说明（1-3句）"
}

## 可用模型
- deepseek/deepseek-v4-flash：快速、低成本，适合多数角色
- zhipu2/glm-5-turbo：中等速度，中文深度分析
- zhipu2/glm-5.1：强推理，适合关键反对者或中立专家
- qwen/qwen3.5-122b-a10b：视觉分析强，一般推演用不上

## 规则
1. personas 数量 5-8 个
2. 必须包含至少 1 个明确反对者
3. 模型选型原则：反对者用更强模型（更尖锐的反驳），支持者可用轻量模型
4. 温度差异化：反对者低温（0.3-0.5），创意角色高温（0.8-1.0），中立者中温（0.5-0.7）
5. 报告生成必须用最强模型 + 最低温度
6. 维度从以下 12 维中选 3-4 个：
   决策类推荐：时间线、成本收益、极端场景、博弈均衡
   创意类推荐：类比迁移、反向思维、约束解除、极限推演
   预测类推荐：变量敏感性、路径依赖、群体动力学、黑天鹅
```

---

## JSON 修复逻辑

LLM 输出的 JSON 可能有格式问题。修复流程：

```
1. 尝试直接 JSON.parse()
2. 失败 → 检测常见问题：
   ├─ 未闭合的 [] 或 {} → 补全
   ├─ 字符串中有换行 → 替换为 \n
   ├─ 尾部截断 → 移除最后一个不完整的条目
   ├─ 多余的注释/文本 → 正则提取 JSON 部分
   └─ 仍失败 → 用规则回退默认配置
```

实现方式：主 Agent 用 `read` 读取子 Agent 输出，在对话中自行修复。不需要额外代码。

---

## 规则回退默认表

当 LLM 配置生成失败时，按问题类型使用硬编码默认：

| 问题类型 | 维度组合 | 轮数 | 温度范围 | 模型策略 |
|---------|---------|------|---------|---------|
| decision | 时间线 + 成本收益 + 极端场景 + 博弈均衡 | 3 | 0.3-0.7 | Flash 为主，反对者用 Pro |
| creative | 类比迁移 + 反向思维 + 约束解除 + 极限推演 | 2 | 0.7-1.0 | 全部 Flash |
| prediction | 变量敏感性 + 路径依赖 + 群体动力学 + 黑天鹅 | 3 | 0.4-0.6 | Flash 为主 |
| unknown（兜底） | 时间线 + 成本收益 + 极端场景 | 2 | 0.5-0.7 | 全部 Flash |

角色默认（任何类型）：
- 1 个明确反对者（Pro 模型，低温 0.3）
- 2-3 个支持者（Flash 模型，中温 0.5-0.7）
- 1-2 个中立/观察者（Flash 模型，中温 0.5）
- 报告：Pro 模型，温度 0.3

---

## 配置 JSON 完整格式

```json
{
  "problem_type": "decision",
  "problem_analysis": "是否应该关闭线下门店，全面转为线上销售",
  "dimensions": ["时间线", "成本收益", "极端场景", "博弈均衡"],
  "rounds": 3,
  "convergence_threshold": 0.8,
  "max_rounds": 5,
  "min_rounds": 2,
  "stale_limit": 2,
  "personas": [
    {
      "name": "焦虑的实体店主",
      "stance": "opposing",
      "model": "deepseek/deepseek-v4-pro",
      "temperature": 0.3
    },
    {
      "name": "电商运营总监",
      "stance": "supportive",
      "model": "deepseek/deepseek-v4-flash",
      "temperature": 0.7
    },
    {
      "name": "财务VP",
      "stance": "neutral",
      "model": "deepseek/deepseek-v4-flash",
      "temperature": 0.5
    }
  ],
  "report_model": "deepseek/deepseek-v4-pro",
  "report_temperature": 0.3,
  "reasoning": "涉及多方利益博弈，选决策类维度。反对者用强模型确保反驳质量。"
}
```

---

## 与 Layer 2 的对接

配置 JSON 的各字段被 Layer 2 的不同阶段消费：

```
deliberation-config.json
  │
  ├─ Phase 0.5: 展示配置给用户确认（可选）
  │
  ├─ Phase 1: 人格工厂
  │   └─ personas[].name + stance → personas-factory.md 的输入
  │   └─ 跳过 Phase 1 中"生成人格列表"的步骤，直接进入"生成完整人格卡"
  │
  ├─ Phase 2-N: 推演
  │   └─ personas[].model → sessions_spawn 的 model 参数
  │   └─ personas[].temperature → 在 task prompt 中指示
  │   └─ dimensions → 每轮 task 中的维度指引
  │   └─ rounds + max_rounds + min_rounds → 控制轮次
  │
  ├─ 收敛检测
  │   └─ convergence_threshold + stale_limit → deliberation-memory.md 逻辑
  │
  └─ Phase 5: 报告
      └─ report_model → ReportAgent 的 sessions_spawn model
      └─ report_temperature → ReportAgent prompt 中的温度指示
```

---

## 执行建议

1. **Phase 0 通常由主 Agent 直接执行**（不需要 spawn 子 Agent）
2. 用 `deepseek/deepseek-v4-flash` 调用配置生成 prompt（低成本，快）
3. 如果生成失败，直接用规则回退表，不要重试
4. 配置保存到 `.openclaw-project/deliberation-config.json` 供后续阶段读取
5. 如果用户已经在消息中指定了角色/轮数等，跳过 Phase 0，用用户指定的参数构建配置
