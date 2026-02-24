---
name: camsnap
description: 从 RTSP/ONVIF 摄像头捕获帧或剪辑。
homepage: https://camsnap.ai
metadata: {"openclaw":{"emoji":"📸","requires":{"bins":["camsnap"]},"install":[{"id":"brew","kind":"brew","formula":"steipete/tap/camsnap","bins":["camsnap"],"label":"Install camsnap (brew)"}]}}
---

# camsnap

使用 `camsnap` 从配置的摄像头抓取快照、剪辑或运动事件。

设置
- 配置文件: `~/.config/camsnap/config.yaml`
- 添加摄像头: `camsnap add --name kitchen --host 192.168.0.10 --user user --pass pass`

常用命令
- 发现: `camsnap discover --info`
- 快照: `camsnap snap kitchen --out shot.jpg`
- 剪辑: `camsnap clip kitchen --dur 5s --out clip.mp4`
- 运动监控: `camsnap watch kitchen --threshold 0.2 --action '...'`
- 诊断: `camsnap doctor --probe`

注意事项
- 需要 `ffmpeg` 在 PATH 中。
- 建议先进行短时间测试捕获,再进行较长剪辑。
