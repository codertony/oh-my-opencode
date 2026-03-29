# 后台 Agent 执行：并行任务管理

> 所属模块：05-features-modules | 三层模型：Layer 3 | 优先级：P2

---

## 架构位置

```
┌─────────────────────────────────────────────────────────────┐
│                    Layer 3: Features                        │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              background-agent                       │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐ │   │
│  │  │   Manager   │  │ Concurrency │  │   Spawner   │ │   │
│  │  │  (任务管理)  │  │  (并发控制)  │  │  (任务启动) │ │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘ │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐ │   │
│  │  │Task Poller  │  │Task History │  │   Types     │ │   │
│  │  │ (状态轮询)   │  │ (历史记录)   │  │ (类型定义)  │ │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘ │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Layer 2: Tools                           │
│              delegate-task (task tool)                      │
└─────────────────────────────────────────────────────────────┘
```

后台 Agent 执行系统位于 Features 层（Layer 3），是 oh-my-opencode 实现多 Agent 并行协作的核心模块。它通过 `BackgroundManager` 管理任务生命周期，支持并发控制、任务队列、结果收集等关键功能。

---

## 核心职责

后台 Agent 执行系统承担以下核心职责：

**1. 并行任务调度**：允许同时启动多个后台 Agent 任务，每个任务在独立的会话中运行。系统通过 `ConcurrencyManager` 实现基于模型/提供商的并发控制，默认每个模型最多 5 个并发任务。

**2. 任务生命周期管理**：从任务创建（pending）、排队等待、开始执行（running）到完成（completed/error/cancelled），系统全程跟踪任务状态。任务状态转换通过事件驱动机制实现，包括会话空闲事件、消息更新事件等。

**3. 结果异步收集**：后台任务采用"fire-and-forget"模式启动，执行完成后通过 `background_output` 工具获取结果。系统会自动通知父会话任务完成状态，支持任务历史记录查询。

**4. 资源隔离与清理**：每个后台任务拥有独立的 OpenCode 会话，任务完成后自动清理资源。系统实现了 Circuit Breaker 机制防止无限循环，支持任务超时中断和手动取消。

---

## 代码索引 (Code Index)

| 功能点 | 文件路径 | 行号 | 函数/类 | 说明 |
|--------|----------|------|---------|------|
| 任务管理器 | `src/features/background-agent/manager.ts` | 129 | `BackgroundManager` | 核心类，管理任务生命周期 |
| 并发控制 | `src/features/background-agent/concurrency.ts` | 15 | `ConcurrencyManager` | 基于模型的并发限制管理 |
| 任务类型定义 | `src/features/background-agent/types.ts` | 29 | `BackgroundTask` | 任务数据结构定义 |
| 任务启动器 | `src/features/background-agent/spawner.ts` | 36 | `startTask()` | 启动后台任务执行 |
| 状态轮询 | `src/features/background-agent/task-poller.ts` | 100 | `checkAndInterruptStaleTasks()` | 检测并中断僵死任务 |
| 任务历史 | `src/features/background-agent/task-history.ts` | 16 | `TaskHistory` | 记录任务执行历史 |
| 常量定义 | `src/features/background-agent/constants.ts` | 1 | `POLLING_INTERVAL_MS` | 轮询间隔等常量 |
| 后台任务执行 | `src/tools/delegate-task/background-task.ts` | 13 | `executeBackgroundTask()` | Task 工具的后台执行逻辑 |
| 任务委托工具 | `src/tools/delegate-task/tools.ts` | 28 | `createDelegateTask()` | Task 工具工厂函数 |
| 结果获取工具 | `src/tools/delegate-task/index.ts` | - | `background_output` | 获取后台任务结果 |

---

## 后台执行机制

### 1. 并行任务执行

后台 Agent 执行系统支持真正的并行任务执行。当调用 `task` 工具并设置 `run_in_background=true` 时，系统会：

1. 创建新的 OpenCode 会话作为子 Agent
2. 在新会话中异步执行提示词
3. 立即返回任务 ID，不等待执行完成
4. 父会话可以继续执行其他操作

```typescript
// src/tools/delegate-task/background-task.ts:27-42
const task = await manager.launch({
  description: args.description,
  prompt: effectivePrompt,
  agent: agentToUse,
  parentSessionID: parentContext.sessionID,
  parentMessageID: parentContext.messageID,
  model: categoryModel,
  fallbackChain,
  skills: args.load_skills.length > 0 ? args.load_skills : undefined,
  skillContent: systemContent,
  category: args.category,
})
```

