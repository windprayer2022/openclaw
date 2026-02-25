# 类型系统与数据模型

> OpenClaw 的核心类型并不集中在 `src/types/`（那里只有第三方 `.d.ts` 声明），而是**分散在各模块**中，通过 barrel export 聚合。整体采用 **「双 Schema 体系」**：配置层用 TypeScript 类型 + Zod 校验，协议层用 TypeBox Schema + `Static<>` 类型提取。

## 1. 类型分布拓扑

```
┌─ CONFIG 层 ─────────────────── src/config/
│  types.ts (barrel) → 31 个聚焦模块
│  ├─ types.base.ts        核心枚举/原语
│  ├─ types.openclaw.ts    ★ 顶层 OpenClawConfig
│  ├─ types.agents.ts      Agent 配置
│  ├─ types.channels.ts    Channel 配置 (含 9 个平台子模块)
│  ├─ types.messages.ts    消息行为配置
│  ├─ types.tools.ts       工具策略 + 媒体理解 + 记忆搜索
│  ├─ types.models.ts      Model/Provider 定义
│  ├─ types.auth.ts        Auth Profile 配置
│  └─ (其余: sandbox, cron, hooks, plugins, skills, tts, ...)
│
├─ SESSION 层 ────────────────── src/config/sessions/
│  types.ts → SessionEntry, SessionOrigin, SessionSkillSnapshot,
│             SessionSystemPromptReport
│
├─ GATEWAY 协议层 ────────────── src/gateway/protocol/schema/
│  frames.ts      线格式 (TypeBox): RequestFrame, ResponseFrame, EventFrame
│  snapshot.ts    全局快照: Snapshot, PresenceEntry, StateVersion
│  types.ts       barrel → 190+ Static<> 类型 (按领域分文件)
│  └─ 11 个领域文件: agent, channels, config, sessions, logs-chat,
│     cron, devices, exec-approvals, wizard, nodes, agents-models-skills
│
├─ CHANNEL 插件层 ────────────── src/channels/plugins/
│  types.core.ts      核心原语: ChannelId, ChannelMeta, ChannelCapabilities,
│                     ChannelAccountSnapshot, ChannelGroupContext
│  types.adapters.ts  15 个 Adapter 类型 (Setup/Config/Outbound/Status/
│                     Gateway/Auth/Heartbeat/Directory/Pairing/Security/
│                     Group/Command/Elevated/Resolver/MessageAction)
│  types.plugin.ts    ChannelPlugin 契约 (20 个插槽)
│
├─ AGENT 运行时层 ────────────── src/agents/
│  auth-profiles/types.ts        Credential 三态, ProfileUsageStats, AuthProfileStore
│  pi-embedded-runner/types.ts   EmbeddedPiRunResult, RunMeta, CompactResult
│  skills/types.ts               SkillEntry, SkillInstallSpec, SkillSnapshot
│
├─ 消息/回复层 ───────────────── src/auto-reply/
│  types.ts → ReplyPayload, GetReplyOptions, ModelSelectedContext
│
├─ 插件 API 层 ───────────────── src/plugins/
│  types.ts (563 行)     OpenClawPluginApi, PluginDefinition, ProviderPlugin,
│                        15 个 Hook 类型, PluginHookHandlerMap (判别联合)
│  runtime/types.ts      PluginRuntime (运行时能力: config/system/media/tools/channel/...)
│
└─ 其他
   src/channels/chat-type.ts     ChatType = "direct" | "group" | "channel"
   src/channels/registry.ts      8 个内置 ChatChannelId + ChannelMeta 元数据
```

## 2. 顶层配置：OpenClawConfig

**文件**: `src/config/types.openclaw.ts`

这是整个系统的**配置根类型**，对应 `openclaw.json5` 配置文件。26 个顶层字段：

