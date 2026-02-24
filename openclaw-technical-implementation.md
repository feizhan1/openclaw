# OpenClaw 项目深度调研报告

## 调研概述

本报告深入分析了 OpenClaw 项目中两个最核心的技术实现:
1. **系统提示词机制** - 如何动态构建和注入上下文化的指令
2. **持久化记忆系统** - 如何存储、检索和管理长期/短期记忆

## 一、系统提示词实现机制

### 1.1 核心架构

OpenClaw 采用**分层动态组装**的系统提示词架构,而非静态模板。每次 Agent 运行时都会根据以下因素重新生成提示词:

**主要构建函数**:
- `/src/agents/system-prompt.ts` - `buildAgentSystemPrompt()` (592 行)
- `/src/agents/pi-embedded-runner/system-prompt.ts` - `buildEmbeddedSystemPrompt()`

**关键特征**:
- **条件性段落** - 根据上下文包含/排除不同部分
- **多级配置** - 支持全局、群组、话题级别的提示词覆盖
- **运行时动态注入** - 工具列表、运行时信息、用户身份等动态生成

### 1.2 工作区引导文件系统

用户可以在工作区目录放置特定文件来影响 Agent 行为:

```
~/.clawdbot/agents/<agentId>/
├── AGENTS.md       # Agent 行为指南
├── SOUL.md         # 人格化语调定义
├── TOOLS.md        # 工具使用指导
├── IDENTITY.md     # 身份和角色设置
├── USER.md         # 用户偏好
├── HEARTBEAT.md    # 定期检查任务
├── BOOTSTRAP.md    # 初始化上下文
└── MEMORY.md       # 长期记忆(索引到向量数据库)
```

这些文件通过 `resolveBootstrapContextForRun()` 加载并注入到系统提示词的 "# Project Context" 部分。

### 1.3 Prompt Mode 系统

OpenClaw 支持三种提示词模式,适应不同的 Agent 类型:

```typescript
type PromptMode = "full" | "minimal" | "none";

// full - 主 Agent (包含所有段落)
// minimal - 子 Agent (仅基础工具、工作区、运行时)
// none - 仅身份行
```

**判断逻辑** (在 `/src/agents/pi-embedded-runner/run/attempt.ts:328`):
```typescript
const promptMode = isSubagentSessionKey(params.sessionKey) ? "minimal" : "full";
```

### 1.4 条件段落组装

系统提示词由多个 `build*Section()` 函数组装,每个返回字符串数组:

| 段落 | 条件 | 包含内容 |
|-----|------|---------|
| Skills Section | `!isMinimal && skillsPrompt` | 可用技能的 XML 列表 |
| Memory Section | `!isMinimal && hasMemoryTools` | 记忆搜索指导 |
| User Identity | `!isMinimal && ownerNumbers` | 主人的联系信息 |
| Time Section | `userTimezone` | 当前时间和时区 |
| Reply Tags | `!isMinimal` | 内联按钮等特殊标签 |
| Messaging | `messageToolHints` | 消息工具的使用指南 |
| Voice (TTS) | `ttsHint` | 语音合成提示 |
| Documentation | `!isMinimal && docsPath` | 项目文档路径 |
| Workspace | always | 工作目录路径 |
| Project Context | `contextFiles.length > 0` | SOUL.md 等文件内容 |
| Runtime | always | 运行时信息行 |

### 1.5 群组/渠道特定提示词

不同的消息渠道可以配置特定的系统提示词覆盖:

**Telegram 群组配置**:
```typescript
// /src/config/types.telegram.ts
type TelegramGroupConfig = {
  systemPrompt?: string;  // 群组级提示词
  skills?: string[];       // 群组可用技能过滤
};

type TelegramTopicConfig = {
  systemPrompt?: string;  // 话题级提示词
  skills?: string[];       // 话题可用技能
};
```

