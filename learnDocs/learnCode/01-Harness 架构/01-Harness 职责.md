# Harness 的 6 大核心职责

> 所属模块：01-harness-architecture | 三层模型：Layer 3 | 优先级：P0

---

## 架构位置

Harness 层位于 OMO 三层架构的最顶层（Layer 3），直接面向 OpenCode 运行时环境。它作为插件与 OpenCode 之间的桥梁，负责将底层能力（Layer 1 和 Layer 2）封装成 OpenCode 可识别的接口。

```
┌─────────────────────────────────────────────────────────────┐
│                    OpenCode Runtime                         │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              Layer 3: Harness                        │   │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐   │   │
│  │  │ Tool    │ │ Session │ │ Context │ │ Model   │   │   │
│  │  │Registry │ │ Manager │ │Injector │ │Fallback │   │   │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────┘   │   │
│  │  ┌─────────┐ ┌─────────┐                           │   │
│  │  │Permission│ │Background│                          │   │
│  │  │ Guard   │ │ Task     │                          │   │
│  │  └─────────┘ └─────────┘                           │   │
│  └─────────────────────────────────────────────────────┘   │
│                         ▲                                   │
│  ┌──────────────────────┼──────────────────────────────┐   │
│  │         Layer 2: Services                           │   │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐   │   │
│  │  │Background│ │ Tmux    │ │ Skill   │ │ MCP     │   │   │
│  │  │ Manager  │ │ Session │ │ Loader  │ │ Manager │   │   │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────┘   │   │
│  └─────────────────────────────────────────────────────┘   │
│                         ▲                                   │
│  ┌──────────────────────┼──────────────────────────────┐   │
│  │         Layer 1: Foundation                         │   │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐   │   │
│  │  │ Config  │ │ Shared  │ │ Hooks   │ │ Tools   │   │   │
│  │  │ Schema  │ │ Utils   │ │ Registry│ │ Factory │   │   │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────┘   │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

---

## 核心职责

Harness 层是 OMO 插件的"门面层"，承担着 6 大核心职责。它不仅是工具与 OpenCode 的粘合剂，更是安全边界、会话管理、上下文组装、任务编排和错误恢复的综合协调者。Harness 的设计哲学是"防御性编程"：每个职责都包含多层保护机制，确保在复杂的多模型、多代理场景下系统依然稳定运行。从工具注册到模型 fallback，从权限守卫到后台任务，Harness 构成了 OMO 插件与外部世界交互的唯一合法通道。

---

## 代码索引 (Code Index)

| 功能点 | 文件路径 | 行号 | 函数/类 | 说明 |
|--------|----------|------|---------|------|
| 工具注册 | `src/plugin/tool-registry.ts` | 43-190 | `createToolRegistry` | 26 个工具的工厂组装 |
| 工具过滤 | `src/plugin/tool-registry.ts` | 157 | `filterDisabledTools` | 根据配置禁用指定工具 |
| 工具守卫 | `src/plugin/hooks/create-tool-guard-hooks.ts` | 44-141 | `createToolGuardHooks` | 12 个工具守卫钩子 |
| 会话列表 | `src/tools/session-manager/tools.ts` | 30-61 | `session_list` | 列出历史会话 |
| 会话读取 | `src/tools/session-manager/tools.ts` | 63-94 | `session_read` | 读取会话消息 |
| 会话搜索 | `src/tools/session-manager/tools.ts` | 96-135 | `session_search` | 搜索会话内容 |
| 会话信息 | `src/tools/session-manager/tools.ts` | 137-155 | `session_info` | 获取会话元数据 |
| AGENTS 注入 | `src/hooks/directory-agents-injector/hook.ts` | 30-87 | `createDirectoryAgentsInjectorHook` | 自动注入目录 AGENTS.md |
| 规则注入 | `src/hooks/rules-injector/hook.ts` | 32-90 | `createRulesInjectorHook` | 条件规则注入 |
| 后台管理器 | `src/features/background-agent/manager.ts` | 129-1472 | `BackgroundManager` | 后台任务生命周期管理 |
| 并发控制 | `src/features/background-agent/concurrency.ts` | 15-137 | `ConcurrencyManager` | 每模型并发限制 |
| 模型 Fallback | `src/hooks/model-fallback/hook.ts` | 215-276 | `createModelFallbackHook` | API 错误自动切换模型 |

---

## 6 大核心职责详解

### 1. 工具执行与注册管理

工具注册是 Harness 最基础的职责。`createToolRegistry` 函数（`src/plugin/tool-registry.ts:43`）是工具组装的中央工厂，负责将 26 个工具通过工厂模式注册到 OpenCode。

**工厂模式实现：**

```typescript
// src/plugin/tool-registry.ts:136-151
const allTools: Record<string, ToolDefinition> = {
  ...builtinTools,
  ...createGrepTools(ctx),
  ...createGlobTools(ctx),
  ...createAstGrepTools(ctx),
  ...createSessionManagerTools(ctx),
  ...backgroundTools,
  call_omo_agent: callOmoAgent,
  ...(lookAt ? { look_at: lookAt } : {}),
  task: delegateTask,
  skill_mcp: skillMcpTool,
  skill: skillTool,
  interactive_bash,
  ...taskToolsRecord,
  ...hashlineToolsRecord,
}
```

每个工具都通过 `createXXXTool` 工厂函数创建，这种设计允许：
- 按需启用/禁用（如 `hashline_edit` 需配置开启）
- 动态参数注入（如 `skillTool` 接收 commands 和 skills）
- 统一的 schema 规范化（`normalizeToolArgSchemas`，第 153-155 行）

**工具数量控制机制**（第 159-184 行）：当工具数量超过 `max_tools` 限制时，Harness 会按优先级自动裁剪低优先级工具（如 `session_list`、`call_omo_agent` 等），确保不超过 OpenCode 的上下文限制。

### 2. 权限守卫与安全边界

Harness 通过 12 个工具守卫钩子（Tool Guard Hooks）在执行前后插入安全检查。这些钩子由 `createToolGuardHooks`（`src/plugin/hooks/create-tool-guard-hooks.ts:44`）统一管理。

**12 个工具守卫钩子：**

| 钩子名称 | 触发时机 | 功能 |
|---------|---------|------|
| `commentChecker` | tool.execute.after | 阻止 AI 生成的注释模式 |
| `toolOutputTruncator` | tool.execute.after | 截断超大工具输出 |
| `directoryAgentsInjector` | tool.execute.after | 注入目录 AGENTS.md |
| `directoryReadmeInjector` | tool.execute.after | 注入目录 README.md |
| `emptyTaskResponseDetector` | tool.execute.after | 检测空任务响应 |
| `rulesInjector` | tool.execute.before | 条件规则注入 |
| `tasksTodowriteDisabler` | tool.execute.before | 任务系统激活时禁用 TodoWrite |
| `writeExistingFileGuard` | tool.execute.before | 写入已存在文件前要求先读取 |
| `hashlineReadEnhancer` | tool.execute.after | 为 Read 输出添加行哈希 |
| `jsonErrorRecovery` | tool.execute.after | JSON 解析错误恢复 |
| `readImageResizer` | tool.execute.after | 调整图片大小以节省上下文 |
| `todoDescriptionOverride` | tool.execute.before | 覆盖 todo 描述 |

**安全边界示例** - `writeExistingFileGuard`：

```typescript
// 伪代码示意
if (tool === "write" && fileExists(args.filePath)) {
  throw new Error("必须先读取文件才能写入已存在的文件");
}
```

这种设计确保了"读-改-写"的安全流程，防止意外覆盖。

### 3. 会话管理与持久化

Harness 提供 4 个会话管理工具（`src/tools/session-manager/tools.ts:30-158`），允许代理查询和恢复历史会话：

**4 个会话工具：**

1. **`session_list`**（第 34-61 行）：列出历史会话，支持按日期和项目路径过滤
2. **`session_read`**（第 63-94 行）：读取指定会话的完整消息历史
3. **`session_search`**（第 96-135 行）：跨会话搜索内容，支持大小写敏感选项
4. **`session_info`**（第 137-155 行）：获取会话元数据（消息数、时间范围等）

**超时保护机制**（第 23-28 行）：

```typescript
function withTimeout<T>(promise: Promise<T>, ms: number, operation: string): Promise<T> {
  return Promise.race([
    promise,
    new Promise<T>((_, reject) => 
      setTimeout(() => reject(new Error(`${operation} timed out after ${ms}ms`)), ms)
    ),
  ])
}
```

所有会话操作都有 60 秒超时保护，防止因存储后端延迟导致代理挂起。

### 4. 上下文组装机制

Harness 通过两个核心钩子实现上下文的自动组装：

**AGENTS.md 注入**（`src/hooks/directory-agents-injector/hook.ts:30-87`）：

```typescript
const toolExecuteAfter = async (input: ToolExecuteInput, output: ToolExecuteOutput) => {
  const toolName = input.tool.toLowerCase();
  if (toolName === "read") {
    await processFilePathForAgentsInjection({
      ctx,
      truncator,
      sessionCaches,
      filePath: output.title,
      sessionID: input.sessionID,
      output,
    });
  }
};
```

当代理读取文件时，Harness 自动查找该文件所在目录的 `AGENTS.md` 并注入到上下文中。这种"就近原则"确保代理获得最相关的上下文。

**规则注入**（`src/hooks/rules-injector/hook.ts:32-90`）：

```typescript
const TRACKED_TOOLS = ["read", "write", "edit", "multiedit"];

