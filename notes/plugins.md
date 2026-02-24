# 插件系统阅读笔记

> 阅读日期: 2026-02-24

## 概述

OpenClaw 的插件系统是 Gateway 启动的第二步（配置加载之后）。它负责发现、加载、注册所有插件，为系统提供工具（Tools）、消息通道（Channels）、LLM 提供者（Providers）、生命周期钩子（Hooks）等扩展能力。

核心代码位于 `src/plugins/`，调用入口在 `src/gateway/server-plugins.ts`。

## 关键文件

| 文件 | 职责 |
|------|------|
| `server-plugins.ts` | Gateway 侧薄封装，调用 `loadOpenClawPlugins()` 并收集 gateway methods |
| `loader.ts` | **主编排器** — `loadOpenClawPlugins()` 协调全流程 |
| `discovery.ts` | 4 来源插件发现 |
| `manifest-registry.ts` | Manifest 加载 + 200ms 缓存 |
| `manifest.ts` | `openclaw.plugin.json` 文件读取 |
| `registry.ts` | 插件注册表 + Plugin API 工厂 |
| `config-state.ts` | 启用/禁用决策 + 配置归一化 |
| `types.ts` | 完整类型定义（API, Hooks, Definition） |
| `bundled-dir.ts` | 内置扩展目录定位 |
| `runtime.ts` | 全局活跃 registry 管理 |
| `hook-runner-global.ts` | 全局 Hook Runner 初始化 |
| `commands.ts` | 插件命令注册/清理 |
| `schema-validator.ts` | JSON Schema 校验（用于插件配置） |

## 5 阶段加载管线

```
阶段 1: 配置归一化 (config-state.ts)
  normalizePluginsConfig(cfg.plugins)
  → NormalizedPluginsConfig { enabled, entries, deny, allow, slots, loadPaths }
      │
阶段 2: 插件发现 (discovery.ts)
  discoverOpenClawPlugins({ workspaceDir, extraPaths })
  → 扫描 4 个来源，产出 candidates[] + diagnostics[]
      │
阶段 3: Manifest 注册 (manifest-registry.ts)
  loadPluginManifestRegistry({ candidates, config, ... })
  → 为每个 candidate 加载 openclaw.plugin.json
      │
阶段 4: 逐插件处理 (loader.ts 主循环)
  对每个 candidate:
    去重 → 启用决策 → configSchema 检查 → jiti 加载 → 解析导出
    → Memory slot → 配置校验 → register(api)
      │
阶段 5: 收尾
  → 缓存 registry → setActivePluginRegistry → initializeGlobalHookRunner
```

## 阶段 1: 配置归一化

`normalizePluginsConfig()` (`config-state.ts`) 将用户配置的 `plugins` 字段转换为标准化结构：

```typescript
NormalizedPluginsConfig = {
  enabled: boolean,            // 全局开关（默认 true）
  entries: Record<id, { enabled?, config? }>,  // 每个插件的配置
  deny: Set<string>,           // 黑名单
  allow: Set<string>,          // 白名单
  slots: { memory?: string },  // 特殊槽位（memory 类型只能有一个）
  loadPaths: string[],         // 额外加载路径
}
```

### 测试环境特殊处理

`applyTestPluginDefaults()`: 在 Vitest 环境下（`process.env.VITEST`），默认设置 `plugins.enabled = false`。避免单测意外加载重量级插件依赖。用户可在测试中显式覆盖。

## 阶段 2: 插件发现

`discoverOpenClawPlugins()` (`discovery.ts`) 按优先级从 4 个来源搜索：

| 优先级 | 来源 | origin 标记 | 路径示例 |
|--------|------|-------------|----------|
| 1 (最高) | 配置显式路径 | `"config"` | `plugins.load.paths[]` 中的路径 |
| 2 | 工作区扩展 | `"workspace"` | `{workspaceDir}/.openclaw/extensions/` |
| 3 | 全局扩展 | `"global"` | `~/.openclaw/extensions/` |
| 4 (最低) | 内置扩展 | `"bundled"` | 项目根 `extensions/` 目录 |

### 扫描逻辑

- **目录型插件**:
  - 有 `package.json` → 读 `openclaw.extensions` 字段（支持多入口）
  - 无 `package.json` → 找 `index.ts` / `index.js`
- **单文件插件**: 直接是 `.ts` 或 `.js` 文件

### 产出

```typescript
PluginCandidate = {
  idHint: string,      // 从目录名/文件名推断的 ID
  source: string,      // 入口文件绝对路径
  rootDir: string,     // 插件根目录
  origin: "config" | "workspace" | "global" | "bundled",
  workspaceDir?: string,
}
```

### 内置扩展目录定位 (`bundled-dir.ts`)

