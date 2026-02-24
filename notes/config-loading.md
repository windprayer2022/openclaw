# 配置文件读取流程阅读笔记

> 阅读日期: 2026-02-24

## 概述

Gateway 启动时的第一步就是读取配置。入口在 `src/gateway/server.impl.ts:175-225`，核心实现在 `src/config/io.ts`。

整个流程是一个 **7 阶段管线**：

```
路径定位 → JSON5 解析 → $include 展开 → ${ENV} 替换 → Legacy 迁移 → Zod 校验 → 默认值 + 插件自动启用
```

## 关键文件

| 文件 | 职责 |
|------|------|
| `config/io.ts` | 核心 I/O：读取、解析、写入、缓存 |
| `config/paths.ts` | 路径解析（多更名历史兼容） |
| `config/includes.ts` | `$include` 指令：模块化配置合并 |
| `config/env-substitution.ts` | `${VAR}` 环境变量替换 |
| `config/legacy.ts` | 旧格式检测 |
| `config/legacy-migrate.ts` | 旧格式自动迁移 |
| `config/legacy.migrations.*.ts` | 具体迁移规则（3 个 part 文件） |
| `config/validation.ts` | Zod + 插件 Schema 校验 |
| `config/zod-schema.ts` | 配置的 Zod Schema 定义 |
| `config/defaults.ts` | 运行时默认值 |
| `config/plugin-auto-enable.ts` | 检测已配置通道/provider → 自动启用插件 |
| `config/env-preserve.ts` | 写入时还原 ${VAR} 引用 |
| `config/normalize-paths.ts` | 路径归一化 |
| `config/runtime-overrides.ts` | 环境变量覆盖配置 |

## 阶段 1：路径定位

**代码**: `config/paths.ts`

产品经历过多次更名 (moldbot → moltbot → clawdbot → openclaw)，路径需要向后兼容。

优先级：
1. `OPENCLAW_CONFIG_PATH` 环境变量
2. `$OPENCLAW_STATE_DIR/openclaw.json`
3. `$STATE_DIR/{clawdbot,moltbot,moldbot}.json` (遗留文件名)
4. `~/.openclaw/openclaw.json` (默认)
5. `~/.clawdbot/` 等遗留目录

`createConfigIO()` 从候选路径中找第一个存在的文件。

## 阶段 2：读取 + JSON5 解析

**代码**: `config/io.ts:658-710`

- 先加载 `.env` 文件 (`loadDotEnv`)
- 用 JSON5 解析（支持注释、尾逗号）
- 文件不存在时不报错，返回空配置 + 全默认值

## 阶段 3：$include 指令解析

**代码**: `config/includes.ts`

支持模块化拆分配置：
```json5
{ "$include": "./base.json5" }
{ "$include": ["./a.json5", "./b.json5"] }
```

- 深度合并：对象递归合并，数组拼接，原始值后者覆盖
- 循环检测：`visited` Set 防循环
- 深度限制：最多 10 层

## 阶段 4：环境变量替换

**代码**: `config/env-substitution.ts`

- `${VAR}` → 替换为 env 变量值
- `$${VAR}` → 转义，输出字面量
- 只匹配大写变量名 `[A-Z_][A-Z0-9_]*`
- 变量不存在 → 抛 `MissingEnvVarError`

关键：在替换前先执行 `applyConfigEnv()`，把 `config.env` 中的值写入 process.env，这样 `${VAR}` 可以引用配置自定义变量。

## 阶段 5：Legacy 旧格式检测 + 自动迁移

**代码**: `config/legacy.ts`, `config/legacy-migrate.ts`

在 `server.impl.ts:176-196`：
1. `findLegacyConfigIssues()` 扫描旧格式字段
2. 发现问题 → `migrateLegacyConfig()` 自动迁移
3. 迁移后写回文件 → 重新读取
4. Nix 模式不自动迁移（配置由外部管理）

