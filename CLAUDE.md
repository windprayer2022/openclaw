# CLAUDE.md - OpenClaw 源码阅读与学习项目

## 目的

这是一个**源码阅读与学习**项目。目标是深入理解 OpenClaw 的设计与实现，并通过编写实验代码调用其功能来验证和加深理解。这不是 OpenClaw 的开发仓库，我们不向上游贡献代码。

## Claude 的工作方式

- **优先帮助理解代码**：解释设计意图、架构决策、数据流、关键抽象
- **追踪阅读进度**：记录已读模块、待读模块、阅读中发现的问题
- **辅助编写实验代码**：当用户想通过调用来验证理解时，帮助编写调用/测试代码
- **实验代码放在 `playground/` 目录**，与 OpenClaw 源码分离
- **不修改 OpenClaw 源码**（`src/`、`extensions/`、`skills/` 等目录只读）

## OpenClaw 是什么

个人 AI 助手平台（多通道 AI 网关），运行在用户设备上，将 16+ 个消息平台（WhatsApp、Telegram、Slack、Discord 等）的消息路由到 AI Agent 后端。本地优先、单用户、注重隐私。

## 技术栈速览

- TypeScript 5.9（严格模式，ESM）/ Node.js ≥22.12 / pnpm 单仓
- Express 5 + WebSocket（Gateway）/ Lit 3 + Vite（Web UI）
- Vitest 测试 / Oxlint + Oxfmt 代码规范 / tsdown 构建
- SwiftUI (iOS/macOS) / Kotlin (Android) 原生客户端

## 核心架构

```
消息平台 (WhatsApp/Telegram/Slack/...)
    ↓
Channel 插件（src/channels/ + extensions/）
    ↓
Gateway（src/gateway/ — WebSocket 控制面）
  ├─ 配置管理、会话状态、通道路由
  ├─ 媒体处理管道（src/media/）
  └─ 安全与访问控制（src/security/）
    ↓
Agent Runtime — Pi（src/agents/ — RPC 模式）
  ├─ LLM 调用、工具执行、流式输出
  └─ Auth Profile 轮换与故障转移
    ↓
Skills（skills/）+ Tools（src/browser/ 等）
    ↓
响应 → Gateway → Channel → 消息平台
```

## 源码目录导读

| 目录 | 内容 | 阅读优先级建议 |
|------|------|----------------|
| `src/gateway/` | WebSocket 控制面，整个系统的中枢 | ★★★ 最先读 |
| `src/agents/` | Agent 运行时、工具定义、Auth Profile | ★★★ 核心 |
| `src/channels/` | 内置消息通道实现 | ★★☆ 挑一个通道深入 |
| `src/config/` | 配置加载与管理 | ★★☆ 理解系统如何初始化 |
| `src/sessions/` | 会话状态管理 | ★★☆ |
| `src/types/` | TypeScript 类型定义 | ★★☆ 理解数据模型的入口 |
| `src/media/` | 图片/音频/视频处理 | ★☆☆ |
| `src/security/` | 配对、访问控制、审计 | ★☆☆ |
| `src/cli/` + `src/commands/` | CLI 入口与命令 | ★☆☆ |
| `src/web/` + `ui/` | Web UI | ★☆☆ |
| `extensions/` | 38+ 扩展插件 | ★☆☆ 挑感兴趣的看 |
| `skills/` | 53+ 技能（apple-notes, github 等）| ★☆☆ 看几个了解模式 |
| `apps/` | 原生客户端（macOS/iOS/Android）| ☆☆☆ |

## 阅读进度

> 随着阅读推进，在此更新

- [x] Gateway 核心流程（含 chat.ts 聊天推理流程）
- [x] Agent 运行时与 LLM 调用（完整调用链、双层重试、工具组装、流式管道、model fallback、auth profile）
- [x] 消息通道插件机制（Channel 系统全貌：Dock/Plugin 两层、20 个 Adapter、入站/出站流、生命周期）
- [x] Session 管理（Session Key 寻址、Store 持久化、SessionEntry 模型、生命周期 reset/freshness/fork、Transcript JSONL、维护机制）
- [x] 类型系统与数据模型（双 Schema 体系、OpenClawConfig 26 字段、190+ 协议类型、15 Adapter、15 Hook、SessionEntry 40+ 字段）
- [x] System Prompt 构建（6 阶段管线、3 种模式、~18 section、9 个 bootstrap 文件、截断策略、cache 友好设计）
- [x] Compaction 机制（4 种触发、三层摘要 Fallback、safeguard 扩展、Memory Flush 预刷写、溢出恢复决策树）
- [x] Skill 系统（6 来源加载合并、5 维资格判定、Snapshot 缓存、Prompt 注入、环境变量注入、5 种安装器、安全扫描、远程节点扩展、热刷新）
- [x] Sandbox 系统（Docker 容器隔离、3 mode × 3 scope、FS Bridge、Tool Policy、Browser Bridge、Registry/Prune、Media Staging）
- [ ] 配置系统
- [ ] 插件/扩展 SDK

## 构建与运行（供实验用）

```bash
pnpm install          # 安装依赖
pnpm build            # 构建
pnpm dev              # 开发模式运行
pnpm test:fast        # 跑单元测试（验证环境可用）
```

## 阅读工作流

- **实时保存状态**：每次讲解完代码后，自动更新 `notes/reading-state.md`（当前位置、下一步建议、待解答问题）。无需用户手动触发，即使对话突然中断也不会丢失进度。
- **恢复阅读**：用户说"继续读代码"、"恢复阅读"或类似意图时，先读 `notes/reading-state.md` 恢复上下文，汇报当前进度，再提供下一步选项。
- **笔记更新**：模块读完后更新本文件的「阅读进度」章节 + `notes/` 目录下对应模块的笔记文件。
- **状态文件结构**：`notes/reading-state.md` 包含：当前位置、已建立的理解、下一步建议、待解答问题。

## 约定

- 实验代码统一放 `playground/` 目录
- 阅读笔记放在 `notes/` 目录（按模块组织，如 `notes/gateway.md`）
- 阅读状态放在 `notes/reading-state.md`（每次讲解后自动更新）
- 阅读进度总览保留在本文件的「阅读进度」章节
- 对源码有疑问时，优先让 Claude 解释而非猜测
