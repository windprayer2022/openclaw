# System Prompt 构建

> 源码位置：`src/agents/system-prompt.ts`（主组装）、`system-prompt-params.ts`（运行时参数）、`pi-embedded-runner/system-prompt.ts`（嵌入式包装）、`bootstrap-files.ts` + `workspace.ts`（工作区文件加载）、`system-prompt-report.ts`（诊断报告）

## 总体架构

System Prompt 通过一条 6 阶段管线构建：

```
① 运行时参数解析 (system-prompt-params.ts)
   ↓  timezone, repoRoot, timeFormat
② 工作区文件加载 (workspace.ts + bootstrap-files.ts)
   ↓  9 个 bootstrap 文件 → hook 覆盖 → 截断
③ Skills Prompt 解析 (skills/workspace.ts)
   ↓  SkillEntry[] → XML <available_skills> 块
④ 工具摘要构建 (tool-summaries.ts)
   ↓  AgentTool[] → name→summary 映射
⑤ System Prompt 组装 (system-prompt.ts)
   ↓  buildAgentSystemPrompt() 拼接 ~18 个 section
⑥ 诊断报告 (system-prompt-report.ts)
   ↓  chars 统计、截断检测、工具/技能明细
```

入口：`runEmbeddedAttempt()`（`pi-embedded-runner/run/attempt.ts`）中调用 `buildEmbeddedSystemPrompt()` → 内部委托 `buildAgentSystemPrompt()`。

## 三种 Prompt 模式

| 模式 | 用途 | 内容 |
|------|------|------|
| `"full"` | 主 Agent 会话 | 全部 ~18 个 section |
| `"minimal"` | 子 Agent（subagent） | 仅 Tooling、Safety、Workspace、Runtime；跳过 Skills/Memory/Docs/Identity/Reply/Messaging/Voice/Silent/Heartbeat/Self-Update/Model Aliases |
| `"none"` | 最简 | 仅 `"You are a personal assistant running inside OpenClaw."` |

判定逻辑：`isSubagentSessionKey(sessionKey) ? "minimal" : "full"`

## System Prompt 完整 Section 列表（full 模式）

按输出顺序：

```
1. 身份声明         "You are a personal assistant running inside OpenClaw."
2. ## Tooling        可用工具列表 + 摘要（22 个核心 + 外部工具）
3. ## Tool Call Style 不要叙述常规工具调用
4. ## Safety         Anthropic constitution 风格的安全护栏
5. ## OpenClaw CLI   gateway 子命令速查
6. ## Skills         扫描 <available_skills>，选一个读 SKILL.md
7. ## Memory Recall  先 memory_search 再回答（需 memory_search 工具可用）
8. ## OpenClaw Self-Update  config.apply / update.run（需 gateway 工具）
9. ## Model Aliases  别名映射（如 Opus → anthropic/claude-opus-4-5）
10. 日期/时间提示     "用 session_status 获取当前时间"（仅有 timezone 时）
11. ## Workspace     工作目录 + workspaceNotes
12. ## Documentation 本地 docs 路径 + 在线链接
13. ## Sandbox       Docker 沙箱信息（sandbox 启用时）
14. ## User Identity Owner 手机号
15. ## Current Date & Time  仅时区（不含实际时间！）
16. ## Workspace Files (injected)  Project Context 标头
17. ## Reply Tags    [[reply_to_current]] 等回复标签
18. ## Messaging     message 工具用法、跨通道路由、inline buttons
19. ## Voice (TTS)   语音提示
20. ## Group Chat / Subagent Context  extraSystemPrompt（条件标题）
21. ## Reactions     Telegram/Signal 反应表情指导
22. ## Reasoning Format  <think>/<final> 标签（reasoning-tag provider）
23. # Project Context 注入的工作区文件（AGENTS.md, SOUL.md 等）
24. ## Silent Replies NO_REPLY 令牌协议
25. ## Heartbeats    HEARTBEAT_OK 协议
26. ## Runtime       key=value 运行时信息行
```

## 工作区 Bootstrap 文件系统

### 9 个文件

从 `~/.openclaw/workspace/` 加载：

| 文件 | 用途 |
|------|------|
| `AGENTS.md` | Agent 配置/指令 |
| `SOUL.md` | 人格与语气（有特殊处理） |
| `TOOLS.md` | 用户自定义工具指南 |
| `IDENTITY.md` | 身份信息 |
| `USER.md` | 用户偏好 |
| `HEARTBEAT.md` | 心跳配置 |
| `BOOTSTRAP.md` | 引导指令 |
| `MEMORY.md` | 记忆文件 |
| `memory.md` | 记忆文件（备选名） |

**子 Agent 过滤**：`SUBAGENT_BOOTSTRAP_ALLOWLIST` 仅保留 `AGENTS.md` + `TOOLS.md`。

### 加载管线

```
loadWorkspaceBootstrapFiles(dir)     ← 从磁盘读取 9 个文件
    ↓
filterBootstrapFilesForSession()     ← 子 Agent 仅保留 2 个
    ↓
applyBootstrapHookOverrides()        ← 触发 agent:bootstrap 内部钩子（插件可修改）
    ↓
buildBootstrapContextFiles()         ← 转为 {path, content} 对 + 截断处理
```