## 阶段 6：Schema 校验

**代码**: `config/validation.ts`

校验层次：
1. Zod Schema 结构/类型校验
2. Agent 目录去重检查
3. Avatar 路径必须在 workspace 内
4. 插件 ID/config schema/channel ID 合法性

## 阶段 7：运行时默认值 + 插件自动启用

默认值叠加链：
```
applyMessageDefaults → applySessionDefaults → applyLoggingDefaults
→ applyAgentDefaults → applyModelDefaults → applyContextPruningDefaults
→ applyCompactionDefaults → applyTalkApiKey → normalizeConfigPaths
→ applyConfigOverrides
```

然后 `applyPluginAutoEnable()` 检测配置中是否有通道/provider 的凭据，自动启用对应插件并写回文件。

## 配置项全景

配置的类型定义在 `src/config/types.openclaw.ts`（`OpenClawConfig`），Zod Schema 在 `src/config/zod-schema.ts`。
类型文件按领域拆分为 30+ 个 `types.*.ts` 子文件。

以下是顶层配置字段概览，具体细节留到各功能模块阅读时再深入。

### 系统基础设施

| 字段 | 类型文件 | 说明 |
|------|----------|------|
| `meta` | (内联) | 元信息：最后写入的 OpenClaw 版本号、时间戳，用于版本前后兼容检测 |
| `env` | (内联) | 环境变量注入：`vars` 键值对、`shellEnv` 控制是否从 login shell 导入变量 |
| `logging` | `types.base.ts` | 日志级别 (silent~trace)、输出文件、console 样式 (pretty/compact/json)、敏感信息脱敏 |
| `diagnostics` | `types.base.ts` | 诊断/遥测：OpenTelemetry (traces/metrics/logs)、cache trace 记录 |
| `update` | (内联) | 自更新：channel (stable/beta/dev)、启动时是否检查更新 |
| `wizard` | (内联) | 引导向导的运行状态记录（上次运行时间、版本等） |

### 核心运行时 — Gateway 服务

| 字段 | 类型文件 | 说明 |
|------|----------|------|
| `gateway` | `types.gateway.ts` | Gateway 服务器配置的主节点 |
| `gateway.port` | | 监听端口（默认 18789） |
| `gateway.mode` | | 运行模式：`local`（本地直连）/ `remote`（远程节点） |
| `gateway.bind` | | 绑定地址策略：auto / loopback / lan / tailnet / custom |
| `gateway.auth` | | Gateway 认证：token 或 password 模式 |
| `gateway.controlUi` | | 内置 Web 控制台：开关、basePath、允许的 Origins |
| `gateway.tls` | | TLS 配置：证书路径或自动生成 |
| `gateway.tailscale` | | Tailscale 暴露模式：off / serve / funnel |
| `gateway.remote` | | 远程连接配置：URL、SSH/直连传输、token 认证 |
| `gateway.reload` | | 配置热重载策略：off / restart / hot / hybrid |
| `gateway.http` | | HTTP 端点：OpenAI Chat Completions API / OpenResponses API 开关及参数 |
| `gateway.nodes` | | 远程节点管理：浏览器代理模式、允许/禁止的命令 |
| `gateway.tools` | | Gateway 级工具白名单/黑名单 |
| `gateway.trustedProxies` | | 信任的反向代理 IP（用于正确获取客户端 IP） |

### AI Agent 系统

