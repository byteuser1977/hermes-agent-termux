# 命令行界面 (CLI)

Hermes Agent 的 CLI 是一个完整的终端用户界面 (TUI) - 不是 Web 界面。它具备多行编辑、斜杠命令自动完成、对话历史、中断和重定向以及流式工具输出功能。专为生活在终端中的用户构建。

## 运行 CLI

```bash
# 启动交互式会话（默认）
hermes

# 单查询模式（非交互式）
hermes chat -q "你好"

# 使用特定模型
hermes chat --model "anthropic/claude-sonnet-4"

# 使用特定提供商
hermes chat --provider nous        # 使用 Nous Portal
hermes chat --provider openrouter  # 强制使用 OpenRouter

# 使用特定工具集
hermes chat --toolsets "web,terminal,skills"

# 启动时预加载一个或多个技能
hermes -s hermes-agent-dev,github-auth
hermes chat -s github-pr-workflow -q "打开草稿 PR"

# 恢复之前的会话
hermes --continue             # 恢复最近的 CLI 会话 (-c)
hermes --resume <会话ID>      # 通过 ID 恢复特定会话 (-r)

# 详细模式（调试输出）
hermes chat --verbose

# 隔离的 git 工作树（用于并行运行多个代理）
hermes -w                         # 工作树中的交互模式
hermes -w -q "修复问题 #123"     # 工作树中的单查询
```

## 界面布局

欢迎横幅会快速显示您的模型、终端后端、工作目录、可用工具和已安装技能。

### 状态栏

状态栏位于输入区域上方，实时更新：

```
 ⚕ claude-sonnet-4-20250514 │ 12.4K/200K │ [██████░░░░] 6% │ $0.06 │ 15m
```

| 元素 | 描述 |
|------|------|
| 模型名称 | 当前模型（超过 26 个字符时截断） |
| Token 数量 | 已使用的上下文 token / 最大上下文窗口 |
| 上下文条 | 带有颜色编码阈值的可视化填充指示器 |
| 成本 | 估算的会话成本（或 `n/a` 用于未知/零定价模型） |
| 时长 | 已用会话时间 |

状态栏根据终端宽度自适应 - 在 ≥ 76 列时显示完整布局，在 52-75 列时显示紧凑布局，在 52 列以下时显示最小布局（仅模型+时长）。

**上下文颜色编码：**

| 颜色 | 阈值 | 含义 |
|------|------|------|
| 绿色 | < 50% | 空间充足 |
| 黄色 | 50-80% | 正在变满 |
| 橙色 | 80-95% | 接近限制 |
| 红色 | ≥ 95% | 即将溢出 - 考虑 `/compress` |

使用 `/usage` 查看详细的费用分解，包括每类别的成本（输入 vs 输出 token）。

### 会话恢复显示

