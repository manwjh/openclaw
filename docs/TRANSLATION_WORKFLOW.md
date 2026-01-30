# 中文翻译工作流程

本文档说明如何保持中文分支 (`zh-cn`) 与上游仓库 (`upstream/main`) 同步，以及如何进行翻译工作。

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

- 中文文档命名规则：`原文件名_cn.md`（例如：`architecture.md` → `architecture_cn.md`）
- 保持原始文档的 YAML front matter 格式
- 保留所有链接（内部链接可能需要更新为中文版本）
- 保留代码块和示例

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

1. **保持同步**：定期同步上游更新，避免分支分叉太远
2. **提交粒度**：每个文档或相关文档组单独提交
3. **提交信息**：使用 `docs: translate xxx to Chinese` 格式
4. **测试链接**：翻译后检查内部链接是否正确
5. **代码不变**：代码块、命令、配置示例保持原样

## 快速同步命令

```bash
# 一键同步脚本
#!/bin/bash
git fetch upstream
git checkout main && git pull upstream main && git push origin main
git checkout zh-cn && git rebase main && git push origin zh-cn
```
