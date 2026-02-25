# Session 运行时消息管理

> 源码位置：`src/agents/pi-embedded-runner/run/attempt.ts`（主流程）、`session-tool-result-guard.ts`（持久化守卫）、`session-tool-result-guard-wrapper.ts`（守卫包装）、`session-file-repair.ts`（文件修复）、`session-manager-init.ts`（初始化修正）、`session-manager-cache.ts`（预热缓存）、`session-write-lock.ts`（写锁）、`pi-embedded-runner/google.ts`（历史清洗）、`pi-embedded-runner/history.ts`（历史裁剪）、`pi-embedded-runner/tool-result-truncation.ts`（工具结果截断）、`transcript-policy.ts`（策略）、`pi-embedded-runner/extensions.ts`（扩展注册）

## 总体架构

Session 运行时消息管理负责将磁盘上的 JSONL 会话文件变成 LLM 可以消费的消息序列。核心对象 `SessionManager`（来自上游 `pi-coding-agent` 库）是一个树状会话存储，OpenClaw 在其外围包裹了 **6 层处理**：

```
磁盘 JSONL 文件
    ↓
① 文件修复层     repairSessionFileIfNeeded — 清除损坏行
    ↓
② 写锁层         acquireSessionWriteLock — 进程级互斥
    ↓
③ SessionManager  SessionManager.open(file) — 上游库加载+内存树
    ↓
④ 守卫层         guardSessionManager — monkey-patch appendMessage
    ↓                ├─ 工具结果截断（capToolResultSize）
                     ├─ 合成缺失 toolResult（synthetic）
                     ├─ 输入溯源标记（InputProvenance）
                     └─ tool_result_persist 钩子
⑤ 初始化修正     prepareSessionManagerForRun — 处理 flush 怪癖
    ↓
⑥ 消息清洗管线   sanitizeSessionHistory → 多步清洗 → replaceMessages
    ↓
⑦ LLM 调用       session.prompt(text, {images?})
```

## 详细流程

### 1. 文件修复 — `repairSessionFileIfNeeded()`

在打开 session 文件之前运行。逐行 `JSON.parse`，清除无法解析的损坏行（如写入中断、非 UTF-8）。修复前备份原文件（`.bak-{pid}-{timestamp}`），原子替换。

### 2. 写锁 — `acquireSessionWriteLock()`

防止同一 session 文件被多个进程并发写入：

- **锁文件**：`{sessionFile}.lock`，用 `fs.open('wx')` 原子创建
- **重入计数**：同进程内 `HELD_LOCKS` Map 支持可重入
- **死锁恢复**：读取锁文件中的 PID，`process.kill(pid, 0)` 检测进程是否存活；stale 阈值 30 分钟
- **进程退出清理**：监听 `SIGINT/SIGTERM/SIGQUIT/SIGABRT` + `process.exit`，同步释放所有锁
- **超时**：默认 10 秒，指数退避轮询（50ms → 1s）

### 3. SessionManager 加载

```typescript
SessionManager.open(sessionFile)  // 上游库：解析 JSONL → 内存树结构
```

上游 `pi-coding-agent` 库的 `SessionManager`：
- 维护 **树状消息历史**（支持分支、branch、leafId）
- `getEntries()` → 所有条目
- `getBranch()` → 当前分支的线性条目序列
- `buildSessionContext()` → `{ messages }` 当前分支的消息数组
- `appendMessage()` → 追加消息并写入 JSONL
- `branch(id)` / `resetLeaf()` → 会话分支操作

### 4. Session Manager 缓存预热 — `prewarmSessionFile()`

在 `SessionManager.open()` 之前读取文件前 4KB，鼓励 OS page cache 预热。45 秒 TTL 缓存跟踪，避免频繁重读。

### 5. 守卫层 — `guardSessionManager()`

通过 **monkey-patch `appendMessage`** 拦截所有写入：

