# 中文翻译工作流程

本文档说明如何保持中文分支 (`zh-cn`) 与上游仓库 (`upstream/main`) 同步，以及如何进行翻译工作。

> 📖 **相关文档**：
> - [国际化指南](i18n/README.md) - 总体规范和原则
> - [术语对照表](i18n/glossary.md) - 标准术语翻译
> - [翻译检查清单](TRANSLATION_CHECKLIST.md) - 提交前检查

## 分支结构

- `main`: 你的主分支，与上游保持同步
- `zh-cn`: 中文翻译分支，基于 `main` 分支

## 同步上游更新

### 1. 获取上游最新更改

```bash
# 获取上游所有更新
git fetch upstream

# 切换到主分支并同步
git checkout main
git pull upstream main

# 推送你的主分支（如果需要）
git push origin main
```

### 2. 将更新合并到中文分支

```bash
# 切换到中文分支
git checkout zh-cn

# 使用 rebase 保持历史干净（推荐）
git rebase main

# 或者使用 merge（如果 rebase 有冲突）
# git merge main

# 推送更新后的中文分支
git push origin zh-cn
```

### 3. 处理冲突

如果 rebase 时出现冲突：

```bash
# 查看冲突文件
git status

# 手动解决冲突后
git add <解决冲突的文件>
git rebase --continue

# 如果放弃 rebase
git rebase --abort
```

## 翻译工作流程

### 1. 开始翻译前

```bash
# 确保在中文分支上
git checkout zh-cn

# 确保是最新的
git pull origin zh-cn
git rebase main  # 如果有新的上游更新
```

### 2. 翻译文档

**准备工作**：
- 阅读 [术语对照表](i18n/glossary.md) 确保术语一致
- 阅读 [翻译检查清单](TRANSLATION_CHECKLIST.md) 了解要求
- 查看英文原版的最后更新时间

**翻译规范**：
- 中文文档命名规则：`原文件名_cn.md`（例如：`architecture.md` → `architecture_cn.md`）
- 保持原始文档的 YAML front matter 格式，并添加翻译元数据：
  ```yaml
  ---
  summary: "中文摘要"
  read_when:
    - 适用场景
  original_path: "docs/concepts/architecture.md"
  translated_date: "2026-01-XX"
  translator: "your-username"
  sync_status: "同步至上游版本 vX.X.X"
  last_sync_date: "2026-01-XX"
  ---
  ```
- 保留所有链接（内部链接：已翻译的指向中文版，未翻译的指向英文版）
- 保留代码块、命令、配置示例（完全保持原样）
- 使用术语表中的标准翻译

### 3. 提交翻译

```bash
# 添加翻译文件
git add docs/xxx/xxx_cn.md

# 提交（使用清晰的提交信息）
git commit -m "docs: translate xxx to Chinese"

# 推送到远程
git push origin zh-cn
```

## 推荐的翻译优先级

### 第一优先级（核心文档）
1. `docs/index.md` - 主入口文档
2. `docs/start/getting-started.md` - 入门指南
3. `docs/concepts/architecture.md` - 架构说明
4. `docs/gateway/configuration.md` - 配置文档

### 第二优先级（重要功能）
5. `docs/start/wizard.md` - 向导设置
6. `docs/gateway/` 目录下的其他重要文档
7. `docs/channels/` 目录下的主要通道文档

### 第三优先级（参考文档）
8. 其他概念文档 (`docs/concepts/`)
9. 平台文档 (`docs/platforms/`)
10. 工具文档 (`docs/tools/`)

## 注意事项

1. **保持同步**：定期同步上游更新，避免分支分叉太远（建议每周至少一次）
2. **提交粒度**：每个文档或相关文档组单独提交
3. **提交信息**：使用 `docs: translate xxx to Chinese` 格式
4. **质量检查**：提交前使用 [翻译检查清单](TRANSLATION_CHECKLIST.md) 检查
5. **术语一致**：使用 [术语对照表](i18n/glossary.md) 中的标准翻译
6. **代码不变**：代码块、命令、配置示例保持原样
7. **链接处理**：已翻译的文档链接指向中文版，未翻译的保持原样

## 版本同步标记

当上游文档更新时，在中文文档的元数据中更新：
- `sync_status`: 当前同步的上游版本（如 `v2026.1.15`）
- `last_sync_date`: 最后同步日期

如果上游文档有重大更新，需要重新翻译或更新翻译。

## 快速同步命令

```bash
# 一键同步脚本
#!/bin/bash
set -e
echo "Fetching upstream..."
git fetch upstream

echo "Syncing main branch..."
git checkout main
git pull upstream main
git push origin main || true

echo "Syncing zh-cn branch..."
git checkout zh-cn
git rebase main
git push origin zh-cn || echo "Push failed, may need force push: git push origin zh-cn --force-with-lease"

echo "Sync complete!"
```

## 进度跟踪

建议维护一个进度跟踪文件 `docs/i18n/PROGRESS.md`，记录：
- 已翻译的文档列表
- 翻译状态（完成/进行中/待翻译）
- 最后更新时间
- 需要更新的文档（上游有更新）

## 质量保证

1. **自检**：使用检查清单
2. **术语检查**：对照术语表
3. **格式检查**：Markdown 格式正确
4. **链接检查**：所有链接有效
5. **同步检查**：与上游版本一致

## 贡献协作

如果有其他贡献者：
1. 在 Issue 中讨论翻译计划
2. 避免多人同时翻译同一文档
3. 提交 PR 时说明翻译范围和检查情况
4. 欢迎审查和反馈
