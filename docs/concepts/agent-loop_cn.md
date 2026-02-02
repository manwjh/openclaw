---
summary: "代理循环生命周期、流和等待语义"
read_when:
  - 需要代理循环或生命周期事件的准确演练时
original_path: "docs/concepts/agent-loop.md"
translated_date: "2026-01-XX"
translator: "manwjh"
sync_status: "同步至上游版本 vX.X.X"
last_sync_date: "2026-01-XX"
---
# 代理循环（OpenClaw）

代理循环是代理的完整"真实"运行：接收 → 上下文组装 → 模型推理 → 工具执行 → 流式回复 → 持久化。这是将消息转换为操作和最终回复的权威路径，同时保持会话状态一致。

在 OpenClaw 中，循环是每个会话的单个序列化运行，当模型思考、调用工具和流式输出时发出生命周期和流事件。本文档解释了该真实循环是如何端到端连接的。

## 入口点
- Gateway RPC：`agent` 和 `agent.wait`。
- CLI：`agent` 命令。

## 工作原理（高级）
1) `agent` RPC 验证参数，解析会话（sessionKey/sessionId），持久化会话元数据，立即返回 `{ runId, acceptedAt }`。
2) `agentCommand` 运行代理：
   - 解析模型 + thinking/verbose 默认值
   - 加载技能快照
   - 调用 `runEmbeddedPiAgent`（pi-agent-core 运行时）
   - 如果嵌入式循环未发出生命周期结束/错误，则发出**生命周期结束/错误**
3) `runEmbeddedPiAgent`：
   - 通过每个会话 + 全局队列序列化运行
   - 解析模型 + 身份验证配置文件并构建 pi 会话
   - 订阅 pi 事件并流式传输助手/工具增量
   - 强制执行超时 → 如果超过则中止运行
   - 返回有效负载 + 使用元数据
4) `subscribeEmbeddedPiSession` 将 pi-agent-core 事件桥接到 OpenClaw `agent` 流：
   - 工具事件 => `stream: "tool"`
   - 助手增量 => `stream: "assistant"`
   - 生命周期事件 => `stream: "lifecycle"` (`phase: "start" | "end" | "error"`)
5) `agent.wait` 使用 `waitForAgentJob`：
   - 等待 `runId` 的**生命周期结束/错误**
   - 返回 `{ status: ok|error|timeout, startedAt, endedAt, error? }`

## 队列 + 并发
- 运行按会话键（会话通道）序列化，并可选择通过全局通道。
- 这防止了工具/会话竞争并保持会话历史一致。
- 消息传递通道可以选择队列模式（collect/steer/followup），这些模式会馈送到此通道系统。
  请参阅[命令队列](/concepts/queue)。

## 会话 + 工作区准备
- 解析并创建工作区；沙箱运行可能会重定向到沙箱工作区根。
- 加载技能（或从快照重用）并注入到环境和提示中。
- 解析引导/上下文文件并注入到系统提示报告中。
- 获取会话写入锁；`SessionManager` 在流式传输之前打开并准备。

## 提示组装 + 系统提示
- 系统提示由 OpenClaw 的基础提示、技能提示、引导上下文和每次运行的覆盖构建。
- 强制执行模型特定的限制和压缩保留令牌。
- 有关模型看到的内容，请参阅[系统提示](/concepts/system-prompt)。

## 钩子点（可以拦截的位置）
OpenClaw 有两个钩子系统：
- **内部钩子**（Gateway 钩子）：用于命令和生命周期事件的事件驱动脚本。
- **插件钩子**：代理/工具生命周期和网关管道内的扩展点。

### 内部钩子（Gateway 钩子）
- **`agent:bootstrap`**：在构建引导文件时运行，在系统提示最终确定之前。使用此功能添加/删除引导上下文文件。
- **命令钩子**：`/new`、`/reset`、`/stop` 和其他命令事件（请参阅 Hooks 文档）。

有关设置和示例，请参阅[钩子](/hooks)。

### 插件钩子（代理 + 网关生命周期）
这些在代理循环或网关管道内运行：
- **`before_agent_start`**：在运行开始之前注入上下文或覆盖系统提示。
- **`agent_end`**：在完成后检查最终消息列表和运行元数据。
- **`before_compaction` / `after_compaction`**：观察或注释压缩周期。
- **`before_tool_call` / `after_tool_call`**：拦截工具参数/结果。
- **`tool_result_persist`**：在将工具结果写入会话转录之前同步转换它们。
- **`message_received` / `message_sending` / `message_sent`**：入站 + 出站消息钩子。
- **`session_start` / `session_end`**：会话生命周期边界。
- **`gateway_start` / `gateway_stop`**：网关生命周期事件。

有关钩子 API 和注册详细信息，请参阅[插件](/plugin#plugin-hooks)。

## 流式传输 + 部分回复
- 助手增量从 pi-agent-core 流式传输并作为 `assistant` 事件发出。
- 块流式传输可以在 `text_end` 或 `message_end` 上发出部分回复。
- 推理流式传输可以作为单独的流或作为块回复发出。
- 有关分块和块回复行为，请参阅[流式传输](/concepts/streaming)。

## 工具执行 + 消息传递工具
- 工具开始/更新/结束事件在 `tool` 流上发出。
- 工具结果在记录/发出之前会针对大小和图像有效负载进行清理。
- 消息传递工具发送会被跟踪以抑制重复的助手确认。

## 回复整形 + 抑制
- 最终有效负载从以下内容组装：
  - 助手文本（和可选的推理）
  - 内联工具摘要（当 verbose + 允许时）
  - 模型错误时的助手错误文本
- `NO_REPLY` 被视为静默令牌并从出站有效负载中过滤。
- 从最终有效负载列表中删除消息传递工具重复项。
- 如果没有可渲染的有效负载保留且工具出错，则发出回退工具错误回复
  （除非消息传递工具已发送用户可见的回复）。

## 压缩 + 重试
- 自动压缩发出 `compaction` 流事件并可以触发重试。
- 重试时，内存缓冲区和工具摘要会重置以避免重复输出。
- 有关压缩管道，请参阅[压缩](/concepts/compaction)。

## 事件流（当前）
- `lifecycle`：由 `subscribeEmbeddedPiSession` 发出（以及 `agentCommand` 的回退）
- `assistant`：从 pi-agent-core 流式传输的增量
- `tool`：从 pi-agent-core 流式传输的工具事件

## 聊天通道处理
- 助手增量被缓冲到聊天 `delta` 消息中。
- 在**生命周期结束/错误**时发出聊天 `final`。

## 超时
- `agent.wait` 默认值：30 秒（仅等待）。`timeoutMs` 参数覆盖。
- 代理运行时：`agents.defaults.timeoutSeconds` 默认 600 秒；在 `runEmbeddedPiAgent` 中止计时器中强制执行。

## 可能提前结束的地方
- 代理超时（中止）
- AbortSignal（取消）
- Gateway 断开连接或 RPC 超时
- `agent.wait` 超时（仅等待，不会停止代理）
