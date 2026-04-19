---
article_id: OBA-oun491ze
tags: [open-source, meka-flow, architecture-deep-analysis.md, react, rust, electron]
type: learning
updated_at: 2026-03-29
---

# Meka Flow 整体架构深度分析

> 一个在无限画布上同时管理多个 AI 编程会话（Claude Code、Codex）的桌面应用，核心理念是"像管理便签一样管理你的 AI 会话"。

## 一、架构全景

```
┌──────────────────────────────────────────────────────────────────┐
│                       Electron 主进程                             │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │                    main.ts (入口)                              │ │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌─────────────┐ │ │
│  │  │ Sidecar  │  │   PTY    │  │  Island  │  │ Chat Popup  │ │ │
│  │  │ Manager  │  │ (xterm)  │  │ Manager │  │  Manager    │ │ │
│  │  └────┬─────┘  └──────────┘  └──────────┘  └─────────────┘ │ │
│  │       │ stdio JSON-RPC                                       │ │
│  │       ▼                                                       │ │
│  │  ┌──────────────────────────────────────────────────────────┐│ │
│  │  │              ai-backend (Rust Sidecar)                   ││ │
│  │  │  ┌───────────┐ ┌────────┐ ┌───────┐ ┌────────────────┐ ││ │
│  │  │  │ Session   │ │  DB    │ │  Git  │ │  Normalizer    │ ││ │
│  │  │  │ Manager   │ │(SQLite)│ │Watch  │ │  (流标准化)    │ ││ │
│  │  │  ├───────────┤ └────────┘ └───────┘ └────────────────┘ ││ │
│  │  │  │ Claude/Codex 子进程管理                                ││ │
│  │  │  └──────────────────────────────────────────────────────┘│ │
│  └──────────────────────────────────────────────────────────────┘ │
│                          ▲ IPC (preload.ts)                       │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │              Renderer (React 19 + Vite 6)                    │ │
│  │  App.tsx (全局状态中心)                                       │ │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐                     │ │
│  │  │ Canvas   │ │  Board   │ │   Tab    │  三种视图模式       │ │
│  │  │  View    │ │  View    │ │  View    │                     │ │
│  │  └────┬─────┘ └────┬─────┘ └────┬─────┘                     │ │
│  │       └────────────┼────────────┘                           │ │
│  │              SessionWindow (核心组件)                         │ │
│  │  ┌──────────────────────────────────────────────────────┐   │ │
│  │  │ MessageRenderer → ContentBlocksView                  │   │ │
│  │  │ ├ TextBlock  ├ CodeBlock  ├ ToolCallBlock            │   │ │
│  │  │ ├ TodoListBlock  ├ SubagentBlock  ├ SkillBlock       │   │ │
│  │  │ ├ AskUserBlock  ├ FileChangesBlock  ├ FormTableBlock │   │ │
│  │  └──────────────────────────────────────────────────────┘   │ │
│  │  ┌──────────────────────────────────────────────────────┐   │ │
│  │  │ Harness Controller (多 Agent 编排)                    │   │ │
│  │  │ Planner → Generator(s) → Evaluator (循环)            │   │ │
│  │  └──────────────────────────────────────────────────────┘   │ │
│  └──────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────┘
```

## 二、三层架构拆解

### 2.1 Electron 主进程层

**文件位置**：`electron/main.ts`

**职责**：窗口管理、IPC 路由、子进程管理

**核心组件**：

| 组件 | 文件 | 职责 |
|------|------|------|
| SidecarManager | `electron/sidecar.ts` | 管理 Rust 后端子进程，JSON-RPC 通信 |
| IslandManager | `electron/islandManager.ts` | macOS Dynamic Island 浮窗子进程 |
| ChatPopupManager | `electron/chatPopupManager.ts` | 聊天弹窗管理 |
| PTY 管理 | `electron/main.ts` 内联 | node-pty 终端实例 |

**关键设计模式**：

1. **单实例锁**：`app.requestSingleInstanceLock()` 确保只有一个实例运行
2. **Sidecar 自动重启**：崩溃后 1 秒自动重启，通知前端 `sidecar.restarted` 事件
3. **安全退出**：`before-quit` 事件拦截，等待前端 flush 数据（最多 3 秒）
4. **方法超时配置**：不同 RPC 方法有不同超时（Git 操作 30-60 秒，session.send 120 秒）

```typescript
// 超时配置示例
const SLOW_METHOD_TIMEOUTS = {
  'git.changes': 60000,
  'session.send': 120000,
  'git.generate_commit_msg': 120000,
};
```

### 2.2 Rust Sidecar 层

**文件位置**：`ai-backend/src/`

**通信协议**：JSON-RPC over stdio（行分隔 JSON）

**请求/响应格式**：

