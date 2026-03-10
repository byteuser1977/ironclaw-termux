# IronClaw API设计文档

## 1. API概述

IronClaw提供RESTful API和实时通信接口（SSE/WebSocket），支持Web UI、第三方集成和自动化任务。

### 1.1 API基础信息

**Base URL**: `http://localhost:3000` (默认)

**认证方式**:
- WebSocket: 通过连接参数传递用户ID
- HTTP API: 无状态，依赖通道认证

**数据格式**: JSON

**字符编码**: UTF-8

### 1.2 API版本

当前版本: v1 (无版本前缀)

## 2. 聊天API

### 2.1 发送消息

**端点**: `POST /api/chat/send`

**请求**:
```json
{
  "content": "用户消息内容",
  "thread_id": "可选的会话ID (UUID)",
  "attachments": [
    {
      "type": "image|audio|document",
      "url": "附件URL",
      "metadata": {}
    }
  ]
}
```

**响应**:
```json
{
  "message_id": "uuid",
  "status": "accepted"
}
```

**状态码**:
- `202 ACCEPTED`: 消息已接受
- `429 TOO_MANY_REQUESTS`: 超过速率限制
- `503 SERVICE_UNAVAILABLE`: 通道未启动

### 2.2 聊天历史

**端点**: `GET /api/chat/history`

**查询参数**:
- `thread_id`: 可选，指定会话ID
- `limit`: 返回条数限制 (默认50)
- `offset`: 偏移量

**响应**:
```json
{
  "thread_id": "uuid",
  "messages": [
    {
      "id": "uuid",
      "role": "user|assistant|system",
      "content": "消息内容",
      "created_at": "2024-01-15T10:30:00Z",
      "tool_calls": [
        {
          "id": "call_id",
          "name": "tool_name",
          "arguments": {},
          "result": "工具结果"
        }
      ]
    }
  ]
}
```

### 2.3 会话列表

**端点**: `GET /api/chat/threads`

**响应**:
```json
{
  "threads": [
    {
      "id": "uuid",
      "title": "会话标题",
      "created_at": "2024-01-15T10:30:00Z",
      "last_activity": "2024-01-15T11:00:00Z",
      "message_count": 10
    }
  ]
}
```

### 2.4 创建会话

**端点**: `POST /api/chat/threads`

**请求**:
```json
{
  "title": "新会话标题"
}
```

**响应**:
```json
{
  "thread_id": "uuid",
  "title": "会话标题",
  "created_at": "2024-01-15T10:30:00Z"
}
```

### 2.5 删除会话

**端点**: `DELETE /api/chat/threads/{thread_id}`

**响应**:
```json
{
  "status": "deleted",
  "thread_id": "uuid"
}
```

### 2.6 审批请求

**端点**: `POST /api/chat/approval`

**请求**:
```json
{
  "request_id": "uuid",
  "action": "approve|always|deny",
  "thread_id": "可选的会话ID"
}
```

**响应**:
```json
{
  "message_id": "uuid",
  "status": "accepted"
}
```

### 2.7 认证令牌

**端点**: `POST /api/chat/auth/token`

**请求**:
```json
{
  "extension_name": "扩展名称",
  "token": "认证令牌"
}
```

**响应**:
```json
{
  "success": true,
  "message": "extension_name authenticated (5 tools loaded)"
}
```

### 2.8 取消认证

**端点**: `POST /api/chat/auth/cancel`

**请求**:
```json
{
  "extension_name": "扩展名称"
}
```

**响应**:
```json
{
  "success": true,
  "message": "Auth cancelled"
}
```

## 3. 任务API

### 3.1 任务列表

**端点**: `GET /api/jobs`

**响应**:
```json
{
  "jobs": [
    {
      "id": "uuid",
      "title": "任务标题",
      "state": "pending|in_progress|completed|failed|cancelled|stuck",
      "user_id": "user_id",
      "created_at": "2024-01-15T10:30:00Z",
      "started_at": "2024-01-15T10:31:00Z",
      "completed_at": "2024-01-15T10:35:00Z"
    }
  ]
}
```

### 3.2 任务摘要

**端点**: `GET /api/jobs/summary`

**响应**:
```json
{
  "total": 100,
  "pending": 10,
  "in_progress": 5,
  "completed": 80,
  "failed": 3,
  "stuck": 2
}
```

### 3.3 任务详情

**端点**: `GET /api/jobs/{job_id}`