这种设计允许同时启动多个探索任务（如并行搜索代码库的不同部分），大幅提升效率。

### 2. 并发控制 (5/model)

系统通过 `ConcurrencyManager` 实现精细的并发控制：

```typescript
// src/features/background-agent/concurrency.ts:24-39
getConcurrencyLimit(model: string): number {
  const modelLimit = this.config?.modelConcurrency?.[model]
  if (modelLimit !== undefined) {
    return modelLimit === 0 ? Infinity : modelLimit
  }
  const provider = model.split('/')[0]
  const providerLimit = this.config?.providerConcurrency?.[provider]
  if (providerLimit !== undefined) {
    return providerLimit === 0 ? Infinity : providerLimit
  }
  const defaultLimit = this.config?.defaultConcurrency
  if (defaultLimit !== undefined) {
    return defaultLimit === 0 ? Infinity : defaultLimit
  }
  return 5  // 默认限制
}
```

并发控制采用 FIFO 队列机制：
- 每个模型/提供商有独立的并发槽位
- 默认每个模型最多 5 个并发任务
- 超出限制的任务进入队列等待
- 任务完成后自动释放槽位给队列中的下一个任务

配置可在 `.opencode/oh-my-opencode.jsonc` 中自定义：

```jsonc
{
  "background_task": {
    "defaultConcurrency": 5,
    "modelConcurrency": {
      "anthropic/claude-opus-4-6": 3,
      "openai/gpt-5.4": 5
    },
    "providerConcurrency": {
      "anthropic": 5,
      "openai": 8
    }
  }
}
```

### 3. Task ID 管理

每个后台任务都有唯一的任务 ID，格式为 `bg_` 前缀加 8 位随机字符：

```typescript
// src/features/background-agent/manager.ts:295-296
const task: BackgroundTask = {
  id: `bg_${crypto.randomUUID().slice(0, 8)}`,
  status: "pending",
  // ...
}
```

任务 ID 用于：
- 通过 `background_output` 工具查询任务状态和结果
- 通过 `background_cancel` 工具取消正在执行的任务
- 在任务历史中追踪任务执行记录
- 父会话接收任务完成通知

### 4. 结果收集

后台任务的结果收集采用异步拉取模式：

```typescript
// src/features/background-agent/types.ts:29-69
export interface BackgroundTask {
  id: string
  sessionID?: string
  status: BackgroundTaskStatus
  result?: string
  error?: string
  progress?: TaskProgress
  // ...
}
```

任务完成后，系统会：
1. 更新任务状态为 `completed` 或 `error`
2. 记录完成时间和结果/错误信息
3. 通知父会话任务已完成
4. 保留任务数据一段时间（默认 30 分钟）供查询

用户通过 `background_output` 工具获取结果：

```typescript
// 使用示例
background_output({
  task_id: "bg_abc12345",
  full_session: true,
  include_thinking: false
})
```

---

## 后台执行流程

后台 Agent 执行的完整流程如下：

1. **任务发起**：用户调用 `task` 工具，设置 `run_in_background=true`
2. **参数解析**：系统解析 category/subagent_type、skills、model 等参数
3. **并发检查**：`ConcurrencyManager` 检查当前模型并发槽位是否可用
4. **任务排队**：如果槽位已满，任务进入 FIFO 队列等待
5. **会话创建**：获取槽位后，创建新的 OpenCode 会话作为子 Agent
6. **提示词注入**：向子会话发送系统提示词和用户提示词
7. **状态跟踪**：任务状态变为 `running`，开始轮询监控
8. **执行监控**：系统每 3 秒轮询一次任务状态，检测完成或错误
9. **完成处理**：任务完成后，更新状态、释放槽位、通知父会话
10. **结果获取**：用户通过 `background_output` 获取执行结果

---

## 流程/架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                        后台 Agent 执行流程                           │
└─────────────────────────────────────────────────────────────────────┘

  用户调用 task()
       │
       ▼
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  参数解析     │────▶│  并发检查     │────▶│  任务排队     │
│  (category)  │     │ (Concurrency)│     │   (Queue)    │
└──────────────┘     └──────────────┘     └──────────────┘
                                                │
                    ┌───────────────────────────┘
                    ▼
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  会话创建     │────▶│  提示词注入   │────▶│  异步执行     │
│ (session.create)   │ (promptAsync)│     │ (fire-and-forget)
└──────────────┘     └──────────────┘     └──────────────┘
                                                │
                    ┌───────────────────────────┘
                    ▼
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  状态轮询     │────▶│  完成检测     │────▶│  结果通知     │
│ (3s interval)│     │ (idle/event) │     │ (notify parent)
└──────────────┘     └──────────────┘     └──────────────┘
                                                │
                    ┌───────────────────────────┘
                    ▼
