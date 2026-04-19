---
article_id: OBA-lwayxfpb
tags: [open-source, meka-flow, ai-cli-integration.md, rust, chrome-extension, mcp]
type: learning
updated_at: 2026-03-29
---

# Meka Flow AI CLI 集成深度分析

> 深入分析 Claude Code 和 Codex CLI 的集成方式：spawn、通信、流解析、中断恢复的完整实现。

## 一、Claude CLI 集成

### 1.1 Spawn 参数详解

**文件位置**: `ai-backend/src/claude/client.rs:21-39`

```rust
// 核心命令参数
cmd.args([
    "-p",                                  // 命令模式（非交互式）
    "--output-format", "stream-json",      // 输出格式：流式 JSON (NDJSON)
    "--input-format", "stream-json",       // 输入格式：流式 JSON
    "--verbose",                           // 详细输出（包含工具调用信息）
    "--no-chrome",                        // 禁用 Chrome 界面
    "--dangerously-skip-permissions",      // 跳过权限检查（自动化模式）
    "--mcp-config", r#"{"mcpServers":{}}"", // MCP 服务器配置（空）
    "--strict-mcp-config",                 // 严格 MCP 配置
]);

// 可选参数
if let Some(m) = model {
    cmd.args(["--model", m]);  // 如 "sonnet", "opus"
}
if let Some(sid) = resume_session_id {
    cmd.args(["--resume", sid]);  // 恢复已有会话
}
```

**关键环境变量处理**（`client.rs:42-43`）：
```rust
cmd.env_remove("CLAUDECODE");            // 防止嵌套 Claude 会话检测
cmd.env_remove("CLAUDE_CODE_ENTRYPOINT"); // 防止入口点冲突
```

### 1.2 Stdin 输入格式

**文件位置**: `ai-backend/src/claude/client.rs:105-126`

```json
{
  "type": "user",
  "message": {
    "role": "user",
    "content": "请帮我分析这个项目"
  }
}
```

每条用户消息作为一行 JSON 通过 stdin 发送给 Claude CLI。

### 1.3 Stdout 输出格式（NDJSON）

**文件位置**: `ai-backend/src/claude/types.rs:9-55`

每行一个 JSON 对象，`type` 字段区分消息类型：

| type | 说明 | 关键字段 |
|------|------|---------|
| `system` (subtype=init) | 系统初始化 | session_id, model, tools[] |
| `assistant` | AI 响应 | message.content[] (text/tool_use) |
| `user` | 用户消息回显 | message.content[] (tool_result) |
| `result` | 会话完成 | is_error, duration_ms, num_turns, usage |

**Assistant 消息的 ContentBlock 类型**：

```rust
// 文本块
ContentBlock::Text { text: String }

// 思考块（跳过）
ContentBlock::Thinking { thinking: String }

// 工具调用
ContentBlock::ToolUse { id: String, name: String, input: Value }

// 工具结果
ContentBlock::ToolResult { is_error: Option<bool>, ... }
```

### 1.4 会话恢复机制

**文件位置**: `ai-backend/src/session/manager.rs:192-241`

```
第一次启动 → Claude 生成 session_id → 保存到 ActiveSession
                                          ↓
下次启动 → 读取 claude_session_id → --resume 参数传递 → 继续上次会话
```

**共享内存槽实现**：
```rust
let claude_sid_slot = Arc::new(Mutex::new(None::<String>));
// Normalizer 解析到 session.init 事件时写入 slot
// 后台任务每 100ms 轮询 slot，写入 ActiveSession
tokio::spawn(async move {
    for _ in 0..50 {
        tokio::time::sleep(Duration::from_millis(100)).await;
        if let Some(csid) = slot.lock().unwrap().clone() {
            sessions_arc.lock().unwrap().get_mut(&sid)?.claude_session_id = Some(csid);
            break;
        }
    }
});
```

### 1.5 中断机制

**文件位置**: `ai-backend/src/claude/client.rs:130-138`

