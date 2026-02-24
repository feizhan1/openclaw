---
name: summarize
description: 从 URL、播客和本地文件中总结或提取文本/转录(对于"转录此 YouTube/视频"是很好的备用方案)。
homepage: https://summarize.sh
metadata: {"openclaw":{"emoji":"🧾","requires":{"bins":["summarize"]},"install":[{"id":"brew","kind":"brew","formula":"steipete/tap/summarize","bins":["summarize"],"label":"Install summarize (brew)"}]}}
---

# Summarize

快速 CLI,用于总结 URL、本地文件和 YouTube 链接。

## 何时使用(触发短语)

当用户询问以下任何内容时立即使用此技能:
- "使用 summarize.sh"
- "这个链接/视频是关于什么的?"
- "总结这个 URL/文章"
- "转录此 YouTube/视频"(尽力而为的转录提取;不需要 `yt-dlp`)

## 快速开始

```bash
summarize "https://example.com" --model google/gemini-3-flash-preview
summarize "/path/to/file.pdf" --model google/gemini-3-flash-preview
summarize "https://youtu.be/dQw4w9WgXcQ" --youtube auto
```

## YouTube: 摘要 vs 转录

尽力而为的转录(仅限 URL):

```bash
summarize "https://youtu.be/dQw4w9WgXcQ" --youtube auto --extract-only
```

如果用户要求转录但内容很长,先返回简洁的摘要,然后询问要展开哪个部分/时间范围。

## 模型 + 密钥

为您选择的提供商设置 API 密钥:
- OpenAI: `OPENAI_API_KEY`
- Anthropic: `ANTHROPIC_API_KEY`
- xAI: `XAI_API_KEY`
- Google: `GEMINI_API_KEY`(别名: `GOOGLE_GENERATIVE_AI_API_KEY`、`GOOGLE_API_KEY`)

如果未设置,默认模型是 `google/gemini-3-flash-preview`。

## 实用标志

- `--length short|medium|long|xl|xxl|<chars>`
- `--max-output-tokens <count>`
- `--extract-only`(仅限 URL)
- `--json`(机器可读)
- `--firecrawl auto|off|always`(备用提取)
- `--youtube auto`(如果设置了 `APIFY_API_TOKEN`,则使用 Apify 备用)

## 配置

可选配置文件: `~/.summarize/config.json`

```json
{ "model": "openai/gpt-5.2" }
```

可选服务:
- `FIRECRAWL_API_KEY` 用于被阻止的网站
- `APIFY_API_TOKEN` 用于 YouTube 备用
