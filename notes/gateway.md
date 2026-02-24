# Gateway 核心流程阅读笔记

> 阅读日期: 2026-02-14

## 概述

Gateway 是 OpenClaw 的中枢神经系统 — 一个基于 WebSocket 的控制面（Control Plane）。所有客户端（iOS/macOS App、Web UI、CLI、Telegram Bot 等）都通过 WebSocket 连接到 Gateway，由 Gateway 统一路由请求、管理状态、协调 Agent 运行。

核心代码位于 `src/gateway/`，约 136 个非测试 TS 文件，~28,000 行。

## 架构分层

```
Layer 1: 入口 & 初始化          server.ts / server.impl.ts
Layer 2: WebSocket 连接管理     server/ws-connection.ts
Layer 3: 消息处理 & 认证        message-handler.ts / auth.ts
Layer 4: 请求分发 & 权限        server-methods.ts
Layer 5: 业务处理器             server-methods/*.ts (24个领域)
Layer 6: 协议定义 & 校验        protocol/ (TypeBox + AJV)
```

## 关键文件

| 文件 | 职责 |
|------|------|
| `server.ts` | 公共 API barrel（仅 3 行 re-export） |
| `server.impl.ts` | 真正的入口 `startGatewayServer()`，组装所有子系统 |
| `server/ws-connection.ts` | WebSocket 连接生命周期管理 |
| `server/ws-connection/message-handler.ts` | 消息解析、握手认证、请求分发 |
| `server-methods.ts` | RPC 路由 + scope 权限检查 |
| `auth.ts` | 认证逻辑（token/password/tailscale/device-token） |
| `auth-rate-limit.ts` | 认证速率限制（按 IP+scope） |
| `protocol/index.ts` | 协议版本、帧格式、Schema 校验器 |
| `server-chat.ts` | 聊天运行注册、Agent 事件聚合 |
| `server-methods/chat.ts` | 聊天/推理的 RPC handler |
| `config-reload.ts` | 配置热重载（file watcher） |
| `server-channels.ts` | Channel 插件初始化/启停 |

## 启动流程 (`startGatewayServer`)

入口在 `server.impl.ts:160`，默认端口 18789。

```
1. 读取配置文件 → 校验 → 自动迁移旧格式
2. 加载插件（plugin registry）
3. 枚举 Channel 插件（WhatsApp/Telegram/Slack/...）← 见下方展开
4. 解析运行时配置（绑定地址、认证模式、TLS）
5. 创建运行时状态（WS Server, HTTP Server, 广播系统）
6. 启动子系统：
   ├─ NodeRegistry（远程节点注册）
   ├─ ChannelManager（消息通道管理）← Channel 在此启动
   ├─ Discovery（mDNS/Bonjour 服务发现）
   ├─ CronService（定时任务）
   ├─ MaintenanceTimers（健康检查、去重清理）
   ├─ AgentEventHandler（Agent 事件监听 → 广播到客户端）
   ├─ HeartbeatRunner（心跳）
   └─ ExecApprovalManager（执行审批）
7. 挂载 WebSocket 处理器
```

### 第 3 步展开：枚举 Channel 插件

> 核心文件: `server-channels.ts` (ChannelManager) + `channels/plugins/index.ts` (枚举) + `channels/dock.ts` (轻量行为)
> 类型定义: `channels/plugins/types.plugin.ts` + `types.core.ts` + `types.adapters.ts`

#### 3.1 枚举过程 (`server.impl.ts:245-252`)

插件加载（步骤 2）完成后，Channel 从 PluginRegistry 中提取出来：

```
listChannelPlugins()                         // channels/plugins/index.ts
  → requireActivePluginRegistry()            // 拿到已加载的 PluginRegistry
  → registry.channels.map(e => e.plugin)     // 取出所有 ChannelPlugin
  → dedupeChannels()                         // 按 id 去重
  → toSorted(CHAT_CHANNEL_ORDER)             // 按内置顺序排序，外部插件排后面
```

对每个 Channel 创建子系统日志器 + RuntimeEnv，并收集它们声明的额外 gateway RPC 方法
（如 WhatsApp 声明 `"web.login.start"`, `"web.login.wait"`）合并到 gatewayMethods。

