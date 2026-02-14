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
3. 枚举 Channel 插件（WhatsApp/Telegram/Slack/...）
4. 解析运行时配置（绑定地址、认证模式、TLS）
5. 创建运行时状态（WS Server, HTTP Server, 广播系统）
6. 启动子系统：
   ├─ NodeRegistry（远程节点注册）
   ├─ ChannelManager（消息通道管理）
   ├─ Discovery（mDNS/Bonjour 服务发现）
   ├─ CronService（定时任务）
   ├─ MaintenanceTimers（健康检查、去重清理）
   ├─ AgentEventHandler（Agent 事件监听 → 广播到客户端）
   ├─ HeartbeatRunner（心跳）
   └─ ExecApprovalManager（执行审批）
7. 挂载 WebSocket 处理器
```

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
- [ ] Channel 插件如何将外部消息路由到 Agent？
- [ ] config-reload.ts 的热重载 vs 重启决策逻辑
- [ ] Node（远程节点）的 pairing 和 command 执行流程
- [ ] 广播系统的 backpressure（dropIfSlow）机制
- [ ] SessionManager 的 parentId 链和 compaction 机制