const toolExecuteAfter = async (input: ToolExecuteInput, output: ToolExecuteOutput) => {
  const toolName = input.tool.toLowerCase();
  if (TRACKED_TOOLS.includes(toolName)) {
    const filePath = getRuleInjectionFilePath(output);
    if (!filePath) return;
    await processFilePathForInjection(filePath, input.sessionID, output);
  }
};
```

规则注入支持条件规则（如只在特定文件类型或目录下生效），通过 `calculateDistance` 计算规则文件与目标文件的"距离"，优先应用最近的规则。

### 5. 后台任务编排

`BackgroundManager`（`src/features/background-agent/manager.ts:129-1472`）是 Harness 的后台任务编排核心，支持同时运行多个子代理。

**并发控制模型**（`src/features/background-agent/concurrency.ts:15-137`）：

```typescript
export class ConcurrencyManager {
  private counts: Map<string, number> = new Map()
  private queues: Map<string, QueueEntry[]> = new Map()

  async acquire(model: string): Promise<void> {
    const limit = this.getConcurrencyLimit(model)  // 默认 5
    const current = this.counts.get(model) ?? 0
    if (current < limit) {
      this.counts.set(model, current + 1)
      return
    }
    // 加入 FIFO 队列等待
    return new Promise<void>((resolve, reject) => {
      const entry: QueueEntry = { resolve, rawReject: reject, settled: false }
      queue.push(entry)
    })
  }
}
```

**任务生命周期**：

```
LaunchInput → pending → [ConcurrencyManager queue] → running → polling → completed/error/cancelled/interrupt
```

**关键特性：**
- 按模型/提供商限制并发（默认每模型 5 个）
- FIFO 队列确保任务按顺序执行
- 深度限制防止子代理无限递归（`getMaxSubagentDepth`）
- 熔断机制检测循环调用（`detectRepetitiveToolUse`，第 954-968 行）

### 6. 模型 Fallback 与错误恢复

当 API 调用失败时，Harness 自动切换到备用模型。`createModelFallbackHook`（`src/hooks/model-fallback/hook.ts:215-276`）实现了这一机制。

**Fallback 链配置示例：**

```typescript
// src/shared/model-requirements.ts
AGENT_MODEL_REQUIREMENTS = {
  sisyphus: {
    fallbackChain: [
      { model: "claude-opus-4-6", providers: ["anthropic"], variant: "max" },
      { model: "kimi-k2.5", providers: ["kimi"], variant: "high" },
      { model: "glm-5", providers: ["z.ai"], variant: "high" },
    ]
  }
}
```

**Fallback 触发流程**（第 65-115 行）：

```typescript
export function setPendingModelFallback(
  sessionID: string,
  agentName: string,
  currentProviderID: string,
  currentModelID: string,
): boolean {
  const requirements = AGENT_MODEL_REQUIREMENTS[agentKey]
  const fallbackChain = requirements?.fallbackChain
  
  if (!fallbackChain || fallbackChain.length === 0) {
    return false  // 无可用 fallback
  }
  
  const state: ModelFallbackState = {
    providerID: currentProviderID,
    modelID: currentModelID,
    fallbackChain,
    attemptCount: 0,
    pending: true,
  }
  pendingModelFallbacks.set(sessionID, state)
  return true
}
```

**错误分类与重试**（`src/shared/model-error-classifier.ts`）：

- **可重试错误**：速率限制、临时网络错误 → 自动重试
- **不可重试错误**：认证失败、无效参数 → 立即 fallback
- **致命错误**：上下文窗口超限 → 触发 compaction

---

## 流程/架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Harness Layer                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐        │
│  │   Tool       │────▶│   Permission │────▶│   Context    │        │
│  │   Registry   │     │   Guard      │     │   Injector   │        │
│  └──────────────┘     └──────────────┘     └──────────────┘        │
│         │                    │                    │                │
│         ▼                    ▼                    ▼                │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐        │
│  │   Session    │◀───▶│   Background │◀───▶│   Model      │        │
│  │   Manager    │     │   Task       │     │   Fallback   │        │
│  └──────────────┘     └──────────────┘     └──────────────┘        │
│         ▲                    ▲                    ▲                │
│         │                    │                    │                │
│         └────────────────────┴────────────────────┘                │
│                              │                                     │
│                              ▼                                     │
│                    ┌──────────────────┐                           │
│                    │   OpenCode API   │                           │
│                    └──────────────────┘                           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘

职责交互关系：
- Tool Registry → Permission Guard：工具执行前必须通过权限检查
- Permission Guard → Context Injector：安全检查通过后注入上下文
- Context Injector → Session Manager：上下文可能包含会话历史
- Session Manager ↔ Background Task：后台任务创建新会话
- Background Task → Model Fallback：任务失败时触发模型切换
- Model Fallback → Tool Registry：fallback 后重新选择工具集
```

