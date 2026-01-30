# 中文翻译项目指南

本文档是 OpenClaw 中文翻译项目的总体指南，包括翻译规范、工作流程和质量标准。

## 项目结构

```
docs/
├── i18n/                    # 国际化相关文档
│   ├── README.md           # 本文件
│   ├── glossary.md         # 术语对照表
│   └── style-guide.md      # 翻译风格指南
├── TRANSLATION_WORKFLOW.md # 工作流程（详细）
├── TRANSLATION_CHECKLIST.md # 翻译检查清单
└── [各目录]/
    ├── xxx.md              # 英文原版
    └── xxx_cn.md           # 中文翻译版
```

## 快速开始

1. **阅读本文档** - 了解项目规范
2. **查看术语表** - `docs/i18n/glossary.md`
3. **阅读工作流程** - `docs/TRANSLATION_WORKFLOW.md`
4. **开始翻译** - 选择优先级文档开始

## 翻译原则

### 1. 准确性优先
- 技术术语必须准确
- 保持原意，不添加个人理解
- 不确定时查阅官方文档或询问

### 2. 一致性
- 使用统一的术语表
- 相同概念在不同文档中保持一致
- 保持风格统一

### 3. 可读性
- 符合中文表达习惯
- 避免生硬直译
- 保持专业但易懂

### 4. 完整性
- 不遗漏任何内容
- 保留所有代码示例
- 保持文档结构完整

## 文件命名规范

- 中文文档：`原文件名_cn.md`
- 示例：`architecture.md` → `architecture_cn.md`
- 位置：与英文原版在同一目录

## 文档元数据

每个中文文档应在 YAML front matter 中添加：

```yaml
---
summary: "中文摘要"
read_when:
  - 适用场景
original_path: "docs/concepts/architecture.md"
translated_date: "2026-01-XX"
translator: "manwjh"
sync_status: "同步至上游版本 vX.X.X"
last_sync_date: "2026-01-XX"
---
```

## 链接处理

### 内部链接
- 英文文档链接：保持原样（指向英文版）
- 中文文档链接：如果目标已翻译，指向中文版；否则保持原样
- 示例：`[架构文档](/concepts/architecture_cn)` 或 `[架构文档](/concepts/architecture)`

### 外部链接
- GitHub 链接：保持原样
- 外部文档：保持原样

## 代码和命令

- **代码块**：完全保持原样，不翻译
- **命令示例**：保持原样
- **配置示例**：保持原样
- **注释**：代码中的注释保持原样

## 进度跟踪

查看翻译进度：`docs/i18n/PROGRESS.md`

## 贡献

欢迎贡献翻译！请遵循：
1. Fork 仓库
2. 创建翻译分支
3. 提交翻译
4. 创建 Pull Request

详细流程见 `docs/TRANSLATION_WORKFLOW.md`
