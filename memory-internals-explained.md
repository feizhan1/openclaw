# OpenClaw 记忆系统内部运作详解

## 数据库结构解析

### 核心表结构

#### 1. meta 表 - 索引元数据
```sql
CREATE TABLE meta (
  key TEXT PRIMARY KEY,
  value TEXT NOT NULL
);
```

**实际数据示例**：
```json
{
  "key": "memory_index_meta_v1",
  "value": {
    "model": "hf:ggml-org/embeddinggemma-300M-GGUF/embeddinggemma-300M-Q8_0.gguf",
    "provider": "local",
    "providerKey": "19237057d843b159...",
    "chunkTokens": 400,
    "chunkOverlap": 80,
    "vectorDims": 768
  }
}
```

**作用**：
- 记录索引配置（模型、提供商、分块参数）
- 用于检测配置变化，触发重索引
- `providerKey` = SHA256(provider + model + baseUrl + headers)

---

#### 2. files 表 - 源文件跟踪
```sql
CREATE TABLE files (
  path TEXT PRIMARY KEY,
  source TEXT NOT NULL DEFAULT 'memory',
  hash TEXT NOT NULL,
  mtime INTEGER NOT NULL,
  size INTEGER NOT NULL
);
```

**实际数据**：
```
path        | source | hash                                              | size
------------|--------|---------------------------------------------------|------
MEMORY.md   | memory | bdc1be78d2a827c41775ab68440cb345f0302d27065873... | 475
```

**作用**：
- 跟踪已索引的文件
- `hash` 用于检测文件内容变化（增量索引）
- `mtime` 记录修改时间
- `source` 区分来源（`memory` 或 `sessions`）

---

#### 3. chunks 表 - 文本块存储
```sql
CREATE TABLE chunks (
  id TEXT PRIMARY KEY,
  path TEXT NOT NULL,
  source TEXT NOT NULL DEFAULT 'memory',
  start_line INTEGER NOT NULL,
  end_line INTEGER NOT NULL,
  hash TEXT NOT NULL,
  model TEXT NOT NULL,
  text TEXT NOT NULL,
  embedding TEXT NOT NULL,
  updated_at INTEGER NOT NULL
);
CREATE INDEX idx_chunks_path ON chunks(path);
CREATE INDEX idx_chunks_source ON chunks(source);
```

**实际数据**：
```
id: 902df866f4c0094ed72bf4b2d2cc17369bfaf577b858e2ac44755df194773953
path: MEMORY.md
source: memory
start_line: 1
end_line: 19
text: # 我的记忆库\n\n## 项目信息\n- **OpenClaw 代码位置**: ...
embedding: [0.636, 0.247, -0.010, -0.412, ...]  // 768 个浮点数
```

**字段说明**：
- `id` = SHA256(文本内容)，用于去重
- `start_line/end_line`：在源文件中的行号范围
- `text`：标准化后的文本（去除多余空格、换行）
- `embedding`：JSON 数组格式的向量（768 维）

---

#### 4. chunks_fts 虚拟表 - 全文检索索引
```sql
CREATE VIRTUAL TABLE chunks_fts USING fts5(
  text,
  id UNINDEXED,
  path UNINDEXED,
  source UNINDEXED,
  model UNINDEXED,
  start_line UNINDEXED,
  end_line UNINDEXED
);
```

**作用**：
- 基于 SQLite FTS5 的全文检索引擎
- 只有 `text` 列被索引，其他列用于返回元数据
- 使用 **BM25** 算法计算相关度
- 支持布尔查询（AND、OR、NOT）

---

#### 5. chunks_vec 虚拟表 - 向量索引
```sql
CREATE VIRTUAL TABLE chunks_vec USING vec0(
  id TEXT PRIMARY KEY,
  embedding FLOAT[768]
);
```

**作用**：
- 基于 `sqlite-vec` 扩展的向量数据库
- 支持**余弦相似度**快速计算
- `vec_distance_cosine(v1, v2)` 返回余弦距离（0-2，越小越相似）
- 原生 C 实现，比 JavaScript 快 100 倍以上

---

