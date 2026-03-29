# Continuation Hooks：7 个任务延续钩子

> 所属模块：03-Hook 系统 | 三层模型：Layer 3 | 优先级：P1

---

## 架构位置

```
┌─────────────────────────────────────────────────────────────┐
│                    Hook 三层模型 (48 Hooks)                  │
├─────────────────────────────────────────────────────────────┤
│ Layer 1: Core Hooks (39)                                    │
│   ├── Session Hooks (23)                                    │
│   ├── Tool Guard Hooks (12)                                 │
│   └── Transform Hooks (4)                                   │
├─────────────────────────────────────────────────────────────┤
│ Layer 2: Continuation Hooks (7)  ◄── 你在这里               │
│   ├── stopContinuationGuard                                 │
│   ├── compactionContextInjector                             │
│   ├── compactionTodoPreserver                               │
│   ├── todoContinuationEnforcer                              │
│   ├── unstableAgentBabysitter                               │
│   ├── backgroundNotificationHook                            │
│   └── atlasHook                                             │
├─────────────────────────────────────────────────────────────┤
│ Layer 3: Skill Hooks (2)                                    │
│   ├── categorySkillReminder                                 │
│   └── autoSlashCommand                                      │
└─────────────────────────────────────────────────────────────┘
```

Continuation Hooks 位于 Hook 系统的第三层（Layer 3），专门负责**任务延续机制**、**后台任务恢复**和 **Session 恢复**等核心功能。这些钩子确保 Agent 在长时间运行任务中保持连续性，防止任务中断或丢失。

---

## 核心职责

Continuation Hooks 的核心职责是**维持 Agent 任务的连续性和完整性**。当 Agent 执行复杂的多步骤任务时，可能会遇到各种中断情况：Session 空闲、上下文压缩、后台任务完成等。这一层钩子通过智能监控和自动恢复机制，确保任务能够无缝继续。

具体来说，Continuation Hooks 负责：

1. **任务延续控制**：通过 `stopContinuationGuard` 提供 `/stop-continuation` 命令，允许用户手动停止自动延续行为
2. **上下文恢复**：在 Session 压缩后，通过 `compactionContextInjector` 和 `compactionTodoPreserver` 恢复关键上下文和待办事项
3. **Boulder 机制**：`todoContinuationEnforcer` 和 `atlasHook` 实现"滚石"机制，当检测到未完成的待办事项时自动注入延续提示
4. **后台任务通知**：`backgroundNotificationHook` 在后台任务完成时通知主 Session
5. **不稳定 Agent 监控**：`unstableAgentBabysitter` 监控长时间无响应的后台任务并提醒主 Session

---

## 代码索引 (Code Index)

| 功能点 | 文件路径 | 行号 | 函数/类 | 说明 |
|--------|----------|------|---------|------|
| 创建所有 Continuation Hooks | `src/plugin/hooks/create-continuation-hooks.ts` | 31-128 | `createContinuationHooks` | 工厂函数，组装 7 个钩子 |
| 停止延续守卫 | `src/hooks/stop-continuation-guard/hook.ts` | 25-116 | `createStopContinuationGuardHook` | 处理 `/stop-continuation` 命令 |
| 压缩上下文注入器 | `src/hooks/compaction-context-injector/hook.ts` | 14-164 | `createCompactionContextInjector` | Session 压缩后恢复上下文 |
| 压缩待办保留器 | `src/hooks/compaction-todo-preserver/hook.ts` | 52-127 | `createCompactionTodoPreserverHook` | 压缩前后保存和恢复待办 |
| Boulder 延续执行器 | `src/hooks/todo-continuation-enforcer/index.ts` | 12-59 | `createTodoContinuationEnforcer` | 主 Session 的延续机制 |
| Boulder 延续处理器 | `src/hooks/todo-continuation-enforcer/handler.ts` | 15-88 | `createTodoContinuationHandler` | 处理 session.idle 等事件 |
| 空闲事件处理 | `src/hooks/todo-continuation-enforcer/idle-event.ts` | 19-194 | `handleSessionIdle` | 核心决策逻辑 |
| Atlas 主控钩子 | `src/hooks/atlas/atlas-hook.ts` | 7-27 | `createAtlasHook` | Boulder Session 的主控器 |
| Atlas 事件处理 | `src/hooks/atlas/event-handler.ts` | 8-104 | `createAtlasEventHandler` | Atlas 的事件路由 |
| Boulder 延续注入 | `src/hooks/atlas/boulder-continuation-injector.ts` | 10-84 | `injectBoulderContinuation` | 注入延续提示到 Session |
| 后台通知钩子 | `src/hooks/background-notification/hook.ts` | 26-42 | `createBackgroundNotificationHook` | 后台任务完成通知 |
| 不稳定 Agent 保姆 | `src/hooks/unstable-agent-babysitter/unstable-agent-babysitter-hook.ts` | 118-176 | `createUnstableAgentBabysitterHook` | 监控卡死的后台任务 |

