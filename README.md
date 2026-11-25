# Kazecode

一个终端里的 AI 编程助手 —— 用自然语言驱动它在你的代码库中读、写、改、跑。

基于 [Textual](https://textual.textualize.io/) 构建 TUI，同时兼容 Anthropic 与 OpenAI 两套协议，
支持 MCP 外部工具、生命周期 Hooks、Skills、子代理与多 Agent 协作。

```
┌─ kazecode ─────────────────────────────────────────────────┐
│ > 帮我看看 user 模块的鉴权逻辑有没有漏洞                    │
│                                                          │
│  ⏺ ReadFile(app/api/user.py)                             │
│  ⏺ Grep("verify_token|@requires_auth")  → 12 处命中      │
│  ⏺ EditFile(app/api/user.py)  ✓ 已应用                   │
└──────────────────────────────────────────────────────────┘
```

## 特性

| 能力 | 说明 |
| --- | --- |
| **TUI 交互** | Textual 驱动的终端界面，支持流式输出、实时工具调用状态、计划模式 |
| **双协议** | 内置 Anthropic / OpenAI 协议适配，任意兼容端点（`base_url` 可配） |
| **13 个内置工具** | `ReadFile` `WriteFile` `EditFile` `Glob` `Grep` `Bash` `Task` `ToolSearch` `Skill` `Agent` `WebFetch` `WebSearch` `TodoWrite` |
| **权限体系** | 4 种权限模式 × read / write / command 三类工具的独立裁决，危险命令识别与沙箱 |
| **Hooks** | 15 个生命周期事件，可在工具执行前后拦截、改写或拒绝 |
| **MCP** | 支持 stdio 与 Streamable HTTP / SSE 两种传输，直接接入外部 MCP 服务器 |
| **Skills** | 可安装的技能包，内置 `commit` `review` `test` `backend-interview` |
| **子代理** | 内置 `explore` `general-purpose` `plan` `verification`，支持派生与并行 |
| **多 Agent 协作** | Team 模式：Lead 负责调度，成员并行执行子任务 |
| **上下文管理** | 分层压缩 + 自动压缩、token 计数、文件历史与回退（rewind） |
| **远程模式** | WebSocket 服务 + 浏览器 UI，可在手机等设备上驱动同一个会话 |
| **会话持久化** | 会话与记忆落盘到 `.kazecode/`，可恢复、可续接 |

## 快速开始

需要 Python 3.11+。

```bash
# 安装
uv sync                      # 推荐（使用项目内的 uv.lock）
# 或
pip install -e .

# 初始化配置
mkdir -p .kazecode
cp .kazecode/config.yaml.example .kazecode/config.yaml
# 编辑 config.yaml 填入你的 API Key

# 运行
kazecode                      # 交互式 TUI
kazecode -p "统计各模块的代码行数"   # 非交互模式，直接输出结果
kazecode --remote             # 远程模式，浏览器访问 http://localhost:18888
```

### 配置

配置文件按以下顺序加载，后者覆盖前者：

1. `~/.kazecode/config.yaml` —— 用户级
2. `.kazecode/config.yaml` —— 项目级
3. `.kazecode/config.local.yaml` —— 项目级本地覆盖（不入库）

```yaml
providers:
  - name: anthropic-official
    protocol: anthropic
    base_url: https://api.anthropic.com
    api_key: "your-api-key-here"
    model: claude-sonnet-4-20250514
    thinking: true

permission_mode: default

mcp_servers:
  - name: context7
    command: npx
    args: ["-y", "@upstash/context7-mcp"]

enable_coordinator_mode: false   # Team 模式下 Lead 只调度、不写代码
```

## 权限模式

| 模式 | 读文件 | 写文件 | 执行命令 |
| --- | --- | --- | --- |
| `default` | 允许 | 询问 | 询问 |
| `acceptEdits` | 允许 | 允许 | 询问 |
| `plan` | 允许 | 询问 | 询问 |
| `bypassPermissions` | 允许 | 允许 | 允许 |

除模式之外，命令还会经过危险操作识别（如 `rm -rf`）与沙箱检查。
运行期可用 `/permission` 临时切换。

## 内置命令

| 命令 | 作用 | 命令 | 作用 |
| --- | --- | --- | --- |
| `/help` | 查看全部命令 | `/clear` | 清空对话 |
| `/status` | 查看状态与 token 占用 | `/compact` | 手动压缩上下文 |
| `/plan` | 进入/退出计划模式 | `/review` | 发起代码审查 |
| `/memory` | 管理长期记忆 | `/session` | 会话管理 |
| `/tasks` | 查看子任务 | `/trace` | 调用链路追踪 |
| `/skill` | 管理 Skills | `/mcp` | 管理 MCP 服务器 |
| `/permission` | 权限模式设置 | `/sandbox` | 沙箱设置 |
| `/worktree` | Git worktree 管理 | `/rewind` | 回退文件改动 |

## Hooks

Hooks 写在 `config.yaml` 的 `hooks:` 段，用于在生命周期节点拦截或改写行为。

支持的事件：`session_start`、`session_end`、`turn_start`、`turn_end`、`pre_tool_use`、
`post_tool_use`、`pre_send`、`post_receive`、`startup`、`shutdown`、`error`、
`compact`、`permission_request`、`file_change`、`command_execute`。

其中 `pre_tool_use` 可以在工具真正执行前拦截 —— 一个用于阻止 `rm -rf /` 的 Hook
应该是真的拦住，而不是形同虚设。

## 目录约定

```
.kazecode/
├── config.yaml          # 配置（不入库）
├── config.local.yaml    # 本地覆盖（不入库）
├── agents/              # 项目级子代理定义
├── skills/              # 项目级 Skills
├── memory/              # 长期记忆
├── sessions/            # 会话历史
└── file-history/        # 文件修改历史（支持 rewind）
```

用户级目录 `~/.kazecode/` 下同样支持 `agents/` 与 `skills/`。

## 项目结构

```
kazecode/
├── agent.py        # 核心 Agent 循环：工具调度、并行执行、错误恢复
├── app.py          # Textual TUI 应用
├── client.py       # Anthropic / OpenAI 协议客户端与流式解析
├── prompts.py      # 系统提示词与各代理/模式的提示模板
├── commands/       # 斜杠命令（16 个内置命令）
├── hooks/          # Hook 引擎：事件、条件匹配、执行器
├── permissions/    # 权限模式、规则、危险命令识别、沙箱
├── agents/         # 子代理加载、派生、任务管理、通知
├── skills/         # Skill 解析、安装与执行
├── mcp/            # MCP 客户端、服务器管理、工具包装
├── memory/         # 长期记忆
├── context/        # 上下文窗口管理
├── filehistory/    # 文件历史与回退
├── remote.py       # WebSocket + 浏览器 UI
└── serialization.py # 会话序列化
```

## 测试

```bash
uv run pytest              # 572 个测试
uv run pytest tests/test_hooks.py -v
```

## License

MIT