```typescript
type OpenClawConfig = {
  meta?       // 版本追踪 (lastTouchedVersion, lastTouchedAt)
  auth?       // Auth Profile 声明 + 轮换策略
  env?        // 环境变量注入 (shellEnv, inline vars)
  wizard?     // 初始化向导状态
  diagnostics? // 诊断 (OTel, cacheTrace)
  logging?    // 日志 (level, consoleStyle, redact)
  update?     // 自动更新 (channel: stable|beta|dev)
  browser?    // 浏览器工具配置
  ui?         // UI 定制 (seamColor, assistant avatar)
  skills?     // 技能配置
  plugins?    // 插件配置
  models?     // 模型提供商 (providers, bedrockDiscovery)
  nodeHost?   // 远程节点
  agents?     // Agent 配置 (defaults + list)
  tools?      // 工具策略 (profile, allow/deny, web/media/links/exec/...)
  bindings?   // Agent-Channel 路由绑定
  broadcast?  // 广播策略
  audio?      // 音频转写 (deprecated)
  messages?   // 消息行为 (prefix, queue, ack, tts)
  commands?   // 命令权限 (native, bash, config, debug, allowFrom)
  approvals?  // 执行审批
  session?    // 会话配置 (scope, reset, maintenance)
  web?        // WhatsApp Web provider
  channels?   // 通道配置 (defaults + 9 内置 + 扩展 [key: string])
  cron?       // 定时任务
  hooks?      // 钩子
  discovery?  // 服务发现
  canvasHost? // Canvas 宿主
  talk?       // 语音对话
  gateway?    // Gateway 行为
  memory?     // 记忆系统
}
```

同文件还定义了：
- **ConfigFileSnapshot** — 配置文件的完整生命周期快照 (raw → parsed → resolved → config → hash → issues)
- **ConfigValidationIssue / LegacyConfigIssue** — 校验/迁移问题

## 3. 基础原语类型 (types.base.ts)

| 类型 | 值 | 用途 |
|------|-----|------|
| `ReplyMode` | `"text" \| "command"` | 回复模式 |
| `TypingMode` | `"never" \| "instant" \| "thinking" \| "message"` | 打字指示器 |
| `SessionScope` | `"per-sender" \| "global"` | 会话隔离粒度 |
| `DmScope` | `"main" \| "per-peer" \| "per-channel-peer" \| "per-account-channel-peer"` | DM 会话隔离 |
| `ReplyToMode` | `"off" \| "first" \| "all"` | 消息引用策略 |
| `GroupPolicy` | `"open" \| "disabled" \| "allowlist"` | 群聊准入策略 |
| `DmPolicy` | `"pairing" \| "allowlist" \| "open" \| "disabled"` | DM 准入策略 |
| `ChatType` | `"direct" \| "group" \| "channel"` | 三种聊天类型 |
| `ChannelId` | `ChatChannelId \| (string & {})` | 通道标识 (8 内置 + 扩展) |
| `ChatChannelId` | 8 个字面量联合 | telegram/whatsapp/discord/irc/googlechat/slack/signal/imessage |

重要的复合配置类型：
- **SessionConfig** — 会话配置（scope、reset 策略、maintenance）
- **SessionResetConfig** — 重置策略 (daily/idle + atHour/idleMinutes)
- **SessionSendPolicyConfig** — 发送策略 (allow/deny + rules 匹配)
- **OutboundRetryConfig** — 出站重试 (attempts, delay, jitter)
- **HumanDelayConfig** — 仿人延迟 (off/natural/custom)
- **MarkdownConfig** — Markdown 渲染 (tables: off/bullets/code)
- **BlockStreamingCoalesceConfig / BlockStreamingChunkConfig** — 流式分块

## 4. Agent 配置类型 (types.agents.ts)

```typescript
type AgentConfig = {
  id: string              // 唯一标识
  default?: boolean       // 是否默认 agent
  name?: string           // 显示名
  workspace?: string      // 工作区路径
  model?: AgentModelConfig  // 模型 (string | {primary, fallbacks})
  skills?: string[]       // 技能白名单
  identity?: IdentityConfig // 身份 (name, emoji, avatar)
  groupChat?              // 群聊 (mentionPatterns, historyLimit)
  subagents?              // 子 agent 策略
  sandbox?                // 沙箱配置 (mode, workspaceAccess, scope, docker, browser)
  tools?: AgentToolsConfig // 工具策略
  // ...
}

type AgentBinding = {
  agentId: string
  match: { channel, accountId?, peer?, guildId?, teamId?, roles? }
}
```

**AgentBinding** 是 Agent-Channel 路由的核心：通过 channel + accountId + peer + guild/team + roles 多维匹配，将特定消息路由到特定 Agent。

## 5. 工具策略类型 (types.tools.ts) — 最复杂的配置模块

