# Git 远程仓库三段式管理方案

## 方案概述

本方案设置三个远程仓库，各司其职：

1. **upstream（上游）** - 原作者仓库（主源）
   - URL: `https://github.com/zetavg/Inventory.git`
   - 用途：拉取最新的 main 分支代码
   - 只读：仅用于 fetch 和 pull

2. **origin（主仓库）** - 个人 GitHub 仓库（备份 1）
   - URL: `https://github.com/JiangLongLiu/Inventory.git`
   - 用途：个人开发代码推送、备份
   - 可读可写：既可以 fetch 也可以 push

3. **gitea（备份仓库）** - Gitea 私有仓库（备份 2）
   - URL: `https://gitea.inote.live:3232/1203/Inventory`
   - 用途：私有备份、团队协作（可选）
   - 可读可写：既可以 fetch 也可以 push

## 配置步骤

### 1. 查看当前远程仓库

```bash
git remote -v
```

### 2. 重命名当前 origin 为 backup（如果需要保留）

```bash
# 当前 origin 已经是 JiangLongLiu/Inventory，保持不变
# 如果需要重命名：git remote rename origin backup
```

### 3. 添加原作者仓库为 upstream

```bash
git remote add upstream https://github.com/zetavg/Inventory.git
```

### 4. 添加 Gitea 仓库为 gitea

```bash
git remote add gitea https://gitea.inote.live:3232/1203/Inventory
```

### 5. 验证配置

```bash
git remote -v
```

预期输出：
```
gitea   https://gitea.inote.live:3232/1203/Inventory (fetch)
gitea   https://gitea.inote.live:3232/1203/Inventory (push)
origin  https://github.com/JiangLongLiu/Inventory.git (fetch)
origin  https://github.com/JiangLongLiu/Inventory.git (push)
upstream        https://github.com/zetavg/Inventory.git (fetch)
upstream        https://github.com/zetavg/Inventory.git (push)
```

### 6. 配置 upstream 为只读（可选但推荐）

```bash
git remote set-url --push upstream no_push
```

这样可以防止误推送到上游仓库。

## 日常工作流程

### 从上游更新代码

```bash
# 1. 确保在 main 分支
git checkout main

# 2. 从上游拉取最新代码
git fetch upstream

# 3. 合并上游的 main 分支
git merge upstream/main

# 或者使用 rebase（保持提交历史整洁）
git rebase upstream/main
```

### 推送到备份仓库

```bash
# 推送到 GitHub（origin）
git push origin main

# 推送到 Gitea（gitea）
git push gitea main

# 同时推送到两个备份仓库
git push origin main && git push gitea main
```

### 开发新功能

```bash
# 1. 从最新的 main 创建功能分支
git checkout main
git pull upstream main
git checkout -b feature/new-feature

# 2. 开发并提交
git add .
git commit -m "feat: 添加新功能"

# 3. 推送到自己的仓库（不推送到 upstream）
git push origin feature/new-feature
git push gitea feature/new-feature
```

### 同步所有远程仓库

```bash
# 方法 1: 手动逐个推送
git push origin --all
git push gitea --all

# 方法 2: 使用别名（见下方配置）
git push-all
```

## Git 配置优化

### 添加 Git 别名

编辑 `.git/config` 或使用命令添加：

```bash
# 同时推送到所有备份仓库
git config alias.push-all '!git push origin "$@" && git push gitea "$@"'

# 从上游更新并推送到备份
git config alias.sync-upstream '!git fetch upstream && git merge upstream/main && git push origin main && git push gitea main'

# 查看所有远程仓库状态
git config alias.remote-status '!git remote -v && echo "" && git branch -vv'
```

使用示例：
```bash
git push-all main
git sync-upstream
git remote-status
```

## 分支管理策略

### main 分支
- 保持与 upstream/main 同步
- 只合并经过测试的代码
- 定期推送到 origin 和 gitea 备份

### 功能分支
- 命名规范：`feature/功能名称`
- 从最新的 main 创建
- 推送到 origin 和 gitea，不推送到 upstream

### 示例分支
- 当前分支：`feature/nocobase-integration-analysis`
- 推送命令：`git push origin feature/nocobase-integration-analysis`

## 冲突处理

### 如果本地有未提交的更改

```bash
# 方法 1: 暂存更改
git stash
git pull upstream main
git stash pop

# 方法 2: 提交更改
git add .
git commit -m "WIP: 临时保存"
git pull upstream main
```

### 如果合并产生冲突

```bash
# 1. 查看冲突文件
git status

# 2. 编辑冲突文件，解决冲突

# 3. 标记为已解决
git add <冲突文件>

# 4. 完成合并
git commit
```

## 安全注意事项

1. **不要直接推送到 upstream**
   - 已配置为 no_push
   - 如需贡献代码，应通过 Pull Request

2. **敏感信息保护**
   - 不要提交密码、密钥等敏感信息
   - 使用 `.gitignore` 排除配置文件

3. **Gitea 仓库访问**
   - 确保 Gitea 仓库已正确创建
   - 配置 SSH 密钥或使用 HTTPS 认证
   - 如果是私有仓库，确保有访问权限

## 实际执行状态

### 当前远程仓库配置

执行时间：2026-01-07

**执行前状态：**
```
origin  https://github.com/JiangLongLiu/Inventory.git (fetch)
origin  https://github.com/JiangLongLiu/Inventory.git (push)
```

**执行步骤：**

1. ✅ 添加 upstream 远程仓库
2. ✅ 添加 gitea 远程仓库
3. ✅ 配置 upstream 为只读
4. ✅ 验证配置

**执行后状态：**
```
gitea   https://gitea.inote.live:3232/1203/Inventory (fetch)
gitea   https://gitea.inote.live:3232/1203/Inventory (push)
origin  https://github.com/JiangLongLiu/Inventory.git (fetch)
origin  https://github.com/JiangLongLiu/Inventory.git (push)
upstream        https://github.com/zetavg/Inventory.git (fetch)
upstream        no_push (push)
```

**配置的 Git 别名：**
- `git push-all`: 同时推送到 origin 和 gitea
- `git sync-upstream`: 从上游同步并推送到备份仓库
- `git remote-status`: 查看所有远程仓库和分支状态

**验证测试：**
```bash
# 查看远程和分支状态
$ git remote-status
✅ 成功显示所有远程仓库和分支信息
```

## 故障排除

### 问题 1: 无法连接到 Gitea 仓库

```bash
# 测试连接
git ls-remote https://gitea.inote.live:3232/1203/Inventory

# 如果失败，检查：
# 1. Gitea 服务是否运行
# 2. 端口 3232 是否开放
# 3. 仓库 URL 是否正确
# 4. 是否有访问权限
```

### 问题 2: 推送被拒绝

```bash
# 如果远程有更新的提交
git pull origin main --rebase
git push origin main

# 如果需要强制推送（谨慎使用）
git push origin main --force-with-lease
```

### 问题 3: upstream 推送失败

这是预期行为（已配置为 no_push），说明配置正确。

## 参考命令速查

```bash
# 查看远程仓库
git remote -v

# 查看分支状态
git branch -vv

# 从上游拉取
git fetch upstream

# 推送到备份
git push origin <branch>
git push gitea <branch>

# 删除远程仓库
git remote remove <name>

# 修改远程仓库 URL
git remote set-url <name> <new-url>
```

---

*最后更新：2026-01-07*
