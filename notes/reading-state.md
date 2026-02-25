# 阅读状态

> 此文件由 Claude 自动维护，每次讲解后更新。用于跨会话恢复阅读上下文。

## 当前位置

- **最后阅读模块**: Sandbox 系统 — Docker 容器隔离、运行时判定(mode/scope)、容器生命周期(configHash 变更检测)、FS Bridge(docker exec)、Tool Policy(allow/deny 过滤)、沙箱化工具替换、Browser Bridge(CDP 容器化)、Registry 持久化、Prune 自动清理、Media Staging、CLI 管理命令
- **最后阅读文件**: `src/agents/sandbox/types.ts`, `types.docker.ts`, `constants.ts`, `config.ts`, `context.ts`, `shared.ts`, `workspace.ts`, `docker.ts`, `fs-bridge.ts`, `runtime-status.ts`, `registry.ts`, `tool-policy.ts`, `browser.ts`, `browser-bridges.ts`, `prune.ts`, `manage.ts`, `config-hash.ts`, `sandbox-paths.ts`, `auto-reply/reply/stage-sandbox-media.ts`, `commands/sandbox.ts`, `agents/pi-tools.ts`(sandbox 集成部分)
- **更新时间**: 2026-02-25

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
10. **Channel 入站消息流** — monitor 接收 → bot handler 预处理 → 防抖队列 → 消息上下文构建（路由、安全、mention、ACK）→ dispatchInboundMessage → Agent；7 层安全过滤
11. **Channel 出站消息流** — ReplyDispatcher 序列化（tool→block→final 保序 + 仿人延迟）→ deliverOutboundPayloads（写前队列 + chunker 分块 + 钩子）→ 通道适配器 sendText/sendMedia → 平台 API；崩溃恢复（指数退避 5s→25s→2m→10m）
12. **Agent Runtime 完整调用链** — dispatchReplyFromConfig → getReplyFromConfig（agent/model 选择、media 理解、session 初始化、指令解析）→ runPreparedReply（system prompt、queue 模式）→ runReplyAgent（steer/followup、block pipeline）→ runAgentTurnWithFallback（model fallback + 错误恢复）→ runEmbeddedPiAgent（双层重试：auth profile 轮换 × overflow compaction）→ runEmbeddedAttempt（session 创建、工具集组装、session.prompt() → LLM API）
13. **工具集组装** — base SDK tools + exec/process/apply_patch + channel tools + openclaw tools；8 层策略过滤链（profile→global→agent→group→sandbox→subagent）
14. **流式响应管道** — pi-agent-core 事件 → subscribeEmbeddedPiSession（状态机：text/reasoning/tool/compaction）→ BlockReplyPipeline（分块+合并）→ callback → ReplyDispatcher
15. **Model Fallback** — primary → config fallbacks 链；FailoverError 触发切换，AbortError 直接抛出
16. **Auth Profile 轮换** — cooldown 跳过 → 尝试 → 失败 markFailure → 下一个 profile；成功 markGood + markUsed
17. **类型系统拓扑** — 类型分散在各模块(不在 src/types/)，通过 barrel export 聚合；配置层 31 个 types.*.ts 模块，协议层 11 个 TypeBox schema 文件导出 190+ Static 类型
18. **双 Schema 体系** — 配置用 Zod (复杂 transform/default)，协议用 TypeBox (标准 JSON Schema, 跨语言 codegen Swift/Kotlin)
19. **OpenClawConfig 顶层结构** — 26 个顶层字段覆盖：auth, env, agents, tools, models, channels, session, messages, commands, plugins, skills, cron, hooks, gateway, memory 等
20. **Plugin API 与 Hook 系统** — OpenClawPluginApi 提供 10 个 register 方法 + 类型安全的 on() 钩子订阅；15 个生命周期 Hook 通过 PluginHookHandlerMap 判别联合实现类型安全
21. **System Prompt 构建** — 6 阶段管线（运行时参数→bootstrap 文件→skills prompt→工具摘要→prompt 组装→诊断报告）；3 种模式（full/minimal/none）；~18 个条件 section；9 个 workspace bootstrap 文件（子 Agent 仅 2 个）；截断策略（head 70% + tail 20%，默认 20K/文件）；不含实际时间（cache 稳定性）；SOUL.md 人格系统
22. **Session 运行时消息管理** — 6 层处理包裹 SessionManager（文件修复→写锁→加载→守卫层 monkey-patch→初始化修正→8 步消息清洗）；Transcript Policy 按 provider 决定清洗策略；工具结果双层截断（持久化 400K + 发送前 30% 上下文窗口）；Synthetic Tool Results 补全缺失 toolResult；历史轮次裁剪（per-DM/per-channel 可配置）；写锁支持可重入+死锁恢复
23. **Compaction 机制** — 4 种触发（overflow 自动重试最多 3 次 / manual Gateway RPC / cache_ttl Anthropic 缓存过期 / safeguard 预防性）；核心算法三层摘要 Fallback（全量→过滤超大→纯文本）；自适应 chunk ratio（0.15-0.4）；safeguard 扩展通过 session_before_compact 钩子增强摘要（History Pruning + Tool Failures + File Ops）；Memory Flush 预刷写（阈值 = ctx - reserve - 4K）；双层 Lane 排队（session 串行 + global 并发控制）；溢出恢复决策树（compaction → tool result truncation → 放弃）
24. **Skill 系统** — 纯 Prompt 注入机制，SKILL.md = YAML frontmatter + Markdown 说明书；6 来源加载合并（extra < bundled < managed < agents-personal < agents-project < workspace）；5 维资格判定（enabled + allowlist + OS + bins/env/config + always）；环境变量三级来源（process.env > config.env > apiKey→primaryEnv）；Snapshot 缓存于 SessionEntry（版本号触发刷新：watch/manual/remote-node）；5 种安装器（brew/node/go/uv/download）+安装前安全扫描；远程 macOS 节点 bin 探测扩展本机资格；chokidar 热监控 + debounce；/skill 命令 + command-dispatch 机制
25. **Sandbox 系统** — Docker 容器隔离层；3 种 mode（off/non-main/all）+ 3 种 scope（session/agent/shared）决定隔离粒度；容器生命周期（configHash 变更检测：冷删重建 vs 热提示手动 recreate）；FS Bridge 通过 docker exec 代理全部文件操作（路径逃逸+symlink 检测）；Tool Policy allow/deny 过滤（默认允许 exec/read/write/edit 等，禁止 browser/channel/gateway）；沙箱化工具替换（read/write/edit/exec/apply_patch/browser 6 个工具被 sandbox 版本替代）；Browser Bridge 独立容器化浏览器（CDP+VNC+noVNC）；双注册表持久化（containers.json + browsers.json）；Prune 5 分钟节流自动清理（24h idle / 7d 最大年龄）；Media Staging 入站媒体暂存到 workspace/media/inbound/；CLI sandbox list/recreate/explain 命令

