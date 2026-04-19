---
article_id: OBA-s2lhf7pp
tags: [open-source, meka-flow, git-integration-and-worktree.md, react, rust, ai]
type: learning
updated_at: 2026-03-29
---

# Meka Flow Git 集成与 Worktree 管理深度分析

> 从 Rust 后端的 Git 命令执行、notify crate 文件监控，到 React 前端的 GitProvider 状态管理和 Diff 面板可视化。

## 一、Git 命令层（Rust）

### 1.1 辅助函数

**文件位置**: `ai-backend/src/git/commands.rs:11-31`

```rust
// 不检查退出码的低层执行
pub(crate) fn run_git_unchecked(dir: &str, args: &[&str]) -> Result<Output, String>

// 检查退出码的高层执行
pub(crate) fn run_git(dir: &str, args: &[&str]) -> Result<String, String>
```

### 1.2 Git 操作清单

**文件位置**: `ai-backend/src/git/commands.rs` + `ai-backend/src/router.rs`

| 方法 | Rust 函数 | Git 命令 | 说明 |
|------|-----------|---------|------|
| git.check_repo | `check_repo` | `git rev-parse --is-inside-work-tree` | 是否 Git 仓库 |
| git.init | `init` | `git init` | 初始化仓库 |
| git.info | `git_info` | `rev-parse + log + rev-list` | 分支/提交/上游差异 |
| git.changes | `git_changes` | `status --porcelain + diff --numstat` | 变更文件列表 |
| git.diff | `git_diff` | `diff / diff --cached / diff --no-index` | 单文件 Diff |
| git.stage_file | `stage_file` | `git add <file>` | 暂存文件 |
| git.unstage_file | `unstage_file` | `git reset HEAD <file>` | 取消暂存 |
| git.discard_file | `discard_file` | `git checkout -- <file>` | 丢弃修改 |
| git.commit | `commit` | `git commit -m <msg>` | 提交 |
| git.branches | `branches` | `git branch -a --format=...` | 分支列表 |
| git.log | `log` | `git log --format=... --name-status` | 提交历史 |
| git.file_tree | `file_tree` | `git ls-files` | 跟踪文件列表 |
| git.file_content | `file_content` | `git show <ref>:<path>` 或直接读 | 文件内容 |

### 1.3 git_info 实现

**文件位置**: `ai-backend/src/git/commands.rs:167-202`

```rust
pub fn git_info(dir: &str) -> Result<GitInfo, String> {
    // 1. 当前分支
    let branch = run_git(dir, &["rev-parse", "--abbrev-ref", "HEAD"])?;

    // 2. 最后提交
    let last_commit = run_git(dir, &["log", "-1", "--format=%h %s"])?;

    // 3. 上游差异（ahead/behind）
    let ahead_behind = run_git(dir, &[
        "rev-list", "--left-right", "--count", "HEAD...@{u}"
    ]);
    // 返回 (ahead, behind) 计数
}
```

### 1.4 git_changes 批量优化

**文件位置**: `ai-backend/src/git/commands.rs:205-263`

```
步骤 1: git status --porcelain -u  → 所有文件状态 (M/A/D/?? 等)
步骤 2: git diff --numstat         → 批量获取 unstaged 增删行数
步骤 3: git diff --cached --numstat → 批量获取 staged 增删行数
步骤 4: 未跟踪文件 → 直接统计行数作为 additions
```

**性能优化**：不逐文件执行 `git diff --stat`，而是批量 `--numstat` 后解析。

### 1.5 git_diff 三级回退

**文件位置**: `ai-backend/src/git/commands.rs:266-285`

```rust
pub fn git_diff(dir: &str, file: &str) -> Result<DiffOutput, String> {
    // 1. Unstaged diff
    if let Ok(out) = run_git(dir, &["diff", "--", file]) { return Ok(out); }
    // 2. Staged diff
    if let Ok(out) = run_git(dir, &["diff", "--cached", "--", file]) { return Ok(out); }
    // 3. Untracked file → 与 /dev/null 对比
    run_git(dir, &["diff", "--no-index", "/dev/null", file])
}
```

---

## 二、文件监控（notify crate）

### 2.1 GitWatcherManager 架构

**文件位置**: `ai-backend/src/git/watcher.rs:40-49`

```rust
pub struct GitWatcherManager {
    watchers: Arc<Mutex<HashMap<String, WatcherEntry>>>,
    // key = 目录路径, value = watcher 实例 + 去抖状态
}
```

### 2.2 事件分类

**文件位置**: `ai-backend/src/git/watcher.rs:10-34`

```rust
fn classify_event(path: &Path, repo_root: &Path) -> Option<&'static str> {
    // .git/HEAD → "head"
    // .git/refs/ → "refs"
    // 其他 → "files"
}
```

### 2.3 Watch 生命周期

```
watch(dir, event_tx)
  ↓
1. 检查是否已监控（防重复）
2. 创建去抖状态 HashMap<String, Instant>
3. 启动 notify::RecommendedWatcher
4. 监控范围：
   - 仓库根目录（工作树文件变更）
   - .git/ 目录（HEAD/refs 变更）
5. 事件处理：
   - classify_event 分类
   - 500ms 去抖窗口
   - 发送 git.changed 事件
```

