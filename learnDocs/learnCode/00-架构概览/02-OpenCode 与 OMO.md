# OpenCode 底座 vs OMO 编排层

> 所属模块：00-architecture-overview | 三层模型：Layer 3 | 优先级：P0

---

## 架构位置

```
┌─────────────────────────────────────────────────────────────────┐
│                        Layer 4: User                            │
│                     (开发者 / AI Agent)                          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                     Layer 3: Harness                            │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              OpenCode Base (底座)                        │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐  │   │
│  │  │ Plugin API  │  │  Tool API   │  │   Hook System   │  │   │
│  │  └─────────────┘  └─────────────┘  └─────────────────┘  │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              │                                  │
│                              ▼                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │         OMO Orchestration Layer (编排层)                 │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐  │   │
│  │  │   Agents    │  │ Categories  │  │  Background     │  │   │
│  │  │  (11个)     │  │   (8个)     │  │    Tasks        │  │   │
│  │  └─────────────┘  └─────────────┘  └─────────────────┘  │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐  │   │
│  │  │ IntentGate  │  │ Hashline    │  │   Todo          │  │   │
│  │  │             │  │   Edit      │  │   Enforcer      │  │   │
│  │  └─────────────┘  └─────────────┘  └─────────────────┘  │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                     Layer 2: Model API                          │
│              (Anthropic / OpenAI / Google / etc.)               │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                     Layer 1: Infrastructure                     │
│                      (Compute / Network)                        │
└─────────────────────────────────────────────────────────────────┘
```

---

## 核心职责

OpenCode 是一个优秀的 AI 编程工具底座，提供了插件系统、工具 API 和生命周期钩子。它解决了"如何让 AI 写代码"的问题。

OMO (Oh My OpenAgent) 是在 OpenCode 之上构建的**编排增强层 (Harness Enhancement Layer)**，解决的是"如何让多个 AI 像一个团队一样协作"的问题。

两者的边界清晰：

- **OpenCode 负责**：插件加载、工具执行、会话管理、基础 UI
- **OMO 负责**：智能任务分发、多模型编排、上下文管理、质量保障

OMO 不是框架 (Framework)，因为它不强制你改变开发方式；它是增强层 (Enhancement Layer)，像安全带和导航系统一样，在 OpenCode 的基础上增加安全性和效率。

---

## 代码索引 (Code Index)

| 功能点 | 文件路径 | 行号 | 函数/类 | 说明 |
|--------|----------|------|---------|------|
| task() 工具实现 | `src/tools/delegate-task/tools.ts` | 28-259 | `createDelegateTask()` | 任务委派核心工厂函数 |
| Category 路由系统 | `src/tools/delegate-task/categories.ts` | 27-77 | `resolveCategoryConfig()` | 8 个内置类别解析 |
| 后台任务执行 | `src/tools/delegate-task/background-task.ts` | 13-108 | `executeBackgroundTask()` | 异步任务启动与管理 |
| Hashline 编辑工具 | `src/tools/hashline-edit/tools.ts` | 14-42 | `createHashlineEditTool()` | 基于哈希锚定的编辑 |
| Todo 强制执行 | `src/hooks/todo-continuation-enforcer/todo.ts` | 3-11 | `getIncompleteCount()` | 检查未完成的 todo |
| 上下文压缩 | `src/hooks/preemptive-compaction.ts` | 65-100 | `createPreemptiveCompactionHook()` | 预emptive 压缩钩子 |
| 压缩后上下文注入 | `src/hooks/compaction-context-injector/compaction-context-prompt.ts` | 6-56 | `COMPACTION_CONTEXT_PROMPT` | 压缩后恢复上下文 |
| Intent Gate | `src/agents/sisyphus/default.ts` | 28-82 | `buildTaskManagementSection()` | 意图分类与任务管理 |
| 8 个内置类别定义 | `src/tools/delegate-task/constants.ts` | 288-297 | `DEFAULT_CATEGORIES` | 类别到模型的映射 |
| 插件入口 | `src/index.ts` | 20-111 | `OhMyOpenCodePlugin()` | 插件初始化流程 |

---

## OpenCode vs OMO 对比表