┌──────────────┐     ┌──────────────┐
│  结果获取     │◀────│  资源清理     │
│(background_output)  │ (cleanup)    │
└──────────────┘     └──────────────┘


┌─────────────────────────────────────────────────────────────────────┐
│                      并发控制模型 (Concurrency)                       │
└─────────────────────────────────────────────────────────────────────┘

                    ┌─────────────────┐
                    │   任务请求       │
                    └────────┬────────┘
                             ▼
                    ┌─────────────────┐
                    │ 获取并发槽位?    │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              │ 是            │              │ 否
              ▼              │              ▼
    ┌─────────────────┐      │    ┌─────────────────┐
    │  立即执行任务    │      │    │  加入等待队列    │
    │ (acquire slot)  │      │    │   (FIFO Queue)  │
    └────────┬────────┘      │    └────────┬────────┘
             │               │             │
             ▼               │             │
    ┌─────────────────┐      │             │ 槽位释放
    │  释放并发槽位    │◀─────┴─────────────┘
    │ (release slot)  │
    └─────────────────┘


┌─────────────────────────────────────────────────────────────────────┐
│                      任务状态机 (State Machine)                       │
└─────────────────────────────────────────────────────────────────────┘

    ┌─────────┐
    │  pending │◀────────── 任务创建
    └────┬────┘
         │ acquire slot
         ▼
    ┌─────────┐
    │ running  │◀────────── 开始执行
    └────┬────┘
         │
    ┌────┴────┬────────┬────────┐
    ▼         ▼        ▼        ▼
┌───────┐ ┌───────┐ ┌───────┐ ┌─────────┐
│completed│ │ error │ │cancelled│ │interrupt│
└───────┘ └───────┘ └───────┘ └─────────┘
    │         │        │        │
    └─────────┴────────┴────────┘
                   │
                   ▼
            ┌─────────────┐
            │  通知父会话  │
            │  资源清理    │
            └─────────────┘