---

## 7 个 Continuation Hooks

### 1. stopContinuationGuard

**功能**：提供 `/stop-continuation` 命令，允许用户手动停止当前 Session 的自动延续行为。

**触发事件**：
- `chat.message`：新用户消息时清除停止状态
- `event`：监听 `session.deleted` 清理资源

**核心代码**（`src/hooks/stop-continuation-guard/hook.ts:33-69`）：

```typescript
const stop = (sessionID: string): void => {
  stoppedSessions.add(sessionID)
  setContinuationMarkerSource(ctx.directory, sessionID, "stop", "stopped", "continuation stopped")
  log(`[${HOOK_NAME}] Continuation stopped for session`, { sessionID })

  const backgroundManager = options?.backgroundManager
  if (!backgroundManager) return

  const cancellableTasks = backgroundManager
    .getAllDescendantTasks(sessionID)
    .filter((task) => task.status === "running" || task.status === "pending")

  if (cancellableTasks.length === 0) return

  void Promise.allSettled(
    cancellableTasks.map(async (task) => {
      await backgroundManager.cancelTask(task.id, {
        source: "stop-continuation",
        reason: "Continuation stopped via /stop-continuation",
        abortSession: task.status === "running",
        skipNotification: true,
      })
    })
  )
}
```

### 2. compactionContextInjector

**功能**：在 Session 压缩（compaction）后自动恢复 Agent 配置（agent、model、tools）。

**触发事件**：
- `session.compacted`：压缩完成后恢复配置
- `session.idle`：监控消息尾部状态
- `message.updated` / `message.part.delta`：跟踪 Assistant 消息输出

**核心代码**（`src/hooks/compaction-context-injector/hook.ts:38-55`）：

```typescript
const capture = async (sessionID: string): Promise<void> => {
  if (!ctx || !sessionID) return

  const promptConfig = await resolveSessionPromptConfig(ctx, sessionID)
  if (!promptConfig.agent && !promptConfig.model && !promptConfig.tools) {
    return
  }

  setCompactionAgentConfigCheckpoint(sessionID, promptConfig)
  log(`[compaction-context-injector] Captured agent checkpoint before compaction`, {
    sessionID,
    agent: promptConfig.agent,
    model: promptConfig.model,
    hasTools: !!promptConfig.tools,
  })
}
```

### 3. compactionTodoPreserver

**功能**：在 Session 压缩前后保存待办事项列表，压缩完成后恢复，防止待办丢失。

**触发事件**：
- `session.compacted`：压缩完成后恢复待办
- `session.deleted`：清理缓存

**核心代码**（`src/hooks/compaction-todo-preserver/hook.ts:57-104`）：