| OpenCode 原生 | OMO 编排增强 |
|---------------|-------------|
| 单模型执行 | 多模型智能路由 (Category 系统) |
| 顺序任务处理 | 并行后台任务 (Background Tasks) |
| 基础文件编辑 | Hashline 锚定编辑 (防冲突) |
| 简单会话管理 | 上下文压缩与恢复 (Compaction) |
| 无内置 Agent 系统 | 11 个专业 Agent (Sisyphus, Hephaestus 等) |
| 直接执行用户指令 | Intent Gate 意图分类 |
| 无强制任务追踪 | Todo 强制执行机制 |
| 标准工具调用 | 工具调用后增强 (LSP, AST-Grep) |
| 单会话上下文 | 跨会话状态保持 (Session Recovery) |
| 基础错误处理 | 多层降级策略 (Model Fallback) |

---

## OMO 的 8 大编排增强

### 1. task() 工具系统

OMO 的核心编排工具，允许 Agent 委派任务给其他 Agent。

```typescript
// src/tools/delegate-task/tools.ts:67-73
// 正确使用 category 委派
`task(category="quick", load_skills=[], description="Fix type error", prompt="...", run_in_background=false)`

// 正确使用 subagent_type 直接调用
`task(subagent_type="explore", load_skills=[], description="Find patterns", prompt="...", run_in_background=true)`
```

task() 支持两种模式：
- **同步模式** (`run_in_background=false`): 等待子 Agent 完成并返回结果
- **后台模式** (`run_in_background=true`): 启动并行任务，返回 task_id 供后续查询

### 2. Category 路由系统

8 个内置类别自动映射到最适合的模型：

```typescript
// src/tools/delegate-task/constants.ts:288-297
export const DEFAULT_CATEGORIES: Record<string, CategoryConfig> = {
  "visual-engineering": { model: "google/gemini-3.1-pro", variant: "high" },
  ultrabrain: { model: "openai/gpt-5.4", variant: "xhigh" },
  deep: { model: "openai/gpt-5.3-codex", variant: "medium" },
  artistry: { model: "google/gemini-3.1-pro", variant: "high" },
  quick: { model: "openai/gpt-5.4-mini" },
  "unspecified-low": { model: "anthropic/claude-sonnet-4-6" },
  "unspecified-high": { model: "anthropic/claude-opus-4-6", variant: "max" },
  writing: { model: "kimi-for-coding/k2p5" },
}
```

每个类别都有专门的系统提示词注入，确保 Agent 以正确的思维模式工作。

### 3. Agent 委派机制

11 个专业 Agent 分工协作：

| Agent | 职责 | 推荐模型 |
|-------|------|----------|
| Sisyphus | 主编排器 | Claude Opus / Kimi K2.5 |
| Hephaestus | 深度自主工作 | GPT-5.4 |
| Prometheus | 战略规划 | Claude Opus |
| Atlas | Todo 编排执行 | Claude Sonnet |
| Oracle | 架构咨询 | GPT-5.4 |
| Librarian | 文档搜索 | MiniMax |
| Explore | 代码库搜索 | Grok |
| Metis | 计划顾问 | Claude Opus |
| Momus | 计划审查 | GPT-5.4 |
| Multimodal-Looker | 视觉分析 | GPT-5.3 |
| Sisyphus-Junior | 类别任务执行 | 可配置 |

### 4. 后台任务执行

```typescript
// src/tools/delegate-task/background-task.ts:27-42
const task = await manager.launch({
  description: args.description,
  prompt: effectivePrompt,
  agent: agentToUse,
  parentSessionID: parentContext.sessionID,
  parentMessageID: parentContext.messageID,
  parentModel: parentContext.model,
  parentAgent: parentContext.agent,
  model: categoryModel,
  fallbackChain,
  skills: args.load_skills.length > 0 ? args.load_skills : undefined,
})
```

后台任务允许同时运行 5+ 个 Agent，像真正的开发团队一样并行工作。

### 5. Todo 强制执行

```typescript
// src/hooks/todo-continuation-enforcer/todo.ts:3-11
export function getIncompleteCount(todos: Todo[]): number {
  return todos.filter(
    (todo) =>
      todo.status !== "completed"
      && todo.status !== "cancelled"
      && todo.status !== "blocked"
      && todo.status !== "deleted",
  ).length
}
```

