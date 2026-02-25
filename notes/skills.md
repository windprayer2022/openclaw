# Skill 系统

> 阅读日期: 2026-02-25

## 概述

Skill 是 OpenClaw 的"知识注入"机制——本质上是一组 **SKILL.md 文件**，每个文件包含 YAML frontmatter（元数据）+ Markdown body（LLM 可读的说明书）。Skill 不是代码插件——它们不执行代码，而是**告诉 Agent 如何使用外部工具/CLI**。

核心思想：通过 System Prompt 注入技能说明，让 LLM"学会"使用 `gh`、`memo`、`curl` 等 CLI 工具。

## 核心文件清单

| 文件 | 职责 |
|------|------|
| `src/agents/skills/types.ts` | 全部类型定义 |
| `src/agents/skills/frontmatter.ts` | SKILL.md frontmatter 解析 |
| `src/agents/skills/workspace.ts` | 核心：加载、过滤、合并、Prompt 构建、Snapshot |
| `src/agents/skills/config.ts` | 资格判定（shouldIncludeSkill）、配置查询 |
| `src/agents/skills/env-overrides.ts` | 运行时环境变量注入 |
| `src/agents/skills/refresh.ts` | 文件监控 + 版本号 bump |
| `src/agents/skills/plugin-skills.ts` | 插件贡献的技能目录 |
| `src/agents/skills/bundled-dir.ts` | 内置技能目录发现 |
| `src/agents/skills/serialize.ts` | 串行化执行（sync 用） |
| `src/agents/skills-install.ts` | 依赖安装（brew/node/go/uv/download） |
| `src/agents/skills-status.ts` | 技能状态报告 |
| `src/auto-reply/skill-commands.ts` | 聊天中的 /skill 命令 |
| `src/gateway/server-methods/skills.ts` | Gateway RPC（status/install/update/bins） |
| `src/infra/skills-remote.ts` | 远程节点技能资格（探测远程 macOS 的 binary） |
| `src/security/skill-scanner.ts` | 安全扫描（检测危险代码模式） |
| `src/cli/skills-cli.ts` | CLI `openclaw skills list/info/check` |

## SKILL.md 文件格式

```markdown
---
name: apple-notes
description: Manage Apple Notes via the `memo` CLI on macOS
homepage: https://github.com/antoniorodr/memo
user-invocable: true           # 用户可通过 /apple-notes 调用（默认 true）
disable-model-invocation: false # 不注入 system prompt（默认 false）
command-dispatch: tool          # 可选：触发确定性工具调用
command-tool: bash              # dispatch 目标工具名
metadata:
  {
    "openclaw": {
      "emoji": "📝",
      "always": false,          # true = 无论资格检查，总是加载
      "skillKey": "apple-notes", # 配置中的 key（默认 = name）
      "primaryEnv": "API_KEY",   # apiKey 注入的环境变量名
      "os": ["darwin"],          # 限定平台
      "requires": {
        "bins": ["memo"],        # 必须全部存在
        "anyBins": ["x", "y"],   # 任一存在即可
        "env": ["API_KEY"],      # 必须有环境变量
        "config": ["browser.enabled"]  # 必须配置项为真
      },
      "install": [
        {
          "id": "brew",
          "kind": "brew",        # brew | node | go | uv | download
          "formula": "antoniorodr/memo/memo",
          "bins": ["memo"],
          "label": "Install memo via Homebrew"
        }
      ]
    }
  }
---

# Apple Notes CLI
Use `memo notes` to manage Apple Notes...
（Markdown body = LLM 可读的技能说明书）
```

## 数据模型

### SkillEntry（内存中的技能记录）

```typescript
type SkillEntry = {
  skill: Skill;                    // pi-coding-agent SDK 的 Skill 对象（name, description, filePath, baseDir, source）
  frontmatter: ParsedSkillFrontmatter;  // 解析后的 YAML frontmatter（key-value）
  metadata?: OpenClawSkillMetadata;     // frontmatter.metadata.openclaw 解析结果
  invocation?: SkillInvocationPolicy;   // userInvocable + disableModelInvocation
};
```

### SkillSnapshot（快照，存入 SessionEntry）