| 字段 | 类型文件 | 说明 |
|------|----------|------|
| `agents` | `types.agents.ts` | Agent 配置的主节点 |
| `agents.defaults` | `types.agent-defaults.ts` | 全局默认值：主模型 (`model.primary`)、fallback 链、并发数、上下文剪枝 (`contextPruning`)、compaction、心跳 (`heartbeat`)、子 Agent (`subagents`) 等 |
| `agents.list` | | Agent 列表：每个 Agent 有 `id`、`dir`（工作目录）、`identity`（名称/头像/人设 prompt）、可覆盖 defaults 的独立配置 |
| `auth` | `types.auth.ts` | AI Provider 认证 Profile：provider（anthropic/openai/google 等）、mode（api_key/oauth）、轮换顺序 (`order`)、计费退避冷却 (`cooldowns`) |
| `models` | `types.models.ts` | 模型定义：按 provider 分组，每个模型有 id/name/contextWindow/maxTokens/cost/reasoning/input 等参数 |
| `tools` | `types.tools.ts` | Agent 可用工具配置 |
| `bindings` | `types.agents.ts` | Agent 绑定规则：将特定通道/session 绑定到特定 Agent |
| `approvals` | `types.approvals.ts` | 执行审批：哪些 Agent 操作需要人工审批 |

### 消息与会话

| 字段 | 类型文件 | 说明 |
|------|----------|------|
| `messages` | `types.messages.ts` | 消息行为：ACK 反应范围 (`ackReactionScope`)、消息格式等 |
| `commands` | `types.messages.ts` | 命令配置：`restart` 是否允许等 |
| `session` | `types.base.ts` | 会话配置：主会话 key |
| `broadcast` | `types.messages.ts` | 广播规则：Agent 事件跨通道广播策略 |
| `audio` | `types.messages.ts` | 音频消息配置 |

### 消息通道

| 字段 | 类型文件 | 说明 |
|------|----------|------|
| `channels` | `types.channels.ts` | 所有消息通道的配置总入口 |
| | `types.telegram.ts` | Telegram：botToken、accounts（多账号）、自定义命令等 |
| | `types.discord.ts` | Discord：token、accounts、presence 等 |
| | `types.slack.ts` | Slack：botToken/appToken/userToken、accounts |
| | `types.whatsapp.ts` | WhatsApp：认证信息 |
| | `types.signal.ts` | Signal：account、HTTP/CLI 接入方式 |
| | `types.imessage.ts` | iMessage：cliPath |
| | `types.irc.ts` | IRC：host/nick/accounts |
| | `types.googlechat.ts` | Google Chat |
| | `types.msteams.ts` | Microsoft Teams |
| | (+ 扩展插件通道) | 38+ 扩展可注册更多通道 |

### 扩展系统

| 字段 | 类型文件 | 说明 |
|------|----------|------|
| `plugins` | `types.plugins.ts` | 插件系统：全局开关、allow/deny 名单、load 路径、slots（如 memory slot）、entries（按插件 ID 的开关和 config）、installs（已安装插件元数据） |
| `skills` | `types.skills.ts` | 技能系统：allowBundled 白名单、加载路径 (`extraDirs`)、watch 热重载、install 偏好（brew/npm/pnpm 等）、entries（按技能的开关/apiKey/env） |

### 功能模块

| 字段 | 类型文件 | 说明 |
|------|----------|------|
| `browser` | `types.browser.ts` | 浏览器控制：Chromium 路径、CDP URL、headless 模式、多 profile 配置 |
| `memory` | `types.memory.ts` | 记忆系统：backend（builtin / qmd）、qmd 搜索模式/路径/索引更新策略 |
| `talk` | `types.gateway.ts` | 语音/TTS：voiceId、模型、输出格式、打断策略 |
| `cron` | `types.cron.ts` | 定时任务：开关、存储路径、最大并发、会话保留期 |
| `hooks` | `types.hooks.ts` | Webhook 入站：token 认证、路径、session 路由、Gmail 集成、请求体限制 |
| `web` | `types.base.ts` | Web UI 连接配置：心跳间隔、重连策略（退避因子/抖动/最大尝试次数） |
| `discovery` | `types.gateway.ts` | 服务发现：mDNS/Bonjour 模式 (off/minimal/full)、广域发现开关 |
| `canvasHost` | `types.gateway.ts` | Canvas Host（可视化画布服务端）：开关、端口、热重载 |
| `nodeHost` | `types.node-host.ts` | 远程节点宿主：浏览器代理开关、允许的 profile |
| `ui` | (内联) | UI 外观：主题色 (`seamColor`)、助手名称和头像 |
| `media` | (内联) | 媒体处理：`preserveFilenames` 是否保留原始文件名 |

