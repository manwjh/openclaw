---
summary: "OpenClaw 内存的工作原理（工作区文件 + 自动内存刷新）"
read_when:
  - 需要内存文件布局和工作流时
  - 需要调整自动压缩前内存刷新时
original_path: "docs/concepts/memory.md"
translated_date: "2026-01-XX"
translator: "manwjh"
sync_status: "同步至上游版本 vX.X.X"
last_sync_date: "2026-01-XX"
---
# 内存

OpenClaw 内存是**代理工作区中的纯 Markdown**。文件是真相来源；模型只"记住"写入磁盘的内容。

内存搜索工具由活动内存插件提供（默认：`memory-core`）。使用 `plugins.slots.memory = "none"` 禁用内存插件。

## 内存文件（Markdown）

默认工作区布局使用两层内存：

- `memory/YYYY-MM-DD.md`
  - 每日日志（仅追加）。
  - 在会话开始时阅读今天和昨天的内容。
- `MEMORY.md`（可选）
  - 精选的长期内存。
  - **仅在主要的私有会话中加载**（永远不在群组上下文中）。

这些文件位于工作区下（`agents.defaults.workspace`，默认 `~/.openclaw/workspace`）。有关完整布局，请参阅[代理工作区](/concepts/agent-workspace)。

## 何时写入内存

- 决策、偏好和持久化事实写入 `MEMORY.md`。
- 日常笔记和运行上下文写入 `memory/YYYY-MM-DD.md`。
- 如果有人说"记住这个"，请写下来（不要将其保留在 RAM 中）。
- 这个领域仍在发展。提醒模型存储内存会有所帮助；它会知道该做什么。
- 如果您希望某些内容保留，**请要求机器人将其**写入内存。

## 自动内存刷新（压缩前 ping）

当会话**接近自动压缩**时，OpenClaw 触发**静默、
代理轮次**，提醒模型在上下文被压缩**之前**写入持久化内存。默认提示明确说明模型*可以回复*，
但通常 `NO_REPLY` 是正确的响应，因此用户永远看不到此轮次。

这由 `agents.defaults.compaction.memoryFlush` 控制：

```json5
{
  agents: {
    defaults: {
      compaction: {
        reserveTokensFloor: 20000,
        memoryFlush: {
          enabled: true,
          softThresholdTokens: 4000,
          systemPrompt: "Session nearing compaction. Store durable memories now.",
          prompt: "Write any lasting notes to memory/YYYY-MM-DD.md; reply with NO_REPLY if nothing to store."
        }
      }
    }
  }
}
```

详细信息：
- **软阈值**：当会话令牌估计超过 `contextWindow - reserveTokensFloor - softThresholdTokens` 时触发刷新。
- 默认**静默**：提示包括 `NO_REPLY`，因此不会传递任何内容。
- **两个提示**：用户提示加上系统提示附加提醒。
- **每个压缩周期一次刷新**（在 `sessions.json` 中跟踪）。
- **工作区必须可写**：如果会话在 `workspaceAccess: "ro"` 或 `"none"` 的沙箱中运行，则跳过刷新。

有关完整的压缩生命周期，请参阅
[会话管理 + 压缩](/reference/session-management-compaction)。

## 向量内存搜索

OpenClaw 可以在 `MEMORY.md` 和 `memory/*.md`（加上
您选择加入的任何额外目录或文件）上构建小型向量索引，以便语义查询可以找到相关
笔记，即使措辞不同。

默认值：
- 默认启用。
- 监视内存文件的更改（防抖）。
- 默认使用远程嵌入。如果未设置 `memorySearch.provider`，OpenClaw 自动选择：
  1. 如果配置了 `memorySearch.local.modelPath` 且文件存在，则为 `local`。
  2. 如果可以解析 OpenAI 密钥，则为 `openai`。
  3. 如果可以解析 Gemini 密钥，则为 `gemini`。
  4. 否则，内存搜索保持禁用，直到配置。
- 本地模式使用 node-llama-cpp，可能需要 `pnpm approve-builds`。
- 使用 sqlite-vec（可用时）在 SQLite 内加速向量搜索。

