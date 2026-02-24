---
name: nano-pdf
description: 使用 nano-pdf CLI 通过自然语言指令编辑 PDF 文件。
homepage: https://pypi.org/project/nano-pdf/
metadata: {"openclaw":{"emoji":"📄","requires":{"bins":["nano-pdf"]},"install":[{"id":"uv","kind":"uv","package":"nano-pdf","bins":["nano-pdf"],"label":"Install nano-pdf (uv)"}]}}
---

# nano-pdf

使用 `nano-pdf` 通过自然语言指令对 PDF 的特定页面应用编辑。

## 快速开始

```bash
nano-pdf edit deck.pdf 1 "Change the title to 'Q3 Results' and fix the typo in the subtitle"
```

注意事项:
- 页码可能是从 0 开始或从 1 开始,具体取决于工具的版本/配置;如果结果差了一页,请使用另一种方式重试。
- 发送之前务必检查输出的 PDF 文件。
