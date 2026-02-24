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