#### 3.2 内置 Channel 列表 (8 个核心通道)

定义在 `channels/registry.ts` 的 `CHAT_CHANNEL_ORDER`：

| 序号 | Channel ID | 标签 | 接入方式 | 特点 |
|------|-----------|------|---------|------|
| 0 | `telegram` | Telegram (Bot API) | Bot Token + 长轮询/Webhook | 最简单的入门方式；支持 group/channel/thread |
| 1 | `whatsapp` | WhatsApp (QR link) | QR 码扫描 + Web 协议 | 用你自己的号码；支持 polls/reactions |
| 2 | `discord` | Discord (Bot API) | Bot Token + Gateway | 支持 threads、slash commands、block streaming |
| 3 | `irc` | IRC (Server + Nick) | IRC 协议 | 经典 IRC；textChunkLimit=350（最短） |
| 4 | `googlechat` | Google Chat (Chat API) | HTTP Webhook | Google Workspace；支持 threads |
| 5 | `slack` | Slack (Socket Mode) | Socket Mode | 支持 threads、slash commands |
| 6 | `signal` | Signal (signal-cli) | signal-cli 链接设备 | 最注重隐私；设置较复杂 |
| 7 | `imessage` | iMessage (imsg) | BlueBubbles/本地 | 仍在开发中 |

#### 3.3 扩展 Channel (extensions/ 目录)

除 8 个内置通道外，`extensions/` 目录提供更多通道：

| Extension | 说明 |
|-----------|------|
| `bluebubbles` | iMessage 的 BlueBubbles 后端 |
| `feishu` | 飞书 |
| `line` | LINE（日韩） |
| `matrix` | Matrix 协议 (Element 等) |
| `mattermost` | Mattermost |
| `msteams` | Microsoft Teams |
| `nextcloud-talk` | Nextcloud Talk |
| `nostr` | Nostr 协议 |
| `tlon` | Tlon/Urbit |
| `zalo` / `zalouser` | Zalo（越南） |

每个扩展导出一个 `ChannelPlugin` 对象，在插件加载阶段被 `register(api)` 注册到 PluginRegistry。

#### 3.4 ChannelPlugin 契约 — 20 个 Adapter 插槽

`ChannelPlugin<ResolvedAccount>` 是核心契约（`types.plugin.ts`），包含 20 个插槽：

```
ChannelPlugin {
  id: ChannelId                        // 唯一标识
  meta: ChannelMeta                    // UI 展示信息（label, blurb, docsPath, systemImage 等）
  capabilities: ChannelCapabilities    // 能力声明（见下方）

  // ─── 配置层 ───
  config: ChannelConfigAdapter         // [必须] 账号列表、解析账号、启用/禁用
  configSchema?: ChannelConfigSchema   // JSON Schema + UI hints（给 Web UI 渲染配置表单）
  setup?: ChannelSetupAdapter          // CLI onboarding：验证输入 + 写入配置
  onboarding?: ChannelOnboardingAdapter // CLI 向导步骤
  reload?: { configPrefixes }          // 热重载：哪些配置前缀变化时需要重启通道

  // ─── 安全层 ───
  security?: ChannelSecurityAdapter    // DM 策略（pairing/allowFrom）+ 安全警告
  pairing?: ChannelPairingAdapter      // 配对审批（idLabel + 通知审批消息）
  auth?: ChannelAuthAdapter            // 通道专用登录流程
  commands?: ChannelCommandAdapter     // 命令权限（是否只允许 owner 执行）
  elevated?: ChannelElevatedAdapter    // allowFrom 降级回退

  // ─── 消息层 ───
  outbound?: ChannelOutboundAdapter    // [核心] 出站消息：sendText/sendMedia/sendPoll + 分块
  mentions?: ChannelMentionAdapter     // @提及剥离（如 Discord 的 <@123>）
  messaging?: ChannelMessagingAdapter  // 目标地址标准化（normalizeTarget）
  threading?: ChannelThreadingAdapter  // 线程/回复策略（replyToMode: off/first/all）
  streaming?: ChannelStreamingAdapter  // 流式输出合并参数（minChars/idleMs）
  agentPrompt?: ChannelAgentPromptAdapter // 给 Agent 的提示词

  // ─── 运行时层 ───
  gateway?: ChannelGatewayAdapter      // [核心] startAccount / stopAccount / login / logout
  status?: ChannelStatusAdapter        // 健康探测（probe）+ 审计 + 状态快照
  heartbeat?: ChannelHeartbeatAdapter  // 心跳就绪检查 + 收件人解析
  directory?: ChannelDirectoryAdapter  // 联系人/群组目录查询
  resolver?: ChannelResolverAdapter    // 目标地址解析（name → id）
  actions?: ChannelMessageActionAdapter // 消息动作（react、pin、delete 等）

  // ─── 工具层 ───
  agentTools?: ChannelAgentToolFactory | ChannelAgentTool[]  // Agent 可调用的通道工具
  gatewayMethods?: string[]            // 额外的 Gateway RPC 方法名
}
```