```json
// 请求（Electron → Rust）
{"id": "req_1", "method": "session.send", "params": {"session_id": "xxx", "text": "hello"}}

// 响应（Rust → Electron）
{"id": "req_1", "result": {"ok": true}}

// 事件（Rust → Electron，无 id）
{"event": "block.start", "data": {"session_id": "xxx", "block_index": 0, "block": {...}}}

// 错误
{"id": "req_1", "error": {"code": 1002, "message": "session_id is required"}}
```

**协议类型**：

| 类型 | 有 id? | 说明 |
|------|--------|------|
| Response | ✅ | 方法调用的返回值 |
| Error | ✅ | 错误响应 |
| Event | ❌ | 流式事件（block.start/delta/stop, message.complete） |

**并发模型**：

```
stdin → 逐行读取 → 解析 JSON → tokio::spawn (每个请求一个 task)
                                        ↓
                                    router::handle_request
                                        ↓
                              stdout → 单线程写入（避免交错）
```

**Router 方法路由表**（`router.rs`，共 40+ 方法）：

| 分类 | 方法 | 说明 |
|------|------|------|
| **会话管理** | session.create/send/list/kill/interrupt/switch_model | AI 会话生命周期 |
| **持久化** | session.save/load/delete/update_messages/update_status/update_position | SQLite 存储 |
| **项目** | project.open/list/update/delete | 项目管理 |
| **Git** | git.check_repo/init/info/changes/diff/stage/unstage/discard/commit/branches/log/file_tree/file_content | 完整 Git 操作 |
| **Worktree** | git.worktrees/create_worktree/merge_worktree/remove_worktree/branch_diff_stats | Git Worktree |
| **文件监控** | git.watch/unwatch | Rust notify crate 实时监控 |
| **AI 提交** | git.generate_commit_msg | 用 AI 生成 commit 消息 |
| **Harness** | harness.save/load/delete | 多 Agent 编排持久化 |
| **设置** | settings.get/set/delete/list | 键值存储 |

### 2.3 React 前端层

**文件位置**：`src/`

**状态管理**：纯 React hooks + prop drilling（无 Redux/Zustand）

**App.tsx 是全局状态中心**，管理：

| 状态 | 类型 | 说明 |
|------|------|------|
| sessions | Session[] | 所有 AI 会话 |
| viewMode | canvas/board/tab | 当前视图 |
| currentProject | DbProject | 当前项目 |
| canvasTransform | {x, y, scale} | 画布变换 |
| projectDir | string | 项目目录 |
| harness groups | HarnessGroup[] | 多 Agent 编排组 |

**自动持久化策略**：

- Sessions：debounce 1 秒后自动保存到 SQLite
- Canvas Transform：debounce 500ms 保存
- View Mode：即时保存
- Harness Groups：debounce 1 秒保存
- Session 删除：对比 `loadedSessionIdsRef` 检测差集，自动删除

**退出前 flush**：

```typescript
// 注册 before-quit 回调
aiBackend.onBeforeQuit(async () => {
  await Promise.all([
    flushSessionSaves(sessions, projectId),
    flushHarnessGroupSaves(groups, projectId),
  ]);
  aiBackend.notifyFlushComplete();
});
```

## 三、核心通信链路

### 3.1 用户发消息 → AI 响应 完整流程

```
[用户在 SessionWindow 输入]
        ↓
[App.tsx setSessions 更新状态]
        ↓
[backend.sendMessage() → window.aiBackend.invoke('session.send')]
        ↓
[Electron preload → ipcRenderer.invoke('sidecar:invoke')]
        ↓
[Electron main → sidecar.invoke() → stdin 写入 JSON]
        ↓
[Rust sidecar → router::handle_request → session_manager.send()]
        ↓
[SessionManager 判断 model 类型]
        ├─ Claude: ClaudeProcess::spawn() → 子进程 claude CLI
        └─ Codex:  CodexProcess::spawn()  → 子进程 codex CLI
        ↓
[AI CLI stdout → 逐行解析 → Normalizer 标准化]
        ↓
[Event channel → stdout → Electron → IPC → Renderer]
        ↓
[SessionWindow 监听 block.start/delta/stop → 渲染 UI]
```

### 3.2 Claude CLI 集成方式

**不是直接调用 API**，而是通过 spawn `claude` CLI 进程：

```rust
// 启动参数
claude -p \
  --output-format stream-json \
  --input-format stream-json \
  --verbose \
  --no-chrome \
  --dangerously-skip-permissions \
  --mcp-config '{"mcpServers":{}}' \
  --strict-mcp-config
```

**关键点**：
- 使用 `--resume <session_id>` 恢复已有会话
- 通过 stdin 写入用户消息（JSON 格式）
- 通过 stdout 读取流式响应（NDJSON 格式）
- SIGINT 中断当前生成（不杀死进程）
- 通过 `--model sonnet/opus` 切换模型