```typescript
const capture = async (sessionID: string): Promise<void> => {
  if (!sessionID) return
  try {
    const response = await ctx.client.session.todo({ path: { id: sessionID } })
    const todos = extractTodos(response)
    if (todos.length === 0) return
    snapshots.set(sessionID, todos)
    log(`[${HOOK_NAME}] Captured todo snapshot`, { sessionID, count: todos.length })
  } catch (err) {
    log(`[${HOOK_NAME}] Failed to capture todos`, { sessionID, error: String(err) })
  }
}

const restore = async (sessionID: string): Promise<void> => {
  const snapshot = snapshots.get(sessionID)
  if (!snapshot || snapshot.length === 0) return

  // ... 检查当前是否已有待办 ...

  const writer = await resolveTodoWriter()
  if (!writer) {
    log(`[${HOOK_NAME}] Skipped restore (Todo.update unavailable)`, { sessionID })
    return
  }

  try {
    await writer({ sessionID, todos: snapshot })
    log(`[${HOOK_NAME}] Restored todos after compaction`, { sessionID, count: snapshot.length })
  } catch (err) {
    log(`[${HOOK_NAME}] Failed to restore todos`, { sessionID, error: String(err) })
  }
}
```

### 4. todoContinuationEnforcer

**功能**：核心 Boulder 机制。当主 Session（Sisyphus）有待办未完成且进入空闲状态时，自动注入延续提示强制继续任务。

**触发事件**：
- `session.idle`：核心决策点
- `session.error`：检测中止错误
- `session.compacted`：设置压缩保护期

**决策流程**（`src/hooks/todo-continuation-enforcer/idle-event.ts:19-194`）：

```typescript
export async function handleSessionIdle(args: {
  ctx: PluginInput
  sessionID: string
  sessionStateStore: SessionStateStore
  backgroundManager?: BackgroundManager
  skipAgents?: string[]
  isContinuationStopped?: (sessionID: string) => boolean
}): Promise<void> {
  // 1. 检查是否在恢复中
  if (state.isRecovering) return

  // 2. 检查是否刚发生中止
  if (state.abortDetectedAt) {
    const timeSinceAbort = Date.now() - state.abortDetectedAt
    if (timeSinceAbort < ABORT_WINDOW_MS) return
  }

  // 3. 检查是否有运行中的后台任务
  const hasRunningBgTasks = backgroundManager
    ? backgroundManager.getTasksByParentSession(sessionID).some((task) => task.status === "running")
    : false
  if (hasRunningBgTasks) return

  // 4. 检查是否有未完成的待办
  const incompleteCount = getIncompleteCount(todos)
  if (incompleteCount === 0) return

  // 5. 检查冷却时间（指数退避）
  const effectiveCooldown = CONTINUATION_COOLDOWN_MS * Math.pow(2, Math.min(state.consecutiveFailures, 5))
  if (state.lastInjectedAt && Date.now() - state.lastInjectedAt < effectiveCooldown) return

  // 6. 检查是否被停止
  if (isContinuationStopped?.(sessionID)) return

  // 7. 启动倒计时注入延续提示
  startCountdown({ ctx, sessionID, incompleteCount, total: todos.length, ... })
}
```

### 5. atlasHook

**功能**：Boulder Session（ralph-loop、子 Agent）的主控器。监控这些 Session 的空闲状态，在任务未完成时注入延续提示。

**触发事件**：
- `session.idle`：决策是否注入延续
- `session.error`：检测中止信号
- `tool.execute.before` / `tool.execute.after`：执行写入策略和验证提醒

**核心代码**（`src/hooks/atlas/atlas-hook.ts:7-27`）：

```typescript
export function createAtlasHook(ctx: PluginInput, options?: AtlasHookOptions) {
  const sessions = new Map<string, SessionState>()
  const pendingFilePaths = new Map<string, string>()
  const pendingTaskRefs = new Map<string, PendingTaskRef>()
  const autoCommit = options?.autoCommit ?? true

  function getState(sessionID: string): SessionState {
    let state = sessions.get(sessionID)
    if (!state) {
      state = { promptFailureCount: 0 }
      sessions.set(sessionID, state)
    }
    return state
  }

  return {
    handler: createAtlasEventHandler({ ctx, options, sessions, getState }),
    "tool.execute.before": createToolExecuteBeforeHandler({ ctx, pendingFilePaths, pendingTaskRefs }),
    "tool.execute.after": createToolExecuteAfterHandler({ ctx, pendingFilePaths, pendingTaskRefs, autoCommit, getState }),
  }
}
```

