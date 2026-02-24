---
name: bird
description: X/Twitter CLI 用于通过 cookies 读取、搜索、发布和互动。
homepage: https://bird.fast
metadata: {"openclaw":{"emoji":"🐦","requires":{"bins":["bird"]},"install":[{"id":"brew","kind":"brew","formula":"steipete/tap/bird","bins":["bird"],"label":"Install bird (brew)","os":["darwin"]},{"id":"npm","kind":"node","package":"@steipete/bird","bins":["bird"],"label":"Install bird (npm)"}]}}
---

# bird 🐦

快速的 X/Twitter CLI,使用 GraphQL + cookie 认证。

## 安装

```bash
# npm/pnpm/bun
npm install -g @steipete/bird

# Homebrew (macOS, 预构建二进制)
brew install steipete/tap/bird

# 一次性使用(无需安装)
bunx @steipete/bird whoami
```

## 认证

`bird` 使用基于 cookie 的认证。

使用 `--auth-token` / `--ct0` 直接传递 cookies,或使用 `--cookie-source` 获取浏览器 cookies。

运行 `bird check` 查看哪个源是活动的。对于 Arc/Brave,使用 `--chrome-profile-dir <path>`。

## 命令

### 账户 & 认证

```bash
bird whoami                    # 显示登录的账户
bird check                     # 显示凭据来源
bird query-ids --fresh         # 刷新 GraphQL 查询 ID 缓存
```

### 读取推文

```bash
bird read <url-or-id>          # 读取单条推文
bird <url-or-id>               # read 的简写
bird thread <url-or-id>        # 完整的对话线程
bird replies <url-or-id>       # 列出推文的回复
```

### 时间线

```bash
bird home                      # 主页时间线(For You)
bird home --following          # Following 时间线
bird user-tweets @handle -n 20 # 用户的个人资料时间线
bird mentions                  # 提到你的推文
bird mentions --user @handle   # 提到另一用户的推文
```

### 搜索

```bash
bird search "query" -n 10
bird search "from:steipete" --all --max-pages 3
```

### 新闻 & 趋势

```bash
bird news -n 10                # 从 Explore 选项卡的 AI 精选
bird news --ai-only            # 仅过滤 AI 精选
bird news --sports             # 体育选项卡
bird news --with-tweets        # 包含相关推文
bird trending                  # news 的别名
```

### 列表

```bash
bird lists                     # 你的列表
bird lists --member-of         # 你是成员的列表
bird list-timeline <id> -n 20  # 列表中的推文
```

### 书签 & 点赞

```bash
bird bookmarks -n 10
bird bookmarks --folder-id <id>           # 特定文件夹
bird bookmarks --include-parent           # 包含父推文
bird bookmarks --author-chain             # 作者的自我回复链
bird bookmarks --full-chain-only          # 完整回复链
bird unbookmark <url-or-id>
bird likes -n 10
```

### 社交图谱

```bash
bird following -n 20           # 你关注的用户
bird followers -n 20           # 关注你的用户
bird following --user <id>     # 另一用户的关注
bird about @handle             # 账户来源/位置信息
```

### 互动操作

```bash
bird follow @handle            # 关注用户
bird unfollow @handle          # 取消关注用户
```

### 发布

```bash
bird tweet "hello world"
bird reply <url-or-id> "nice thread!"
bird tweet "check this out" --media image.png --alt "description"
```

**⚠️ 发布风险**: 发布更可能受到速率限制;如果被阻止,请改用浏览器工具。

## 媒体上传

```bash
bird tweet "hi" --media img.png --alt "description"
bird tweet "pics" --media a.jpg --media b.jpg  # 最多 4 张图片
bird tweet "video" --media clip.mp4            # 或 1 个视频
```

## 分页

支持分页的命令: `replies`、`thread`、`search`、`bookmarks`、`likes`、`list-timeline`、`following`、`followers`、`user-tweets`

```bash
bird bookmarks --all                    # 获取所有页面
bird bookmarks --max-pages 3            # 限制页面
bird bookmarks --cursor <cursor>        # 从游标恢复
bird replies <id> --all --delay 1000    # 页面间延迟(毫秒)
```

## 输出选项

```bash
--json          # JSON 输出
--json-full     # JSON 包含原始 API 响应
--plain         # 无表情符号,无颜色(脚本友好)
--no-emoji      # 禁用表情符号
--no-color      # 禁用 ANSI 颜色(或设置 NO_COLOR=1)
--quote-depth n # JSON 中的最大引用推文深度(默认: 1)
```

## 全局选项

```bash
--auth-token <token>       # 设置 auth_token cookie
--ct0 <token>              # 设置 ct0 cookie
--cookie-source <source>   # 浏览器 cookies 的 cookie 源(可重复)
--chrome-profile <name>    # Chrome 配置文件名称
--chrome-profile-dir <path> # Chrome/Chromium 配置文件目录或 cookie DB 路径
--firefox-profile <name>   # Firefox 配置文件
--timeout <ms>             # 请求超时
--cookie-timeout <ms>      # Cookie 提取超时
```

## 配置文件

`~/.config/bird/config.json5`(全局)或 `./.birdrc.json5`(项目):

```json5
{
  cookieSource: ["chrome"],
  chromeProfileDir: "/path/to/Arc/Profile",
  timeoutMs: 20000,
  quoteDepth: 1
}
```

环境变量: `BIRD_TIMEOUT_MS`、`BIRD_COOKIE_TIMEOUT_MS`、`BIRD_QUOTE_DEPTH`

## 故障排除

### 查询 ID 过期(404 错误)
```bash
bird query-ids --fresh
```

### Cookie 提取失败
- 检查浏览器是否登录 X
- 尝试不同的 `--cookie-source`
- 对于 Arc/Brave: 使用 `--chrome-profile-dir`

---

**TL;DR**: 使用 CLI 读取/搜索/互动。小心发布或使用浏览器。🐦