---

## 关键代码片段

### 片段 1：工具注册工厂模式

```typescript
// src/plugin/tool-registry.ts:43-60
export function createToolRegistry(args: {
  ctx: PluginContext
  pluginConfig: OhMyOpenCodeConfig
  managers: Pick<Managers, "backgroundManager" | "tmuxSessionManager" | "skillMcpManager">
  skillContext: SkillContext
  availableCategories: AvailableCategory[]
}): ToolRegistryResult {
  const { ctx, pluginConfig, managers, skillContext, availableCategories } = args

  const backgroundTools = createBackgroundTools(managers.backgroundManager, ctx.client)
  const callOmoAgent = createCallOmoAgent(
    ctx,
    managers.backgroundManager,
    pluginConfig.disabled_agents ?? [],
    pluginConfig.agents,
    pluginConfig.categories,
  )
  // ... 更多工厂调用
}
```

### 片段 2：并发控制实现

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

### 片段 3：模型 Fallback 应用

```typescript
// src/hooks/model-fallback/hook.ts:219-274
return {
  "chat.message": async (
    input: ChatMessageInput,
    output: ChatMessageHandlerOutput,
  ): Promise<void> => {
    const { sessionID } = input
    if (!sessionID) return

    const fallback = getNextFallback(sessionID)
    if (!fallback) return

    output.message["model"] = {
      providerID: fallback.providerID,
      modelID: fallback.modelID,
    }
    if (fallback.variant !== undefined) {
      output.message["variant"] = fallback.variant
    }
    
    // 通知用户
    if (toast) {
      await Promise.resolve(
        toast({
          title: "Model fallback",
          message: `Using ${fallback.providerID}/${fallback.modelID}`,
          variant: "warning",
          duration: 5000,
        }),
      )
    }
  },
}
```

