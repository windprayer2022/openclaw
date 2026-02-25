# Agent Runtime — Pi Embedded

> OpenClaw 的 AI 推理引擎。将用户消息送入 LLM，管理工具调用循环，产生流式响应。

## 调用链全景

```
Channel inbound message
  ↓
dispatchReplyFromConfig()          ← src/auto-reply/reply/dispatch-from-config.ts
  │  TTS 处理、跨通道路由、hook(message_received)
  ↓
getReplyFromConfig()               ← src/auto-reply/reply/get-reply.ts
  │  Agent 解析、Model 选择、Media/Link 理解
  │  Session 初始化、指令解析、内联命令
  ↓
runPreparedReply()                 ← src/auto-reply/reply/get-reply-run.ts
  │  System prompt 拼装、Session 文件解析
  │  Queue 模式决策(steer/followup/interrupt)
  ↓
runReplyAgent()                    ← src/auto-reply/reply/agent-runner.ts
  │  Steer 注入(isStreaming → queueEmbeddedPiMessage)
  │  Followup 排队(isActive → enqueueFollowupRun)
  │  Memory flush、Block reply pipeline
  ↓
runAgentTurnWithFallback()         ← src/auto-reply/reply/agent-runner-execution.ts
  │  runWithModelFallback() 包装
  │  CLI provider → runCliAgent()
  │  标准 provider →
  ↓
runEmbeddedPiAgent()               ← src/agents/pi-embedded-runner/run.ts
  │  模型解析、Context Window Guard
  │  Auth Profile 轮换循环
  │  Context Overflow → 自动 Compaction → 重试
  ↓
runEmbeddedAttempt()               ← src/agents/pi-embedded-runner/run/attempt.ts
  │  创建 AgentSession(pi-coding-agent SDK)
  │  构建 System Prompt、工具集
  │  session.prompt() → LLM API 调用
  │  subscribeEmbeddedPiSession() → 流式事件处理
  ↓
subscribeEmbeddedPiSession()       ← src/agents/pi-embedded-subscribe.ts
  │  监听 pi-agent-core 事件流
  │  message_start/update/end → 文本/推理流
  │  tool_execution_start/update/end → 工具摘要
  │  auto_compaction_start/end → Compaction 追踪
  ↓
响应 → ReplyDispatcher → Channel → 消息平台
```

## 核心模块分组

### 1. 入口编排层 (src/auto-reply/reply/)

| 文件 | 职责 |
|------|------|
| dispatch-from-config.ts | 顶层入口：去重、TTS、hook、跨通道路由 |
| get-reply.ts | 中间编排：agent 选择、model 选择、media/link 理解、session 初始化、指令解析 |
| get-reply-run.ts | 下层编排：system prompt 拼装、queue 模式决策、session 文件路径 |
| agent-runner.ts | 执行包装：steer/followup 逻辑、block reply pipeline、usage 统计 |
| agent-runner-execution.ts | 执行核心：model fallback 循环、错误恢复（context overflow/role ordering/transient HTTP） |

### 2. Pi Embedded Runner (src/agents/pi-embedded-runner/)

| 文件 | 职责 |
|------|------|
| run.ts | `runEmbeddedPiAgent` — 双层重试循环（auth profile × overflow compaction） |
| run/attempt.ts | `runEmbeddedAttempt` — 单次 LLM 调用的完整生命周期 |
| run/params.ts | 参数类型定义 |
| run/payloads.ts | 构建最终回复 payload |
| runs.ts | 活跃运行注册表（abort/steer/active 查询） |
| lanes.ts | 会话级 & 全局级命令队列 lane |
| compact.ts | 自动 compaction（context overflow 恢复） |
| model.ts | 模型解析（provider → model 注册表） |
| system-prompt.ts | System prompt 构建（运行时信息、skills、bootstrap files、sandbox info） |
| history.ts | 历史消息截断（DM/群组不同限制） |
| session-manager-init.ts | SessionManager 初始化与修复 |
| tool-result-truncation.ts | 过大 tool result 截断 |

### 3. 流式订阅系统 (pi-embedded-subscribe*)

| 文件 | 职责 |
|------|------|
| pi-embedded-subscribe.ts | 主订阅函数，管理状态机 |
| pi-embedded-subscribe.handlers.ts | 事件分发（switch on event type） |
| .handlers.messages.ts | message_start/update/end — 文本流、推理流、block reply |
| .handlers.tools.ts | tool_execution_start/update/end — 工具摘要、typing 信号 |
| .handlers.lifecycle.ts | agent_start/end、auto_compaction_start/end |
| pi-embedded-block-chunker.ts | Block reply 分块器（段落/换行/句子分割） |

### 4. 工具系统 (pi-tools*)

| 文件 | 职责 |
|------|------|
| pi-tools.ts | `createOpenClawCodingTools` — 工具集组装入口 |
| pi-tools.policy.ts | 多层工具策略过滤（profile/global/agent/group/sandbox/subagent） |
| pi-tools.read.ts | Read 工具增强（sandbox 适配、Claude 兼容） |
| pi-tools.schema.ts | JSON Schema 归一化（Gemini/OpenAI 兼容） |
| pi-tools.abort.ts | AbortSignal 包装 |
| pi-tools.before-tool-call.ts | before_tool_call hook 包装 |
| pi-tool-definition-adapter.ts | 客户端工具（OpenResponses hosted tools）适配 |

### 5. Auth Profile 系统 (auth-profiles/)

| 文件 | 职责 |
|------|------|
| profiles.ts | Profile CRUD |
| store.ts | 持久化存储 |
| order.ts | 轮换顺序解析（lastGood/lastUsed/配置顺序） |
| session-override.ts | 会话级 profile 覆盖 |
| oauth.ts | OAuth token 刷新 |
| doctor.ts | Profile 健康检查 |

