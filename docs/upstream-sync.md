# 上游同步与版本管理简明指南

## 目标

- 长期跟踪 `upstream/main`，保持本地与个人 fork (`origin`) 同步，减少冲突与历史污染。

## 日常同步 main（建议每次开发前后）

- 确认工作区干净或已提交：`git status`.
- 拉取上游：`git fetch upstream`.
- 变基本地 main：`git rebase upstream/main`.
- 按需快速检查：`pnpm check` 或 `pnpm test`.
- 推送到 fork：`git push --force-with-lease origin main`.

## 功能分支工作流

- 基于最新 main 开分支：`git checkout -b feature/x`.
- 开发时用 `scripts/committer "<msg>" <files...>` 小步提交。
- 准备提交 PR 前：`git fetch upstream && git rebase upstream/main`.
- 解决冲突、复测后推送：`git push --force-with-lease origin feature/x`.
- PR 目标通常指向 `upstream/main`，除非另有要求。

## 冲突处理

- 变基冲突：解决后 `git add <file> && git rebase --continue`.
- 实在搞不定：`git rebase --abort` 回到变基前状态再评估。
- 避免使用 `git stash`（团队约束）；通过提交保留工作。

## 标签与版本

- 更新标签：`git fetch upstream --tags`.
- 查看标签提交：`git rev-list -n 1 v2026.2.22`.
- 基于标签建只读分支：`git branch release/v2026.2.22 v2026.2.22`.

## 只跟踪发布版本的做法

- 始终以标签为基线：`git fetch upstream --tags` 后，检出需要的发布标签，如 `git checkout v2026.2.22`（或 `git switch --detach v2026.2.22`）。
- 如果需要在发布版本上开发，基于标签新建分支：`git checkout -b feature/x v2026.2.22`，后续仅与后续发布标签对齐，不与 `upstream/main` 变基。
- 更新到最新发布时，创建/快进你的稳定分支到最新标签：`git branch -f stable v2026.2.23`，然后在本地用 `git checkout stable` 或让工作分支以新标签为基线重新分支。
- 不混用开发主分支：避免将 `upstream/main` 直接合并到你的发布跟踪分支，减少引入未发布变化。
- 同步到 fork：若需要共享发布跟踪分支，`git push --force-with-lease origin stable`（或对应分支名）。

## 常用速查

- 分支分歧：`git status -sb`.
- 最近提交：`git log --oneline -n 5`.
- 强制更新 fork：`git push --force-with-lease origin main`.
- 变基当前分支到上游：`git fetch upstream && git rebase upstream/main`.

## 注意事项

- 不用 `git reset --hard`、`git pull --rebase --autostash`，防止丢失工作。
- push 时总用 `--force-with-lease`，避免覆盖他人更新。
- 推前检查未跟踪文件，避免私密/大文件误推。**\* End Patch**");
