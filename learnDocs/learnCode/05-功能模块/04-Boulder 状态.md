# Boulder 机制：任务持久化与状态恢复

> 所属模块：05-features-modules | 三层模型：Layer 3 | 优先级：P2

---

## 架构位置

```
┌─────────────────────────────────────────────────────────────┐
│                    Layer 3: Features                        │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                 boulder-state                       │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐ │   │
│  │  │   Types     │  │   Storage   │  │  Top-Level  │ │   │
│  │  │ (类型定义)   │  │ (状态存储)   │  │   Task      │ │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘ │   │
│  │  ┌─────────────┐  ┌─────────────┐                   │   │
│  │  │  Constants  │  │ Worktree    │                   │   │
│  │  │ (常量定义)   │  │   Sync      │                   │   │
│  │  └─────────────┘  └─────────────┘                   │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│                    Layer 2: Tools                           │
│              delegate-task (task tool)                      │
└─────────────────────────────────────────────────────────────┘
```

Boulder State 模块位于 Features 层（Layer 3），是 oh-my-opencode 实现任务持久化和会话连续性的核心组件。它以希腊神话中 Sisyphus 的巨石命名，象征着需要持续推进的永恒任务。该模块负责管理 Prometheus 规划器生成的任务计划状态，确保即使在会话中断后也能恢复工作进度。

---

## 核心职责

Boulder State 机制承担以下核心职责：

**1. 任务状态持久化**：将当前活跃的计划文件路径、开始时间、参与的会话 ID 等关键信息持久化到磁盘。通过 `boulder.json` 文件，系统能够在进程重启或会话中断后恢复工作状态。这种持久化机制是 Sisyphus 编排器实现长时间运行任务的基石。

**2. 会话连续性保障**：记录所有参与执行计划的会话 ID，支持跨会话的任务追踪。当用户在不同时间、不同终端继续工作时，系统能够识别这是同一个计划的延续，而非全新的任务。`task_sessions` 映射还维护了子任务与会话的关联关系，支持子 Agent 会话的复用。

**3. 计划进度追踪**：通过解析 Markdown 计划文件中的复选框状态，实时计算任务完成进度。系统能够识别 TODO 章节和 Final Verification Wave 章节中的未完成任务，帮助 Sisyphus 确定下一步工作重点。

**4. Git Worktree 同步**：在基于 worktree 的工作流中（如 PR 工作流），Boulder State 支持将状态从 worktree 同步回主仓库，确保状态不随 worktree 的清理而丢失。

---

## 代码索引 (Code Index)

| 功能点 | 文件路径 | 行号 | 函数/类 | 说明 |
|--------|----------|------|---------|------|
| 类型定义 | `src/features/boulder-state/types.ts` | 8 | `BoulderState` | 核心状态数据结构 |
| 类型定义 | `src/features/boulder-state/types.ts` | 25 | `PlanProgress` | 计划进度数据结构 |
| 类型定义 | `src/features/boulder-state/types.ts` | 34 | `TaskSessionState` | 任务会话状态 |
| 类型定义 | `src/features/boulder-state/types.ts` | 51 | `TopLevelTaskRef` | 顶层任务引用 |
| 状态读取 | `src/features/boulder-state/storage.ts` | 18 | `readBoulderState()` | 读取 boulder.json |
| 状态写入 | `src/features/boulder-state/storage.ts` | 43 | `writeBoulderState()` | 写入 boulder.json |
| 会话追加 | `src/features/boulder-state/storage.ts` | 59 | `appendSessionId()` | 追加会话 ID |
| 状态清理 | `src/features/boulder-state/storage.ts` | 79 | `clearBoulderState()` | 清除状态文件 |
| 任务会话获取 | `src/features/boulder-state/storage.ts` | 93 | `getTaskSessionState()` | 获取任务会话状态 |
| 任务会话更新 | `src/features/boulder-state/storage.ts` | 102 | `upsertTaskSessionState()` | 更新任务会话状态 |
| 计划发现 | `src/features/boulder-state/storage.ts` | 145 | `findPrometheusPlans()` | 查找所有计划文件 |
| 进度计算 | `src/features/boulder-state/storage.ts` | 171 | `getPlanProgress()` | 计算计划完成进度 |
| 状态创建 | `src/features/boulder-state/storage.ts` | 206 | `createBoulderState()` | 创建新状态对象 |
| 当前任务读取 | `src/features/boulder-state/top-level-task.ts` | 35 | `readCurrentTopLevelTask()` | 读取当前顶层任务 |
| Worktree 同步 | `src/features/boulder-state/worktree-sync.ts` | 6 | `syncSisyphusStateFromWorktree()` | 同步 worktree 状态 |
| 常量定义 | `src/features/boulder-state/constants.ts` | 5 | `BOULDER_DIR` | .sisyphus 目录常量 |
| 常量定义 | `src/features/boulder-state/constants.ts` | 6 | `BOULDER_FILE` | boulder.json 文件名 |

