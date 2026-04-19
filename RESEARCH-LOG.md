# 研究日志

## 2026-03-29: 整体架构深度分析

**研究主题**: 整体架构深度分析

**研究问题**: Meka Flow 的整体架构是如何设计的？

**仓库**: [meka-flow](https://github.com/osvald-go2/meka-flow)

**核心发现**:
- 三层架构：Electron 主进程（窗口/IPC/PTY）→ Rust Sidecar（JSON-RPC/AI会话/SQLite）→ React 前端（无限画布/多会话/Harness 编排）
- AI 集成方式：不直接调用 API，而是 spawn claude/codex CLI 子进程，通过 stdin/stdout 通信
- 流标准化器（Normalizer）：将 Claude 和 Codex 两种不同的输出格式统一为标准 block 事件
- Harness 多 Agent 编排：Planner → Generator → Evaluator 闭环循环
- 数据持久化：前端 debounce 1s + 退出前 flush + SQLite 嵌入式存储
- IPC 安全设计：preload.ts 限制性通道（仅 chat-popup: 前缀）

**进度（持续更新）**:
- questions: 1
- notes: 1
- guides: 0
- skill templates: 0
- runnable skills: 0

---