远程嵌入**需要**嵌入提供程序的 API 密钥。OpenClaw
从身份验证配置文件、`models.providers.*.apiKey` 或环境
变量解析密钥。Codex OAuth 仅涵盖聊天/完成，**不**满足
内存搜索的嵌入。对于 Gemini，使用 `GEMINI_API_KEY` 或
`models.providers.google.apiKey`。使用自定义 OpenAI 兼容端点时，
设置 `memorySearch.remote.apiKey`（和可选的 `memorySearch.remote.headers`）。

### 额外内存路径

如果您想索引默认工作区布局之外的 Markdown 文件，请添加
显式路径：

```json5
agents: {
  defaults: {
    memorySearch: {
      extraPaths: ["../team-docs", "/srv/shared-notes/overview.md"]
    }
  }
}
```

注意事项：
- 路径可以是绝对路径或相对于工作区。
- 递归扫描目录以查找 `.md` 文件。
- 仅索引 Markdown 文件。
- 忽略符号链接（文件或目录）。

### Gemini 嵌入（原生）

将提供程序设置为 `gemini` 以直接使用 Gemini 嵌入 API：

```json5
agents: {
  defaults: {
    memorySearch: {
      provider: "gemini",
      model: "gemini-embedding-001",
      remote: {
        apiKey: "YOUR_GEMINI_API_KEY"
      }
    }
  }
}
```

注意事项：
- `remote.baseUrl` 是可选的（默认为 Gemini API 基础 URL）。
- `remote.headers` 允许您在需要时添加额外的标头。
- 默认模型：`gemini-embedding-001`。

如果您想使用**自定义 OpenAI 兼容端点**（OpenRouter、vLLM 或代理），
您可以使用带有 OpenAI 提供程序的 `remote` 配置：

```json5
agents: {
  defaults: {
    memorySearch: {
      provider: "openai",
      model: "text-embedding-3-small",
      remote: {
        baseUrl: "https://api.example.com/v1/",
        apiKey: "YOUR_OPENAI_COMPAT_API_KEY",
        headers: { "X-Custom-Header": "value" }
      }
    }
  }
}
```

如果您不想设置 API 密钥，请使用 `memorySearch.provider = "local"` 或设置
`memorySearch.fallback = "none"`。

回退：
- `memorySearch.fallback` 可以是 `openai`、`gemini`、`local` 或 `none`。
- 回退提供程序仅在主要嵌入提供程序失败时使用。

批量索引（OpenAI + Gemini）：
- 默认启用 OpenAI 和 Gemini 嵌入。设置 `agents.defaults.memorySearch.remote.batch.enabled = false` 以禁用。
- 默认行为等待批量完成；如果需要，调整 `remote.batch.wait`、`remote.batch.pollIntervalMs` 和 `remote.batch.timeoutMinutes`。
- 设置 `remote.batch.concurrency` 以控制我们并行提交的批量作业数量（默认值：2）。
- 当 `memorySearch.provider = "openai"` 或 `"gemini"` 并使用相应的 API 密钥时，批量模式适用。
- Gemini 批量作业使用异步嵌入批量端点，需要 Gemini Batch API 可用性。

为什么 OpenAI 批量快速且便宜：
- 对于大型回填，OpenAI 通常是我们支持的最快选项，因为我们可以在一个批量作业中提交许多嵌入请求，并让 OpenAI 异步处理它们。
- OpenAI 为 Batch API 工作负载提供折扣定价，因此大型索引运行通常比同步发送相同请求更便宜。
- 有关详细信息，请参阅 OpenAI Batch API 文档和定价：
  - https://platform.openai.com/docs/api-reference/batch
  - https://platform.openai.com/pricing

配置示例：

```json5
agents: {
  defaults: {
    memorySearch: {
      provider: "openai",
      model: "text-embedding-3-small",
      fallback: "openai",
      remote: {
        batch: { enabled: true, concurrency: 2 }
      },
      sync: { watch: true }
    }
  }
}
```

工具：
- `memory_search` — 返回带有文件 + 行范围的片段。
- `memory_get` — 按路径读取内存文件内容。

