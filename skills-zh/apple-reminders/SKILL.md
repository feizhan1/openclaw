---
name: apple-reminders
description: 在 macOS 上通过 `remindctl` CLI 管理 Apple Reminders(列出、添加、编辑、完成、删除)。支持列表、日期过滤和 JSON/纯文本输出。
homepage: https://github.com/steipete/remindctl
metadata: {"openclaw":{"emoji":"⏰","os":["darwin"],"requires":{"bins":["remindctl"]},"install":[{"id":"brew","kind":"brew","formula":"steipete/tap/remindctl","bins":["remindctl"],"label":"Install remindctl via Homebrew"}]}}
---

# Apple Reminders CLI (remindctl)

使用 `remindctl` 直接从终端管理 Apple Reminders。支持列表过滤、基于日期的视图和脚本输出。

设置
- 安装(Homebrew): `brew install steipete/tap/remindctl`
- 从源码: `pnpm install && pnpm build`(二进制文件在 `./bin/remindctl`)
- 仅限 macOS;提示时授予 Reminders 权限。

权限
- 检查状态: `remindctl status`
- 请求访问: `remindctl authorize`

查看提醒
- 默认(今天): `remindctl`
- 今天: `remindctl today`
- 明天: `remindctl tomorrow`
- 本周: `remindctl week`
- 过期: `remindctl overdue`
- 即将到来: `remindctl upcoming`
- 已完成: `remindctl completed`
- 全部: `remindctl all`
- 特定日期: `remindctl 2026-01-04`

管理列表
- 列出所有列表: `remindctl list`
- 显示列表: `remindctl list Work`
- 创建列表: `remindctl list Projects --create`
- 重命名列表: `remindctl list Work --rename Office`
- 删除列表: `remindctl list Work --delete`

创建提醒
- 快速添加: `remindctl add "Buy milk"`
- 带列表 + 截止日期: `remindctl add --title "Call mom" --list Personal --due tomorrow`

编辑提醒
- 编辑标题/截止日期: `remindctl edit 1 --title "New title" --due 2026-01-04`

完成提醒
- 按 ID 完成: `remindctl complete 1 2 3`

删除提醒
- 按 ID 删除: `remindctl delete 4A83 --force`

输出格式
- JSON(脚本): `remindctl today --json`
- 纯文本 TSV: `remindctl today --plain`
- 仅计数: `remindctl today --quiet`

日期格式
`--due` 和日期过滤器接受的格式:
- `today`、`tomorrow`、`yesterday`
- `YYYY-MM-DD`
- `YYYY-MM-DD HH:mm`
- ISO 8601 (`2026-01-04T12:34:56Z`)

注意事项
- 仅限 macOS。
- 如果访问被拒绝,请在系统设置 → 隐私与安全 → 提醒事项中启用 Terminal/remindctl。
- 如果通过 SSH 运行,请在运行命令的 Mac 上授予访问权限。
