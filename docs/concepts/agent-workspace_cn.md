---
summary: "代理工作区：位置、布局和备份策略"
read_when:
  - 需要解释代理工作区或其文件布局时
  - 想要备份或迁移代理工作区时
original_path: "docs/concepts/agent-workspace.md"
translated_date: "2026-01-XX"
translator: "manwjh"
sync_status: "同步至上游版本 vX.X.X"
last_sync_date: "2026-01-XX"
---
# 代理工作区

工作区是代理的家。它是用于文件工具和工作区上下文的唯一工作目录。请保持其私密性，并将其视为内存。

这与 `~/.openclaw/` 是分开的，后者存储配置、凭据和会话。

**重要提示：** 工作区是**默认的 cwd**，不是硬沙箱。工具相对于工作区解析相对路径，但绝对路径仍可以访问主机上的其他地方，除非启用了沙箱。如果需要隔离，请使用 [`agents.defaults.sandbox`](/gateway/sandboxing)（和/或每个代理的沙箱配置）。
当启用沙箱且 `workspaceAccess` 不是 `"rw"` 时，工具在 `~/.openclaw/sandboxes` 下的沙箱工作区内运行，而不是您的主机工作区。

## 默认位置

- 默认：`~/.openclaw/workspace`
- 如果设置了 `OPENCLAW_PROFILE` 且不是 `"default"`，默认值变为 `~/.openclaw/workspace-<profile>`。
- 在 `~/.openclaw/openclaw.json` 中覆盖：

```json5
{
  agent: {
    workspace: "~/.openclaw/workspace"
  }
}
```

`openclaw onboard`、`openclaw configure` 或 `openclaw setup` 将创建工作区，并在缺少引导文件时填充它们。

如果您已经自己管理工作区文件，可以禁用引导文件创建：

```json5
{ agent: { skipBootstrap: true } }
```

## 额外的工作区文件夹

较旧的安装可能已创建 `~/openclaw`。保留多个工作区目录可能会导致混淆的身份验证或状态漂移，因为一次只有一个工作区处于活动状态。

**建议：** 保留单个活动工作区。如果您不再使用额外的文件夹，请将它们归档或移动到废纸篓（例如 `trash ~/openclaw`）。
如果您有意保留多个工作区，请确保 `agents.defaults.workspace` 指向活动的工作区。

`openclaw doctor` 在检测到额外的工作区目录时会发出警告。

## 工作区文件映射（每个文件的含义）

这些是 OpenClaw 在工作区内期望的标准文件：

- `AGENTS.md`
  - 代理的操作说明以及它应如何使用内存。
  - 在每个会话开始时加载。
  - 放置规则、优先级和"如何行为"细节的好地方。

- `SOUL.md`
  - 角色、语调和边界。
  - 每个会话都加载。

- `USER.md`
  - 用户是谁以及如何称呼他们。
  - 每个会话都加载。

- `IDENTITY.md`
  - 代理的名称、氛围和表情符号。
  - 在引导仪式期间创建/更新。

- `TOOLS.md`
  - 关于您的本地工具和约定的注释。
  - 不控制工具的可用性；它只是指导。

- `HEARTBEAT.md`
  - 心跳运行的可选小清单。
  - 保持简短以避免令牌消耗。

- `BOOT.md`
  - 在启用内部钩子时，网关重启时执行的可选启动清单。
  - 保持简短；使用消息工具进行出站发送。

- `BOOTSTRAP.md`
  - 一次性首次运行仪式。
  - 仅为新工作区创建。
  - 仪式完成后删除它。

- `memory/YYYY-MM-DD.md`
  - 每日内存日志（每天一个文件）。
  - 建议在会话开始时阅读今天和昨天的内容。

- `MEMORY.md`（可选）
  - 精选的长期内存。
  - 仅在主要的私有会话中加载（不在共享/组上下文中）。

有关工作流和自动内存刷新，请参阅[内存](/concepts/memory)。

