---
summary: "多代理路由配置：每个代理的模型、工作区、沙箱和绑定"
read_when:
  - 配置多个隔离的代理时
  - 需要为不同代理设置不同的模型时
  - 设置代理路由和绑定时
original_path: "docs/gateway/configuration.md#multi-agent-routing"
translated_date: "2026-01-XX"
translator: "manwjh"
sync_status: "同步至上游版本 vX.X.X"
last_sync_date: "2026-01-XX"
---
# 多代理路由配置

在一个 Gateway 内运行多个隔离的代理（独立的工作区、`agentDir`、会话）。
入站消息通过绑定路由到代理。

## 配置结构

### `agents.list[]`：每个代理的覆盖

- `id`：稳定的代理 ID（必需）。
- `default`：可选；当设置多个时，第一个获胜并记录警告。
  如果未设置，列表中的**第一个条目**是默认代理。
- `name`：代理的显示名称。
- `workspace`：默认 `~/.openclaw/workspace-<agentId>`（对于 `main`，回退到 `agents.defaults.workspace`）。
- `agentDir`：默认 `~/.openclaw/agents/<agentId>/agent`。
- `model`：每个代理的默认模型，为该代理覆盖 `agents.defaults.model`。
  - 字符串形式：`"provider/model"`，仅覆盖 `agents.defaults.model.primary`
  - 对象形式：`{ primary, fallbacks }`（fallbacks 覆盖 `agents.defaults.model.fallbacks`；`[]` 禁用该代理的全局 fallbacks）
- `identity`：每个代理的名称/主题/表情符号（用于提及模式和确认反应）。
- `groupChat`：每个代理的提及门控（`mentionPatterns`）。
- `sandbox`：每个代理的沙箱配置（覆盖 `agents.defaults.sandbox`）。
  - `mode`：`"off"` | `"non-main"` | `"all"`
  - `workspaceAccess`：`"none"` | `"ro"` | `"rw"`
  - `scope`：`"session"` | `"agent"` | `"shared"`
  - `workspaceRoot`：自定义沙箱工作区根
  - `docker`：每个代理的 docker 覆盖（例如 `image`、`network`、`env`、`setupCommand`、限制；当 `scope: "shared"` 时忽略）
  - `browser`：每个代理的沙箱浏览器覆盖（当 `scope: "shared"` 时忽略）
  - `prune`：每个代理的沙箱清理覆盖（当 `scope: "shared"` 时忽略）
- `subagents`：每个代理的子代理默认值。
  - `allowAgents`：从此代理进行 `sessions_spawn` 的代理 ID 允许列表（`["*"]` = 允许任何；默认：仅相同代理）
- `tools`：每个代理的工具限制（在沙箱工具策略之前应用）。
  - `profile`：基础工具配置文件（在 allow/deny 之前应用）
  - `allow`：允许的工具名称数组
  - `deny`：拒绝的工具名称数组（deny 获胜）
- `agents.defaults`：共享的代理默认值（模型、工作区、沙箱等）。

### `bindings[]`：将入站消息路由到 `agentId`

- `match.channel`（必需）
- `match.accountId`（可选；`*` = 任何账户；省略 = 默认账户）
- `match.peer`（可选；`{ kind: dm|group|channel, id }`）
- `match.guildId` / `match.teamId`（可选；特定于通道）

## 确定性匹配顺序

1) `match.peer`
2) `match.guildId`
3) `match.teamId`
4) `match.accountId`（精确，无 peer/guild/team）
5) `match.accountId: "*"`（通道范围，无 peer/guild/team）
6) 默认代理（`agents.list[].default`，否则是第一个列表条目，否则 `"main"`）

在每个匹配层级内，`bindings` 中的第一个匹配条目获胜。

## 每个代理的访问配置文件（多代理）

每个代理可以携带自己的沙箱 + 工具策略。使用此功能在一个网关中混合访问
级别：
- **完全访问**（个人代理）
- **只读**工具 + 工作区
- **无文件系统访问**（仅消息传递/会话工具）

有关优先级和
其他示例，请参阅[多代理沙箱和工具](/multi-agent-sandbox-tools)。

### 完全访问（无沙箱）