#### 6. embedding_cache 表 - 嵌入缓存
```sql
CREATE TABLE embedding_cache (
  provider TEXT NOT NULL,
  model TEXT NOT NULL,
  provider_key TEXT NOT NULL,
  hash TEXT NOT NULL,
  embedding TEXT NOT NULL,
  dims INTEGER,
  updated_at INTEGER NOT NULL,
  PRIMARY KEY (provider, model, provider_key, hash)
);
```

**作用**：
- 缓存文本块的嵌入向量
- 避免重复计算（尤其是远程 API 调用）
- 重索引时从旧数据库迁移缓存
- 最多缓存 50,000 条（可配置）

---

## 完整搜索流程

### 用户查询："OpenClaw 记忆系统"

### 步骤 1：查询预处理
```typescript
// src/memory/manager.ts line 262
async search(query: string, opts?: SearchOptions) {
  const cleaned = query.trim();
  if (!cleaned) return [];

  // 同步脏数据（如果配置了 onSearch）
  if (this.config.sync.onSearch && this.dirty) {
    await this.sync({ reason: 'search' });
  }

  // 并行执行向量搜索和关键词搜索
  const [vectorResults, keywordResults] = await Promise.all([
    this.searchVector(cleaned, opts),
    this.searchKeyword(cleaned, opts)
  ]);

  // 混合合并结果
  return mergeHybridResults({
    vector: vectorResults,
    keyword: keywordResults,
    vectorWeight: 0.7,
    textWeight: 0.3
  });
}
```

---

### 步骤 2A：向量搜索

#### 2A.1 生成查询向量
```typescript
// 调用嵌入模型
const queryEmbedding = await embedProvider.embed(["OpenClaw 记忆系统"]);
// 返回：[0.523, -0.182, 0.745, ..., 0.091]  // 768 维向量
```

**本地模型流程**：
```
用户查询文本
    ↓
node-llama-cpp 加载 GGUF 模型
    ↓
C++ 推理引擎（Metal/CUDA/CPU）
    ↓
返回 768 维浮点数向量
```

#### 2A.2 向量数据库查询
```sql
-- src/memory/manager-search.ts line 34-43
SELECT c.id, c.path, c.start_line, c.end_line, c.text, c.source,
       vec_distance_cosine(v.embedding, ?) AS dist
  FROM chunks_vec v
  JOIN chunks c ON c.id = v.id
 WHERE c.model = ?
 ORDER BY dist ASC
 LIMIT 6;
```

**参数绑定**：
```javascript
params = [
  Buffer.from(new Float32Array(queryEmbedding).buffer),  // 查询向量
  "hf:ggml-org/embeddinggemma-300M-GGUF/...",           // 模型名
  6                                                      // 限制结果数
]
```

**SQL 执行过程**：
1. `chunks_vec` 虚拟表计算所有向量与查询向量的余弦距离
2. 使用 SIMD 加速（AVX2/NEON）并行计算
3. 按距离升序排序（最小距离 = 最相似）
4. 返回 top-6 结果

**返回数据**：
```javascript
[
  {
    id: "902df866...",
    path: "MEMORY.md",
    start_line: 1,
    end_line: 19,
    text: "# 我的记忆库\n\n## 项目信息...",
    source: "memory",
    dist: 0.234  // 余弦距离
  }
]
```

#### 2A.3 分数转换
```typescript
// src/memory/manager-search.ts line 64
score = 1 - dist  // 0.766 (距离 0.234 转为相似度)
```

**余弦距离 → 余弦相似度**：
- 距离 0 → 相似度 1.0（完全相同）
- 距离 1 → 相似度 0.0（正交）
- 距离 2 → 相似度 -1.0（完全相反）

---

### 步骤 2B：关键词搜索

#### 2B.1 构建 FTS 查询
```typescript
// src/memory/hybrid.ts line 23-32
function buildFtsQuery(raw: string): string | null {
  const tokens = raw.match(/[A-Za-z0-9_]+/g) ?? [];
  if (tokens.length === 0) return null;
  const quoted = tokens.map(t => `"${t}"`);
  return quoted.join(" AND ");
}

buildFtsQuery("OpenClaw 记忆系统")
// 返回：""OpenClaw" AND "记忆" AND "系统""
```

