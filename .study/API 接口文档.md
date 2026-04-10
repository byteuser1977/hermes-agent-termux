# API 接口文档

Hermes Agent 提供多种 API 接口，包括内部 Python API、LLM API 适配、消息平台网关和 ACP 协议。

---

## 1. 内部 Python API

### 1.1 AIAgent 类接口

```python
from run_agent import AIAgent

# 初始化
agent = AIAgent(
    model="anthropic/claude-opus-4.6",
    max_iterations=60,
    enabled_toolsets=["core", "web"],
    disabled_toolsets=[],
    quiet_mode=False,
    save_trajectories=False,
    platform="cli",
    session_id=None,
    skip_context_files=False,
    skip_memory=False
)

# 简单对话接口
response = agent.chat("帮我写一个 Python 脚本")
print(response)  # str

# 完整对话接口
result = agent.run_conversation(
    user_message="分析这个项目",
    system_message=None,
    conversation_history=None,
    task_id="task_123"
)
print(result["final_response"])  # str
print(result["messages"])        # list[dict]
```

### 1.2 参数说明

| 参数 | 类型 | 默认值 | 描述 |
|------|------|--------|------|
| `model` | str | `"anthropic/claude-opus-4.6"` | LLM 模型标识符 (格式：`provider:model`) |
| `max_iterations` | int | `60` | 最大 API 调用次数 |
| `enabled_toolsets` | list | `None` | 启用的工具集列表 |
| `disabled_toolsets` | list | `None` | 禁用的工具集列表 |
| `quiet_mode` | bool | `False` | 静默模式（减少输出） |
| `save_trajectories` | bool | `False` | 保存轨迹用于训练 |
| `platform` | str | `None` | 平台标识 (`cli`, `telegram`, `discord`, etc.) |
| `session_id` | str | `None` | 会话 ID (UUID) |
| `skip_context_files` | bool | `False` | 跳过上下文文件注入 |
| `skip_memory` | bool | `False` | 跳过记忆加载 |

### 1.3 返回格式

**`chat()` 方法**：
```python
str  # 最终响应文本
```

**`run_conversation()` 方法**：
```python
{
    "final_response": str,      # 最终响应
    "messages": [               # 完整消息历史
        {
            "role": "system/user/assistant/tool",
            "content": str,
            "tool_calls": [...],  # 仅 assistant
            "tool_call_id": str,  # 仅 tool
            "reasoning": str      # 仅 assistant (可选)
        }
    ]
}
```

---

## 2. LLM API 适配

Hermes Agent 支持多种 LLM 提供商，通过统一的 OpenAI 兼容接口调用。

### 2.1 支持的提供商

| 提供商 | 环境变量 | 基础 URL |
|--------|---------|---------|
| OpenRouter | `OPENROUTER_API_KEY` | `https://openrouter.ai/api/v1` |
| Anthropic | `ANTHROPIC_API_KEY` | `https://api.anthropic.com/v1` |
| OpenAI | `OPENAI_API_KEY` | `https://api.openai.com/v1` |
| z.ai/GLM | `Z_API_KEY` | `https://api.z.ai/api/paas/v4` |
| Kimi/Moonshot | `MOONSHOT_API_KEY` | `https://api.moonshot.cn/v1` |
| MiniMax | `MINIMAX_API_KEY` | `https://api.minimax.chat/v1` |
| Nous Portal | `NOUS_API_KEY` | `https://api.nousresearch.com/v1` |

### 2.2 请求格式 (OpenAI 兼容)

```http
POST /v1/chat/completions
Content-Type: application/json
Authorization: Bearer {API_KEY}

{
    "model": "claude-opus-4.6",
    "messages": [
        {
            "role": "system",
            "content": "You are Hermes Agent..."
        },
        {
            "role": "user",
            "content": "帮我写一个 Python 脚本"
        }
    ],
    "tools": [
        {
            "type": "function",
            "function": {
                "name": "read_file",
                "description": "读取文件内容",
                "parameters": {
                    "type": "object",
                    "properties": {
                        "path": {
                            "type": "string",
                            "description": "文件路径"
                        }
                    },
                    "required": ["path"]
                }
            }
        }
    ],
    "tool_choice": "auto",
    "max_tokens": 4096,
    "temperature": 0.7
}
```

### 2.3 响应格式

