# LingCode

LingCode 是一个纯 Java 实现的终端 AI 编程助手（Coding Agent）。它提供交互式 TUI、非交互式命令行、Remote Web 控制三种运行模式，内置文件读写/编辑、命令执行、代码搜索等工具，并支持 MCP 扩展、Skills 技能系统、Hooks 事件钩子、多 Agent 团队协作、会话持久化与自动记忆等能力。

## 特性

- **三种运行模式**
  - TUI 交互模式（默认）：基于 JLine + 自研 Tea 架构的终端界面，支持流式输出、Markdown 渲染、权限审批对话框、Plan 模式审批
  - Print 模式（`-p`）：非交互式执行，结果输出到 stdout，支持 `stream-json` 格式
  - Remote 模式（`--remote`）：启动 HTTP + WebSocket 服务器（Javalin），通过浏览器远程控制 Agent
- **多模型支持**：Anthropic 原生协议、OpenAI 官方协议、OpenAI 兼容协议（可对接 DeepSeek 等第三方服务），支持 thinking 模式与上下文窗口自动探测
- **内置工具**：`ReadFile` / `WriteFile` / `EditFile` / `Bash` / `Glob` / `Grep` / `AskUser` / `ToolSearch` / `ExitPlanMode` / 任务清单（`TaskCreate` / `TaskGet` / `TaskList` / `TaskUpdate`）/ Worktree 进出 / 子 Agent（`Agent`）/ 技能加载（`LoadSkill`）等
- **MCP 扩展**：支持 stdio 与 Streamable HTTP / SSE 两种传输方式接入外部 MCP Server
- **Skills 技能系统**：从 `.lingcode/skills/` 加载 SKILL.md 技能，支持安装、热重载、内联/子进程执行
- **Hooks 事件钩子**：在 `session_start` / `turn_start` / `pre_tool_use` / `post_tool_use` 等 9 类事件上挂载 command / prompt / http / agent 动作，可拦截或改写工具调用
- **权限系统**：`default` / `acceptEdits` / `plan` / `bypass` 四种权限模式，写文件与执行命令默认需终端审批
- **OS 级沙箱**：macOS（seatbelt）/ Linux（bubblewrap）下对 Bash 命令做文件系统与网络隔离
- **会话与记忆**：会话以 JSONL 持久化，支持 `/resume` 恢复、`/rewind` 回退、上下文自动压缩；自动记忆分用户级与项目级存储
- **多 Agent 协作**：子 Agent（Agent 工具）派生任务；Team 模式通过 tmux / iTerm 拉起多个队友进程，共享邮箱与任务板；可选 Coordinator 模式（Lead 只调度不写码）
- **Git Worktree 隔离**：Agent 可进入独立 worktree 工作，避免污染主工作区

## 环境要求

- JDK 21+
- Gradle（项目自带 wrapper）

## 构建

```bash
# Windows
gradlew.bat shadowJar

# Linux / macOS
./gradlew shadowJar
```

产物为 `build/libs/lingcode.jar`（shaded 可执行 fat-jar）。

## 快速开始

1. 准备配置文件。复制示例并按需修改：

   ```bash
   copy .lingcode\config.yaml.example .lingcode\config.yaml
   ```

2. 填入你的 API Key 后启动：

   ```bash
   java -jar build\libs\lingcode.jar
   ```

### 命令行参数

| 参数 | 说明 |
| --- | --- |
| `-p "prompt"` | Print 模式：非交互执行单条指令 |
| `--output-format text\|stream-json` | Print 模式的输出格式 |
| `--remote[=addr]` | Remote 模式，默认监听 `:18888` |
| `<config-path>` | 指定配置文件路径（位置参数） |

也可通过环境变量 `LINGCODE_CONFIG` 指定配置文件。

## 配置说明

配置文件按以下顺序查找并逐层合并（后者覆盖前者）：

1. `~/.lingcode/config.yaml`（用户全局）
2. `<项目>/.lingcode/config.yaml`
3. `<项目>/.lingcode/config.local.yaml`（本地覆盖，建议加入 .gitignore）

完整示例：