### 6. backgroundNotificationHook

**功能**：将后台任务事件路由到 BackgroundManager，并在聊天消息中注入待处理的通知。

**触发事件**：
- `event`：所有后台任务相关事件
- `chat.message`：注入待处理通知到消息中

**核心代码**（`src/hooks/background-notification/hook.ts:26-42`）：

```typescript
export function createBackgroundNotificationHook(manager: BackgroundManager) {
  const eventHandler = async ({ event }: EventInput) => {
    manager.handleEvent(event)
  }

  const chatMessageHandler = async (
    input: ChatMessageInput,
    output: ChatMessageOutput,
  ): Promise<void> => {
    manager.injectPendingNotificationsIntoChatMessage(output, input.sessionID)
  }

  return {
    "chat.message": chatMessageHandler,
    event: eventHandler,
  }
}
```

### 7. unstableAgentBabysitter

**功能**：监控后台任务的健康状态。当某个后台任务长时间（默认 2 分钟）没有新消息时，向主 Session 注入提醒。

**触发事件**：
- `session.idle`：检查所有后台任务的最后消息时间

**核心代码**（`src/hooks/unstable-agent-babysitter/unstable-agent-babysitter-hook.ts:118-176`）：

```typescript
export function createUnstableAgentBabysitterHook(ctx: BabysitterContext, options: BabysitterOptions) {
  const reminderCooldowns = new Map<string, number>()

  const eventHandler = async ({ event }: { event: { type: string; properties?: unknown } }) => {
    if (event.type !== "session.idle") return

    const props = event.properties as Record<string, unknown> | undefined
    const sessionID = props?.sessionID as string | undefined
    if (!sessionID) return

    const mainSessionID = getMainSessionID()
    if (!mainSessionID || sessionID !== mainSessionID) return

    const tasks = options.backgroundManager.getTasksByParentSession(mainSessionID)
    if (tasks.length === 0) return

    const timeoutMs = options.config?.timeout_ms ?? DEFAULT_TIMEOUT_MS
    const now = Date.now()

    for (const task of tasks) {
      if (task.status !== "running") continue
      if (!isUnstableTask(task)) continue

      const lastMessageAt = task.progress?.lastMessageAt
      if (!lastMessageAt) continue

      const idleMs = now - lastMessageAt.getTime()
      if (idleMs < timeoutMs) continue

      const lastReminderAt = reminderCooldowns.get(task.id)
      if (lastReminderAt && now - lastReminderAt < COOLDOWN_MS) continue

      const summary = task.sessionID ? await getThinkingSummary(ctx, task.sessionID) : null
      const reminder = buildReminder(task, summary, idleMs)
      const { agent, model, tools } = await resolveMainSessionTarget(ctx, mainSessionID)

      try {
        await ctx.client.session.promptAsync({
          path: { id: mainSessionID },
          body: {
            ...(agent ? { agent } : {}),
            ...(model ? { model } : {}),
            ...(tools ? { tools } : {}),
            parts: [createInternalAgentTextPart(reminder)],
          },
          query: { directory: ctx.directory },
        })
        reminderCooldowns.set(task.id, now)
        log(`[${HOOK_NAME}] Reminder injected`, { taskId: task.id, sessionID: mainSessionID })
      } catch (error) {
        log(`[${HOOK_NAME}] Reminder injection failed`, { taskId: task.id, error: String(error) })
      }
    }
  }

  return { event: eventHandler }
}
```

---

## 任务延续流程

Continuation Hooks 的任务延续流程遵循以下原则：

1. **分层处理**：主 Session 由 `todoContinuationEnforcer` 处理，Boulder Session（子 Agent、ralph-loop）由 `atlasHook` 处理
2. **智能决策**：每个钩子都有多层决策门，避免不必要的延续注入
3. **指数退避**：连续失败时增加冷却时间，防止无限循环
4. **用户可控**：通过 `/stop-continuation` 命令可随时停止自动延续