- `skills/`（可选）
  - 工作区特定的技能。
  - 当名称冲突时覆盖托管/捆绑的技能。

- `canvas/`（可选）
  - 用于节点显示的 Canvas UI 文件（例如 `canvas/index.html`）。

如果缺少任何引导文件，OpenClaw 会在会话中注入"缺少文件"标记并继续。大型引导文件在注入时会被截断；
通过 `agents.defaults.bootstrapMaxChars`（默认值：20000）调整限制。
`openclaw setup` 可以重新创建缺少的默认值，而不会覆盖现有文件。

## 不在工作区中的内容

这些位于 `~/.openclaw/` 下，不应提交到工作区仓库：

- `~/.openclaw/openclaw.json`（配置）
- `~/.openclaw/credentials/`（OAuth 令牌、API 密钥）
- `~/.openclaw/agents/<agentId>/sessions/`（会话转录 + 元数据）
- `~/.openclaw/skills/`（托管技能）

如果您需要迁移会话或配置，请单独复制它们，并将它们排除在版本控制之外。

## Git 备份（推荐，私有）

将工作区视为私有内存。将其放在**私有** git 仓库中，以便备份和恢复。

在运行 Gateway 的机器上运行这些步骤（工作区所在的位置）。

### 1) 初始化仓库

如果已安装 git，全新的工作区会自动初始化。如果此工作区还不是仓库，请运行：

```bash
cd ~/.openclaw/workspace
git init
git add AGENTS.md SOUL.md TOOLS.md IDENTITY.md USER.md HEARTBEAT.md memory/
git commit -m "Add agent workspace"
```

### 2) 添加私有远程（适合初学者的选项）

选项 A：GitHub Web UI

1. 在 GitHub 上创建一个新的**私有**仓库。
2. 不要使用 README 初始化（避免合并冲突）。
3. 复制 HTTPS 远程 URL。
4. 添加远程并推送：

```bash
git branch -M main
git remote add origin <https-url>
git push -u origin main
```

选项 B：GitHub CLI (`gh`)

```bash
gh auth login
gh repo create openclaw-workspace --private --source . --remote origin --push
```

选项 C：GitLab Web UI

1. 在 GitLab 上创建一个新的**私有**仓库。
2. 不要使用 README 初始化（避免合并冲突）。
3. 复制 HTTPS 远程 URL。
4. 添加远程并推送：

```bash
git branch -M main
git remote add origin <https-url>
git push -u origin main
```

### 3) 持续更新

```bash
git status
git add .
git commit -m "Update memory"
git push
```

## 不要提交密钥

即使在私有仓库中，也要避免在工作区中存储密钥：

- API 密钥、OAuth 令牌、密码或私有凭据。
- `~/.openclaw/` 下的任何内容。
- 聊天或敏感附件的原始转储。

如果您必须存储敏感引用，请使用占位符，并将真实密钥保存在其他地方（密码管理器、环境变量或 `~/.openclaw/`）。

建议的 `.gitignore` 起始文件：

```gitignore
.DS_Store
.env
**/*.key
**/*.pem
**/secrets*
```

## 将工作区移动到新机器

1. 将仓库克隆到所需路径（默认 `~/.openclaw/workspace`）。
2. 在 `~/.openclaw/openclaw.json` 中将 `agents.defaults.workspace` 设置为该路径。
3. 运行 `openclaw setup --workspace <path>` 以填充任何缺少的文件。
4. 如果您需要会话，请单独从旧机器复制 `~/.openclaw/agents/<agentId>/sessions/`。

## 高级说明

- 多代理路由可以为每个代理使用不同的工作区。有关路由配置，请参阅[通道路由](/concepts/channel-routing)。
- 如果启用了 `agents.defaults.sandbox`，非主会话可以使用 `agents.defaults.sandbox.workspaceRoot` 下的每个会话沙箱工作区。