### 2.4 去抖策略

**文件位置**: `ai-backend/src/git/watcher.rs:66-100`

```rust
let debounce_state: Arc<Mutex<HashMap<String, Instant>>> = ...;
let debounce_ms = Duration::from_millis(500);

// 每个事件类型独立去抖
for event in events {
    let kind = classify_event(&event.path, &repo_root);
    let key = format!("{}/{}", dir, kind);

    let mut state = debounce_state.lock().unwrap();
    let last_fire = state.get(&key).copied().unwrap_or(Instant::now() - debounce_ms);

    if last_fire.elapsed() >= debounce_ms {
        state.insert(key.clone(), Instant::now());
        // 发送事件
        event_tx.send(Event::new("git.changed", json!({
            "dir": dir, "kind": kind
        })));
    }
    // 否则忽略（在去抖窗口内）
}
```

### 2.5 前端响应

```typescript
// 前端 GitProvider 监听
window.aiBackend.on('git.changed', (data) => {
  const { dir, kind } = data;
  const kinds = new Set<string>();
  kinds.add(kind);

  const flushPending = () => {
    if (kinds.has('files') || kinds.has('head')) refreshChanges();
    if (kinds.has('refs')) refreshBranches();
    if (kinds.has('head')) { refreshInfo(); refreshLog(); }
  };

  // 前端再 debounce 200ms
  setTimeout(flushPending, 200);
});
```

---

## 三、Worktree 管理

### 3.1 创建流程

**文件位置**: `ai-backend/src/git/worktree.rs:157-272`

```
create_worktree(project_dir, branch, base)
  ↓
1. 检查分支是否已存在
   ├─ 存在 → 检查是否已有 Worktree 关联
   │         ├─ 有 → 直接返回已有路径
   │         └─ 无 → 创建新 Worktree
   └─ 不存在 → 创建新分支 + Worktree
  ↓
2. 计算 Worktree 路径：.meka-flow/worktrees/<safe_branch>
  ↓
3. 执行创建
   ├─ 分支存在：git worktree add <path> <branch>
   └─ 新分支：  git worktree add -b <branch> <path> <base>
  ↓
4. 写入元数据：.meka-flow-meta.json
  ↓
5. 添加到 .gitignore（如果不存在）
```

### 3.2 与 AI 会话的绑定

```
Session.worktree = "/path/to/.meka-flow/worktrees/feature-x"
                    ↓
GitProvider overrideDir 参数
                    ↓
影响 Git 面板显示的当前分支、变更列表
                    ↓
SessionWindow 终端 cwd 使用 worktree 路径
```

**App.tsx 中的绑定逻辑**（`src/App.tsx:172-175`）：
```typescript
const activeSession = dedupedSessions.find(s => s.id === activeSessionId);
const terminalCwd = (activeSession?.worktree && activeSession.worktree !== 'default')
  ? activeSession.worktree
  : projectDir ?? '';
```

### 3.3 合并流程

**文件位置**: `ai-backend/src/git/worktree.rs:276-330`

```
merge_worktree(project_dir, wt_path, target_branch?)
  ↓
1. 获取 Worktree 分支名
2. 切换到目标分支（默认 base branch）
3. 执行 git merge <worktree_branch>
   ├─ 成功 → 返回合并结果
   └─ 冲突 → 自动 git merge --abort + 返回错误
```

### 3.4 清理流程

**文件位置**: `ai-backend/src/git/worktree.rs:333-363`

```
remove_worktree(project_dir, wt_path, branch)
  ↓
1. git worktree remove <path> --force
2. git branch -D <branch>
```

### 3.5 Diff 统计

**文件位置**: `ai-backend/src/git/worktree.rs:366-449`

```rust
pub fn branch_diff_stats(dir: &str, base_branch: Option<&str>) -> Result<BranchDiffStats, String> {
    // 四维统计：
    // 1. 已提交差异：git diff base...HEAD --numstat
    // 2. 未暂存更改：git diff --numstat
    // 3. 已暂存更改：git diff --cached --numstat
    // 4. 未跟踪文件：按行数统计

    // 汇总为 { total_additions, total_deletions, files_changed: [...] }
}
```

---

## 四、前端 Git 集成

### 4.1 GitProvider Context

**文件位置**: `src/contexts/GitProvider.tsx`

**状态接口**（行号：53-62）：
```typescript
interface GitState {
  isRepo: boolean;
  info: GitInfo;              // 分支、最后提交、ahead/behind
  changes: FileChange[];      // 变更文件列表
  branches: BranchInfo[];     // 分支列表
  worktrees: WorktreeInfo[];  // Worktree 列表
  log: CommitInfo[];          // 提交历史
  fileTree: TreeNode[];       // 文件树
  loading: boolean;
}
```

