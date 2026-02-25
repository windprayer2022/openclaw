# Session 管理系统

> 源码阅读笔记 — 2026-02-25

## 概述

Session 是 OpenClaw 中贯穿消息生命周期的核心状态单元。每个用户/群聊与 Agent 的对话对应一个 Session，承载会话元数据、LLM 配置覆盖、token 统计、transcript 文件路径等。

## 关键文件

| 文件 | 职责 |
|------|------|
| `src/config/sessions/types.ts` | SessionEntry 类型定义、mergeSessionEntry |
| `src/config/sessions/store.ts` | 持久化层：缓存、写锁队列、CRUD、维护 |
| `src/config/sessions/paths.ts` | 路径解析：store 路径、transcript 路径 |
| `src/config/sessions/reset.ts` | 重置策略：daily/idle 模式、freshness 评估 |
| `src/config/sessions/transcript.ts` | Transcript 写入：JSONL header + message |
| `src/config/sessions/metadata.ts` | 入站消息元数据 → SessionEntry patch |
| `src/config/sessions/group.ts` | 群聊 key 解析、displayName 生成 |
| `src/config/sessions/main-session.ts` | 主 session key 解析 |
| `src/sessions/session-key-utils.ts` | Session key 解析工具 |
| `src/sessions/send-policy.ts` | 发送策略（allow/deny 规则） |
| `src/sessions/input-provenance.ts` | 消息来源标记 |
| `src/sessions/transcript-events.ts` | Transcript 变更事件广播 |
| `src/auto-reply/reply/session.ts` | initSessionState() — 每条消息的 session 初始化入口 |

## 1. Session Key 寻址

格式: `agent:{agentId}:{rest}`

示例:
- `agent:main:webchat:user@example.com` — webchat 直聊
- `agent:main:telegram:group:chat_id` — Telegram 群聊
- `agent:main:slack:channel:general` — Slack 频道
- `agent:main:cron:daily:run:task-id` — 定时任务
- `subagent:sandbox-001` — 子 Agent
- `acp:agent-123` — ACP 协议

解析工具（session-key-utils.ts）:
- `parseAgentSessionKey()` — 提取 agentId + rest
- `isCronRunSessionKey()` / `isSubagentSessionKey()` / `isAcpSessionKey()` — 类型判断
- `resolveThreadParentSessionKey()` — `:thread:` / `:topic:` 标记提取父 key

群聊 key 解析（group.ts）: `resolveGroupSessionKey(ctx)` 从 MsgContext 推断群聊归属 → `{key, channel, id, chatType}`

## 2. Session Store 持久化

文件: `~/.openclaw/state/agents/{agentId}/sessions/sessions.json`

结构: `Record<string, SessionEntry>` — sessionKey → 元数据字典

### 三层机制

**(a) 内存缓存**
- `SESSION_STORE_CACHE: Map<storePath, {store, loadedAt, mtimeMs}>`
- TTL 45s（可配）+ 文件 mtime 双重校验
- 返回 structuredClone() 防止外部修改污染缓存
- 写入时立即 invalidate

**(b) 写锁队列**
- 每 storePath 一个 LOCK_QUEUES，进程内串行
- 跨进程 acquireSessionWriteLock（文件锁）
- timeout 10s，stale 30s
- `updateSessionStore()` 在锁内 re-read 避免覆盖并发写

**(c) 原子写入**
- 非 Windows: writeFile(tmp) → rename → chmod(0o600)
- Windows: 直接 writeFile + 依赖写锁

**(d) 加载迁移**
- `provider` → `channel`, `lastProvider` → `lastChannel`, `room` → `groupChannel`

## 3. SessionEntry 数据模型（~50+ 字段）

| 分组 | 字段 |
|------|------|
| 身份 | sessionId, updatedAt, sessionFile |
| 继承 | spawnedBy |
| LLM 覆盖 | modelOverride, providerOverride, authProfileOverride, thinkingLevel, verboseLevel, reasoningLevel, elevatedLevel |
| 执行环境 | execHost, execSecurity, execAsk, execNode |
| 队列 | queueMode, queueDebounceMs, queueCap, queueDrop |
| Token | inputTokens, outputTokens, totalTokens, totalTokensFresh, contextTokens |
| Compaction | compactionCount, memoryFlushAt, memoryFlushCompactionCount |
| 群组 | chatType, channel, groupId, subject, groupChannel, space, displayName |
| 路由 | lastChannel, lastTo, lastAccountId, lastThreadId, deliveryContext |
| 来源 | origin: SessionOrigin |
| 快照 | skillsSnapshot, systemPromptReport |
| 心跳 | lastHeartbeatText, lastHeartbeatSentAt |

