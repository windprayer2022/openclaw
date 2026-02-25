# 架构决策笔记

> 记录阅读过程中发现的跨模块架构决策与设计选择。

## AD-1: TypeBox+AJV vs Zod — 双 Schema 体系

> 发现日期: 2026-02-24

### 现象

项目**同时使用了两套 Schema 库**，各有明确分工：

| 场景 | 选择 | 文件 |
|------|------|------|
| Agent 工具定义（发给 LLM） | **TypeBox** | `src/agents/tools/*.ts` (30+ 文件) |
| WebSocket 协议 schema | **TypeBox** + **AJV** 运行时校验 | `src/gateway/protocol/schema/` |
| 配置文件校验 | **Zod** | `src/config/zod-schema*.ts` (14 文件) |
| 插件/扩展配置 | **Zod** | `extensions/*/config-schema.ts` |

### 为什么工具/协议层用 TypeBox+AJV

**核心原因：LLM API 需要原生 JSON Schema。**

TypeBox 的 `Type.Object(...)` 产出的就是标准 JSON Schema 对象，可以直接塞进 OpenAI/Anthropic/Google 的 tool calling 请求，零转换：

```ts
// src/agents/tools/web-search.ts
const WebSearchSchema = Type.Object({
  query: Type.String({ description: "Search query string." }),
});
// 产物就是 { type: "object", properties: { query: { type: "string", ... } } }
```

Zod 需要额外的 `zod-to-json-schema` 转换，且转出来的 schema 可能包含 LLM 提供商不认识的扩展字段。

**关键证据** — `src/agents/schema/typebox.ts:13` 的注释：

> *Avoid `Type.Union([Type.Literal(...)])` which compiles to `anyOf`. Some providers reject `anyOf` in tool schemas; a flat string enum is safer.*

说明作者需要**精确控制输出的 JSON Schema 结构**以适配不同 LLM 提供商，TypeBox 给了这种控制力。

**其他因素：**

1. **性能** — AJV 将 schema 编译为优化过的验证函数（见 `src/plugins/schema-validator.ts` 的缓存机制），对高频 WebSocket 消息校验很重要
2. **Schema 可序列化** — TypeBox schema 本身就是 JSON 对象，可直接发给客户端。`src/gateway/protocol/schema/protocol-schemas.ts` 导出了完整的 `Record<string, TSchema>` schema registry

### 为什么配置层用 Zod

配置只在启动时校验一次，性能不敏感。Zod 的 API 对复杂业务规则更友好：

- `.strict()` 拒绝多余字段
- `.superRefine()` 做跨字段校验（如 `zod-schema.ts:617` 检查 broadcast 引用的 agent id 必须存在）
- `.register(sensitive)` 标记敏感字段（用于日志脱敏）
- `.refine()` 做条件约束（如浏览器 profile 必须设 `cdpPort` 或 `cdpUrl`）
- `.toJSONSchema()` 为 Web UI 配置编辑器生成 schema（见 `src/config/schema.ts:9`）

### 总结

务实的选择 — 用 TypeBox 在 LLM/协议边界精确控制 JSON Schema 输出，用 Zod 在配置层享受更好的校验 DX。不是非此即彼，而是各取所长。

---

## AD-2: Plugin Hook 系统 — 集中定义 + 分散调用

> 发现日期: 2026-02-25

### 现象

15 个生命周期 Hook 的**类型定义和注册逻辑集中在 3 个文件**，但**调用点分散在 ~10 个文件、4 个子系统**中：

| 调用者（子系统） | 触发的 Hook | 调用文件 |
|----------------|------------|---------|
| Gateway 启停 | `gateway_start`, `gateway_stop` | `gateway/server.impl.ts` |
| 消息分发 | `message_received` | `auto-reply/reply/dispatch-from-config.ts` |
| Session 管理 | `session_start`, `session_end` | `auto-reply/reply/session.ts` |
| 命令处理 | `before_reset` | `auto-reply/reply/commands-core.ts` |
| Agent 运行 | `before_agent_start`, `agent_end` | `agents/pi-embedded-runner/run/attempt.ts` |
| 上下文压缩 | `before_compaction`, `after_compaction` | `agents/pi-embedded-runner/compact.ts` |
| 工具拦截 | `before_tool_call` | `agents/pi-tools.before-tool-call.ts` |
| 工具结果 | `after_tool_call` | `agents/pi-tool-definition-adapter.ts` |
| 工具持久化 | `tool_result_persist` (同步) | `agents/session-tool-result-guard-wrapper.ts` |
| 消息出站 | `message_sending`, `message_sent` | `infra/outbound/deliver.ts` |

集中的部分：

| 职责 | 文件 |
|------|------|
| 类型定义（PluginHookHandlerMap 判别联合） | `plugins/types.ts` |
| 注册（registerTypedHook / api.on） | `plugins/registry.ts` |
| 调度（createHookRunner → runVoidHook / runModifyingHook） | `plugins/hooks.ts` |

`hookRunner` 是单一实例，Gateway 启动时创建，通过依赖注入传递到各子系统。