#### 3.5 capabilities — 通道能力声明

```typescript
ChannelCapabilities {
  chatTypes: ("direct"|"group"|"channel"|"thread")[]  // 支持的会话类型
  polls?: boolean           // 投票
  reactions?: boolean       // 表情回应
  edit?: boolean           // 编辑已发消息
  unsend?: boolean         // 撤回消息
  reply?: boolean          // 引用回复
  effects?: boolean        // 消息特效
  groupManagement?: boolean // 群管理
  threads?: boolean        // 线程
  media?: boolean          // 图片/视频/文件
  nativeCommands?: boolean // 原生命令（如 Telegram /command、Discord slash）
  blockStreaming?: boolean // 块模式流式输出（非实时编辑已发消息）
}
```

各通道能力对比：

| 通道 | chatTypes | polls | reactions | threads | media | nativeCmd | blockStream |
|------|----------|-------|-----------|---------|-------|-----------|-------------|
| Telegram | direct,group,channel,thread | - | ✓ | ✓ | ✓ | ✓ | ✓ |
| WhatsApp | direct,group | ✓ | ✓ | - | ✓ | - | - |
| Discord | direct,channel,thread | ✓ | ✓ | ✓ | ✓ | ✓ | - |
| IRC | direct,group | - | - | - | ✓ | - | ✓ |
| Google Chat | direct,group,thread | - | ✓ | ✓ | ✓ | - | ✓ |
| Slack | direct,channel,thread | - | ✓ | ✓ | ✓ | ✓ | - |
| Signal | direct,group | - | ✓ | - | ✓ | - | - |
| iMessage | direct,group | - | ✓ | - | ✓ | - | - |

#### 3.6 ChannelManager — 启停生命周期 (`server-channels.ts`)

Gateway 创建 `ChannelManager` 管理所有通道的运行态：

```
createChannelManager({ loadConfig, channelLogs, channelRuntimeEnvs })
  → 返回 { startChannels, startChannel, stopChannel, markChannelLoggedOut, getRuntimeSnapshot }
```

**内部状态** — 每个 Channel 一个 `ChannelRuntimeStore`：
```
ChannelRuntimeStore {
  aborts: Map<accountId, AbortController>   // 每个账号的中止控制器
  tasks: Map<accountId, Promise>            // 运行中的 startAccount 任务
  runtimes: Map<accountId, ChannelAccountSnapshot>  // 运行时快照
}
```