```
原始 appendMessage
    ↓ 被替换为 guardedAppend
    ↓
角色判断:
├─ assistant → sanitizeToolCallInputs + 记录 pending toolCall IDs
├─ toolResult → 匹配 pending ID + capToolResultSize (400K 硬限) + hook
└─ 其他角色 → 刷新 pending synthetic results
```

**关键行为**：
- **Synthetic Tool Results**：如果 assistant 发出了 toolCall 但没有收到对应 toolResult（如中断），自动合成占位 toolResult 以满足 provider 严格要求
- **工具结果大小限制**：单个 toolResult 不超过 400K 字符，超限按比例截断各 text block
- **InputProvenance 标记**：给 user message 添加溯源信息（来自哪个 session/channel）
- **tool_result_persist 钩子**：插件可在 toolResult 写入前变换消息内容

### 6. 初始化修正 — `prepareSessionManagerForRun()`

处理 pi-coding-agent 的一个持久化怪癖：
- 如果文件存在但**没有 assistant 消息**，SessionManager 会标记 `flushed=true`，导致首条 user message 不被持久化
- 修正方法：清空文件内容，重置为仅含 header，将 `flushed` 设为 false

### 7. 消息清洗管线 — `sanitizeSessionHistory()`

这是核心环节，在 `session.prompt()` 之前执行 **8 步清洗**：

```
activeSession.messages（从 SessionManager 加载的原始消息）
    ↓
① annotateInterSessionUserMessages   — 跨 session 消息添加 [Inter-session message] 前缀
    ↓
② sanitizeSessionMessagesImages       — 清理/移除过期图片引用
    ↓
③ sanitizeAntigravityThinkingBlocks   — Google/Gemini thinking block 签名标准化
    ↓
④ sanitizeToolCallInputs             — 清理无效 toolCall input
    ↓
⑤ sanitizeToolUseResultPairing       — 修复 toolUse/toolResult 配对（孤儿匹配）
    ↓
⑥ stripToolResultDetails             — 移除 toolResult.details（防止不可信数据回流模型）
    ↓
⑦ downgradeOpenAIReasoningBlocks     — 模型切换时降级 reasoning blocks
    ↓
⑧ applyGoogleTurnOrderingFix         — Google API 要求首条是 user（非 assistant）
```

清洗后还有 **验证 + 裁剪**：
- `validateGeminiTurns()` / `validateAnthropicTurns()` — provider 特定的 turn 顺序验证
- `limitHistoryTurns()` — 按配置裁剪历史轮次（只保留最近 N 轮 user turn）
- 裁剪后再次 `sanitizeToolUseResultPairing()` 修复被截断产生的孤儿

最终：`activeSession.agent.replaceMessages(cleaned)` 用清洗后的消息替换内存中的历史。

### 8. Prompt 执行

```typescript
// 钩子注入上下文
let effectivePrompt = params.prompt;
if (hookRunner?.hasHooks("before_agent_start")) {
  const hookResult = await hookRunner.runBeforeAgentStart({
    prompt: params.prompt,
    messages: activeSession.messages,
  });
  if (hookResult?.prependContext) {
    effectivePrompt = `${hookResult.prependContext}\n\n${params.prompt}`;
  }
}

// 修复孤儿 user message（防止连续 user turn）
if (leafEntry?.type === "message" && leafEntry.message.role === "user") {
  sessionManager.branch(leafEntry.parentId);  // 分支回退
}

// 检测并加载图片
const imageResult = await detectAndLoadPromptImages({...});
injectHistoryImagesIntoMessages(activeSession.messages, imageResult.historyImagesByIndex);

// 调用 LLM
await activeSession.prompt(effectivePrompt, { images: imageResult.images });
```

实际发给 LLM API 的结构：
```
┌─ system:     buildAgentSystemPrompt() 输出      ← 静态指令
├─ messages[]: 清洗后的 session 历史                ← 经过 8 步清洗
├─ user:       effectivePrompt + images            ← 当前输入
└─              （LLM 流式回复 → 写入 session）
```

