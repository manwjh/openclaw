---
summary: "代理运行时（嵌入式 p-mono）、工作区契约和会话引导"
read_when:
  - 更改代理运行时、工作区引导或会话行为时
original_path: "docs/concepts/agent.md"
translated_date: "2026-01-XX"
translator: "manwjh"
sync_status: "同步至上游版本 vX.X.X"
last_sync_date: "2026-01-XX"
---
# 代理运行时 🤖

OpenClaw 运行从 **p-mono** 派生的单个嵌入式代理运行时。

## 工作区（必需）

OpenClaw 使用单个代理工作区目录（`agents.defaults.workspace`）作为代理用于工具和上下文的**唯一**工作目录（`cwd`）。

建议：如果缺少 `~/.openclaw/openclaw.json`，请使用 `openclaw setup` 创建它并初始化工作区文件。

完整的工作区布局 + 备份指南：[代理工作区](/concepts/agent-workspace)

如果启用了 `agents.defaults.sandbox`，非主会话可以使用 `agents.defaults.sandbox.workspaceRoot` 下的每个会话工作区覆盖此设置（请参阅[网关配置](/gateway/configuration)）。

## 引导文件（注入）

在 `agents.defaults.workspace` 内，OpenClaw 期望这些用户可编辑的文件：
- `AGENTS.md` — 操作说明 + "内存"
- `SOUL.md` — 角色、边界、语调
- `TOOLS.md` — 用户维护的工具注释（例如 `imsg`、`sag`、约定）
- `BOOTSTRAP.md` — 一次性首次运行仪式（完成后删除）
- `IDENTITY.md` — 代理名称/氛围/表情符号
- `USER.md` — 用户配置文件 + 首选地址

在新会话的第一轮中，OpenClaw 将这些文件的内容直接注入到代理上下文中。

空白文件会被跳过。大型文件会被修剪和截断，并带有标记，以便提示保持精简（读取文件以获取完整内容）。

如果缺少文件，OpenClaw 会注入单个"缺少文件"标记行（`openclaw setup` 将创建安全的默认模板）。

`BOOTSTRAP.md` 仅为**全新工作区**创建（不存在其他引导文件）。如果您在完成仪式后删除它，它不应在后续重启时重新创建。

要完全禁用引导文件创建（对于预填充的工作区），请设置：

```json5
{ agent: { skipBootstrap: true } }
```

## 内置工具

核心工具（read/exec/edit/write 和相关系统工具）始终可用，
受工具策略约束。`apply_patch` 是可选的，由 `tools.exec.applyPatch` 控制。`TOOLS.md` **不**控制哪些工具存在；它是关于*您*希望如何使用它们的指导。

## 技能

OpenClaw 从三个位置加载技能（工作区在名称冲突时获胜）：
- 捆绑（随安装一起提供）
- 托管/本地：`~/.openclaw/skills`
- 工作区：`<workspace>/skills`

技能可以通过配置/环境进行控制（请参阅[网关配置](/gateway/configuration)中的 `skills`）。

## p-mono 集成

OpenClaw 重用 p-mono 代码库的片段（模型/工具），但**会话管理、发现和工具连接是 OpenClaw 拥有的**。

- 没有 p-coding 代理运行时。
- 不查阅 `~/.pi/agent` 或 `<workspace>/.pi` 设置。

## 会话

会话转录以 JSONL 格式存储在：
- `~/.openclaw/agents/<agentId>/sessions/<SessionId>.jsonl`

会话 ID 是稳定的，由 OpenClaw 选择。
不读取旧版 Pi/Tau 会话文件夹。

## 流式传输时的转向

当队列模式为 `steer` 时，入站消息会注入到当前运行中。
在**每次工具调用之后**检查队列；如果存在排队的消息，
跳过当前助手消息的剩余工具调用（工具结果错误，显示 "Skipped due to queued user message."），然后在下一个助手响应之前注入排队的用户消息。

当队列模式为 `followup` 或 `collect` 时，入站消息会保留直到当前轮次结束，然后使用排队的有效负载开始新的代理轮次。有关模式 + 防抖/上限行为，请参阅[队列](/concepts/queue)。

块流式传输在完成时立即发送已完成的助手块；它**默认关闭**（`agents.defaults.blockStreamingDefault: "off"`）。
通过 `agents.defaults.blockStreamingBreak` 调整边界（`text_end` 与 `message_end`；默认为 text_end）。
通过 `agents.defaults.blockStreamingChunk` 控制软块分块（默认为 800–1200 字符；优先段落换行，然后是换行符；最后是句子）。
通过 `agents.defaults.blockStreamingCoalesce` 合并流式块以减少单行垃圾邮件（发送前基于空闲的合并）。非 Telegram 通道需要显式 `*.blockStreaming: true` 才能启用块回复。
详细工具摘要在工具开始时发出（无防抖）；当可用时，Control UI 通过代理事件流式传输工具输出。
更多详细信息：[流式传输 + 分块](/concepts/streaming)。

## 模型引用

配置中的模型引用（例如 `agents.defaults.model` 和 `agents.defaults.models`）通过在**第一个**`/` 上拆分来解析。

- 配置模型时使用 `provider/model`。
- 如果模型 ID 本身包含 `/`（OpenRouter 风格），请包含提供程序前缀（例如：`openrouter/moonshotai/kimi-k2`）。
- 如果您省略提供程序，OpenClaw 将输入视为别名或**默认提供程序**的模型（仅在模型 ID 中没有 `/` 时有效）。

## 配置（最小）

至少，设置：
- `agents.defaults.workspace`
- `channels.whatsapp.allowFrom`（强烈推荐）

---

*下一步：[群聊](/concepts/group-messages)* 🦞