```json
{
    "id": "chatcmpl-abc123",
    "object": "chat.completion",
    "created": 1234567890,
    "model": "claude-opus-4.6",
    "choices": [
        {
            "index": 0,
            "message": {
                "role": "assistant",
                "content": "我来帮你...",
                "tool_calls": [
                    {
                        "id": "call_abc123",
                        "type": "function",
                        "function": {
                            "name": "read_file",
                            "arguments": "{\"path\": \"./main.py\"}"
                        }
                    }
                ]
            },
            "finish_reason": "tool_calls"
        }
    ],
    "usage": {
        "prompt_tokens": 100,
        "completion_tokens": 50,
        "total_tokens": 150
    }
}
```

---

## 3. 工具调用 API

### 3.1 工具 Schema 格式

每个工具使用 OpenAI function calling schema：

```json
{
    "type": "function",
    "function": {
        "name": "read_file",
        "description": "读取文件内容并返回",
        "parameters": {
            "type": "object",
            "properties": {
                "path": {
                    "type": "string",
                    "description": "要读取的文件路径"
                },
                "limit": {
                    "type": "integer",
                    "description": "最大读取行数",
                    "default": 2000
                }
            },
            "required": ["path"]
        }
    }
}
```

### 3.2 工具调用流程

```
1. Agent 发送请求（包含 tools 参数）
   ↓
2. LLM 返回响应（包含 tool_calls）
   ↓
3. Agent 执行工具
   ↓
4. Agent 发送 tool_result
   ↓
5. LLM 继续生成或返回最终响应
```

### 3.3 工具结果格式

```json
{
    "role": "tool",
    "tool_call_id": "call_abc123",
    "content": "{\"success\": true, \"data\": \"文件内容...\"}"
}
```

---

## 4. 消息平台网关 API

### 4.1 网关命令接口

所有平台共享同一套斜杠命令系统：

| 命令 | 参数 | 描述 | 示例 |
|------|------|------|------|
| `/new` | 无 | 开始新对话 | `/new` |
| `/reset` | 无 | 重置当前会话 | `/reset` |
| `/model` | `<provider:model>` | 切换模型 | `/model openai/gpt-4` |
| `/tools` | 无 | 配置工具 | `/tools` |
| `/skills` | 无 | 浏览技能 | `/skills` |
| `/compress` | 无 | 压缩上下文 | `/compress` |
| `/usage` | `[--days N]` | 查看用量 | `/usage --days 7` |
| `/insights` | `[days]` | 记忆洞察 | `/insights 30` |
| `/help` | 无 | 显示帮助 | `/help` |
| `/stop` | 无 | 停止当前任务 | `/stop` |

### 4.2 平台特定 API

#### Telegram Bot API

```python
# 发送消息
POST https://api.telegram.org/bot{TOKEN}/sendMessage
{
    "chat_id": 123456,
    "text": "/model anthropic/claude-3"
}

# 接收更新
GET https://api.telegram.org/bot{TOKEN}/getUpdates
```

#### Discord API

```python
# 发送消息
POST https://discord.com/api/v10/channels/{CHANNEL_ID}/messages
Authorization: Bot {TOKEN}
{
    "content": "/model anthropic/claude-3"
}

# 接收事件 (WebSocket)
wss://gateway.discord.gg/?v=10&encoding=json
```

#### Slack API

```python
# 发送消息
POST https://slack.com/api/chat.postMessage
Authorization: Bearer {TOKEN}
{
    "channel": "C123456",
    "text": "/model anthropic/claude-3"
}

# 接收事件 (HTTP)
POST /slack/events
{
    "type": "event_callback",
    "event": {
        "type": "message",
        "text": "/model anthropic/claude-3"
    }
}
```

---

## 5. ACP (Agent Communication Protocol)

ACP 是用于 IDE 集成的协议，支持 VS Code、Zed 和 JetBrains。

### 5.1 ACP 消息格式

**初始化请求**：
```json
{
    "type": "initialize",
    "version": "1.0",
    "capabilities": {
        "chat": true,
        "tool_call": true
    }
}
```

**初始化响应**：
```json
{
    "type": "initialize",
    "version": "1.0",
    "session_id": "abc123"
}
```

**聊天请求**：
```json
{
    "type": "chat",
    "session_id": "abc123",
    "message": "解释这段代码",
    "context": {
        "file": "main.py",
        "selection": {
            "start": 10,
            "end": 20
        }
    }
}
```