## Transcript Policy（转录策略）

`resolveTranscriptPolicy()` 根据 provider/model 决定清洗策略：

| 策略项 | Anthropic | Google | OpenAI | Mistral |
|--------|-----------|--------|--------|---------|
| sanitizeMode | full | full | images-only | full |
| sanitizeToolCallIds | ✓ | ✓ | ✓ | ✓(strict9) |
| repairToolUseResultPairing | ✓ | ✓ | ✗ | ✗ |
| normalizeAntigravityThinking | ✗ | Claude via Google | ✗ | ✗ |
| applyGoogleTurnOrdering | ✗ | ✓ | ✗ | ✗ |
| validateTurns | Anthropic | Gemini | ✗ | ✗ |
| allowSyntheticToolResults | ✓ | ✓ | ✗ | ✗ |

## 工具结果截断 — `tool-result-truncation.ts`

两个层次的截断：

### 持久化时截断（Guard 层）
- `capToolResultSize()` 在 `appendMessage` 时截断
- 硬限：**400K 字符**（`HARD_MAX_TOOL_RESULT_CHARS`）
- 多个 text block 按比例分配预算，最低保留 2K 字符

### 发送前截断（Session 级别）
- `truncateOversizedToolResultsInSession()` — 修改 session 文件（分支 + 重写）
- `truncateOversizedToolResultsInMessages()` — 仅内存修改
- 动态限制：**上下文窗口的 30%**（`MAX_TOOL_RESULT_CONTEXT_SHARE`）
- 按 4 chars ≈ 1 token 估算

### 启发式检测
- `sessionLikelyHasOversizedToolResults()` — 判断 overflow 是否由超大 toolResult 引起，决定是否尝试截断恢复

## History Limit（历史裁剪）

`limitHistoryTurns()` — 从尾部计数 user turn，超出限制则截断前部。

配置来源（`getHistoryLimitFromSessionKey()`）：
- DM session → `channels.{provider}.dms.{userId}.historyLimit` → `channels.{provider}.dmHistoryLimit`
- Group/Channel session → `channels.{provider}.historyLimit`

## 扩展系统 — `extensions.ts`

`buildEmbeddedExtensionPaths()` 注册 pi-coding-agent 扩展：

1. **Compaction Safeguard** — 当 `compaction.mode === "safeguard"` 时启用，设置 `maxHistoryShare` 防止历史占用过多上下文
2. **Context Pruning** — 当 `contextPruning.mode === "cache-ttl"` 时启用，基于缓存 TTL 修剪旧的工具调用结果

## 数据流全景图

```
用户消息 (from Channel/Gateway)
    ↓
runEmbeddedAttempt()
    ↓
┌─ repairSessionFileIfNeeded()      修复损坏 JSONL
├─ acquireSessionWriteLock()         进程级写锁
├─ prewarmSessionFile()              OS page cache 预热
├─ SessionManager.open(file)         加载 JSONL → 内存树
├─ guardSessionManager()             monkey-patch appendMessage
├─ prepareSessionManagerForRun()     修正 flush 怪癖
├─ buildEmbeddedExtensionPaths()     注册 compaction/pruning 扩展
├─ createAgentSession()              创建 pi-coding-agent session
├─ applySystemPromptOverrideToSession() 设置 system prompt
├─ sanitizeSessionHistory()          8 步消息清洗
├─ limitHistoryTurns()               历史轮次裁剪
├─ replaceMessages()                 用清洗后消息替换内存
├─ before_agent_start hook           插件注入上下文
├─ 孤儿 user message 修复            防连续 user turn
├─ detectAndLoadPromptImages()       图片检测与加载
└─ session.prompt(text, {images})    → LLM API
    ↓
流式回复 → guardedAppend → JSONL 写入
    ↓
响应 → ReplyDispatcher → Channel → 用户
```
