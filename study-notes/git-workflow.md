# Git 工作流速查表 — Custom_Main 定制开发 + 上游同步

> 仓库：fork 自 `deepseek-ai/deepseek-harness`（上游），自己的仓库为 `DoneJoe/deepseek-harness-hy`（origin）。
> 策略：**Merge 同步** / **纯定制不提 PR** / **功能分支开发**。

## 远程仓库

| 远程名 | 地址 | 用途 |
|---|---|---|
| `origin` | `https://github.com/DoneJoe/deepseek-harness-hy.git` | 自己的 fork，推送定制代码 |
| `upstream` | `https://github.com/deepseek-ai/deepseek-harness.git` | 官方仓库，只读取 |

**认证说明（2026-09-20 配置）**：origin 走 HTTPS，由 `gh` CLI 提供凭据（`gh auth setup-git` 已配置）。
SSH 密钥 `~/.ssh/github_ssh` 有密码保护，非交互环境无法使用；如需 SSH，请先在终端 `ssh-add ~/.ssh/github_ssh`。

## 核心约定

`master` 保持纯净，只作上游镜像，**永不直接提交定制代码**。好处：

- `git log master..Custom_Main` / `git diff master...Custom_Main` 随时精确看出全部定制
- 同步冲突只会发生在 merge 进 `Custom_Main` 这一步；master 更新永远是无冲突快进

## 日常开发（功能分支）

```powershell
git checkout Custom_Main
git pull                                    # 基于最新的定制主线
git checkout -b feature/xxx                 # 切功能分支
# ...开发、git add / git commit...
git checkout Custom_Main
git merge --no-ff feature/xxx               # 合并回定制主线
git push origin Custom_Main
git branch -d feature/xxx                   # 删除已合并的功能分支
```

## 同步上游更新（建议每周，或每次开新功能前）

```powershell
# ① 更新本地 master 为上游最新（--ff-only 保证纯净，失败即报警）
git checkout master
git fetch upstream
git merge --ff-only upstream/master
git push origin master                      # 也可用 GitHub 网页 "Sync fork" 按钮代替

# ② 合并进定制主线
git checkout Custom_Main
git merge master
#   无冲突 → 自动产生 merge commit；有冲突 → 解决后 git add . && git commit
git push origin Custom_Main
```

## 冲突处理

- 冲突基本只出现在你定制过的文件上，逐一解决后提交
- 合并到一半想放弃：`git merge --abort` 安全回到合并前
- 上游大版本更新后，先跑一遍项目构建/测试再 push

## 常用检查命令

```powershell
git branch -vv                              # 确认追踪关系
git log --oneline --graph -10               # 查看合并历史
git diff master...Custom_Main --stat        # 查看全部定制改动（应只有自己的改动）
```