**startChannel 流程**（`server-channels.ts:96`）：
```
startChannel(channelId, accountId?)
  │
  ├─ getChannelPlugin(channelId)
  │   → 从 PluginRegistry 查找 plugin
  │   → 检查 plugin.gateway.startAccount 是否存在
  │
  ├─ listAccountIds(cfg)
  │   → 一个通道可以有多个账号（如多个 Telegram bot）
  │
  └─ 对每个 accountId 并行启动：
      │
      ├─ 检查是否已在运行（tasks.has(id) → 跳过）
      │
      ├─ 检查是否启用 (plugin.config.isEnabled)
      │   → 未启用: setRuntime({ running:false, lastError:"disabled" })
      │
      ├─ 检查是否已配置 (plugin.config.isConfigured)
      │   → 未配置: setRuntime({ running:false, lastError:"not configured" })
      │
      └─ 启动:
          ├─ 创建 AbortController
          ├─ setRuntime({ running:true, lastStartAt })
          └─ plugin.gateway.startAccount(ctx)  ← 核心！
              │  ctx = { cfg, accountId, account, runtime, abortSignal, log, getStatus, setStatus }
              │
              ├─ Telegram: monitorTelegramProvider()  → 长轮询/webhook 监听
              ├─ WhatsApp: startWhatsAppWeb()         → Puppeteer/WS 连接
              ├─ Discord: startDiscordGateway()       → Discord Gateway WS
              ├─ Slack: startSlackSocketMode()        → Slack Socket Mode
              ├─ Signal: startSignalMonitor()         → signal-cli REST 轮询
              └─ ...
```

**stopChannel 流程**（`server-channels.ts:181`）：
```
stopChannel(channelId, accountId?)
  → 收集所有已知 accountId
  → 对每个 account:
     1. abort.abort()               // 发送 AbortSignal
     2. plugin.gateway.stopAccount() // 调用插件的停止钩子
     3. await task                   // 等待启动任务结束
     4. 清理 store 状态
```

**startChannels**（`server-channels.ts:236`）— 启动时遍历所有已注册的 Channel：
```
for plugin of listChannelPlugins():
  await startChannel(plugin.id)    // 顺序启动（不是并行）
```

#### 3.7 Dock — 轻量行为层 (`channels/dock.ts`)

`ChannelDock` 是 `ChannelPlugin` 的子集投影，提供给**共享代码路径**（如回复流程、命令认证、Agent prompt）：

```
ChannelDock = {
  id, capabilities, commands, outbound.textChunkLimit,
  streaming, elevated, config.{resolveAllowFrom, formatAllowFrom},
  groups, mentions, threading, agentPrompt
}
```

设计目的：避免共享代码 `import` 整个 ChannelPlugin（会拉入 baileys/Discord Gateway/Telegram monitor 等重依赖）。
- 8 个内置通道: 硬编码在 `DOCKS` 对象中
- 扩展通道: 通过 `buildDockFromPlugin()` 从 ChannelPlugin 提取

#### 3.8 以 Telegram 为例的完整 Adapter 实现

```
extensions/telegram/src/channel.ts → 导出 telegramPlugin: ChannelPlugin

telegramPlugin = {
  id: "telegram",
  meta: { label:"Telegram", selectionLabel:"Telegram (Bot API)", ... },
  capabilities: { chatTypes:["direct","group","channel","thread"], reactions, threads, media, nativeCommands, blockStreaming },
  reload: { configPrefixes: ["channels.telegram"] },

  config: {                                    // 配置适配
    listAccountIds: listTelegramAccountIds,    // 从 cfg.channels.telegram.accounts 枚举
    resolveAccount: resolveTelegramAccount,    // 读取 token/allowFrom/proxy 等
    isConfigured: account => Boolean(account.token),
    ...
  },

  security: {                                  // 安全适配
    resolveDmPolicy: → { policy:"pairing", allowFrom:[...] }
    collectWarnings: → groupPolicy="open" 时发出警告
  },

  outbound: {                                  // 出站消息
    deliveryMode: "direct",                    // 直接调 Telegram API（不走 gateway 转发）
    textChunkLimit: 4000,                      // 单条消息上限
    chunkerMode: "markdown",                   // 按 Markdown 格式分块
    sendText: → sendMessageTelegram(to, text, { messageThreadId, replyToMessageId })
    sendMedia: → sendMessageTelegram(to, text, { mediaUrl, ... })
  },

  gateway: {                                   // 运行时生命周期
    startAccount: async (ctx) => {
      1. probeTelegram() → 探测 bot info (username)
      2. monitorTelegramProvider({             // 核心：启动消息监听
           token, accountId, config, runtime, abortSignal,
           useWebhook, webhookUrl, webhookSecret, ...
         })
    },
    logoutAccount: → 从配置删除 botToken → writeConfigFile
  },

  status: {                                    // 健康探测
    probeAccount: → probeTelegram(token, timeoutMs, proxy)
    auditAccount: → auditGroupMembership (检查 bot 是否在声明的群中)
    buildAccountSnapshot: → { configured, tokenSource, running, mode:"polling"|"webhook", probe, ... }
    collectStatusIssues: collectTelegramStatusIssues
  },

  directory: {                                 // 联系人目录
    listPeers: → listTelegramDirectoryPeersFromConfig
    listGroups: → listTelegramDirectoryGroupsFromConfig
  },

  setup: { ... },    // CLI 向导
  pairing: { ... },  // 配对审批
  actions: { ... },  // 消息动作
  ...
}
```