---

## Boulder 状态机制

### 1. 任务状态存储

Boulder State 的核心是 `boulder.json` 文件，存储在项目的 `.sisyphus/` 目录下。该文件包含以下关键信息：

```typescript
// src/features/boulder-state/types.ts:8-23
export interface BoulderState {
  /** Absolute path to the active plan file */
  active_plan: string
  /** ISO timestamp when work started */
  started_at: string
  /** Session IDs that have worked on this plan */
  session_ids: string[]
  /** Plan name derived from filename */
  plan_name: string
  /** Agent type to use when resuming (e.g., 'atlas') */
  agent?: string
  /** Absolute path to the git worktree root where work happens */
  worktree_path?: string
  /** Preferred reusable subagent sessions keyed by current top-level plan task */
  task_sessions?: Record<string, TaskSessionState>
}
```

存储操作通过 `storage.ts` 中的函数实现：

```typescript
// src/features/boulder-state/storage.ts:43-57
export function writeBoulderState(directory: string, state: BoulderState): boolean {
  const filePath = getBoulderFilePath(directory)

  try {
    const dir = dirname(filePath)
    if (!existsSync(dir)) {
      mkdirSync(dir, { recursive: true })
    }

    writeFileSync(filePath, JSON.stringify(state, null, 2), "utf-8")
    return true
  } catch {
    return false
  }
}
```

### 2. 状态恢复

当系统启动或检测到已有计划时，会读取 `boulder.json` 恢复工作状态：

```typescript
// src/features/boulder-state/storage.ts:18-41
export function readBoulderState(directory: string): BoulderState | null {
  const filePath = getBoulderFilePath(directory)

  if (!existsSync(filePath)) {
    return null
  }

  try {
    const content = readFileSync(filePath, "utf-8")
    const parsed = JSON.parse(content)
    if (!parsed || typeof parsed !== "object" || Array.isArray(parsed)) {
      return null
    }
    if (!Array.isArray(parsed.session_ids)) {
      parsed.session_ids = []
    }
    if (!parsed.task_sessions || typeof parsed.task_sessions !== "object") {
      parsed.task_sessions = {}
    }
    return parsed as BoulderState
  } catch {
    return null
  }
}
```

状态恢复时会进行数据校验和默认值填充，确保向后兼容性。

### 3. 会话连续性

Boulder State 通过 `session_ids` 数组追踪所有参与计划执行的会话。每次新会话加入时，会追加到数组中：

```typescript
// src/features/boulder-state/storage.ts:59-77
export function appendSessionId(directory: string, sessionId: string): BoulderState | null {
  const state = readBoulderState(directory)
  if (!state) return null

  if (!state.session_ids?.includes(sessionId)) {
    if (!Array.isArray(state.session_ids)) {
      state.session_ids = []
    }
    const originalSessionIds = [...state.session_ids]
    state.session_ids.push(sessionId)
    if (writeBoulderState(directory, state)) {
      return state
    }
    state.session_ids = originalSessionIds
    return null
  }

  return state
}
```

此外，`task_sessions` 映射支持为每个顶层任务维护一个首选的子 Agent 会话，实现会话复用：

