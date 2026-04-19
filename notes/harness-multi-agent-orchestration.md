---
article_id: OBA-bdqwrvqk
tags: [open-source, meka-flow, harness-multi-agent-orchestration.md, typescript, ai-agent, ai]
type: learning
updated_at: 2026-03-29
---

# Meka Flow Harness 多 Agent 编排系统深度分析

> 通过 Planner → Generator → Evaluator 闭环循环实现多 AI Agent 协作开发的编排系统。

## 一、概念模型

### 1.1 核心数据结构

**HarnessGroup**（`src/types.ts:84-93`）：
```typescript
export interface HarnessGroup {
  id: string;                      // 组唯一标识符
  name: string;                    // 组名称
  connections: HarnessConnection[]; // 角色连接关系
  maxRetries: number;              // 最大重试次数（默认 3）
  status: HarnessGroupStatus;      // idle | running | paused | completed | failed
  currentSprint: number;           // 当前 Sprint 序号
  currentRound: number;            // 当前 Round 序号
  harnessDir: string;              // 文件产物存储目录
}
```

**HarnessConnection**（`src/types.ts:76-82`）：
```typescript
export interface HarnessConnection {
  id: string;
  fromSessionId: string;     // 源会话 ID
  toSessionId: string;       // 目标会话 ID
  fromRole: HarnessRole;     // planner | generator | evaluator
  toRole: HarnessRole;
}
```

**HarnessRunState**（内部状态）：
```typescript
export interface HarnessRunState {
  pendingGenerators: string[];              // 待完成的 Generator 列表
  pendingStep: 'generator' | 'evaluator' | null;
  deferredCompletion?: string | null;       // 暂停时存储的完成会话 ID
}
```

### 1.2 角色职责

| 角色 | 标识 | 职责 | 触发时机 |
|------|------|------|---------|
| **Planner** | P (蓝色) | 制定实现计划 | Sprint 启动时 |
| **Generator** | G (绿色) | 根据计划实现功能 | Planner 完成后 |
| **Evaluator** | E (橙色) | 评估实现质量 | Generator(s) 完成后 |

### 1.3 状态机

```
idle → running → {paused → running} → completed
                                    → failed
```

## 二、Pipeline 执行引擎

### 2.1 核心执行流程

**文件位置**: `src/services/harnessController.ts:225-341`

```
[用户点击启动]
      ↓
startPipeline() → currentSprint++, currentRound=0, status=running
      ↓
[用户向 Planner 发送初始 prompt]
      ↓
Planner 完成 → advancePipeline 检测到 Planner 完成
      ↓
写入 plan.md → 构建 Generator prompt → dispatchGenerators
      ↓
Generator(s) 完成 → advancePipeline 检测到
      ↓
写入 result.md → 所有 Generator 完成 → dispatchEvaluators
      ↓
Evaluator 完成 → advancePipeline 检测到
      ↓
写入 review-1.md → parseVerdict 判定
      ├─ PASS → status=completed ✓
      ├─ FAIL + round < maxRetries → 构建 revision prompt → 重新 dispatchGenerators
      └─ FAIL + round >= maxRetries → status=failed ✗
```

### 2.2 advancePipeline 核心逻辑

**文件位置**: `src/services/harnessController.ts:225-341`