**ToolsConfig** (全局工具配置) 涵盖：
- **profile** — 4 个预置工具集 (minimal / coding / messaging / full)
- **allow / deny / alsoAllow** — 白名单/黑名单
- **byProvider** — 按模型提供商覆写
- **web** — 网页搜索 (brave/perplexity/grok) + 网页抓取 (readability/firecrawl)
- **media** — 图片/音频/视频理解 (多模型 fallback, CLI/Provider 双模式)
- **links** — 链接理解
- **message** — 消息工具 (跨上下文发送, 广播)
- **agentToAgent** — Agent 间通信
- **elevated** — 提权执行 (allowFrom 控制)
- **exec** — 命令执行 (host: sandbox/gateway/node, security: deny/allowlist/full)
- **subagents / sandbox** — 子 agent/沙箱的工具策略

**MemorySearchConfig** (记忆搜索配置) 极其详细：
- provider (openai/gemini/local/voyage)
- local (node-llama-cpp GGUF)
- store (SQLite + sqlite-vec 向量)
- chunking + sync + query (hybrid BM25+vector)

## 6. Model Provider 类型 (types.models.ts)

```typescript
type ModelApi = "openai-completions" | "openai-responses" |
  "anthropic-messages" | "google-generative-ai" |
  "github-copilot" | "bedrock-converse-stream" | "ollama"

type ModelDefinitionConfig = {
  id, name, api?, reasoning: boolean,
  input: ("text"|"image")[],
  cost: { input, output, cacheRead, cacheWrite },
  contextWindow, maxTokens,
  headers?, compat?: ModelCompatConfig
}

type ModelProviderConfig = {
  baseUrl, apiKey?, auth?, api?, headers?,
  models: ModelDefinitionConfig[]
}
```

**ModelCompatConfig** 处理不同 API 的兼容性差异：supportsStore、thinkingFormat (openai/zai/qwen)、maxTokensField 等。

## 7. Gateway 协议类型 — 双 Schema 体系

**TypeBox Schema** 定义线格式 → **`Static<>`** 提取 TS 类型：

```
schema/*.ts (TypeBox)  →  schema/types.ts (Static<> barrel)  →  运行时代码
     ↓                                                            ↑
  AJV 运行时校验                                           TypeScript 编译时类型
```

### 线格式 (frames.ts)
```typescript
RequestFrame  = { type: "req",   id, method, params? }
ResponseFrame = { type: "res",   id, ok, payload?, error? }
EventFrame    = { type: "event", event, payload?, seq?, stateVersion? }
GatewayFrame  = Union(Request | Response | Event, discriminator: "type")
```

### 握手
```
Client → ConnectParams { minProtocol, maxProtocol, client, caps?, auth?, device? }
Server → HelloOk { protocol, server, features, snapshot, policy, auth? }
```

HelloOk 中的 **Snapshot** 携带系统全局状态：
```typescript
Snapshot = { presence[], health, stateVersion, uptimeMs, configPath?, sessionDefaults? }
```

### 190+ 协议类型 (按领域)
每个领域文件定义 TypeBox Schema，types.ts 用 `Static<>` 统一导出：
- **agent**: AgentEvent, PollParams, WakeParams, AgentIdentityParams
- **sessions**: SessionsList/Preview/Resolve/Patch/Reset/Delete/Compact/Usage
- **config**: ConfigGet/Set/Apply/Patch/Schema
- **channels**: ChannelsStatus, TalkMode, WebLogin
- **agents-models-skills**: AgentsCreate/Update/Delete/FilesList, ModelsList, SkillsInstall/Status
- **cron**: CronAdd/Update/Remove/Run/Status
- **devices**: DevicePairList/Approve/Reject, TokenRotate/Revoke
- **exec-approvals**: ExecApprovalsGet/Set, ApprovalRequest/Resolve
- **wizard**: WizardStart/Next/Cancel/Status
- **nodes**: NodePairRequest/Approve/Reject, NodeInvoke/Event
- **logs-chat**: ChatAbort, ChatInject, ChatEvent, LogsTail

## 8. Channel 插件类型 — 三层架构