本地模式：
- 设置 `agents.defaults.memorySearch.provider = "local"`。
- 提供 `agents.defaults.memorySearch.local.modelPath`（GGUF 或 `hf:` URI）。
- 可选：设置 `agents.defaults.memorySearch.fallback = "none"` 以避免远程回退。

### 内存工具的工作原理

- `memory_search` 语义搜索来自 `MEMORY.md` + `memory/**/*.md` 的 Markdown 块（~400 令牌目标，80 令牌重叠）。它返回片段文本（上限 ~700 字符）、文件路径、行范围、分数、提供程序/模型，以及我们是否从本地 → 远程嵌入回退。不返回完整文件有效负载。
- `memory_get` 读取特定的内存 Markdown 文件（相对于工作区），可选地从起始行开始读取 N 行。仅当在 `memorySearch.extraPaths` 中明确列出时，才允许 `MEMORY.md` / `memory/` 之外的路径。
- 仅当 `memorySearch.enabled` 为代理解析为 true 时，这两个工具才启用。

### 索引的内容（以及何时）

- 文件类型：仅 Markdown（`MEMORY.md`、`memory/**/*.md`，加上 `memorySearch.extraPaths` 下的任何 `.md` 文件）。
- 索引存储：每个代理的 SQLite 位于 `~/.openclaw/memory/<agentId>.sqlite`（可通过 `agents.defaults.memorySearch.store.path` 配置，支持 `{agentId}` 令牌）。
- 新鲜度：`MEMORY.md`、`memory/` 和 `memorySearch.extraPaths` 上的监视器将索引标记为脏（防抖 1.5 秒）。同步在会话开始时、搜索时或按间隔安排，并异步运行。会话转录使用增量阈值触发后台同步。
- 重新索引触发器：索引存储嵌入**提供程序/模型 + 端点指纹 + 分块参数**。如果其中任何一项更改，OpenClaw 会自动重置并重新索引整个存储。

### 混合搜索（BM25 + 向量）

启用后，OpenClaw 结合：
- **向量相似性**（语义匹配，措辞可以不同）
- **BM25 关键字相关性**（精确令牌，如 ID、环境变量、代码符号）

如果您的平台上无法使用全文搜索，OpenClaw 会回退到仅向量搜索。

#### 为什么混合？

向量搜索在"这意味着相同的事情"方面表现出色：
- "Mac Studio 网关主机" vs "运行网关的机器"
- "防抖文件更新" vs "避免每次写入时索引"

但它在精确、高信号令牌方面可能较弱：
- ID（`a828e60`、`b3b9895a…`）
- 代码符号（`memorySearch.query.hybrid`）
- 错误字符串（"sqlite-vec 不可用"）

BM25（全文）相反：在精确令牌方面强，在释义方面弱。
混合搜索是实用的中间立场：**使用两种检索信号**，以便您获得
"自然语言"查询和"大海捞针"查询的良好结果。

#### 我们如何合并结果（当前设计）

实现草图：

1) 从双方检索候选池：
- **向量**：按余弦相似性排序的前 `maxResults * candidateMultiplier`。
- **BM25**：按 FTS5 BM25 排名排序的前 `maxResults * candidateMultiplier`（越低越好）。

2) 将 BM25 排名转换为 0..1 左右的分数：
- `textScore = 1 / (1 + max(0, bm25Rank))`

3) 按块 ID 合并候选并计算加权分数：
- `finalScore = vectorWeight * vectorScore + textWeight * textScore`

注意事项：
- `vectorWeight` + `textWeight` 在配置解析中归一化为 1.0，因此权重表现为百分比。
- 如果嵌入不可用（或提供程序返回零向量），我们仍运行 BM25 并返回关键字匹配。
- 如果无法创建 FTS5，我们保持仅向量搜索（无硬失败）。

这不是"IR 理论完美"，但它简单、快速，并且倾向于改善真实笔记的召回率/精确度。
如果我们以后想要更高级，常见的下一步是 Reciprocal Rank Fusion (RRF) 或在混合之前进行分数归一化
（最小/最大或 z 分数）。

配置：

```json5
agents: {
  defaults: {
    memorySearch: {
      query: {
        hybrid: {
          enabled: true,
          vectorWeight: 0.7,
          textWeight: 0.3,
          candidateMultiplier: 4
        }
      }
    }
  }
}
```