```typescript
const advancePipeline = useCallback(async (groupId, completedSessionId) => {
  const group = groupsRef.current.find(g => g.id === groupId);
  if (!group || !dir) return;

  // 暂停处理：存储 deferredCompletion，等待恢复
  if (group.status === 'paused') {
    setRunState(groupId, { deferredCompletion: completedSessionId });
    return;
  }

  const planners = getSessionsByRole(group, 'planner');
  const generators = getSessionsByRole(group, 'generator');
  const evaluators = getSessionsByRole(group, 'evaluator');

  // ── Planner 完成分支 ──
  if (planners.includes(completedSessionId)) {
    const planContent = getLastAssistantText(completedSessionId);
    await writeHarnessFile(dir, group.id, group.currentSprint, 'plan.md', planContent);
    const prompt = buildGeneratorPrompt(group.currentSprint, planContent);
    await dispatchGenerators(groupId, group, generators, prompt);
  }

  // ── Generator 完成分支 ──
  if (generators.includes(completedSessionId)) {
    const resultContent = getLastAssistantText(completedSessionId);
    const filename = group.currentRound > 0
      ? `result-${group.currentRound + 1}.md` : 'result.md';
    await writeHarnessFile(dir, group.id, group.currentSprint, filename, resultContent);

    // 检查是否还有待处理的 Generator
    const pending = runState.pendingGenerators.filter(id => id !== completedSessionId);
    setRunState(groupId, { pendingGenerators: pending });
    if (pending.length > 0) return;

    // 所有 Generator 完成 → 派发 Evaluators
    const evalPrompt = buildEvaluatorPrompt(sprint, round, planContent, resultContent);
    await dispatchEvaluators(groupId, evaluators, evalPrompt);
  }

  // ── Evaluator 完成分支 ──
  if (evaluators.includes(completedSessionId)) {
    const reviewContent = getLastAssistantText(completedSessionId);
    await writeHarnessFile(dir, group.id, group.currentSprint, `review-${round}.md`, reviewContent);

    const verdict = parseVerdict(reviewContent);
    if (verdict === 'PASS') {
      updateGroup(groupId, { status: 'completed' });
    } else if (round >= maxRetries) {
      updateGroup(groupId, { status: 'failed' });
    } else {
      // 修订循环
      updateGroup(groupId, { currentRound: nextRound });
      const revisionPrompt = buildRevisionPrompt(sprint, nextRound, plan, result, review);
      await dispatchGenerators(groupId, group, generators, revisionPrompt);
    }
  }
}, [...]);
```

### 2.3 Sprint / Round 概念

| 概念 | 说明 | 何时递增 |
|------|------|---------|
| **Sprint** | 完整的功能开发周期 | 每次 `startPipeline()` |
| **Round** | 单次实现+评估循环 | Evaluator 判定 FAIL 后 |

```
Sprint 1:
  Round 0: plan.md → result.md → review-1.md (FAIL)
  Round 1: result-2.md → review-2.md (FAIL)
  Round 2: result-3.md → review-3.md (PASS) → completed
```

### 2.4 文件产物结构

```
.harness/<groupId>/
└── sprint-1/
    ├── plan.md            # Planner 输出
    ├── result.md          # Generator 第一轮输出
    ├── review-1.md        # Evaluator 第一轮评审
    ├── result-2.md        # Generator 第二轮修订
    ├── review-2.md        # Evaluator 第二轮评审
    └── result-3.md        # Generator 第三轮修订
```

**文件名白名单校验**（`src/services/harnessFiles.ts:12`）：
```typescript
const ALLOWED_FILENAMES = /^(plan|result(-\d+)?|review-\d+)\.md$/;
```

## 三、提示词工程

### 3.1 Generator 提示词

**文件位置**: `src/services/harnessPrompts.ts`

```typescript
export function buildGeneratorPrompt(sprint: number, planContent: string): string {
  return `## Harness Task [Sprint ${sprint}]

You are a Generator. Implement the following plan.

### Plan
${planContent}

### Requirements
- Output your implementation result and key decisions
- If anything is unclear, use your best judgment`;
}
```

### 3.2 Evaluator 提示词

```typescript
export function buildEvaluatorPrompt(
  sprint: number, round: number,
  planContent: string, resultContent: string
): string {
  return `## Harness Review [Sprint ${sprint}, Round ${round}]

You are an Evaluator. Evaluate the implementation against the plan.

### Original Plan
${planContent}

### Implementation Result
${resultContent}

