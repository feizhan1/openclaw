# Canvas 技能

在连接的 OpenClaw 节点(Mac app、iOS、Android)上显示 HTML 内容。

## 概述

canvas 工具让你在任何连接的节点的画布视图上呈现 Web 内容。适用于:
- 显示游戏、可视化、仪表板
- 展示生成的 HTML 内容
- 交互式演示

## 工作原理

### 架构

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────┐
│  Canvas Host    │────▶│   Node Bridge    │────▶│  Node App   │
│  (HTTP Server)  │     │  (TCP Server)    │     │ (Mac/iOS/   │
│  Port 18793     │     │  Port 18790      │     │  Android)   │
└─────────────────┘     └──────────────────┘     └─────────────┘
```

1. **Canvas Host Server**: 从 `canvasHost.root` 目录提供静态 HTML/CSS/JS 文件
2. **Node Bridge**: 将画布 URL 传递给连接的节点
3. **Node Apps**: 在 WebView 中渲染内容

### Tailscale 集成

画布主机服务器根据 `gateway.bind` 设置进行绑定:

| 绑定模式 | 服务器绑定到 | Canvas URL 使用 |
|-----------|-----------------|-----------------|
| `loopback` | 127.0.0.1 | localhost (仅本地) |
| `lan` | LAN 接口 | LAN IP 地址 |
| `tailnet` | Tailscale 接口 | Tailscale 主机名 |
| `auto` | 最佳可用 | Tailscale > LAN > loopback |

**关键点:** `canvasHostHostForBridge` 从 `bridgeHost` 派生。当绑定到 Tailscale 时,节点接收如下 URL:
```
http://<tailscale-hostname>:18793/__moltbot__/canvas/<file>.html
```

这就是为什么 localhost URL 不起作用 - 节点从网桥接收 Tailscale 主机名!

## 操作

| 操作 | 描述 |
|--------|-------------|
| `present` | 显示画布,可选目标 URL |
| `hide` | 隐藏画布 |
| `navigate` | 导航到新 URL |
| `eval` | 在画布中执行 JavaScript |
| `snapshot` | 捕获画布截图 |

## 配置

在 `~/.clawdbot/openclaw.json` 中:

```json
{
  "canvasHost": {
    "enabled": true,
    "port": 18793,
    "root": "/Users/you/clawd/canvas",
    "liveReload": true
  },
  "gateway": {
    "bind": "auto"
  }
}
```

### 实时重载

当 `liveReload: true`(默认)时,画布主机:
- 监视根目录的更改(通过 chokidar)
- 将 WebSocket 客户端注入 HTML 文件
- 文件更改时自动重新加载连接的画布

非常适合开发!

## 工作流程

### 1. 创建 HTML 内容

将文件放在画布根目录(默认 `~/clawd/canvas/`)中:

```bash
cat > ~/clawd/canvas/my-game.html << 'HTML'
<!DOCTYPE html>
<html>
<head><title>My Game</title></head>
<body>
  <h1>Hello Canvas!</h1>
</body>
</html>
HTML
```

### 2. 查找画布主机 URL

检查网关的绑定方式:
```bash
cat ~/.clawdbot/openclaw.json | jq '.gateway.bind'
```

然后构造 URL:
- **loopback**: `http://127.0.0.1:18793/__moltbot__/canvas/<file>.html`
- **lan/tailnet/auto**: `http://<hostname>:18793/__moltbot__/canvas/<file>.html`

查找 Tailscale 主机名:
```bash
tailscale status --json | jq -r '.Self.DNSName' | sed 's/\.$//'
```

### 3. 查找连接的节点

```bash
openclaw nodes list
```

寻找具有画布功能的 Mac/iOS/Android 节点。

### 4. 呈现内容

```
canvas action:present node:<node-id> target:<full-url>
```

**示例:**
```
canvas action:present node:mac-63599bc4-b54d-4392-9048-b97abd58343a target:http://peters-mac-studio-1.sheep-coho.ts.net:18793/__moltbot__/canvas/snake.html
```

### 5. 导航、截图或隐藏

```
canvas action:navigate node:<node-id> url:<new-url>
canvas action:snapshot node:<node-id>
canvas action:hide node:<node-id>
```

## 调试

### 白屏 / 内容未加载

**原因:** 服务器绑定和节点预期之间的 URL 不匹配。

**调试步骤:**
1. 检查服务器绑定: `cat ~/.clawdbot/openclaw.json | jq '.gateway.bind'`
2. 检查画布在哪个端口: `lsof -i :18793`
3. 直接测试 URL: `curl http://<hostname>:18793/__moltbot__/canvas/<file>.html`

**解决方案:** 使用与绑定模式匹配的完整主机名,而不是 localhost。

### "node required" 错误

始终指定 `node:<node-id>` 参数。

### "node not connected" 错误

节点离线。使用 `openclaw nodes list` 查找在线节点。

### 内容未更新

如果实时重载不起作用:
1. 检查配置中的 `liveReload: true`
2. 确保文件在画布根目录中
3. 检查日志中的监视器错误

## URL 路径结构

画布主机从 `/__moltbot__/canvas/` 前缀提供服务:

```
http://<host>:18793/__moltbot__/canvas/index.html  → ~/clawd/canvas/index.html
http://<host>:18793/__moltbot__/canvas/games/snake.html → ~/clawd/canvas/games/snake.html
```

`/__moltbot__/canvas/` 前缀由 `CANVAS_HOST_PATH` 常量定义。

## 提示

- 保持 HTML 自包含(内联 CSS/JS)以获得最佳效果
- 使用默认的 index.html 作为测试页面(包含网桥诊断)
- 画布会持续存在,直到你 `hide` 它或导航离开
- 实时重载使开发速度快 - 只需保存它就会更新!
- A2UI JSON 推送是 WIP - 目前使用 HTML 文件
