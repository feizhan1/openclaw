# OpenClaw 记忆系统运作流程速览

## 核心流程图

```
📝 创建/修改 MEMORY.md
        ↓
👁️ 文件监控检测 (1.5s 防抖)
        ↓
📊 分块处理 (400 tokens/块, 80 tokens 重叠)
        ↓
🧠 生成向量 (768 维嵌入)
        ↓
💾 存入 SQLite
   ├─ chunks 表 (文本 + 向量 JSON)
   ├─ chunks_vec (向量索引 - sqlite-vec)
   └─ chunks_fts (全文索引 - FTS5)
        ↓
🔍 用户搜索 "OpenClaw 记忆"
        ↓
⚡ 并行搜索
   ├─ 向量搜索 (余弦相似度) → 分数 0.766
   └─ 关键词搜索 (BM25) → 分数 0.299
        ↓
🔀 混合合并
   最终分数 = 0.7 × 0.766 + 0.3 × 0.299 = 0.626
        ↓
📋 返回 top-6 结果 (score >= 0.35)
```

## 数据存储结构

### 文件系统
```
~/.openclaw/
├── workspace/
│   ├── MEMORY.md          ← 你手动编写的记忆
│   └── memory/*.md        ← 额外的笔记
└── memory/
    └── main.sqlite        ← 系统自动生成的索引 (3.1MB)
```

### SQLite 数据库内部
```sql
-- 1. 文件跟踪
files 表:
  path: "MEMORY.md"
  hash: "bdc1be78..." (用于增量检测)
  size: 475 bytes

-- 2. 文本块
chunks 表:
  id: "902df866..." (SHA256)
  text: "# 我的记忆库\n\n## 项目信息..."
  start_line: 1, end_line: 19
  embedding: [0.636, 0.247, ...] (768 个浮点数, JSON 格式)

-- 3. 向量索引 (快速相似度计算)
chunks_vec 虚拟表:
  id: "902df866..."
  embedding: <binary FLOAT[768]>

-- 4. 全文索引 (快速关键词查找)
chunks_fts 虚拟表:
  text: "我的记忆库 项目信息 OpenClaw..."
  (自动分词、倒排索引)

-- 5. 嵌入缓存 (避免重复计算)
embedding_cache 表:
  hash: "902df866..."
  embedding: [0.636, 0.247, ...]
  (重索引时复用)
```

## 关键算法

### 1. 向量相似度 (余弦相似度)
```javascript
// 计算两个向量的夹角余弦值
function cosineSimilarity(a, b) {
  const dot = a.reduce((sum, ai, i) => sum + ai * b[i], 0);
  const magA = Math.sqrt(a.reduce((sum, ai) => sum + ai * ai, 0));
  const magB = Math.sqrt(b.reduce((sum, bi) => sum + bi * bi, 0));
  return dot / (magA * magB);  // 范围: -1 到 1
}

// 示例
查询向量: [0.5, 0.3, 0.8]
文档向量: [0.6, 0.2, 0.7]
相似度 = 0.92 (非常相似)
```

### 2. BM25 关键词匹配
```
BM25 = Σ (IDF × 词频权重)

IDF (逆文档频率) = 稀有词权重高
词频权重 = 出现越多越高，但有饱和点

示例：
"OpenClaw" (稀有词) → IDF = 3.5
"记忆" (常见词) → IDF = 1.2
"系统" (常见词) → IDF = 1.0

文档A: "OpenClaw 记忆系统" → BM25 = 5.7
文档B: "记忆和系统" → BM25 = 2.2
```

### 3. 混合评分
```javascript
// 加权组合两种搜索结果
finalScore = 0.7 × vectorScore + 0.3 × textScore

// 为什么这样分配？
// - 向量搜索擅长语义相似（"猫" ≈ "小猫咪"）
// - 关键词搜索擅长精确匹配（变量名、错误代码）
// - 70:30 是实验得出的最优比例

示例：
查询: "数据库优化"
结果A: 向量 0.9 + 关键词 0.3 = 0.72 (语义相关但词不同)
结果B: 向量 0.5 + 关键词 0.9 = 0.62 (词完全匹配)
排序: A > B (语义优先)
```

## 实际数据示例

### 你的当前索引状态
```bash
$ sqlite3 ~/.openclaw/memory/main.sqlite

# 查看索引的文件
sqlite> SELECT * FROM files;
path        | source | hash         | size
MEMORY.md   | memory | bdc1be78...  | 475

# 查看文本块
sqlite> SELECT id, start_line, end_line, substr(text, 1, 50) FROM chunks;
id              | start | end | text_preview
902df866...     | 1     | 19  | # 我的记忆库\n\n## 项目信息...

# 查看向量维度
sqlite> SELECT length(embedding) FROM chunks LIMIT 1;
15464  (768 个浮点数 × 约 20 字节)
```

