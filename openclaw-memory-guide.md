# OpenClaw 长期记忆系统指南

## 概述

OpenClaw 使用**向量检索 + 全文搜索**的混合记忆系统，让 AI 能够记住和检索历史信息。

### 核心特性
- **混合搜索**: 语义相似度（70%）+ 关键词匹配（30%）
- **懒加载索引**: 首次使用时自动创建，按需同步
- **多种嵌入**: 支持 OpenAI、Gemini、本地模型
- **自动同步**: 文件变化、会话启动、搜索前自动更新

## 数据存储

### 源文件（手动编写）
```
~/.openclaw/workspace/
├── MEMORY.md          # 主记忆文件（精选知识）
└── memory/            # 记忆笔记目录
    └── *.md           # 额外的记忆文件
```

### 索引数据库（自动生成）
```
~/.openclaw/memory/
└── {agentId}.sqlite   # 向量索引 + 全文检索索引
```

## 快速开始

### 1. 创建记忆文件
```bash
cat > ~/.openclaw/workspace/MEMORY.md << 'EOF'
# 我的记忆库

## 项目信息
- 项目名称：xxx
- 代码位置：/path/to/project

## 重要笔记
- 关键技术栈和架构决策

## 待办事项
- [ ] 任务清单
EOF
```

### 2. 配置嵌入模型

#### 选项 A：本地模型（无需 API，推荐测试）
编辑 `~/.openclaw/openclaw.json`：
```json
{
  "agents": {
    "defaults": {
      "memorySearch": {
        "enabled": true,
        "provider": "local"
      }
    }
  }
}
```

#### 选项 B：OpenAI（需要 API 密钥）
```json
{
  "agents": {
    "defaults": {
      "memorySearch": {
        "enabled": true,
        "provider": "openai",
        "model": "text-embedding-3-small",
        "remote": {
          "apiKey": "${OPENAI_API_KEY}"
        }
      }
    }
  }
}
```

### 3. 触发索引创建

#### 方法 1：CLI 命令（手动）
```bash
# 确保使用 Node >= 22
nvm use 22

# 创建/更新索引
pnpm openclaw memory index --verbose

# 查看状态
pnpm openclaw memory status --deep
```

#### 方法 2：自动触发（推荐）
索引会在以下情况自动创建：
- 🚀 启动新会话（`sync.onSessionStart: true`）
- 👁️ 检测到文件变化（`sync.watch: true`，1.5s 延迟）
- 🔍 执行记忆搜索前（`sync.onSearch: true`）

## 常用命令

### 查看状态
```bash
pnpm openclaw memory status          # 基本状态
pnpm openclaw memory status --deep   # 详细状态（探测嵌入）
pnpm openclaw memory status --index  # 显示状态并重新索引
```

### 重新索引
```bash
pnpm openclaw memory index           # 增量索引
pnpm openclaw memory index --force   # 强制完整重索引
pnpm openclaw memory index --verbose # 显示详细进度
```

### 搜索记忆
```bash
pnpm openclaw memory search "查询关键词"
pnpm openclaw memory search "查询" --max-results 10
pnpm openclaw memory search "查询" --min-score 0.5
pnpm openclaw memory search "查询" --json  # JSON 输出
```

## 配置参考

### 完整配置示例
```json
{
  "agents": {
    "defaults": {
      "memorySearch": {
        "enabled": true,
        "provider": "auto",
        "sources": ["memory", "sessions"],
        "extraPaths": ["/path/to/other/docs"],

        "remote": {
          "baseUrl": "https://api.openai.com/v1",
          "apiKey": "${OPENAI_API_KEY}",
          "batch": {
            "enabled": true,
            "wait": true,
            "concurrency": 2
          }
        },

        "fallback": "gemini",
        "model": "text-embedding-3-small",

        "local": {
          "modelPath": "hf:ggml-org/embeddinggemma-300M-GGUF/embeddinggemma-300M-Q8_0.gguf"
        },

        "store": {
          "driver": "sqlite",
          "path": "~/.openclaw/memory/{agentId}.sqlite",
          "vector": {
            "enabled": true
          }
        },

        "chunking": {
          "tokens": 400,
          "overlap": 80
        },

        "sync": {
          "onSessionStart": true,
          "onSearch": true,
          "watch": true,
          "watchDebounceMs": 1500,
          "intervalMinutes": 0,
          "sessions": {
            "deltaBytes": 100000,
            "deltaMessages": 50
          }
        },

        "query": {
          "maxResults": 6,
          "minScore": 0.35,
          "hybrid": {
            "enabled": true,
            "vectorWeight": 0.7,
            "textWeight": 0.3
          }
        },

        "cache": {
          "enabled": true,
          "maxEntries": 50000
        }
      }
    }
  }
}
```

### 关键配置项说明