**查询预处理**：
- 提取字母、数字、下划线的词元
- 过滤中文（FTS5 默认不支持中文分词）
- 每个词元用双引号包裹（精确匹配）
- 使用 AND 连接（所有词都要出现）

#### 2B.2 FTS5 查询
```sql
-- src/memory/manager-search.ts line 150-157
SELECT id, path, source, start_line, end_line, text,
       bm25(chunks_fts) AS rank
  FROM chunks_fts
 WHERE chunks_fts MATCH ? AND model = ?
 ORDER BY rank ASC
 LIMIT 24;  -- candidateMultiplier × maxResults (4 × 6)
```

**参数**：
```javascript
params = [
  '"OpenClaw" AND "记忆" AND "系统"',  // FTS 查询
  "hf:ggml-org/...",                   // 模型名
  24                                    // 获取更多候选（过滤后返回 6 个）
]
```

**BM25 算法**：
```
BM25(D, Q) = Σ IDF(qi) × (f(qi, D) × (k1 + 1)) / (f(qi, D) + k1 × (1 - b + b × |D| / avgdl))

其中：
- D = 文档（文本块）
- Q = 查询（词元集合）
- f(qi, D) = 词元 qi 在文档 D 中的频率
- IDF(qi) = 逆文档频率（稀有词权重更高）
- k1 = 1.2（词频饱和参数）
- b = 0.75（长度归一化）
```

**返回数据**：
```javascript
[
  {
    id: "902df866...",
    path: "MEMORY.md",
    text: "...OpenClaw...记忆...系统...",
    rank: -2.345  // 负数 = 更相关
  }
]
```

#### 2B.3 BM25 分数归一化
```typescript
// src/memory/hybrid.ts line 34-37
function bm25RankToScore(rank: number): number {
  const normalized = Math.max(0, rank);  // -2.345 → 2.345
  return 1 / (1 + normalized);           // 0.299
}
```

**归一化公式**：
```
score = 1 / (1 + |rank|)

rank = -10 → score = 0.091
rank = -2  → score = 0.333
rank = -1  → score = 0.5
rank = 0   → score = 1.0
```

---

### 步骤 3：混合结果合并

```typescript
// src/memory/hybrid.ts line 39-111
function mergeHybridResults(params: {
  vector: HybridVectorResult[],
  keyword: HybridKeywordResult[],
  vectorWeight: 0.7,
  textWeight: 0.3
}) {
  const byId = new Map();

  // 1. 添加向量搜索结果
  for (const r of params.vector) {
    byId.set(r.id, {
      ...r,
      vectorScore: r.score,  // 0.766
      textScore: 0
    });
  }

  // 2. 合并关键词搜索结果
  for (const r of params.keyword) {
    const existing = byId.get(r.id);
    if (existing) {
      existing.textScore = r.score;  // 0.299
    } else {
      byId.set(r.id, {
        ...r,
        vectorScore: 0,
        textScore: r.score
      });
    }
  }

  // 3. 计算加权分数
  const merged = Array.from(byId.values()).map(entry => {
    const score =
      params.vectorWeight * entry.vectorScore +
      params.textWeight * entry.textScore;
    // 0.7 × 0.766 + 0.3 × 0.299 = 0.626
    return { ...entry, score };
  });

  // 4. 按分数降序排序
  return merged.sort((a, b) => b.score - a.score);
}
```

**合并示例**：

| Chunk ID | 向量分数 | 关键词分数 | 最终分数 | 排名 |
|----------|---------|-----------|---------|-----|
| abc123   | 0.850   | 0.450     | 0.730   | 1   |
| def456   | 0.766   | 0.299     | 0.626   | 2   |
| ghi789   | 0.620   | 0.000     | 0.434   | 3   |
| jkl012   | 0.000   | 0.550     | 0.165   | 4   |

**计算公式**：
```
final_score = 0.7 × vector_score + 0.3 × text_score
```

---

### 步骤 4：过滤和返回

