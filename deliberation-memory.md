# 推演记忆系统 — Deliberation Memory

## 用途

为 Layer 2 的多轮推演提供结构化记忆存储和跨轮注入机制。
让推演从"一次性碰撞"升级为"迭代收敛"。

核心功能：
1. **结构化存储**：每轮推演结果写入 JSONL 日志
2. **收敛检测**：自动判断观点是否已稳定
3. **跨轮注入**：下一轮 Agent 看到上一轮的匿名摘要
4. **归档回顾**：推演完成后保存完整记录

---

## 1. 记忆存储格式（JSONL）

每轮推演结束后，将结构化结果追加到 `.openclaw-project/deliberation-log.jsonl`：

```jsonl
{"round":1,"persona":"焦虑的小店主","model":"deepseek-v4-flash","temperature":0.7,"stance":"opposing","key_points":["租金压力不可承受","远程影响客户体验"],"confidence":0.8,"changed_from":null,"timestamp":"2026-06-06T12:00:00+08:00"}
{"round":1,"persona":"野心勃勃的CTO","model":"glm-5-turbo","temperature":0.4,"stance":"supportive","key_points":["技术已成熟","远程协作工具链完善"],"confidence":0.9,"changed_from":null,"timestamp":"2026-06-06T12:01:00+08:00"}
{"round":2,"persona":"焦虑的小店主","model":"deepseek-v4-flash","temperature":0.7,"stance":"partially-supportive","key_points":["如果混合制可以考虑","远程工具需要培训"],"confidence":0.6,"changed_from":"opposing","timestamp":"2026-06-06T12:05:00+08:00"}
```

### 字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `round` | int | 轮次（从 1 开始） |
| `persona` | string | 角色名 |
| `model` | string | 使用的模型 |
| `temperature` | float | 使用的温度 |
| `stance` | string | 本轮立场（supportive/opposing/neutral/observer/partially-*） |
| `key_points` | string[] | 核心观点列表（2-4 条） |
| `confidence` | float | 信心度 0-1 |
| `changed_from` | string\|null | 上一轮立场（第一轮为 null） |
| `timestamp` | string | ISO 8601 时间戳 |

---

## 2. 收敛检测算法

每轮结束后，主 Agent 计算收敛指标：

```python
changed_count = 0
for persona in all_personas:
    if current_round[persona].stance != previous_round[persona].stance:
        changed_count += 1

stability_rate = 1 - (changed_count / total_personas)
```

### 判断逻辑

```
IF stability_rate >= convergence_threshold AND round >= min_rounds:
    → 收敛达成，进入 ReportAgent 阶段
    → 在日志中标记 "converged"

ELIF stability_rate < prev_stability_rate AND round >= 2:
    → 分歧加大，继续推演（更精彩的碰撞）

ELIF stability_rate == prev_stability_rate AND round >= 2:
    → stale_counter += 1
    → IF stale_counter >= 2: 强制停止（陷入僵局）
    → ELSE: 继续推演

ELSE:
    → 继续推演
```

### 特殊情况处理

| 情况 | 检测条件 | 处理 |
|------|---------|------|
| 快速收敛 | round=1 即达成 | 仍需至少 2 轮（min_rounds=2） |
| 全员一致 | 所有 stance 相同 | 停止 + 标记"⚠️ 缺少反对者" |
| 僵局 | 连续 2 轮 stability_rate 不变 | 强制停止 + 标记"⚠️ 推演僵局" |
| 分歧扩散 | stability_rate 逐轮下降 | 继续（可能正在发现深层分歧） |
| 超过最大轮数 | round > max_rounds | 强制停止 |

---

## 3. 跨轮上下文注入

下一轮 spawn 子 Agent 时，在 task 中注入上一轮的匿名摘要：

### 注入格式

```markdown
## 上一轮推演结果（匿名摘要）

共 {N} 位参与者，立场分布：支持 {n_supportive} / 反对 {n_opposing} / 中立 {n_neutral}

- 参与者A（上一轮立场：支持）：核心观点 1，核心观点 2
- 参与者B（上一轮立场：反对）：核心观点 1，核心观点 2
- 参与者C（上一轮立场：中立）：核心观点 1，核心观点 2

## 本轮任务

你的角色：{persona_name}
你的初始立场：{stance}
你的上轮立场：{changed_from}（{changed_from === null ? "首轮" : "与上轮相同/已变化"}）

请基于上轮推演结果，对你的观点做出修正或坚持。
- 如果修正：说明是什么论据说服了你
- 如果坚持：说明为什么其他观点没有说服力
- 如果有新观点：提出本轮的新洞察

围绕以下维度展开：{dimensions}
```

