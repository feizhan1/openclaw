# OpenClaw 历史会话使用机制详解

## 📊 当前状态概览

你的会话数据：
```
会话文件: ~/.openclaw/agents/main/sessions/
  ├── 42ef286d-4b48-477a-9dae-c4fb27a525de.jsonl (428KB, 292行)
  └── b97c670c-40ae-4fb0-8c79-bddc5ac0acce.jsonl (52KB, 60行)

会话索引: ❌ 未启用 (sessionMemory: false)
```

## 🎯 OpenClaw 利用历史会话的 5 种方式

### 1. **当前会话的上下文窗口** ✅ 正在使用

#### 工作原理
```typescript
// src/agents/pi-embedded-runner/run/attempt.ts
const sessionManager = SessionManager.open(sessionFile);

// 加载历史消息到 LLM 上下文
const history = sessionManager.getHistory();
const messages = [
  systemPrompt,
  ...history,  // ← 历史对话
  newUserMessage
];

// 发送给 LLM
const response = await llm.chat(messages);
```

#### 实际效果
```
轮次 1:
你: 我在开发 OpenClaw 项目
AI: 好的，了解了 ✅

轮次 2:
你: 项目用什么语言？
AI: 根据你刚才说的，OpenClaw 是 TypeScript 项目 ✅
    (从 JSONL 加载了轮次 1 的上下文)

轮次 3:
你: 数据库用什么？
AI: OpenClaw 使用 SQLite 存储记忆索引 ✅
    (仍然记得这是 OpenClaw 项目)
```

#### 上下文窗口限制
```
Claude Sonnet 4.5: ~200K tokens
Opus 4.5: ~200K tokens

你的会话: 428KB ≈ 100K tokens
状态: ✅ 在窗口内
```

---

### 2. **会话恢复和继续** ✅ 支持

#### 使用方式
```bash
# 查看所有会话
pnpm openclaw sessions list

# 继续之前的会话
pnpm openclaw chat --session 42ef286d-4b48-477a-9dae-c4fb27a525de
```

#### 工作原理
```typescript
// src/config/sessions/store.ts
function loadSessionStore(storePath: string) {
  // 1. 读取 sessions.json（会话元数据）
  const store = JSON5.parse(fs.readFileSync(storePath));

  // 2. 找到会话 ID 对应的 JSONL 文件
  const entry = store[sessionKey];
  const sessionFile = entry.sessionFile; // *.jsonl

  // 3. 加载完整对话历史
  const manager = SessionManager.open(sessionFile);
  return manager.getHistory();
}
```

#### 会话元数据（sessions.json）
```json
{
  "42ef286d-4b48-477a-9dae-c4fb27a525de": {
    "sessionId": "42ef286d-4b48-477a-9dae-c4fb27a525de",
    "sessionFile": "~/.openclaw/agents/main/sessions/42ef286d....jsonl",
    "channel": "cli",
    "createdAt": "2026-02-05T03:26:00Z",
    "updatedAt": "2026-02-05T03:26:45Z",
    "messageCount": 288
  }
}
```

---

### 3. **AI 工具：查询其他会话** ✅ 支持

#### sessions_history 工具
AI 可以主动查询其他会话的历史：

```typescript
// src/agents/tools/sessions-history-tool.ts
{
  name: "sessions_history",
  description: "Fetch message history for a session.",
  parameters: {
    sessionKey: string,      // 会话 ID
    limit?: number,          // 返回消息数
    includeTools?: boolean   // 是否包含工具调用
  }
}
```

#### 使用示例
```
你: 我之前讨论过什么主题？
AI: 让我查询你的历史会话...
    → 调用 sessions_history 工具
    → 扫描所有会话的消息
    → 返回：你讨论过记忆系统、JSONL 格式、向量搜索等
```

#### 安全限制
```typescript
// 沙箱模式下的访问控制
if (sandboxed && visibility === "spawned") {
  // 只能访问自己创建的子会话
  const ok = await isSpawnedSessionAllowed({
    requesterSessionKey,
    targetSessionKey
  });
  if (!ok) return { error: "Session not visible" };
}
```

---

### 4. **可选：会话记忆索引** ❌ 你未启用

#### 如果启用后的效果
```json
{
  "memorySearch": {
    "sources": ["memory", "sessions"],
    "experimental": { "sessionMemory": true }
  }
}
```