```typescript
// src/features/boulder-state/storage.ts:102-139
export function upsertTaskSessionState(
  directory: string,
  input: {
    taskKey: string
    taskLabel: string
    taskTitle: string
    sessionId: string
    agent?: string
    category?: string
  },
): BoulderState | null {
  const state = readBoulderState(directory)
  if (!state) return null

  const taskSessions = state.task_sessions ?? {}
  taskSessions[input.taskKey] = {
    task_key: input.taskKey,
    task_label: input.taskLabel,
    task_title: input.taskTitle,
    session_id: input.sessionId,
    ...(input.agent !== undefined ? { agent: input.agent } : {}),
    ...(input.category !== undefined ? { category: input.category } : {}),
    updated_at: new Date().toISOString(),
  }

  state.task_sessions = taskSessions
  if (writeBoulderState(directory, state)) {
    return state
  }
  return null
}
```

---

## 状态持久化流程

Boulder State 的完整持久化流程如下：

1. **计划创建**：Prometheus 规划器创建 Markdown 计划文件（`.sisyphus/plans/{name}.md`）
2. **状态初始化**：Sisyphus 初始化 Boulder State，记录 `active_plan` 路径和开始时间
3. **会话注册**：当前会话 ID 被添加到 `session_ids` 数组
4. **任务执行**：执行计划中的任务，更新 `task_sessions` 映射
5. **进度追踪**：通过解析计划文件中的复选框状态计算完成进度
6. **状态同步**：如果使用 worktree，在适当时候同步状态回主仓库
7. **计划完成**：所有任务完成后，清理 Boulder State 文件

---

## 流程/架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Boulder 状态持久化流程                          │
└─────────────────────────────────────────────────────────────────────┘

  Prometheus 创建计划
         │
         ▼
  ┌──────────────┐
  │ 生成 .md 文件 │
  │.sisyphus/plans│
  └──────┬───────┘
         │
         ▼
  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
  │ Sisyphus 启动 │────▶│ 初始化状态   │────▶│ 注册会话 ID  │
  │   计划执行   │     │boulder.json  │     │session_ids[] │
  └──────────────┘     └──────┬───────┘     └──────┬───────┘
                              │                    │
                              ▼                    ▼
                       ┌──────────────┐     ┌──────────────┐
                       │ 记录计划路径  │     │ 追踪任务会话  │
                       │ active_plan  │     │task_sessions │
                       └──────┬───────┘     └──────┬───────┘
                              │                    │
                              └────────┬───────────┘
                                       ▼
                              ┌──────────────┐
                              │  执行任务    │
                              │ 更新复选框   │
                              └──────┬───────┘
                                     │
                     ┌───────────────┼───────────────┐
                     ▼               ▼               ▼
              ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
              │  进度计算     │ │ 会话复用     │ │ Worktree 同步│
              │getPlanProgress│ │复用子 Agent  │ │ 状态回写     │
              └──────┬───────┘ └──────┬───────┘ └──────┬───────┘
                     │               │               │
                     └───────────────┼───────────────┘
                                     ▼
                              ┌──────────────┐
                              │  计划完成    │
                              │ 清理状态文件 │
                              └──────────────┘


┌─────────────────────────────────────────────────────────────────────┐
│                      计划文件结构示例                                │
└─────────────────────────────────────────────────────────────────────┘

  .sisyphus/plans/feature-x.md

  # Plan: feature-x

  ## TODOs

  - [x] 1. 分析现有代码结构
  - [ ] 2. 实现核心功能
  - [ ] 3. 添加单元测试
  - [ ] 4. 更新文档

  ## Final Verification Wave

  - [ ] F1. 运行完整测试套件
  - [ ] F2. 代码审查检查清单


┌─────────────────────────────────────────────────────────────────────┐
│                      boulder.json 结构                               │
└─────────────────────────────────────────────────────────────────────┘

  {
    "active_plan": "/project/.sisyphus/plans/feature-x.md",
    "started_at": "2026-03-28T10:30:00.000Z",
    "session_ids": ["ses_abc123", "ses_def456"],
    "plan_name": "feature-x",
    "agent": "atlas",
    "worktree_path": "/project/.git/worktrees/pr-branch",
    "task_sessions": {
      "todo:2": {
        "task_key": "todo:2",
        "task_label": "2",
        "task_title": "实现核心功能",
        "session_id": "ses_sub789",
        "agent": "hephaestus",
        "category": "deep",
        "updated_at": "2026-03-28T11:00:00.000Z"
      }
    }
  }