**响应**:
```json
{
  "id": "uuid",
  "title": "任务标题",
  "description": "任务描述",
  "state": "in_progress",
  "user_id": "user_id",
  "created_at": "2024-01-15T10:30:00Z",
  "started_at": "2024-01-15T10:31:00Z",
  "completed_at": null,
  "elapsed_secs": 120,
  "project_dir": "/workspace/project",
  "browse_url": "/projects/project/",
  "job_mode": "worker|claude_code",
  "transitions": [
    {
      "from": "pending",
      "to": "in_progress",
      "timestamp": "2024-01-15T10:31:00Z",
      "reason": null
    }
  ],
  "actions": [
    {
      "id": "uuid",
      "tool_name": "tool_name",
      "input": {},
      "output": "工具输出",
      "cost": 0.01,
      "duration_ms": 1000,
      "success": true,
      "created_at": "2024-01-15T10:32:00Z"
    }
  ]
}
```

### 3.4 取消任务

**端点**: `POST /api/jobs/{job_id}/cancel`

**响应**:
```json
{
  "status": "cancelled",
  "job_id": "uuid"
}
```

### 3.5 重试任务

**端点**: `POST /api/jobs/{job_id}/retry`

**响应**:
```json
{
  "status": "retrying",
  "job_id": "uuid",
  "new_job_id": "new_uuid"
}
```

## 4. 记忆/工作空间API

### 4.1 文件树

**端点**: `GET /api/memory/tree`

**查询参数**:
- `depth`: 树深度 (可选)

**响应**:
```json
{
  "entries": [
    {
      "path": "context/",
      "is_dir": true
    },
    {
      "path": "README.md",
      "is_dir": false
    }
  ]
}
```

### 4.2 列出目录

**端点**: `GET /api/memory/list`

**查询参数**:
- `path`: 目录路径 (默认根目录)

**响应**:
```json
{
  "path": "context/",
  "entries": [
    {
      "name": "vision.md",
      "path": "context/vision.md",
      "is_dir": false,
      "updated_at": "2024-01-15T10:30:00Z"
    }
  ]
}
```

### 4.3 读取文件

**端点**: `GET /api/memory/read`

**查询参数**:
- `path`: 文件路径 (必需)

**响应**:
```json
{
  "path": "context/vision.md",
  "content": "文件内容",
  "updated_at": "2024-01-15T10:30:00Z"
}
```

### 4.4 写入文件

**端点**: `POST /api/memory/write`

**请求**:
```json
{
  "path": "context/vision.md",
  "content": "文件内容"
}
```

**响应**:
```json
{
  "path": "context/vision.md",
  "status": "written"
}
```

### 4.5 搜索记忆

**端点**: `POST /api/memory/search`

**请求**:
```json
{
  "query": "搜索查询",
  "limit": 10
}
```

**响应**:
```json
{
  "results": [
    {
      "path": "context/vision.md",
      "content": "匹配的内容片段",
      "score": 0.95
    }
  ]
}
```

### 4.6 删除文件

**端点**: `DELETE /api/memory/delete`

**查询参数**:
- `path`: 文件路径 (必需)

**响应**:
```json
{
  "path": "context/vision.md",
  "status": "deleted"
}
```

## 5. 自动化任务API

### 5.1 任务列表

**端点**: `GET /api/routines`

**响应**:
```json
{
  "routines": [
    {
      "id": "uuid",
      "name": "任务名称",
      "description": "任务描述",
      "enabled": true,
      "trigger_type": "cron|event|webhook",
      "last_run_at": "2024-01-15T10:30:00Z",
      "next_fire_at": "2024-01-15T11:00:00Z",
      "run_count": 100,
      "consecutive_failures": 0
    }
  ]
}
```

### 5.2 任务摘要

**端点**: `GET /api/routines/summary`

**响应**:
```json
{
  "total": 10,
  "enabled": 8,
  "disabled": 2,
  "failing": 1,
  "runs_today": 15
}
```

### 5.3 任务详情

**端点**: `GET /api/routines/{routine_id}`

**响应**:
```json
{
  "id": "uuid",
  "name": "任务名称",
  "description": "任务描述",
  "enabled": true,
  "trigger": {
    "type": "cron",
    "expression": "0 */6 * * *",
    "timezone": "UTC"
  },
  "action": {
    "type": "message",
    "content": "执行消息"
  },
  "guardrails": {
    "max_runs_per_day": 24,
    "min_interval_minutes": 60
  },
  "notify": {
    "on_success": false,
    "on_failure": true
  },
  "last_run_at": "2024-01-15T10:30:00Z",
  "next_fire_at": "2024-01-15T16:00:00Z",
  "run_count": 100,
  "consecutive_failures": 0,
  "created_at": "2024-01-01T00:00:00Z",
  "recent_runs": [
    {
      "id": "uuid",
      "trigger_type": "cron",
      "started_at": "2024-01-15T10:30:00Z",
      "completed_at": "2024-01-15T10:31:00Z",
      "status": "completed",
      "result_summary": "成功",
      "tokens_used": 100
    }
  ]
}
```

### 5.4 创建任务

**端点**: `POST /api/routines`