### 8.1 Core 原语 (types.core.ts)
- **ChannelMeta** — 通道元数据 (label, blurb, docsPath, order, aliases, systemImage)
- **ChannelCapabilities** — 通道能力 (chatTypes, polls, reactions, edit, threads, nativeCommands, ...)
- **ChannelAccountSnapshot** — 账户运行时快照 (30+ 字段：enabled, configured, connected, lastError, ...)
- **ChannelAccountState** — 6 态：linked / not linked / configured / not configured / enabled / disabled
- **ChannelStatusIssue** — 状态问题 (kind: intent/permissions/config/auth/runtime)
- **ChannelGroupContext** — 群聊上下文 (groupId, groupChannel, senderId, ...)

### 8.2 Adapter 接口 (types.adapters.ts) — 15 个
| Adapter | 职责 |
|---------|------|
| SetupAdapter | 账户设置 (resolveAccountId, applyAccountConfig, validateInput) |
| ConfigAdapter\<R\> | 配置读写 (listAccountIds, resolveAccount, isEnabled, isConfigured) |
| GroupAdapter | 群组策略 (requireMention, groupIntroHint, toolPolicy) |
| OutboundAdapter | 出站发送 (sendText, sendMedia, sendPoll, chunker, resolveTarget) |
| StatusAdapter\<R,P,A\> | 状态探测 (probeAccount, auditAccount, buildAccountSnapshot) |
| GatewayAdapter\<R\> | 生命周期 (startAccount, stopAccount, loginWithQr, logoutAccount) |
| AuthAdapter | 认证 (login) |
| HeartbeatAdapter | 心跳 (checkReady, resolveRecipients) |
| DirectoryAdapter | 通讯录 (self, listPeers, listGroups, listGroupMembers) |
| PairingAdapter | 配对 (idLabel, normalizeAllowEntry, notifyApproval) |
| SecurityAdapter\<R\> | 安全 (resolveDmPolicy, collectWarnings) |
| ElevatedAdapter | 提权 (allowFromFallback) |
| CommandAdapter | 命令 (enforceOwnerForCommands, skipWhenConfigEmpty) |
| ResolverAdapter | 目标解析 (resolveTargets) |
| MessageActionAdapter | 消息动作 (handleAction, supportsButtons, supportsCards) |

还有 3 个嵌入式 Adapter (定义在 types.core.ts)：
- **MentionAdapter** — @提及处理
- **StreamingAdapter** — 流式参数默认值
- **ThreadingAdapter** — 线程/引用策略

### 8.3 ChannelPlugin 契约 (types.plugin.ts)
组合以上所有 Adapter 为一个完整的插件契约 (20 个插槽)。

## 9. Plugin API 类型 (src/plugins/types.ts)

### OpenClawPluginApi — 插件注册 API
```typescript
type OpenClawPluginApi = {
  id, name, version?, description?, source
  config: OpenClawConfig
  pluginConfig?: Record<string, unknown>
  runtime: PluginRuntime          // 运行时能力
  logger: PluginLogger
  registerTool(...)               // 注册 Agent 工具
  registerHook(events, handler)   // 注册钩子
  registerHttpHandler(...)        // 注册 HTTP 处理器
  registerHttpRoute(...)          // 注册 HTTP 路由
  registerChannel(...)            // 注册通道插件
  registerGatewayMethod(...)      // 注册 Gateway RPC 方法
  registerCli(...)                // 注册 CLI 命令
  registerService(...)            // 注册后台服务
  registerProvider(...)           // 注册 Model Provider
  registerCommand(...)            // 注册聊天命令
  on<K>(hookName, handler)        // 类型安全钩子订阅
}
```

### 15 个生命周期 Hook
按时序：
1. **gateway_start / gateway_stop** — Gateway 启停
2. **session_start / session_end** — 会话开始/结束
3. **before_agent_start / agent_end** — Agent 运行前后
4. **message_received / message_sending / message_sent** — 消息收发
5. **before_tool_call / after_tool_call / tool_result_persist** — 工具调用
6. **before_compaction / after_compaction** — 上下文压缩
7. **before_reset** — 会话重置

Hook 签名通过 **PluginHookHandlerMap** 实现判别联合，确保类型安全：
```typescript
type PluginHookHandlerMap = {
  before_agent_start: (event, ctx) => Result | void
  message_sending: (event, ctx) => { content?, cancel? } | void  // 可修改/取消
  before_tool_call: (event, ctx) => { params?, block? } | void   // 可修改参数/拦截
  tool_result_persist: (event, ctx) => { message? } | void       // 可修改持久化内容
  // ... 其余为纯通知 (void)
}
```