**注入流程** (在 `/src/telegram/bot-message-context.ts:545-551`):
```typescript
const systemPromptParts = [
  groupConfig?.systemPrompt?.trim() || null,
  topicConfig?.systemPrompt?.trim() || null,
].filter((entry): entry is string => Boolean(entry));

const groupSystemPrompt =
  systemPromptParts.length > 0
    ? systemPromptParts.join("\n\n")
    : undefined;
```

然后在 `/src/auto-reply/reply/get-reply-run.ts:181-182` 作为 `extraSystemPrompt` 注入到运行参数中。

### 1.6 运行时信息行

每个 Agent 运行都会包含一个标准化的运行时信息行:

```
Runtime: agent=work | host=hostname | repo=/repo | os=macOS (arm64) | node=v20 | model=anthropic/claude | channel=telegram | capabilities=inlineButtons | thinking=low
```

构建函数: `buildRuntimeLine()` (在 `/src/agents/system-prompt.ts:548-551`)

### 1.7 提示词版本管理

**技能快照版本** (在 `/src/auto-reply/reply/session-updates.ts:158-175`):
- 工作区技能有版本号追踪
- 当技能文件变化时版本递增
- Session 记录上次使用的版本号
- 版本不匹配时自动刷新技能列表

**配置变化检测**:
- 每次运行都重新生成系统提示词
- 确保反映最新的配置、文件和运行时状态

---

## 二、持久化记忆系统

### 2.1 分层存储架构

OpenClaw 采用**双层存储**设计,分别管理短期和长期记忆:

#### Layer 1: Agent Sessions (短期记忆)
- **存储位置**: `~/.clawdbot/agents/<agentId>/sessions/*.jsonl`
- **格式**: JSONL (每行一个 JSON 对象)
- **内容**: 完整的对话历史(用户消息、助手响应、工具调用、工具结果)
- **用途**: 会话上下文维护、对话历史回放

**JSONL 格式示例**:
```jsonl
{"type":"message","timestamp":"2024-01-31T10:00:00.000Z","message":{"role":"user","content":"What is OpenClaw?"}}
{"type":"message","timestamp":"2024-01-31T10:00:05.000Z","message":{"role":"assistant","content":[{"type":"text","text":"OpenClaw is..."}]}}
{"type":"message","timestamp":"2024-01-31T10:01:00.000Z","message":{"role":"toolResult","toolCallId":"call_123","content":"..."}}
```

#### Layer 2: Memory Index (长期记忆)
- **存储位置**: `~/.clawdbot/memory/<agentId>.sqlite`
- **格式**: SQLite3 数据库 + sqlite-vec 扩展
- **内容**: 向量化的记忆块(从 MEMORY.md 或会话历史提取)
- **用途**: 语义搜索、长期知识保留、跨会话记忆

