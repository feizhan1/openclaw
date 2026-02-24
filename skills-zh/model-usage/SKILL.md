---
name: model-usage
description: 使用 CodexBar CLI 本地成本使用情况按模型汇总 Codex 或 Claude 的使用情况,包括当前(最近)模型或完整的模型分解。当询问 codexbar 的模型级使用/成本数据时,或当你需要从 codexbar 成本 JSON 获取可脚本化的按模型汇总时触发。
metadata: {"openclaw":{"emoji":"📊","os":["darwin"],"requires":{"bins":["codexbar"]},"install":[{"id":"brew-cask","kind":"brew","cask":"steipete/tap/codexbar","bins":["codexbar"],"label":"Install CodexBar (brew cask)"}]}}
---

# Model usage

## 概述
从 CodexBar 的本地成本日志获取按模型的使用成本。支持 Codex 或 Claude 的"当前模型"(最近的每日条目)或"所有模型"汇总。

TODO: 一旦 CodexBar CLI 的 Linux 安装路径文档化,添加 Linux CLI 支持指导。

## 快速开始
1) 通过 CodexBar CLI 获取成本 JSON 或传递 JSON 文件。
2) 使用捆绑的脚本按模型汇总。

```bash
python {baseDir}/scripts/model_usage.py --provider codex --mode current
python {baseDir}/scripts/model_usage.py --provider codex --mode all
python {baseDir}/scripts/model_usage.py --provider claude --mode all --format json --pretty
```

## 当前模型逻辑
- 使用具有 `modelBreakdowns` 的最近每日行。
- 选择该行中成本最高的模型。
- 当缺少分解时,回退到 `modelsUsed` 中的最后一个条目。
- 当需要特定模型时,使用 `--model <name>` 覆盖。

## 输入
- 默认: 运行 `codexbar cost --format json --provider <codex|claude>`。
- 文件或标准输入:

```bash
codexbar cost --provider codex --format json > /tmp/cost.json
python {baseDir}/scripts/model_usage.py --input /tmp/cost.json --mode all
cat /tmp/cost.json | python {baseDir}/scripts/model_usage.py --input - --mode current
```

## 输出
- 文本(默认)或 JSON(`--format json --pretty`)。
- 值仅为每个模型的成本;CodexBar 输出中未按模型分割令牌。

## 参考
- 阅读 `references/codexbar-cli.md` 了解 CLI 标志和成本 JSON 字段。