**请求**:
```json
{
  "name": "任务名称",
  "description": "任务描述",
  "enabled": true,
  "trigger": {
    "type": "cron",
    "expression": "0 */6 * * *",
    "timezone": "UTC"
  },
  "action": {
    "type": "message",
    "content": "执行消息"
  },
  "guardrails": {
    "max_runs_per_day": 24,
    "min_interval_minutes": 60
  },
  "notify": {
    "on_success": false,
    "on_failure": true
  }
}
```

**响应**:
```json
{
  "id": "uuid",
  "status": "created"
}
```

### 5.5 更新任务

**端点**: `PUT /api/routines/{routine_id}`

**请求**: 同创建任务

**响应**:
```json
{
  "id": "uuid",
  "status": "updated"
}
```

### 5.6 删除任务

**端点**: `DELETE /api/routines/{routine_id}`

**响应**:
```json
{
  "status": "deleted",
  "routine_id": "uuid"
}
```

### 5.7 触发任务

**端点**: `POST /api/routines/{routine_id}/trigger`

**响应**:
```json
{
  "status": "triggered",
  "routine_id": "uuid",
  "run_id": "run_uuid"
}
```

### 5.8 切换任务状态

**端点**: `POST /api/routines/{routine_id}/toggle`

**请求**:
```json
{
  "enabled": true
}
```

**响应**:
```json
{
  "status": "enabled",
  "routine_id": "uuid"
}
```

## 6. 设置API

### 6.1 获取设置

**端点**: `GET /api/settings`

**响应**:
```json
{
  "settings": {
    "llm_backend": "nearai",
    "llm_model": "claude-3-5-sonnet-20241022",
    "max_iterations": 50,
    "temperature": 0.7,
    "heartbeat_enabled": true,
    "heartbeat_interval": 1800
  }
}
```

### 6.2 更新设置

**端点**: `PUT /api/settings`

**请求**:
```json
{
  "llm_model": "claude-3-5-sonnet-20241022",
  "max_iterations": 50,
  "temperature": 0.7
}
```

**响应**:
```json
{
  "status": "updated"
}
```

### 6.3 重置设置

**端点**: `POST /api/settings/reset`

**响应**:
```json
{
  "status": "reset"
}
```

## 7. 扩展API

### 7.1 扩展列表

**端点**: `GET /api/extensions`

**响应**:
```json
{
  "extensions": [
    {
      "name": "github",
      "version": "1.0.0",
      "type": "tool",
      "description": "GitHub集成",
      "capabilities": ["repositories", "issues", "pull_requests"],
      "auth_required": true,
      "authenticated": false
    }
  ]
}
```

### 7.2 扩展详情

**端点**: `GET /api/extensions/{extension_name}`

**响应**:
```json
{
  "name": "github",
  "version": "1.0.0",
  "type": "tool",
  "description": "GitHub集成",
  "capabilities": ["repositories", "issues", "pull_requests"],
  "auth_required": true,
  "authenticated": true,
  "tools": [
    {
      "name": "github_list_repos",
      "description": "列出仓库"
    }
  ],
  "setup_url": "https://github.com/settings/tokens",
  "auth_url": "https://github.com/login/oauth/authorize"
}
```

### 7.3 安装扩展

**端点**: `POST /api/extensions/install`

**请求**:
```json
{
  "name": "github",
  "version": "1.0.0"
}
```

**响应**:
```json
{
  "status": "installing",
  "extension_name": "github"
}
```

### 7.4 卸载扩展

**端点**: `DELETE /api/extensions/{extension_name}`

**响应**:
```json
{
  "status": "uninstalled",
  "extension_name": "github"
}
```

## 8. 日志API

### 8.1 日志流 (SSE)

**端点**: `GET /api/logs/stream`

**查询参数**:
- `level`: 日志级别 (DEBUG|INFO|WARN|ERROR)
- `since`: 起始时间 (ISO 8601)

**响应**: Server-Sent Events流

```
event: log
data: {"level":"INFO","message":"Starting IronClaw...","timestamp":"2024-01-15T10:30:00Z"}

event: log
data: {"level":"INFO","message":"Loaded configuration","timestamp":"2024-01-15T10:30:01Z"}
```

### 8.2 日志查询

**端点**: `GET /api/logs`

**查询参数**:
- `level`: 日志级别
- `limit`: 返回条数 (默认100)
- `offset`: 偏移量

**响应**:
```json
{
  "logs": [
    {
      "level": "INFO",
      "message": "Starting IronClaw...",
      "timestamp": "2024-01-15T10:30:00Z"
    }
  ]
}
```

## 9. WebSocket API

### 9.1 连接

**端点**: `ws://localhost:3000/ws`

**连接参数**:
- `user_id`: 用户ID (必需)
- `thread_id`: 会话ID (可选)

### 9.2 消息格式