### 3.3 Normalizer（流标准化器）

**职责**：将不同 AI 提供商的原始输出统一为标准化的 block 事件。

```
Claude 原始输出 → ClaudeJson 枚举 → parser.rs → 标准 block 事件
Codex 原始输出  → CodexJson 枚举 → normalizer.rs → 标准 block 事件
```

**Block 类型映射**：

| Claude 工具 | → Block 类型 | 说明 |
|-------------|-------------|------|
| TodoWrite | todolist | 任务清单 |
| Agent | subagent | 子 Agent |
| AskUserQuestion | askuser | 用户交互 |
| Skill | skill | Skill 调用 |
| Bash/Read/Write/Edit | tool_call | 通用工具调用 |
| 文本内容 | text | 纯文本 |

## 四、多 Agent 编排系统（Harness）

### 4.1 架构设计

```
┌─────────────────────────────────────────────┐
│             HarnessGroup                      │
│  connections: [Connection, Connection, ...]   │
│  maxRetries: 3                                │
│  status: idle/running/paused/completed/failed │
│                                               │
│  ┌─────────┐    ┌─────────┐    ┌──────────┐ │
│  │ Planner │───→│Generator│───→│ Evaluator│ │
│  │(规划者) │    │(生成者) │    │(评估者)  │ │
│  └─────────┘    └────┬────┘    └────┬─────┘ │
│                      │              │         │
│                      └──── PASS? ──→┘         │
│                            │                   │
│                          FAIL → 重试循环       │
│                            │                   │
│                          达到 maxRetries →     │
│                            status=failed       │
└─────────────────────────────────────────────┘
```

### 4.2 Pipeline 执行流程

1. **Planner 阶段**：向 planner 会话发送初始 prompt
2. **Generator 阶段**：Planner 完成后，将其输出作为 prompt 发给 Generator(s)
3. **Evaluator 阶段**：所有 Generator 完成后，将结果发给 Evaluator
4. **判定**：
   - `PASS` → `status = completed`
   - `FAIL` → 进入下一轮 revision，重新 dispatch Generator
5. **重试**：达到 `maxRetries` 仍未通过 → `status = failed`

### 4.3 关键实现

```typescript
// advancePipeline 核心逻辑（简化）
const advancePipeline = (groupId, completedSessionId) => {
  if (planners.includes(completedSessionId)) {
    // Planner 完成 → 将输出写入文件 → 派发 Generators
    writeHarnessFile(dir, groupId, sprint, 'plan.md', planContent);
    dispatchGenerators(groupId, generators, generatorPrompt);
  }
  if (generators.includes(completedSessionId)) {
    // Generator 完成 → 写入结果 → 所有 Generator 完成后派发 Evaluators
    writeHarnessFile(dir, groupId, sprint, 'result.md', resultContent);
    if (pendingGenerators.length === 0) {
      dispatchEvaluators(groupId, evaluators, evalPrompt);
    }
  }
  if (evaluators.includes(completedSessionId)) {
    // Evaluator 完成 → 判定 PASS/FAIL
    writeHarnessFile(dir, groupId, sprint, 'review.md', reviewContent);
    if (verdict === 'PASS') status = 'completed';
    else if (round >= maxRetries) status = 'failed';
    else dispatchGenerators(...);  // 重试
  }
};
```

**产物文件结构**：

```
.harness/<groupId>/
└── sprint-1/
    ├── plan.md          # Planner 输出
    ├── result.md        # Generator 第一轮输出
    ├── review-1.md      # Evaluator 第一轮评估
    ├── result-2.md      # Generator 第二轮修改
    └── review-2.md      # Evaluator 第二轮评估
```

## 五、IPC 通信桥接（preload.ts）

### 5.1 暴露的 API 命名空间

`window.aiBackend` 暴露了以下命名空间：

| 命名空间 | 方法 | 说明 |
|----------|------|------|
| **基础** | invoke/on/off/onAll | JSON-RPC 调用和事件监听 |
| **窗口** | toggleMaximize/startWindowDrag/windowDragging | macOS 窗口控制 |
| **文件** | openDirectory/getWorkingDir/getLastProjectDir | 目录操作 |
| **PTY** | ptySpawn/ptyWrite/ptyResize/ptyKill/onPtyData/onPtyExit | 终端 |
| **Island** | island.toggle/getStatus/onStatusChanged | Dynamic Island |
| **Chat Popup** | chatPopup.getSession/close/syncMetadata/syncMessages | 聊天弹窗 |
| **Harness** | harness.writeFile/readFile/mkdir | 多 Agent 文件 I/O |
| **安全 IPC** | ipcOn/ipcOff/ipcSend (仅 chat-popup: 前缀) | 限制性通道 |