当检测到未完成的 todo 时，系统会在 session.idle 事件后注入 continuation prompt，强制 Agent 继续工作直到任务完成。

### 6. 上下文压缩恢复

当上下文窗口接近限制 (78% 阈值) 时，OMO 会触发 preemptive compaction：

```typescript
// src/hooks/preemptive-compaction.ts:11-13
const PREEMPTIVE_COMPACTION_TIMEOUT_MS = 120_000
const PREEMPTIVE_COMPACTION_THRESHOLD = 0.78
const PREEMPTIVE_COMPACTION_COOLDOWN_MS = 60_000
```

压缩后通过 `compaction-context-injector` 恢复关键上下文，包括：
- 用户原始请求
- 已完成的工作
- 剩余任务
- 活跃工作上下文
- 委派的 Agent Sessions

### 7. Intent Gate

在 Sisyphus 的 prompt 中内置了意图分类逻辑：

```typescript
// src/agents/sisyphus/default.ts:28-82
// Task Management Section 包含意图判断逻辑
// 自动识别用户请求是研究、实现、调查还是修复
```

Intent Gate 确保 Agent 先理解用户真正想要什么，而不是字面执行指令。

### 8. Hash-anchored Edits

```typescript
// src/tools/hashline-edit/tools.ts:14-42
export function createHashlineEditTool(ctx?: PluginContext): ToolDefinition {
  return tool({
    description: HASHLINE_EDIT_DESCRIPTION,
    args: {
      filePath: tool.schema.string().describe("Absolute path to the file to edit"),
      edits: tool.schema.array(/* LINE#ID format */),
    },
    execute: async (args: HashlineEditArgs, context: ToolContext) => 
      executeHashlineEditTool(args, context, ctx),
  })
}
```

每行代码都有内容哈希 (`42#VK`)，编辑时验证哈希匹配，防止基于过时行号的错误修改。Grok Code Fast 1 的成功率从 6.7% 提升到 68.3%。

---

## 流程/架构图

```
User Request
    │
    ▼
┌─────────────────────────────────────┐
│         Intent Gate                 │  ← OMO: 分类真实意图
│   (研究/实现/调查/修复)              │
└─────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────┐
│         Sisyphus                    │  ← OMO: 主编排器
│   (计划 + 委派 + 监控)               │
└─────────────────────────────────────┘
    │
    ├──────────────────┬──────────────────┬──────────────────┐
    ▼                  ▼                  ▼                  ▼
┌─────────┐      ┌─────────┐      ┌─────────┐      ┌─────────┐
│Prometheus│      │  Atlas  │      │ Oracle  │      │Explore/ │
│(规划)   │      │(执行)   │      │(咨询)   │      │Librarian│
└─────────┘      └─────────┘      └─────────┘      └─────────┘
    │                  │                  │                  │
    ▼                  ▼                  ▼                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                     Category Router                              │
│  ┌──────────┬──────────┬──────────┬──────────┬──────────┐       │
│  │ visual   │ultrabrain│  deep    │  quick   │ writing  │       │
│  │ gemini   │ gpt-5.4  │gpt-5.3   │gpt-5.4-m │  kimi    │       │
│  └──────────┴──────────┴──────────┴──────────┴──────────┘       │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────┐
│         OpenCode Base               │  ← OpenCode: 执行层
│   (Tool API + Hook System)          │
└─────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────┐
│         Model API                   │
│   (Claude/GPT/Gemini/etc.)          │
└─────────────────────────────────────┘
```

---

## 关键代码片段

### 片段 1: task() 工具的 category 与 subagent_type 互斥校验

```typescript
// src/tools/delegate-task/tools.ts:112-123
if (args.category && args.subagent_type) {
  throw new Error(
    `Invalid arguments: 'category' and 'subagent_type' are mutually exclusive. ` +
    `Provide EXACTLY ONE.\n` +
    `  - You provided: category="${args.category}", subagent_type="${args.subagent_type}"\n` +
    `  - Use category for task delegation (e.g., category="${categoryExamples.split(", ")[0]}")\n` +
    `  - Use subagent_type for direct agent invocation (e.g., subagent_type="explore")\n` +
    `  - Valid subagent_type values: explore, librarian, oracle, metis, momus`
  )
}
if (args.category) {
  args.subagent_type = SISYPHUS_JUNIOR_AGENT
}
```