### 嵌入缓存

OpenClaw 可以在 SQLite 中缓存**块嵌入**，以便重新索引和频繁更新（尤其是会话转录）不会重新嵌入未更改的文本。

配置：

```json5
agents: {
  defaults: {
    memorySearch: {
      cache: {
        enabled: true,
        maxEntries: 50000
      }
    }
  }
}
```

### 会话内存搜索（实验性）

您可以选择索引**会话转录**并通过 `memory_search` 显示它们。
这由实验标志控制。

```json5
agents: {
  defaults: {
    memorySearch: {
      experimental: { sessionMemory: true },
      sources: ["memory", "sessions"]
    }
  }
}
```

注意事项：
- 会话索引是**选择加入**（默认关闭）。
- 会话更新会防抖并**异步索引**，一旦它们超过增量阈值（尽力而为）。
- `memory_search` 永远不会阻塞索引；结果可能稍微过时，直到后台同步完成。
- 结果仍仅包括片段；`memory_get` 仍然仅限于内存文件。
- 会话索引按代理隔离（仅索引该代理的会话日志）。
- 会话日志位于磁盘上（`~/.openclaw/agents/<agentId>/sessions/*.jsonl`）。任何具有文件系统访问权限的进程/用户都可以读取它们，因此将磁盘访问视为信任边界。为了更严格的隔离，请在单独的 OS 用户或主机下运行代理。

增量阈值（显示默认值）：

```json5
agents: {
  defaults: {
    memorySearch: {
      sync: {
        sessions: {
          deltaBytes: 100000,   // ~100 KB
          deltaMessages: 50     // JSONL 行
        }
      }
    }
  }
}
```

### SQLite 向量加速（sqlite-vec）

当 sqlite-vec 扩展可用时，OpenClaw 将嵌入存储在
SQLite 虚拟表（`vec0`）中，并在
数据库中执行向量距离查询。这使搜索保持快速，而无需将每个嵌入加载到 JS 中。

配置（可选）：

```json5
agents: {
  defaults: {
    memorySearch: {
      store: {
        vector: {
          enabled: true,
          extensionPath: "/path/to/sqlite-vec"
        }
      }
    }
  }
}
```

注意事项：
- `enabled` 默认为 true；禁用时，搜索回退到存储嵌入的进程内
  余弦相似性。
- 如果缺少 sqlite-vec 扩展或加载失败，OpenClaw 记录
  错误并继续使用 JS 回退（无向量表）。
- `extensionPath` 覆盖捆绑的 sqlite-vec 路径（对于自定义构建
  或非标准安装位置很有用）。

### 本地嵌入自动下载

- 默认本地嵌入模型：`hf:ggml-org/embeddinggemma-300M-GGUF/embeddinggemma-300M-Q8_0.gguf`（~0.6 GB）。
- 当 `memorySearch.provider = "local"` 时，`node-llama-cpp` 解析 `modelPath`；如果缺少 GGUF，它会**自动下载**到缓存（或如果设置了 `local.modelCacheDir`），然后加载它。下载在重试时恢复。
- 原生构建要求：运行 `pnpm approve-builds`，选择 `node-llama-cpp`，然后 `pnpm rebuild node-llama-cpp`。
- 回退：如果本地设置失败且 `memorySearch.fallback = "openai"`，我们自动切换到远程嵌入（`openai/text-embedding-3-small`，除非覆盖）并记录原因。

### 自定义 OpenAI 兼容端点示例

```json5
agents: {
  defaults: {
    memorySearch: {
      provider: "openai",
      model: "text-embedding-3-small",
      remote: {
        baseUrl: "https://api.example.com/v1/",
        apiKey: "YOUR_REMOTE_API_KEY",
        headers: {
          "X-Organization": "org-id",
          "X-Project": "project-id"
        }
      }
    }
  }
}
```

注意事项：
- `remote.*` 优先于 `models.providers.openai.*`。
- `remote.headers` 与 OpenAI 标头合并；远程在键冲突时获胜。省略 `remote.headers` 以使用 OpenAI 默认值。
