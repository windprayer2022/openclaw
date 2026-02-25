# Channel 系统

> 消息平台桥梁层。将 16+ 个外部平台（Telegram、WhatsApp、Discord、Slack……）统一为 Agent 可消费的标准消息接口。

## 一、两层架构：Dock（轻）vs Plugin（重）

| | Dock（轻量层） | Plugin（重量层） |
|---|---|---|
| 导入成本 | 几乎为零 | 可能拉入 WebSocket 客户端、Puppeteer、HTTP 服务器等 |
| 包含内容 | meta、capabilities、chunker 参数、mention 剥离规则 | 完整 20 个 Adapter 插槽 + monitor 监听器 + 登录流程 |
| 使用场景 | 共享代码中查询能力 | Gateway 启动时真正运行通道 |
| 入口文件 | `src/channels/dock.ts` | `src/channels/plugins/index.ts` |

**设计理由**：很多场景只需元信息（如 "Telegram 最大 4000 字符"），不需要启动 Bot。分层避免 `import` 连带拉入重依赖。

关键函数：
- `getChannelDock(id)` — 廉价，只拿轻量信息
- `listChannelDocks()` — 所有通道的轻量视图
- `getChannelPlugin(id)` — 完整插件（含 monitor、登录等）
- `listChannelPlugins()` — 所有完整插件（昂贵）

## 二、ChannelPlugin 契约（20 个 Adapter 插槽）

`src/channels/plugins/types.plugin.ts` 定义了 `ChannelPlugin` 类型。

### 必填（2 个）

| Adapter | 职责 | 关键方法 |
|---------|------|----------|
| **`meta`** | 身份信息卡 | `id`, `label`, `selectionLabel`, `docsPath`, `blurb`, `systemImage`, `order` |
| **`config`** | 配置管理 | `listAccountIds()`, `resolveAccount()`, `isConfigured()`, `isEnabled()`, `defaultAccountId()` |

### 生命周期（3 个）

| Adapter | 职责 | 关键方法 |
|---------|------|----------|
| **`gateway`** | 启停监听 | `startAccount(ctx)`, `stopAccount(ctx)`, `loginWithQrStart()`, `loginWithQrWait()`, `logoutAccount()` |
| **`setup`** | 初始配置引导 | 账号 ID 解析、名称应用、输入验证 |
| **`onboarding`** | CLI 安装向导钩子 | — |

### 消息收发（4 个）

| Adapter | 职责 | 关键方法 |
|---------|------|----------|
| **`outbound`** | 出站发送 | `sendText()`, `sendMedia()`, `sendPayload()`, `sendPoll()`, `chunker`, `resolveTarget()`, `deliveryMode` |
| **`messaging`** | 目标格式化 | `normalizeTarget()`, `formatTargetDisplay()`, `targetResolver` |
| **`streaming`** | 流式输出参数 | `blockStreamingCoalesceDefaults: { minChars, idleMs }` |
| **`threading`** | 回复线程模式 | mode: `"off"` / `"first"` / `"all"` |

### 安全与访问控制（5 个）

| Adapter | 职责 |
|---------|------|
| **`security`** | DM 策略（open/allowlist/pairing/disabled）、安全警告收集 |
| **`pairing`** | 配对/白名单管理、审批通知 |
| **`auth`** | 交互式登录（CLI 环境） |
| **`elevated`** | 提权命令（仅 owner 可用） |
| **`commands`** | 命令级授权规则 |

### 平台特性（5 个）

| Adapter | 职责 |
|---------|------|
| **`groups`** | 群组行为：是否要求 @mention、工具策略、intro 提示 |
| **`mentions`** | 剥离平台特有的 mention 格式（`@user`、`<@id>` 等） |
| **`actions`** | 消息操作：列出可用操作、执行操作（桥接 Agent 工具） |
| **`directory`** | 目录查询：联系人、群组、成员 |
| **`resolver`** | 名称 → ID 解析 |

### 运维（3 个）

| Adapter | 职责 |
|---------|------|
| **`status`** | 健康探针（probe）、审计、快照构建 |
| **`heartbeat`** | 心跳检查 + 收件人解析 |
| **`agentPrompt`** | 为 LLM 注入通道相关的工具提示 |

### 附加字段