**数据库表结构**:
```sql
-- 元数据
CREATE TABLE meta (
  key TEXT PRIMARY KEY,
  value TEXT NOT NULL
);

-- 追踪已索引的文件
CREATE TABLE files (
  path TEXT PRIMARY KEY,
  source TEXT NOT NULL DEFAULT 'memory',  -- 'memory' 或 'sessions'
  hash TEXT NOT NULL,
  mtime INTEGER NOT NULL,
  size INTEGER NOT NULL
);

-- 记忆块主表
CREATE TABLE chunks (
  id TEXT PRIMARY KEY,
  path TEXT NOT NULL,
  source TEXT NOT NULL DEFAULT 'memory',
  start_line INTEGER NOT NULL,
  end_line INTEGER NOT NULL,
  hash TEXT NOT NULL,
  model TEXT NOT NULL,  -- 嵌入模型
  text TEXT NOT NULL,   -- 原始文本
  embedding TEXT NOT NULL,  -- JSON array of floats
  updated_at INTEGER NOT NULL
);

-- 向量搜索表 (sqlite-vec 虚拟表)
CREATE VIRTUAL TABLE chunks_vec USING vec0(
  id TEXT PRIMARY KEY,
  embedding FLOAT[1536]  -- 或 768 for Gemini/Local
);

-- 全文搜索表 (FTS5)
CREATE VIRTUAL TABLE chunks_fts USING fts5(
  text,
  id UNINDEXED,
  path UNINDEXED,
  source UNINDEXED,
  model UNINDEXED,
  start_line UNINDEXED,
  end_line UNINDEXED
);

-- 嵌入缓存(避免重复计算)
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

### 2.2 混合搜索系统

OpenClaw 实现了业界领先的**向量 + 关键字混合搜索**:

**三种查询模式** (在 `/src/memory/manager.ts` 和 `/src/memory/hybrid.ts`):

1. **向量搜索 (Vector Search)**:
```typescript
async searchVector(queryVec: number[], limit: number): MemorySearchResult[]
```
- 使用 `vec_distance_cosine(embedding, queryVec)` 计算相似度
- 返回距离最近的 N 个结果
- 分数 = `1 - distance` (越接近 1 越相似)

2. **关键字搜索 (Keyword/FTS Search)**:
```typescript
async searchKeyword(query: string, limit: number): MemorySearchResult[]
```
- 使用 SQLite FTS5 的 BM25 算法
- 全文索引,支持词干、停用词过滤
- 返回 BM25 分数最高的 N 个结果

3. **混合搜索 (Hybrid - 默认)**:
```typescript
const merged = mergeHybridResults({
  vector: vectorResults,
  keyword: keywordResults,
  vectorWeight: 0.7,      // 可配置
  textWeight: 0.3         // 可配置
});
```

**默认配置** (在 `/src/agents/memory-search.ts`):
```typescript
type ResolvedMemorySearchConfig = {
  query: {
    maxResults: 6,           // 返回最多 6 条结果
    minScore: 0.35,          // 最低相似度 0.35
    hybrid: {
      enabled: true,
      vectorWeight: 0.7,     // 向量占 70% 权重
      textWeight: 0.3,       // 关键字占 30% 权重
      candidateMultiplier: 4 // 从向量和关键字各取 24 个候选
    }
  }
};
```

**搜索结果结构**:
```typescript
type MemorySearchResult = {
  path: string;        // "memory/MEMORY.md" or "sessions/session-123.jsonl"
  startLine: number;   // 块的起始行号
  endLine: number;     // 块的结束行号
  score: number;       // 综合相似度 (0-1)
  snippet: string;     // 文本片段 (max 700 chars)
  source: "memory" | "sessions"
};
```

### 2.3 向量嵌入系统

**支持的嵌入提供者** (在 `/src/memory/embeddings.ts`):

| 提供者 | 模型 | 维度 | 备注 |
|-------|------|------|------|
| OpenAI | `text-embedding-3-small` | 1536 | 默认选择 |
| Gemini | `gemini-embedding-001` | 768 | 支持批处理 API |
| Local | `embeddinggemma-300M` | 768 | 使用 node-llama-cpp |

**自动提供者选择** ("auto" 模式):
```typescript
// 优先级顺序:
1. 本地模型 (如果模型文件存在)
2. OpenAI (如果有 API 密钥)
3. Gemini (如果有 API 密钥)
4. Fallback 提供者
```

**批处理嵌入** (降低成本):
```typescript
const batchConfig = {
  enabled: true,
  wait: true,              // 等待批处理完成
  concurrency: 2,          // 并发批次数
  pollIntervalMs: 2000,    // 轮询间隔
  timeoutMinutes: 60       // 最多等待 60 分钟
};

