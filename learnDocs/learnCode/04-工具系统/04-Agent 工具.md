# Agent 工具：delegate-task, call-omo-agent

> 所属模块：04-tools-system | 三层模型：Layer 3 | 优先级：P2

---

## 架构位置

```
┌─────────────────────────────────────────────────────────────────┐
│                        Layer 3: Tools                           │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              Agent Invocation Tools                      │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │   │
│  │  │ delegate-task│  │call_omo_agent│  │background_*  │   │   │
│  │  │   (task)     │  │              │  │              │   │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘   │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                     Layer 2: Features                           │
│         BackgroundManager │ SessionManager │ TaskToast          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                     Layer 1: Agents                             │
│    Sisyphus-Junior │ Explore │ Librarian │ Oracle │ Hephaestus  │
└─────────────────────────────────────────────────────────────────┘
```

Agent 调用工具位于工具层（Layer 3），是连接主 Agent 与各类专业子 Agent 的桥梁。它们负责将任务委派给合适的子 Agent，并管理任务的执行生命周期。

---

## 核心职责

Agent 调用工具承担着以下核心职责：

**任务委派与路由**：根据任务类型和领域，将工作分配给最适合的子 Agent。通过 Category 系统或直接的 Agent 类型指定，确保任务由具备相应专长的 Agent 处理。

**执行模式管理**：支持同步（sync）和异步（background）两种执行模式。同步模式阻塞等待结果返回，适用于需要立即响应的任务；异步模式启动后台任务，适用于并行探索和长时间运行的任务。

**模型解析与回退**：自动解析模型配置，处理模型可用性检查，并在主模型不可用时执行智能回退（fallback）策略，确保任务能够可靠完成。

**会话连续性维护**：支持通过 session_id 继续之前的会话，保持上下文连贯性，避免重复传递大量背景信息，节省 token 消耗。

**技能注入**：支持加载特定技能（skills）到委派 Agent 的上下文中，为任务提供额外的领域知识和工具能力。

---

## 代码索引 (Code Index)

| 功能点 | 文件路径 | 行号 | 函数/类 | 说明 |
|--------|----------|------|---------|------|
| Task 工具创建 | src/tools/delegate-task/tools.ts | 28-259 | createDelegateTask | 主入口，处理参数验证和路由 |
| Category 配置解析 | src/tools/delegate-task/categories.ts | 27-77 | resolveCategoryConfig | 解析 Category 模型配置 |
| Category 执行解析 | src/tools/delegate-task/category-resolver.ts | 40-256 | resolveCategoryExecution | 完整的 Category 路由逻辑 |
| Subagent 解析 | src/tools/delegate-task/subagent-resolver.ts | 18-224 | resolveSubagentExecution | 直接 Agent 调用解析 |
| 后台任务执行 | src/tools/delegate-task/background-task.ts | 13-108 | executeBackgroundTask | 异步任务启动与管理 |
| 同步任务执行 | src/tools/delegate-task/sync-task.ts | 14-199 | executeSyncTask | 同步任务执行与轮询 |
| Call OMO Agent | src/tools/call-omo-agent/tools.ts | 59-132 | createCallOmoAgent | 直接 Agent 调用工具 |
| 后台执行器 | src/tools/call-omo-agent/background-executor.ts | 12-96 | executeBackground | explore/librarian 后台执行 |
| 同步执行器 | src/tools/call-omo-agent/sync-executor.ts | 37-124 | executeSync | explore/librarian 同步执行 |
| 默认 Category 配置 | src/tools/delegate-task/constants.ts | 288-297 | DEFAULT_CATEGORIES | 8 个内置 Category 定义 |

---

## Agent 调用工具

### 1. delegate-task (task() 工具)

`task()` 是 oh-my-opencode 的核心任务委派工具，支持两种调用模式：

**Category 模式**：通过 `category` 参数指定任务领域，系统自动选择合适的模型和 Agent。

```typescript
// src/tools/delegate-task/tools.ts:121-123
if (args.category) {
  args.subagent_type = SISYPHUS_JUNIOR_AGENT
}
```

当指定 category 时，系统会自动使用 Sisyphus-Junior Agent，并根据 Category 配置解析对应的模型。

**Direct Agent 模式**：通过 `subagent_type` 直接指定 Agent 类型（如 explore、librarian、oracle 等）。

```typescript
// src/tools/delegate-task/tools.ts:231-239
} else {
  const resolution = await resolveSubagentExecution(args, options, parentContext.agent, categoryExamples)
  if (resolution.error) {
    return resolution.error
  }
  agentToUse = resolution.agentToUse
  categoryModel = resolution.categoryModel
  fallbackChain = resolution.fallbackChain
}
```

**关键参数**：
- `category`: 任务领域（如 quick、deep、visual-engineering 等）
- `subagent_type`: 直接指定 Agent 类型
- `run_in_background`: 执行模式（true=异步，false=同步）
- `load_skills`: 要加载的技能列表
- `session_id`: 继续已有会话