### 配置项之间的关系

```
                    ┌──────────────┐
                    │  env / meta   │  ← 基础设施层
                    └──────┬───────┘
                           │
        ┌──────────────────┼──────────────────┐
        ▼                  ▼                  ▼
   ┌─────────┐      ┌──────────┐       ┌──────────┐
   │ gateway  │      │  auth    │       │  models  │  ← 运行时核心
   │ (服务器) │      │ (凭据)   │       │ (模型定义)│
   └────┬────┘      └────┬─────┘       └────┬─────┘
        │                │                   │
        │           ┌────┴────┐              │
        │           ▼         ▼              │
        │      ┌────────┐ ┌───────┐          │
        │      │ agents │ │ tools │ ←────────┘  ← AI 执行层
        │      └───┬────┘ └───┬───┘
        │          │          │
   ┌────┴──────────┴──────────┴────┐
   ▼                               ▼
┌──────────┐                 ┌──────────────┐
│ channels │                 │ skills/plugins│  ← 接入 & 扩展层
│ (16+通道) │                 │ (53+技能/38+) │
└──────────┘                 └──────────────┘
        │                          │
        └──────────┬───────────────┘
                   ▼
          ┌────────────────┐
          │ memory / cron / │  ← 辅助功能层
          │ hooks / browser │
          └────────────────┘
```

## 零配置下的默认值

当配置文件不存在时，系统可以正常启动。默认值来自两层：

1. **`readConfigFileSnapshot` / `loadConfig` 中的 defaults 管线** — 在 `config/defaults.ts` 中，对空 `{}` 逐步叠加
2. **各子系统自身的硬编码常量** — 分散在 `config/paths.ts`、`config/agent-limits.ts`、`agents/defaults.ts`、`gateway/server-runtime-config.ts` 等文件

### 第一层：defaults 管线对空配置的作用

对空 `{}` 执行 defaults 链后，`readConfigFileSnapshot().config` 的结果：

```jsonc
{
  // applyMessageDefaults — config/defaults.ts:113
  "messages": {
    "ackReactionScope": "group-mentions"
  },

  // applyAgentDefaults — config/defaults.ts:294
  // （空 {} 没有 agents，但函数会创建 agents.defaults）
  "agents": {
    "defaults": {
      "maxConcurrent": 4,                          // DEFAULT_AGENT_MAX_CONCURRENT — config/agent-limits.ts:3
      "subagents": {
        "maxConcurrent": 8                          // DEFAULT_SUBAGENT_MAX_CONCURRENT — config/agent-limits.ts:4
      },
      // applyCompactionDefaults — config/defaults.ts:443
      "compaction": {
        "mode": "safeguard"
      }
    }
  }

  // applySessionDefaults — 无 session.mainKey 字段 → 不触发
  // applyLoggingDefaults — 无 logging 字段 → 不触发
  // applyModelDefaults — 无 models.providers 和 agents.defaults.models → 不触发
  // applyContextPruningDefaults — 无 Anthropic auth profile 且无 ANTHROPIC_API_KEY env → 不触发
  // applyTalkApiKey — 若无 ELEVENLABS_API_KEY env 也不在 shell profile 中 → 不触发
}
```

> 注意：`loadConfig()`（非 snapshot 版本）在文件不存在时直接返回 `{}`，不走 defaults 管线。
> 但 Gateway 启动后各子系统会用自己的默认值，见下面第二层。

### 第二层：子系统硬编码默认值

以下是 Gateway 启动时各子系统在没有配置项时使用的硬编码默认值：

#### Gateway 服务器