### 6. 模型系统

| 文件 | 职责 |
|------|------|
| model-selection.ts | 模型字符串解析、alias 索引 |
| model-fallback.ts | `runWithModelFallback` — 配置的备选模型链 |
| model-auth.ts | API key 解析（api-key/aws-sdk/oauth） |
| model-catalog.ts | 内置模型目录 |
| models-config.ts | models.json 加载 |
| context-window-guard.ts | Context window 检查与警告 |

### 7. 其他子系统

| 文件 | 职责 |
|------|------|
| system-prompt.ts / system-prompt-params.ts | System prompt 模板引擎 |
| sandbox.ts / sandbox/ | Docker sandbox 隔离环境 |
| skills.ts / skills/ | Skill 加载与 prompt 注入 |
| compaction.ts | Session compaction（压缩历史对话） |
| workspace.ts / workspace-run.ts | Agent 工作目录管理 |
| cli-runner.ts | CLI backend 执行器（Claude Code 等外部 Agent） |

## 关键设计

### 双层重试循环 (runEmbeddedPiAgent)

```
外层: Auth Profile 轮换
  for each profileCandidate:
    if inCooldown → skip
    applyApiKeyInfo(candidate)

    内层: LLM 调用 + Context Overflow 恢复
    while true:
      runEmbeddedAttempt(prompt, model, auth...)

      if contextOverflow:
        compactEmbeddedPiSessionDirect() → retry
        (最多 3 次)
        truncateOversizedToolResults() → retry

      if authError/rateLimit/timeout:
        markAuthProfileFailure()
        advanceAuthProfile() → continue 外层

      if thinkingLevelUnsupported:
        pickFallbackThinkingLevel() → retry

      success → markAuthProfileGood() → return
```

### 模型 Fallback 链 (runWithModelFallback)

```
candidates = [primary, ...config.agents.defaults.model.fallbacks]
for each candidate:
  if allProfilesInCooldown(candidate.provider) → skip
  try: result = run(candidate.provider, candidate.model)
  catch FailoverError → try next candidate
  catch AbortError → rethrow (用户主动中止)
  catch other → rethrow (非 failover 错误不 fallback)
```

### Session 管理 (runEmbeddedAttempt)

1. `SessionManager.open(sessionFile)` — 打开/创建会话文件
2. `sanitizeSessionHistory()` — 清理历史（provider 特定修复）
3. `validateGeminiTurns()` / `validateAnthropicTurns()` — 消息格式校验
4. `limitHistoryTurns()` — 截断历史到配置的上限
5. `sanitizeToolUseResultPairing()` — 修复 tool_use/tool_result 配对
6. `session.prompt(text, { images })` — 发送 prompt 到 LLM
7. Hook: `before_agent_start` → `agent_end`

### 工具集组装 (createOpenClawCodingTools)

```
base tools (from pi-coding-agent SDK):
  read, write, edit, (bash stripped)

+ exec tool (sandbox-aware, elevated bash)
+ process tool (background process management)
+ apply_patch tool (OpenAI Codex 模式)
+ channel agent tools (login 等)
+ openclaw tools:
  - browser, web_fetch, web_search
  - image, tts, canvas
  - message (跨通道消息发送)
  - sessions_send, sessions_list, sessions_history
  - memory_search
  - cron, nodes, gateway
  - discord/slack/telegram/whatsapp actions

过滤链:
  ownerOnly → profile → providerProfile → global → globalProvider
  → agent → agentProvider → group → sandbox → subagent
  → normalize schemas → wrap hooks → wrap abort
```

### 流式响应管道

```
pi-agent-core SDK 事件流
  ↓
subscribeEmbeddedPiSession (事件处理器)
  ├─ message_update → deltaBuffer → onPartialReply (typing 信号)
  ├─ message_end → assistantTexts[] (最终文本收集)
  ├─ tool_execution_* → toolMetas[] + onToolResult (工具摘要)
  └─ auto_compaction_* → compaction 追踪
  ↓
BlockReplyPipeline (可选，block streaming 模式)
  ├─ EmbeddedBlockChunker (段落/句子/换行分块)
  ├─ coalescing (合并小块)
  └─ audioAsVoice buffer
  ↓
onBlockReply / onPartialReply / onToolResult 回调
  ↓
ReplyDispatcher → Channel 适配器 → 消息平台
```

### Queue 模式

| 模式 | 行为 | 场景 |
|------|------|------|
| steer | 正在流式输出时注入新消息到当前 agent | 用户连续发消息 |
| followup | 正在运行时排队，完成后再处理 | 默认行为 |
| collect | 类似 followup，但合并多条消息 | 批量处理 |
| steer-backlog | steer + followup 结合 | 既注入又排队 |
| interrupt | 清空队列 + abort 当前运行 | /stop 等 |

## 依赖的外部 SDK

- `@mariozechner/pi-agent-core` — Agent 消息模型、事件定义
- `@mariozechner/pi-ai` — `streamSimple()` LLM API 调用
- `@mariozechner/pi-coding-agent` — `createAgentSession()`, `SessionManager`, `SettingsManager`, coding tools (read/write/edit)

## 关键常量

- `DEFAULT_PROVIDER = "anthropic"`, `DEFAULT_MODEL = "claude-sonnet-4-20250514"`
- `DEFAULT_CONTEXT_TOKENS = 200_000`
- `MAX_OVERFLOW_COMPACTION_ATTEMPTS = 3`
- `BLOCK_REPLY_SEND_TIMEOUT_MS = 15_000`
- `TRANSIENT_HTTP_RETRY_DELAY_MS = 2_500`
