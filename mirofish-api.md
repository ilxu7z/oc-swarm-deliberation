# MiroFish API 调用文档

## 环境信息

- **MiroFish Docker**: `http://192.168.3.180:3300` (前端)
- **后端 API**: `http://192.168.3.180:5501`
- **容器名**: `mirofish`
- **配置**: `/Users/chee/mirofish/.env`
- **LLM**: 智谱 `glm-4.7-flash` @ `https://open.bigmodel.cn/api/paas/v4`

## 在 OpenClaw 中调用

使用 `exec curl` 调用 MiroFish API：

```bash
curl -s -X POST http://192.168.3.180:5501/api/simulation \
  -H "Content-Type: application/json" \
  -d '{
    "seed_material": "...",
    "prediction_requirement": "...",
    "agent_count": 50,
    "max_rounds": 100
  }'
```

## API 端点（基于 MiroFish 前端交互推测）

> ⚠️ 以下端点为推测，需验证。MiroFish 实际 API 端点需查看源码确认。

### 1. 创建模拟

```bash
POST http://192.168.3.180:5501/api/simulation

Request:
{
  "seed_material": "种子材料文本（事件/新闻/小说/报告）",
  "prediction_requirement": "用自然语言描述推演需求",
  "agent_count": 50,       // 参与模拟的 Agent 数量
  "max_rounds": 100,       // 最大模拟轮次
  "environment_type": "social_network"  // 可选：环境类型
}

Response (推测):
{
  "simulation_id": "sim_abc123",
  "status": "started",
  "estimated_time": "5-15 minutes"
}
```

### 2. 查询模拟状态

```bash
GET http://192.168.3.180:5501/api/simulation/{simulation_id}

Response (推测):
{
  "simulation_id": "sim_abc123",
  "status": "running",     // running / completed / failed
  "current_round": 42,
  "total_rounds": 100,
  "elapsed_time": "3m 12s"
}
```

### 3. 获取预测报告

```bash
GET http://192.168.3.180:5501/api/simulation/{simulation_id}/report

Response (推测):
{
  "report": "完整的预测报告文本...",
  "key_findings": ["发现1", "发现2", ...],
  "recommendations": ["建议1", "建议2", ...]
}
```

### 4. 与模拟世界中的 Agent 对话

```bash
POST http://192.168.3.180:5501/api/simulation/{simulation_id}/chat

Request:
{
  "agent_id": "角色ID或名称",
  "message": "你的问题..."
}

Response (推测):
{
  "agent_id": "...",
  "agent_name": "...",
  "response": "Agent 的回复..."
}
```

### 5. 列出模拟世界中的 Agent

```bash
GET http://192.168.3.180:5501/api/simulation/{simulation_id}/agents

Response (推测):
{
  "agents": [
    {"id": "agent_001", "name": "...", "role": "...", "status": "active"},
    ...
  ]
}
```

## 实际 API 验证方法

如需验证实际 API，可以：

```bash
# 查看 MiroFish 后端源码的 API 路由
docker exec mirofish ls /app/backend/
docker exec mirofish cat /app/backend/app.py  # 或其他主文件

# 或通过浏览器访问 MiroFish 前端，用 DevTools Network 面板
# 捕获实际 API 调用
```

## 容器管理

```bash
# 重启
docker restart mirofish

# 查看日志
docker logs mirofish --tail 50

# 通过 docker-compose 操作
cd /Users/chee/mirofish && docker-compose up -d --force-recreate
```

## 在 Layer 3 中的使用流程

1. 主 Agent 准备种子材料（搜集/分析背景）
2. `exec curl POST /api/simulation` → 获取 `simulation_id`
3. 告知用户："MiroFish 模拟已启动，ID: {id}"
4. 定期 `exec curl GET /api/simulation/{id}` 查询进度
5. 完成后 `exec curl GET /api/simulation/{id}/report` 获取报告
6. 主 Agent 读报告 → 提炼 → 输出策略简报
7. 可选：`exec curl POST /api/simulation/{id}/chat` 追问特定 Agent