```

---

## 关键代码片段

### 片段 1: 计划进度计算

```typescript
// src/features/boulder-state/storage.ts:168-194
export function getPlanProgress(planPath: string): PlanProgress {
  if (!existsSync(planPath)) {
    return { total: 0, completed: 0, isComplete: true }
  }

  try {
    const content = readFileSync(planPath, "utf-8")
    
    // Match markdown checkboxes: - [ ] or - [x] or - [X]
    const uncheckedMatches = content.match(/^\s*[-*]\s*\[\s*\]/gm) || []
    const checkedMatches = content.match(/^\s*[-*]\s*\[[xX]\]/gm) || []

    const total = uncheckedMatches.length + checkedMatches.length
    const completed = checkedMatches.length

    return {
      total,
      completed,
      isComplete: total > 0 && completed === total,
    }
  } catch {
    return { total: 0, completed: 0, isComplete: true }
  }
}
```

### 片段 2: 当前顶层任务识别

```typescript
// src/features/boulder-state/top-level-task.ts:35-77
export function readCurrentTopLevelTask(planPath: string): TopLevelTaskRef | null {
  if (!existsSync(planPath)) {
    return null
  }

  try {
    const content = readFileSync(planPath, "utf-8")
    const lines = content.split(/\r?\n/)
    let section: PlanSection = "other"

    for (const line of lines) {
      if (SECOND_LEVEL_HEADING_PATTERN.test(line)) {
        section = TODO_HEADING_PATTERN.test(line)
          ? "todo"
          : FINAL_VERIFICATION_HEADING_PATTERN.test(line)
            ? "final-wave"
            : "other"
      }

      const uncheckedTaskMatch = line.match(UNCHECKED_CHECKBOX_PATTERN)
      if (!uncheckedTaskMatch) continue
      if (uncheckedTaskMatch[1].length > 0) continue
      if (section !== "todo" && section !== "final-wave") continue

      const taskRef = buildTaskRef(section, uncheckedTaskMatch[2].trim())
      if (taskRef) return taskRef
    }

    return null
  } catch {
    return null
  }
}
```

### 片段 3: Worktree 状态同步

```typescript
// src/features/boulder-state/worktree-sync.ts:6-34
export function syncSisyphusStateFromWorktree(worktreePath: string, mainRepoPath: string): boolean {
  const srcDir = join(worktreePath, BOULDER_DIR)
  const destDir = join(mainRepoPath, BOULDER_DIR)

  if (!existsSync(srcDir)) {
    log("[worktree-sync] No .sisyphus directory in worktree, nothing to sync", { worktreePath })
    return true
  }

  try {
    if (!existsSync(destDir)) {
      mkdirSync(destDir, { recursive: true })
    }

    cpSync(srcDir, destDir, { recursive: true, force: true })
    log("[worktree-sync] Synced .sisyphus state from worktree to main repo", {
      worktreePath,
      mainRepoPath,
    })
    return true
  } catch (err) {
    log("[worktree-sync] Failed to sync .sisyphus state", {
      worktreePath,
      mainRepoPath,
      error: String(err),
    })
    return false
  }
}
```

---

## 依赖关系

Boulder State 模块依赖以下组件：

**1. 文件系统模块**：Node.js 内置 `fs` 模块用于读写 `boulder.json` 文件

**2. 日志系统**：`src/shared/logger.ts` 提供 `log()` 函数用于记录同步操作

**3. Prometheus 规划器**：生成计划文件（`.sisyphus/plans/*.md`），Boulder State 追踪这些文件的执行状态

**4. Sisyphus 编排器**：读取 Boulder State 恢复工作上下文，更新任务执行状态

**5. Worktree 系统**：在 PR 工作流中，Boulder State 与 Git worktree 配合实现状态同步

---

## 实战示例

### 示例 1: 会话中断恢复

场景：用户正在执行一个复杂的重构任务，突然需要关闭电脑。第二天重新打开项目时，系统能够自动恢复工作状态。

```typescript
// 系统启动时检查 Boulder State
const boulderState = readBoulderState(projectDir)

if (boulderState) {
  // 发现已有活跃计划
  console.log(`发现未完成的计划: ${boulderState.plan_name}`)
  console.log(`计划文件: ${boulderState.active_plan}`)
  console.log(`参与会话数: ${boulderState.session_ids.length}`)
  
  // 计算当前进度
  const progress = getPlanProgress(boulderState.active_plan)
  console.log(`进度: ${progress.completed}/${progress.total} (${
    Math.round((progress.completed / progress.total) * 100)
  }%)`)
  
  // 识别当前需要执行的任务
  const currentTask = readCurrentTopLevelTask(boulderState.active_plan)
  if (currentTask) {
    console.log(`当前任务: [${currentTask.section}] ${currentTask.label}. ${currentTask.title}`)
    
    // 检查是否有可复用的子 Agent 会话
    const taskKey = `${currentTask.section}:${currentTask.label.toLowerCase()}`
    const taskSession = getTaskSessionState(projectDir, taskKey)
    
    if (taskSession) {
      console.log(`发现可复用的子 Agent 会话: ${taskSession.session_id}`)
      // 复用该会话继续工作
    }
  }
  
  // 注册当前会话
  appendSessionId(projectDir, currentSessionId)
} else {
  // 没有活跃计划，创建新计划
  console.log("没有活跃计划，需要创建新计划")
}
```

这个示例展示了系统如何利用 Boulder State 实现会话中断后的无缝恢复，包括进度追踪、任务识别和会话复用。

### 示例 2: 跨会话任务延续

场景：一个大型功能开发需要多天完成，每天结束时保存状态，第二天继续。

```typescript
// 第一天：初始化计划
const planPath = "/project/.sisyphus/plans/feature-auth.md"
const sessionId = "ses_day1_001"

// 创建 Boulder State
const state = createBoulderState(planPath, sessionId, "atlas")
writeBoulderState(projectDir, state)

// 执行第一个任务
const task1Key = "todo:1"
const subagentSession = await startSubagent({
  description: "实现登录 API",
  prompt: "实现 /api/auth/login 端点...",
  category: "backend"
})

// 记录任务会话关联
upsertTaskSessionState(projectDir, {
  taskKey: task1Key,
  taskLabel: "1",
  taskTitle: "实现登录 API",
  sessionId: subagentSession.id,
  agent: "hephaestus",
  category: "backend"
})

// 标记任务完成（在计划文件中勾选）
// ...

// 第二天：恢复工作
const restoredState = readBoulderState(projectDir)
if (restoredState) {
  // 追加新会话
  appendSessionId(projectDir, "ses_day2_001")
  
  // 获取当前任务
  const currentTask = readCurrentTopLevelTask(restoredState.active_plan)
  if (currentTask) {
    const taskKey = `${currentTask.section}:${currentTask.label.toLowerCase()}`
    
    // 检查是否有可复用的会话
    const prevSession = getTaskSessionState(projectDir, taskKey)
    if (prevSession) {
      // 复用之前的子 Agent 会话，保持上下文连续性
      console.log(`复用会话 ${prevSession.session_id} 继续任务`)
    } else {
      // 创建新的子 Agent 会话
      console.log(`为新任务 ${taskKey} 创建会话`)
    }
  }
}

// 如果使用 worktree，在 PR 合并前同步状态
if (state.worktree_path) {
  syncSisyphusStateFromWorktree(state.worktree_path, projectDir)
}
```

这个示例展示了 Boulder State 如何支持跨多天的长时间运行任务，通过 `task_sessions` 映射实现子 Agent 会话的复用，避免重复初始化带来的上下文丢失。

---

## 交叉引用

- 参见：[后台 Agent 执行](./01-后台 Agent.md) - 了解后台任务的执行机制和并发控制
- 参见：[Prometheus 规划器](../02-core-agents/prometheus-planner.md) - 了解计划文件的生成和结构
- 参见：[Sisyphus 编排器](../02-core-agents/sisyphus-orchestrator.md) - 了解如何利用 Boulder State 实现任务编排
- 参见：[会话持久化](../01-Harness 架构/06-会话持久化.md) - 了解 OpenCode 会话的生命周期管理
- 参见：[Worktree PR 工作流](../03-workflow-patterns/worktree-pr-workflow.md) - 了解基于 worktree 的 PR 开发流程