| 字段 | 职责 |
|------|------|
| **`capabilities`** | 功能开关声明（见下文§三） |
| **`agentTools`** | 通道自带的 Agent 工具（如 WhatsApp 登录工具） |
| **`configSchema`** | 配置 JSON Schema + UI 提示 |
| **`reload`** | 热重载前缀匹配规则（`configPrefixes`, `noopPrefixes`） |

## 三、Meta 与 Capabilities

### Meta — "这个通道是谁"

纯描述性，用于 UI 展示和文档链接。定义在 `src/channels/plugins/types.core.ts`。

```ts
type ChannelMeta = {
  id: ChannelId;
  label: string;              // "Telegram"
  selectionLabel: string;     // "Telegram (Bot API)"
  detailLabel?: string;       // "Telegram Bot"
  docsPath: string;           // "/channels/telegram"
  blurb: string;              // 一句话简介
  systemImage?: string;       // SF Symbols 图标名，如 "paperplane"
  order?: number;             // 排序权重
  aliases?: string[];         // 别名（如 imessage → "imsg"）
  // ...其余 UI 微调选项
};
```

静态注册表在 `src/channels/registry.ts` 的 `CHAT_CHANNEL_META` 对象中，包含 8 个内置通道的 Meta。

### Capabilities — "这个通道能做什么"

功能开关声明，系统据此决定运行时行为。

```ts
type ChannelCapabilities = {
  chatTypes: Array<"direct" | "group" | "channel" | "thread">;  // 必填
  reactions?: boolean;       // 表情反应
  edit?: boolean;            // 编辑已发消息
  unsend?: boolean;          // 撤回消息
  reply?: boolean;           // 引用回复
  threads?: boolean;         // 话题/线程
  media?: boolean;           // 图片/音频/视频
  polls?: boolean;           // 投票
  effects?: boolean;         // 消息特效
  nativeCommands?: boolean;  // 平台原生命令（如 /start）
  blockStreaming?: boolean;   // 禁用流式，一次性发完整回复
  groupManagement?: boolean; // 群组管理
};
```

运行时影响举例：
- `media: false` → 跳过图片/音频处理管道
- `blockStreaming: true` → 不走流式输出，攒完整条回复再发
- `chatTypes` 不含 `"group"` → 不处理群消息

### Telegram 实例

```ts
// meta (registry.ts:29-41)
telegram: {
  id: "telegram",
  label: "Telegram",
  selectionLabel: "Telegram (Bot API)",
  detailLabel: "Telegram Bot",
  docsPath: "/channels/telegram",
  blurb: "simplest way to get started — register a bot with @BotFather and get going.",
  systemImage: "paperplane",
}

// capabilities (extensions/telegram/src/channel.ts:94-101)
capabilities: {
  chatTypes: ["direct", "group", "channel", "thread"],
  reactions: true,
  threads: true,
  media: true,
  nativeCommands: true,
  blockStreaming: true,
}
```

## 四、入站消息流（外部 → Agent）

以 Telegram 为例，一条消息从用户手机到 Agent 的完整路径：

