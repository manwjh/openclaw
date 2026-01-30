---
summary: "WebSocket 网关架构、组件和客户端流程"
read_when:
  - 开发网关协议、客户端或传输层时
---
# 网关架构

最后更新：2026-01-22

## 概述

- 单个长连接的 **Gateway（网关）** 拥有所有消息传递界面（通过 Baileys 接入 WhatsApp、通过 grammY 接入 Telegram、Slack、Discord、Signal、iMessage、WebChat）。
- 控制平面客户端（macOS 应用、CLI、Web UI、自动化工具）通过配置的绑定主机（默认为 `127.0.0.1:18789`）上的 **WebSocket** 连接到网关。
- **Nodes（节点）**（macOS/iOS/Android/无头设备）也通过 **WebSocket** 连接，但声明 `role: node` 并带有明确的功能和命令。
- 每个主机一个网关；它是唯一打开 WhatsApp 会话的地方。
- 一个 **canvas host（画布主机）**（默认端口 `18793`）提供代理可编辑的 HTML 和 A2UI。

## 组件和流程

### Gateway（网关守护进程）
- 维护提供商连接。
- 暴露类型化的 WS API（请求、响应、服务器推送事件）。
- 根据 JSON Schema 验证入站帧。
- 发出事件，如 `agent`、`chat`、`presence`、`health`、`heartbeat`、`cron`。

### Clients（客户端：mac 应用 / CLI / web 管理界面）
- 每个客户端一个 WS 连接。
- 发送请求（`health`、`status`、`send`、`agent`、`system-presence`）。
- 订阅事件（`tick`、`agent`、`presence`、`shutdown`）。

### Nodes（节点：macOS / iOS / Android / 无头设备）
- 使用 `role: node` 连接到 **相同的 WS 服务器**。
- 在 `connect` 中提供设备标识；配对是 **基于设备的**（角色为 `node`），批准记录保存在设备配对存储中。
- 暴露命令，如 `canvas.*`、`camera.*`、`screen.record`、`location.get`。

协议详情：
- [网关协议](/gateway/protocol)

### WebChat（网页聊天）
- 使用 Gateway WS API 获取聊天历史和发送消息的静态 UI。
- 在远程设置中，通过与其他客户端相同的 SSH/Tailscale 隧道连接。

## 连接生命周期（单个客户端）

```
Client                    Gateway
  |                          |
  |---- req:connect -------->|
  |<------ res (ok) ---------|   (或 res error + close)
  |   (payload=hello-ok 携带快照: presence + health)
  |                          |
  |<------ event:presence ---|
  |<------ event:tick -------|
  |                          |
  |------- req:agent ------->|
  |<------ res:agent --------|   (确认: {runId,status:"accepted"})
  |<------ event:agent ------|   (流式传输)
  |<------ res:agent --------|   (最终: {runId,status,summary})
  |                          |
```

## 线路协议（摘要）

- 传输：WebSocket，使用 JSON 负载的文本帧。
- 第一帧 **必须** 是 `connect`。
- 握手后：
  - 请求：`{type:"req", id, method, params}` → `{type:"res", id, ok, payload|error}`
  - 事件：`{type:"event", event, payload, seq?, stateVersion?}`
- 如果设置了 `OPENCLAW_GATEWAY_TOKEN`（或 `--token`），`connect.params.auth.token` 必须匹配，否则套接字关闭。
- 幂等键是带副作用的方法（`send`、`agent`）所必需的，以便安全重试；服务器保留一个短期的去重缓存。
- 节点必须在 `connect` 中包含 `role: "node"` 以及功能/命令/权限。

## 配对 + 本地信任

- 所有 WS 客户端（操作员 + 节点）在 `connect` 时都包含一个 **设备标识**。
- 新设备 ID 需要配对批准；网关为后续连接颁发 **设备令牌**。
- **本地** 连接（回环或网关主机自己的 tailnet 地址）可以自动批准，以保持同主机的用户体验流畅。
- **非本地** 连接必须签署 `connect.challenge` 随机数并需要明确批准。
- 网关认证（`gateway.auth.*`）仍然适用于 **所有** 连接，无论是本地还是远程。

详情：[网关协议](/gateway/protocol)、[配对](/start/pairing)、[安全](/gateway/security)。

## 协议类型定义和代码生成

- TypeBox schemas 定义协议。
- JSON Schema 从这些 schemas 生成。
- Swift 模型从 JSON Schema 生成。

## 远程访问

- 首选：Tailscale 或 VPN。
- 替代方案：SSH 隧道
  ```bash
  ssh -N -L 18789:127.0.0.1:18789 user@host
  ```
- 相同的握手 + 认证令牌通过隧道应用。
- 在远程设置中可以为 WS 启用 TLS + 可选的证书固定。

## 运维快照

- 启动：`openclaw gateway`（前台运行，日志输出到 stdout）。
- 健康检查：通过 WS 发送 `health`（也包含在 `hello-ok` 中）。
- 监督：使用 launchd/systemd 实现自动重启。

## 不变量

- 每个主机恰好一个 Gateway 控制单个 Baileys 会话。
- 握手是强制性的；任何非 JSON 或首帧非 connect 都会导致硬关闭。
- 事件不会重放；客户端必须在出现间隙时刷新。
