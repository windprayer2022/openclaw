# Compaction 机制

> 源码阅读笔记 — 2026-02-25

## 概述

Compaction 是 OpenClaw 的**上下文压缩**机制。当 LLM 对话上下文接近模型窗口限制时，自动将历史消息摘要化以释放空间，同时在压缩前提供"记忆刷写"（Memory Flush）机会让 Agent 将重要信息持久化到磁盘。

## 关键文件

| 文件 | 职责 |
|------|------|
| `src/agents/compaction.ts` | **核心算法**：token 估算、分块、摘要生成、渐进式 fallback、历史裁剪 |
| `src/agents/pi-embedded-runner/compact.ts` | **顶层编排**：构建完整 session 环境、调用 `session.compact()`、lane 排队 |
| `src/agents/pi-embedded-runner/run.ts` | **溢出触发**：context overflow 时自动 compaction 重试循环 |
| `src/agents/pi-extensions/compaction-safeguard.ts` | **safeguard 扩展**：`session_before_compact` 钩子，增强摘要（分阶段+文件操作+工具失败） |
| `src/agents/pi-extensions/compaction-safeguard-runtime.ts` | session 级运行时参数注入（WeakMap by SessionManager） |
| `src/agents/pi-embedded-runner/extensions.ts` | 注册 safeguard 扩展 + 注入运行时参数 |
| `src/agents/pi-settings.ts` | compaction reserve tokens 地板值管理 |
| `src/auto-reply/reply/memory-flush.ts` | Memory Flush 设置解析 + 阈值判断 |
| `src/auto-reply/reply/agent-runner-memory.ts` | Memory Flush 执行逻辑 |
| `src/agents/pi-embedded-subscribe.handlers.lifecycle.ts` | 流式事件处理：compaction start/end 生命周期 |

## 1. 四种触发方式

| 触发方式 | trigger 值 | 触发位置 | 场景 |
|----------|-----------|----------|------|
| **overflow** | `"overflow"` | `run.ts` L534-583 | LLM 返回 context overflow 错误时自动触发 |
| **manual** | `"manual"` | Gateway `sessions.compact` | 用户通过 WebSocket RPC 手动触发 |
| **cache_ttl** | `"cache_ttl"` | context-pruning 扩展 | Anthropic prompt cache 过期时触发 |
| **safeguard** | `"safeguard"` | compaction-safeguard 扩展 | 预防性压缩（通过 `session_before_compact` 钩子） |

### 1.1 overflow 触发（自动 compaction 重试）

`runEmbeddedPiAgent()` 中的主循环检测到 context overflow 错误后：

```
context_overflow 检测（promptError 或 assistantErrorText）
  → isLikelyContextOverflowError() 确认
  → isCompactionFailureError() 排除已失败的 compaction
  → overflowCompactionAttempts < MAX_OVERFLOW_COMPACTION_ATTEMPTS (3)
  → compactEmbeddedPiSessionDirect({trigger: "overflow"})
  → 成功? continue（重试 prompt）
  → 失败? 尝试截断超大 tool result → 再次重试
  → 都失败? 返回错误让用户 /reset
```

关键常量：`MAX_OVERFLOW_COMPACTION_ATTEMPTS = 3`

### 1.2 manual 触发（Gateway RPC）

Gateway `sessions.compact` 做的是**简化版 transcript 裁剪**（不是 LLM 摘要）：
- 读取 transcript JSONL 文件
- 归档到 `.bak`
- 只保留最后 `maxLines` 行（默认 100）
- 清零 token 计数（inputTokens, outputTokens, totalTokens, totalTokensFresh, contextTokens）

### 1.3 Memory Flush — 压缩前的记忆刷写

在正式推理 run 结束后、下一次推理前，检查是否需要运行 Memory Flush：

```
shouldRunMemoryFlush() 判断：
  totalTokens >= contextWindow - reserveTokens - softThresholdTokens (4000)
  且 memoryFlushCompactionCount ≠ compactionCount（本轮未 flush 过）
  → 注入特殊 prompt 让 Agent 将记忆写入 memory/YYYY-MM-DD.md
  → flush 完成后更新 memoryFlushAt + memoryFlushCompactionCount
```

默认 prompt：
```
Pre-compaction memory flush.
Store durable memories now (use memory/YYYY-MM-DD.md; create memory/ if needed).
IMPORTANT: If the file already exists, APPEND new content only.
If nothing to store, reply with <NO_REPLY>.
```

## 2. 核心 Compaction 算法（compaction.ts）

### 2.1 Token 估算

```typescript
estimateMessagesTokens(messages)
  → stripToolResultDetails(messages)  // 安全：去除 toolResult.details
  → reduce(estimateTokens(msg))       // 来自 pi-coding-agent SDK
```

### 2.2 消息分块策略

**按 token 份额均分**：`splitMessagesByTokenShare(messages, parts=2)`
- 按 token 均分为 N 份（默认 2），用于分阶段摘要

**按最大 token 分块**：`chunkMessagesByMaxTokens(messages, maxTokens)`
- 每块不超过 maxTokens，超大单条消息单独成块