```
用户发送 "你好" 到 Telegram Bot
        ↓
┌─ Phase 1: 平台 API 接收 ──────────────────────────┐
│  monitorTelegramProvider()                         │
│  src/telegram/monitor.ts                           │
│  - 长轮询 getUpdates 或 Webhook 接收               │
│  - grammY 框架解析 Update 对象                     │
└────────────────────────────────────────────────────┘
        ↓
┌─ Phase 2: Bot Handler 预处理 ─────────────────────┐
│  bot.on("message", ctx => ...)                     │
│  src/telegram/bot-handlers.ts                      │
│  - 跳过无效更新 (shouldSkipUpdate)                 │
│  - 检查群组/话题是否启用                            │
│  - 长文本拼接（Telegram 拆分 ≥4000 字符的粘贴）     │
│  - 媒体组合并（多图消息，200ms 超时）               │
│  - 下载媒体附件                                     │
└────────────────────────────────────────────────────┘
        ↓
┌─ Phase 3: 防抖队列 ──────────────────────────────┐
│  inboundDebouncer.enqueue()                       │
│  - key: channel:account:conversation:sender       │
│  - 快速连续消息合并，可配置延迟后 flush            │
└────────────────────────────────────────────────────┘
        ↓
┌─ Phase 4: 消息上下文构建 ────────────────────────┐
│  buildTelegramMessageContext()                    │
│  src/telegram/bot-message-context.ts              │
│  ├─ recordChannelActivity() — 记录通道活动        │
│  ├─ resolveAgentRoute() — 路由到哪个 Agent        │
│  ├─ DM 策略执行 — open/allowlist/pairing/disabled │
│  ├─ 群组过滤 — 群级 allowFrom 覆盖               │
│  ├─ @mention 检测 — 群聊中是否提到了 Bot          │
│  ├─ 文本组装 — 链接展开、位置信息拼接             │
│  ├─ ACK 反应 — 是否发确认 emoji                   │
│  └─ recordInboundSession() — 会话记录             │
│                                                    │
│  输出: MsgContext {                                │
│    Body, From, To, SessionKey, AccountId,          │
│    ChatType, Provider, MediaPath, WasMentioned,    │
│    InboundHistory, ReplyToId, ...                  │
│  }                                                 │
└────────────────────────────────────────────────────┘
        ↓
┌─ Phase 5: 分发到 Agent ──────────────────────────┐
│  dispatchReplyWithBufferedBlockDispatcher()        │
│  src/telegram/bot-message-dispatch.ts             │
│  - 创建 ReplyDispatcher（序列化回复队列）          │
│  - 设置流式输出（Telegram draft 编辑）             │
│  - → dispatchReplyFromConfig()                     │
│    → Agent Runtime → LLM 推理 → 工具执行          │
└────────────────────────────────────────────────────┘
        ↓
      （进入出站流程）
```

### 各通道调用入口差异

| 通道 | 入口函数 | 调用方式 |
|------|---------|---------|
| Telegram | `dispatchReplyWithBufferedBlockDispatcher()` | 专用包装 |
| Discord | `dispatchInboundMessage()` | 通用入口 |
| Slack | `dispatchInboundMessage()` | 通用入口 |
| Gateway WebChat | `dispatchInboundMessage()` | 通用入口 |

`dispatchInboundMessage()` 定义在 `src/auto-reply/dispatch.ts`，内部调用 `dispatchReplyFromConfig()`。

### 7 层安全过滤

消息在到达 Agent 之前可能被拦截：

1. **群组 disabled** — 配置中关闭了该群
2. **话题 disabled** — 配置中关闭了该话题
3. **DM 策略** — `disabled`（拒绝）/ `pairing`（配对码）/ `allowlist`（白名单）/ `open`（放行）
4. **allowFrom 匹配** — 发送者不在白名单中
5. **群组策略** — `open` / `allowlist` / `disabled`
6. **@mention 门控** — 群聊中未 @Bot 则忽略（除非 `requireMention: false`）
7. **命令授权** — 控制命令需额外权限检查

## 五、出站消息流（Agent → 外部）

```
Agent 输出: ReplyPayload[] (text, media, buttons...)
        ↓
┌─ Phase 1: ReplyDispatcher 序列化 ────────────────┐
│  src/auto-reply/reply/reply-dispatcher.ts         │
│  sendToolResult() / sendBlockReply() / sendFinal() │
│  - 严格保序: tool → block → final                  │
│  - block 之间插入仿人延迟（humanDelay）            │
│  - pending 计数器防止 Gateway 过早重启             │
└────────────────────────────────────────────────────┘
        ↓
┌─ Phase 2: deliverOutboundPayloads() ─────────────┐
│  src/infra/outbound/deliver.ts                    │
│  ① 写前队列（Write-ahead queue）                   │
│     - 持久化到磁盘，崩溃后可恢复                   │
│                                                    │
│  ② 加载通道适配器                                  │
│     loadChannelOutboundAdapter("telegram")         │
│     src/channels/plugins/outbound/load.ts          │
│                                                    │
│  ③ 执行 message_sending 钩子                      │
│     - 插件可取消或修改内容                         │
│                                                    │
│  ④ 按类型发送:                                     │
│     ├─ 纯文本 → chunker 分块 → sendText()         │
│     ├─ 媒体   → sendMedia()                       │
│     └─ 富内容 → sendPayload() (按钮、引用等)      │
│                                                    │
│  ⑤ 执行 message_sent 钩子                         │
│                                                    │
│  ⑥ 成功 → ackDelivery() 从队列移除               │
│     失败 → failDelivery() 递增重试计数            │
│                                                    │
│  ⑦ 可选：镜像到会话 transcript                    │
└────────────────────────────────────────────────────┘
        ↓
┌─ Phase 3: 平台发送 ─────────────────────────────┐
│  两种投递模式:                                    │
│                                                    │
│  "direct" — 直接 HTTP 调用平台 API                │
│  (Telegram、Discord、Slack、Signal)               │
│                                                    │
│  "gateway" — 通过 Gateway WebSocket 转发          │
│  (WhatsApp、Matrix 等需要持久连接的平台)           │
└────────────────────────────────────────────────────┘
        ↓
用户手机收到回复
```