| 配置路径 | 默认值 | 来源 |
|----------|--------|------|
| `gateway.port` | `18789` | `config/paths.ts:216` (`DEFAULT_GATEWAY_PORT`) |
| `gateway.bind` | `"loopback"` | `gateway/server-runtime-config.ts:43` |
| `gateway.auth.mode` | `"token"` | `gateway/auth.ts:202` (无 password 时 fallback) |
| `gateway.auth.token` | `$OPENCLAW_GATEWAY_TOKEN` 环境变量 | `gateway/auth.ts:196` |
| `gateway.controlUi.enabled` | `true` | `gateway/server-runtime-config.ts:47` |
| `gateway.http.chatCompletions.enabled` | `false` | `gateway/server-runtime-config.ts:50` |
| `gateway.http.responses.enabled` | `false` | `gateway/server-runtime-config.ts:53` |
| `gateway.tailscale.mode` | `"off"` | `gateway/server-runtime-config.ts:72` |
| `canvasHost.enabled` | `true` (除非 `OPENCLAW_SKIP_CANVAS_HOST=1`) | `gateway/server-runtime-config.ts:86` |

#### 衍生端口

| 端口 | 默认值 | 来源 |
|------|--------|------|
| Bridge | `18790` (gateway + 1) | `config/port-defaults.ts:15` |
| Browser Control | `18791` (gateway + 2) | `config/port-defaults.ts:16` |
| Canvas Host | `18793` (gateway + 4) | `config/port-defaults.ts:17` |
| Browser CDP 范围 | `18800-18899` | `config/port-defaults.ts:18-19` |

#### Agent 系统

| 配置路径 | 默认值 | 来源 |
|----------|--------|------|
| 默认 Agent ID | `"main"` | `routing/session-key.ts:11` (`DEFAULT_AGENT_ID`) |
| 默认会话 key | `"main"` | `routing/session-key.ts:12` (`DEFAULT_MAIN_KEY`) |
| 默认 Provider | `"anthropic"` | `agents/defaults.ts:3` |
| 默认模型 | `"claude-opus-4-6"` | `agents/defaults.ts:4` |
| 默认上下文窗口 | `200,000` tokens | `agents/defaults.ts:6` (`DEFAULT_CONTEXT_TOKENS`) |
| Agent 最大并发 | `4` | `config/agent-limits.ts:3` |
| 子 Agent 最大并发 | `8` | `config/agent-limits.ts:4` |
| Compaction 模式 | `"safeguard"` | `config/defaults.ts:461` |
| 消息 ACK 范围 | `"group-mentions"` | `config/defaults.ts:121` |
| 模型最大输出 tokens | `8192` (不超过 contextWindow) | `config/defaults.ts:35` (`DEFAULT_MODEL_MAX_TOKENS`) |
| 模型输入类型 | `["text"]` | `config/defaults.ts:34` |
| 模型 cost | `{ input: 0, output: 0, cacheRead: 0, cacheWrite: 0 }` | `config/defaults.ts:28` |

#### Agent 工作目录

| 路径 | 默认值 | 来源 |
|------|--------|------|
| Agent workspace | `~/.openclaw/workspace/` | `agents/workspace.ts:19` |
| Agent 数据目录 | `~/.openclaw/agents/main/agent/` | `agents/agent-scope.ts:191` |

#### 模型别名（内置快捷名）

| 别名 | 映射到 | 来源 |
|------|--------|------|
| `opus` | `anthropic/claude-opus-4-6` | `config/defaults.ts:16` |
| `sonnet` | `anthropic/claude-sonnet-4-5` | `config/defaults.ts:17` |
| `gpt` | `openai/gpt-5.2` | `config/defaults.ts:20` |
| `gpt-mini` | `openai/gpt-5-mini` | `config/defaults.ts:21` |
| `gemini` | `google/gemini-3-pro-preview` | `config/defaults.ts:24` |
| `gemini-flash` | `google/gemini-3-flash-preview` | `config/defaults.ts:25` |

