# Meka Flow 研究项目

> 本项目用于深度研究 [meka-flow](https://github.com/osvald-go2/meka-flow) 开源项目的技术实现。

## 项目简介

Meka Flow 是一个在无限画布上同时管理多个 AI 编程会话（Claude Code、Codex）的桌面应用。

### 核心特性

- **三种视图**：无限画布（Canvas）、看板视图（Board）、标签页视图（Tab）
- **多 AI Agent 支持**：Claude Code + Codex
- **富消息渲染**：流式响应、代码高亮、工具调用可视化
- **Git 集成**：实时文件变更监控、Diff 查看器、Worktree 支持
- **桌面体验**：Electron + Rust sidecar + Dynamic Island

### 技术栈

| 层 | 技术 |
|----|------|
| 前端 | React 19 · TypeScript · Vite 6 · Tailwind CSS 4 |
| 桌面 | Electron · electron-vite · electron-builder |
| 后端 | Rust (Tokio) · SQLite (rusqlite) · JSON-RPC over stdio |
| 动画 | Motion (Framer Motion) |
| 终端 | xterm.js · node-pty |

## 研究课题

研究笔记存放在 `notes/` 目录，按主题组织。

## 目录结构

```
meka-flow-study/
├── meka-flow/          # 源码（从 GitHub 克隆）
├── notes/              # 研究笔记
├── scripts/            # 工具脚本
├── .study-meta.json    # 项目元数据
├── CLAUDE.md           # 本文件
└── RESEARCH-LOG.md     # 研究日志
```
