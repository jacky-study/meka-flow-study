---
article_id: OBA-2prbyf9r
tags: [open-source, meka-flow, canvas-and-views.md, react, typescript, cli]
type: learning
updated_at: 2026-03-29
---

# Meka Flow 无限画布与视图系统深度分析

> 基于 CSS Transform 的无限画布实现，支持拖拽/缩放/自动布局，配合三种视图模式切换。

## 一、无限画布实现

### 1.1 核心技术方案：CSS Transform

**文件位置**: `src/components/CanvasView.tsx:171-299`

```typescript
// 画布变换状态（提升到 App.tsx:154）
const [canvasTransform, setCanvasTransform] = useState({ x: 0, y: 0, scale: 1 });

// 容器样式应用
<div
  className="absolute top-0 left-0"
  style={{
    transform: `translate(${transform.x}px, ${transform.y}px) scale(${transform.scale})`,
    transformOrigin: '0 0',
  }}
>
  {/* Session 窗口们 */}
</div>
```

**为什么不选 Canvas API？**
- CSS Transform 方案可直接复用 React 组件
- 无需自行处理事件坐标转换
- 浏览器硬件加速自动开启 GPU 合成

### 1.2 拖拽实现

| 模式 | 触发条件 | 行为 |
|------|---------|------|
| **手工具** | `toolMode === 'hand'` | 拖拽画布平移 |
| **选择工具** | `toolMode === 'select'` | 框选 Session 或区域选择 |

```typescript
// 拖拽核心逻辑
const handleMouseMove = (e: React.MouseEvent) => {
  if (!isDragging) return;
  const dx = e.clientX - dragStart.x;
  const dy = e.clientY - dragStart.y;
  setTransform(prev => ({ ...prev, x: prev.x + dx, y: prev.y + dy }));
  setDragStart({ x: e.clientX, y: e.clientY });
};
```

### 1.3 缩放实现

**文件位置**: `src/components/CanvasView.tsx:215-235`

```typescript
// 滚轮缩放 — 以鼠标位置为中心
if (e.ctrlKey || e.metaKey) {
  const zoomFactor = e.deltaY > 0 ? 0.9 : 1.1;
  setTransform(prev => {
    const newScale = Math.max(0.1, Math.min(prev.scale * zoomFactor, 3));
    // 以鼠标位置为中心缩放
    const rect = container.getBoundingClientRect();
    const mouseX = e.clientX - rect.left;
    const mouseY = e.clientY - rect.top;
    const newX = mouseX - (mouseX - prev.x) * (newScale / prev.scale);
    const newY = mouseY - (mouseY - prev.y) * (newScale / prev.scale);
    return { x: newX, y: newY, scale: newScale };
  });
}
```

**缩放范围**: 0.1x ~ 3x

### 1.4 自动布局算法

**文件位置**: `src/App.tsx:26-91`

`findNextGridPosition()` 三步策略：

```
步骤 1: 在最后一行最右会话的右侧放置
  ┌─────┐ ┌─────┐ ┌─────┐
  │  A  │ │  B  │ │ 新! │  ← 尝试这里
  └─────┘ └─────┘ └─────┘
  ┌─────┐
  │  C  │
  └─────┘

步骤 2: 新建一行，对齐第一行左侧
  ┌─────┐ ┌─────┐
  │  A  │ │  B  │
  └─────┘ └─────┘
  ┌─────┐ ┌─────┐
  │  C  │ │ 新! │  ← 尝试这里
  └─────┘ └─────┘

步骤 3: 兜底 — 在 y=0 最右侧放置
  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐
  │  A  │ │  B  │ │  C  │ │ 新! │  ← 兜底
  └─────┘ └─────┘ └─────┘ └─────┘
```

**碰撞检测**：
```typescript
const hasAnyCollision = (nx: number, ny: number): boolean =>
  sessions.some(s => {
    const eLeft = s.position.x - gap;
    const eRight = s.position.x + w + gap;
    return nx < eRight && nx + w > eLeft && ny < eBottom && ny + h > eTop;
  });
```

### 1.5 视口聚焦

**文件位置**: `src/components/CanvasView.tsx:150-169`