**Worktree 绑定**（行号：145-150）：
```typescript
// overrideDir 影响当前活跃 Worktree 的判断
const activeWt = overrideDir || projectDir;
const adjusted = worktrees.map(wt => ({
  ...wt,
  is_current: activeWt ? wt.path === activeWt : wt.is_current,
}));
```

**文件树构建**（行号：18-51）：
```typescript
function buildFileTree(paths: string[]): TreeNode[]
// 扁平路径 → 嵌套树形结构
// "src/components/App.tsx" → { name: "src", children: [{ name: "components", children: [...] }] }
```

### 4.2 GitPanel Tab 系统

**文件位置**: `src/components/git/GitPanel.tsx`

| Tab | 组件 | 功能 |
|-----|------|------|
| Changes | `ChangesTab` | 变更文件列表、暂存/取消暂存/丢弃 |
| Git | `GitTab` | 提交历史（CommitGraph）、分支管理 |
| Files | `FilesTab` | 文件浏览器、文件内容查看 |

**面板特性**：
- 可拖拽调整宽度（280-800px，默认 420px）
- 左边缘拖拽手柄
- 变更文件计数徽章

### 4.3 DiffView 实现

**文件位置**: `src/components/git/DiffView.tsx`

**颜色方案**（行号：28-82）：

| 行类型 | 背景 | 文字 | 行号颜色 |
|--------|------|------|---------|
| 添加 (+) | `bg-emerald-500/[0.08]` | `text-emerald-300` | `text-emerald-400/50` |
| 删除 (-) | `bg-red-500/[0.10]` | `text-red-300` | `text-red-400/50` |
| 上下文 | `bg-transparent` | `text-zinc-300` | `text-zinc-600` |

**语法高亮**：`src/utils/syntaxHighlight.ts` 使用 highlight.js 按文件扩展名选择语言。

### 4.4 CommitGraph 可视化

**文件位置**: `src/components/git/CommitGraph.tsx`

```
● abc1234 feat: add harness system                    2h ago
│  by: jacky
│  M src/services/harnessController.ts
│  A src/components/harness/ConnectionLine.tsx
│
● def5678 fix: canvas zoom center calculation         5h ago
│  by: jacky
│  M src/components/CanvasView.tsx
│
● ghi9012 initial commit                              1d ago
   by: jacky
```

- 垂直连线：`w-px flex-1 bg-white/15`
- 节点：蓝色小圆点
- 变更文件状态标签（M/A/D/R）

### 4.5 MergeDialog

**文件位置**: `src/components/git/MergeDialog.tsx`

```
┌──────────────────────────────────────────┐
│ Merge Worktree Branch                     │
│                                          │
│ Source: feature-harness                   │
│ Target: [main ▾]                          │
│                                          │
│ Diff Stats:                              │
│ +245 -32 across 12 files                 │
│                                          │
│ [Cancel]              [Merge]            │
└──────────────────────────────────────────┘
```

### 4.6 DiscardWorktreeDialog

**文件位置**: `src/components/git/DiscardWorktreeDialog.tsx`

- 显示 Worktree 分支名和路径
- 确认后调用 `remove_worktree` + 删除 Session

---

## 五、端到端流程

### 5.1 文件变更全链路

```
用户在 AI 会话中编辑文件
  ↓
notify crate 检测到文件系统变更
  ↓
classify_event → "files"
  ↓
500ms 去抖 → 发送 git.changed 事件
  ↓
Rust stdout → Electron → IPC → GitProvider
  ↓
前端再 debounce 200ms
  ↓
refreshChanges() → backend.gitChanges(dir)
  ↓
Rust: git status --porcelain + git diff --numstat
  ↓
返回 FileChange[] → 更新 React state
  ↓
ChangesTab 重新渲染
```

### 5.2 Worktree 会话绑定全链路

```
创建新会话 → 选择 "New Branch"
  ↓
NewSessionModal 调用 backend.createWorktree(dir, branch, base)
  ↓
Rust: git worktree add → 返回路径
  ↓
Session.worktree = "/path/to/.meka-flow/worktrees/feature-x"
  ↓
AI CLI 进程使用 worktree 路径作为 working_dir
  ↓
文件变更隔离在 worktree 中
  ↓
GitProvider 使用 worktree 路径显示变更
  ↓
完成后通过 MergeDialog 合并到主分支
```

---

## 六、设计亮点

### 6.1 双层去抖

```
Rust 层: notify 事件 → 500ms 去抖 → git.changed 事件
前端层:  git.changed 事件 → 200ms 去抖 → API 调用
```

避免频繁刷新 Git 状态，同时保持实时性。

### 6.2 Worktree 隔离

每个 AI 会话可以独立工作在自己的 Worktree 分支上：
- 不影响主分支代码
- 支持并行开发多个功能
- 完成后通过 Merge 合并

### 6.3 三级 Diff 回退

unstaged → staged → untracked 自动回退，确保任何文件状态都能显示 Diff。

### 6.4 .meka-flow 目录管理

```
.meka-flow/
└── worktrees/
    ├── feature-a/     # Worktree 目录
    ├── feature-b/
    └── ...
```

自动添加到 `.gitignore`，不影响仓库状态。