### 2. call-omo-agent

`call_omo_agent` 是轻量级的 Agent 调用工具，仅支持直接调用 explore 和 librarian 两个 Agent。

```typescript
// src/tools/call-omo-agent/constants.ts:1-9
export const ALLOWED_AGENTS = [
  "explore",
  "librarian",
  "oracle",
  "hephaestus",
  "metis",
  "momus",
  "multimodal-looker",
] as const
```

与 `task()` 的区别：
- 不支持 Category 路由
- 不支持技能加载（load_skills）
- 更轻量，适合快速代码搜索和文档查询

```typescript
// src/tools/call-omo-agent/tools.ts:84-130
async execute(args: CallOmoAgentArgs, toolContext) {
  // ...验证 Agent 类型...
  
  if (args.run_in_background) {
    return await executeBackground(args, toolCtx, backgroundManager, ctx.client, fallbackChain, resolvedModel)
  }
  
  // 同步执行...
  return await executeSync(args, toolCtx, ctx, undefined, fallbackChain, spawnReservation, resolvedModel)
}
```

### 3. 后台任务管理

后台任务通过 `BackgroundManager` 进行管理，支持以下操作：

**启动后台任务**：
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

**查询任务结果**：使用 `background_output` 工具查询后台任务状态和结果。

```typescript
// 参数说明
{
  task_id: string           // 任务 ID
  block?: boolean           // 是否阻塞等待
  timeout?: number          // 超时时间
  full_session?: boolean    // 返回完整会话消息
  include_thinking?: boolean // 包含思考过程
}
```

**取消任务**：使用 `background_cancel` 工具取消正在运行的后台任务。

---

## Agent 委派流程

Agent 委派的完整执行流程如下：

1. **参数验证**：检查 category 和 subagent_type 的互斥性，验证必需参数
2. **技能解析**：加载指定的 skills，注入到 Agent 上下文中
3. **父上下文解析**：获取父会话的模型、Agent 类型等信息
4. **Category/Agent 解析**：
   - Category 模式：解析 Category 配置，确定模型和 Agent
   - Direct 模式：验证 Agent 可用性，解析模型配置
5. **模型回退链构建**：根据配置构建模型回退链
6. **系统提示构建**：组合技能内容、Category 追加提示等
7. **执行模式选择**：
   - 后台模式：调用 BackgroundManager.launch()
   - 同步模式：创建会话 → 发送提示 → 轮询完成 → 获取结果
8. **结果返回**：返回任务 ID（后台）或执行结果（同步）

---

## 流程/架构图

```
┌────────────────────────────────────────────────────────────────────┐
│                        Agent Delegation Flow                       │
└────────────────────────────────────────────────────────────────────┘

  User Request
       │
       ▼
┌──────────────┐
│  Parameter   │
│  Validation  │
└──────────────┘
       │
       ▼
┌──────────────┐     ┌─────────────────┐
│   Category   │────▶│ Category Config │
│   Specified? │     │   Resolution    │
└──────────────┘     └─────────────────┘
       │ No                    │
       ▼                       ▼
┌──────────────┐     ┌─────────────────┐
│   Direct     │────▶│  Subagent       │
│   Agent?     │     │  Resolution     │
└──────────────┘     └─────────────────┘
       │
       ▼
┌──────────────┐
│ Skill Loading│
└──────────────┘
       │
       ▼
┌──────────────┐
│ Model        │
│ Resolution   │
└──────────────┘
       │
       ▼
┌─────────────────────────────────────────┐
│         Execution Mode Branch           │
│  ┌─────────────┐    ┌─────────────┐    │
│  │ Background  │    │   Sync      │    │
│  │ (async)     │    │ (blocking)  │    │
│  └─────────────┘    └─────────────┘    │
│         │                  │           │
│         ▼                  ▼           │
│  ┌─────────────┐    ┌─────────────┐    │
│  │ Launch via  │    │ Create      │    │
│  │ Background  │    │ Session     │    │
│  │ Manager     │    │             │    │
│  └─────────────┘    └─────────────┘    │
│         │                  │           │
│         ▼                  ▼           │
│  ┌─────────────┐    ┌─────────────┐    │
│  │ Return      │    │ Send Prompt │    │
│  │ Task ID     │    │             │    │
│  └─────────────┘    └─────────────┘    │
│         │                  │           │
│         ▼                  ▼           │
│  ┌─────────────┐    ┌─────────────┐    │
│  │ Poll via    │    │ Poll Until  │    │
│  │ background  │    │ Idle        │    │
│  │ _output     │    │             │    │
│  └─────────────┘    └─────────────┘    │
│         │                  │           │
│         ▼                  ▼           │
│  ┌─────────────┐    ┌─────────────┐    │
│  │ Return      │    │ Fetch &     │    │
│  │ Result      │    │ Return      │    │
│  └─────────────┘    └─────────────┘    │
└─────────────────────────────────────────┘
```

---

## 关键代码片段

### 片段 1：Category 模型解析