```typescript
// 聚焦到指定 Session — 居中显示
useEffect(() => {
  if (focusedSessionId && containerRef.current) {
    const session = sessions.find(s => s.id === focusedSessionId);
    const newX = (container.width / 2) - session.position.x - (sessionWidth / 2);
    const newY = (container.height / 2) - session.position.y - (sessionHeight / 2);
    setTransform({ x: newX, y: newY, scale: 1 });
  }
}, [focusedSessionId]);
```

### 1.6 画布状态持久化

**文件位置**: `src/App.tsx:628-639`

```typescript
// debounce 500ms 自动保存到 SQLite
useEffect(() => {
  const timeout = setTimeout(() => {
    backend.updateProject({
      ...currentProject,
      canvas_x: canvasTransform.x,
      canvas_y: canvasTransform.y,
      canvas_zoom: canvasTransform.scale,
    });
  }, 500);
  return () => clearTimeout(timeout);
}, [canvasTransform, currentProject]);
```

---

## 二、SessionWindow 组件

### 2.1 组件 Props

**文件位置**: `src/components/SessionWindow.tsx:21-36`

```typescript
type SessionWindowProps = {
  session: Session;
  onUpdate: (s: Session) => void;
  onClose?: () => void;
  onDelete?: () => void;
  fullScreen?: boolean;
  height?: number;
  animateHeight?: boolean;
  variant?: 'default' | 'tab' | 'popup';  // 三种变体
  projectDir?: string | null;
  onToggleGitPanel?: () => void;
  onCopySession?: (title: string) => void;
  onOpenFileInPanel?: (path: string) => void;
  onOpenDiffInPanel?: (path: string) => void;
};
```

### 2.2 消息渲染流水线

```
SessionWindow
  └── MessageRenderer
        ├── 文本消息 → TextBlock (markdown 渲染)
        └── blocks[] → ContentBlocksView
              ├── text       → TextBlock
              ├── code       → CodeBlock (syntax highlighting)
              ├── tool_call  → ToolCallBlock (可折叠摘要)
              ├── todolist   → TodoListBlock (状态复选框)
              ├── subagent   → SubagentBlock (子 Agent 指示器)
              ├── askuser    → AskUserBlock (选项式交互)
              ├── skill      → SkillBlock (Skill 调用状态)
              ├── file_changes → FileChangesBlock
              └── form_table → FormTableBlock
```

**连续工具调用聚合**（`ContentBlocksView.tsx:14-93`）：
```typescript
// 连续的 tool_call 自动聚合成可折叠摘要
const groups: GroupItem[] = [];
for (const block of visibleBlocks) {
  if (isToolBlock(block)) {
    const last = groups[groups.length - 1];
    if (last?.kind === 'tool_group') {
      last.blocks.push(block);  // 追加到现有工具组
    } else {
      groups.push({ kind: 'tool_group', blocks: [block] });  // 新建工具组
    }
  } else {
    groups.push({ kind: 'block', block });  // 独立块
  }
}
```

### 2.3 输入框与 Skill 选择器

**文件位置**: `src/components/SessionWindow.tsx:836-872`

```typescript
// 输入 "/" 触发 Skill 自动补全
const handleInputChange = async (e) => {
  const val = e.target.value;
  if (val.startsWith('/') && val.length >= 1) {
    const query = val.slice(1).split(' ')[0].toLowerCase();
    const filtered = currentSkills.filter(s =>
      s.name.toLowerCase().includes(query)
    );
    setPickerOpen(filtered.length > 0);
  }
};
```

**Skill 来源**（`src/services/skillScanner.ts`）：
1. 项目级：`<projectDir>/.claude/skills/`
2. 用户级：`~/.claude/skills/`
3. 插件级：读取 `installed_plugins.json`

### 2.4 状态指示器

**文件位置**: `src/components/SessionWindow.tsx:209-234`

| 状态 | 颜色 | 动画 | 说明 |
|------|------|------|------|
| inbox | 灰色 | 无 | 初始状态 |
| inprocess | 蓝色 | pulse | 正在发送 |
| thinking | 黄色 | pulse | AI 思考中 |
| running | 绿色 | pulse | 工具执行中 |
| done | 翠绿 | 无 | 完成 |
| error | 红色 | 无 | 出错 |

### 2.5 窗口调整大小

**文件位置**: `src/components/SessionWindow.tsx:124-198`