`resolveBundledPluginsDir()` 查找顺序:
1. `OPENCLAW_BUNDLED_PLUGINS_DIR` 环境变量
2. 可执行文件旁的 `extensions/` 目录
3. 从模块位置向上遍历查找包根的 `extensions/`

## 阶段 3: Manifest 注册

`loadPluginManifestRegistry()` (`manifest-registry.ts`) 为每个 candidate 加载 `openclaw.plugin.json`。

### Manifest 格式 (`manifest.ts`)

```typescript
PluginManifest = {
  id: string,              // 必须 — 插件唯一 ID
  configSchema: object,    // 必须 — JSON Schema（即使为空 {}）
  kind?: "memory" | ...,   // 插件类型
  channels?: string[],     // 声明的通道
  providers?: string[],    // 声明的 LLM 提供者
  skills?: string[],       // 声明的技能
  name?: string,
  description?: string,
  version?: string,
  uiHints?: object,        // 配置 UI 提示
}
```

**关键约束**: `id` 和 `configSchema` 都是必须字段。缺少 configSchema 的插件在阶段 4 会被标记为 error。

**缓存**: 200ms TTL，避免热重载时重复读盘。

## 阶段 4: 逐插件处理

`loader.ts` 主循环中对每个 candidate 执行 7 个检查：

### ① 去重

`seenIds` Map 保证同 ID 只加载第一个（高优先级来源）。后续同 ID 的 candidate 标记为 `disabled`，error 信息为 `"overridden by {origin} plugin"`。

### ② 启用/禁用决策 (`resolveEnableState`)

决策链（短路求值）：

```
全局 enabled=false           → 禁用（"plugins globally disabled"）
  → deny 名单命中            → 禁用（"denied by config"）
    → allow 名单存在但未命中  → 禁用（"not in allow list"）
      → per-entry enabled=false → 禁用
        → bundled 插件 → 检查 BUNDLED_ENABLED_BY_DEFAULT
          → 非 bundled 插件    → 默认启用
```

默认启用的内置插件 (`BUNDLED_ENABLED_BY_DEFAULT`):
- `device-pair` — 设备配对
- `phone-control` — 手机控制
- `talk-voice` — 语音对话

### ③ configSchema 检查

Manifest 必须有 configSchema，否则标记 `status: "error"`。

### ④ jiti 加载模块

