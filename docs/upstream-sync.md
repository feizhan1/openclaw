# 上游同步与版本管理简明指南

## 目标

- 方便二次开发，同时可选保持开发版（upstream/main）或发布版（标签）的一致性。

## 日常同步 main（追踪开发版）

- 确认工作区干净或已提交：`git status`.
- 拉取上游：`git fetch upstream`.
- 变基本地 main：`git rebase upstream/main`.
- 按需检查：`pnpm check` 或 `pnpm test`.
- 推送到 fork：`git push --force-with-lease origin main`.

## 功能分支工作流

- 基于最新 main 开分支：`git checkout -b feature/x main`.
- 开发时用 `scripts/committer "<msg>" <files...>` 小步提交。
- 定期对齐上游：在分支上执行 `git fetch upstream && git rebase upstream/main`.
- 推送备份/协作：`git push --force-with-lease origin feature/x`.

## 只跟踪发布版本的做法

- 始终以标签为基线：`git fetch upstream --tags` 后检出发布标签，如 `git checkout v2026.2.22`（或 `git switch --detach v2026.2.22`）。
- 在发布版上开发：`git checkout -b feature/x v2026.2.22`，后续只对齐新发布标签，不与 `upstream/main` 变基。
- 更新到最新发布：`git branch -f stable v2026.2.xx && git push --force-with-lease origin stable`，工作分支以更新后的 `stable` 为基线重新切分支或变基。
- 不混用开发主分支：避免把 `upstream/main` 合入发布跟踪分支。

## 稳定发布长期跟踪（推荐稳定性）

- 目标：`stable` 分支始终指向最新发布标签，保证可复现；所有开发分支从 `stable` 派生。
- 初次建立：
  - `git fetch upstream --tags`
  - `git branch -f stable <最新发布标签>`
  - `git push --force-with-lease origin stable`
- 日常更新（上游有新发布时）：
  - `git fetch upstream --tags`
  - 确认最新标签，如 `latest=$(git tag --sort=-creatordate | rg '^v2026\\.' | head -n1)`
  - 快进 `stable`：`git branch -f stable "$latest"`
  - 推送：`git push --force-with-lease origin stable`
- 开发基线：`git checkout -b feature/x stable`；跟进新发布时，切新分支或将现有分支变基到更新后的 `stable`。
- 防篡改（可选）：在 fork 上再打镜像标签（如 `fork-v2026.2.23`）指向同一 commit，防上游重打标签。
- 自动化示例（本地脚本，定时执行）：
  ```bash
  #!/usr/bin/env bash
  set -euo pipefail
  git fetch upstream --tags
  latest=$(git tag --sort=-creatordate | rg '^v2026\\.' | head -n1)
  [ -z "$latest" ] && { echo "no tag found"; exit 1; }
  git branch -f stable "$latest"
  git push --force-with-lease origin stable
  ```

## 常用速查

- 分支分歧：`git status -sb`.
- 最近提交：`git log --oneline -n 5`.
- 强制更新 fork：`git push --force-with-lease origin main`.
- 变基当前分支到上游：`git fetch upstream && git rebase upstream/main`.

## 注意事项

- 不用 `git reset --hard`、`git pull --rebase --autostash`，防止丢失工作。
- push 一律用 `--force-with-lease`，避免覆盖他人更新。
- 推前检查未跟踪文件，避免私密/大文件误推。
