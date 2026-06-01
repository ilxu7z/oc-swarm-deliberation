# Swarm Thinking v3.0 — 群体智能推演引擎

> 从 [MiroFish](https://github.com/666ghj/MiroFish) 多智能体社会模拟引擎 + [OASIS](https://github.com/camel-ai/oasis) 提炼的思维方法论

## 核心思想

当面对复杂决策/创意问题时，人的思维容易陷入单一视角的盲区。此方法通过**构建多个人格化的利益相关方，让他们在模拟推演中相互碰撞、辩论、博弈，最终从涌现的群体行为中找到最优解。**

**v3.0 核心升级**：从「单体 LLM 角色扮演」进化到「真正多次元的群体智能」——用 OpenClaw 的 `sessions_spawn` 启动独立子 Agent，或用 MiroFish Docker 引擎跑完整的社会模拟。

## 三层架构

| Layer | 模式 | 方式 | 适用 |
|-------|------|------|------|
| **Layer 1** | 轻量 | 单体 LLM 角色扮演（6 步法） | ≤4 角色、快速决策 |
| **Layer 2** | 标准 | sessions_spawn 并行 Agent | 5-8 角色、中等复杂度 |
| **Layer 3** | 深度 | MiroFish Docker 引擎 | 8+ 角色、长期推演 |

## 适用场景

- 🧠 头脑风暴 — 创意发散、产品方向评估
- 📊 决策分析 — 方案对比、风险评估
- 🔮 未来推演 — 策略沙盘、舆情预测、情景模拟
- 🎮 创意沙盘 — 小说结局推演、脑洞探索

## 快速开始

将此 Skill 安装到 OpenClaw 使用：

```bash
git clone https://github.com/ilxu7z/swarm-thinking.git ~/.openclaw/skills/swarm-thinking
```

激活：说「头脑风暴」「帮我分析」「推演一下」「模拟一下」等关键词即可自动触发。

## 文件结构

| 文件 | 内容 |
|------|------|
| `SKILL.md` | 主技能 — 三层架构 + 12维矩阵 + 执行协议 |
| `personas-factory.md` | 人格工厂 prompt — 自动生成有张力的人格卡 |
| `dimensions-matrix.md` | 12 维推演矩阵 — 按问题类型智能匹配 |
| `parallel-agent-guide.md` | sessions_spawn 并行 Agent 执行指南 |
| `report-agent.md` | ReportAgent 收敛分析 prompt |
| `mirofish-api.md` | MiroFish Docker API 调用文档 |

## 来源

- MiroFish: https://github.com/666ghj/MiroFish — 盛大集团孵化的多智能体社会模拟引擎
- OASIS: https://github.com/camel-ai/oasis — CAMEL-AI 的百万级 Agent 社交模拟