**决策门检查顺序**：
1. Session 类型检查（是否 Boulder Session）
2. 中止信号检查（是否刚发生错误/中止）
3. 后台任务检查（是否有运行中的子任务）
4. 待办状态检查（是否有未完成的待办）
5. 冷却时间检查（是否满足最小间隔）
6. 失败次数检查（是否超过最大连续失败次数）
7. 用户停止检查（是否被 `/stop-continuation` 停止）

---

## 流程/架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Continuation Hooks 架构图                         │
└─────────────────────────────────────────────────────────────────────┘

                              用户输入
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        stopContinuationGuard                        │
│  处理 /stop-continuation 命令，管理停止状态                          │
└─────────────────────────────────────────────────────────────────────┘
                                 │
            ┌────────────────────┼────────────────────┐
            │                    │                    │
            ▼                    ▼                    ▼
┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│ todoContinuation │  │    atlasHook     │  │ backgroundNotif  │
│    Enforcer      │  │  (Boulder主控)   │  │   icationHook    │
│                  │  │                  │  │                  │
│ 处理主Session的   │  │ 处理子Agent/     │  │ 后台任务完成通知  │
│ 延续机制          │  │ ralph-loop延续   │  │                  │
└──────────────────┘  └──────────────────┘  └──────────────────┘
            │                    │                    │
            │         ┌─────────┴─────────┐           │
            │         │                   │           │
            │         ▼                   ▼           │
            │  ┌──────────────┐   ┌──────────────┐   │
            │  │ compaction   │   │ unstableAgent│   │
            │  │ Context      │   │ Babysitter   │   │
            │  │ Injector     │   │              │   │
            │  │              │   │ 监控卡死任务  │   │
            │  │ 压缩后恢复    │   │ 注入提醒      │   │
            │  │ 上下文        │   │              │   │
            │  └──────────────┘   └──────────────┘   │
            │         │                   │           │
            │         ▼                   │           │
            │  ┌──────────────┐           │           │
            │  │ compaction   │◄──────────┘           │
            │  │ Todo         │                       │
            └─►│ Preserver    │◄──────────────────────┘
               │              │
               │ 压缩后恢复    │
               │ 待办事项      │
               └──────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                         决策流程 (Decision Gate)                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   session.idle                                                      │
│       │                                                             │
│       ▼                                                             │
│   ┌─────────────┐    否    ┌─────────────┐                         │
│   │ 是Boulder   │─────────►│ 跳过        │                         │
│   │ Session?    │          │             │                         │
│   └─────────────┘          └─────────────┘                         │
│       │ 是                                                          │
│       ▼                                                             │
│   ┌─────────────┐    是    ┌─────────────┐                         │
│   │ 有中止信号? │─────────►│ 跳过        │                         │
│   └─────────────┘          │ (3s保护期)  │                         │
│       │ 否                 └─────────────┘                         │
│       ▼                                                             │
│   ┌─────────────┐    是    ┌─────────────┐                         │
│   │ 后台任务    │─────────►│ 跳过        │                         │
│   │ 运行中?     │          │             │                         │
│   └─────────────┘          └─────────────┘                         │
│       │ 否                                                          │
│       ▼                                                             │
│   ┌─────────────┐    否    ┌─────────────┐                         │
│   │ 有待办      │─────────►│ 重置状态    │                         │
│   │ 未完成?     │          │ 正常结束    │                         │
│   └─────────────┘          └─────────────┘                         │
│       │ 是                                                          │
│       ▼                                                             │
│   ┌─────────────┐    是    ┌─────────────┐                         │
│   │ 冷却时间    │─────────►│ 跳过        │                         │
│   │ 未到?       │          │ (指数退避)  │                         │
│   └─────────────┘          └─────────────┘                         │
│       │ 否                                                          │
│       ▼                                                             │
│   ┌─────────────┐    是    ┌─────────────┐                         │
│   │ 被用户停止? │─────────►│ 跳过        │                         │
│   └─────────────┘          └─────────────┘                         │
│       │ 否                                                          │
│       ▼                                                             │
│   ┌─────────────┐                                                   │
│   │ 启动2秒倒计时 │───► 注入 CONTINUATION_PROMPT                     │
│   └─────────────┘                                                   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 关键代码片段