恢复之前的会话时（`hermes -c` 或 `hermes --resume <id>`），"之前对话"面板会出现在横幅和输入提示之间，显示对话历史的紧凑摘要。有关详细信息和配置，请参阅[会话 - 恢复时的对话摘要](sessions.md#conversation-recap-on-resume)。

## 键绑定

| 键 | 动作 |
|----|------|
| `Enter` | 发送消息 |
| `Alt+Enter` 或 `Ctrl+J` | 新行（多行输入） |
| `Alt+V` | 当终端支持时，从剪贴板粘贴图片 |
| `Ctrl+V` | 粘贴文本并选择性地附加剪贴板图片 |
| `Ctrl+B` | 启用语音模式时开始/停止语音录制（`voice.record_key`，默认：`ctrl+b`） |
| `Ctrl+C` | 中断代理（2 秒内双击强制退出） |
| `Ctrl+D` | 退出 |
| `Ctrl+Z` | 将 Hermes 挂起到后台（仅 Unix）。在 shell 中运行 `fg` 恢复。 |
| `Tab` | 接受自动建议（幽灵文本）或自动完成斜杠命令 |

## 斜杠命令

输入 `/` 查看自动完成下拉菜单。Hermes 支持大量 CLI 斜杠命令、动态技能命令和用户定义的快速命令。

常用示例：

| 命令 | 描述 |
|------|------|
| `/help` | 显示命令帮助 |
| `/model` | 显示或更改当前模型 |
| `/tools` | 列出当前可用工具 |
| `/skills browse` | 浏览技能中心和官方可选技能 |
| `/background <提示>` | 在单独的后台会话中运行提示 |
| `/skin` | 显示或切换活动的 CLI 主题 |
| `/voice on` | 启用 CLI 语音模式（按 `Ctrl+B` 录音） |
| `/voice tts` | 切换 Hermes 回复的语音播放 |
| `/reasoning high` | 增加推理努力 |
| `/title 我的会话` | 命名当前会话 |

有关完整的内置 CLI 和消息列表，请参阅[斜杠命令参考](../reference/slash-commands.md)。

有关设置、提供商、静音调优和消息/Discord 语音使用，请参阅[语音模式](features/voice-mode.md)。

:::tip
命令不区分大小写 - `/HELP` 与 `/help` 效果相同。已安装的技能也会自动成为斜杠命令。
:::

## 快速命令

您可以定义运行 shell 命令的自定义命令，而无需调用 LLM。这些命令在 CLI 和消息平台（Telegram、Discord 等）中均有效。

```yaml
# ~/.hermes/config.yaml
quick_commands:
  status:
    type: exec
    command: systemctl status hermes-agent
  gpu:
    type: exec
    command: nvidia-smi --query-gpu=utilization.gpu,memory.used --format=csv,noheader
```

然后在任何聊天中输入 `/status` 或 `/gpu`。有关更多示例，请参阅[配置指南](/docs/user-guide/configuration#quick-commands)。

## 启动时预加载技能

如果您已经知道会话需要哪些技能，可以在启动时传递它们：

```bash
hermes -s hermes-agent-dev,github-auth
hermes chat -s github-pr-workflow -s github-auth
```

Hermes 会在第一轮之前将每个命名的技能加载到会话提示中。同一标志在交互模式和单查询模式下均有效。

## 技能斜杠命令

`~/.hermes/skills/` 中的每个已安装技能都会自动注册为斜杠命令。技能名称成为命令：

```
/gif-search funny cats
/axolotl help me fine-tune Llama 3 on my dataset
/github-pr-workflow create a PR for the auth refactor

# 仅技能名称会加载它并让代理询问您需要什么：
/excalidraw
```

## 人格

设置预定义的人格以更改代理的语气：

```
/personality pirate
/personality kawaii
/personality concise
```

内置人格包括：`helpful`（有帮助）、`concise`（简洁）、`technical`（技术性）、`creative`（创造性）、`teacher`（教师）、`kawaii`（可爱）、`catgirl`（猫娘）、`pirate`（海盗）、`shakespeare`（莎士比亚）、`surfer`（冲浪者）、`noir`（黑色电影）、`uwu`、`philosopher`（哲学家）、`hype`（热情）。

您还可以在 `~/.hermes/config.yaml` 中定义自定义人格：

```yaml
personalities:
  helpful: "You are a helpful, friendly AI assistant."
  kawaii: "You are a kawaii assistant! Use cute expressions..."
  pirate: "Arrr! Ye be talkin' to Captain Hermes..."
  # 添加您自己的！
```

## 多行输入

有两种方法输入多行消息：

1. **`Alt+Enter` 或 `Ctrl+J`** - 插入新行
2. **反斜杠续行** - 以 `\` 结尾一行以继续：

```
❯ 编写一个函数：\
  1. 接受数字列表\
  2. 返回总和
```

:::info
支持粘贴多行文本 - 使用 `Alt+Enter` 或 `Ctrl+J` 插入换行符，或直接粘贴内容。
:::

## 中断代理

您可以在任何时候中断代理：

- **在代理工作时键入新消息 + Enter** - 它会中断并处理您的新指令
- **`Ctrl+C`** - 中断当前操作（2 秒内双击强制退出）
- 进行中的终端命令会立即被终止（SIGTERM，1 秒后 SIGKILL）
- 中断期间键入的多条消息会合并为一个提示

### 忙碌输入模式

`display.busy_input_mode` 配置键控制在代理工作时按 Enter 时的行为：

| 模式 | 行为 |
|------|------|
| `"interrupt"`（默认） | 您的消息会中断当前操作并立即处理 |
| `"queue"` | 您的消息会静默排队，并在代理完成后作为下一轮发送 |

```yaml
# ~/.hermes/config.yaml
display:
  busy_input_mode: "queue"   # 或 "interrupt"（默认）
```

队列模式在您想准备后续消息而不意外取消进行中的工作时很有用。未知值会回退到 `"interrupt"`。

### 挂起到后台

在 Unix 系统上，按 **`Ctrl+Z`** 将 Hermes 挂起到后台 - 就像任何终端进程一样。shell 会打印确认：

```
Hermes Agent 已挂起。运行 `fg` 恢复 Hermes Agent。
```

在 shell 中输入 `fg` 以恢复会话，恢复到您离开的确切位置。Windows 不支持此功能。

## 工具进度显示

CLI 在代理工作时显示动画反馈：

**思考动画**（API 调用期间）：
```
  ◜ (｡•́︿•̀｡) 思考中... (1.2秒)
  ◠ (⊙_⊙) 沉思中... (2.4秒)
  ✧٩(ˊᗜˋ*)و✧ 搞定！(3.1秒)
```

**工具执行摘要：**
```
  ┊ 💻 terminal `ls -la` (0.3秒)
  ┊ 🔍 web_search (1.2秒)
  ┊ 📄 web_extract (2.1秒)
```

使用 `/verbose` 循环切换显示模式：`关闭 → 新建 → 全部 → 详细`。此命令也可以为消息平台启用 - 请参阅[配置](/docs/user-guide/configuration#display-settings)。

### 工具预览长度

`display.tool_preview_length` 配置键控制工具调用预览行中显示的最大字符数（例如文件路径、终端命令）。默认值为 `0`，表示无限制 - 显示完整路径和命令。

```yaml
# ~/.hermes/config.yaml
display:
  tool_preview_length: 80   # 将工具预览截断为 80 个字符（0 = 无限制）
```

在窄终端或工具参数包含非常长的文件路径时很有用。

## 会话管理

### 恢复会话

退出 CLI 会话时，会打印恢复命令：

```
使用以下命令恢复此会话：
  hermes --resume 20260225_143052_a1b2c3

会话:        20260225_143052_a1b2c3
时长:       12分34秒
消息:       28 (5 个用户，18 个工具调用)
```

恢复选项：

```bash
hermes --continue                          # 恢复最近的 CLI 会话
hermes -c                                  # 简短形式
hermes -c "我的项目"                     # 恢复命名会话（行线中的最新）
hermes --resume 20260225_143052_a1b2c3     # 通过 ID 恢复特定会话
hermes --resume "重构认证"         # 通过标题恢复
hermes -r 20260225_143052_a1b2c3           # 简短形式
```

恢复会从 SQLite 恢复完整的对话历史。代理会看到所有之前的消息、工具调用和响应 - 就像您从未离开过一样。

使用 `/title 我的会话名称` 在聊天中命名当前会话，或在命令行中使用 `hermes sessions rename <id> <title>`。使用 `hermes sessions list` 浏览过去的会话。

### 会话存储

CLI 会话存储在 Hermes 的 SQLite 状态数据库 `~/.hermes/state.db` 下。数据库保留：

- 会话元数据（ID、标题、时间戳、token 计数器）
- 消息历史
- 压缩/恢复会话的行系
- 由 `session_search` 使用的全文搜索索引

某些消息适配器还保留每个平台的转录文件，但 CLI 本身从 SQLite 会话存储恢复。

### 上下文压缩

接近上下文限制时会自动摘要长对话：

```yaml
# 在 ~/.hermes/config.yaml 中
compression:
  enabled: true
  threshold: 0.50    # 默认在上下文限制的 50% 时压缩
  summary_model: "google/gemini-3-flash-preview"  # 用于摘要的模型
```

压缩触发时，中间轮次会被摘要，而前 3 轮和最后 4 轮始终保留。

## 后台会话

在单独的后台会话中运行提示，同时继续使用 CLI 进行其他工作：

```
/background 分析 /var/log 中的日志并总结今天的任何错误
```

Hermes 立即确认任务并给您返回提示：

```
🔄 后台任务 #1 已启动："分析 /var/log 中的日志并总结..."
   任务 ID: bg_143022_a1b2c3
```

### 工作原理

每个 `/background` 提示都会在守护线程中生成一个**完全独立的代理会话**：

- **隔离的对话** - 后台代理不知道您当前会话的历史。它仅接收您提供的提示。