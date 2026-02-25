# Sandbox 系统

> Docker 容器隔离层，让非主会话的 Agent 在受限环境中执行代码和文件操作。

## 核心文件

| 文件 | 职责 |
|------|------|
| `src/agents/sandbox/types.ts` | SandboxContext、SandboxConfig、SandboxToolPolicy 等类型 |
| `src/agents/sandbox/types.docker.ts` | SandboxDockerConfig（22 个 Docker 参数） |
| `src/config/types.sandbox.ts` | 用户配置侧的 Settings 类型（全部可选） |
| `src/agents/sandbox/constants.ts` | 默认镜像、端口、容器前缀、默认 tool allow/deny |
| `src/agents/sandbox/config.ts` | 配置解析入口：resolveSandboxConfigForAgent() |
| `src/agents/sandbox/context.ts` | 运行时入口：resolveSandboxContext() |
| `src/agents/sandbox/runtime-status.ts` | 运行时判定：resolveSandboxRuntimeStatus() |
| `src/agents/sandbox/shared.ts` | scopeKey 解析、slugify、workspace 路径 |
| `src/agents/sandbox/docker.ts` | Docker 容器生命周期：create/start/exec |
| `src/agents/sandbox/fs-bridge.ts` | 文件系统代理：通过 docker exec 操作容器内文件 |
| `src/agents/sandbox/tool-policy.ts` | 工具 allow/deny 过滤 |
| `src/agents/sandbox/browser.ts` | 沙箱浏览器容器：CDP + VNC |
| `src/agents/sandbox/browser-bridges.ts` | Browser Bridge 缓存 Map |
| `src/agents/sandbox/registry.ts` | 双注册表：containers.json + browsers.json |
| `src/agents/sandbox/prune.ts` | 自动清理（idle/age 策略） |
| `src/agents/sandbox/manage.ts` | CLI 管理操作：list/remove |
| `src/agents/sandbox/config-hash.ts` | 配置变更检测（SHA1 指纹） |
| `src/agents/sandbox/workspace.ts` | Workspace 初始化：seed bootstrap 文件 |
| `src/agents/sandbox-paths.ts` | 路径安全：逃逸检测 + symlink 检测 |
| `src/auto-reply/reply/stage-sandbox-media.ts` | 入站媒体暂存到沙箱 workspace |
| `src/commands/sandbox.ts` | CLI 命令：sandbox list/recreate |

## 1. 运行时判定

### mode — 谁被沙箱化

```
mode = "off"      → 所有会话直接运行（默认）
mode = "non-main" → 非主会话沙箱化，主会话直接运行
mode = "all"      → 所有会话都沙箱化
```

判定核心：`shouldSandboxSession()` 比较 sessionKey 与 agent 的 mainSessionKey。

### scope — 容器粒度

```
scope = "session" → scopeKey = sessionKey          每会话独享
scope = "agent"   → scopeKey = "agent:<agentId>"   同 Agent 共享
scope = "shared"  → scopeKey = "shared"            全局共享
```

### workspaceAccess — 文件权限

```
"none" → 沙箱使用独立 workspace（与 agent workspace 隔离）
"ro"   → 只读挂载 agent workspace
"rw"   → 直接使用 agent workspace（最大权限）
```

## 2. 配置解析（双层合并）

`resolveSandboxConfigForAgent(cfg, agentId)`:

```
agents.defaults.sandbox  (全局默认)
        ↓ 被 agent 级覆盖
agents.list[agentId].sandbox (Agent 级)
```

子配置合并规则：
- 大多数字段：agent ?? global ?? 内置默认
- `env`：global + agent 合并
- `ulimits`：global + agent 合并
- `binds`：global + agent 拼接

SandboxConfig 最终结构（8 个字段）：
- mode: "off" | "non-main" | "all"
- scope: "session" | "agent" | "shared"
- workspaceAccess: "none" | "ro" | "rw"
- workspaceRoot: 沙箱 workspace 根目录
- docker: SandboxDockerConfig（22 个 Docker 参数）
- browser: SandboxBrowserConfig（12 个浏览器参数）
- tools: SandboxToolPolicy（allow/deny 列表）
- prune: SandboxPruneConfig（idle/age 策略）

## 3. Docker 容器生命周期

### 容器命名

```
containerName = prefix + slugify(scopeKey)  (截断到 63 字符)
slugify = safeChars[:32] + "-" + sha1[:8]
```

### ensureSandboxContainer() 流程

```
① 计算 configHash (docker 配置 + workspace 路径的 SHA1)
② 检查容器是否存在 (docker inspect)
   ├─ 存在 + hash 匹配 → 确保 running
   ├─ 存在 + hash 不匹配 + 热容器(5min内) → 提示 recreate
   ├─ 存在 + hash 不匹配 + 冷容器 → docker rm -f → 重建
   └─ 不存在 → 创建
③ createSandboxContainer():
   - ensureDockerImage() (默认 pull debian:bookworm-slim 并 tag)
   - docker create (安全加固参数 + workspace 挂载)
   - docker start
   - docker exec setupCommand (可选)
④ updateRegistry()
```

### Docker 安全加固

```
--read-only                        只读根文件系统
--tmpfs /tmp,/var/tmp,/run         临时写入区
--network none                     无网络
--cap-drop ALL                     丢弃所有 capabilities
--security-opt no-new-privileges   禁止提权
--pids-limit / --memory / --cpus   资源限制
--seccomp / --apparmor             可选安全 profile
```

### Workspace 挂载

```
主挂载: sandboxWorkspaceDir → /workspace
辅挂载: agentWorkspaceDir → /agent (当 workspaceAccess ≠ none)
启动命令: sleep infinity
```

## 4. FS Bridge — 文件系统代理

