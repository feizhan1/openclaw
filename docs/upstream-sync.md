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

- 目标：`stable` 分支始终指向“最新正式发布标签”（排除 beta），保证可复现；所有开发分支从 `stable` 派生。
- 适用：不想引入未发布改动，只吃正式版；便于回溯和审计。
- 发布标签定义：`vYYYY.M.D` 系列；如要严格跳过 beta，用过滤命令排除 `beta`。

### 初次建立

1. 拉标签：`git fetch upstream --tags`
2. 取最新正式标签：`latest=$(git tag --sort=-creatordate | rg '^v2026\\.' | rg -v beta | head -n1)`
3. 创建/快进 `stable`：`git branch -f stable "$latest"`
4. 推送到 fork：`git push --force-with-lease origin stable`
5. 记录当前发布哈希（可选）：`git rev-parse "$latest"` 便于审计或防重打标签

### 日常更新（发现新发布时）

1. `git fetch upstream --tags`
2. 获取最新正式标签（同上命令）
3. 快进 `stable`：`git branch -f stable "$latest"`
4. 推送：`git push --force-with-lease origin stable`
5. 验证：`git describe --tags --exact-match stable || git rev-parse stable`

### 开发与升级

- 新开发分支：`git checkout -b feature/x stable`
- 新发布出现后：
  - 更稳妥：基于更新后的 `stable` 重建分支。
  - 或：在原分支上 `git rebase stable`。
- 禁止把 `upstream/main` 直接合入/变基到 `stable`，避免未发布改动进入稳定线。

### 防篡改与回滚（可选）

- 镜像标签到 fork：`git tag fork-$latest $latest && git push --force-with-lease origin fork-$latest`
- 记录哈希到文件（示例）：`git rev-parse "$latest" >> RELEASE_HASHES.md`
- 如需回滚 `stable` 到旧发布：`git branch -f stable <旧标签或哈希> && git push --force-with-lease origin stable`

### 自动化示例（跳过 beta）

```bash
#!/usr/bin/env bash
set -euo pipefail
git fetch upstream --tags
latest=$(git tag --sort=-creatordate | rg '^v2026\\.' | rg -v beta | head -n1)
[ -z "$latest" ] && { echo "no release tag found"; exit 0; }
git branch -f stable "$latest"
git push --force-with-lease origin stable
git describe --tags --exact-match stable || git rev-parse stable
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
