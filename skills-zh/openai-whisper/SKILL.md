---
name: openai-whisper
description: 使用 Whisper CLI 进行本地语音转文本(无需 API 密钥)。
homepage: https://openai.com/research/whisper
metadata: {"openclaw":{"emoji":"🎙️","requires":{"bins":["whisper"]},"install":[{"id":"brew","kind":"brew","formula":"openai-whisper","bins":["whisper"],"label":"Install OpenAI Whisper (brew)"}]}}
---

# Whisper (CLI)

使用 `whisper` 在本地转录音频。

快速开始
- `whisper /path/audio.mp3 --model medium --output_format txt --output_dir .`
- `whisper /path/audio.m4a --task translate --output_format srt`

注意事项
- 首次运行时模型会下载到 `~/.cache/whisper`。
- 此安装中 `--model` 默认为 `turbo`。
- 使用较小的模型以提高速度,较大的模型以提高准确性。