#### 3.9 设计要点

1. **多账号架构** — 一个 Channel 可运行多个账号（如 3 个 Telegram bot），每个账号独立启停
2. **能力声明驱动** — `capabilities` 让上层代码按能力分支（如有 `polls` 才提供投票工具），而非按通道名 if/else
3. **Dock vs Plugin 分层** — 轻量 Dock 给共享路径，完整 Plugin 只在启动/运行时加载，避免拉入 puppeteer 等重依赖
4. **AbortController 生命周期** — 每个 account 一个 AbortController，stopChannel 只需 abort() + 等待
5. **顺序启动** — `startChannels` 逐个 Channel 串行启动（避免并发抢占资源），但每个 Channel 的多账号并行启动
6. **配置热重载** — `reload.configPrefixes` 声明哪些配置路径变化时需要重启通道（如 `channels.telegram` 变化 → 重启 Telegram）

## WebSocket 连接生命周期

```
客户端发起连接
  → wss.on("connection") → 分配 connId(UUID), 启动握手超时定时器
  → 服务端发送 connect.challenge { nonce, ts }
  → 客户端发送 connect 请求 { type:"req", method:"connect", params:ConnectParams }
  → 校验流程：
      1. 协议版本协商 (minProtocol/maxProtocol)
      2. 角色校验 (operator / node)
      3. 浏览器 Origin 检查 (Control UI / Webchat)
      4. 设备身份验证 (publicKey → deriveId → 签名验证 → nonce)
      5. 认证 (token / password / tailscale / device-token)
      6. 速率限制检查
  → 成功: 创建 GatewayWsClient, 加入 clients Set
  → 失败: 返回错误 + close(1008, reason)
```

## 认证机制

4 种认证方式：

| 方式 | 说明 |
|------|------|
| token | 预共享令牌（最常用） |
| password | 密码认证 |
| tailscale | Tailscale 网络身份（whois 查询） |
| device-token | 设备配对令牌（移动客户端） |

额外机制：
- 速率限制：按 IP+scope 跟踪失败认证，触发退避
- 本地信任：loopback 连接可降低认证要求
- 代理感知：识别 X-Forwarded-For / X-Real-IP，需配置 trustedProxies

## 请求分发 & 权限模型

认证通过后，所有消息走统一 RPC：

```
RequestFrame  → handleGatewayRequest()
                  ├── authorizeGatewayMethod()  // scope 权限检查
                  └── handler(req, params, client, respond, context)
```

5 个权限 scope：
- `operator.admin` → 完全访问
- `operator.read` → 只读方法 (health, logs.tail, channels.status, ...)
- `operator.write` → 写方法 (send, chat.send, chat.abort, ...)
- `operator.approvals` → 审批相关
- `operator.pairing` → 设备/节点配对

24 个领域 handler：
connect / logs / voicewake / health / channels / chat / cron / devices /
exec-approvals / web / models / config / wizard / talk / tts / skills /
sessions / system / update / nodes / send / usage / agent / agents / browser

## 协议帧格式

WebSocket 上的 3 种 JSON 帧：

```typescript
RequestFrame  = { type: "req", id: string, method: string, params?: any }
ResponseFrame = { type: "res", id: string, ok: boolean, result?: any, error?: ErrorShape }
EventFrame    = { type: "event", event: string, payload?: any }
```