### 出站适配器实现文件

| 通道 | 文件 | 分块限制 | 投递模式 |
|------|------|---------|---------|
| Telegram | `src/channels/plugins/outbound/telegram.ts` | 4000 (HTML) | direct |
| WhatsApp | `src/channels/plugins/outbound/whatsapp.ts` | 4000 | gateway |
| Discord | `src/channels/plugins/outbound/discord.ts` | 2000 | direct |
| Slack | `src/channels/plugins/outbound/slack.ts` | 4000 | direct |

### 崩溃恢复机制

写前队列 `src/infra/outbound/delivery-queue.ts` 确保消息不因进程崩溃丢失：

```
入队 → enqueueDelivery()（持久化到磁盘）
成功 → ackDelivery()（移除）
失败 → failDelivery()（递增 retryCount）

Gateway 重启 → recoverPendingDeliveries()（按入队时间排序恢复）

指数退避重试:
  第 1 次:  5 秒
  第 2 次: 25 秒
  第 3 次:  2 分钟
  第 4 次: 10 分钟
  第 5 次: 超过 MAX_RETRIES=5，移入 failed/ 目录
```

## 六、Channel 生命周期（ChannelManager）

`src/gateway/server-channels.ts` 中的 `createChannelManager()` 管理通道启停。

```
Gateway 启动
    ↓
ChannelManager.startChannels()
    ↓
遍历所有已注册通道的每个账号:
  ├─ plugin.config.isEnabled(account)    → 未启用则跳过
  ├─ plugin.config.isConfigured(account) → 未配置则跳过
  ├─ 创建 AbortController
  ├─ plugin.gateway.startAccount(ctx)    → 启动 Monitor
  │   ctx: { cfg, accountId, account, abortSignal, getStatus(), setStatus() }
  └─ 记录运行时快照
    ↓
Monitor 持续运行（轮询 / WebSocket / SSE）
    ↓
停止时:
  ├─ AbortController.abort()             → Monitor 收到信号
  ├─ plugin.gateway.stopAccount(ctx)     → 清理资源
  └─ 更新快照: running=false, lastStopAt
```

### 运行时快照字段

```ts
{
  running, connected, enabled, configured,
  lastStartAt, lastStopAt, lastConnectedAt,
  lastError, lastEventAt, lastMessageAt,
  mode, dmPolicy, allowFrom, /* 凭据来源 */
}
```

### 多账号支持

一个通道可有多个账号（如两个 Discord Bot），每个账号独立管理：独立凭据、独立 Monitor 进程、独立 AbortController、独立运行时快照。

## 七、通道注册表

`src/channels/registry.ts` 管理静态元数据。

### 8 个内置通道

```
telegram, whatsapp, discord, irc, googlechat, slack, signal, imessage
```

别名映射：`imsg → imessage`, `gchat → googlechat`, `google-chat → googlechat`, `internet-relay-chat → irc`

### 外部通道

通过插件加载器从 `extensions/` 目录或 npm 包发现注册，10+ 个扩展通道。

### 各通道能力对比

| 通道 | 投递模式 | 分块 | 聊天类型 | 特色 |
|------|---------|------|---------|------|
| Telegram | direct | 4000 (HTML) | DM/群/频道/线程 | 原生命令、反应、block streaming |
| WhatsApp | gateway | 4000 | DM/群 | QR 登录、GIF、投票 |
| Discord | direct | 2000 | DM/群/线程 | 静默发送、投票（10 选项） |
| Slack | direct | 4000 | DM/频道/线程 | Socket Mode、thread_ts |
| Signal | direct | — | DM/群 | signal-cli 链接设备 |
| IRC | direct | — | DM/频道 | 经典 IRC 路由 |
| iMessage | direct | — | DM | 实验中 |
| Google Chat | direct | — | DM/群 | HTTP Webhook |

