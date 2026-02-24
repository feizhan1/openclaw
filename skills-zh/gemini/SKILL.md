---
name: gemini
description: 用于一次性问答、摘要和生成的 Gemini CLI。
homepage: https://ai.google.dev/
metadata: {"openclaw":{"emoji":"♊️","requires":{"bins":["gemini"]},"install":[{"id":"brew","kind":"brew","formula":"gemini-cli","bins":["gemini"],"label":"Install Gemini CLI (brew)"}]}}
---

# Gemini CLI

在一次性模式下使用 Gemini,通过位置提示(避免交互模式)。

快速开始
- `gemini "Answer this question..."`
- `gemini --model <name> "Prompt..."`
- `gemini --output-format json "Return JSON"`

扩展
- 列出: `gemini --list-extensions`
- 管理: `gemini extensions <command>`

注意事项
- 如果需要认证,以交互方式运行一次 `gemini` 并按照登录流程操作。
- 为安全起见避免使用 `--yolo`。