## 已完成的笔记

- `notes/gateway.md` — Gateway 核心流程 + Chat 聊天推理流程
- `notes/architecture-decisions.md` — 跨模块架构决策（AD-1: TypeBox+AJV vs Zod 双 Schema 体系）
- `notes/config-loading.md` — 配置文件读取流程（7 阶段管线）
- `notes/plugins.md` — 插件系统（5 阶段加载管线、4 来源发现、Plugin API、15 种钩子）
- `notes/channels.md` — Channel 系统全貌（Dock vs Plugin 两层架构、20 个 Adapter、入站/出站消息流、生命周期、崩溃恢复）
- `notes/agent-runtime.md` — Agent Runtime 全貌（完整调用链、双层重试、工具组装、流式管道、model fallback、auth profile）
- `notes/sessions.md` — Session 管理全貌（Session Key 寻址、Store 持久化、SessionEntry 模型、生命周期、Transcript、维护机制）
- `notes/types-data-model.md` — 类型系统与数据模型（类型拓扑、OpenClawConfig、协议类型、Adapter/Hook/Plugin 类型、设计模式）
- `notes/system-prompt.md` — System Prompt 构建（6 阶段管线、3 种模式、~18 section、bootstrap 文件、截断、Skills/工具摘要、诊断报告）
- `notes/session-runtime.md` — Session 运行时消息管理（6 层处理、8 步清洗、Transcript Policy、工具结果截断、历史裁剪、写锁）
- `notes/compaction.md` — Compaction 机制（4 种触发、三层摘要 Fallback、safeguard 扩展、Memory Flush、溢出恢复决策树）
- `notes/skills.md` — Skill 系统（6 来源加载、5 维资格判定、Snapshot 缓存、Prompt 注入、环境变量注入、安装器、安全扫描、远程节点、热刷新）
- `notes/sandbox.md` — Sandbox 系统（Docker 容器隔离、mode/scope、FS Bridge、Tool Policy、Browser Bridge、Registry、Prune、Media Staging）

## 下一步建议

1. **Media 处理管道** (`src/media/`): 图片/音频/视频理解的多模型 fallback。（优先级 ★☆☆）
5. **Security / Pairing** (`src/security/`): 配对、访问控制、审计。（优先级 ★☆☆）

## 待解答的问题

- [x] dispatchReplyFromConfig 到 Agent Runtime 的完整调用链
- [x] Channel 插件枚举、ChannelPlugin 契约（20 个 Adapter）、ChannelManager 启停生命周期
- [x] Channel 插件如何将外部消息路由到 Agent？（inbound 消息流：monitor → dispatchInboundMessage）
- [x] Channel 出站消息流（ReplyDispatcher → 写前队列 → 通道适配器 → 平台 API）
- [x] 配置文件读取流程（路径解析、JSON5、$include、${ENV}、legacy 迁移、校验、默认值）
- [x] 插件加载流程（发现、Manifest、启用决策、jiti 加载、注册）
- [x] 类型系统拓扑（分散在各模块、barrel export、双 Schema 体系）
- [ ] config-reload.ts 的热重载 vs 重启决策逻辑
- [ ] Node（远程节点）的 pairing 和 command 执行流程
- [ ] 广播系统的 backpressure（dropIfSlow）机制
- [x] Session Store 持久化（缓存+写锁+原子写入）、SessionEntry 模型、Session Key 寻址
- [x] Session 生命周期（initSessionState: reset/freshness/fork）
- [x] Session Transcript JSONL 写入 + 维护机制（prune/cap/rotate）
- [x] Compaction 机制（上下文压缩、记忆刷写）
- [ ] Hook Runner 的调度逻辑（并发/串行、错误处理）
- [ ] 插件运行时 createPluginRuntime() 的内部结构
- [x] System Prompt 模板引擎的完整参数列表和组装逻辑（6 阶段管线、3 种模式、~18 section、bootstrap 文件截断）
- [x] Sandbox Docker 容器的创建和 filesystem bridge 机制