| 配置项 | 说明 | 默认值 |
|--------|------|--------|
| `enabled` | 启用记忆搜索 | `true` |
| `provider` | 嵌入提供商：`auto` / `openai` / `gemini` / `local` | `auto` |
| `sources` | 数据源：`["memory"]` 或 `["memory", "sessions"]` | `["memory"]` |
| `extraPaths` | 额外的记忆文件路径 | `[]` |
| `chunking.tokens` | 每个文本块的大小（词元） | `400` |
| `chunking.overlap` | 块之间重叠大小 | `80` |
| `sync.watch` | 启用文件监控 | `true` |
| `sync.onSessionStart` | 会话启动时同步 | `true` |
| `query.maxResults` | 最大搜索结果数 | `6` |
| `query.minScore` | 最小相似度分数（0-1） | `0.35` |
| `hybrid.vectorWeight` | 向量搜索权重 | `0.7` |
| `hybrid.textWeight` | 关键词搜索权重 | `0.3` |

## AI 工具集成

AI 自动可用的工具：

### memory_search
语义搜索记忆库
```typescript
memory_search({
  query: "查询内容",
  maxResults?: 6,
  minScore?: 0.35
})
```

### memory_get
读取特定文件的指定行
```typescript
memory_get({
  path: "MEMORY.md",
  from?: 10,    // 起始行（1-indexed）
  lines?: 20    // 读取行数
})
```

## 技术架构

### 数据库结构
- **files 表**: 源文件元数据（路径、哈希、修改时间）
- **chunks 表**: 文本块（ID、路径、行号、文本、向量）
- **chunks_fts 虚拟表**: FTS5 全文检索索引
- **chunks_vec 虚拟表**: sqlite-vec 向量索引（768 维）
- **embedding_cache 表**: 嵌入缓存（避免重复计算）

### 搜索流程
1. 并行执行向量搜索和 BM25 关键词搜索
2. 按 chunk ID 合并结果
3. 加权计算最终分数：`0.7 × 向量分数 + 0.3 × 关键词分数`
4. 过滤低于阈值的结果（默认 0.35）
5. 返回 top-N 结果（默认 6 条）

### 重索引机制
- **增量索引**: 仅更新变更的文件（基于哈希）
- **完整重索引**: 删除所有索引，重新构建（使用临时数据库保证原子性）
- **嵌入缓存**: 跨重索引保留，避免重复计算

## 故障排查

### 问题 1: 索引未自动创建
**原因**: Gateway 未运行或 Node 版本过低
**解决**:
```bash
# 检查 Gateway
ps aux | grep openclaw-gateway

# 升级 Node
nvm install 22 && nvm use 22

# 手动触发
pnpm openclaw memory index
```

### 问题 2: API 密钥错误
**原因**: 未配置 OpenAI/Gemini API 密钥
**解决**: 切换到本地模型或配置 API 密钥
```json
{
  "memorySearch": {
    "provider": "local"  // 使用本地模型
  }
}
```

### 问题 3: 搜索结果不准确
**原因**: 索引过期或分数阈值过高
**解决**:
```bash
# 强制重新索引
pnpm openclaw memory index --force

# 降低分数阈值
# 在配置中设置 query.minScore: 0.2
```

### 问题 4: 索引速度慢
**原因**: 文件太大或使用远程批处理 API
**解决**:
```json
{
  "memorySearch": {
    "provider": "local",  // 本地模型更快
    "chunking": {
      "tokens": 300      // 减小块大小
    }
  }
}
```

## 最佳实践

### 记忆文件组织
```
MEMORY.md                  # 核心知识（保持精简，< 200 行）
memory/
├── daily-notes/          # 日常笔记
│   ├── 2026-02.md
│   └── 2026-01.md
├── projects/             # 项目笔记
│   ├── project-a.md
│   └── project-b.md
└── references/           # 参考资料
    └── tech-stack.md
```

### 写作建议
1. **使用清晰的标题**: 方便语义分块
2. **包含关键词**: 帮助关键词搜索
3. **避免过长段落**: 保持块大小合理
4. **定期整理**: 合并相似主题，删除过时信息
5. **使用 Markdown 格式**: 代码块、列表、强调等

### 性能优化
- 保持 MEMORY.md 精简（< 5000 行）
- 使用 `extraPaths` 分离大型文档
- 禁用不需要的数据源（如 sessions）
- 定期清理过期内容
- 本地模型适合快速开发，远程模型适合生产

## 参考资源

- **源码**: `/Users/feizhan/code/openclaw/src/memory/`
- **CLI**: `/Users/feizhan/code/openclaw/src/cli/memory-cli.ts`
- **配置**: `~/.openclaw/openclaw.json`
- **数据**: `~/.openclaw/memory/` 和 `~/.openclaw/workspace/`
