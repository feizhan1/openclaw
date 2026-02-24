---
name: obsidian
description: 使用 Obsidian vaults(纯 Markdown 笔记)并通过 obsidian-cli 自动化。
homepage: https://help.obsidian.md
metadata: {"openclaw":{"emoji":"💎","requires":{"bins":["obsidian-cli"]},"install":[{"id":"brew","kind":"brew","formula":"yakitrak/yakitrak/obsidian-cli","bins":["obsidian-cli"],"label":"Install obsidian-cli (brew)"}]}}
---

# Obsidian

Obsidian vault = 磁盘上的普通文件夹。

Vault 结构(典型)
- 笔记: `*.md`(纯文本 Markdown;可用任何编辑器编辑)
- 配置: `.obsidian/`(工作区 + 插件设置;通常不要从脚本中触碰)
- 画布: `*.canvas` (JSON)
- 附件: 在 Obsidian 设置中选择的任何文件夹(图像/PDF等)

## 查找活动的 vault

Obsidian 桌面在此处跟踪 vault(真实来源):
- `~/Library/Application Support/obsidian/obsidian.json`

`obsidian-cli` 从该文件解析 vault;vault 名称通常是**文件夹名称**(路径后缀)。

快速"哪个 vault 是活动的 / 笔记在哪里?"
- 如果已设置默认值: `obsidian-cli print-default --path-only`
- 否则,读取 `~/Library/Application Support/obsidian/obsidian.json` 并使用 `"open": true` 的 vault 条目。

注意事项
- 多个 vault 很常见(iCloud vs `~/Documents`、工作/个人等)。不要猜测;读取配置。
- 避免在脚本中硬编码 vault 路径;优先读取配置或使用 `print-default`。

## obsidian-cli 快速开始

选择默认 vault(一次):
- `obsidian-cli set-default "<vault-folder-name>"`
- `obsidian-cli print-default` / `obsidian-cli print-default --path-only`

搜索
- `obsidian-cli search "query"`(笔记名称)
- `obsidian-cli search-content "query"`(笔记内部;显示片段 + 行)

创建
- `obsidian-cli create "Folder/New note" --content "..." --open`
- 需要 Obsidian URI 处理程序(`obsidian://…`)工作(已安装 Obsidian)。
- 避免通过 URI 在"隐藏"点文件夹(例如 `.something/...`)下创建笔记;Obsidian 可能会拒绝。

移动/重命名(安全重构)
- `obsidian-cli move "old/path/note" "new/path/note"`
- 在整个 vault 中更新 `[[wikilinks]]` 和常见的 Markdown 链接(这是相对于 `mv` 的主要优势)。

删除
- `obsidian-cli delete "path/note"`

适当时优先直接编辑:打开 `.md` 文件并更改它;Obsidian 会识别它。