```typescript
// 三方向调整：bottom / right / bottom-right
const handleResize = (e, direction) => {
  const startX = e.clientX;
  const startWidth = session.width ?? SESSION_WIDTH;
  const startHeight = session.height ?? SESSION_DEFAULT_HEIGHT;

  const handleMouseMove = (e) => {
    if (direction.includes('right')) {
      newWidth = Math.max(MIN_WIDTH, Math.min(MAX_WIDTH, startWidth + dx));
    }
    if (direction.includes('bottom')) {
      newHeight = Math.max(MIN_HEIGHT, startHeight + dy);
    }
    onUpdate({ ...session, width: newWidth, height: newHeight });
  };
};
```

---

## 三、三种视图模式

### 3.1 Canvas 视图

| 特性 | 实现 |
|------|------|
| 无限画布 | CSS Transform translate + scale |
| 拖拽 | 手工具模式 / 空白区域拖拽 |
| 缩放 | Ctrl+滚轮，鼠标中心缩放 |
| 自动布局 | findNextGridPosition 网格算法 |
| 多选 | 选择工具框选 |
| 广播 | 批量向选中会话发送消息 |
| Harness | 连接线、角色标识 |

### 3.2 Board 视图（看板）

```
┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
│  Inbox   │ │In Process│ │  Review  │ │   Done   │
│ (灰色)   │ │ (蓝色)   │ │ (琥珀)   │ │ (翠绿)   │
│          │ │          │ │          │ │          │
│ Session1 │ │ Session3 │ │ Session5 │ │ Session6 │
│ Session2 │ │ Session4 │ │          │ │ Session7 │
└──────────┘ └──────────┘ └──────────┘ └──────────┘
```

- 按状态分四列
- 右侧面板显示活动会话详情

### 3.3 Tab 视图

```
┌──────────────────────────────────────┐
│ [Session 1] [Session 2] [Session 3] │  ← 标签栏
├──────────────────────────────────────┤
│                                      │
│         当前会话内容                  │
│                                      │
└──────────────────────────────────────┘
```

- 左侧会话列表（320px 宽）
- 搜索过滤 + 时间排序
- 标签式切换

---

## 四、动画和交互

### 4.1 Motion (Framer Motion) 使用

**文件位置**: `src/components/HomePage.tsx:54-68`

```typescript
<motion.div
  initial={{ opacity: 0, scale: 0.95, y: 20 }}
  animate={{ opacity: 1, scale: 1, y: 0 }}
  transition={{ duration: 0.7, ease: "easeOut" }}
>
  {/* 3D 鼠标跟踪效果 */}
  <div style={{
    background: 'radial-gradient(600px circle at var(--mouse-x) var(--mouse-y), rgba(255,255,255,0.06), transparent 40%)'
  }} />
</motion.div>
```

### 4.2 CSS 自定义动画

**文件位置**: `src/index.css:44-50`

```css
/* 呼吸灯动画 — 状态指示 */
@keyframes breathe {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.4; }
}

/* 思考点动画 — AI 生成中 */
@keyframes thinking-dot {
  0%, 80%, 100% { opacity: 0.2; transform: scale(0.8); }
  40% { opacity: 1; transform: scale(1); }
}
.thinking-dot:nth-child(2) { animation-delay: 0.2s; }
.thinking-dot:nth-child(3) { animation-delay: 0.4s; }
```

### 4.3 键盘快捷键

**文件位置**: `src/App.tsx:184-223`

| 快捷键 | 功能 | 条件 |
|--------|------|------|
| `Cmd+N` | 新建会话 | 全局 |
| `Cmd+K` | 聚焦搜索 | 全局 |
| `Cmd+`` ` | 切换终端 | 全局 |
| `Cmd+1~9` | 切换到第 N 个会话 | 非输入框 |
| `Cmd+S` | （被阻止默认行为） | — |

### 4.4 工具调用颜色方案

**文件位置**: `src/components/message/ToolCallBlock.tsx:7-18`

| 工具 | 颜色 | 说明 |
|------|------|------|
| Bash | 绿色 | 命令执行 |
| Read | 蓝色 | 文件读取 |
| Write/Edit | 橙色 | 文件修改 |
| Glob/Grep | 紫色 | 搜索 |
| TaskCreate/Update | 青色 | 任务管理 |