所有文件操作通过 `docker exec -i <container> sh -c <script>` 执行：

| 方法 | 容器内命令 |
|------|-----------|
| readFile | `cat -- "$1"` |
| writeFile | `mkdir -p dir && cat >"$1"` (stdin 传数据) |
| mkdirp | `mkdir -p -- "$1"` |
| remove | `rm [-rf] -- "$1"` |
| rename | `mkdir -p dir && mv -- "$1" "$2"` |
| stat | `stat -c "%F\|%s\|%Y" -- "$1"` |

### 路径映射

```
host:      /data/sandboxes/session-abc/file.txt
container: /workspace/file.txt
```

`resolveSandboxFsPath()`: host path → relative path → container path

### 安全保护 (sandbox-paths.ts)

- 路径逃逸检测：relative 以 `..` 开头或为绝对路径 → 抛错
- 符号链接检测：逐层 lstat() 检查 → 发现 symlink 拒绝
- 写操作前检查 workspaceAccess ≠ "ro"

## 5. Tool Policy — 工具过滤

### 默认策略

```
allow: exec, process, read, write, edit, apply_patch, image, sessions_*
deny:  browser, canvas, nodes, cron, gateway, <所有 channel ID>
```

### 优先级

agent 级 > global 级 > 内置默认

### 匹配逻辑

1. deny 命中 → 拒绝（deny 优先）
2. allow 为空 → 全部允许
3. allow 不为空 → 必须在 allow 列表中

支持 `*` 通配符。`image` 工具总是被自动加入 allow（除非在 deny 中）。

### 集成

Tool policy 作为 pi-tools.ts 8 层工具过滤链的第 5 层。

## 6. 沙箱化工具替换 (pi-tools.ts)

sandbox 启用时：

| 原始工具 | 替换为 |
|---------|--------|
| read | createSandboxedReadTool (通过 FS Bridge) |
| write | createSandboxedWriteTool (通过 FS Bridge) |
| edit | createSandboxedEditTool (通过 FS Bridge) |
| exec | createExecTool({sandbox: containerName + workdir}) |
| apply_patch | createApplyPatchTool({sandbox: root + bridge}) |
| browser | 指向 sandbox browser bridgeUrl |

当 workspaceAccess="ro" 时，write/edit/apply_patch 不可用。

## 7. Sandbox Browser — 容器化浏览器

`ensureSandboxBrowser()`:

- 独立容器：`openclaw-sandbox-browser:bookworm-slim`
- 网络：bridge（需加载页面）
- 端口随机映射：CDP / VNC / noVNC
- 等待 CDP 可达：轮询 /json/version，最长 12s
- 启动 Browser Bridge Server（HTTP 代理层）
- BROWSER_BRIDGES Map 按 scopeKey 缓存复用
- noVNC Web 访问：可视化查看浏览器操作

## 8. Registry & Prune

### 双注册表

- `~/.openclaw/sandbox/containers.json`：沙箱容器
- `~/.openclaw/sandbox/browsers.json`：浏览器容器

每个 entry：containerName, sessionKey, createdAtMs, lastUsedAtMs, image, configHash

### Prune 策略

- 节流：每 5 分钟最多一次
- idle > 24h 或 age > 7d → docker rm -f + 移除注册
- 浏览器清理额外停止 browser bridge server
- 在 resolveSandboxContext() 时自动调用

## 9. Media Staging

`stageSandboxMedia()`:

入站媒体（通道发来的附件）处理：
1. 确保沙箱 workspace 存在
2. 文件复制到 `workspace/media/inbound/`（处理重名）
3. 改写 ctx.MediaPath 为沙箱内相对路径
4. 远程节点媒体通过 scp 拉取

安全限制：本地路径必须在 media 目录内。

## 10. 核心流程总览

```
resolveSandboxContext(config, sessionKey, workspaceDir)
  ├─ resolveSandboxRuntimeStatus()     ← 判定是否沙箱化
  │    ├─ resolveSessionAgentId()
  │    ├─ resolveSandboxConfigForAgent()
  │    └─ shouldSandboxSession(mode, sessionKey, mainSessionKey)
  ├─ resolveSandboxConfigForAgent()    ← 合并配置
  ├─ maybePruneSandboxes()             ← 自动清理
  ├─ ensureSandboxWorkspace()          ← 初始化 workspace
  │    ├─ seed 7 个 bootstrap 文件
  │    ├─ ensureAgentWorkspace()
  │    └─ syncSkillsToWorkspace()
  ├─ ensureSandboxContainer()          ← Docker 容器
  │    ├─ computeSandboxConfigHash()
  │    ├─ dockerContainerState()
  │    ├─ createSandboxContainer()
  │    └─ updateRegistry()
  ├─ ensureSandboxBrowser()            ← 浏览器容器
  │    ├─ buildSandboxCreateArgs()
  │    ├─ waitForSandboxCdp()
  │    ├─ startBrowserBridgeServer()
  │    └─ updateBrowserRegistry()
  └─ createSandboxFsBridge()           ← 文件系统代理
       └─ SandboxFsBridgeImpl (docker exec)
```

## 设计亮点

1. **configHash 热/冷判定**：避免重建正在使用的容器，同时自动清理过时容器
2. **FS Bridge 纯 shell 代理**：不需要容器内安装任何额外软件，只要有 `sh` 就能工作
3. **路径安全双重保护**：逃逸检测 + symlink 检测，防止容器内路径操纵
4. **Tool Policy 层叠**：与 Agent Runtime 的 8 层工具过滤链无缝集成
5. **Browser 隔离**：浏览器在独立容器中运行，通过 CDP 远程控制
6. **Prune 自动清理**：不需要 cron job，在每次使用时顺便清理