**客户端到服务器**:
```json
{
  "type": "message|command",
  "content": "消息内容",
  "thread_id": "uuid"
}
```

**服务器到客户端**:
```json
{
  "type": "message|event|error",
  "data": {}
}
```

### 9.3 事件类型

**消息事件**:
```json
{
  "type": "message",
  "data": {
    "role": "assistant",
    "content": "回复内容",
    "thread_id": "uuid"
  }
}
```

**工具调用事件**:
```json
{
  "type": "tool_call",
  "data": {
    "tool_name": "tool_name",
    "arguments": {},
    "status": "calling|completed|failed"
  }
}
```

**任务状态事件**:
```json
{
  "type": "job_status",
  "data": {
    "job_id": "uuid",
    "status": "in_progress",
    "message": "任务进行中"
  }
}
```

**错误事件**:
```json
{
  "type": "error",
  "data": {
    "code": "ERROR_CODE",
    "message": "错误消息"
  }
}
```

## 10. SSE事件API

### 10.1 连接

**端点**: `GET /api/events`

**查询参数**:
- `user_id`: 用户ID (必需)
- `thread_id`: 会话ID (可选)

### 10.2 事件类型

**消息事件**:
```
event: message
data: {"role":"assistant","content":"回复内容"}
```

**工具调用事件**:
```
event: tool_call
data: {"tool_name":"tool_name","status":"calling"}
```

**任务更新事件**:
```
event: job_update
data: {"job_id":"uuid","status":"in_progress"}
```

**认证事件**:
```
event: auth_required
data: {"extension_name":"github","instructions":"请访问..."}
```

```
event: auth_completed
data: {"extension_name":"github","success":true}
```

**心跳事件**:
```
event: heartbeat
data: {"timestamp":"2024-01-15T10:30:00Z"}
```

## 11. 错误响应

### 11.1 错误格式

```json
{
  "error": {
    "code": "ERROR_CODE",
    "message": "错误描述",
    "details": {}
  }
}
```

### 11.2 常见错误码

| 错误码 | HTTP状态 | 描述 |
|--------|----------|------|
| `INVALID_REQUEST` | 400 | 请求格式错误 |
| `UNAUTHORIZED` | 401 | 未授权 |
| `FORBIDDEN` | 403 | 禁止访问 |
| `NOT_FOUND` | 404 | 资源未找到 |
| `RATE_LIMITED` | 429 | 超过速率限制 |
| `INTERNAL_ERROR` | 500 | 内部服务器错误 |
| `SERVICE_UNAVAILABLE` | 503 | 服务不可用 |

## 12. 速率限制

### 12.1 限制规则

| 端点类型 | 限制 | 时间窗口 |
|----------|------|----------|
| 聊天消息 | 60 | 1分钟 |
| API请求 | 300 | 1分钟 |
| WebSocket连接 | 5 | 1分钟 |

### 12.2 响应头

```
X-RateLimit-Limit: 60
X-RateLimit-Remaining: 59
X-RateLimit-Reset: 1705318800
```

## 13. 分页

### 13.1 查询参数

- `limit`: 每页条数 (默认20, 最大100)
- `offset`: 偏移量 (默认0)

### 13.2 响应头

```
X-Total-Count: 100
X-Page-Count: 5
X-Current-Page: 1
```

## 14. CORS

支持跨域资源共享，默认允许所有来源（生产环境应配置）。

**响应头**:
```
Access-Control-Allow-Origin: *
Access-Control-Allow-Methods: GET, POST, PUT, DELETE, OPTIONS
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Max-Age: 86400
```

## 15. 安全考虑

### 15.1 输入验证

所有输入都经过验证和净化：
- 长度限制
- 格式检查
- 内容过滤

### 15.2 输出过滤

所有输出都经过安全检查：
- 提示注入检测
- 凭证泄露扫描
- 内容净化

### 15.3 认证

- WebSocket: 通过连接参数验证
- HTTP API: 无状态，依赖通道认证
- 扩展API: OAuth令牌验证

## 16. 最佳实践

### 16.1 客户端实现

1. **使用SSE进行实时更新**: 比轮询更高效
2. **处理重连**: WebSocket和SSE都应实现自动重连
3. **速率限制**: 遵守返回的速率限制头
4. **错误处理**: 实现指数退避重试
5. **分页**: 使用分页参数处理大量数据

### 16.2 性能优化

1. **批量操作**: 尽可能使用批量API
2. **缓存**: 缓存不常变的数据
3. **压缩**: 启用gzip压缩
4. **连接复用**: 使用HTTP/2或WebSocket

### 16.3 安全实践

1. **验证输入**: 始终验证和净化用户输入
2. **使用HTTPS**: 生产环境必须使用HTTPS
3. **令牌保护**: 安全存储认证令牌
4. **最小权限**: 只请求必要的权限