```rust
pub fn interrupt(&self) -> Result<(), String> {
    let pid = self.cached_pid.ok_or("process already exited")?;
    unsafe { libc::kill(pid as i32, libc::SIGINT) };  // 发送 SIGINT
    Ok(())
}
```

- 使用 `SIGINT`（不是 SIGTERM），优雅停止当前生成
- 不终止整个进程，保留会话状态
- 用户可以继续发送新消息

---

## 二、Codex CLI 集成

### 2.1 Spawn 参数详解

**文件位置**: `ai-backend/src/codex/client.rs:23-36`

```rust
// 模型参数（必须放在 exec 之前）
if let Some(m) = model {
    cmd.args(["-m", m]);  // 如 "gpt-5.4", "gpt-5.4-mini"
}

// 新建会话 vs 恢复会话
if let Some(thread_id) = resume_thread_id {
    cmd.args(["exec", "resume", thread_id, prompt, "--json", "--full-auto"]);
} else {
    cmd.args(["exec", "--json", "--full-auto", prompt]);
}
```

### 2.2 与 Claude 的核心差异

| 维度 | Claude CLI | Codex CLI |
|------|------------|-----------|
| **通信方式** | stdin/stdout 双向流 | 命令行参数 + stdout 单向 |
| **输入格式** | 流式 JSON (stdin) | 命令行直接传 prompt |
| **会话恢复** | `--resume <session_id>` | `exec resume <thread_id> <prompt>` |
| **输出格式** | 多种消息类型 | item 生命周期事件 |
| **模型切换** | 同类型内可切换 | 仅 Codex 内可切换 |
| **stdin** | `Stdio::piped()` | `Stdio::null()` |

### 2.3 Thread ID 恢复机制

**文件位置**: `ai-backend/src/codex/normalizer.rs:43-55`

```rust
// ThreadStarted 事件捕获
CodexEvent::ThreadStarted { thread_id } => {
    if let Some(ref tid) = thread_id {
        *codex_tid_slot.lock().unwrap() = Some(tid.clone());
    }
}
```

恢复流程与 Claude 类似：生成 → 共享内存槽 → 轮询写入 → 下次恢复使用。

---

## 三、流标准化器（Normalizer）

### 3.1 标准化架构

```
Claude 原始 NDJSON → parser.rs → 标准 block 事件
Codex 原始 JSONL   → normalizer.rs → 标准 block 事件
                                ↓
                    统一的 block.start/delta/stop 事件
                                ↓
                         前端只需处理一种格式
```

### 3.2 Block 事件类型映射

**文件位置**: `ai-backend/src/normalizer/parser.rs:164-258`

| Claude 工具 | → Block 类型 | 说明 |
|-------------|-------------|------|
| 文本内容 | `text` | 纯文本 |
| TodoWrite | `todolist` | 任务清单（含状态映射） |
| Agent | `subagent` | 子 Agent（launched 状态） |
| AskUserQuestion | `askuser` | 用户交互（含选项） |
| Skill | `skill` | Skill 调用 |
| Bash/Read/Write/Edit | `tool_call` | 通用工具调用 |
| Glob/Grep | `tool_call` | 搜索工具 |

### 3.3 Block 生命周期

```
block.start → [block.delta]* → block.stop
```

**事件格式**：

```json
// block.start — 创建新块
{"event": "block.start", "data": {"session_id": "xxx", "block_index": 0, "block": {...}}}

// block.delta — 增量更新（流式文本追加）
{"event": "block.delta", "data": {"session_id": "xxx", "block_index": 0, "delta": "..."}}

// block.stop — 标记完成
{"event": "block.stop", "data": {"session_id": "xxx", "block_index": 0, "status": "done"}}
```

### 3.4 工具调用的特殊处理

- **TodoWrite**: `completed` → `done` 状态映射，支持 `todos`/`items`/`tasks` 多种字段名兼容
- **AskUser**: 支持新格式（questions 数组 + options）和旧格式（单 question 字段）
- **Tool Result**: 通过 `block_index - 1` 回溯更新上一个 tool_call 的状态