## 八、Gateway 与 Channel 的关系

### 总体关系

Gateway 是中枢调度器，Channel 是外部触角。两者在同一个 Node.js 进程内，通过直接函数调用交互，不走 WebSocket 也不走 RPC。

```
                    同一个 Node.js 进程
  ┌──────────────────────────────────────────────────┐
  │                                                  │
  │  Gateway（大脑）                                  │
  │  ├─ ChannelManager — 管理所有 Channel 的启停      │
  │  ├─ RPC 处理器 — chat.send / send 等             │
  │  ├─ Agent 调度 — dispatchReplyFromConfig          │
  │  └─ WebSocket Server — 服务 Web UI / 移动端      │
  │                                                  │
  │  Channel（手脚）                                  │
  │  ├─ Monitor — 独立连接平台 API，收消息            │
  │  └─ Outbound — 被 Gateway 调用，发消息            │
  │                                                  │
  └──────────────────────────────────────────────────┘
         ↕                                    ↕
    平台 API                           WebSocket 客户端
  (Telegram/Discord)                  (Web UI/移动端)
```

### 三个核心关系

**1. Gateway 管 Channel 的生死**

Gateway 启动时通过 ChannelManager 逐个启动 Channel（`plugin.gateway.startAccount()`），关闭时逐个停止（`AbortController.abort()`）。Channel 不能自己启动自己。

**2. 入站方向：Channel 主动，Gateway 被动**

Channel 的 Monitor 独立连着平台（轮询/WebSocket），收到消息后主动调用 `dispatchInboundMessage()` 把消息推给 Gateway 侧的 Agent 调度逻辑。Gateway 不知道消息什么时候来，只是等着被调用。

```
Channel Monitor ──函数调用──→ dispatchInboundMessage() → Agent
```

**3. 出站方向：Gateway 主动，Channel 被动**

Agent 产生回复后，Gateway 侧的 `deliverOutboundPayloads()` 主动调用 Channel 的 `outbound.sendText()`。Channel 的出站适配器只是一个被调用的函数，不主动运行。

```
Agent → ReplyDispatcher → deliverOutboundPayloads() ──函数调用──→ Channel Outbound
```

### Bot 与 Channel 的关系

Bot（如 grammY Bot、discord.js Client）是 Channel 的内部实现细节——即 Channel 用来跟平台 API 通信的客户端对象。外部（Gateway、Agent）永远不直接接触 Bot。

| Channel | Bot 对应物 | SDK |
|---------|-----------|-----|
| Telegram | grammY `Bot` 实例 | grammy |
| Discord | discord.js `Client` 实例 | discord.js |
| Slack | Socket Mode `App` 实例 | @slack/bolt |
| WhatsApp | Baileys `WASocket` 实例 | @whiskeysockets/baileys |
| Signal | 无 SDK，直接 HTTP 调 signal-cli REST | fetch |

```
Channel 插件
  ├─ gateway.startAccount()
  │     └─ createBot({ token })        ← Bot 在这里被创建
  │           ├─ bot.on("message")     ← 用 Bot 收消息
  │           └─ run(bot)              ← 用 Bot 跑监听
  └─ outbound.sendText()
        └─ bot.api.sendMessage()       ← 用 Bot 发消息
```

## 九、编写新 Channel 插件

### 三类工作量

编写新 Channel 时，各部分的工作量分三类：

**从零编写（平台特有）：**

| 部分 | 说明 |
|------|------|
| Monitor 监听器 | 用平台 SDK 接收消息，构建 MsgContext。工作量最大 |
| 发送函数 | sendText / sendMedia / sendPayload，调用平台 API |
| 账号解析 | resolveAccount — 凭据结构各平台不同（token / appId+secret / QR 等） |
| 消息上下文构建 | 如 `buildTelegramMessageContext()`，约 60% 是平台特有逻辑（ID 提取、mention 检测、文本组装、媒体解析等），可参照已有实现但无法直接复用 |

**需要适配但有模式可循（换参数）：**