```yaml
providers:
  - name: anthropic-official
    protocol: anthropic            # anthropic | openai | openai-compat
    base_url: https://api.anthropic.com
    api_key: "your-api-key-here"   # 也可用环境变量 ANTHROPIC_API_KEY / OPENAI_API_KEY
    model: claude-sonnet-4-20250514
    thinking: true
    # context_window: 200000       # 可选，不设则自动从服务端探测
    # max_output_tokens: 8192

permission_mode: default           # default | acceptEdits | plan | bypass

mcp_servers:
  - name: context7
    command: npx                   # stdio 传输
    args: ["-y", "@upstash/context7-mcp"]
  # - name: my-http-mcp            # HTTP 传输
  #   url: https://example.com/mcp
  #   transport: http              # http（默认）| sse
  #   headers:
  #     Authorization: "Bearer ${MY_TOKEN}"

hooks:
  - id: my-hook
    event: pre_tool_use            # session_start / session_end / turn_start / turn_end
                                   # / pre_send / post_receive / pre_tool_use / post_tool_use / shutdown
    type: command                  # command | prompt | http | agent
    command: "echo hello"
    # condition / reject / once / async / on_error / timeout 等可选

sandbox:
  enabled: false                   # 开启后 Bash 命令在 OS 沙箱内执行（仅 macOS / Linux）
  auto_allow: true                 # 沙箱内命令是否免审批
  network_enabled: false           # 沙箱是否放行网络

enable_coordinator_mode: false     # true 时 Team 的 Lead 只能调度不能写代码
# enable_fork: true                # 省略 subagent_type 时是否走 fork，默认开启
```

## TUI 斜杠命令

| 命令 | 说明 |
| --- | --- |
| `/help` | 查看命令列表 |
| `/status` | 查看当前模型、会话、用量等状态 |
| `/clear` | 清空当前对话 |
| `/compact` | 手动压缩上下文 |
| `/plan` | 切换 Plan 模式（只读规划，执行前需审批） |
| `/session` | 会话管理 |
| `/resume` | 恢复历史会话 |
| `/rewind` | 回退到之前的检查点 |
| `/permission` | 权限管理 |
| `/memory` | 管理自动记忆 |
| `/skills` | 查看技能（`/skills reload` 热重载） |
| `/review` | 审查当前代码改动 |
| `/sandbox` | 管理 Bash 沙箱 |
| `/mcp` | 查看 MCP Server 状态 |

自定义命令：在 `~/.lingcode/commands/` 或 `<项目>/.lingcode/commands/` 下放置 `.md` 文件即可注册为 `/命令名`（子目录以 `:` 连接，如 `git/log.md` → `/git:log`），支持 YAML frontmatter 定义 `description` / `argument-hint` / `aliases`。

## 目录结构

```
.lingcode/
├── config.yaml          # 项目级配置
├── config.local.yaml    # 本地覆盖配置（可选）
├── commands/            # 自定义斜杠命令（.md）
├── skills/              # 技能（每个技能一个目录，含 SKILL.md）
├── sessions/            # 会话持久化（JSONL，自动生成）
└── memory/              # 项目级自动记忆（自动生成）
```

## 源码结构

```
src/main/java/com/lingcode/
├── LingCode.java        # 入口：TUI / Print / Remote / Teammate 四种入口分发
├── agent/               # Agent 主循环与流式执行器
├── llm/                 # Anthropic / OpenAI / OpenAI 兼容客户端与流事件
├── tool/                # 工具注册表与内置工具实现
├── tui/                 # TUI 界面（tea 架构、对话框、Markdown 渲染）
├── command/             # 斜杠命令注册与加载
├── config/              # YAML 配置解析（providers / hooks / sandbox / mcp）
├── permission/          # 权限模式与审批检查
├── hook/                # 事件钩子引擎
├── mcp/                 # MCP Server 连接与工具桥接
├── skill/               # 技能目录、安装与执行
├── session/ compact/    # 会话持久化与上下文压缩
├── memory/              # 自动记忆管理与整合
├── subagent/ teams/     # 子 Agent 派生与多 Agent 团队协作
├── worktree/            # Git worktree 管理
├── sandbox/             # seatbelt / bubblewrap 沙箱
├── remote/              # Remote 模式 HTTP + WebSocket 服务
├── print/               # Print 模式（非交互执行与输出格式）
├── plan/ task/ prompt/  # Plan 模式、任务清单、系统提示词构建
└── conversation/ history/ filehistory/ toolresult/
```

## 测试

```bash
# Windows
gradlew.bat test

# Linux / macOS
./gradlew test
```

测试覆盖配置解析、权限判定、上下文压缩、会话恢复、工具配对、团队协作协议、Hook 引擎等核心模块。

## 许可证

[MIT](LICENSE)