Schema 用 TypeBox 定义，AJV 运行时校验。

## 设计亮点

1. **RPC over WebSocket** — 不用 REST，在 WS 上实现完整 RPC（请求-响应 + 事件推送），支持双向实时通信
2. **分层权限** — 5 个 scope 粒度，细到每个 method
3. **多认证并行** — token + device + tailscale 同时支持，优雅降级
4. **Challenge-Response 握手** — 服务端先发 nonce，客户端需签名证明设备身份，防重放
5. **热重载** — config-reload.ts 监听配置文件变更，动态决定重启 vs 热更新
6. **插件扩展** — Channel 和 Plugin 可注册自己的 gateway method，统一进入 handler map

## Chat 聊天推理流程

> 核心文件: `server-methods/chat.ts` + `server-chat.ts` + `chat-abort.ts`
> 依赖: `auto-reply/dispatch.ts` → `auto-reply/reply/dispatch-from-config.ts` → Agent Runtime

### chat.ts 暴露的 4 个 RPC 方法

| 方法 | 功能 |
|------|------|
| `chat.history` | 拉取会话历史（带分页、字节上限、thinking/verbose 级别） |
| `chat.send` | **核心方法** — 发送消息并触发 Agent 推理 |
| `chat.abort` | 中止正在运行的推理（按 runId 或整个 session） |
| `chat.inject` | 注入 assistant 消息到 transcript（不触发推理） |

### `chat.send` 完整流程

```
客户端调用 chat.send { sessionKey, message, thinking?, attachments?, idempotencyKey }
    │
    ▼
1. 参数校验 (validateChatSendParams)
    │
    ▼
2. 停止命令检查 — 如果消息是 "/stop"，直接 abort 该 session 的所有 run
    │
    ▼
3. 幂等性去重 (dedupe)
   ├─ 如果 idempotencyKey 已有缓存结果 → 直接返回缓存
   └─ 如果 idempotencyKey 对应的 run 还在执行 → 返回 { status: "in_flight" }
    │
    ▼
4. 解析附件 (parseMessageWithAttachments)
   — 图片 base64 处理，5MB 上限
    │
    ▼
5. 加载 session 信息 (loadSessionEntry)
   — 获取 sessionId、会话配置、关联的 Agent 等
    │
    ▼
6. 发送策略检查 (resolveSendPolicy)
   — 某些 session 可能禁止发送 → 返回 "deny"
    │
    ▼
7. 创建 AbortController + 注册到 chatAbortControllers
   — 用于后续 chat.abort 取消
    │
    ▼
8. **立即 respond → ACK** { runId, status:"started" }
   （不等推理完成，先告诉客户端"已接受"）
    │
    ▼
9. 构造 MsgContext（消息上下文）
   ├─ Body: 原始消息（给 UI 显示）
   ├─ BodyForAgent: 注入了时间戳的消息（给 Agent 看）
   ├─ BodyForCommands: 如果有 thinking 参数 → "/think {level} {msg}"
   ├─ SessionKey, Provider: "internal"
   └─ SenderId/SenderName: 来自客户端 connect 信息
    │
    ▼
10. 创建 ReplyDispatcher（回复分发器）
    — deliver 回调: 收集 final reply 文本
    │
    ▼
11. **异步调用 dispatchInboundMessage()** ← 这是关键！
    │  └→ finalizeInboundContext() — 规范化消息上下文
    │  └→ dispatchReplyFromConfig() — 根据配置调度到 Agent
    │       └→ ... → Agent Runtime (Pi) → LLM API
    │
    ├─ 成功路径 (.then):
    │   ├─ 如果 agentRunStarted=false（Agent 没启动，可能是命令/模板回复）
    │   │   → 将 finalReplyParts 写入 transcript
    │   │   → broadcastChatFinal() 通知客户端
    │   └─ 记录到 dedupe 缓存
    │
    ├─ 失败路径 (.catch):
    │   → broadcastChatError() 通知客户端
    │   → 记录到 dedupe 缓存
    │
    └─ 清理 (.finally):
        → 从 chatAbortControllers 中删除
```