```json5
{
  agents: {
    list: [
      {
        id: "personal",
        workspace: "~/.openclaw/workspace-personal",
        sandbox: { mode: "off" }
      }
    ]
  }
}
```

### 只读工具 + 只读工作区

```json5
{
  agents: {
    list: [
      {
        id: "family",
        workspace: "~/.openclaw/workspace-family",
        sandbox: {
          mode: "all",
          scope: "agent",
          workspaceAccess: "ro"
        },
        tools: {
          allow: ["read", "sessions_list", "sessions_history", "sessions_send", "sessions_spawn", "session_status"],
          deny: ["write", "edit", "apply_patch", "exec", "process", "browser"]
        }
      }
    ]
  }
}
```

### 无文件系统访问（启用消息传递/会话工具）

```json5
{
  agents: {
    list: [
      {
        id: "public",
        workspace: "~/.openclaw/workspace-public",
        sandbox: {
          mode: "all",
          scope: "agent",
          workspaceAccess: "none"
        },
        tools: {
          allow: ["sessions_list", "sessions_history", "sessions_send", "sessions_spawn", "session_status", "whatsapp", "telegram", "slack", "discord", "gateway"],
          deny: ["read", "write", "edit", "apply_patch", "exec", "process", "browser", "canvas", "nodes", "cron", "gateway", "image"]
        }
      }
    ]
  }
}
```

## 示例：两个 WhatsApp 账户 → 两个代理

```json5
{
  agents: {
    list: [
      { id: "home", default: true, workspace: "~/.openclaw/workspace-home" },
      { id: "work", workspace: "~/.openclaw/workspace-work" }
    ]
  },
  bindings: [
    { agentId: "home", match: { channel: "whatsapp", accountId: "personal" } },
    { agentId: "work", match: { channel: "whatsapp", accountId: "biz" } }
  ],
  channels: {
    whatsapp: {
      accounts: {
        personal: {},
        biz: {},
      }
    }
  }
}
```

## 每个代理的模型配置示例

### 字符串形式（仅覆盖 primary）

```json5
{
  agents: {
    list: [
      {
        id: "chat",
        model: "anthropic/claude-sonnet-4-5"  // 仅覆盖 primary
      }
    ]
  }
}
```

### 对象形式（配置 primary 和 fallbacks）

```json5
{
  agents: {
    list: [
      {
        id: "opus",
        model: {
          primary: "anthropic/claude-opus-4-5",
          fallbacks: ["openrouter/deepseek/deepseek-r1:free"]
        }
      },
      {
        id: "chat",
        model: {
          primary: "anthropic/claude-sonnet-4-5",
          fallbacks: []  // 禁用全局 fallbacks
        }
      }
    ]
  }
}
```

## 按通道路由不同模型

```json5
{
  agents: {
    list: [
      {
        id: "chat",
        name: "Everyday",
        workspace: "~/.openclaw/workspace-chat",
        model: "anthropic/claude-sonnet-4-5"
      },
      {
        id: "opus",
        name: "Deep Work",
        workspace: "~/.openclaw/workspace-opus",
        model: "anthropic/claude-opus-4-5"
      }
    ]
  },
  bindings: [
    { agentId: "chat", match: { channel: "whatsapp" } },
    { agentId: "opus", match: { channel: "telegram" } }
  ]
}
```

## 同一通道，一个对等到不同模型

```json5
{
  agents: {
    list: [
      { 
        id: "chat", 
        name: "Everyday", 
        workspace: "~/.openclaw/workspace-chat", 
        model: "anthropic/claude-sonnet-4-5" 
      },
      { 
        id: "opus", 
        name: "Deep Work", 
        workspace: "~/.openclaw/workspace-opus", 
        model: "anthropic/claude-opus-4-5" 
      }
    ]
  },
  bindings: [
    { agentId: "opus", match: { channel: "whatsapp", peer: { kind: "dm", id: "+15551234567" } } },
    { agentId: "chat", match: { channel: "whatsapp" } }
  ]
}
```

对等绑定总是获胜，因此将它们放在通道范围规则之上。

## 相关文档

- [多代理路由](/concepts/multi-agent) - 概念和详细示例
- [模型 CLI](/concepts/models) - 模型选择和配置
- [网关配置](/gateway/configuration) - 完整配置参考