### 搜索结果示例
```bash
$ pnpm openclaw memory search "OpenClaw 项目"

0.760 MEMORY.md:1-19
# 我的记忆库

## 项目信息
- **OpenClaw 代码位置**: /Users/feizhan/code/openclaw
- **当前分支**: main

## 学习笔记
- OpenClaw 是一个个人 AI 助手，可以连接多个消息平台
- 使用向量检索的混合搜索（BM25 + 语义相似度）
```

**分数解读**：
- 0.760 = 76% 相关度
- 0.350 = 最低阈值（可配置）
- 1.000 = 完全匹配（几乎不可能）

## 性能数据

### 本地嵌入模型
```
模型: embeddinggemma-300M-Q8_0.gguf
大小: 328MB
首次下载: ~12 秒 (27MB/s)
嵌入速度: ~30 块/秒 (CPU)
向量维度: 768
质量: 中等
```

### 索引统计
```
源文件: 1 个 (MEMORY.md, 475 bytes)
文本块: 1 个 (第 1-19 行)
向量索引: 1 个 (15.5KB)
全文索引: ~2KB
缓存条目: 1 个
总数据库大小: 3.1MB
```

### 搜索性能
```
向量搜索: <10ms (sqlite-vec SIMD 加速)
关键词搜索: <5ms (FTS5 倒排索引)
合并结果: <1ms
总耗时: ~15ms
```

## 常见问题

### Q: 为什么需要 768 维向量？
A: 维度越高，语义表示越精确。768 是 Transformer 模型的标准输出维度，平衡了精度和性能。

### Q: 为什么分块要重叠 80 tokens？
A: 避免在句子/段落中间切断，捕获跨块的上下文关系。

### Q: BM25 和 TF-IDF 有什么区别？
A: BM25 改进了 TF-IDF，加入了文档长度归一化和词频饱和机制，更适合现代搜索。

### Q: 为什么不全用向量搜索？
A: 向量搜索对精确匹配（代码、变量名、错误信息）效果差，需要结合关键词搜索。

### Q: 索引会占用多少空间？
A: 约为源文件的 5-10 倍。1MB 文本 → 5-10MB 索引（大部分是向量数据）。

## 动手实验

### 实验 1：观察分块
```bash
# 查看你的文本如何被分块
sqlite3 ~/.openclaw/memory/main.sqlite << 'EOF'
SELECT
  'Chunk ' || ROW_NUMBER() OVER() as num,
  start_line || '-' || end_line as lines,
  length(text) as chars,
  substr(text, 1, 100) as preview
FROM chunks;
EOF
```

### 实验 2：对比搜索算法
```bash
# 仅向量搜索（修改配置）
{
  "memorySearch": {
    "query": {
      "hybrid": {
        "vectorWeight": 1.0,
        "textWeight": 0.0
      }
    }
  }
}

# 仅关键词搜索
{
  "memorySearch": {
    "query": {
      "hybrid": {
        "vectorWeight": 0.0,
        "textWeight": 1.0
      }
    }
  }
}

# 对比搜索结果的差异
```

### 实验 3：查看向量数据
```bash
# 提取并可视化向量
sqlite3 ~/.openclaw/memory/main.sqlite << 'EOF'
SELECT
  substr(embedding, 1, 200) as first_few_dims,
  json_array_length(embedding) as total_dims
FROM chunks
LIMIT 1;
EOF
```

### 实验 4：测试搜索召回率
```bash
# 添加测试内容到 MEMORY.md
echo "## 测试记录
- 今天完成了向量搜索的优化
- 使用了余弦相似度算法
- 性能提升了 10 倍
" >> ~/.openclaw/workspace/MEMORY.md

# 等待索引（或手动触发）
pnpm openclaw memory index

# 尝试不同查询
pnpm openclaw memory search "向量"
pnpm openclaw memory search "优化"
pnpm openclaw memory search "cosine similarity"  # 跨语言？
pnpm openclaw memory search "性能"
```

## 下一步学习

1. **详细内部机制**: 阅读 `memory-internals-explained.md`
2. **配置优化**: 阅读 `openclaw-memory-guide.md`
3. **源码深入**: 浏览 `src/memory/` 目录
4. **实际应用**: 在 AI 对话中观察 `memory_search` 工具的使用

## 核心要点总结

| 概念 | 解释 | 类比 |
|------|------|------|
| **向量嵌入** | 将文本转换为数字向量 | 就像给每个句子分配一个 GPS 坐标 |
| **余弦相似度** | 计算两个向量的夹角 | 两个箭头指向越接近，语义越相似 |
| **BM25** | 关键词相关度算法 | Google 搜索的核心算法之一 |
| **混合搜索** | 向量 + 关键词组合 | 既懂语义，又认识准确词汇 |
| **分块** | 将长文本切成小段 | 方便索引和检索，避免"信息过载" |
| **sqlite-vec** | 向量数据库扩展 | 让 SQLite 支持向量运算 |
| **FTS5** | 全文搜索引擎 | SQLite 内置的 Google 搜索 |

---

现在你已经理解记忆系统的运作原理了！🎓

试试添加更多内容到 MEMORY.md，观察系统如何自动索引和搜索。