### Provider Plugin
```typescript
type ProviderPlugin = {
  id, label, docsPath?, aliases?, envVars?,
  models?: ModelProviderConfig,
  auth: ProviderAuthMethod[],      // 多种认证方式
  formatApiKey?, refreshOAuth?
}
```

## 10. Auth Profile 类型 (src/agents/auth-profiles/types.ts)

凭据三态判别联合：
```typescript
type AuthProfileCredential =
  | { type: "api_key";  provider, key?, email? }
  | { type: "token";    provider, token, expires?, email? }
  | { type: "oauth";    provider, clientId?, email?, ...OAuthCredentials }
```

**AuthProfileStore** — 持久化存储：
```typescript
{ version, profiles: Record<id, Credential>,
  order?: Record<agentId, profileIds[]>,
  lastGood?: Record<provider, profileId>,
  usageStats?: Record<id, ProfileUsageStats> }
```

**ProfileUsageStats** — 追踪轮换状态：lastUsed, cooldownUntil, disabledUntil, errorCount, failureCounts

## 11. SessionEntry — 最"厚"的运行时状态

**文件**: `src/config/sessions/types.ts`

SessionEntry 是整个系统中字段最多的类型 (40+ 字段)，它代表一个会话的完整运行时状态：

| 领域 | 字段 |
|------|------|
| 标识 | sessionId, sessionFile, spawnedBy |
| 时间 | updatedAt |
| 聊天类型 | chatType |
| Agent 控制 | thinkingLevel, verboseLevel, reasoningLevel, elevatedLevel |
| TTS | ttsAuto |
| 执行 | execHost, execSecurity, execAsk, execNode |
| 模型覆写 | providerOverride, modelOverride, authProfileOverride (+ source, compactionCount) |
| 群组 | groupActivation, groupActivationNeedsSystemIntro |
| 策略 | sendPolicy, queueMode, queueDebounceMs, queueCap, queueDrop |
| Token 追踪 | inputTokens, outputTokens, totalTokens, totalTokensFresh, contextTokens |
| 模型信息 | modelProvider, model |
| 压缩 | compactionCount, memoryFlushAt, memoryFlushCompactionCount |
| 路由 | channel, groupId, groupChannel, space, origin, deliveryContext |
| 最后消息 | lastChannel, lastTo, lastAccountId, lastThreadId |
| 心跳 | lastHeartbeatText, lastHeartbeatSentAt |
| 快照 | skillsSnapshot, systemPromptReport |
| 显示 | label, displayName, subject |
| 外部会话 | cliSessionIds, claudeCliSessionId, responseUsage |
| 系统 | systemSent, abortedLastRun |

## 12. 设计模式总结

### 12.1 类型组织策略
- **barrel export** — 每个模块的 `types.ts` 聚合所有子模块类型
- **按领域拆分** — `types.{domain}.ts` 命名约定，编辑局部性好
- **不超过 ~300 行/文件** — 配置类型模块普遍 30-130 行

### 12.2 双 Schema 体系 (详见 AD-1)
| 用途 | Schema | 校验 | 类型提取 |
|------|--------|------|----------|
| 配置 (openclaw.json5) | Zod | `OpenClawSchema.parse()` | TypeScript 手写类型 |
| Gateway 协议 (WebSocket) | TypeBox | AJV compiled | `Static<typeof Schema>` |

为什么不统一？
- **配置**: Zod 对复杂 transform/default/refinement 支持好，适合配置校验
- **协议**: TypeBox 生成标准 JSON Schema，适合跨语言 codegen (Swift/Kotlin 客户端)

### 12.3 Adapter 模式
Channel 系统用 **15+ Adapter 接口** 将关注点分离，每个通道插件按需实现。这是 OpenClaw 最密集的类型设计区域。

### 12.4 类型安全钩子
`PluginHookHandlerMap` 用映射类型 + 判别联合，让 `api.on("before_tool_call", handler)` 的 handler 参数自动推导为正确的事件+上下文类型。

### 12.5 渐进式可选 (Optional-heavy)
几乎所有配置字段都是 `?` 可选的。系统在 `config.ts` / `defaults.ts` 里提供运行时默认值，用户只需配置想改的部分。这是 "Convention over Configuration" 的体现。