**聊天响应**：
```json
{
    "type": "chat",
    "session_id": "abc123",
    "response": "这段代码实现了...",
    "tool_calls": [
        {
            "name": "read_file",
            "arguments": {"path": "main.py"}
        }
    ]
}
```

**工具调用请求**：
```json
{
    "type": "tool_call",
    "session_id": "abc123",
    "tool_name": "read_file",
    "arguments": {"path": "main.py"}
}
```

**工具调用响应**：
```json
{
    "type": "tool_call",
    "session_id": "abc123",
    "result": "{\"success\": true, \"data\": \"...\"}"
}
```

### 5.2 ACP 传输层

ACP 使用 JSON-RPC over WebSocket 或 stdio：

**WebSocket**：
```
ws://localhost:8765/acp
```

**Stdio** (用于 IDE 插件)：
```
Content-Length: 123\r\n
\r\n
{"type": "chat", ...}
```

---

## 6. MCP (Model Context Protocol)

Hermes Agent 支持作为 MCP 客户端连接外部服务器。

### 6.1 MCP 配置格式

```yaml
# ~/.hermes/config.yaml
mcp:
  servers:
    filesystem:
      command: npx
      args: ["-y", "@modelcontextprotocol/server-filesystem", "~/"]
    github:
      command: npx
      args: ["-y", "@modelcontextprotocol/server-github"]
    postgres:
      command: npx
      args: ["-y", "@modelcontextprotocol/server-postgres", "postgresql://localhost"]
```

### 6.2 MCP 工具调用

```python
def mcp_call(server: str, tool: str, arguments: dict) -> str:
    """
    调用 MCP 服务器工具
    
    参数:
        server: MCP 服务器名称
        tool: 工具名称
        arguments: 工具参数
    
    返回:
        JSON 格式结果
    """
```

**示例**：
```json
{
    "server": "filesystem",
    "tool": "read_file",
    "arguments": {"path": "/home/user/file.txt"}
}
```

---

## 7. 定时任务 API

### 7.1 添加定时任务

```python
from cron.scheduler import CronScheduler

scheduler = CronScheduler()

# 添加任务
scheduler.add_job(
    schedule="0 9 * * *",           # 每天 9:00
    command="生成今日工作报告",      # 要执行的命令
    platform="telegram",            # 交付平台
    chat_id="123456"                # 目标聊天 ID
)

# 运行调度器
await scheduler.run()
```

### 7.2 Cron 表达式

```
* * * * *
│ │ │ │ │
│ │ │ │ └─ 星期 (0-6, 0=周日)
│ │ │ └─── 月份 (1-12)
│ │ └───── 日期 (1-31)
│ └─────── 小时 (0-23)
└───────── 分钟 (0-59)
```

**特殊语法**：
- `*` - 任意值
- `,` - 分隔多个值：`0 9,17 * * *` (9:00 和 17:00)
- `-` - 范围：`0 9-17 * * *` (9:00 到 17:00 每小时)
- `/` - 步长：`*/5 * * * *` (每 5 分钟)

---

## 8. 会话管理 API

### 8.1 SessionDB 接口

```python
from hermes_state import SessionDB

db = SessionDB()

# 创建会话
session_id = db.create_session(platform="cli")

# 添加消息
db.add_message(
    session_id=session_id,
    role="user",
    content="你好"
)

# 加载历史
messages = db.load_session_history(session_id, limit=50)

# 搜索
results = db.search(query="Python", session_id=None)

# 删除会话
db.delete_session(session_id)

# 列出会话
sessions = db.list_sessions(platform="cli")
```

### 8.2 搜索语法 (FTS5)

```sql
-- 精确匹配
SELECT * FROM messages_fts WHERE content MATCH '"exact phrase"';

-- 前缀匹配
SELECT * FROM messages_fts WHERE content MATCH 'pyth*';

-- 逻辑运算符
SELECT * FROM messages_fts WHERE content MATCH 'Python AND Django';
SELECT * FROM messages_fts WHERE content MATCH 'Python OR Django';
SELECT * FROM messages_fts WHERE content MATCH 'Python NOT Django';

-- 字段限定
SELECT * FROM messages_fts WHERE content MATCH 'role:user AND content:Python';
```

---

## 9. 用量统计 API

### 9.1 查询用量