```typescript
// src/memory/manager.ts line 290-295
const filtered = merged.filter(r => r.score >= minScore);  // 默认 0.35
const topResults = filtered.slice(0, maxResults);          // 默认 6

return topResults.map(r => ({
  path: r.path,
  startLine: r.startLine,
  endLine: r.endLine,
  score: r.score,
  snippet: truncate(r.text, 700),  // 截断到 700 字符
  source: r.source
}));
```

**最终返回**：
```json
{
  "results": [
    {
      "path": "MEMORY.md",
      "startLine": 1,
      "endLine": 19,
      "score": 0.626,
      "snippet": "# 我的记忆库\n\n## 项目信息\n- **OpenClaw 代码位置**: ...",
      "source": "memory"
    }
  ]
}
```

---

## 索引创建流程

### 触发场景

#### 1. 手动触发
```bash
pnpm openclaw memory index
```

#### 2. 文件监控触发
```typescript
// src/memory/manager.ts line 812-850
chokidar.watch([
  "MEMORY.md",
  "memory.md",
  "memory/*.md",
  ...extraPaths
], {
  ignoreInitial: true,
  awaitWriteFinish: {
    stabilityThreshold: 1500,  // 等待 1.5 秒确保写入完成
    pollInterval: 100
  }
})
.on('add', () => this.markDirty())
.on('change', () => this.markDirty())
.on('unlink', () => this.markDirty());
```

**防抖机制**：
```
文件修改
   ↓ (等待 1.5s，期间若有新修改则重置计时器)
检测到稳定
   ↓
标记 dirty = true
   ↓ (下次搜索或会话启动时)
触发同步
```

#### 3. 会话启动触发
```typescript
// src/memory/manager.ts line 252
if (config.sync.onSessionStart && this.dirty) {
  await this.sync({ reason: 'session-start' });
}
```

#### 4. 搜索前触发
```typescript
// src/memory/manager.ts line 262
if (config.sync.onSearch && this.dirty) {
  await this.sync({ reason: 'search' });
}
```

---

### 同步流程

```typescript
// src/memory/manager.ts line 383-775
async sync(opts: { reason: string, force?: boolean }) {
  // 1. 检查是否需要完整重索引
  const needsReindex =
    opts.force ||
    providerChanged ||
    modelChanged ||
    chunkingChanged;

  if (needsReindex) {
    return await this.runSafeReindex();
  }

  // 2. 增量索引
  const memoryFiles = await listMemoryFiles(workspace);
  const sessionFiles = config.sessionMemory
    ? await listSessionFiles(agentId)
    : [];

  // 3. 检测变化
  const toIndex = [];
  const toDelete = [];

  for (const file of memoryFiles) {
    const hash = await hashFile(file);
    const existing = db.get("SELECT hash FROM files WHERE path = ?", file);

    if (!existing || existing.hash !== hash) {
      toIndex.push(file);
    }
  }

  for (const existing of db.all("SELECT path FROM files")) {
    if (!memoryFiles.includes(existing.path)) {
      toDelete.push(existing.path);
    }
  }

  // 4. 删除过期块
  for (const path of toDelete) {
    db.run("DELETE FROM chunks WHERE path = ?", path);
    db.run("DELETE FROM files WHERE path = ?", path);
    db.run("DELETE FROM chunks_vec WHERE id IN (SELECT id FROM chunks WHERE path = ?)", path);
  }

  // 5. 索引新文件/变更文件
  for (const file of toIndex) {
    const content = await fs.readFile(file, 'utf-8');
    const chunks = chunkMarkdown(content, {
      tokens: 400,
      overlap: 80
    });

    for (const chunk of chunks) {
      const id = sha256(chunk.text);

      // 检查缓存
      let embedding = await getFromCache(id);

      if (!embedding) {
        // 调用嵌入模型
        embedding = await embedProvider.embed([chunk.text]);
        await saveToCache(id, embedding);
      }

      // 保存到数据库
      db.run(`
        INSERT OR REPLACE INTO chunks
        (id, path, source, start_line, end_line, hash, model, text, embedding, updated_at)
        VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
      `, [id, file, 'memory', chunk.startLine, chunk.endLine,
          chunk.hash, model, chunk.text, JSON.stringify(embedding), Date.now()]);

      // 更新向量索引
      db.run(`
        INSERT OR REPLACE INTO chunks_vec (id, embedding)
        VALUES (?, ?)
      `, [id, Buffer.from(new Float32Array(embedding).buffer)]);

      // 更新全文索引
      db.run(`
        INSERT OR REPLACE INTO chunks_fts
        (text, id, path, source, model, start_line, end_line)
        VALUES (?, ?, ?, ?, ?, ?, ?)
      `, [chunk.text, id, file, 'memory', model, chunk.startLine, chunk.endLine]);
    }

    // 更新文件记录
    const hash = await hashFile(file);
    const stat = await fs.stat(file);
    db.run(`
      INSERT OR REPLACE INTO files (path, source, hash, mtime, size)
      VALUES (?, ?, ?, ?, ?)
    `, [file, 'memory', hash, stat.mtimeMs, stat.size]);
  }

  this.dirty = false;
}
```