**自适应 chunk ratio**：`computeAdaptiveChunkRatio(messages, contextWindow)`
- 基础比例 `BASE_CHUNK_RATIO = 0.4`（每块最大占上下文 40%）
- 当平均消息 > 10% 上下文时，动态降低到 `MIN_CHUNK_RATIO = 0.15`
- 安全边距 `SAFETY_MARGIN = 1.2`（token 估算 20% 容差）

### 2.3 三层摘要 Fallback

```
summarizeInStages()
  ├─ 消息 < minMessagesForSplit(4) 或 totalTokens < maxChunkTokens?
  │   → 直接 summarizeWithFallback()
  └─ 分片 → 每片独立摘要 → 合并摘要
       └─ MERGE_SUMMARIES_INSTRUCTIONS: "Merge these partial summaries..."

summarizeWithFallback()
  ├─ 尝试 summarizeChunks()（全量）
  ├─ 失败 → 过滤超大消息（> 50% 上下文）→ 仅摘要小消息 + oversized notes
  └─ 都失败 → 文本 fallback: "Context contained N messages..."

summarizeChunks()
  → chunkMessagesByMaxTokens()
  → 逐块 generateSummary()（来自 SDK，LLM 调用）
  → 后一块的 previousSummary = 前一块摘要（滚动摘要）
```

### 2.4 历史裁剪（History Pruning for Context Share）

`pruneHistoryForContextShare()` — 当 new content 超过 maxHistoryShare 时裁剪老消息：

```
budgetTokens = maxContextTokens × maxHistoryShare (默认 0.5)
while keptMessages > budgetTokens:
  splitMessagesByTokenShare(keptMessages, 2)
  丢弃第一块（最老的）
  repairToolUseResultPairing（修复孤立的 toolResult）
  keptMessages = 剩余部分
```

## 3. Compaction 编排层（compact.ts）

### 3.1 `compactEmbeddedPiSessionDirect()` — 直接执行

完整流程：

```
1. 解析 model + API key（与正常推理相同）
2. 解析 workspace + sandbox
3. ensureSessionHeader()
4. 加载 skill entries + skill env overrides
5. 构建 bootstrap context files
6. 创建工具集（createOpenClawCodingTools）
7. 构建完整 system prompt（同正常推理）
8. 获取写锁 → repairSessionFile → prewarmSessionFile
9. 解析 transcript policy + guardSessionManager
10. ensurePiCompactionReserveTokens（≥ 20,000 tokens）
11. buildEmbeddedExtensionPaths（注册 safeguard 扩展）
12. createAgentSession + applySystemPromptOverride

—— 消息预处理（同正常推理） ——
13. sanitizeSessionHistory → validateTurns → limitHistoryTurns → repairPairing

—— 钩子 ——
14. before_compaction hook (fire-and-forget)

—— 核心调用 ——
15. session.compact(customInstructions) → SDK compaction
16. 估算压缩后 tokens

—— 后处理 ——
17. after_compaction hook (fire-and-forget)
18. flushPendingToolResultsAfterIdle
19. session.dispose()
20. 释放写锁
```

### 3.2 `compactEmbeddedPiSession()` — 带 Lane 排队

```
enqueueCommandInLane(sessionLane,    // per-session 串行
  enqueueCommandInLane(globalLane,   // 全局并发控制
    compactEmbeddedPiSessionDirect()
  )
)
```

双层 Lane 防止：
- 同一 session 并发 compaction（session lane 串行）
- 全局 compaction 资源竞争（global lane 并发限制）

## 4. Safeguard 扩展（compaction-safeguard.ts）

通过 `session_before_compact` 事件钩子接管 compaction 的摘要生成：

### 4.1 增强摘要内容

| 内容 | 来源 | 格式 |
|------|------|------|
| 历史摘要 | `summarizeInStages()` | LLM 生成的文本 |
| Split Turn 前缀 | 分裂轮次的前半部分单独摘要 | "**Turn Context (split turn):**" |
| 工具失败记录 | `collectToolFailures()` | "## Tool Failures" section（最多 8 条，每条 240 字符） |
| 文件操作记录 | `computeFileLists()` | `<read-files>` + `<modified-files>` XML tags |

### 4.2 History Pruning 集成

当 new content（非摘要区域）占用超过 maxHistoryShare（默认 50%）的上下文时：

```
pruneHistoryForContextShare()
  → 丢弃最老的 chunk(s)
  → 对丢弃的消息也做 summarizeInStages()（作为 droppedSummary）
  → droppedSummary 作为 previousSummary 喂给主摘要
```

### 4.3 Fallback 策略

```
无 model? → fallbackSummary（截断通知 + 工具失败 + 文件操作）
无 apiKey? → 同上
摘要失败? → 同上（catch 兜底）
```

## 5. 运行时参数注入

通过 WeakMap 注入 session 级配置（避免全局污染）：