## 4. Session 生命周期（initSessionState）

每条入站消息调用 `initSessionState(ctx, cfg, commandAuthorized)`:

```
解析 sessionKey → loadSessionStore → 检查 reset trigger (/new, /reset)
  → 解析 resetType (direct|group|thread) → 解析 resetPolicy (daily|idle)
  → evaluateSessionFreshness()
  → 决定: 复用(fresh) or 新建
  → deriveGroupSessionPatch() / forkSessionFromParent()
  → updateSessionStore() 写入
  → archiveSessionTranscripts() (reset 时归档)
  → 触发 session_end + session_start 钩子
```

### Reset 策略分层

```
session.reset (基础) → session.resetByType.{direct|group|thread} → session.resetByChannel.{channel}
```

- **daily**: updatedAt < 今天 atHour(默认4点) → 过期
- **idle**: now > updatedAt + idleMinutes(默认60分钟) → 过期

### Reset Trigger

- `/new`, `/reset`（大小写不敏感）
- 支持附带消息: `/new 帮我写个函数` → 重置但保留文字
- 重置时保留 thinking/verbose/reasoning/ttsAuto 覆盖，清零 compaction/token

### Session Fork（线程）

线程消息有 ParentSessionKey 时:
1. 优先用 SessionManager.createBranchedSession(leafId)
2. 失败回退: 手动创建新 .jsonl，header 记录 parentSession

## 5. Transcript 日志

路径: `~/.openclaw/state/agents/{agentId}/sessions/{sessionId}.jsonl`
线程: `{sessionId}-topic-{topicId}.jsonl`

JSONL 格式:
- 首行 header: `{type: "session", version, id, timestamp, cwd}`
- 后续行: `{type: "message", role, content, usage, ...}`

写入流程 (`appendAssistantMessageToSessionTranscript`):
1. resolveSessionFilePath → 定位文件
2. ensureSessionHeader → 写头行
3. SessionManager.open().appendMessage() → 追加消息
4. updateSessionStore → 更新 sessionFile
5. emitSessionTranscriptUpdate → 广播事件

## 6. Session 维护

| 机制 | 默认阈值 | 行为 |
|------|----------|------|
| pruneStaleEntries | 30 天 | 删除过期 entry |
| capEntryCount | 500 条 | 保留最新 N 条 |
| rotateSessionFile | 10 MB | rename .bak + 保留 3 备份 |

两种模式 (`session.maintenance.mode`):
- **warn** (默认): 仅日志 + 回调，不清理
- **auto**: saveSessionStore 时自动 prune + cap + rotate

## 7. 辅助机制

### Send Policy

`resolveSendPolicy()`: entry 覆盖 → 规则匹配(channel/chatType/keyPrefix) → 默认 allow

### Input Provenance

标记消息来源: external_user | inter_session | internal_system

### Transcript Events

观察者模式: `onSessionTranscriptUpdate()` 注册 → `emitSessionTranscriptUpdate()` 广播

## 架构总结

```
initSessionState() — 每条消息入口
  ├─ Session Key Resolution
  ├─ Reset / Freshness 评估
  └─ Fork (线程)
         ↓
Session Store (config/sessions/store)
  ├─ Cache (45s + mtime)
  ├─ Lock Queue (进程内串行 + 跨进程 flock)
  └─ Maintenance (prune/cap/rotate)
  │
  sessions.json: {sessionKey → SessionEntry}
         ↓
Transcript (config/sessions/transcript)
  {sessionId}.jsonl: header + messages
```

## 8. Session 与 Gateway / Channel / Agent 的关系

Session 是连接其他三个模块的中枢数据结构。

### 全局关系图

```
┌──────────────────────────────────────────────────────────────┐
│                     Gateway (WebSocket)                       │
│  ┌────────────────┐  ┌───────────────────────────────────┐   │
│  │ sessions.list  │  │ chat.send                         │   │
│  │ sessions.patch │  │   → dispatchReplyFromConfig       │   │
│  │ sessions.reset │  │   → getReplyFromConfig            │   │
│  │ sessions.delete│  │   → initSessionState() ←──────┐  │   │
│  │ sessions.compact  │                               │  │   │
│  └───────┬────────┘  └──────────┬────────────────────┼──┘   │
│          │ 查询/管理            │ 消息处理           │       │
└──────────┼──────────────────────┼────────────────────┼───────┘
           │                      │                    │
           ↓                      ↓                    │
┌─────────────────────┐   ┌──────────────┐            │
│  Session Store      │   │  Channel     │     ① 构造 SessionKey
│  sessions.json      │←──│  Adapter     │────────────┘
│  {key→SessionEntry} │   └──────────────┘
│  Transcript         │          ↑
│  {id}.jsonl         │          │ ④ 回复路由
└─────────┬───────────┘          │
          │ ③ 读取上下文         │
          ↓                      │
┌──────────────────────────────────────────┐
│  Agent Runtime (Pi)                       │
│  SessionManager.open(transcript.jsonl)    │
│  session.prompt() → LLM → 工具 → 回复    │
└──────────────────────────────────────────┘
```