---

### 原子性重索引

```typescript
// src/memory/manager.ts line 1351-1445
async runSafeReindex() {
  const tmpDbPath = `${this.dbPath}.tmp-${uuid()}`;

  try {
    // 1. 创建临时数据库
    const tmpDb = new DatabaseSync(tmpDbPath);
    await ensureMemoryIndexSchema(tmpDb);

    // 2. 播种嵌入缓存（从旧数据库迁移）
    await seedEmbeddingCache({
      sourceDb: this.db,
      targetDb: tmpDb
    });

    // 3. 在临时数据库上构建新索引
    await this.syncImpl({
      db: tmpDb,
      force: true,
      progress: opts.progress
    });

    // 4. 原子性替换
    this.db.close();
    fs.renameSync(tmpDbPath, this.dbPath);
    this.db = new DatabaseSync(this.dbPath);

  } catch (err) {
    // 5. 失败时清理
    if (fs.existsSync(tmpDbPath)) {
      fs.unlinkSync(tmpDbPath);
    }
    throw err;
  }
}
```

**原子性保证**：
- 使用临时文件避免破坏原有索引
- 只有完全成功后才替换主数据库
- 失败时自动回滚，旧索引不受影响
- `fs.renameSync()` 是原子操作（POSIX 保证）

---

## 文本分块算法

```typescript
// src/memory/internal.ts line 144-195
function chunkMarkdown(content: string, opts: {
  tokens: 400,
  overlap: 80
}): Array<{ startLine, endLine, text, hash }> {
  const maxChars = opts.tokens * 4;  // 400 × 4 = 1600 字符
  const overlapChars = opts.overlap * 4;  // 80 × 4 = 320 字符

  const lines = content.split('\n');
  const chunks = [];
  let buffer = [];
  let bufferChars = 0;
  let startLine = 1;

  for (let i = 0; i < lines.length; i++) {
    const line = lines[i];
    buffer.push(line);
    bufferChars += line.length;

    // 达到最大长度
    if (bufferChars >= maxChars) {
      // 尝试在段落边界分割
      let splitIndex = buffer.length;
      for (let j = buffer.length - 1; j >= 0; j--) {
        if (buffer[j].trim() === '' || buffer[j].startsWith('#')) {
          splitIndex = j;
          break;
        }
      }

      const chunkText = buffer.slice(0, splitIndex).join('\n');
      chunks.push({
        startLine,
        endLine: startLine + splitIndex - 1,
        text: normalize(chunkText),
        hash: sha256(chunkText)
      });

      // 重叠部分
      const overlapLines = Math.floor(overlapChars / 80);  // 估算行数
      buffer = buffer.slice(Math.max(0, splitIndex - overlapLines));
      startLine = startLine + splitIndex - buffer.length;
      bufferChars = buffer.join('\n').length;
    }
  }

  // 最后一个块
  if (buffer.length > 0) {
    chunks.push({
      startLine,
      endLine: lines.length,
      text: normalize(buffer.join('\n')),
      hash: sha256(buffer.join('\n'))
    });
  }

  return chunks;
}
```