### 片段 2: Hashline 编辑的 LINE#ID 格式

```typescript
// src/tools/hashline-edit/tools.ts:21-38
edits: tool.schema
  .array(
    tool.schema.object({
      op: tool.schema
        .union([
          tool.schema.literal("replace"),
          tool.schema.literal("append"),
          tool.schema.literal("prepend"),
        ])
        .describe("Hashline edit operation mode"),
      pos: tool.schema.string().optional().describe("Primary anchor in LINE#ID format"),
      end: tool.schema.string().optional().describe("Range end anchor in LINE#ID format"),
      lines: tool.schema
        .union([tool.schema.array(tool.schema.string()), tool.schema.string(), tool.schema.null()])
        .describe("Replacement or inserted lines"),
    })
  )
```

---

## 依赖关系

OMO 对 OpenCode 的依赖是单向且明确的：

```
OMO (Layer 3 Enhancement)
    │
    ├─depends on──→ OpenCode Plugin API (config, tool, hook handlers)
    ├─depends on──→ OpenCode Tool API (tool definition, execution)
    ├─depends on──→ OpenCode Hook System (lifecycle events)
    └─depends on──→ OpenCode Session API (session management)
    │
    ▼
OpenCode Base (Layer 3 Foundation)
    │
    ├─depends on──→ Model APIs (Layer 2)
    └─depends on──→ Infrastructure (Layer 1)
```

OMO 不会修改 OpenCode 的核心代码，而是通过插件机制扩展功能。这种设计确保：
1. OpenCode 升级时 OMO 兼容性好
2. 用户可以选择只使用 OpenCode 而不启用 OMO
3. OMO 的增强功能可以独立迭代

---

## 实战示例：OMO 编排增强场景

**场景**：用户要求"重构这个项目的日志系统"

### 纯 OpenCode 的处理方式：
1. 单模型顺序执行
2. 可能遗漏某些文件
3. 上下文超限后丢失早期信息
4. 编辑冲突风险

### OMO 的编排增强：

```typescript
// Step 1: Intent Gate 识别这是"重构"任务，需要规划
// Step 2: Prometheus 进行战略规划

task(
  subagent_type="prometheus",
  description="Plan logging refactor",
  prompt="Analyze the logging system and create a refactoring plan...",
  run_in_background=false
)

// Step 3: 并行启动多个 Explore Agent 搜索代码模式
task(
  subagent_type="explore",
  description="Find console.log patterns",
  prompt="Search for all console.log usages...",
  run_in_background=true
)
task(
  subagent_type="explore", 
  description="Find logger imports",
  prompt="Search for logger module imports...",
  run_in_background=true
)

// Step 4: Atlas 根据计划分发任务到不同 Category
// - quick category: 简单的 console.log 替换
// - deep category: 复杂的日志级别重构
// - visual-engineering: 日志 UI 组件更新

// Step 5: Hashline Edit 确保所有修改准确无误
// Step 6: Todo Enforcer 确保没有遗漏的任务
// Step 7: 上下文压缩保持会话活跃
```

通过 OMO 的编排增强，一个复杂的重构任务被分解为可并行执行的子任务，每个子任务由最适合的模型处理，最终结果通过 Hashline 编辑安全合并。

---

## 交叉引用

- 参见：[三层架构模型](./01-三层架构模型.md) - 了解 Layer 1/2/3 的完整划分
- 参见：[初始化流程](./03-初始化流程.md) - OMO 插件的 5 步初始化流程详解
- 参见：[工具注册系统](../04-工具系统/01-工具注册.md) - 26 个工具的注册机制
- 参见：[Agent 系统](../02-agent-syste./03-Agent 编排.md) - 11 个 Agent 的详细设计
- 参见：[Hook 系统](../03-hook-syste./01-Hook 分层.md) - 48 个生命周期钩子的分层架构
- 参见：[Category 路由](../05-delegation-system/category-routing.md) - 8 个内置类别的路由逻辑