---

## 依赖关系

6 个职责之间存在以下依赖关系：

```
Tool Registry (基础)
    │
    ▼
Permission Guard (安全层)
    │
    ▼
Context Injector (上下文层)
    │
    ▼
Session Manager (状态层)
    │
    ▼
Background Task (执行层)
    │
    ▼
Model Fallback (恢复层)
```

**依赖说明：**

1. **Permission Guard 依赖 Tool Registry**：守卫需要知道有哪些工具才能进行权限检查
2. **Context Injector 依赖 Permission Guard**：只有安全检查通过后才注入上下文
3. **Session Manager 依赖 Context Injector**：会话历史可能包含注入的上下文
4. **Background Task 依赖 Session Manager**：后台任务需要创建和管理新会话
5. **Model Fallback 依赖 Background Task**：任务执行失败时才需要 fallback

**循环依赖避免：**
- 通过接口抽象（如 `PluginContext`）打破直接依赖
- 通过事件机制（如 `session.idle`、`session.error`）解耦通知
- 通过配置而非运行时依赖传递参数

---

## 实战示例

### 示例 1: 工具注册流程

**场景**：用户安装 OMO 插件后，首次启动时的工具注册流程。

**流程步骤**：

1. **配置加载**：`loadPluginConfig` 读取用户配置和项目配置
2. **管理器创建**：`createManagers` 初始化 BackgroundManager、TmuxSessionManager 等
3. **工具工厂调用**：`createToolRegistry` 按顺序调用各工具工厂：
   ```typescript
   const backgroundTools = createBackgroundTools(managers.backgroundManager, ctx.client)
   const callOmoAgent = createCallOmoAgent(ctx, managers.backgroundManager, ...)
   const delegateTask = createDelegateTask({ manager: managers.backgroundManager, ... })
   ```
