# Inventory 项目 Git 常用命令指南

本文档记录 Inventory 项目开发中常用的 Git 操作命令和最佳实践。

---

## 目录

- [远程仓库管理](#远程仓库管理)
- [分支管理](#分支管理)
- [日常开发流程](#日常开发流程)
- [代码同步](#代码同步)
- [提交规范](#提交规范)
- [高级操作](#高级操作)
- [故障处理](#故障处理)

---

## 远程仓库管理

### 查看远程仓库

```bash
# 查看所有远程仓库（简单）
git remote

# 查看远程仓库详细信息（URL）
git remote -v

# 使用自定义别名查看状态
git remote-status
```

**输出示例：**
```
gitea   https://gitea.inote.live:3232/1203/Inventory (fetch)
gitea   https://gitea.inote.live:3232/1203/Inventory (push)
origin  https://github.com/JiangLongLiu/Inventory.git (fetch)
origin  https://github.com/JiangLongLiu/Inventory.git (push)
upstream        https://github.com/zetavg/Inventory.git (fetch)
upstream        no_push (push)
```

### 添加/删除远程仓库

```bash
# 添加新的远程仓库
git remote add <名称> <URL>

# 删除远程仓库
git remote remove <名称>

# 修改远程仓库 URL
git remote set-url <名称> <新URL>
```

**示例：**
```bash
# 添加备份仓库
git remote add backup https://example.com/repo.git

# 删除备份仓库
git remote remove backup
```

---

## 分支管理

### 查看分支

```bash
# 查看本地分支
git branch

# 查看所有分支（包括远程）
git branch -a

# 查看分支详细信息（显示跟踪关系）
git branch -vv

# 查看已合并的分支
git branch --merged

# 查看未合并的分支
git branch --no-merged
```

### 创建和切换分支

```bash
# 创建新分支
git branch <分支名>

# 切换到指定分支
git checkout <分支名>

# 创建并切换到新分支（推荐）
git checkout -b <分支名>

# 新语法（Git 2.23+）
git switch <分支名>              # 切换分支
git switch -c <分支名>           # 创建并切换
```

### 分支命名规范

```bash
# 功能开发
git checkout -b feature/功能描述
git checkout -b feature/nocobase-integration

# Bug 修复
git checkout -b fix/问题描述
git checkout -b fix/login-error

# 文档更新
git checkout -b docs/文档描述
git checkout -b docs/api-documentation

# 重构代码
git checkout -b refactor/重构描述
git checkout -b refactor/code-cleanup

# 性能优化
git checkout -b perf/优化描述
git checkout -b perf/database-query

# 测试相关
git checkout -b test/测试描述
git checkout -b test/unit-tests
```

### 删除分支

```bash
# 删除本地分支（已合并）
git branch -d <分支名>

# 强制删除本地分支（未合并也删除）
git branch -D <分支名>

# 删除远程分支
git push origin --delete <分支名>
git push gitea --delete <分支名>
```

---

## 日常开发流程

### 1. 开始新功能开发

```bash
# 步骤 1: 确保 main 分支是最新的
git checkout main
git fetch upstream
git merge upstream/main

# 或者使用 pull（fetch + merge）
git pull upstream main

# 步骤 2: 创建功能分支
git checkout -b feature/新功能名称

# 步骤 3: 开发代码...
# （编写代码、测试）

# 步骤 4: 查看修改
git status
git diff

# 步骤 5: 添加文件到暂存区
git add .                    # 添加所有修改
git add <文件路径>           # 添加指定文件
git add *.js                # 添加所有 .js 文件

# 步骤 6: 提交更改
git commit -m "feat: 添加新功能描述"

# 步骤 7: 推送到远程仓库
git push origin feature/新功能名称
git push gitea feature/新功能名称

# 或使用别名同时推送到两个仓库
git push-all feature/新功能名称
```

### 2. 继续在现有分支上开发

```bash
# 切换到开发分支
git checkout feature/现有功能

# 拉取远程更新（如果多人协作）
git pull origin feature/现有功能

# 继续开发...
git add .
git commit -m "feat: 继续完善功能"
git push origin feature/现有功能
```

### 3. 完成功能开发

```bash
# 步骤 1: 确保功能分支代码最新
git checkout feature/功能名称

# 步骤 2: 从 main 合并最新代码（避免冲突）
git checkout main
git pull upstream main
git checkout feature/功能名称
git merge main

# 或使用 rebase（保持提交历史整洁）
git rebase main

# 步骤 3: 解决冲突（如果有）
# 编辑冲突文件...
git add <冲突文件>
git rebase --continue        # 如果使用 rebase
git commit                   # 如果使用 merge

# 步骤 4: 测试验证

# 步骤 5: 合并到 main 分支
git checkout main
git merge feature/功能名称

# 步骤 6: 推送到所有远程仓库
git push origin main
git push gitea main

# 或使用别名
git push-all main

# 步骤 7: 删除功能分支（可选）
git branch -d feature/功能名称
git push origin --delete feature/功能名称
git push gitea --delete feature/功能名称
```

---

## 代码同步

### 从上游同步最新代码

```bash
# 方法 1: 手动步骤
git checkout main
git fetch upstream
git merge upstream/main
git push origin main
git push gitea main

# 方法 2: 使用 pull（fetch + merge）
git checkout main
git pull upstream main
git push origin main
git push gitea main

# 方法 3: 使用自定义别名（推荐）
git checkout main
git sync-upstream
```

### 同步到备份仓库

```bash
# 推送单个分支
git push origin <分支名>
git push gitea <分支名>

# 使用别名同时推送
git push-all <分支名>

# 推送所有分支
git push origin --all
git push gitea --all

# 推送所有分支和标签
git push origin --all --tags
git push gitea --all --tags
```

### 拉取远程分支

```bash
# 查看远程分支
git branch -r

# 拉取并跟踪远程分支
git checkout --track origin/远程分支名

# 或者
git checkout -b 本地分支名 origin/远程分支名

# 示例
git checkout --track origin/feature/new-feature
```

---

## 提交规范

### Commit Message 格式

采用 [Conventional Commits](https://www.conventionalcommits.org/) 规范：

```
<类型>(<范围>): <描述>

[可选的正文]

[可选的脚注]
```

### 常用类型

| 类型 | 说明 | 示例 |
|------|------|------|
| `feat` | 新功能 | `feat: 添加 NocoBase 集成` |
| `fix` | 修复 Bug | `fix: 修复登录失败问题` |
| `docs` | 文档更新 | `docs: 更新 API 文档` |
| `style` | 代码格式（不影响代码运行） | `style: 格式化代码` |
| `refactor` | 重构（不是新增功能也不是修复 Bug） | `refactor: 重构数据层` |
| `perf` | 性能优化 | `perf: 优化数据库查询` |
| `test` | 添加测试 | `test: 添加单元测试` |
| `chore` | 构建过程或辅助工具的变动 | `chore: 更新依赖` |
| `ci` | CI 配置文件和脚本的变动 | `ci: 更新 GitHub Actions` |
| `build` | 影响构建系统或外部依赖 | `build: 升级 webpack` |
| `revert` | 回退之前的提交 | `revert: 回退 commit abc123` |

### 提交示例

```bash
# 添加新功能
git commit -m "feat: 添加 RFID 扫描功能"

# 修复 Bug
git commit -m "fix: 修复资产列表显示错误"

# 更新文档
git commit -m "docs: 添加 NocoBase 集成方案文档"

# 多行提交信息
git commit -m "feat: 添加双向同步功能

- 实现 NocoBase API 客户端
- 添加数据转换逻辑
- 支持增量同步和全量同步

Closes #123"

# 修改上次提交信息
git commit --amend -m "新的提交信息"

# 修改上次提交（添加遗漏的文件）
git add 遗漏的文件
git commit --amend --no-edit
```

### 提交最佳实践

1. **提交要小而频繁**
   - 每个提交只做一件事
   - 便于代码审查和问题追踪

2. **提交前检查**
   ```bash
   # 查看将要提交的内容
   git diff --staged
   
   # 运行测试
   npm test
   yarn test
   ```

3. **提交信息要清晰**
   - 使用现在时态："添加功能" 而不是 "添加了功能"
   - 首行不超过 50 字符
   - 详细描述在空行后
   - 解释 "为什么" 而不只是 "做了什么"

---

## 高级操作

### 暂存修改（Stash）

```bash
# 暂存当前修改
git stash

# 暂存包括未跟踪文件
git stash -u

# 暂存时添加描述
git stash save "WIP: 正在开发的功能"

# 查看暂存列表
git stash list

# 应用最近的暂存
git stash apply

# 应用并删除最近的暂存
git stash pop

# 应用指定的暂存
git stash apply stash@{0}

# 删除暂存
git stash drop stash@{0}

# 清空所有暂存
git stash clear
```

**使用场景：**
```bash
# 场景：正在开发时需要切换分支处理紧急问题
git stash save "WIP: 开发中的功能"
git checkout main
git checkout -b fix/urgent-bug
# 修复 bug...
git commit -m "fix: 修复紧急问题"
git checkout feature/原来的分支
git stash pop
```

### 查看历史

```bash
# 查看提交历史
git log

# 简洁的单行显示
git log --oneline

# 查看最近 N 次提交
git log -n 5
git log --oneline -5

# 查看带分支图的历史
git log --oneline --graph --all

# 查看指定文件的历史
git log -- <文件路径>

# 查看某个作者的提交
git log --author="作者名"

# 查看指定时间范围的提交
git log --since="2024-01-01"
git log --since="2 weeks ago"

# 查看提交详细信息
git show <commit-hash>
git show HEAD
git show HEAD~1    # 上一个提交
```

### 撤销和重置

```bash
# 撤销工作区的修改（未 add）
git checkout -- <文件>
git restore <文件>          # 新语法

# 撤销暂存区的修改（已 add，未 commit）
git reset HEAD <文件>
git restore --staged <文件>  # 新语法

# 撤销最后一次提交（保留修改）
git reset --soft HEAD~1

# 撤销最后一次提交（不保留修改）
git reset --hard HEAD~1

# 撤销指定提交（创建新的反向提交）
git revert <commit-hash>

# 重置到远程分支状态
git fetch origin
git reset --hard origin/main
```

**警告：** `git reset --hard` 会丢失所有未提交的修改，使用前请确认！

### 合并和变基

```bash
# 合并分支（Merge）
git checkout main
git merge feature/分支名

# 创建合并提交（即使可以快进）
git merge --no-ff feature/分支名

# 变基（Rebase）- 使提交历史更整洁
git checkout feature/分支名
git rebase main

# 交互式变基（可以修改、合并、删除提交）
git rebase -i HEAD~3

# 中止变基
git rebase --abort

# 继续变基（解决冲突后）
git add <冲突文件>
git rebase --continue
```

**何时使用 Merge vs Rebase：**
- **Merge**: 保留完整的历史记录，适合多人协作
- **Rebase**: 使历史更整洁线性，适合个人开发分支

### 标签管理

```bash
# 创建轻量标签
git tag v1.0.0

# 创建带注释的标签
git tag -a v1.0.0 -m "版本 1.0.0 发布"

# 查看所有标签
git tag

# 查看标签信息
git show v1.0.0

# 推送标签到远程
git push origin v1.0.0

# 推送所有标签
git push origin --tags

# 删除本地标签
git tag -d v1.0.0

# 删除远程标签
git push origin :refs/tags/v1.0.0
git push origin --delete v1.0.0
```

### Cherry-pick（精选提交）

```bash
# 将指定提交应用到当前分支
git cherry-pick <commit-hash>

# 精选多个提交
git cherry-pick <hash1> <hash2>

# 精选提交范围
git cherry-pick <hash1>..<hash2>

# 中止 cherry-pick
git cherry-pick --abort

# 继续 cherry-pick（解决冲突后）
git add <冲突文件>
git cherry-pick --continue
```

---

## 故障处理

### 问题 1: 提交到了错误的分支

```bash
# 方法 1: 使用 cherry-pick
git checkout 正确的分支
git cherry-pick 错误分支的commit-hash
git checkout 错误的分支
git reset --hard HEAD~1

# 方法 2: 如果还没推送，直接重置
git reset --soft HEAD~1
git stash
git checkout 正确的分支
git stash pop
git add .
git commit -m "提交信息"
```

### 问题 2: 需要修改最近的提交

```bash
# 修改最近一次提交的信息
git commit --amend -m "新的提交信息"

# 添加遗漏的文件到最近一次提交
git add 遗漏的文件
git commit --amend --no-edit

# 如果已经推送，需要强制推送（谨慎使用）
git push origin 分支名 --force-with-lease
```

### 问题 3: 合并冲突

```bash
# 步骤 1: 查看冲突文件
git status

# 步骤 2: 打开冲突文件，查找冲突标记
# <<<<<<< HEAD
# 当前分支的内容
# =======
# 要合并的分支的内容
# >>>>>>> 分支名

# 步骤 3: 编辑文件，解决冲突

# 步骤 4: 标记为已解决
git add <冲突文件>

# 步骤 5: 完成合并
git commit              # 如果是 merge
git rebase --continue   # 如果是 rebase

# 放弃合并
git merge --abort
git rebase --abort
```

### 问题 4: 误删除了分支

```bash
# 查找被删除分支的最后一次提交
git reflog

# 根据 reflog 找到 commit hash，重新创建分支
git branch 分支名 <commit-hash>
```

### 问题 5: 需要撤销已推送的提交

```bash
# 方法 1: 创建反向提交（推荐，不改变历史）
git revert <commit-hash>
git push origin main

# 方法 2: 重置并强制推送（危险，会改变历史）
git reset --hard HEAD~1
git push origin main --force-with-lease
```

**注意：** 强制推送会改变远程历史，可能影响其他协作者！

### 问题 6: 拉取代码时出现冲突

```bash
# 情况 1: 还没开始合并
git pull --rebase origin main

# 情况 2: 已经出现冲突
# 解决冲突后
git add <冲突文件>
git rebase --continue

# 或者放弃 pull
git rebase --abort
```

### 问题 7: 清理不需要的文件

```bash
# 清理未跟踪的文件（查看会删除什么）
git clean -n

# 清理未跟踪的文件
git clean -f

# 清理未跟踪的文件和目录
git clean -fd

# 清理包括 .gitignore 中的文件
git clean -fdx
```

### 问题 8: 查找引入 Bug 的提交

```bash
# 使用 git bisect 二分查找
git bisect start
git bisect bad                 # 标记当前版本为坏版本
git bisect good <commit-hash>  # 标记已知的好版本

# Git 会自动检出中间版本，测试后标记
git bisect good  # 或 git bisect bad

# 重复直到找到问题提交
# 结束 bisect
git bisect reset
```

---

## 配置和优化

### 常用配置

```bash
# 设置用户信息
git config --global user.name "你的名字"
git config --global user.email "your.email@example.com"

# 设置默认编辑器
git config --global core.editor "code --wait"  # VS Code
git config --global core.editor "vim"          # Vim

# 设置默认分支名
git config --global init.defaultBranch main

# 启用颜色输出
git config --global color.ui auto

# 设置合并工具
git config --global merge.tool vscode
git config --global mergetool.vscode.cmd 'code --wait $MERGED'

# 查看所有配置
git config --list

# 查看特定配置
git config user.name
```

### Git 别名（本项目已配置）

```bash
# 查看已配置的别名
git config --get-regexp alias

# 使用别名
git push-all main              # 同时推送到 origin 和 gitea
git sync-upstream              # 从上游同步并推送到备份
git remote-status              # 查看远程仓库和分支状态
```

### 忽略文件

编辑 `.gitignore` 文件：
```
# 依赖目录
node_modules/
/App/node_modules/

# 构建输出
dist/
build/

# 环境变量
.env
.env.local

# IDE 配置
.idea/
.vscode/
*.swp

# 日志文件
*.log

# 操作系统文件
.DS_Store
Thumbs.db
```

---

## 快速参考

### 常用命令速查表

| 操作 | 命令 |
|------|------|
| 查看状态 | `git status` |
| 查看修改 | `git diff` |
| 添加文件 | `git add .` |
| 提交 | `git commit -m "message"` |
| 推送 | `git push origin main` |
| 拉取 | `git pull origin main` |
| 查看历史 | `git log --oneline` |
| 创建分支 | `git checkout -b feature/name` |
| 切换分支 | `git checkout main` |
| 合并分支 | `git merge feature/name` |
| 删除分支 | `git branch -d feature/name` |
| 暂存修改 | `git stash` |
| 恢复暂存 | `git stash pop` |
| 撤销修改 | `git restore <file>` |
| 查看远程 | `git remote -v` |

### 本项目特定命令

```bash
# 查看项目远程仓库和分支状态
git remote-status

# 从上游同步最新代码
git checkout main
git sync-upstream

# 开发新功能
git checkout main
git pull upstream main
git checkout -b feature/功能名称
# 开发...
git add .
git commit -m "feat: 功能描述"
git push-all feature/功能名称

# 推送到所有备份仓库
git push-all main
```

---

## 工作流程图

### 标准开发流程

```
1. 同步上游代码
   git checkout main
   git pull upstream main
   
2. 创建功能分支
   git checkout -b feature/new-feature
   
3. 开发和提交
   git add .
   git commit -m "feat: 添加新功能"
   
4. 推送到备份仓库
   git push-all feature/new-feature
   
5. 合并到 main（功能完成后）
   git checkout main
   git merge feature/new-feature
   
6. 推送 main 到备份
   git push-all main
   
7. 删除功能分支（可选）
   git branch -d feature/new-feature
```

---

## 参考资源

- [Git 官方文档](https://git-scm.com/doc)
- [Pro Git 书籍（中文版）](https://git-scm.com/book/zh/v2)
- [Conventional Commits 规范](https://www.conventionalcommits.org/zh-hans/)
- [GitHub Git 备忘单](https://training.github.com/downloads/zh_CN/github-git-cheat-sheet/)

---

*最后更新：2026-01-07*
*适用版本：Git 2.x+*