```typescript
// src/tools/delegate-task/categories.ts:54-67
// Model priority for categories: user override > category default > system default
// Categories have explicit models - no inheritance from parent session
const model = resolveModel({
  userModel: userConfig?.model,
  inheritedModel: defaultConfig?.model, // Category's built-in model takes precedence over system default
  systemDefault: systemDefaultModel,
})
const isUserConfiguredModel = normalizeModel(userConfig?.model) !== undefined
const config: CategoryConfig = {
  ...defaultConfig,
  ...userConfig,
  model,
  variant: userConfig?.variant ?? defaultConfig?.variant,
}
```

### 片段 2：同步任务执行流程

```typescript
// src/tools/delegate-task/sync-task.ts:48-82
const createSessionResult = await deps.createSyncSession(client, {
  parentSessionID: parentContext.sessionID,
  agentToUse,
  description: args.description,
  defaultDirectory: directory,
})

if (!createSessionResult.ok) {
  spawnReservation?.rollback()
  return createSessionResult.error
}

const sessionID = createSessionResult.sessionID
spawnReservation?.commit()
syncSessionID = sessionID
subagentSessions.add(sessionID)
syncSubagentSessions.add(sessionID)
setSessionAgent(sessionID, agentToUse)
setSessionFallbackChain(sessionID, fallbackChain)

if (args.category) {
  SessionCategoryRegistry.register(sessionID, args.category)
}
```

### 片段 3：后台任务启动

```typescript
// src/tools/call-omo-agent/background-executor.ts:47-57
const task = await manager.launch({
  description: args.description,
  prompt: args.prompt,
  agent: args.subagent_type,
  parentSessionID: toolContext.sessionID,
  parentMessageID: toolContext.messageID,
  parentAgent,
  parentTools: getSessionTools(toolContext.sessionID),
  model,
  fallbackChain,
})
```

---

## 依赖关系

Agent 调用工具依赖以下核心组件：

**BackgroundManager** (`src/features/background-agent/`)：管理后台任务的整个生命周期，包括任务启动、状态跟踪、结果通知等。

**Category 配置系统** (`src/tools/delegate-task/categories.ts`, `constants.ts`)：定义 8 个内置 Category 及其对应的模型配置。

**模型解析与回退** (`src/shared/model-resolver.ts`, `fallback-chain-from-models.ts`)：处理模型字符串解析、可用性检查和回退链构建。

**Session 状态管理** (`src/features/claude-code-session-state.ts`)：跟踪子 Agent 会话状态，管理会话 Agent 类型和回退链。

**Skill 系统** (`src/tools/delegate-task/skill-resolver.ts`)：解析和加载技能内容，注入到 Agent 上下文中。

---

## 实战示例

### 示例 1: 前端任务委派

使用 Category 模式委派前端 UI 任务：

```typescript
// 委派一个 React 组件开发任务
task({
  category: "visual-engineering",
  load_skills: ["frontend-ui-ux"],
  description: "Create login form component",
  prompt: `Create a React login form component with:
1. Email and password inputs
2. Form validation
3. Submit button with loading state
4. Error message display

Use the project's design system and follow existing component patterns.`,
  run_in_background: false
})
```

这个示例：
- 使用 `visual-engineering` Category，自动路由到 Gemini Pro 模型
- 加载 `frontend-ui-ux` 技能，注入前端开发最佳实践
- 同步执行，等待结果返回

### 示例 2: 后台 Agent 执行

并行启动多个探索任务：

```typescript
// 并行搜索代码库中的模式
const task1 = call_omo_agent({
  subagent_type: "explore",
  description: "Find auth patterns",
  prompt: "Search for authentication-related files and patterns in the codebase. Look for: login, logout, session management, JWT handling.",
  run_in_background: true
})

const task2 = call_omo_agent({
  subagent_type: "librarian",
  description: "Research API patterns",
  prompt: "Search for REST API design patterns and best practices for user management endpoints.",
  run_in_background: true
})

// 稍后获取结果
const result1 = background_output({ task_id: task1.task_id })
const result2 = background_output({ task_id: task2.task_id })
```

这个示例：
- 同时启动 explore 和 librarian 两个后台任务
- 使用 `run_in_background: true` 实现并行执行
- 通过 `background_output` 查询任务结果

---

## 交叉引用

- 参见：[Agent 编排](../02-核心 Agent/03-Agent 编排.md) - 了解 Agent 之间的协作机制
- 参见：[主要 Agent](../02-核心 Agent/01-主要 Agent.md) - 了解 Sisyphus、Hephaestus 等主要 Agent 的职责
- 参见：[Category 路由](../02-core-agents/category-routing.md) - 深入了解 Category 系统的模型路由机制
- 参见：[后台任务管理](../03-features/background-tasks.md) - 了解 BackgroundManager 的详细实现
- 参见：[技能系统](../05-skill./03-Skill 系统.md) - 了解技能加载和注入机制
- 参见：[模型回退](../03-features/model-fallback.md) - 了解模型不可用时的回退策略