### 片段 1：Continuation Hooks 工厂组装

文件：`src/plugin/hooks/create-continuation-hooks.ts:31-128`

```typescript
export function createContinuationHooks(args: {
  ctx: PluginContext
  pluginConfig: OhMyOpenCodeConfig
  isHookEnabled: (hookName: HookName) => boolean
  safeHookEnabled: boolean
  backgroundManager: BackgroundManager
  sessionRecovery: SessionRecovery
}): ContinuationHooks {
  const {
    ctx,
    pluginConfig,
    isHookEnabled,
    safeHookEnabled,
    backgroundManager,
    sessionRecovery,
  } = args

  const safeHook = <T>(hookName: HookName, factory: () => T): T | null =>
    safeCreateHook(hookName, factory, { enabled: safeHookEnabled })

  const stopContinuationGuard = isHookEnabled("stop-continuation-guard")
    ? safeHook("stop-continuation-guard", () =>
        createStopContinuationGuardHook(ctx, { backgroundManager }))
    : null

  const compactionContextInjector = isHookEnabled("compaction-context-injector")
    ? safeHook("compaction-context-injector", () =>
        createCompactionContextInjector({ ctx, backgroundManager }))
    : null

  const compactionTodoPreserver = isHookEnabled("compaction-todo-preserver")
    ? safeHook("compaction-todo-preserver", () => createCompactionTodoPreserverHook(ctx))
    : null

  const todoContinuationEnforcer = isHookEnabled("todo-continuation-enforcer")
    ? safeHook("todo-continuation-enforcer", () =>
      createTodoContinuationEnforcer(ctx, {
          backgroundManager,
          isContinuationStopped: stopContinuationGuard?.isStopped,
        }))
    : null

  // ... 其他钩子组装 ...

  return {
    stopContinuationGuard,
    compactionContextInjector,
    compactionTodoPreserver,
    todoContinuationEnforcer,
    unstableAgentBabysitter,
    backgroundNotificationHook,
    atlasHook,
  }
}
```

### 片段 2：Boulder 延续注入

文件：`src/hooks/atlas/boulder-continuation-injector.ts:10-45`

```typescript
export async function injectBoulderContinuation(input: {
  ctx: PluginInput
  sessionID: string
  planName: string
  remaining: number
  total: number
  agent?: string
  worktreePath?: string
  preferredTaskSessionId?: string
  preferredTaskTitle?: string
  backgroundManager?: BackgroundManager
  sessionState: SessionState
}): Promise<void> {
  const hasRunningBgTasks = backgroundManager
    ? backgroundManager.getTasksByParentSession(sessionID).some((t: { status: string }) => t.status === "running")
    : false

  if (hasRunningBgTasks) {
    log(`[${HOOK_NAME}] Skipped injection: background tasks running`, { sessionID })
    return
  }

  const worktreeContext = worktreePath ? `\n\n[Worktree: ${worktreePath}]` : ""
  const preferredSessionContext = preferredTaskSessionId
    ? `\n\n[Preferred reuse session for current top-level plan task${preferredTaskTitle ? `: ${preferredTaskTitle}` : ""}: ${preferredTaskSessionId}]`
    : ""
  const prompt =
    BOULDER_CONTINUATION_PROMPT.replace(/{PLAN_NAME}/g, planName) +
    `\n\n[Status: ${total - remaining}/${total} completed, ${remaining} remaining]` +
    preferredSessionContext +
    worktreeContext

  // ... 注入提示 ...
}
```

### 片段 3：指数退避冷却计算

文件：`src/hooks/todo-continuation-enforcer/idle-event.ts:125-130`

```typescript
const effectiveCooldown =
  CONTINUATION_COOLDOWN_MS * Math.pow(2, Math.min(state.consecutiveFailures, 5))
if (state.lastInjectedAt && Date.now() - state.lastInjectedAt < effectiveCooldown) {
  log(`[${HOOK_NAME}] Skipped: cooldown active`, { sessionID, effectiveCooldown, consecutiveFailures: state.consecutiveFailures })
  return
}
```

