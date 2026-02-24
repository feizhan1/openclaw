---
name: goplaces
description: 通过 goplaces CLI 查询 Google Places API (New),用于文本搜索、地点详情、解析和评论。用于人性化的地点查找或脚本的 JSON 输出。
homepage: https://github.com/steipete/goplaces
metadata: {"openclaw":{"emoji":"📍","requires":{"bins":["goplaces"],"env":["GOOGLE_PLACES_API_KEY"]},"primaryEnv":"GOOGLE_PLACES_API_KEY","install":[{"id":"brew","kind":"brew","formula":"steipete/tap/goplaces","bins":["goplaces"],"label":"Install goplaces (brew)"}]}}
---

# goplaces

现代 Google Places API (New) CLI。默认人性化输出,`--json` 用于脚本。

安装
- Homebrew: `brew install steipete/tap/goplaces`

配置
- `GOOGLE_PLACES_API_KEY` 必需。
- 可选: `GOOGLE_PLACES_BASE_URL` 用于测试/代理。

常用命令
- 搜索: `goplaces search "coffee" --open-now --min-rating 4 --limit 5`
- 偏好: `goplaces search "pizza" --lat 40.8 --lng -73.9 --radius-m 3000`
- 分页: `goplaces search "pizza" --page-token "NEXT_PAGE_TOKEN"`
- 解析: `goplaces resolve "Soho, London" --limit 5`
- 详情: `goplaces details <place_id> --reviews`
- JSON: `goplaces search "sushi" --json`

注意事项
- `--no-color` 或 `NO_COLOR` 禁用 ANSI 颜色。
- 价格等级: 0..4(免费 → 非常昂贵)。
- 类型过滤器仅发送第一个 `--type` 值(API 接受一个)。
