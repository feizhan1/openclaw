---
name: apple-notes
description: 在 macOS 上通过 `memo` CLI 管理 Apple Notes(创建、查看、编辑、删除、搜索、移动和导出笔记)。当用户要求 OpenClaw 添加笔记、列出笔记、搜索笔记或管理笔记文件夹时使用。
homepage: https://github.com/antoniorodr/memo
metadata: {"openclaw":{"emoji":"📝","os":["darwin"],"requires":{"bins":["memo"]},"install":[{"id":"brew","kind":"brew","formula":"antoniorodr/memo/memo","bins":["memo"],"label":"Install memo via Homebrew"}]}}
---

# Apple Notes CLI

使用 `memo notes` 直接从终端管理 Apple Notes。可以创建、查看、编辑、删除、搜索、在文件夹间移动笔记,以及导出为 HTML/Markdown。

设置
- 安装(Homebrew): `brew tap antoniorodr/memo && brew install antoniorodr/memo/memo`
- 手动安装(pip): `pip install .`(克隆仓库后)
- 仅限 macOS;如果提示,请授予 Notes.app 的自动化访问权限。

查看笔记
- 列出所有笔记: `memo notes`
- 按文件夹过滤: `memo notes -f "Folder Name"`
- 搜索笔记(模糊): `memo notes -s "query"`

创建笔记
- 添加新笔记: `memo notes -a`
  - 打开交互式编辑器来撰写笔记。
- 快速添加带标题: `memo notes -a "Note Title"`

编辑笔记
- 编辑现有笔记: `memo notes -e`
  - 交互式选择要编辑的笔记。

删除笔记
- 删除笔记: `memo notes -d`
  - 交互式选择要删除的笔记。

移动笔记
- 移动笔记到文件夹: `memo notes -m`
  - 交互式选择笔记和目标文件夹。

导出笔记
- 导出为 HTML/Markdown: `memo notes -ex`
  - 导出选定的笔记;使用 Mistune 进行 markdown 处理。

限制
- 无法编辑包含图像或附件的笔记。
- 交互式提示可能需要终端访问权限。

注意事项
- 仅限 macOS。
- 需要 Apple Notes.app 可访问。
- 对于自动化,请在系统设置 > 隐私与安全 > 自动化中授予权限。