### 5.2 安全设计

```typescript
// 限制性 IPC — 只允许 chat-popup: 前缀的通道
ipcOn: (channel, callback) => {
  if (!channel.startsWith('chat-popup:'))
    throw new Error(`ipcOn: channel "${channel}" not allowed`);
  ipcRenderer.on(channel, callback);
},
```

## 六、数据持久化层

### 6.1 SQLite 表结构

| 表 | 文件 | 说明 |
|----|------|------|
| projects | `db/projects.rs` | 项目（id, name, path, view_mode, canvas 变换） |
| sessions | `db/sessions.rs` | 会话（id, project_id, title, model, status, position, messages JSON, git_branch, worktree） |
| harness_groups | `db/harness_groups.rs` | 编排组（id, project_id, name, connections JSON, maxRetries, status, sprint/round） |
| settings | `db/settings.rs` | 设置（key-value 键值对） |

### 6.2 持久化策略

```
前端 React State（内存）
  ↓ debounce 1s
backend.saveSession() → IPC → Rust → SQLite
  ↓
应用退出时 flush 所有待保存数据（before-quit 钩子）
```

## 七、设计亮点

### 7.1 Sidecar 进程隔离

Rust 后端作为独立进程运行，好处：
- **崩溃隔离**：Rust 崩溃不影响 Electron 主进程，自动重启
- **语言优势**：Rust 处理并发、文件监控、CLI 子进程更高效安全
- **独立升级**：可以单独更新 Rust 后端

### 7.2 流标准化器（Normalizer）

将不同 AI 提供商的输出统一为标准化事件：
- Claude 和 Codex 有不同的 JSON 格式
- Normalizer 将两者统一为 `block.start/stop` + `message.complete` 事件
- 前端只需处理一种格式

### 7.3 画布自动布局

`findNextGridPosition()` 算法：
1. 将现有会话按 Y 坐标分组为"行"
2. 尝试在最后一行右侧放置
3. 放不下则在最下方新建一行
4. 兜底放在最右侧 y=0 处

### 7.4 Harness 多 Agent 闭环

Planner → Generator → Evaluator 形成自动评估循环：
- 自动重试机制（最多 maxRetries 次）
- 暂停/恢复支持（deferredCompletion）
- 产物文件持久化到 `.harness/` 目录

### 7.5 Skill 扫描机制

扫描三个来源的 SKILL.md 文件：
1. **项目级**：`<projectDir>/.claude/skills/`
2. **用户级**：`~/.claude/skills/`
3. **插件级**：读取 `installed_plugins.json` 和 `settings.json`，扫描已启用的 Claude 插件的 skills 目录

## 八、可复用设计模式

### 8.1 Sidecar 模式

```
主进程 ←JSON-RPC over stdio→ Sidecar（Rust/Go）
```

**适用场景**：Electron 应用需要高性能后端

**实现要点**：
- 行分隔 JSON（NDJSON）
- 请求用 `id` 匹配响应
- 事件用 `event` 字段标识
- 单线程 stdout 写入避免交错
- 自动重启机制

### 8.2 流标准化器模式

```
Provider A → Normalizer A ──→ 标准化事件
Provider B → Normalizer B ──→ 标准化事件
                    ↓
              统一的 UI 渲染
```

**适用场景**：多提供商集成

### 8.3 Harness 多 Agent 模式

```
Orchestrator 监听 message.complete → 判断角色 → 派发下一步
```

**适用场景**：多 Agent 协作流水线

## 九、技术栈总结

| 维度 | 选择 | 理由 |
|------|------|------|
| 桌面框架 | Electron | 生态成熟、跨平台、Web 技术栈 |
| 前端框架 | React 19 + TypeScript | 组件化、类型安全 |
| 构建工具 | Vite 6 + electron-vite | 快速 HMR、原生 ESM |
| 后端语言 | Rust (Tokio) | 高性能、内存安全、异步运行时 |
| 数据库 | SQLite (rusqlite) | 嵌入式、零配置、ACID |
| 文件监控 | notify crate | 跨平台 fs events |
| 终端 | xterm.js + node-pty | 真实终端体验 |
| 样式 | Tailwind CSS 4 | 原子化 CSS、快速开发 |
| 动画 | Motion (Framer Motion) | React 声明式动画 |
| AI 集成 | CLI 子进程（非 API） | 复用现有 CLI 认证和会话管理 |

## 十、参考资料

- [GitHub 仓库](https://github.com/osvald-go2/meka-flow)
- [Electron 文档](https://www.electronjs.org/docs)
- [Tokio 异步运行时](https://tokio.rs/)
- [notify crate 文件监控](https://docs.rs/notify)