---

## 依赖关系

Continuation Hooks 依赖以下核心组件：

| 依赖组件 | 路径 | 用途 |
|----------|------|------|
| BackgroundManager | `src/features/background-agent` | 管理后台任务生命周期 |
| SessionRecovery | `src/hooks/session-recovery` | Session 恢复机制 |
| PluginContext | `src/plugin/types` | OpenCode 插件上下文 |
| safeCreateHook | `src/shared/safe-create-hook` | 安全钩子创建包装器 |
| Logger | `src/shared/logger` | 日志记录 |

---

## 实战示例

### 示例 1：后台任务恢复

**场景**：用户启动了一个复杂的代码重构任务，Sisyphus 将工作委托给多个后台 Agent（Hephaestus、Oracle）并行处理。主 Session 进入空闲状态。

**流程**：

1. 主 Session 进入 `session.idle` 状态
2. `todoContinuationEnforcer` 检测到有待办未完成
3. 检查后台任务状态：`backgroundManager.getTasksByParentSession(sessionID)`
4. 发现有运行中的后台任务，**跳过延续注入**
5. 后台任务陆续完成，`backgroundNotificationHook` 接收完成事件
6. 所有后台任务完成后，主 Session 再次进入 `session.idle`
7. 这次没有运行中的后台任务，启动 2 秒倒计时
8. 倒计时结束后注入 `CONTINUATION_PROMPT`，Agent 继续处理待办

```typescript
// 关键代码逻辑
const hasRunningBgTasks = backgroundManager
  ? backgroundManager.getTasksByParentSession(sessionID).some((task) => task.status === "running")
  : false

if (hasRunningBgTasks) {
  log(`[${HOOK_NAME}] Skipped: background tasks running`, { sessionID })
  return  // 等待后台任务完成
}
```

### 示例 2：Session 恢复

**场景**：长时间运行的 Session 触发了上下文压缩（compaction），导致 Agent 配置和待办事项丢失。

**流程**：

1. Session 触发 `session.compacted` 事件
2. `compactionContextInjector` 从之前保存的 checkpoint 恢复 agent、model、tools 配置
3. `compactionTodoPreserver` 从 snapshot 恢复待办事项列表
4. Session 继续运行，Agent 保持原有配置和任务状态

```typescript
// compactionContextInjector 恢复配置
const recoverCheckpointedAgentConfig = async (sessionID: string, trigger: string) => {
  const checkpoint = getCompactionAgentConfigCheckpoint(sessionID)
  if (!checkpoint) return

  // 恢复 agent、model、tools
  await ctx.client.session.prompt({
    path: { id: sessionID },
    body: {
      agent: checkpoint.agent,
      model: checkpoint.model,
      tools: checkpoint.tools,
      parts: [{ type: "text", text: COMPACTION_CONTEXT_PROMPT }],
    },
  })
}

// compactionTodoPreserver 恢复待办
const restore = async (sessionID: string): Promise<void> => {
  const snapshot = snapshots.get(sessionID)
  if (!snapshot || snapshot.length === 0) return

  const writer = await resolveTodoWriter()
  if (!writer) return

  await writer({ sessionID, todos: snapshot })
  log(`[${HOOK_NAME}] Restored todos after compaction`, { sessionID, count: snapshot.length })
}
```

---

## 交叉引用

- 参见：[Hook 分层](./01-Hook%20分层.md) - 了解 Hook 三层模型的整体架构
- 参见：[Session Hooks](./02-Session%20Hooks.md) - 了解 Layer 1 的 Session 相关钩子
- 参见：[后台 Agent](../05-功能模块/01-后台%20Agent.md) - 了解 BackgroundManager 的详细实现
- 参见：[Atlas 架构](../05-功能模块/02-Atlas%20架构.md) - 深入了解 Boulder Session 的主控机制
- 参见：[Session 压缩](../05-功能模块/03-Session%20压缩.md) - 了解 compaction 机制和恢复策略
- 参见：[Ralph Loop](../05-功能模块/04-Ralph%20Loop.md) - 了解自引用开发循环的实现