使用 [jiti](https://github.com/unjs/jiti) 实现 TypeScript 运行时编译：

```typescript
const jiti = createJiti(import.meta.url, {
  interopDefault: true,
  extensions: [".ts", ".tsx", ".mts", ".cts", ".js", ".mjs", ".cjs", ".json"],
  alias: { "openclaw/plugin-sdk": resolvePluginSdkAlias() },
});
mod = jiti(candidate.source);
```

**SDK 别名映射** (`resolvePluginSdkAlias`):
- 开发环境: `src/plugin-sdk/index.ts`
- 生产环境: `dist/plugin-sdk/index.js`
- 从模块位置向上最多 6 层查找

### ⑤ 解析导出 (`resolvePluginModuleExport`)

```
模块导出 → 取 .default（如果有）
  → 是函数 → 视为 register 函数
  → 是对象 → 取 .register 或 .activate 属性
```

支持两种插件写法:
```typescript
// 写法 1: 直接导出函数
export default (api: OpenClawPluginApi) => {
  api.registerTool({ ... });
};

// 写法 2: 导出定义对象
export default {
  id: "my-plugin",
  name: "My Plugin",
  register(api: OpenClawPluginApi) {
    api.registerTool({ ... });
  },
};
```

如果导出对象的 `id` 与 manifest 的 `id` 不一致，会产生 warn 诊断。`kind` 不一致同理。

### ⑥ Memory Slot 决策 (`resolveMemorySlotDecision`)

- `kind === "memory"` 的插件最多只能激活一个
- 通过 `plugins.slots.memory` 指定使用哪个
- 未指定 slot 时，取第一个发现的 memory 插件
- 已有选中的 memory 插件后，其余 memory 插件自动禁用

### ⑦ 调用 register(api)

```typescript
const api = createApi(record, { config, pluginConfig });
const result = register(api);
// 如果 result 是 Promise → warn（async registration is ignored）
```

**注册必须同步完成**。异步返回 Promise 会被忽略（只产生 warn 诊断），保证启动顺序确定性。

## Plugin API

`createApi()` (`registry.ts`) 返回的 `OpenClawPluginApi` 是插件的全部能力入口：

| 方法 | 注册内容 | 存储位置 |
|------|----------|----------|
| `registerTool(def)` | Agent 可调用的工具 | `registry.tools[]` |
| `registerHook(name, handler)` | 生命周期钩子 | `registry.hooks[]` |
| `registerChannel(def)` | 消息通道 | `registry.channels[]` |
| `registerProvider(def)` | LLM 提供者 | `registry.providers[]` |
| `registerGatewayMethod(name, handler)` | Gateway RPC 方法 | `registry.gatewayHandlers{}` |
| `registerHttpHandler(handler)` | HTTP 中间件 | `registry.httpHandlers[]` |
| `registerHttpRoute(route)` | HTTP 路由 | `registry.httpRoutes[]` |
| `registerCli(registrar)` | CLI 子命令 | `registry.cliRegistrars[]` |
| `registerService(def)` | 后台服务（start/stop） | `registry.services[]` |
| `registerCommand(def)` | 用户命令（/xxx） | `registry.commands[]` |
| `on(event, handler)` | 类型安全的事件监听 | `registry.typedHooks[]` |

API 上的便捷属性:
- `api.config` — 全局 OpenClawConfig
- `api.pluginConfig` — 当前插件的已校验配置值
- `api.logger` — 带插件 ID 前缀的日志器

### PluginRegistry 完整结构

```typescript
PluginRegistry = {
  plugins: PluginRecord[],        // 所有插件记录（含 disabled/error）
  tools: ToolDefinition[],
  hooks: HookEntry[],
  typedHooks: TypedHookEntry[],
  channels: ChannelDefinition[],
  providers: ProviderPlugin[],
  gatewayHandlers: Record<string, GatewayRequestHandler>,
  httpHandlers: HttpHandler[],
  httpRoutes: HttpRoute[],
  cliRegistrars: CliRegistrar[],
  services: ServiceDefinition[],
  commands: PluginCommandDefinition[],
  diagnostics: PluginDiagnostic[],
}
```

### PluginRecord 状态

```typescript
PluginRecord = {
  id, name, description, version,
  source,          // 入口文件路径
  origin,          // "config" | "workspace" | "global" | "bundled"
  kind,            // "memory" | undefined
  enabled: boolean,
  status: "loaded" | "disabled" | "error",
  error?: string,  // 失败/禁用原因
  // 注册统计
  toolNames: string[],
  hookNames: string[],
  channelIds: string[],
  providerIds: string[],
  gatewayMethods: string[],
  cliCommands: string[],
  services: string[],
  commands: string[],
  httpHandlers: number,
  hookCount: number,
  // 配置相关
  configSchema: boolean,
  configUiHints?: object,
  configJsonSchema?: object,
}
```

## 15 种生命周期钩子

```
Agent 相关:
  before_agent_start   — Agent 启动前（可修改 prompt）
  agent_end            — Agent 运行结束

会话相关:
  session_start        — 会话创建
  session_end          — 会话结束
  before_compaction    — 上下文压缩前
  after_compaction     — 上下文压缩后
  before_reset         — 会话重置前

消息相关:
  message_received     — 收到消息
  message_sending      — 即将发送回复
  message_sent         — 回复已发送

工具相关:
  before_tool_call     — 工具调用前（可拦截/修改）
  after_tool_call      — 工具调用后
  tool_result_persist  — 工具结果持久化

Gateway 相关:
  gateway_start        — Gateway 启动完成
  gateway_stop         — Gateway 关闭
```

## 阶段 5: 收尾

```typescript
// 1. 缓存 registry（key = workspaceDir + 序列化的 plugins 配置）
registryCache.set(cacheKey, registry);

// 2. 设置为全局活跃 registry（供其他模块通过 runtime.ts 访问）
setActivePluginRegistry(registry, cacheKey);

// 3. 初始化全局 Hook Runner（聚合所有插件的 hooks，供运行时调用）
initializeGlobalHookRunner(registry);
```

## 设计亮点

1. **4 层发现 + 优先级覆盖** — 用户可在配置路径、工作区、全局、内置 4 个层级放置插件，同 ID 高优先级覆盖低优先级
2. **jiti 运行时编译** — 插件可直接用 TypeScript 编写，无需预编译
3. **SDK 别名映射** — `openclaw/plugin-sdk` 自动映射到正确路径，插件不需要关心是源码还是 dist
4. **Memory Slot 互斥** — 通过 slot 机制保证同类型插件只有一个激活，避免冲突
5. **同步注册** — 注册必须同步完成，保证启动顺序确定性
6. **诊断系统** — 全程收集 diagnostics（warn/error），不因单个插件失败而中断整个加载流程
7. **缓存层** — registry 按 (workspaceDir + config) 缓存，热重载时避免重复加载
8. **双写法支持** — 函数导出或对象导出均可，降低插件作者的门槛

## 待深入

- [ ] `hook-runner-global.ts` — Hook Runner 的具体调度逻辑（并发/串行、错误处理）
- [ ] `runtime/index.ts` — 插件运行时（createPluginRuntime）的内部结构
- [ ] 具体插件示例 — 挑一个内置插件（如 device-pair）看完整的注册流程
- [ ] `plugin-sdk/` — 插件 SDK 提供了哪些开发辅助