### 流式输出机制 (server-chat.ts)

Agent 运行时通过事件系统发出流式数据，由 `createAgentEventHandler` 处理：

```
Agent Runtime 发出 AgentEventPayload
    │
    ▼
事件处理器 (onAgentEvent 回调)
    │
    ├── evt.stream === "assistant" && 有 text
    │   → emitChatDelta()
    │     ├─ 更新 buffer (chatRunState.buffers)
    │     ├─ 节流: 每 150ms 最多发一次 delta
    │     └─ broadcast("chat", { state:"delta", message:{...} })
    │        → 所有 WebSocket 客户端收到实时打字流
    │
    ├── evt.stream === "tool"
    │   → 只发给注册了 TOOL_EVENTS 能力的特定 connId
    │   → verbose 级别控制是否包含 result/partialResult
    │
    └── evt.stream === "lifecycle" && phase === "end" | "error"
        → emitChatFinal()
          ├─ broadcast("chat", { state:"final", message:{...} })
          └─ 清理 buffer、deltaSentAt
```

### 状态管理 — 多个 Map 协同

```
chatAbortControllers: Map<runId, { controller, sessionId, sessionKey, expiresAtMs }>
  → 跟踪活跃的推理 run，支持 abort

chatRunState.registry: Map<sessionId, ChatRunEntry[]>
  → 将 Agent 内部 runId 映射到客户端 clientRunId + sessionKey

chatRunState.buffers: Map<clientRunId, string>
  → 累积 assistant 流式输出文本

chatRunState.deltaSentAt: Map<clientRunId, number>
  → delta 节流: 记录上次发送时间，150ms 内不重复发

dedupe: Map<"chat:{idempotencyKey}", { ts, ok, payload, error }>
  → 幂等性缓存: 防止重复提交

agentRunSeq: Map<runId, number>
  → 事件序列号，检测 seq gap
```

### chat.abort 机制

```
chat.abort { sessionKey, runId? }
  │
  ├─ 有 runId → abortChatRunById()
  │   → controller.abort() → AbortSignal 传播到 Agent
  │   → broadcastChatAborted()
  │
  └─ 无 runId → abortChatRunsForSessionKey()
      → 遍历所有 chatAbortControllers
      → 找到 sessionKey 匹配的全部 abort
```

另外，用户发 "/stop" 消息也会触发 abort（chat.send 开头检查 `isChatStopCommandText`）。

### chat.inject — 静默注入

不触发 Agent，直接往 transcript 写入一条 assistant 消息：
1. 使用 `SessionManager.open()` + `appendMessage()` 保证 parentId 链正确
2. broadcast("chat", { state:"final" }) 通知 UI 刷新

用途: 系统通知、heartbeat 回复、外部工具结果注入等。

### 设计亮点

1. **Fire-and-forget ACK** — chat.send 立即返回 ACK，推理异步执行。客户端通过 EventFrame 接收流式更新。
2. **幂等性保证** — 用 idempotencyKey 防重复提交，支持客户端重连后恢复。
3. **150ms 节流** — delta 事件不是每个 token 都发，而是攒到 150ms 发一次，减少 WS 消息量。
4. **Tool 事件精准投递** — tool 事件只发给声明了 TOOL_EVENTS 能力的特定连接，不广播给所有人。
5. **Heartbeat 抑制** — 心跳 run 的广播根据 visibility 配置决定是否发给 webchat。

## 疑问 & 待深入

- [x] chat.ts 的聊天推理流程（Agent 调用、流式输出、abort）
- [ ] dispatchReplyFromConfig 到 Agent Runtime 的完整调用链
- [x] Channel 插件枚举 + ChannelPlugin 契约 + ChannelManager 启停生命周期
- [ ] Channel inbound 消息流：monitor 收到消息后如何路由到 dispatchInboundMessage → Agent？
- [ ] config-reload.ts 的热重载 vs 重启决策逻辑
- [ ] Node（远程节点）的 pairing 和 command 执行流程
- [ ] 广播系统的 backpressure（dropIfSlow）机制
- [ ] SessionManager 的 parentId 链和 compaction 机制
