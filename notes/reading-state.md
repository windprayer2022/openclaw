# 阅读状态

> 此文件由 Claude 自动维护，每次讲解后更新。用于跨会话恢复阅读上下文。

## 当前位置

- **最后阅读模块**: Channel 插件系统 — Gateway 启动时的 Channel 枚举、ChannelPlugin 契约、ChannelManager 启停生命周期
- **最后阅读文件**: `src/gateway/server-channels.ts`, `src/channels/plugins/index.ts`, `src/channels/plugins/types.plugin.ts`, `src/channels/plugins/types.core.ts`, `src/channels/plugins/types.adapters.ts`, `src/channels/dock.ts`, `src/channels/registry.ts`, `src/channels/plugins/catalog.ts`, `src/channels/plugins/load.ts`, `extensions/telegram/src/channel.ts`, `extensions/whatsapp/src/channel.ts`
- **更新时间**: 2026-02-24

## 已建立的理解

1. **Gateway 整体架构（6 层）** — 入口/初始化 → WS 连接管理 → 消息处理/认证 → 请求分发/权限 → 业务处理器 → 协议定义/校验
2. **启动流程** — `startGatewayServer()` 组装配置、插件、通道、WS Server 等子系统
3. **WebSocket 连接生命周期** — challenge-response 握手、4 种认证方式、scope 权限模型
4. **RPC 分发** — 24 个领域 handler，统一的 `handleGatewayRequest()` 路由
5. **chat.send 完整流程** — 异步 fire-and-forget ACK，dispatchInboundMessage 触发 Agent，流式 delta 事件 150ms 节流
6. **chat.abort / chat.inject** — 中止推理 + 静默注入消息
7. **配置文件读取流程** — 7 阶段管线：路径定位 → JSON5 解析 → $include 展开 → ${ENV} 替换 → Legacy 迁移 → Zod 校验 → 默认值 + 插件自动启用
8. **插件加载流程** — 5 阶段管线：配置归一化 → 4 来源发现 → Manifest 注册 → 逐插件处理（启用决策 + jiti 加载 + register(api)）→ 收尾（缓存 + Hook Runner）
9. **Channel 插件系统** — ChannelPlugin 20 个 Adapter 插槽契约、8 个内置通道 + 10+ 扩展通道、ChannelManager 启停生命周期（多账号、AbortController）、Dock 轻量行为层分离重依赖

## 已完成的笔记

- `notes/gateway.md` — Gateway 核心流程 + Chat 聊天推理流程
- `notes/architecture-decisions.md` — 跨模块架构决策（AD-1: TypeBox+AJV vs Zod 双 Schema 体系）
- `notes/config-loading.md` — 配置文件读取流程（7 阶段管线）
- `notes/plugins.md` — 插件系统（5 阶段加载管线、4 来源发现、Plugin API、15 种钩子）

## 下一步建议

1. **往下游 — Agent Runtime** (`src/agents/`): `dispatchReplyFromConfig` → Agent Runtime (Pi) → LLM API 的完整调用链。理解消息如何真正到达大模型。（优先级 ★★★）
2. **往上游 — Channel 插件** (`src/channels/` + `extensions/`): 外部消息（Telegram/WhatsApp）怎么进入同样的 dispatchInboundMessage 流程。（优先级 ★★☆）
3. **横向 — Session 管理**: transcript 持久化、SessionManager 的 parentId 链、compaction 机制。（优先级 ★★☆）

## 待解答的问题

- [ ] dispatchReplyFromConfig 到 Agent Runtime 的完整调用链
- [x] Channel 插件枚举、ChannelPlugin 契约（20 个 Adapter）、ChannelManager 启停生命周期
- [ ] Channel 插件如何将外部消息路由到 Agent？（inbound 消息流：monitor → dispatchInboundMessage）
- [x] 配置文件读取流程（路径解析、JSON5、$include、${ENV}、legacy 迁移、校验、默认值）
- [x] 插件加载流程（发现、Manifest、启用决策、jiti 加载、注册）
- [ ] config-reload.ts 的热重载 vs 重启决策逻辑
- [ ] Node（远程节点）的 pairing 和 command 执行流程
- [ ] 广播系统的 backpressure（dropIfSlow）机制
- [ ] SessionManager 的 parentId 链和 compaction 机制
- [ ] Hook Runner 的调度逻辑（并发/串行、错误处理）
- [ ] 插件运行时 createPluginRuntime() 的内部结构