```typescript
type SkillSnapshot = {
  prompt: string;           // 格式化后的技能 prompt（注入 system prompt）
  skills: Array<{ name: string; primaryEnv?: string }>;  // 技能列表
  resolvedSkills?: Skill[]; // 完整 Skill 对象（可选，用于环境变量注入）
  version?: number;         // 快照版本号
};
```

## 6 来源加载与合并

`loadSkillEntries()` 从 6 个目录加载技能，**按优先级从低到高**合并（同名覆盖）：

```
1. extra          — config.skills.load.extraDirs + 插件贡献目录  [最低]
2. bundled        — 内置技能（skills/ 目录，source: "openclaw-bundled"）
3. managed        — ~/.config/openclaw/skills/（clawhub 安装的）
4. agents-personal — ~/.agents/skills/（个人级 agents 标准目录）
5. agents-project — <workspace>/.agents/skills/（项目级 agents 标准目录）
6. workspace      — <workspace>/skills/（工作区级）                [最高]
```

每个 Skill 从其 SKILL.md 读取 frontmatter，解析 `metadata.openclaw` 得到 `OpenClawSkillMetadata`，解析 `user-invocable` / `disable-model-invocation` 得到 `SkillInvocationPolicy`。

内置技能目录发现（`bundled-dir.ts`）：
1. `OPENCLAW_BUNDLED_SKILLS_DIR` 环境变量
2. 可执行文件旁的 `skills/` 目录（bun --compile 场景）
3. 从 package root 向上搜索 `skills/` 目录（npm/dev 场景）

## 资格判定（shouldIncludeSkill）

5 维检查，**全部通过**才会 include：

```
1. enabled ≠ false     — config.skills.entries[skillKey].enabled
2. bundled allowlist    — config.skills.allowBundled（空 = 全允许）
3. OS 匹配             — metadata.os（空 = 全平台）+ 远程节点平台
4. always = true        → 直接通过（跳过 bins/env/config 检查）
5. 依赖检查：
   a. bins: 全部必须在 PATH（或远程节点有）
   b. anyBins: 至少一个在 PATH（或远程节点有）
   c. env: 全部必须有环境变量 OR config 中有 skillConfig.env[name] OR (primaryEnv + apiKey)
   d. config: 全部配置路径必须为真值
```

**环境变量三级来源**：process.env > config.skills.entries[key].env > config.skills.entries[key].apiKey（→ primaryEnv）

## Prompt 注入流程

```
buildWorkspaceSkillSnapshot()
  ├─ loadSkillEntries()          → 6 来源加载合并
  ├─ filterSkillEntries()        → shouldIncludeSkill 过滤 + skillFilter 名单
  ├─ 过滤 disableModelInvocation → 不注入 prompt 的技能仍在 skills 列表中
  ├─ formatSkillsForPrompt()     → pi-coding-agent SDK 方法，格式化 Markdown
  └─ 返回 SkillSnapshot { prompt, skills, resolvedSkills, version }

→ snapshot.prompt 注入 system prompt 的 skills section（非 minimal 模式）
→ snapshot 存入 SessionEntry，跨 turn 缓存
→ 通过 version 号判断是否需要刷新（文件变更 / 远程节点变更 / 手动 bump）
```

## 环境变量注入

`applySkillEnvOverrides()` / `applySkillEnvOverridesFromSnapshot()`

在 Agent 运行前（attempt.ts / compact.ts）调用：
1. 遍历 eligible skills
2. 从 config.skills.entries[key].env 注入缺失的环境变量
3. 从 config.skills.entries[key].apiKey 注入到 primaryEnv
4. 返回 restore 函数（运行结束后恢复原值）

**关键设计**：不覆盖已有环境变量（`!process.env[envKey]` 检查）。

## 文件监控与版本管理（refresh.ts）

- `ensureSkillsWatcher()`: 使用 **chokidar** 监控所有技能目录
  - 监控 add/change/unlink 事件
  - debounce 250ms（可配置）
  - 忽略 .git/node_modules/dist/venv 等
- `bumpSkillsSnapshotVersion()`: 3 种触发原因
  - `watch`: 文件变更
  - `manual`: 手动（如安装后）
  - `remote-node`: 远程节点连接/bin 变更
