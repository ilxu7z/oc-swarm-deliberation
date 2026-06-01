# Parallel Agent Guide — sessions_spawn 执行指南

## 用途

指导主 Agent 如何使用 OpenClaw 的 `sessions_spawn` 工具实现 Layer 2 的并行多 Agent 推演。

## 核心原则

1. **每个 Agent 独立 spawn** — context="isolated"，不能互相读到上下文
2. **Agent 之间只看匿名摘要** — 防止预设立场影响判断
3. **至少 2 轮** — 第一轮给独立视角，第二轮才有碰撞涌现
4. **用轻量模型** — 5-8 个 Agent × 2-3 轮，成本可控

## 执行流程

### Phase 1: 准备人格卡

```
主 Agent 调 GLM-5.1 或自己分析 → 生成 N 张人格卡（personas-factory.md）
→ 从 12 维矩阵选 3-4 个维度
→ 输出给用户确认（可选，若用户说"直接开始"则跳过）
```

### Phase 2: 第一轮 — 独立推演

```
对每张人格卡，spawn 一个子 Agent：

FOR EACH persona IN persona_list:
  sessions_spawn(
    task: 构建的子 Agent 任务文本（见下方模板）,
    taskName: "swarm-{角色缩写}-R1",
    context: "isolated",
    model: "deepseek/deepseek-v4-flash"  // 轻量模型
  )

然后 sessions_yield 等待全部完成。
```

**子 Agent 任务模板（第一轮）**：

```markdown
你是【角色名】。你正在参与一场关于以下问题的群体推演。

## 你的人格
立场: [立场]
核心诉求: [核心诉求]
信息水平: [知道什么 / 不知道什么]
情绪基调: [情绪]
隐藏假设: [偏见/盲区 — 注意：作为这个角色，你不会意识到自己的偏见]
利益关系: [与其他人物的关系]

## 推演问题
[问题描述 + 必要背景信息]

## 当前维度: [维度名]
[维度的核心问题]

## 要求
1. 完全从你的立场、知识、情绪出发思考
2. 不要使用你不知道的信息
3. 给出具体的、可验证的观点，不要空话
4. 观点要有推理链，不是凭空断言

## 输出格式
【我的判断】(核心立场 1-2 句)
【推理链】(你的逻辑过程，一步步推导)
【我看到的盲区】(你认为是别人忽略的，基于你的视角)
【我的情绪驱动】(情感因素如何影响你的判断)
```

### Phase 3: 第二轮 — 碰撞交叉

```
1. 收集第一轮所有 Agent 的输出
2. 提取每个 Agent 的核心观点（1-2 句，匿名化）
3. 对每个 Agent spawn 第二轮：

FOR EACH persona IN persona_list:
  sessions_spawn(
    task: 同第一轮模板 + 附加以下内容,
    taskName: "swarm-{角色缩写}-R2",
    context: "isolated"
  )
```

**第二轮附加内容**：

```markdown
## 第一轮其他观点（匿名）
以下是你没有看到的其他参与者第一轮的核心观点：

- 观点A: [摘要]
- 观点B: [摘要]
- 观点C: [摘要]
...

请对这些观点做出回应：
1. 有哪个观点改变了你的看法？为什么？
2. 有哪个观点你强烈不同意？为什么？
3. 你最初的观点有没有需要修正的地方？
4. 有没有哪个角度是所有人都忽略了的？

## 输出格式（追加在标准格式后）
【对其他观点的回应】
- 被说服的点: ...
- 坚持反对的点: ...
- 修正的判断: ...
- 集体盲区: ...
```

### Phase 4: 第三轮（可选）

```
仅当第二轮仍存在 2 个以上核心分歧时启动。
聚焦分歧点，让 Agent 进行最后的立场声明。

taskName: "swarm-{角色缩写}-R3"
附加内容格式：列出当前仍存在的分歧点，让 Agent 给出最终立场和底线。
```

### Phase 5: 收敛合成

```
spawn 一个 ReportAgent:

sessions_spawn(
  task: report-agent.md 的完整 prompt + 所有轮次所有观点 + 人格卡清单,
  taskName: "swarm-report",
  context: "isolated",
  model: "deepseek/deepseek-v4-pro"  // 强模型做收敛
)

sessions_yield → 收到报告 → 主 Agent 提炼输出给用户
```

## 成本估算

| 配置 | 每轮 Token | 每轮成本(估算) |
|------|-----------|-------------|
| DeepSeek V4 Flash × 6 Agent | ~12K per agent = ~72K | ~$0.10 |
| DeepSeek V4 Flash × 6 Agent × 2 轮 | ~144K | ~$0.20 |
| DeepSeek V4 Flash × 8 Agent × 3 轮 | ~288K | ~$0.40 |
| DeepSeek V4 Pro (ReportAgent) | ~20K | ~$0.05 |
| **典型 6 Agent × 2 轮 + Report** | **~164K** | **~$0.25** |

> 以 glm-4.7-flash 替代，成本更低但推理质量略降。

## 并行 vs 串行

```
✅ 并行 (sessions_spawn 一次发全部):
   Phase 2: spawn 6 agents → 一次 yield → 全部完成
   耗时: ~30-60s（取决于最慢的 Agent）

❌ 串行 (一个个 spawn):
   耗时: 6 × 30s = 3min+

结论：必须并行。sessions_spawn 支持并行，不需要一个个发。
```

## 失败处理

| 情况 | 处理 |
|------|------|
| 某个 Agent 超时/失败 | 用它的首轮观点作为输入，不让它参与第二轮 |
| 所有 Agent 高度一致 | 人格设计有问题，增加对立立场重新生成 |
| ReportAgent 质量差 | 换强模型重新跑，或主 Agent 自己收敛 |
| 成本担心 | 缩减到 4 Agent × 2 轮 |