// 超时设置
const EMBEDDING_QUERY_TIMEOUT_REMOTE_MS = 60_000;    // 1 分钟
const EMBEDDING_BATCH_TIMEOUT_REMOTE_MS = 120_000;   // 2 分钟
const EMBEDDING_QUERY_TIMEOUT_LOCAL_MS = 300_000;    // 5 分钟
const EMBEDDING_BATCH_TIMEOUT_LOCAL_MS = 600_000;    // 10 分钟
```

**嵌入缓存机制**:
- 基于文本哈希的缓存(避免重复调用 API)
- 支持缓存条目数量限制(LRU 策略)
- 自动修剪陈旧条目

### 2.4 记忆同步和索引

**同步策略** (在 `/src/memory/manager.ts`):

```typescript
sync: {
  onSessionStart: boolean;      // 会话开始时同步
  onSearch: boolean;            // 搜索前同步
  watch: boolean;               // 监控文件变化
  watchDebounceMs: 1500,        // 防抖延迟
  intervalMinutes: number,      // 定期同步间隔
  sessions: {
    deltaBytes: 100_000,        // 100KB 变化触发同步
    deltaMessages: 50           // 50 条消息变化触发同步
  }
};
```

**文件监控** (使用 `chokidar`):
- 监控 `memory/` 目录和 `MEMORY.md` 文件
- 自动检测新增、修改、删除
- 防抖处理(避免频繁同步)

**增量同步逻辑**:
1. 计算文件哈希(基于内容和 mtime)
2. 与数据库中的哈希比较
3. 仅处理变化的文件
4. 删除已删除文件的记录
5. 更新或插入新的 chunks

### 2.5 上下文压缩 (Compaction)

当对话历史接近上下文窗口限制时,自动触发压缩:

**核心算法** (在 `/src/agents/compaction.ts`):

```typescript
const BASE_CHUNK_RATIO = 0.4;        // 保留 40% 上下文用于历史
const MIN_CHUNK_RATIO = 0.15;        // 最小 15%
const SAFETY_MARGIN = 1.2;           // 20% 安全边际

// 自适应计算
function computeAdaptiveChunkRatio(
  messages: AgentMessage[],
  contextWindow: number
): number {
  const totalTokens = estimateMessagesTokens(messages);
  const avgTokens = totalTokens / messages.length;
  const safeAvgTokens = avgTokens * SAFETY_MARGIN;

  // 如果平均消息大小 > 10% 上下文窗口,降低保留比例
  if (safeAvgTokens / contextWindow > 0.1) {
    return Math.max(
      MIN_CHUNK_RATIO,
      BASE_CHUNK_RATIO - reduction
    );
  }
  return BASE_CHUNK_RATIO;
}
```

**分阶段摘要** (Progressive Summarization):
```typescript
async summarizeInStages(params: {
  messages: AgentMessage[];
  model: Model;
  apiKey: string;
  parts?: 2,                    // 分成 2 部分
  maxChunkTokens: number;
  contextWindow: number;
  previousSummary?: string;
}): Promise<string>

// 流程:
// 1. 按令牌份额分割消息成多部分
// 2. 并行摘要每个部分
// 3. 合并部分摘要成最终摘要
// 4. 保留: 决策、TODO、约束、关键信息
```

**消息修复** (在 `/src/agents/session-transcript-repair.ts`):
- 确保 tool call 和 tool result 正确配对
- 删除孤立的 toolResult (无对应的 toolCall)
- 删除重复的 toolResult (相同 ID)
- 为缺失的 tool result 插入合成错误

### 2.6 会话历史限制

**DM 历史限制** (在 `/src/agents/pi-embedded-runner/history.ts`):

```typescript
// 三级配置优先级:
1. 每个 DM 的 historyLimit (dms[userId].historyLimit)
2. 提供商级别的 dmHistoryLimit
3. 未定义(使用所有历史)

export function limitHistoryTurns(
  messages: AgentMessage[],
  limit: number | undefined
): AgentMessage[] {
  // 保留最后 N 个用户轮换 + 关联的助手响应
}
```

**配置示例**:
```yaml
channels:
  telegram:
    dmHistoryLimit: 20  # 默认保留 20 轮
    dms:
      "user123":
        historyLimit: 50  # 特定用户保留 50 轮
