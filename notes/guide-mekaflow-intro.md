---
article_id: OBA-mekaflow01
tags: [open-source, electron, ai-agent, canvas, guide]
type: tutorial
updated_at: 2026-04-19
---

# Meka Flow 使用指南

> 一个在无限画布上同时管理多个 AI 编程会话的桌面应用——像管理便签一样管理你的 Claude Code 和 Codex 会话。

## 这个工具解决什么问题

如果你同时用 Claude Code 开发多个功能，肯定遇到过这些场景：

- 开了 5 个终端窗口，切来切去找不到哪个是哪个
- 想让多个 AI Agent 协作（一个写代码、一个 Review），但没有统一的编排界面
- 想同时看多个会话的输出，屏幕空间根本不够用

**Meka Flow** 就是来解决这些问题的。它把每个 AI 会话变成一张可以拖拽的「便签卡片」，放在一个无限大的画布上。你可以自由排列、缩放、分组，甚至用一个看板来追踪每个任务的进度。

## 快速上手

### 1. 克隆项目

```bash
git clone https://github.com/osvald-go2/meka-flow.git
cd meka-flow
```

### 2. 安装依赖

需要 Node.js >= 18 和 Rust 工具链：

```bash
# 安装前端依赖
npm install

# 安装 Rust（如果没有）
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

### 3. 启动开发模式

```bash
# 构建 Rust 后端 + 启动 Electron
npm run dev:electron
```

启动后你会看到三个 AI 会话卡片摆在画布上，可以自由拖拽和缩放。

### 4. 如果只想看前端（不需要 Rust）

```bash
npm run dev
# 访问 http://localhost:3000
```

> [!note] Web 模式下 Rust 后端功能（会话持久化、Git 集成）不可用，仅适合预览界面。

## 核心概念

### 1. Session（会话）

每个 Session 就是一个独立的 AI 对话窗口，绑定一个 Claude Code 或 Codex CLI 子进程。后台通过 stdin/stdout 以 JSON 流格式通信。

```
Session = AI 对话窗口 + CLI 子进程 + Git 分支（可选）
```

### 2. 三种视图

| 视图 | 用途 | 适合场景 |
|------|------|---------|
| **Canvas（画布）** | 自由拖拽、缩放、批量广播 | 同时看多个会话 |
| **Board（看板）** | inbox → in process → review → done | 追踪任务进度 |
| **Tab（标签页）** | 传统标签页 + 搜索栏 | 专注单个会话 |

### 3. Harness（多 Agent 编排）

这是 Meka Flow 的核心亮点。你可以把多个 Session 组成一个「Harness 组」，定义角色关系：

- **Planner**：分析需求，拆分任务
- **Generator**：根据任务写代码
- **Evaluator**：Review 生成结果，决定是否通过

三者形成闭环循环，直到任务通过或达到最大重试次数。

### 4. 流标准化器（Normalizer）

Claude Code 和 Codex 的输出格式完全不同。Normalizer 是一个 Rust 组件，把两种格式统一成标准 block 事件，前端只需要写一套渲染逻辑。

```
Claude 输出 ──┐
              ├──→ Normalizer ──→ 统一的 Block 事件 ──→ React 渲染
Codex 输出 ──┘
```

## 架构一览

整个应用分三层：

```
┌─────────────────────────────────────┐
│        React 前端（Renderer）         │
│  画布 / 看板 / 标签页 + 消息渲染       │
└──────────────┬──────────────────────┘
               │ Electron IPC
┌──────────────▼──────────────────────┐
│        Electron 主进程               │
│  窗口管理 / PTY 终端 / IPC 桥接       │
└──────────────┬──────────────────────┘
               │ stdio JSON-RPC
┌──────────────▼──────────────────────┐
│        Rust Sidecar（ai-backend）     │
│  AI 子进程管理 / SQLite / Git 监控    │
└─────────────────────────────────────┘
```

**关键设计**：前端不直接调用 AI API，而是由 Rust 后端 spawn CLI 子进程。这样 AI 会话的完整状态都在本地管理，支持断线恢复。

## 适合谁用

| 角色 | 场景 |
|------|------|
| **AI 重度用户** | 同时跑 3 个以上 Claude Code 会话，需要统一管理 |
| **多 Agent 开发者** | 想尝试 Planner-Generator-Evaluator 协作模式 |
| **Electron/Rust 学习者** | 想学习 Electron + Rust sidecar 的架构模式 |
| **开源贡献者** | 项目 MIT 协议，架构清晰，适合贡献代码 |

## 进阶阅读

以下是已有的深度研究笔记，按主题分类：

**架构与设计**
- [architecture-deep-analysis.md](./architecture-deep-analysis.md) — 三层架构全景，IPC 安全设计，数据持久化策略

**AI 集成**
- [ai-cli-integration.md](./ai-cli-integration.md) — Claude Code / Codex CLI 的 spawn 参数、流式 JSON 解析、会话恢复
- [harness-multi-agent-orchestration.md](./harness-multi-agent-orchestration.md) — 多 Agent 编排系统的数据结构与闭环循环

**前端实现**
- [canvas-and-views.md](./canvas-and-views.md) — CSS Transform 无限画布实现、三种视图模式切换、自动网格布局

**工程实践**
- [git-integration-and-worktree.md](./git-integration-and-worktree.md) — Rust 后端 Git 命令封装、文件变更监控、Worktree 集成