### 关键规则

1. **匿名**：不暴露其他参与者的具体身份（"参与者A"，不写"焦虑的小店主"）
2. **摘要**：只注入 key_points（2-4 条），不注入完整发言
3. **立场对比**：明确标注该参与者的上轮立场和变化方向
4. **维度聚焦**：提示 Agent 围绕选定维度展开，避免跑题

---

## 4. 主 Agent 如何读写日志

### Phase 2（第一轮）结束后

```
1. 从 N 个子 Agent 的返回中提取 key_points 和 stance
2. 构造 JSONL 记录（changed_from = null）
3. 追加写入 deliberation-log.jsonl
4. 计算 stability_rate = 1.0（首轮无变化参考）
5. 决定是否继续（首轮总是继续，除非 N=1）
```

### Phase 3（第二轮及以后）结束后

```
1. 从子 Agent 返回中提取 key_points 和 stance
2. 对比上轮 stance，填充 changed_from
3. 追加写入 deliberation-log.jsonl
4. 计算 stability_rate
5. 运行收敛检测算法
6. IF 未收敛 → 读取本轮日志，生成匿名摘要，注入下一轮 task
7. IF 已收敛 → 进入 Phase 5（ReportAgent）
```

### Phase 5（ReportAgent）之前

```
将完整 deliberation-log.jsonl 的内容格式化后注入 ReportAgent 的 task：
- 每轮的观点汇总
- 收敛曲线数据
- 关键转折点标记
```

---

## 5. 归档与回顾

推演完成后，主 Agent 执行归档：

### 归档文件

保存到 `memory/swarm-simulations/{YYYY-MM-DD}-{主题}.md`：

```markdown
# 推演归档：{主题}

- 日期：{YYYY-MM-DD}
- 问题类型：{problem_type}
- 参与者：{N}个 × {M}轮
- 模型组合：{model_list}
- 收敛轮次：{converged_at_round}
- 最终 stability_rate：{value}

## 配置

{deliberation-config.json 内容}

## 收敛曲线

| 轮次 | stability_rate | 立场变化数 | 备注 |
|------|---------------|-----------|------|
| 1 | - | - | 首轮 |
| 2 | 0.6 | 3/6 | 大幅分歧 |
| 3 | 0.83 | 1/6 | 趋于收敛 |

## 关键转折

- 第 2 轮：参与者X 从反对转为观望，原因是...
- 第 3 轮：参与者Y 提出新论点...

## 最终报告摘要

{report 核心发现的精简版}
```

### 用途

- 跨推演对比：看看不同问题的推演模式有何不同
- 经验积累：哪些模型组合效果最好？哪些维度最有用？
- 复盘：推演是否真的帮到了决策？

---

## 6. 与 SKILL.md Layer 2 的对接

```
Layer 2 (v4.0):
  │
  ├─ Phase 0: 智能配置 → 读 config-generator.md
  │   └─ 输出 deliberation-config.json
  │
  ├─ Phase 1: 人格工厂 → 读 personas-factory.md
  │
  ├─ Phase 2: 第一轮推演 → 读 parallel-agent-guide.md
  │   └─ 结束后 → [写] deliberation-log.jsonl (round=1)
  │
  ├─ Phase 3: 碰撞推演 → 读 parallel-agent-guide.md
  │   ├─ 开始前 → [读] deliberation-log.jsonl (上轮摘要)
  │   └─ 结束后 → [写] deliberation-log.jsonl (round=N) + 收敛检测
  │   └─ IF 收敛 → 跳到 Phase 5
  │   └─ IF 未收敛 → 继续循环
  │
  ├─ Phase 4: 深度追问（可选）
  │
  └─ Phase 5: 报告生成 → 读 report-agent.md
      └─ 输入包含完整 deliberation-log.jsonl
```