```

### 2.7 文本分块策略

**分块配置** (在 `/src/memory/manager.ts`):
```typescript
const DEFAULT_CHUNK_TOKENS = 400;         // 每块 400 tokens
const DEFAULT_CHUNK_OVERLAP = 80;         // 20% 重叠
const SNIPPET_MAX_CHARS = 700;            // 搜索结果最多 700 字符
```

**分块逻辑**:
1. 按段落分割 Markdown
2. 估算每个段落的 token 数
3. 合并小段落直到接近目标大小
4. 保留 80 token 重叠(避免语义断裂)
5. 为每个块计算嵌入向量

### 2.8 会话文件处理

**会话文件读取** (在 `/src/memory/session-files.ts`):

```typescript
async function buildSessionEntry(absPath: string): SessionFileEntry {
  const raw = await fs.readFile(absPath, "utf-8");
  const lines = raw.split("\n");
  const collected: string[] = [];

  for (const line of lines) {
    if (!line.trim()) continue;

    const record = JSON.parse(line);
    if (record.type !== "message") continue;

    const { message } = record;
    if (message.role === "user" || message.role === "assistant") {
      const text = extractSessionText(message.content);
      if (text) {
        const label = message.role === "user" ? "User" : "Assistant";
        collected.push(`${label}: ${text}`);
      }
    }
  }

  return {
    path: "sessions/main-session.jsonl",
    absPath,
    mtimeMs: stat.mtimeMs,
    size: stat.size,
    hash: hashText(collected.join("\n")),
    content: collected.join("\n")  // 规范化文本用于索引
  };
}
```

**文本规范化**:
```typescript
function normalizeSessionText(value: string): string {
  return value
    .replace(/\s*\n+\s*/g, " ")    // 换行 → 空格
    .replace(/\s+/g, " ")          // 多个空格 → 单个
    .trim();
}
```

---

## 三、Agent 上下文管理流程

### 3.1 执行流程图

```
用户输入
    ↓
runEmbeddedPiAgent() 启动
    ├── 解析模型和上下文窗口
    ├── 认证配置解析
    └── 多次尝试循环(包括故障转移)
        ↓
runEmbeddedAttempt()
    ├── 初始化 SessionManager (JSONL 读取)
    ├── 加载历史消息
    │   ├── limitHistoryTurns (DM 历史限制)
    │   ├── validateGeminiTurns (Gemini 轮换验证)
    │   └── validateAnthropicTurns (Anthropic 轮换验证)
    ├── 构建系统提示词
    │   ├── buildAgentSystemPrompt()
    │   ├── 加载 Bootstrap 文件
    │   ├── 注入工具定义
    │   ├── 注入运行时信息
    │   └── 应用群组/渠道特定提示词
    ├── 创建 Agent Session
    ├── 订阅会话事件 (流式输出、工具调用)
    ├── 运行 Agent 循环
    │   ├── session.steer(userMessage)
    │   ├── Agent 思考和工具调用
    │   ├── 执行工具
    │   └── 收集响应
    ├── 检查上下文溢出
    │   └── 如果超限 → compactEmbeddedPiSessionDirect()
    │       ├── 计算自适应压缩比例
    │       ├── 分阶段摘要生成
    │       └── 替换历史消息
    └── 构建响应 payload
        ↓
SessionManager 自动持久化 (JSONL 追加)
        ↓
返回 EmbeddedPiRunResult
```

### 3.2 上下文窗口防护

**文件**: `/src/agents/context-window-guard.ts`

```typescript
// 硬限制
const CONTEXT_WINDOW_HARD_MIN_TOKENS = 16_000;   // 低于此值阻止运行
const CONTEXT_WINDOW_WARN_BELOW_TOKENS = 32_000; // 低于此值发出警告