#### 工作原理
```
会话文件 (JSONL)
    ↓
提取所有 user/assistant 消息
    ↓
分块 (400 tokens)
    ↓
生成向量嵌入
    ↓
存入 SQLite (与 MEMORY.md 一起)
    ↓
memory_search 可以搜索历史对话
```

#### 搜索对比

**未启用 sessionMemory**：
```
memory_search("向量搜索")
→ 只搜索 MEMORY.md
→ 返回：你手动写的笔记
```

**启用后**：
```
memory_search("向量搜索")
→ 搜索 MEMORY.md + 所有历史对话
→ 返回：
  1. MEMORY.md 中的笔记
  2. 3 天前你和 AI 讨论向量搜索的对话
  3. 上周提到余弦相似度的会话
```

#### 索引更新触发
```typescript
// src/memory/manager.ts line 867-927
async processSessionDeltaBatch() {
  for (const sessionFile of pendingFiles) {
    const delta = await updateSessionDelta(sessionFile);

    // 检查增量阈值
    if (delta.pendingBytes >= 100_000 ||      // 100KB
        delta.pendingMessages >= 50) {         // 50 条消息

      // 触发重新索引
      await this.sync({ reason: 'session-delta' });
    }
  }
}
```

---

### 5. **会话压缩（Compaction）** ✅ 自动

#### 为什么需要压缩？
```
问题：上下文窗口有限 (200K tokens)
对话越来越长 → 超出窗口 → 无法继续

解决：智能压缩历史对话
```

#### 压缩策略
```typescript
// src/agents/pi-embedded-runner/compact.ts
async compact(sessionFile: string, opts: CompactOptions) {
  const history = SessionManager.open(sessionFile).getHistory();

  // 1. 保留最近的消息（不压缩）
  const recent = history.slice(-opts.keepRecent);  // 默认 10 条

  // 2. 压缩旧消息
  const toCompress = history.slice(0, -opts.keepRecent);
  const summary = await llm.summarize(toCompress, {
    prompt: "总结以下对话的关键信息、决策和上下文"
  });

  // 3. 生成新的会话历史
  return [
    { role: "system", content: `[压缩的历史]: ${summary}` },
    ...recent
  ];
}
```

#### 压缩效果
```
压缩前 (180K tokens):
[288 条完整消息]

压缩后 (50K tokens):
[系统消息: "之前讨论了记忆系统架构、JSONL 格式、向量搜索等..."]
[最近 10 条完整消息]

节省: 130K tokens
效果: 可以继续对话了！
```

#### 触发条件
```typescript
// src/agents/pi-embedded-runner/run/attempt.ts
if (tokenCount > maxTokens * 0.8) {  // 达到 80% 阈值
  await compact(sessionFile);
}
```

---

## 📈 会话使用流程图

```
用户发送消息
    ↓
[1] 加载会话历史 (JSONL → 内存)
    ↓
[2] 检查上下文窗口
    ├─ 未满 → 直接使用
    └─ 接近满 → 压缩旧消息
    ↓
[3] 构建 LLM 输入
    [系统提示]
    [历史消息 1-287]
    [新用户消息]
    ↓
[4] LLM 生成回复
    ↓
[5] 追加到 JSONL
    {"type":"message","role":"assistant","content":"..."}
    ↓
[6] 可选：触发记忆索引
    (如果启用 sessionMemory 且超过阈值)
    ↓
[7] 返回给用户
```

---

## 🔍 实际数据分析

### 你的会话统计
```bash
# 查看消息分布
$ grep '"type":"message"' ~/.openclaw/agents/main/sessions/*.jsonl | wc -l
288

# 查看用户消息 vs AI 回复
$ grep '"role":"user"' *.jsonl | wc -l
144  # 用户消息

$ grep '"role":"assistant"' *.jsonl | wc -l
144  # AI 回复

# 查看会话持续时间
$ head -1 42ef286d-4b48-477a-9dae-c4fb27a525de.jsonl | jq .timestamp
"2026-02-04T19:26:00Z"

$ tail -1 42ef286d-4b48-477a-9dae-c4fb27a525de.jsonl | jq .timestamp
"2026-02-05T03:26:45Z"

持续时间: 8 小时
```

### 会话内容分析
```bash
# 提取所有用户消息主题
$ grep '"role":"user"' *.jsonl | jq -r '.message.content[0].text' | head -10

输出：
"这个项目的长期记忆是怎么处理的"
"如何触发索引创建"
"我没有发现数据库文件"
"创建/修改 MEMORY.md是自动完成的吗？"
"和AI的交流会自动写入MEMORY.md吗？"
"jsonl是什么？有什么作用"
"保存的对话用来做什么？"
"用Postgres数据库保存，不是比jsonl更好吗？"
```