#### 条件默认值（仅在检测到 Anthropic 认证时生效）

当环境变量中存在 `ANTHROPIC_API_KEY` 或 `ANTHROPIC_OAUTH_TOKEN` 时，`applyContextPruningDefaults` 会额外设置：

| 配置路径 | 默认值 (api_key) | 默认值 (oauth) | 来源 |
|----------|-------------------|----------------|------|
| `agents.defaults.contextPruning.mode` | `"cache-ttl"` | `"cache-ttl"` | `config/defaults.ts:369` |
| `agents.defaults.contextPruning.ttl` | `"1h"` | `"1h"` | `config/defaults.ts:370` |
| `agents.defaults.heartbeat.every` | `"30m"` | `"1h"` | `config/defaults.ts:378-381` |
| Anthropic 模型 `cacheRetention` | `"short"` | (不设) | `config/defaults.ts:399` |

#### 配置缓存

| 参数 | 默认值 | 来源 |
|------|--------|------|
| 缓存 TTL | `200ms` | `config/io.ts:1081` (`DEFAULT_CONFIG_CACHE_MS`) |
| 配置备份轮换数 | `5` | `config/io.ts:70` (`CONFIG_BACKUP_COUNT`) |

#### Shell Env Fallback

当配置文件不存在时，如果 `shouldEnableShellEnvFallback` 为真，会尝试从 login shell 导入以下 key：

```
OPENAI_API_KEY, ANTHROPIC_API_KEY, ANTHROPIC_OAUTH_TOKEN,
GEMINI_API_KEY, ZAI_API_KEY, OPENROUTER_API_KEY,
AI_GATEWAY_API_KEY, MINIMAX_API_KEY, SYNTHETIC_API_KEY,
ELEVENLABS_API_KEY, TELEGRAM_BOT_TOKEN, DISCORD_BOT_TOKEN,
SLACK_BOT_TOKEN, SLACK_APP_TOKEN,
OPENCLAW_GATEWAY_TOKEN, OPENCLAW_GATEWAY_PASSWORD
```

来源: `config/io.ts:51-68` (`SHELL_ENV_EXPECTED_KEYS`)

### 零配置启动的最小要求

虽然系统可以零配置启动，但 `assertGatewayAuthConfigured()` 会检查认证。
因为默认 bind 是 loopback (127.0.0.1)，loopback 模式下允许无认证。
如果 bind 到 LAN (0.0.0.0) 则**必须**设置 token 或 password，否则启动会报错。

## ConfigFileSnapshot 数据结构

| 字段 | 含义 |
|------|------|
| `raw` | 原始文件文本 |
| `parsed` | JSON5 解析后对象 |
| `resolved` | $include + ${ENV} 替换后，未加默认值 |
| `config` | 最终配置（加了所有默认值） |
| `valid` | 校验是否通过 |
| `hash` | SHA-256，用于变更检测 |
| `issues` / `warnings` | 校验问题 |
| `legacyIssues` | 旧格式问题 |

## 设计亮点

1. **写入时环境变量还原** — 读取时 `${API_KEY}` 被替换为实际值，写回时通过 `restoreEnvVarRefs` 还原为 `${...}` 引用，避免密钥明文入文件
2. **配置写入审计** — 每次写配置记录 audit log（SHA-256、变更路径、大小、异常检测），写入 `config-audit.jsonl`
3. **原子写入 + 备份轮换** — 先写 tmp → rename（原子），5 份 `.bak` 轮换备份
4. **读写分层** — `resolved`（无默认值）用于 set/unset 操作，`config`（有默认值）用于运行时，防止默认值泄漏到文件
5. **200ms 配置缓存** — `loadConfig()` 有内存 TTL 缓存，减少磁盘 I/O
6. **Shell env fallback** — 没有配置文件时，可从 shell profile 中提取 API key 等关键环境变量