### 为什么不集中调用

根本原因：**15 个 hook 跨越了无法线性化的控制流**。

一条消息的完整路径包含队列边界、异步断裂、循环和条件分支：

```
inbound (同步 ACK)
  → 防抖队列 (异步, 可能合并多条)
    → Agent (异步, 内含多轮 tool call 循环, 可能触发 compaction)
      → outbound (异步, 可能多条 block reply, 仿人延迟)
```

具体障碍：

| 阻碍集中化的因素 | 举例 |
|----------------|------|
| **异步断裂** | `message_received` 在入站触发，`message_sending` 在出站触发，中间隔着整个 Agent 推理 |
| **循环** | `before/after_tool_call` 在 Agent 推理循环内可能执行多次 |
| **同步约束** | `tool_result_persist` 是唯一的同步 hook（JSONL 热路径），不能放进 async 管道 |
| **条件分支** | `before/after_compaction` 只在上下文溢出时触发 |
| **生命周期粒度不同** | `gateway_start/stop` 是进程级，`session_start/end` 是会话级 |

### 备选方案与权衡

**方案 A：事件总线（EventEmitter）**
- 可以集中 emit，但**修改型 hook 需要同步返回值**（`message_sending` 返回 `{ cancel: true }` 取消发送），EventEmitter 做不到
- 丧失类型安全 — emit 的参数类型无法约束

**方案 B：集中式 orchestrator**
- 一个函数串联所有 hook，但 Agent 内部的 tool call 循环和 compaction 是 Agent Runtime 自己管的逻辑
- 强行拉到外层 orchestrator → Agent Runtime 丧失封装性，orchestrator 变成上帝函数

**方案 C：切面编程（AOP）**
- 用装饰器/Proxy 自动注入 hook，理论上最优雅
- 但 TypeScript 装饰器对函数参数类型推导弱，运行时 Proxy 有性能开销

### 当前方案的权衡

| 优点 | 缺点 |
|------|------|
| 每个 hook 精确出现在业务发生点，语义清晰 | 新增 hook 需要找到正确的调用位置（grep） |
| 修改型 hook 直接拿返回值影响控制流 | 调用点分散在 ~10 个文件 |
| 不需要把 Agent 内部循环暴露给外层 | hookRunner 需要层层依赖注入 |
| 同步/异步 hook 可以自然区分 | 没有一个地方能一眼纵览完整生命周期 |

### Hook 的两种执行模式

| 模式 | 执行方式 | 用于 |
|------|---------|------|
| **通知型** (runVoidHook) | `Promise.all` **并行**，返回 void | 11 个 hook（agent_end, message_received, message_sent 等） |
| **修改型** (runModifyingHook) | **串行**按 priority 降序，合并返回值 | 3 个 hook + 1 个同步特例 |

修改型 hook 的合并策略各不同：

| Hook | 合并策略 |
|------|---------|
| `before_agent_start` | `systemPrompt` 后者覆盖；`prependContext` 拼接（`\n\n`） |
| `message_sending` | `content`, `cancel` 后者覆盖 |
| `before_tool_call` | `params`, `block`, `blockReason` 后者覆盖 |
| `tool_result_persist` | **同步链式**：上一个 handler 的 message 输出是下一个的输入 |

### 类型安全机制

核心是泛型参数 `K extends PluginHookName` 在 `api.on()` 调用点被推导，通过 `PluginHookHandlerMap[K]` 索引类型传播到 event、ctx 和返回值。插件代码**零手动类型标注**：

```typescript
// K 被推导为 "message_sending"
api.on("message_sending", async (event, ctx) => {
  // event 自动推导为 { to: string; content: string; metadata?: ... }
  // ctx   自动推导为 { channelId: string; accountId?: string; ... }
  // 返回值约束为 { content?: string; cancel?: boolean } | void
  if (shouldBlock) return { cancel: true };
});
```

类型流：

```
api.on<K>("message_sending", handler)
  → K = "message_sending"                          (字面量推导)
  → handler: PluginHookHandlerMap["message_sending"]  (索引类型)
  → (event: MessageSendingEvent, ctx: MessageContext)  (参数展开)
    => MessageSendingResult | void                     (返回值约束)
```

注册侧 `registerTypedHook<K>` 保持泛型，存入 `registry.typedHooks`。调度侧 `getHooksForName<K>` 按 hookName 过滤并用 `as PluginHookRegistration<K>` 还原泛型。

### 错误隔离

所有 hook 默认 `catchErrors: true` — 一个插件的 hook 崩溃不影响其他插件和主流程。错误被 catch + 日志记录（标注来源 pluginId）。`tool_result_persist` 额外检测误返回 Promise 的情况（此 hook 是同步的），打 warn 并跳过。

### 总结

「集中定义 + 集中实例 + 分散调用」是在复杂异步控制流中嵌入生命周期 hook 的务实选择。代价是调用点需要 grep 才能找全，但换来了每个 hook 在业务最近位置触发、修改型 hook 直接影响控制流、同步/异步自然区分等好处。