**主题分析**：你主要讨论了记忆系统、JSONL 格式、数据存储等话题。

---

## 🎯 未来可能的使用方式（当前未实现）

### 1. 跨会话搜索
```
功能：在所有历史会话中搜索关键词
命令：openclaw sessions search "向量搜索"
状态：未实现（但可以用 grep）

手动实现：
$ grep -r "向量搜索" ~/.openclaw/agents/main/sessions/*.jsonl
```

### 2. 会话导出和分析
```
功能：导出会话为 Markdown/HTML
命令：openclaw sessions export --format markdown
状态：未实现

手动实现：
$ jq -r 'select(.type=="message") |
  "\(.message.role): \(.message.content[0].text)"' \
  session.jsonl > export.txt
```

### 3. 会话合并
```
功能：合并多个相关会话
命令：openclaw sessions merge session1 session2
状态：未实现
```

### 4. 自动标签和分类
```
功能：AI 自动为会话添加标签
示例：[技术讨论] [记忆系统] [2026-02-05]
状态：未实现
```

---

## 🔑 关键发现总结

| 功能 | 状态 | 用途 |
|------|------|------|
| **当前会话上下文** | ✅ 活跃 | AI 记住本次对话 |
| **会话恢复** | ✅ 支持 | 继续之前的对话 |
| **跨会话查询工具** | ✅ 支持 | AI 可查询其他会话 |
| **会话记忆索引** | ❌ 未启用 | 可语义搜索历史对话 |
| **会话压缩** | ✅ 自动 | 防止上下文溢出 |
| **JSONL 存储** | ✅ 活跃 | 持久化所有对话 |

---

## 💡 推荐配置

### 场景 A：当前方式（推荐大多数用户）
```json
{
  "memorySearch": {
    "sources": ["memory"],
    "experimental": { "sessionMemory": false }
  }
}
```

**优点**：
- ✅ 干净、可控
- ✅ 索引小、速度快
- ✅ MEMORY.md 精选知识

**缺点**：
- ❌ 不能搜索历史对话
- ❌ 需要手动摘要重要信息

---

### 场景 B：启用会话索引（高级用户）
```json
{
  "memorySearch": {
    "sources": ["memory", "sessions"],
    "experimental": { "sessionMemory": true },
    "sync": {
      "sessions": {
        "deltaBytes": 100000,
        "deltaMessages": 50
      }
    }
  }
}
```

**优点**：
- ✅ 可搜索所有历史对话
- ✅ AI 自动记住讨论细节

**缺点**：
- ❌ 索引变大（480KB → ~5MB）
- ❌ 索引速度慢
- ❌ 可能包含无关信息

---

## 🧪 动手实验

### 实验 1：查看会话工具
```bash
# 在对话中问 AI
"列出我的所有会话"

# AI 会调用 sessions_list 工具
→ 返回你的 2 个会话及其元数据
```

### 实验 2：查询历史会话
```bash
"查询我在会话 42ef286d 中讨论了什么"

# AI 会调用 sessions_history 工具
→ 返回那个会话的消息列表
```

### 实验 3：手动分析会话
```bash
# 提取关键对话
jq -r 'select(.type=="message" and .message.role=="user") |
  .message.content[0].text' \
  ~/.openclaw/agents/main/sessions/42ef286d-*.jsonl \
  | head -20

# 统计对话轮次
grep -c '"type":"message"' *.jsonl

# 查找特定主题
grep -i "记忆系统" *.jsonl | jq -r '.message.content[0].text'
```

---

## 总结：OpenClaw 如何利用历史会话

```
✅ 实时使用（当前会话）
   → 加载到 LLM 上下文窗口
   → AI 记住本次对话的所有内容

✅ 持久化存储（JSONL）
   → 所有对话自动保存
   → 可恢复和继续

✅ 跨会话查询（AI 工具）
   → sessions_history 工具
   → AI 可主动查询其他会话

⚠️ 可选索引（未启用）
   → sessionMemory 功能
   → 可语义搜索历史对话

✅ 智能压缩（自动）
   → 防止上下文溢出
   → 保留关键信息
```

**核心设计理念**：历史会话主要用于**恢复和查询**，而不是主动搜索。如果需要长期记住，应该手动摘要到 MEMORY.md。🎯
