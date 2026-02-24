---
name: bear-notes
description: 通过 grizzly CLI 创建、搜索和管理 Bear 笔记。
homepage: https://bear.app
metadata: {"openclaw":{"emoji":"🐻","os":["darwin"],"requires":{"bins":["grizzly"]},"install":[{"id":"go","kind":"go","module":"github.com/tylerwince/grizzly/cmd/grizzly@latest","bins":["grizzly"],"label":"Install grizzly (go)"}]}}
---

# Bear Notes

使用 `grizzly` 在 macOS 上的 Bear 中创建、读取和管理笔记。

要求
- 已安装并运行 Bear 应用
- 某些操作(add-text、tags、open-note --selected)需要 Bear 应用令牌(存储在 `~/.config/grizzly/token` 中)

## 获取 Bear 令牌

对于需要令牌的操作(add-text、tags、open-note --selected),你需要一个认证令牌:
1. 打开 Bear → Help → API Token → Copy Token
2. 保存它: `echo "YOUR_TOKEN" > ~/.config/grizzly/token`

## 常用命令

创建笔记
```bash
echo "Note content here" | grizzly create --title "My Note" --tag work
grizzly create --title "Quick Note" --tag inbox < /dev/null
```

通过 ID 打开/读取笔记
```bash
grizzly open-note --id "NOTE_ID" --enable-callback --json
```

向笔记追加文本
```bash
echo "Additional content" | grizzly add-text --id "NOTE_ID" --mode append --token-file ~/.config/grizzly/token
```

列出所有标签
```bash
grizzly tags --enable-callback --json --token-file ~/.config/grizzly/token
```

搜索笔记(通过 open-tag)
```bash
grizzly open-tag --name "work" --enable-callback --json
```

## 选项

常用标志:
- `--dry-run` — 预览 URL 而不执行
- `--print-url` — 显示 x-callback-url
- `--enable-callback` — 等待 Bear 的响应(读取数据时需要)
- `--json` — 输出为 JSON(使用回调时)
- `--token-file PATH` — Bear API 令牌文件的路径

## 配置

Grizzly 按优先级顺序从以下位置读取配置:
1. CLI 标志
2. 环境变量(`GRIZZLY_TOKEN_FILE`、`GRIZZLY_CALLBACK_URL`、`GRIZZLY_TIMEOUT`)
3. 当前目录中的 `.grizzly.toml`
4. `~/.config/grizzly/config.toml`

示例 `~/.config/grizzly/config.toml`:
```toml
token_file = "~/.config/grizzly/token"
callback_url = "http://127.0.0.1:42123/success"
timeout = "5s"
```

## 注意事项

- 命令运行时 Bear 必须在运行
- 笔记 ID 是 Bear 的内部标识符(在笔记信息中可见或通过回调获取)
- 当需要从 Bear 读取数据时使用 `--enable-callback`
- 某些操作需要有效的令牌(add-text、tags、open-note --selected)