// 解析优先级 (从高到低):
1. Model.contextWindow (模型自报)
2. ModelsConfig (用户覆盖)
3. agents.defaults.contextTokens (全局配置)
4. DEFAULT_CONTEXT_TOKENS (200,000 for Claude Opus 4.5)
```

### 3.3 Provider 特定的处理差异

**Gemini 特殊要求** (在 `/src/agents/pi-embedded-runner/google.ts`):
- 严格的轮换顺序: 用户→助手→工具→用户
- 合并连续的助手消息
- Schema 清理(移除 Gemini 不支持的关键字)
- 思考块签名清理

**Anthropic 特殊要求** (在 `/src/agents/pi-embedded-helpers/turns.ts`):
- 严格的用户→助手模式
- 合并连续的用户消息
- Tool calls 必须在 assistant 消息内

### 3.4 思考和推理模式

**思考级别** (in `/src/auto-reply/thinking.ts`):
```typescript
type ThinkLevel = "off" | "minimal" | "low" | "medium" | "high" | "xhigh";
```

**推理级别**:
```typescript
type ReasoningLevel = "off" | "on" | "stream";

// "off"    - 禁用推理
// "on"     - 收集推理,在响应末尾发送(非流式)
// "stream" - 立即流式传输推理块
```

**故障转移逻辑**:
- 如果模型不支持请求的思考级别
- 从错误消息中提取支持的级别
- 自动降级到支持的最高级别
- 重试运行

---

## 四、关键文件路径索引

### 系统提示词相关

| 文件 | 功能 |
|------|------|
| `/src/agents/system-prompt.ts` | 核心构建函数 (592 行) |
| `/src/agents/system-prompt-params.ts` | 参数类型定义 |
| `/src/agents/system-prompt-report.ts` | 使用统计和分析 |
| `/src/agents/pi-embedded-runner/system-prompt.ts` | Pi Embedded 系统提示词 |
| `/src/agents/pi-embedded-runner/run/attempt.ts` | 运行时注入点 |
| `/src/agents/workspace.ts` | 工作区文件常量 |
| `/src/agents/bootstrap-files.ts` | 引导文件处理 |
| `/src/agents/skills.ts` | 技能系统 |
| `/src/auto-reply/heartbeat.ts` | 心跳提示词 |
| `/src/telegram/bot-message-context.ts` | Telegram 消息上下文 |
| `/src/auto-reply/reply/get-reply-run.ts` | 回复运行配置 |

### 持久化记忆相关

| 文件 | 功能 |
|------|------|
| `/src/memory/manager.ts` | 内存索引管理器 (2232 行) |
| `/src/memory/memory-schema.ts` | 数据库模式 |
| `/src/memory/session-files.ts` | 会话文件处理 |
| `/src/memory/manager-search.ts` | 搜索实现 |
| `/src/memory/hybrid.ts` | 混合搜索 |
| `/src/memory/embeddings.ts` | 嵌入提供者接口 |
| `/src/memory/embeddings-openai.ts` | OpenAI 嵌入 |
| `/src/memory/embeddings-gemini.ts` | Gemini 嵌入 |
| `/src/memory/node-llama.ts` | 本地 LLaMA 嵌入 |
| `/src/memory/sqlite-vec.ts` | SQLite 向量扩展 |
| `/src/agents/compaction.ts` | 上下文压缩 (346 行) |
| `/src/agents/memory-search.ts` | 记忆搜索配置 |
| `/src/agents/session-transcript-repair.ts` | 会话转录修复 |

### Agent 上下文管理相关

| 文件 | 功能 |
|------|------|
| `/src/agents/pi-embedded-runner/run.ts` | 主运行流程 (679 行) |
| `/src/agents/pi-embedded-runner/run/attempt.ts` | 单次尝试实现 |
| `/src/agents/pi-embedded-runner/run/params.ts` | 运行参数类型 |
| `/src/agents/context-window-guard.ts` | 上下文窗口管理 |
| `/src/agents/pi-embedded-runner/session-manager-init.ts` | SessionManager 初始化 |
| `/src/agents/session-write-lock.ts` | 文件锁定机制 |
| `/src/agents/pi-embedded-runner/history.ts` | 历史限制 |
| `/src/agents/pi-embedded-subscribe.ts` | 事件订阅 |
| `/src/agents/pi-embedded-runner/google.ts` | Gemini 特殊处理 |
| `/src/agents/pi-embedded-helpers/turns.ts` | Anthropic 轮换验证 |
| `/src/auto-reply/thinking.ts` | 思考级别管理 |

---

## 五、核心创新点总结

### 系统提示词创新
1. **动态组装架构** - 非静态模板,每次运行重新生成
2. **分层配置覆盖** - 全局→群组→话题三级配置
3. **条件性段落** - 根据上下文智能包含/排除
4. **工作区引导** - 用户可通过文件定制 Agent 行为
5. **版本化技能** - 自动检测和刷新技能变化

### 持久化记忆创新
1. **混合搜索** - 向量 + 关键字加权组合,准确性更高
2. **增量同步** - 基于哈希的智能同步,避免重复处理
3. **自适应压缩** - 根据消息大小动态调整上下文预算
4. **批处理优化** - 支持异步批 API,大幅降低成本
5. **多提供者嵌入** - 自动选择本地/OpenAI/Gemini
6. **插件扩展** - 支持 LanceDB 等替代后端

### Agent 上下文管理创新
1. **多级历史限制** - 全局→提供商→用户三级配置
2. **Provider 适配** - 自动处理 Anthropic/Gemini/OpenAI 差异
3. **智能故障转移** - 思考级别、模型、认证的自动降级
4. **分阶段摘要** - 并行压缩,保留关键信息
5. **文件锁定** - 多进程安全的会话持久化

---

## 六、架构优势分析

### 可扩展性
- **插件系统** - 内存后端可替换(LanceDB, Pinecone 等)
- **提供者无关** - 统一接口适配多种 LLM
- **钩子系统** - 支持自定义拦截和修改

### 性能优化
- **批处理嵌入** - 成本降低 50%+
- **嵌入缓存** - 避免重复 API 调用
- **增量同步** - 仅处理变化的文件
- **并发执行** - Lane 系统串行化会话操作

### 可靠性
- **文件锁定** - 防止并发写入冲突
- **转录修复** - 自动修正工具调用配对
- **陈旧锁检测** - 避免死锁
- **故障转移** - 多级降级策略

### 用户体验
- **透明配置** - YAML 配置易读易写
- **智能默认** - 合理的默认值,无需手动调优
- **增量学习** - 自动捕获和索引对话历史
- **语义搜索** - 自然语言查询长期记忆

---

## 七、潜在改进方向

### 系统提示词
1. **提示词压缩** - 使用指令压缩技术减少 token 消耗
2. **A/B 测试** - 支持多版本提示词效果对比
3. **动态优先级** - 根据查询类型调整段落顺序

### 持久化记忆
1. **主动学习** - 自动识别和提取重要信息
2. **遗忘曲线** - 基于时间衰减的记忆权重
3. **关系图谱** - 构建实体和概念的知识图谱
4. **多模态记忆** - 索引图片、音频、视频

### Agent 上下文管理
1. **预测性压缩** - 提前触发压缩,避免运行时延迟
2. **并行摘要** - 进一步优化分阶段摘要性能
3. **上下文缓存** - 利用 Anthropic Prompt Caching 降低成本

---

## 八、总结

OpenClaw 的系统提示词和持久化记忆系统展现了**工程精湛性**和**架构前瞻性**:

1. **分层设计** - 清晰的关注点分离,易于维护和扩展
2. **智能自动化** - 最小化用户配置,自动优化性能
3. **多提供者支持** - 统一抽象适配不同 LLM 的特性
4. **可靠性保障** - 文件锁、转录修复、故障转移等机制
5. **性能优先** - 批处理、缓存、增量同步等优化

这套系统不仅满足当前需求,更为未来的功能扩展(如多模态记忆、知识图谱)奠定了坚实基础。

---

**调研完成日期**: 2026-01-31
**调研深度**: Very Thorough
**代码库版本**: main branch (commit bc432d843)
