---
summary: "会话管理规则、键和聊天的持久化"
read_when:
  - 修改会话处理或存储时
original_path: "docs/concepts/session.md"
translated_date: "2026-01-XX"
translator: "manwjh"
sync_status: "同步至上游版本 vX.X.X"
last_sync_date: "2026-01-XX"
---
# 会话管理

OpenClaw 将**每个代理一个直接聊天会话**视为主要会话。直接聊天折叠到 `agent:<agentId>:<mainKey>`（默认 `main`），而群组/频道聊天获得自己的键。`session.mainKey` 会被遵守。

使用 `session.dmScope` 控制**直接消息**的分组方式：
- `main`（默认）：所有 DM 共享主会话以保持连续性。
- `per-peer`：按发送者 ID 跨通道隔离。
- `per-channel-peer`：按通道 + 发送者隔离（推荐用于多用户收件箱）。
- `per-account-channel-peer`：按账户 + 通道 + 发送者隔离（推荐用于多账户收件箱）。
使用 `session.identityLinks` 将提供程序前缀的对等 ID 映射到规范身份，以便在使用 `per-peer`、`per-channel-peer` 或 `per-account-channel-peer` 时，同一人在跨通道共享 DM 会话。

## Gateway 是真相来源
所有会话状态都**由网关拥有**（"主" OpenClaw）。UI 客户端（macOS 应用、WebChat 等）必须查询网关以获取会话列表和令牌计数，而不是读取本地文件。

- 在**远程模式**中，您关心的会话存储位于远程网关主机上，而不是您的 Mac 上。
- UI 中显示的令牌计数来自网关的存储字段（`inputTokens`、`outputTokens`、`totalTokens`、`contextTokens`）。客户端不会解析 JSONL 转录来"修复"总数。

## 状态存储位置
- 在**网关主机**上：
  - 存储文件：`~/.openclaw/agents/<agentId>/sessions/sessions.json`（每个代理）。
  - 转录：`~/.openclaw/agents/<agentId>/sessions/<SessionId>.jsonl`（Telegram 主题会话使用 `.../<SessionId>-topic-<threadId>.jsonl`）。
  - 存储是一个映射 `sessionKey -> { sessionId, updatedAt, ... }`。删除条目是安全的；它们会按需重新创建。
  - 群组条目可能包括 `displayName`、`channel`、`subject`、`room` 和 `space` 以在 UI 中标记会话。
  - 会话条目包括 `origin` 元数据（标签 + 路由提示），以便 UI 可以解释会话的来源。
  - OpenClaw **不**读取旧版 Pi/Tau 会话文件夹。

## 会话修剪
OpenClaw 默认在 LLM 调用之前从内存上下文中修剪**旧工具结果**。
这**不会**重写 JSONL 历史。请参阅[/concepts/session-pruning](/concepts/session-pruning)。

## 压缩前内存刷新
当会话接近自动压缩时，OpenClaw 可以运行**静默内存刷新**
轮次，提醒模型将持久化笔记写入磁盘。这仅在
工作区可写时运行。请参阅[内存](/concepts/memory)和
[压缩](/concepts/compaction)。

## 映射传输 → 会话键
- 直接聊天遵循 `session.dmScope`（默认 `main`）。
  - `main`：`agent:<agentId>:<mainKey>`（跨设备/通道的连续性）。
    - 多个电话号码和通道可以映射到同一个代理主键；它们充当进入一个对话的传输。
  - `per-peer`：`agent:<agentId>:dm:<peerId>`。
  - `per-channel-peer`：`agent:<agentId>:<channel>:dm:<peerId>`。
  - `per-account-channel-peer`：`agent:<agentId>:<channel>:<accountId>:dm:<peerId>`（accountId 默认为 `default`）。
  - 如果 `session.identityLinks` 匹配提供程序前缀的对等 ID（例如 `telegram:123`），规范键替换 `<peerId>`，以便同一人在跨通道共享会话。
- 群组聊天隔离状态：`agent:<agentId>:<channel>:group:<id>`（房间/通道使用 `agent:<agentId>:<channel>:channel:<id>`）。
  - Telegram 论坛主题在群组 ID 后附加 `:topic:<threadId>` 以进行隔离。
  - 仍识别旧版 `group:<id>` 键以进行迁移。
- 入站上下文可能仍使用 `group:<id>`；通道从 `Provider` 推断并规范化为规范形式 `agent:<agentId>:<channel>:group:<id>`。
- 其他来源：
  - Cron 作业：`cron:<job.id>`
  - Webhooks：`hook:<uuid>`（除非由钩子显式设置）
  - 节点运行：`node-<nodeId>`

