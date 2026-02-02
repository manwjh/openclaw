---
summary: "OpenClaw 系统提示包含的内容以及如何组装"
read_when:
  - 编辑系统提示文本、工具列表或时间/心跳部分时
  - 更改工作区引导或技能注入行为时
original_path: "docs/concepts/system-prompt.md"
translated_date: "2026-01-XX"
translator: "manwjh"
sync_status: "同步至上游版本 vX.X.X"
last_sync_date: "2026-01-XX"
---
# 系统提示

OpenClaw 为每个代理运行构建自定义系统提示。提示是** OpenClaw 拥有的**，不使用 p-coding-agent 默认提示。

提示由 OpenClaw 组装并注入到每个代理运行中。

## 结构

提示有意紧凑并使用固定部分：

- **工具**：当前工具列表 + 简短描述。
- **技能**（可用时）：告诉模型如何按需加载技能说明。
- **OpenClaw 自我更新**：如何运行 `config.apply` 和 `update.run`。
- **工作区**：工作目录（`agents.defaults.workspace`）。
- **文档**：OpenClaw 文档的本地路径（仓库或 npm 包）以及何时阅读它们。
- **工作区文件（注入）**：指示下面包含引导文件。
- **沙箱**（启用时）：指示沙箱运行时、沙箱路径以及是否可用提升执行。
- **当前日期和时间**：用户本地时间、时区和时间格式。
- **回复标签**：支持提供程序的可选回复标签语法。
- **心跳**：心跳提示和确认行为。
- **运行时**：主机、OS、node、模型、仓库根（检测到时）、思考级别（一行）。
- **推理**：当前可见性级别 + /reasoning 切换提示。

## 提示模式

OpenClaw 可以为子代理呈现较小的系统提示。运行时为每个运行设置一个
`promptMode`（不是面向用户的配置）：

- `full`（默认）：包括上面的所有部分。
- `minimal`：用于子代理；省略**技能**、**内存回忆**、**OpenClaw
  自我更新**、**模型别名**、**用户身份**、**回复标签**、
  **消息传递**、**静默回复**和**心跳**。工具、工作区、
  沙箱、当前日期和时间（已知时）、运行时和注入的上下文保持
  可用。
- `none`：仅返回基础身份行。

当 `promptMode=minimal` 时，额外注入的提示标记为**子代理
上下文**而不是**群组聊天上下文**。

## 工作区引导注入

引导文件被修剪并附加在**项目上下文**下，以便模型看到身份和配置文件上下文，而无需显式读取：

- `AGENTS.md`
- `SOUL.md`
- `TOOLS.md`
- `IDENTITY.md`
- `USER.md`
- `HEARTBEAT.md`
- `BOOTSTRAP.md`（仅在全新工作区上）

大型文件会被截断并带有标记。每个文件的最大大小由
`agents.defaults.bootstrapMaxChars`（默认值：20000）控制。缺少的文件注入一个
简短的缺少文件标记。

内部钩子可以通过 `agent:bootstrap` 拦截此步骤，以改变或替换
注入的引导文件（例如，将 `SOUL.md` 交换为备用角色）。

要检查每个注入文件贡献了多少（原始 vs 注入、截断，加上工具模式开销），请使用 `/context list` 或 `/context detail`。请参阅[上下文](/concepts/context)。

## 时间处理

当
用户时区已知时，系统提示包括专用的**当前日期和时间**部分。为了保持提示缓存稳定，它现在仅包括
**时区**（无动态时钟或时间格式）。

当代理需要当前时间时，使用 `session_status`；状态卡
包括时间戳行。

配置：

- `agents.defaults.userTimezone`
- `agents.defaults.timeFormat`（`auto` | `12` | `24`）

有关完整行为详细信息，请参阅[日期和时间](/date-time)。

## 技能

当存在符合条件的技能时，OpenClaw 注入一个紧凑的**可用技能列表**
（`formatSkillsForPrompt`），其中包括每个技能的**文件路径**。该
提示指示模型使用 `read` 加载列出的
位置（工作区、托管或捆绑）的 SKILL.md。如果没有符合条件的技能，则省略
技能部分。

```
<available_skills>
  <skill>
    <name>...</name>
    <description>...</description>
    <location>...</location>
  </skill>
</available_skills>
```

这使基础提示保持小，同时仍启用有针对性的技能使用。

## 文档

可用时，系统提示包括一个**文档**部分，指向
本地 OpenClaw 文档目录（仓库工作区中的 `docs/` 或捆绑的 npm
包文档），还记录了公共镜像、源仓库、社区 Discord 和
ClawdHub (https://clawdhub.com) 用于技能发现。提示指示模型首先查阅本地文档
以了解 OpenClaw 行为、命令、配置或架构，并在可能时运行
`openclaw status` 本身（仅在缺少访问权限时询问用户）。