- 版本号 = `Math.max(Date.now(), current + 1)`
- `getSkillsSnapshotVersion(workspaceDir)`: `max(globalVersion, workspaceVersion)`
- 通过 listener 机制通知订阅者

## 技能命令（/skill）

### 注册流程

`buildWorkspaceSkillCommandSpecs()`:
1. 加载 + 过滤技能
2. 过滤 `userInvocable !== false` 的技能
3. 技能名 → 命令名（sanitize: 小写 + 下划线，最长 32 字符）
4. 避免与内置命令/其他技能名冲突（dedup: 追加 `_2` `_3`...）
5. 可选 dispatch: `command-dispatch: tool` → 直接调用指定工具

### 调用解析

`resolveSkillCommandInvocation()`:
- `/skill <name> [args]` → 查找技能命令
- `/<name> [args]` → 直接匹配技能命令名
- 支持模糊匹配（lowercase + normalize）

## 技能安装（skills-install.ts）

5 种安装器：

| kind | 命令 |
|------|------|
| `brew` | `brew install <formula>` |
| `node` | `npm/pnpm/yarn/bun install -g --ignore-scripts <package>` |
| `go` | `go install <module>` |
| `uv` | `uv tool install <package>` |
| `download` | HTTP 下载 + 可选解压（tar.gz/tar.bz2/zip） |

**自动依赖安装**：若 `uv`/`go` 不在 PATH 但有 `brew`，会先 `brew install uv/go`。

安装前会运行 **安全扫描**（`scanDirectoryWithSummary`），对技能目录中的 JS/TS 文件检测：
- critical: child_process exec/spawn、eval/new Function、crypto mining、env+网络（credential harvesting）
- warn: 文件读取+网络发送（exfiltration）、hex/base64 混淆、非标准端口 WebSocket

## Gateway RPC（4 个方法）

| 方法 | 功能 |
|------|------|
| `skills.status` | 返回完整技能状态报告（资格、缺失依赖、安装选项） |
| `skills.bins` | 收集所有技能需要的 binary 列表 |
| `skills.install` | 执行技能安装 |
| `skills.update` | 更新技能配置（enabled/apiKey/env），写入 config 文件 |

## 远程节点技能资格（skills-remote.ts）

当 macOS 远程节点连接时：
1. 从节点元数据缓存已知 bin 列表
2. 通过 `system.which` / `system.run`（shell 脚本探测）检查远程有哪些 binary
3. 将远程 bin 纳入资格判定（`eligibility.remote.hasBin()`）
4. 若 bin 集合变更，`bumpSkillsSnapshotVersion({ reason: "remote-node" })`
5. 非 macOS 节点或不支持 system.run 的节点被忽略

**用途**：本机是 Linux 但连了远程 macOS，仍然能启用 macOS-only 技能（如 apple-notes），Agent 通过 `nodes.run` 在远程执行。

## Sandbox Sync

`syncSkillsToWorkspace()`: 将所有技能目录复制到 sandbox workspace 的 `skills/` 目录（Docker 沙箱场景），使用 serialize 保证串行。

## CLI 子命令

```
openclaw skills              # 默认 list
openclaw skills list         # 列表（--json / --eligible / --verbose）
openclaw skills info <name>  # 单个技能详情
openclaw skills check        # 汇总检查（ready / disabled / blocked / missing）
```

## 设计亮点

1. **纯 Prompt 注入**：技能不执行代码，只是教 LLM 如何用工具。安全且易扩展。
2. **6 来源合并**：从内置到工作区，层层覆盖，用户可以 fork 内置技能。
3. **Snapshot 缓存**：技能 prompt 在 session 级别缓存，通过版本号判断是否刷新。
4. **远程节点扩展**：macOS 技能可以在远程节点执行，突破本机平台限制。
5. **安全扫描**：安装前扫描 JS/TS 代码，检测危险模式。
6. **环境变量隔离**：注入前检查不覆盖，运行后 restore，无副作用。
7. **热刷新**：chokidar 监控技能目录，文件变更自动 bump 版本号。