**分块示例**：
```
原始文件（2000 字符）：
┌─────────────────────────────┐
│ # 标题 1                    │ ← 第 1-5 行
│ 内容 A                      │
│                             │
│ ## 子标题                   │
│ 内容 B                      │
├─────────────────────────────┤ ← 第一个块（1-15 行）
│ 内容 C                      │ ← 第 6-10 行（重叠）
│                             │
│ # 标题 2                    │
│ 内容 D                      │
│ 内容 E                      │
├─────────────────────────────┤ ← 第二个块（10-25 行）
│ 内容 F                      │ ← 第 21-25 行（重叠）
│                             │
│ # 标题 3                    │
│ 内容 G                      │
└─────────────────────────────┘ ← 第三个块（21-30 行）
```

**为什么要重叠**：
- 避免在句子/段落中间分割
- 捕获跨块的上下文关系
- 提高搜索召回率

---

## 嵌入缓存机制

```typescript
// src/memory/manager.ts line 720-755
async seedEmbeddingCache(params: {
  sourceDb: DatabaseSync,
  targetDb: DatabaseSync
}) {
  // 从旧数据库读取所有缓存
  const cached = params.sourceDb.prepare(`
    SELECT provider, model, provider_key, hash, embedding, dims, updated_at
    FROM embedding_cache
  `).all();

  // 插入到新数据库
  const stmt = params.targetDb.prepare(`
    INSERT OR REPLACE INTO embedding_cache
    (provider, model, provider_key, hash, embedding, dims, updated_at)
    VALUES (?, ?, ?, ?, ?, ?, ?)
  `);

  for (const entry of cached) {
    stmt.run([
      entry.provider,
      entry.model,
      entry.provider_key,
      entry.hash,
      entry.embedding,
      entry.dims,
      entry.updated_at
    ]);
  }
}
```

**缓存键计算**：
```typescript
// src/memory/provider-key.ts
function computeProviderKey(config: {
  provider: string,
  model: string,
  baseUrl?: string,
  headers?: Record<string, string>
}): string {
  const normalized = {
    provider: config.provider,
    baseUrl: config.baseUrl,
    model: config.model,
    headerNames: Object.keys(config.headers || {}).sort()
  };
  return sha256(JSON.stringify(normalized));
}
```

**缓存查找流程**：
```
1. 计算文本块哈希：hash = sha256(text)
2. 查询缓存：
   SELECT embedding
   FROM embedding_cache
   WHERE provider = ?
     AND model = ?
     AND provider_key = ?
     AND hash = ?
3. 命中 → 直接返回
4. 未命中 → 调用嵌入模型 → 保存到缓存
```

**缓存淘汰**：
```typescript
// 超过最大条目数时删除最旧的条目
if (cacheSize > maxEntries) {
  db.run(`
    DELETE FROM embedding_cache
    WHERE rowid IN (
      SELECT rowid FROM embedding_cache
      ORDER BY updated_at ASC
      LIMIT ?
    )
  `, [cacheSize - maxEntries]);
}
```

---

## 性能优化技巧

### 1. 批量嵌入
```typescript
// 单次嵌入：调用 API 100 次
for (const chunk of chunks) {
  const embedding = await embed([chunk.text]);
}

// 批量嵌入：调用 API 1 次
const texts = chunks.map(c => c.text);
const embeddings = await embed(texts);  // OpenAI 支持批量
```

**OpenAI 批量 API**：
- 最多 2048 个文本/请求
- 显著降低延迟（100 次 → 1 次）
- 降低成本（批量折扣）

### 2. 向量索引加速
```c
// sqlite-vec 使用 SIMD 指令加速
// AVX2 示例（一次计算 8 个浮点数）
__m256 a = _mm256_loadu_ps(&vec1[i]);
__m256 b = _mm256_loadu_ps(&vec2[i]);
__m256 prod = _mm256_mul_ps(a, b);
dotProduct += _mm256_reduce_add_ps(prod);
```

**性能对比**：
- JavaScript 循环：~10,000 向量/秒
- sqlite-vec (SIMD)：~1,000,000 向量/秒
- 加速比：100x

### 3. 增量索引
```typescript
// 只处理变更的文件
const changed = files.filter(f =>
  !existingFiles.has(f.path) ||
  existingFiles.get(f.path).hash !== f.hash
);

// 只删除不存在的文件
const deleted = existingFiles.filter(f =>
  !files.has(f.path)
);
```