```python
from hermes_state import SessionDB

db = SessionDB()

# 今日用量
today = db.get_usage_stats(date="today")
print(f"Tokens: {today['total_tokens']}, Cost: ${today['total_cost_usd']}")

# 按模型统计
stats = db.get_usage_by_model(days=7)
for model, data in stats.items():
    print(f"{model}: {data['tokens']} tokens, ${data['cost']}")

# 趋势
trend = db.get_usage_trend(days=30)
for date, tokens in trend.items():
    print(f"{date}: {tokens} tokens")
```

### 9.2 返回格式

```json
{
    "date": "2024-01-15",
    "model": "anthropic/claude-opus-4.6",
    "platform": "cli",
    "total_tokens": 10000,
    "input_tokens": 6000,
    "output_tokens": 4000,
    "api_calls": 50,
    "tool_calls": 30,
    "total_cost_usd": 0.15
}
```

---

## 10. 错误处理

### 10.1 错误格式

```json
{
    "error": {
        "type": "invalid_request_error",
        "message": "Invalid model name",
        "code": "invalid_model"
    }
}
```

### 10.2 错误类型

| 错误类型 | HTTP 状态码 | 描述 |
|----------|-----------|------|
| `invalid_request_error` | 400 | 请求参数错误 |
| `authentication_error` | 401 | API 密钥无效 |
| `permission_error` | 403 | 权限不足 |
| `not_found_error` | 404 | 资源不存在 |
| `rate_limit_error` | 429 | 超出速率限制 |
| `internal_error` | 500 | 服务器内部错误 |

### 10.3 重试逻辑

```python
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=4, max=10)
)
def call_llm_api(messages, tools):
    response = client.chat.completions.create(...)
    return response
```

---

## 11. 速率限制

### 11.1 提供商限制

| 提供商 | 限制 |
|--------|------|
| OpenRouter | 取决于具体模型 |
| Anthropic | 50-500 requests/min |
| OpenAI | 60 requests/min (GPT-4) |
| z.ai | 100 requests/min |

### 11.2 本地限制

```python
# ~/.hermes/config.yaml
rate_limit:
  requests_per_minute: 30
  tokens_per_minute: 100000
  burst_size: 10
```

---

## 12. 认证与授权

### 12.1 API 密钥管理

```bash
# 查看密钥
hermes auth list

# 添加密钥
hermes auth set openrouter sk-...

# 删除密钥
hermes auth delete openrouter
```

### 12.2 网关用户认证

```yaml
# ~/.hermes/config.yaml
gateway:
  allowed_users:
    telegram:
      - 123456  # User ID
    discord:
      - 789012  # User ID
  pairing:
    enabled: true
    code_length: 6
    expiry_seconds: 300
```

---

## 13. Webhook API

### 13.1 配置 Webhook

```yaml
# ~/.hermes/config.yaml
webhook:
  enabled: true
  url: https://example.com/hermes-webhook
  events:
    - conversation_start
    - conversation_end
    - tool_execution
    - error
```

### 13.2 Webhook 负载

```json
{
    "event": "conversation_end",
    "timestamp": 1234567890,
    "session_id": "abc123",
    "platform": "telegram",
    "data": {
        "user_message": "帮我写代码",
        "agent_response": "好的...",
        "tokens_used": 1000,
        "duration_seconds": 30
    }
}
```

---

## 14. 健康检查

### 14.1 状态查询

```bash
hermes status
```

**输出**：
```
Hermes Agent Status
-------------------
Version: 0.8.0
Model: anthropic/claude-opus-4.6
Platform: cli
Session: abc123
API Keys: ✓ OpenRouter, ✓ Anthropic
Tools: 40 enabled
Memory: enabled
Cron: running
```

### 14.2 Doctor 诊断

```bash
hermes doctor
```

**检查项**：
- Python 版本
- 依赖包
- API 密钥
- 数据库完整性
- 工具可用性
- 权限检查

---

## 15. 最佳实践

### 15.1 错误处理

```python
try:
    response = agent.chat(message)
except RateLimitError:
    print("速率限制，等待后重试")
except AuthenticationError:
    print("API 密钥无效")
except Exception as e:
    print(f"未知错误：{e}")
```

### 15.2 资源清理

```python
# 使用上下文管理器
with AIAgent(...) as agent:
    response = agent.chat("Hello")
# 自动清理资源
```

### 15.3 日志记录

```python
import logging

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)

logger = logging.getLogger('hermes')
logger.info("Agent started")
```
