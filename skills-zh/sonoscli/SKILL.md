---
name: sonoscli
description: 控制 Sonos 扬声器(发现/状态/播放/音量/分组)。
homepage: https://sonoscli.sh
metadata: {"openclaw":{"emoji":"🔊","requires":{"bins":["sonos"]},"install":[{"id":"go","kind":"go","module":"github.com/steipete/sonoscli/cmd/sonos@latest","bins":["sonos"],"label":"Install sonoscli (go)"}]}}
---

# Sonos CLI

使用 `sonos` 控制本地网络上的 Sonos 扬声器。

快速开始
- `sonos discover`
- `sonos status --name "Kitchen"`
- `sonos play|pause|stop --name "Kitchen"`
- `sonos volume set 15 --name "Kitchen"`

常用任务
- 分组: `sonos group status|join|unjoin|party|solo`
- 收藏: `sonos favorites list|open`
- 队列: `sonos queue list|play|clear`
- Spotify 搜索(通过 SMAPI): `sonos smapi search --service "Spotify" --category tracks "query"`

注意事项
- 如果 SSDP 失败,指定 `--ip <speaker-ip>`。
- Spotify Web API 搜索是可选的,需要 `SPOTIFY_CLIENT_ID/SECRET`。