```typescript
// extensions.ts
setCompactionSafeguardRuntime(sessionManager, {
  maxHistoryShare: compactionCfg?.maxHistoryShare,   // 默认 0.5
  contextWindowTokens: contextWindowInfo.tokens,      // 模型上下文窗口
});

// compaction-safeguard.ts 中读取
const runtime = getCompactionSafeguardRuntime(ctx.sessionManager);
```

## 6. 配置 Schema

```typescript
agents.defaults.compaction: {
  mode?: "default" | "safeguard",        // 压缩模式
  reserveTokensFloor?: number,           // 最小预留 token（默认 20,000）
  maxHistoryShare?: number,              // 历史占上下文比例（0.1-0.9，默认 0.5）
  memoryFlush?: {
    enabled?: boolean,                   // 默认 true
    softThresholdTokens?: number,        // 提前多少 token 触发 flush（默认 4,000）
    prompt?: string,                     // flush 时的用户 prompt
    systemPrompt?: string,               // flush 时的系统 prompt
  }
}
```

## 7. 流式事件处理

`pi-embedded-subscribe.handlers.lifecycle.ts` 中处理 compaction 生命周期事件：

```
handleAutoCompactionStart(ctx):
  state.compactionInFlight = true
  incrementCompactionCount
  ensureCompactionPromise    // 创建 Promise 等待 compaction 完成
  emit "compaction" {phase: "start"}
  before_compaction hook (fire-and-forget)

handleAutoCompactionEnd(ctx, evt):
  state.compactionInFlight = false
  willRetry?
    → noteCompactionRetry + resetForCompactionRetry
    → 不 resolve promise（等下一轮）
  !willRetry?
    → maybeResolveCompactionWait  // resolve promise
    → after_compaction hook (fire-and-forget)
  emit "compaction" {phase: "end", willRetry}
```

调用方可以用 `waitForCompactionRetry()` 等待 compaction + retry 全部完成。

## 8. 溢出恢复完整决策树

```
context overflow 检测
  │
  ├─ isCompactionFailureError? → 直接放弃（返回错误）
  │
  ├─ overflowCompactionAttempts < 3?
  │   → compactEmbeddedPiSessionDirect({trigger: "overflow"})
  │   → 成功? autoCompactionCount++, continue（重试 prompt）
  │   → 失败? 记录 warn
  │
  ├─ !toolResultTruncationAttempted?
  │   → sessionLikelyHasOversizedToolResults()?
  │     → truncateOversizedToolResultsInSession()
  │     → 成功? reset overflowCompactionAttempts=0, continue
  │     → 失败? 记录 warn
  │
  └─ 所有尝试都失败
      → 返回用户友好错误:
        "Context overflow: prompt too large... Try /reset or larger-context model."
```

## 架构总结

```
                   触发源
    ┌──────────┬──────────┬──────────┐
  overflow   manual   cache_ttl  safeguard
    │          │          │          │
    ↓          ↓          ↓          ↓
┌──────────────────────────────────────┐
│  Lane 排队（session + global）         │
│  compactEmbeddedPiSession()          │
└──────────────┬───────────────────────┘
               ↓
┌──────────────────────────────────────┐
│  编排层 compactEmbeddedPiSessionDirect │
│  ├─ 环境构建（同正常推理）               │
│  ├─ 写锁 + session repair             │
│  ├─ 消息预处理                         │
│  ├─ before_compaction hook             │
│  ├─ session.compact() → SDK            │
│  ├─ after_compaction hook              │
│  └─ cleanup                            │
└──────────────┬───────────────────────┘
               ↓
┌──────────────────────────────────────┐
│  SDK session.compact()                │
│  ├─ session_before_compact 事件        │
│  │   └─ safeguard 扩展接管              │
│  │       ├─ History Pruning            │
│  │       ├─ summarizeInStages()        │
│  │       ├─ Tool Failures section      │
│  │       └─ File Operations section    │
│  └─ 替换消息: summary + kept messages   │
└──────────────────────────────────────┘

Memory Flush（压缩前记忆刷写）
┌──────────────────────────────────────┐
│  shouldRunMemoryFlush()               │
│  totalTokens ≥ ctx - reserve - soft   │
│  且本轮未 flush                        │
│  → runEmbeddedPiAgent({               │
│      prompt: "Store durable memories"  │
│    })                                  │
│  → 更新 memoryFlushAt                  │
└──────────────────────────────────────┘
```

### 设计亮点

1. **渐进式 Fallback**：全量摘要 → 过滤超大消息 → 纯文本 fallback，确保 compaction 不会因单条超大消息而彻底失败
2. **自适应 Chunk Ratio**：根据消息平均大小动态调整分块比例，避免单块超过 LLM 上下文
3. **Memory Flush**：在压缩前给 Agent 一次机会将重要上下文持久化到文件，降低信息损失
4. **双层 Lane 排队**：session 级串行 + global 级并发控制，防止资源竞争
5. **Tool Failures + File Operations 保留**：compaction 后仍保留工具失败记录和文件操作记录，避免上下文丢失导致重复错误
6. **安全措施**：`stripToolResultDetails()` 确保不将不可信的 toolResult.details 喂给 LLM；`SAFETY_MARGIN = 1.2` 为 token 估算提供 20% 容差