### ① Channel → Session: 构造 SessionKey

Channel Adapter 收到消息后，首先确定消息属于哪个 Session。

关键文件: `src/routing/resolve-route.ts`, `src/routing/session-key.ts`

```
resolveAgentRoute(cfg, channel, accountId, peer)
  → 遍历 cfg.bindings[] (7 级优先级) 确定 agentId
  → buildAgentSessionKey(agentId, channel, accountId, peer, dmScope)
  → buildAgentPeerSessionKey()
```

dmScope 决定 DM 会话隔离粒度:
- `"main"` → `agent:main:main` (所有 DM 共享一个 session)
- `"per-peer"` → `agent:main:direct:alice`
- `"per-channel-peer"` → `agent:main:telegram:direct:alice`
- `"per-account-channel-peer"` → `agent:main:telegram:bot1:direct:alice`

群聊: `agent:main:telegram:group:chat_id`
线程: `agent:main:telegram:group:chat_id:thread:topic_123`

Channel Adapter 还调用 `recordInboundSession()` (`src/channels/session.ts`):
- `recordSessionMetaFromInbound()` → 更新 origin, displayName, chatType
- `updateLastRoute()` → 记录 lastChannel, lastTo, lastAccountId, lastThreadId (回复路由的关键)

### ② Session → Agent: 初始化会话状态

`getReplyFromConfig()` 调用 `initSessionState()` 产生 `SessionInitResult`:

Agent Runtime 从中获取:
- `sessionEntry.sessionFile` → 打开 SessionManager 读取历史上下文
- `sessionEntry.modelOverride` → 选择 LLM 模型
- `sessionEntry.authProfileOverride` → 选择 API 凭证
- `sessionEntry.thinkingLevel` → 设置 thinking 参数
- `sessionEntry.compactionCount` → 判断是否需要 compaction
- `isNewSession` → 决定是否发 system prompt

### ③ Agent → Session: 写回状态

Agent 完成推理后:
1. `SessionManager.appendMessage()` → 追加 assistant 消息到 transcript
2. `updateSessionStore()` → 写回 totalTokens, compactionCount, model, updatedAt
3. `emitSessionTranscriptUpdate()` → 通知 Gateway 有新消息

### ④ Session → Channel: 回复路由

Session 的 `deliveryContext` 是路由地图:

```
routeReply({                           // src/auto-reply/reply/route-reply.ts
  channel: "telegram",                 // ← deliveryContext.channel
  to: "chat_id_123",                   // ← lastTo
  threadId: "topic_456",               // ← lastThreadId
  accountId: "bot1",                   // ← lastAccountId
})
  → deliverOutboundPayloads()
  → Channel Adapter.sendText()
  → 平台 API
```

当 dmScope="main" 时多平台共享 session，但 lastChannel/lastTo 记录最后消息来源，确保回复路由正确。

### ⑤ Gateway → Session: 管理 API

Gateway 通过 WebSocket RPC 暴露 6 个方法 (`src/gateway/server-methods/sessions.ts`):

| RPC 方法 | 功能 | 读/写 |
|----------|------|-------|
| sessions.list | 列出所有会话（搜索、过滤） | 读 |
| sessions.preview | 会话消息预览（从 transcript） | 读 |
| sessions.patch | 修改会话参数 | 写 |
| sessions.reset | 重置会话（归档旧 transcript） | 写 |
| sessions.delete | 删除会话（中止 Agent，归档） | 写 |
| sessions.compact | 手动触发上下文压缩 | 写 |

### 四模块关系矩阵

| 关系 | Session 角色 | 关键数据流 |
|------|-------------|------------|
| Channel → Session | 寻址目标 | 构造 SessionKey，写入 origin/deliveryContext |
| Session → Agent | 状态供给 | 提供 sessionId, modelOverride, transcript 路径 |
| Agent → Session | 结果存储 | 写回 token 统计, compaction, 追加 transcript |
| Session → Channel | 路由地图 | deliveryContext 指示回复送往哪个平台/聊天/线程 |
| Gateway → Session | 管理接口 | 6 个 RPC 方法 (list/preview/patch/reset/delete/compact) |