```

---

## 关键代码片段

### 片段 1: 并发管理器实现

```typescript
// src/features/background-agent/concurrency.ts:41-69
async acquire(model: string): Promise<void> {
  const limit = this.getConcurrencyLimit(model)
  if (limit === Infinity) {
    return
  }

  const current = this.counts.get(model) ?? 0
  if (current < limit) {
    this.counts.set(model, current + 1)
    return
  }

  return new Promise<void>((resolve, reject) => {
    const queue = this.queues.get(model) ?? []
    const entry: QueueEntry = {
      resolve: () => {
        if (entry.settled) return
        entry.settled = true
        resolve()
      },
      rawReject: reject,
      settled: false,
    }
    queue.push(entry)
    this.queues.set(model, queue)
  })
}
```

### 片段 2: 任务启动流程

```typescript
// src/features/background-agent/manager.ts:272-358
async launch(input: LaunchInput): Promise<BackgroundTask> {
  // 并发槽位预留
  const spawnReservation = await this.reserveSubagentSpawn(input.parentSessionID)
  
  // 创建任务对象
  const task: BackgroundTask = {
    id: `bg_${crypto.randomUUID().slice(0, 8)}`,
    status: "pending",
    queuedAt: new Date(),
    // ...
  }
  
  // 加入队列
  const key = this.getConcurrencyKeyFromInput(input)
  const queue = this.queuesByKey.get(key) ?? []
  queue.push({ task, input })
  this.queuesByKey.set(key, queue)
  
  // 触发处理
  this.processKey(key)
  
  return { ...task }
}
```

### 片段 3: 僵死任务检测

```typescript
// src/features/background-agent/task-poller.ts:100-163
export async function checkAndInterruptStaleTasks(args: {
  tasks: Iterable<BackgroundTask>
  client: OpencodeClient
  config: BackgroundTaskConfig | undefined
  concurrencyManager: ConcurrencyManager
  notifyParentSession: (task: BackgroundTask) => Promise<void>
}): Promise<void> {
  const staleTimeoutMs = config?.staleTimeoutMs ?? DEFAULT_STALE_TIMEOUT_MS
  
  for (const task of tasks) {
    if (task.status !== "running") continue
    
    const timeSinceLastUpdate = now - task.progress.lastUpdate.getTime()
    if (timeSinceLastUpdate <= staleTimeoutMs) continue
    
    // 标记为取消并释放资源
    task.status = "cancelled"
    task.error = `Stale timeout (no activity for ${staleMinutes}min)`
    task.completedAt = new Date()
    
    if (task.concurrencyKey) {
      concurrencyManager.release(task.concurrencyKey)
    }
    
    await notifyParentSession(task)
  }
}
```

---

## 依赖关系

后台 Agent 执行系统依赖以下组件：

**1. OpenCode SDK**：通过 `client.session.create()` 和 `client.session.promptAsync()` 创建会话和发送提示词

**2. Task Toast Manager**：`src/features/task-toast-manager` 提供任务进度通知 UI

**3. Session Category Registry**：`src/shared/session-category-registry.ts` 跟踪会话分类信息

**4. Model Fallback 系统**：`src/hooks/model-fallback/` 提供模型故障转移支持

**5. Tool Metadata Store**：`src/features/tool-metadata-store` 存储工具执行元数据

**6. Claude Code Session State**：`src/features/claude-code-session-state` 跟踪子 Agent 会话

---

## 实战示例

### 示例 1: 并行探索

场景：需要同时搜索代码库的多个模块以了解架构

```typescript
// 同时启动多个探索任务
const tasks = await Promise.all([
  task({
    subagent_type: "explore",
    description: "探索 auth 模块",
    prompt: "探索 src/auth/ 目录，了解认证系统的架构和关键文件",
    run_in_background: true,
    load_skills: []
  }),
  task({
    subagent_type: "explore",
    description: "探索 database 模块",
    prompt: "探索 src/database/ 目录，了解数据库访问层的设计",
    run_in_background: true,
    load_skills: []
  }),
  task({
    subagent_type: "explore",
    description: "探索 api 路由",
    prompt: "探索 src/api/ 目录，了解 REST API 的路由结构",
    run_in_background: true,
    load_skills: []
  })
])

// 获取任务 ID
const taskIds = tasks.map(t => t.task_id)

// 稍后获取结果
for (const taskId of taskIds) {
  const result = await background_output({ task_id: taskId })
  console.log(`任务 ${taskId} 结果:`, result)
}
```

这个示例展示了如何利用后台执行并行探索代码库的不同部分，每个探索任务在独立的 Agent 会话中执行，互不阻塞。

### 示例 2: 并发文档编写

场景：需要为多个 API 端点同时生成文档

```typescript
// 定义需要文档化的端点
const endpoints = [
  { name: "用户登录", path: "/api/auth/login" },
  { name: "用户注册", path: "/api/auth/register" },
  { name: "获取用户信息", path: "/api/user/profile" },
  { name: "更新用户设置", path: "/api/user/settings" }
]

// 为每个端点启动后台文档编写任务
const docTasks = await Promise.all(
  endpoints.map(ep =>
    task({
      category: "writing",
      description: `编写 ${ep.name} 文档`,
      prompt: `为 ${ep.path} 端点编写详细的 API 文档，包括请求参数、响应格式和示例`,
      run_in_background: true,
      load_skills: ["documentation"]
    })
  )
)

// 等待所有任务完成并收集结果
const docs = await Promise.all(
  docTasks.map(t =>
    background_output({
      task_id: t.task_id,
      block: true,
      timeout: 120000
    })
  )
)

// 合并所有文档
const combinedDocs = docs.join("\n\n---\n\n")
```

这个示例展示了如何使用 `category="writing"` 启动多个文档编写任务，利用后台执行实现并行处理，大幅提升文档编写效率。

---

## 交叉引用

- 参见：[Agent 工具](../04-工具系统/04-Agent 工具.md) - 了解 `task` 和 `call_omo_agent` 工具的详细用法
- 参见：[会话持久化](../01-Harness 架构/06-会话持久化.md) - 了解后台任务会话的生命周期管理
- 参见：[Task 系统](../02-核心 Agent/03-Agent 编排.md) - 了解 Agent 编排和任务委托机制
- 参见：[并发配置](../06-configuration/background-task-config.md) - 了解如何配置并发限制和超时参数
- 参见：[Subagent 类型](../02-core-agents/subagent-types.md) - 了解可用的 subagent_type 及其适用场景