4. **工具组装**：将所有工具合并到 `allTools` 对象
5. **Schema 规范化**：`normalizeToolArgSchemas` 统一参数格式
6. **过滤禁用工具**：`filterDisabledTools` 移除配置中禁用的工具
7. **数量裁剪**：如果超过 `max_tools`，按优先级移除低优先级工具
8. **返回注册表**：将 `filteredTools` 返回给 OpenCode

**关键代码路径**：
```
index.ts → createTools() → createToolRegistry() → 工厂函数 → filterDisabledTools → 返回
```

### 示例 2: 模型 Fallback 场景

**场景**：Sisyphus 代理使用 Claude Opus 时遇到 Anthropic API 速率限制，自动切换到 Kimi。

**执行流程**：

1. **API 错误发生**：OpenCode 触发 `session.error` 事件
2. **错误分类**：`model-error-classifier.ts` 判断为可重试错误
3. **设置 Pending Fallback**：`setPendingModelFallback` 记录 fallback 状态：
   ```typescript
   const state: ModelFallbackState = {
     providerID: "anthropic",
     modelID: "claude-opus-4-6",
     fallbackChain: [...],
     attemptCount: 0,
     pending: true,
   }
   ```
4. **下次请求拦截**：`chat.message` 钩子检查到 pending fallback
5. **获取下一个模型**：`getNextFallback` 从 fallback 链中选择 Kimi：
   ```typescript
   const fallback = fallbackChain[0]  // { model: "kimi-k2.5", providers: ["kimi"] }
   const providerID = selectFallbackProvider(fallback.providers, "anthropic")
   ```
6. **修改输出模型**：将 `output.message["model"]` 设置为 Kimi
7. **用户通知**：显示 toast 通知 "Model fallback: Using kimi/kimi-k2.5"
8. **清除状态**：`clearPendingModelFallback` 清理 fallback 状态

**Fallback 链配置**：
```typescript
sisyphus: {
  fallbackChain: [
    { model: "claude-opus-4-6", providers: ["anthropic"], variant: "max" },
    { model: "kimi-k2.5", providers: ["kimi"], variant: "high" },
    { model: "glm-5", providers: ["z.ai"], variant: "high" },
  ]
}
```

---

## 故障分析：如果 Harness 失败会怎样？

### 故障场景：BackgroundManager 并发泄漏

**假设**：`ConcurrencyManager.release()` 由于 bug 未被调用，导致并发槽位永久占用。

**故障传播**：

1. **直接影响**：该模型的并发计数器永远达不到上限，新任务无法获取槽位
2. **队列堆积**：新任务在 FIFO 队列中无限等待
3. **内存泄漏**：`queuesByKey` Map 持续增长
4. **级联效应**：
   - 用户看到任务一直处于 "pending" 状态
   - 代理无法启动新的后台任务
   - 如果该模型是主要模型（如 Claude），整个系统几乎不可用

**检测机制**：

```typescript
// manager.ts 中的熔断机制（第 954-968 行）
const loopDetection = detectRepetitiveToolUse(task.progress.toolCallWindow)
if (loopDetection.triggered) {
  void this.cancelTask(task.id, {
    source: "circuit-breaker",
    reason: `Subagent called ${loopDetection.toolName} ${loopDetection.repeatedCount} consecutive times`,
  })
}
```

**恢复策略**：

1. **自动恢复**：任务超时（`TASK_TTL_MS`）后自动清理
2. **手动恢复**：重启 OpenCode 会话，重置 `BackgroundManager` 状态
3. **配置绕过**：临时切换到其他模型（利用 Model Fallback）

**预防措施**：
- 使用 `try/finally` 确保 `release()` 总是被调用
- 定期健康检查（`getCount`、`getQueueLength`）
- 设置任务最大执行时间（`maxIdleTimeMs`）

---

## 交叉引用

- 参见：[三层架构模型](../00-架构概览/01-三层架构模型.md) - 了解 Harness 在整个架构中的位置
- 参见：[OpenCode vs OMO 分层关系](../00-架构概览/02-OpenCode 与 OMO.md) - 理解 Harness 与 OpenCode 的边界
- 参见：[工具注册系统](../04-工具系统/01-工具注册.md) - 深入了解 26 个工具的实现细节
- 参见：[后台任务系统](../05-background-tasks/task-lifecycle.md) - 完整的后台任务生命周期文档
- 参见：[模型 Fallback 机制](../06-error-handling/model-fallback.md) - 错误恢复和模型切换的详细说明