| 部分 | 说明 |
|------|------|
| capabilities | 如实声明平台能力 |
| security | 换配置路径和 normalizeEntry |
| mentions | 换 stripPatterns 正则 |
| messaging | 换 normalizeTarget 和 hint |
| threading | 换默认 replyToMode |
| outbound.chunker | 换分块限制和格式转换 |

**可直接复用的共享工具函数：**

```
setAccountEnabledInConfigSection()
deleteAccountFromConfigSection()
migrateBaseNameToDefaultAccount()
applyAccountNameToChannelSection()
buildChannelConfigSchema()
formatPairingApproveHint()
normalizeAccountId()
getChatChannelMeta()
dispatchInboundMessage()
deliverOutboundPayloads()
recordInboundSession()
recordChannelActivity()
resolveAgentRoute()
```

### 最小示例

```ts
const myPlugin: ChannelPlugin = {
  id: "mychannel",
  meta: { id: "mychannel", label: "My Channel", docsPath: "/channels/my", blurb: "..." },
  capabilities: { chatTypes: ["direct", "group"], media: true },

  // 必填：配置管理
  config: {
    listAccountIds: (cfg) => ["default"],
    resolveAccount: (cfg, id) => ({ token: cfg.channels.mychannel.token }),
    isConfigured: (account) => Boolean(account.token),
    isEnabled: (account) => true,
  },

  // 入站：启动监听器
  gateway: {
    startAccount: async (ctx) => {
      // 启动 WebSocket/轮询，收到消息后构建 MsgContext
      // 调用 dispatchInboundMessage() 路由到 Agent
    },
    stopAccount: async (ctx) => { /* 清理 */ },
  },

  // 出站：发送消息
  outbound: {
    deliveryMode: "direct",
    textChunkLimit: 2000,
    sendText: async ({ to, text }) => {
      // 调用平台 API 发送
      return { channel: "mychannel", messageId: "..." };
    },
    sendMedia: async ({ to, text, mediaUrl }) => {
      return { channel: "mychannel", messageId: "..." };
    },
  },
};
```

只需 `config` + `gateway` + `outbound` 三个核心 Adapter 即可接入新平台，其余 17 个可选。

## 十、关键文件索引

| 文件 | 职责 |
|------|------|
| `src/channels/plugins/types.plugin.ts` | ChannelPlugin 契约定义（20 个 Adapter 插槽） |
| `src/channels/plugins/types.adapters.ts` | 14 个 Adapter 接口定义 |
| `src/channels/plugins/types.core.ts` | 核心类型：Capabilities、Meta、Messaging/Streaming/Threading |
| `src/channels/registry.ts` | 静态注册表：8 个内置通道的 Meta + 别名映射 |
| `src/channels/dock.ts` | 轻量 Dock 层：不导入重依赖的通道行为视图 |
| `src/channels/plugins/catalog.ts` | 插件目录（可安装插件列表） |
| `src/channels/plugins/load.ts` | 插件加载：发现 + jiti 加载 + 缓存 |
| `src/channels/plugins/outbound/load.ts` | 出站适配器按需加载与缓存 |
| `src/channels/plugins/outbound/{telegram,whatsapp,discord,slack}.ts` | 各通道出站实现 |
| `src/gateway/server-channels.ts` | ChannelManager：通道启停生命周期 |
| `src/auto-reply/dispatch.ts` | `dispatchInboundMessage()` 入站消息分发入口 |
| `src/auto-reply/reply/reply-dispatcher.ts` | ReplyDispatcher：出站回复序列化 |
| `src/infra/outbound/deliver.ts` | `deliverOutboundPayloads()` 出站投递核心 |
| `src/infra/outbound/delivery-queue.ts` | 写前队列 + 崩溃恢复 + 指数退避 |
| `src/infra/outbound/message.ts` | 高层发送 API（direct vs gateway 模式） |
| `src/infra/outbound/targets.ts` | 出站目标解析与验证 |
| `src/telegram/monitor.ts` | Telegram Monitor（轮询/Webhook） |
| `src/telegram/bot-handlers.ts` | Telegram Bot Handler 注册 |
| `src/telegram/bot-message-context.ts` | Telegram 消息上下文构建 |
| `src/telegram/bot-message-dispatch.ts` | Telegram 消息分发到 Agent |
