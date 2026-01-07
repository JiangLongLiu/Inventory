# Gitea Token 说明

## 文件
- `gitea.token` - Gitea 仓库访问令牌

## 使用方法

配置 Git 远程仓库时使用：
```bash
token=$(cat tokens/gitea.token)
git remote set-url gitea https://liujianglong:${token}@gitea.inote.live:3232/1203/Inventory
```

## 安全说明
- 此目录下的所有文件都已添加到 .gitignore
- 请勿将令牌文件提交到版本控制系统
- 如需备份，请使用加密存储