### Evaluation Requirements
- Score on: functional completeness, code quality, plan adherence (1-10 each)
- Final verdict: PASS or FAIL
- If FAIL, provide specific revision suggestions
- Last line MUST be: \`VERDICT: PASS\` or \`VERDICT: FAIL\``;
}
```

### 3.3 Revision 提示词

```typescript
export function buildRevisionPrompt(
  sprint: number, round: number,
  planContent: string, resultContent: string, reviewContent: string
): string {
  return `## Harness Revision [Sprint ${sprint}, Round ${round}]

You are a Generator. The Evaluator rejected your implementation. Revise based on feedback.

### Original Plan
${planContent}

### Your Previous Implementation
${resultContent}

### Evaluator Feedback
${reviewContent}

### Requirements
- Address each feedback item
- Output the complete revised result`;
}
```

### 3.4 Prompt 注入链路

```
Planner 输出 → plan.md → 作为 ${planContent} 注入 Generator prompt
Generator 输出 → result.md → 作为 ${resultContent} 注入 Evaluator prompt
Evaluator 输出 → review.md → 作为 ${reviewContent} 注入 Revision prompt
```

## 四、评估闭环

### 4.1 Verdict 判定

**文件位置**: `src/services/harnessPrompts.ts:64-73`

```typescript
export function parseVerdict(evaluatorOutput: string): Verdict {
  const lines = evaluatorOutput.trim().split('\n');
  // 从后向前搜索 VERDICT 行
  for (let i = lines.length - 1; i >= 0; i--) {
    const line = lines[i].trim();
    if (line === 'VERDICT: PASS') return 'PASS';
    if (line === 'VERDICT: FAIL') return 'FAIL';
  }
  return 'UNPARSEABLE';
}
```

### 4.2 重试循环

```
Round 0: Generator → Evaluator → FAIL
Round 1: Generator(revision) → Evaluator → FAIL
Round 2: Generator(revision) → Evaluator → PASS ✓
                                     ↑
                              如果 round >= maxRetries → failed
```

## 五、暂停与恢复

### 5.1 deferredCompletion 机制

当 Pipeline 暂停时，如果有会话刚好完成：
1. 不立即处理完成事件
2. 存储到 `deferredCompletion` 字段
3. 恢复时重放该完成事件

```typescript
// 暂停时
if (group.status === 'paused') {
  setRunState(groupId, { deferredCompletion: completedSessionId });
  return;
}

// 恢复时
const resumePipeline = useCallback((groupId: string) => {
  updateGroup(groupId, { status: 'running' });
  const runState = getRunState(groupId);
  if (runState.deferredCompletion) {
    const deferredId = runState.deferredCompletion;
    setRunState(groupId, { deferredCompletion: null });
    setTimeout(() => advancePipeline(groupId, deferredId), 100);
  }
}, [...]);
```

### 5.2 状态持久化

Harness Group 状态通过 debounce 1s 自动持久化到 SQLite：
- `running` 状态恢复时降级为 `paused`（因为 Pipeline 无法自动恢复 mid-flight）
- `harnessDir` 中的文件产物保留，恢复后可读取

## 六、Canvas 可视化

### 6.1 连接线渲染

**文件位置**: `src/components/harness/ConnectionLine.tsx`

| 角色连接 | 颜色 | 说明 |
|----------|------|------|
| planner → generator | `#3b82f6` (蓝) | 规划到实现 |
| generator → evaluator | `#f97316` (橙) | 实现到评估 |
| evaluator → generator | `#ef4444` (红) | 修订反馈 |

**贝塞尔曲线**：根据源/目标的相对位置智能计算控制点，curvature = min(dist * 0.3, 80)

**运行动画**：SVG `strokeDasharray` + `animate` 标签实现流动虚线效果

### 6.2 角色标识

```
┌─────────────────────────┐
│ [P] planner-session     │  ← 蓝色徽章
│ [G] generator-session   │  ← 绿色徽章
│ [E] evaluator-session   │  ← 橙色徽章
└─────────────────────────┘
```

### 6.3 控制栏交互

| 状态 | 可用操作 |
|------|---------|
| idle | ▶ 启动 |
| running | ⏸ 暂停, ⏹ 停止 |
| paused | ▶ 恢复, ⏹ 停止 |
| completed | ▶ 重新启动（新 Sprint） |
| failed | ▶ 重新启动（新 Sprint） |

### 6.4 连接创建流程

```
1. 拖动 Session 锚点 → handleAnchorDragStart
2. 拖到目标 Session → handleAnchorDragEnd
3. 弹出 RolePickerModal → 选择 fromRole / toRole / groupName
4. 确认 → harness.addConnection(group.id, fromId, toId, fromRole, toRole)
```

## 七、设计亮点

### 7.1 文件产物驱动

- 每个 Round 的产物都以 Markdown 文件形式持久化
- 下游角色通过读取文件获取上游上下文
- 支持跨 Sprint 上下文共享

### 7.2 并行 Generator 支持

- 多个 Generator 可同时运行
- `pendingGenerators` 数组跟踪完成进度
- 所有 Generator 完成后才触发 Evaluator

### 7.3 优雅的暂停/恢复

- `deferredCompletion` 机制不丢失暂停期间的完成事件
- 恢复时 100ms 延迟重放，确保状态已更新