## 生命周期
- 重置策略：会话会重复使用直到过期，并在下一条入站消息时评估过期。
- 每日重置：默认为**网关主机本地时间凌晨 4:00**。一旦会话的最后更新早于最近的每日重置时间，会话就会过期。
- 空闲重置（可选）：`idleMinutes` 添加滑动空闲窗口。当同时配置每日和空闲重置时，**无论哪个先过期**都会强制新会话。
- 旧版仅空闲：如果您设置 `session.idleMinutes` 而没有任何 `session.reset`/`resetByType` 配置，OpenClaw 保持仅空闲模式以保持向后兼容性。
- 按类型覆盖（可选）：`resetByType` 允许您覆盖 `dm`、`group` 和 `thread` 会话的策略（thread = Slack/Discord 线程、Telegram 主题、连接器提供时的 Matrix 线程）。
- 按通道覆盖（可选）：`resetByChannel` 覆盖通道的重置策略（适用于该通道的所有会话类型，并优先于 `reset`/`resetByType`）。
- 重置触发器：精确的 `/new` 或 `/reset`（加上 `resetTriggers` 中的任何额外项）启动新的会话 ID 并传递消息的其余部分。`/new <model>` 接受模型别名、`provider/model` 或提供程序名称（模糊匹配）以设置新会话模型。如果单独发送 `/new` 或 `/reset`，OpenClaw 运行简短的"hello"问候轮次以确认重置。
- 手动重置：从存储中删除特定键或删除 JSONL 转录；下一条消息会重新创建它们。
- 隔离的 cron 作业始终为每次运行生成新的 `sessionId`（无空闲重用）。

## 发送策略（可选）
阻止特定会话类型的传递，而无需列出单个 ID。

```json5
{
  session: {
    sendPolicy: {
      rules: [
        { action: "deny", match: { channel: "discord", chatType: "group" } },
        { action: "deny", match: { keyPrefix: "cron:" } }
      ],
      default: "allow"
    }
  }
}
```

运行时覆盖（仅所有者）：
- `/send on` → 允许此会话
- `/send off` → 拒绝此会话
- `/send inherit` → 清除覆盖并使用配置规则
将这些作为独立消息发送，以便它们注册。

## 配置（可选重命名示例）
```json5
// ~/.openclaw/openclaw.json
{
  session: {
    scope: "per-sender",      // 保持群组键分离
    dmScope: "main",          // DM 连续性（为共享收件箱设置 per-channel-peer/per-account-channel-peer）
    identityLinks: {
      alice: ["telegram:123456789", "discord:987654321012345678"]
    },
    reset: {
      // 默认值：mode=daily，atHour=4（网关主机本地时间）。
      // 如果您还设置 idleMinutes，无论哪个先过期都会获胜。
      mode: "daily",
      atHour: 4,
      idleMinutes: 120
    },
    resetByType: {
      thread: { mode: "daily", atHour: 4 },
      dm: { mode: "idle", idleMinutes: 240 },
      group: { mode: "idle", idleMinutes: 120 }
    },
    resetByChannel: {
      discord: { mode: "idle", idleMinutes: 10080 }
    },
    resetTriggers: ["/new", "/reset"],
    store: "~/.openclaw/agents/{agentId}/sessions/sessions.json",
    mainKey: "main",
  }
}
```

## 检查
- `openclaw status` — 显示存储路径和最近的会话。
- `openclaw sessions --json` — 转储每个条目（使用 `--active <minutes>` 过滤）。
- `openclaw gateway call sessions.list --params '{}'` — 从运行的网关获取会话（使用 `--url`/`--token` 进行远程网关访问）。
- 在聊天中作为独立消息发送 `/status` 以查看代理是否可访问、会话上下文使用了多少、当前的 thinking/verbose 切换，以及您的 WhatsApp web 凭据上次刷新的时间（有助于发现重新链接需求）。
- 发送 `/context list` 或 `/context detail` 以查看系统提示和注入的工作区文件中的内容（以及最大的上下文贡献者）。
- 作为独立消息发送 `/stop` 以中止当前运行，清除该会话的排队后续操作，并停止从中产生的任何子代理运行（回复包括停止计数）。
- 作为独立消息发送 `/compact`（可选说明）以总结较旧的上下文并释放窗口空间。请参阅[/concepts/compaction](/concepts/compaction)。
- 可以直接打开 JSONL 转录以查看完整轮次。

## 提示
- 保持主键专用于 1:1 流量；让群组保持自己的键。
- 自动化清理时，删除单个键而不是整个存储，以保留其他地方的上下文。

## 会话来源元数据
每个会话条目在 `origin` 中记录其来源（尽力而为）：
- `label`：人类标签（从对话标签 + 群组主题/通道解析）
- `provider`：规范化的通道 ID（包括扩展）
- `from`/`to`：来自入站信封的原始路由 ID
- `accountId`：提供程序账户 ID（多账户时）
- `threadId`：通道支持时的线程/主题 ID
来源字段为直接消息、通道和群组填充。如果
连接器仅更新传递路由（例如，保持 DM 主会话
新鲜），它仍应提供入站上下文，以便会话保持其
解释器元数据。扩展可以通过在入站中发送 `ConversationLabel`、
`GroupSubject`、`GroupChannel`、`GroupSpace` 和 `SenderName` 来执行此操作
上下文并调用 `recordSessionMetaFromInbound`（或传递相同的上下文
到 `updateLastRoute`）。
