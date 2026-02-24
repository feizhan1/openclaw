---
name: clawdhub
description: 使用 ClawdHub CLI 从 clawdhub.com 搜索、安装、更新和发布代理技能。当你需要动态获取新技能、将已安装的技能同步到最新版本或特定版本,或使用 npm 安装的 clawdhub CLI 发布新的/更新的技能文件夹时使用。
metadata: {"openclaw":{"requires":{"bins":["clawdhub"]},"install":[{"id":"node","kind":"node","package":"clawdhub","bins":["clawdhub"],"label":"Install ClawdHub CLI (npm)"}]}}
---

# ClawdHub CLI

安装
```bash
npm i -g clawdhub
```

认证(发布)
```bash
clawdhub login
clawdhub whoami
```

搜索
```bash
clawdhub search "postgres backups"
```

安装
```bash
clawdhub install my-skill
clawdhub install my-skill --version 1.2.3
```

更新(基于哈希匹配 + 升级)
```bash
clawdhub update my-skill
clawdhub update my-skill --version 1.2.3
clawdhub update --all
clawdhub update my-skill --force
clawdhub update --all --no-input --force
```

列表
```bash
clawdhub list
```

发布
```bash
clawdhub publish ./my-skill --slug my-skill --name "My Skill" --version 1.2.0 --changelog "Fixes + docs"
```

注意事项
- 默认注册表: https://clawdhub.com(使用 CLAWDHUB_REGISTRY 或 --registry 覆盖)
- 默认工作目录: cwd(回退到 OpenClaw 工作区);安装目录: ./skills(使用 --workdir / --dir / CLAWDHUB_WORKDIR 覆盖)
- update 命令会哈希本地文件,解析匹配版本,并升级到最新版本,除非设置了 --version