---

## 四、进程生命周期管理

### 4.1 创建和清理

```
SessionManager.send() → 判断 model 类型
  ├─ Claude: ClaudeProcess::spawn() → Arc<ClaudeProcess>
  └─ Codex:  CodexProcess::spawn()  → Arc<CodexProcess>
       ↓
  Normalizer tokio::task 解析输出
       ↓
  完成后清理进程引用
```

**清理方式**：`kill()` 移除 HashMap 条目 → Arc 引用计数归零 → 进程 Drop → 自动终止

### 4.2 崩溃检测

**文件位置**: `ai-backend/src/codex/normalizer.rs:196-216`

```rust
// Codex 崩溃检测：检查是否收到 turn.completed
if !turn_completed {
    // 收集 stderr 输出
    let detail = stderr_lines.join("\n");
    // 发送错误事件给前端
    event_tx.send(Event::new("message.error", json!({
        "session_id": session_id,
        "error": detail,
        "agent": "codex",
    })));
}
```

### 4.3 Sidecar 超时机制

**文件位置**: `electron/sidecar.ts:93-113`

```typescript
// 默认 15 秒超时，部分方法有特殊超时
const SLOW_METHOD_TIMEOUTS = {
  'git.changes': 60000,
  'session.send': 120000,
  'git.generate_commit_msg': 120000,
};

// 超时后清理 pending 请求
const timer = setTimeout(() => {
    this.pendingRequests.delete(id);
    reject(new Error(`sidecar invoke '${method}' timed out`));
}, timeoutMs);
```

### 4.4 Sidecar 自动重启

**文件位置**: `electron/sidecar.ts:56-67`

```typescript
// 进程退出事件
childProcess.on('close', (code) => {
    // 1. 清理所有 pending 请求
    this.pendingRequests.forEach(({ reject }) => reject(new Error('sidecar crashed')));
    // 2. 发出 crashed 事件
    mainWindow.webContents.send('sidecar.crashed', { code });
    // 3. 自动重启
    setTimeout(() => this.start(), 1000);
});
```

---

## 五、模型管理

### 5.1 模型 CLI 标志映射

**文件位置**: `ai-backend/src/session/manager.rs:10-18`

| 模型 ID | CLI 标志 | 提供商 |
|---------|---------|--------|
| claude-sonnet-4-6 | sonnet | Claude |
| claude-opus-4-6 | opus | Claude |
| codex-gpt-5-4 | gpt-5.4 | Codex |
| codex-gpt-5-4-mini | gpt-5.4-mini | Codex |

### 5.2 模型切换限制

```rust
// 跨提供商切换不支持
let old_is_codex = is_codex_model(&active.info.model);
let new_is_codex = is_codex_model(&new_model);
if old_is_codex != new_is_codex {
    return Err("cross-provider model switch not supported");
}
// 同提供商切换：清空进程引用，下次 send 时重新 spawn
active.claude_process = None;
active.info.model = new_model;
```

---

## 六、关键设计洞察

### 6.1 不直接调 API，而是 CLI 子进程

**为什么不用 Anthropic SDK？**
- 复用 `claude` CLI 的认证体系（OAuth/API Key）
- 复用 CLI 的会话管理（session_id, conversation history）
- 复用 CLI 的工具执行环境（Bash, Read, Write, Edit 等）
- 自动获取 CLI 的新功能（模型更新、新工具等）

### 6.2 共享内存槽模式

```
Normalizer task 写入 → Arc<Mutex<Option<String>>> → 后台轮询 task 读取 → ActiveSession
```

这个模式解决了 tokio task 与共享状态之间的异步通信问题，避免了复杂的 channel 同步。

### 6.3 SIGINT 而非 SIGTERM

使用 SIGINT 的好处：
- Claude CLI 的 signal handler 可以优雅停止当前生成
- 不会丢失已生成的内容
- 进程继续存活，可以接收新的用户消息