### 截断策略

- 默认上限：**20,000 字符/文件**（可通过 `agents.defaults.bootstrapMaxChars` 配置）
- 超限处理：**head 70% + 截断标记 + tail 20%**
- 缺失文件 → `[MISSING] Expected at: <path>`
- 空/纯空白 → 跳过

## 工具摘要系统

22 个核心工具有硬编码摘要：

```
read, write, edit, apply_patch, grep, find, ls,
exec, process, web_search, web_fetch, browser, canvas,
nodes, cron, message, gateway, agents_list,
sessions_list, sessions_history, sessions_send, session_status, image
```

- 外部工具摘要来自 `tool.description || tool.label`
- 渲染顺序：先按 `toolOrder` 固定顺序，额外工具按字母排序追加
- 工具名 **保留调用方大小写**（如 `Read` 而非 `read`），但去重按小写

## Skills Prompt

```
resolveSkillsPromptForRun()
  ├─ 有 snapshot.prompt? → 直接用
  └─ 有 entries? → buildWorkspaceSkillsPrompt()
       ├─ loadSkillEntries() → filterSkillEntries()
       ├─ 排除 disableModelInvocation
       └─ formatSkillsForPrompt() → XML <available_skills> 块
```

System Prompt 中的 Skills section 指导 Agent：
1. 扫描 `<available_skills>` 的 `<description>` 条目
2. 恰好一个匹配 → 读其 SKILL.md → 遵循
3. 多个匹配 → 选最具体的
4. 无匹配 → 不读任何 SKILL.md

## 关键设计决策

### 1. 不在 System Prompt 中包含实际时间

只包含时区（如 `Time zone: America/Chicago`），不包含当前日期/时间。原因：**prompt cache 稳定性**。如果每次请求时间不同，system prompt 就变了，缓存失效。Agent 通过 `session_status` 工具获取当前时间。

### 2. SOUL.md 人格系统

如果注入的 context files 中有 `SOUL.md`，会添加特殊指令：
> "If SOUL.md is present, embody its persona and tone. Avoid stiff, generic replies."

### 3. 安全段（Safety）始终包含

即使 `"minimal"` 模式也保留 Safety section：
- 无自主目标（self-preservation, replication, power-seeking）
- 优先安全与人类监督
- 冲突时暂停询问
- 不操纵他人、不修改自身 system prompt

### 4. 推理标签提示

对支持 reasoning tag 的 provider（如某些模型），注入 `<think>/<final>` 格式要求：
- 所有内部推理必须在 `<think>` 内
- 用户可见回复在 `<final>` 内
- 只有 `<final>` 内容展示给用户

### 5. 条件 Section 策略

大量 section 根据运行时状态条件包含：
- Skills → 有 skillsPrompt 才包含
- Memory → 有 memory_search/memory_get 工具才包含
- Self-Update → 有 gateway 工具才包含
- Sandbox → sandbox 启用才包含
- Reactions → 有 reactionGuidance 才包含
- Model Aliases → 有 modelAliasLines 才包含

## 诊断报告

`buildSystemPromptReport()` 生成诊断结构，用于 `/status` 命令展示：

```typescript
{
  systemPrompt: { chars, projectContextChars, nonProjectContextChars }
  injectedWorkspaceFiles: [{ name, path, missing, rawChars, injectedChars, truncated }]
  skills: { promptChars, entries: [{ name, blockChars }] }
  tools: { listChars, schemaChars, entries: [{ name, summaryChars, schemaChars, propertiesCount }] }
}
```

## Runtime 信息行

最后一行格式：
```
Runtime: agent=work | host=macbook | repo=/code | os=Darwin (arm64) | node=v22 | model=anthropic/claude-opus-4-5 | default_model=... | shell=zsh | channel=telegram | capabilities=inlineButtons | thinking=off
```

## 文件索引

| 文件 | 职责 |
|------|------|
| `system-prompt.ts` | 核心组装函数 `buildAgentSystemPrompt()` + 各 section builder |
| `system-prompt-params.ts` | `buildSystemPromptParams()` — 解析 timezone/repoRoot/timeFormat |
| `pi-embedded-runner/system-prompt.ts` | `buildEmbeddedSystemPrompt()` — 薄包装层，提取 toolNames/toolSummaries |
| `tool-summaries.ts` | `buildToolSummaryMap()` — 从 AgentTool[] 提取 name→description |
| `workspace.ts` | `loadWorkspaceBootstrapFiles()` — 加载 9 个工作区文件 |
| `bootstrap-files.ts` | `resolveBootstrapContextForRun()` — 完整 bootstrap 管线 |
| `bootstrap-hooks.ts` | `applyBootstrapHookOverrides()` — agent:bootstrap 钩子 |
| `pi-embedded-helpers/bootstrap.ts` | `buildBootstrapContextFiles()` — 截断 + 上下文构建 |
| `skills/workspace.ts` | `resolveSkillsPromptForRun()` / `buildWorkspaceSkillsPrompt()` |
| `system-prompt-report.ts` | `buildSystemPromptReport()` — 诊断报告 |
