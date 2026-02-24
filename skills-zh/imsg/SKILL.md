---
name: imsg
description: iMessage/SMS CLI 用于列出聊天、历史、监视和发送。
homepage: https://imsg.to
metadata: {"openclaw":{"emoji":"📨","os":["darwin"],"requires":{"bins":["imsg"]},"install":[{"id":"brew","kind":"brew","formula":"steipete/tap/imsg","bins":["imsg"],"label":"Install imsg (brew)"}]}}
---

# imsg

使用 `imsg` 在 macOS 上读取和发送 Messages.app 的 iMessage/SMS。

要求
- Messages.app 已登录
- 终端的完全磁盘访问权限
- 控制 Messages.app 的自动化权限(用于发送)

常用命令
- 列出聊天: `imsg chats --limit 10 --json`
- 历史: `imsg history --chat-id 1 --limit 20 --attachments --json`
- 监视: `imsg watch --chat-id 1 --attachments`
- 发送: `imsg send --to "+14155551212" --text "hi" --file /path/pic.jpg`

注意事项
- `--service imessage|sms|auto` 控制传递方式。
- 发送前确认收件人 + 消息。