**效果**：
- 完整重索引：1000 文件 × 5 秒 = 1.4 小时
- 增量索引：10 变更文件 × 5 秒 = 50 秒

### 4. 本地嵌入 vs 远程 API

| 维度 | 本地模型 | OpenAI API |
|------|----------|------------|
| **首次启动** | 慢（下载 328MB） | 快（无需下载） |
| **嵌入速度** | 20-50 块/秒 | 200-500 块/秒（批量） |
| **成本** | 免费 | $0.00002/1K tokens |
| **隐私** | 完全离线 | 数据上传到 OpenAI |
| **质量** | 中等（768 维） | 优秀（1536 维） |

**推荐**：
- 开发/测试：本地模型
- 生产/大规模：OpenAI API

---

## 故障排查实战

### 场景 1：搜索无结果

**检查清单**：
```bash
# 1. 检查索引是否创建
ls -lh ~/.openclaw/memory/main.sqlite

# 2. 检查文件是否被索引
sqlite3 ~/.openclaw/memory/main.sqlite "SELECT * FROM files;"

# 3. 检查块数量
sqlite3 ~/.openclaw/memory/main.sqlite "SELECT COUNT(*) FROM chunks;"

# 4. 检查向量索引
sqlite3 ~/.openclaw/memory/main.sqlite "SELECT COUNT(*) FROM chunks_vec;"

# 5. 手动搜索测试
pnpm openclaw memory search "测试" --min-score 0.1
```

### 场景 2：索引速度慢

**诊断**：
```bash
# 检查嵌入提供商
pnpm openclaw memory status --deep

# 查看批处理状态
sqlite3 ~/.openclaw/memory/main.sqlite "SELECT value FROM meta WHERE key = 'memory_index_meta_v1';"
```

**优化**：
```json
{
  "memorySearch": {
    "provider": "local",  // 切换到本地模型
    "chunking": {
      "tokens": 300  // 减小块大小
    }
  }
}
```

### 场景 3：向量扩展加载失败

**错误信息**：
```
Vector: unavailable
Vector error: dlopen failed: image not found
```

**解决**：
```bash
# 重新安装 sqlite-vec
cd /Users/feizhan/code/openclaw
pnpm rebuild sqlite-vec

# 或指定扩展路径
{
  "memorySearch": {
    "store": {
      "vector": {
        "extensionPath": "/path/to/vec0.dylib"
      }
    }
  }
}
```

---

## 总结：数据流全景图

```
用户创建 MEMORY.md
        ↓
文件监控检测变化 (chokidar)
        ↓
标记 dirty = true
        ↓
下次搜索/会话启动时触发同步
        ↓
读取文件内容 → 计算哈希 → 检测变化
        ↓
分块 (400 tokens, 80 overlap)
        ↓
嵌入模型生成向量 (768 维)
        ↓
保存到 SQLite
   ├→ chunks 表（文本 + 向量）
   ├→ chunks_vec 虚拟表（向量索引）
   ├→ chunks_fts 虚拟表（全文索引）
   └→ embedding_cache（缓存）
        ↓
用户发起搜索
        ↓
并行执行
   ├→ 向量搜索（余弦相似度）
   └→ 关键词搜索（BM25）
        ↓
混合合并（0.7 × 向量 + 0.3 × 关键词）
        ↓
过滤低分结果（< 0.35）
        ↓
返回 top-6 结果
```

---

## 相关文件位置

| 组件 | 文件路径 |
|------|---------|
| **搜索流程** | `src/memory/manager-search.ts` |
| **混合算法** | `src/memory/hybrid.ts` |
| **分块逻辑** | `src/memory/internal.ts` |
| **数据库架构** | `src/memory/memory-schema.ts` |
| **嵌入提供商** | `src/memory/embeddings.ts` |
| **文件监控** | `src/memory/manager.ts` (line 812) |
| **CLI 命令** | `src/cli/memory-cli.ts` |
| **数据库** | `~/.openclaw/memory/main.sqlite` |
| **源文件** | `~/.openclaw/workspace/MEMORY.md` |

---

现在你已经完全理解了 OpenClaw 记忆系统的内部运作机制！🎉
