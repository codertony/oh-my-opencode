# Hook 分层架构：5 层 48 钩子

> 所属模块：03-hooks-system | 三层模型：Layer 3 | 优先级：P1

---

## 架构位置

```
┌─────────────────────────────────────────────────────────────┐
│                    OpenCode Plugin Layer                    │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────┐   │
│  │              Layer 3: Hook System                    │   │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌──────────┐  │   │
│  │  │ Session │ │ Tool    │ │Transform│ │Continuation│  │   │
│  │  │ (23)    │ │ Guard   │ │ (4)     │ │ (7)      │  │   │
│  │  │         │ │ (12)    │ │         │ │          │  │   │
│  │  └─────────┘ └─────────┘ └─────────┘ └──────────┘  │   │
│  │                    ┌─────────┐                      │   │
│  │                    │ Skill   │                      │   │
│  │                    │ (2)     │                      │   │
│  │                    └─────────┘                      │   │
│  └─────────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────────┤
│  Layer 2: Tool Registry (26 tools)                          │
├─────────────────────────────────────────────────────────────┤
│  Layer 1: Agent System (11 agents)                          │
└─────────────────────────────────────────────────────────────┘
```

钩子系统位于插件架构的第三层，承上启下连接 OpenCode 的事件系统与插件的业务逻辑。48 个钩子按职责划分为 5 个层级，每个层级处理特定类型的事件。

---

## 核心职责

钩子系统的核心职责是拦截和处理 OpenCode 的生命周期事件，在关键节点注入自定义逻辑。它实现了以下能力：

**事件拦截**：通过 OpenCode 的 8 个钩子处理器（config、tool、chat.message、chat.params、chat.headers、event、tool.execute.before、tool.execute.after）拦截系统事件。

**逻辑注入**：在会话创建、消息处理、工具执行前后等关键节点注入业务逻辑，如上下文注入、权限检查、错误恢复等。

**状态管理**：维护会话级别的状态，包括上下文窗口监控、待办事项追踪、失败重试计数等。

**跨层协调**：协调 Agent 层、Tool 层与 OpenCode 层的交互，确保各组件协同工作。

---

## 代码索引 (Code Index)

| 功能点 | 文件路径 | 行号 | 函数/类 | 说明 |
|--------|----------|------|---------|------|
| Session Hooks 创建 | `src/plugin/hooks/create-session-hooks.ts` | 67-298 | `createSessionHooks` | 创建 23 个会话钩子 |
| Tool Guard Hooks 创建 | `src/plugin/hooks/create-tool-guard-hooks.ts` | 44-141 | `createToolGuardHooks` | 创建 12 个工具守卫钩子 |
| Transform Hooks 创建 | `src/plugin/hooks/create-transform-hooks.ts` | 22-72 | `createTransformHooks` | 创建 4 个转换钩子 |
| Continuation Hooks 创建 | `src/plugin/hooks/create-continuation-hooks.ts` | 31-128 | `createContinuationHooks` | 创建 7 个延续钩子 |
| Skill Hooks 创建 | `src/plugin/hooks/create-skill-hooks.ts` | 14-49 | `createSkillHooks` | 创建 2 个技能钩子 |
| Core Hooks 聚合 | `src/plugin/hooks/create-core-hooks.ts` | 9-46 | `createCoreHooks` | 聚合 Session + Guard + Transform |
| 钩子总入口 | `src/create-hooks.ts` | 28-87 | `createHooks` | 组合所有 5 层钩子 |
| 上下文窗口监控 | `src/hooks/context-window-monitor.ts` | 33-113 | `createContextWindowMonitorHook` | 监控 token 使用量 |
| 待办延续执行器 | `src/hooks/todo-continuation-enforcer/index.ts` | 12-59 | `createTodoContinuationEnforcer` | Boulder 机制核心 |
| 注释检查器 | `src/hooks/comment-checker/index.ts` | 1 | `createCommentCheckerHooks` | 防止 AI 生成注释 |
| Atlas 主控 | `src/hooks/atlas/index.ts` | 1-3 | `createAtlasHook` | Boulder 会话主控 |
| 关键词检测 | `src/hooks/keyword-detector/index.ts` | 1-5 | `createKeywordDetectorHook` | 检测 ultrawork 等模式 |
| 安全钩子包装 | `src/shared/safe-create-hook.ts` | 1-50 | `safeCreateHook` | 错误隔离包装器 |
| 钩子索引导出 | `src/hooks/index.ts` | 1-56 | - | 所有钩子工厂函数导出 |

---

## 5 层钩子架构

### 1. Session Hooks (23 个)

Session Hooks 是会话生命周期钩子，在会话创建、消息处理、事件触发等阶段执行。它们负责会话级别的状态管理和功能增强。

**核心钩子列表**：

| 钩子名称 | 事件类型 | 功能说明 |
|----------|----------|----------|
| `contextWindowMonitor` | `tool.execute.after` | 监控上下文窗口使用率，超过 70% 时发出提醒 |
| `preemptiveCompaction` | `session.idle` | 在达到上下文限制前主动触发压缩 |
| `sessionRecovery` | `session.error` | 自动从可恢复错误中恢复会话 |
| `sessionNotification` | `session.idle` | 会话完成时发送系统通知 |
| `thinkMode` | `chat.params` | 动态切换模型的思考模式（extended thinking） |
| `modelFallback` | `chat.params` | 模型出错时自动降级到备用模型 |
| `anthropicContextWindowLimitRecovery` | `session.error` | Anthropic 上下文限制恢复策略 |
| `autoUpdateChecker` | `session.created` | 检查插件更新 |
| `agentUsageReminder` | `chat.message` | 提醒可用的 Agent |
| `nonInteractiveEnv` | `chat.message` | 非交互环境行为调整 |
| `interactiveBashSession` | `tool.execute` | Tmux 会话管理 |
| `ralphLoop` | `event` | 自引用开发循环（Boulder 延续） |
| `editErrorRecovery` | `tool.execute.after` | 文件编辑失败重试 |
| `delegateTaskRetry` | `tool.execute.after` | 任务委托失败重试 |
| `startWork` | `chat.message` | `/start-work` 命令处理 |
| `prometheusMdOnly` | `tool.execute.before` | 强制 Prometheus 只写 Markdown |
| `sisyphusJuniorNotepad` | `chat.message` | 子代理记事本注入 |
| `noSisyphusGpt` | `chat.message` | 阻止 Sisyphus 使用 GPT 模型 |
| `noHephaestusNonGpt` | `chat.message` | 阻止 Hephaestus 使用非 GPT 模型 |
| `questionLabelTruncator` | `tool.execute.before` | 截断过长的问题标签 |
| `taskResumeInfo` | `chat.message` | 任务恢复信息注入 |
| `anthropicEffort` | `chat.params` | 调整 Anthropic 推理努力级别 |
| `runtimeFallback` | `event` | API 错误时自动切换模型 |
| `legacyPluginToast` | `event` | 旧版插件兼容性提示 |

### 2. Tool Guard Hooks (12 个)

Tool Guard Hooks 是工具执行守卫钩子，在工具执行前后进行权限检查和输出处理。它们确保工具使用的安全性和输出质量。

**核心钩子列表**：

| 钩子名称 | 事件类型 | 功能说明 |
|----------|----------|----------|
| `commentChecker` | `tool.execute.after` | 检查并阻止 AI 生成的注释模式 |
| `toolOutputTruncator` | `tool.execute.after` | 截断超长的工具输出 |
| `directoryAgentsInjector` | `tool.execute.before` | 注入目录 AGENTS.md 到上下文 |
| `directoryReadmeInjector` | `tool.execute.before` | 注入目录 README.md 到上下文 |
| `emptyTaskResponseDetector` | `tool.execute.after` | 检测空的任务响应 |
| `rulesInjector` | `tool.execute.before` | 条件规则注入（AGENTS.md、配置） |
| `tasksTodowriteDisabler` | `tool.execute.before` | 任务系统激活时禁用 TodoWrite |
| `writeExistingFileGuard` | `tool.execute.before` | 写入已存在文件前要求先读取 |
| `hashlineReadEnhancer` | `tool.execute.after` | 为 Read 输出添加行哈希（LINE#ID） |
| `jsonErrorRecovery` | `tool.execute.after` | 检测 JSON 解析错误并注入修复提示 |
| `readImageResizer` | `tool.execute.after` | 调整图片大小以优化上下文效率 |
| `todoDescriptionOverride` | `tool.execute.before` | 覆盖待办事项描述 |
| `webfetchRedirectGuard` | `tool.execute.before` | WebFetch 重定向守卫 |

### 3. Transform Hooks (4 个)

Transform Hooks 是消息转换钩子，在消息发送到模型前对消息进行转换和增强。它们负责上下文注入和消息验证。

**核心钩子列表**：

| 钩子名称 | 事件类型 | 功能说明 |
|----------|----------|----------|
| `claudeCodeHooks` | `messages.transform` | Claude Code settings.json 兼容层 |
| `keywordDetector` | `messages.transform` | 检测 ultrawork/search/analyze 等模式关键词 |
| `contextInjectorMessagesTransform` | `messages.transform` | 注入 AGENTS.md/README.md 到消息上下文 |
| `thinkingBlockValidator` | `messages.transform` | 验证 thinking 块结构有效性 |

### 4. Continuation Hooks (7 个)

Continuation Hooks 是任务延续钩子，负责在会话空闲或中断时强制任务继续执行。这是 "Boulder" 机制的核心实现。

**核心钩子列表**：

| 钩子名称 | 事件类型 | 功能说明 |
|----------|----------|----------|
| `stopContinuationGuard` | `chat.message` | `/stop-continuation` 命令处理 |
| `compactionContextInjector` | `session.compacted` | 压缩后重新注入上下文 |
| `compactionTodoPreserver` | `session.compacted` | 压缩过程中保留待办事项 |
| `todoContinuationEnforcer` | `session.idle` | **Boulder**：待办未完成时强制延续 |
| `unstableAgentBabysitter` | `session.idle` | 监控不稳定 Agent 行为 |
| `backgroundNotificationHook` | `event` | 后台任务完成通知 |
| `atlasHook` | `event` | Boulder/后台会话的主控钩子 |

### 5. Skill Hooks (2 个)

Skill Hooks 是技能激活钩子，负责在适当时机提醒和激活技能功能。

**核心钩子列表**：

| 钩子名称 | 事件类型 | 功能说明 |
|----------|----------|----------|
| `categorySkillReminder` | `chat.message` | 提醒类别+技能委托 |
| `autoSlashCommand` | `chat.message` | 自动检测 `/command` 命令模式 |

---

## 钩子执行顺序

钩子按照以下顺序执行，形成完整的处理链：

```
用户输入
    ↓
┌─────────────────────────────────────────┐
│ 1. Session Hooks (chat.message)         │
│    - keywordDetector                    │
│    - agentUsageReminder                 │
│    - categorySkillReminder              │
│    - autoSlashCommand                   │
└─────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────┐
│ 2. Transform Hooks                      │
│    - messages.transform                 │
│    - contextInjectorMessagesTransform   │
│    - thinkingBlockValidator             │
└─────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────┐
│ 3. Tool Guard Hooks (tool.execute.before)│
│    - rulesInjector                      │
│    - writeExistingFileGuard             │
│    - tasksTodowriteDisabler             │
└─────────────────────────────────────────┘
    ↓
工具执行
    ↓
┌─────────────────────────────────────────┐
│ 4. Tool Guard Hooks (tool.execute.after)│
│    - commentChecker                     │
│    - toolOutputTruncator                │
│    - hashlineReadEnhancer               │
└─────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────┐
│ 5. Session Hooks (tool.execute.after)   │
│    - contextWindowMonitor               │
│    - editErrorRecovery                  │
└─────────────────────────────────────────┘
    ↓
会话空闲
    ↓
┌─────────────────────────────────────────┐
│ 6. Continuation Hooks (session.idle)    │
│    - todoContinuationEnforcer           │
│    - atlasHook                          │
│    - unstableAgentBabysitter            │
└─────────────────────────────────────────┘
```

---

## 流程/架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                     OpenCode 事件流                              │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  Session Tier (23 hooks)                                        │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐               │
│  │ chat.message│ │chat.params  │ │session.idle │               │
│  └─────────────┘ └─────────────┘ └─────────────┘               │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  Transform Tier (4 hooks)                                       │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ messages.transform                                       │   │
│  │  - claudeCodeHooks                                       │   │
│  │  - keywordDetector                                       │   │
│  │  - contextInjectorMessagesTransform                      │   │
│  │  - thinkingBlockValidator                                │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  Tool Guard Tier (12 hooks)                                     │
│  ┌─────────────────┐ ┌─────────────────┐                       │
│  │ tool.execute    │ │ tool.execute    │                       │
│  │ .before         │ │ .after          │                       │
│  │  - rulesInjector│ │  - commentCheck │                       │
│  │  - writeGuard   │ │  - outputTrunc  │                       │
│  └─────────────────┘ └─────────────────┘                       │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  Continuation Tier (7 hooks)                                    │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐               │
│  │ session.idle│ │session.error│ │session.comp │               │
│  │  - todoEnf  │ │  - recovery │ │  - injector │               │
│  │  - atlas    │ │             │ │  - preserver│               │
│  └─────────────┘ └─────────────┘ └─────────────┘               │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  Skill Tier (2 hooks)                                           │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ chat.message (reminder hooks)                            │   │
│  │  - categorySkillReminder                                 │   │
│  │  - autoSlashCommand                                      │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 关键代码片段

### 片段 1：钩子创建入口

```typescript
// src/create-hooks.ts:28-87
export function createHooks(args: {
  ctx: PluginContext
  pluginConfig: OhMyOpenCodeConfig
  modelCacheState: ModelCacheState
  backgroundManager: BackgroundManager
  isHookEnabled: (hookName: HookName) => boolean
  safeHookEnabled: boolean
  mergedSkills: LoadedSkill[]
  availableSkills: AvailableSkill[]
}) {
  const core = createCoreHooks({
    ctx,
    pluginConfig,
    modelCacheState,
    isHookEnabled,
    safeHookEnabled,
  })

  const continuation = createContinuationHooks({
    ctx,
    pluginConfig,
    isHookEnabled,
    safeHookEnabled,
    backgroundManager,
    sessionRecovery: core.sessionRecovery,
  })

  const skill = createSkillHooks({
    ctx,
    pluginConfig,
    isHookEnabled,
    safeHookEnabled,
    mergedSkills,
    availableSkills,
  })

  return {
    ...core,
    ...continuation,
    ...skill,
    disposeHooks: (): void => {
      disposeCreatedHooks(hooks)
    },
  }
}
```

### 片段 2：安全钩子包装器

```typescript
// src/shared/safe-create-hook.ts:1-30
export function safeCreateHook<T>(
  hookName: string,
  factory: () => T,
  options: { enabled?: boolean } = {}
): T | null {
  if (!options.enabled) {
    return factory()
  }

  try {
    return factory()
  } catch (error) {
    log(`Failed to create hook "${hookName}":`, error)
    return null
  }
}
```

### 片段 3：待办延续执行器核心逻辑

```typescript
// src/hooks/todo-continuation-enforcer/index.ts:12-59
export function createTodoContinuationEnforcer(
  ctx: PluginInput,
  options: TodoContinuationEnforcerOptions = {}
): TodoContinuationEnforcer {
  const {
    backgroundManager,
    skipAgents = DEFAULT_SKIP_AGENTS,
    isContinuationStopped,
  } = options

  const sessionStateStore = createSessionStateStore()

  const markRecovering = (sessionID: string): void => {
    const state = sessionStateStore.getState(sessionID)
    state.isRecovering = true
    sessionStateStore.cancelCountdown(sessionID)
  }

  const handler = createTodoContinuationHandler({
    ctx,
    sessionStateStore,
    backgroundManager,
    skipAgents,
    isContinuationStopped,
  })

  return {
    handler,
    markRecovering,
    markRecoveryComplete,
    cancelAllCountdowns,
    dispose: () => sessionStateStore.shutdown(),
  }
}
```

---

## 依赖关系

钩子系统依赖以下组件：

**向上依赖**：
- OpenCode Plugin API：提供事件订阅和消息注入能力
- Agent 系统：获取当前会话的 Agent 信息

**向下依赖**：
- Tool Registry：26 个工具供钩子调用
- Background Manager：后台任务管理
- Config System：钩子启用/禁用配置

**同级依赖**：
- Session Recovery：会话恢复状态共享
- Model Cache State：模型上下文限制信息

---

## 实战示例

### 示例：钩子链处理 `/ultrawork` 命令

当用户输入 `/ultrawork refactor this codebase` 时，钩子链按以下顺序执行：

**步骤 1：Session Hooks - keywordDetector**

```typescript
// src/hooks/keyword-detector/hook.ts
export function createKeywordDetectorHook(ctx: PluginInput, collector: ContextCollector) {
  return async (input: ChatMessageInput, output: ChatMessageOutput) => {
    const text = extractPromptText(input.message.parts)
    const detected = detectKeywordsWithType(text, input.agent, input.model)
    
    // 检测到 "ultrawork" 关键词
    if (detected.some(d => d.type === 'ultrawork')) {
      // 注入 ultrawork 模式系统提示
      output.systemMessages.push(getUltraworkMessage(input.agent, input.model))
    }
  }
}
```

**步骤 2：Transform Hooks - contextInjectorMessagesTransform**

```typescript
// 在消息发送到模型前，注入 AGENTS.md 和 README.md
const contextInjector = createContextInjectorMessagesTransformHook(collector)
// 将项目上下文注入到消息中
```

**步骤 3：Tool Guard Hooks - rulesInjector**

```typescript
// src/hooks/rules-injector/hook.ts
// 在工具执行前，根据当前模式注入相应的规则
if (isUltraworkMode) {
  // 注入 ultrawork 规则：并行代理、深度探索等
}
```

**步骤 4：Session Hooks - todoContinuationEnforcer**

```typescript
// src/hooks/todo-continuation-enforcer/handler.ts
// 会话空闲时检查待办事项
if (hasIncompleteTodos(sessionID) && !isContinuationStopped(sessionID)) {
  // 启动 2 秒倒计时
  startCountdown(sessionID, () => {
    // 注入延续提示，强制 Agent 继续工作
    injectContinuationPrompt(sessionID)
  })
}
```

**步骤 5：Continuation Hooks - atlasHook**

```typescript
// src/hooks/atlas/atlas-hook.ts
// Atlas 监控 Boulder 会话
if (isBoulderSession(sessionID) && hasIncompleteWork(sessionID)) {
  // 决策是否注入延续提示
  injectBoulderContinuation(sessionID)
}
```

这个钩子链确保了：
1. 用户输入被正确识别为 ultrawork 模式
2. 项目上下文被注入到模型
3. 相应的规则被激活
4. 任务未完成时自动强制延续
5. Boulder 会话得到适当监控

---

## 交叉引用

- 参见：[Session Hooks 详解](./02-Session Hooks.md) - 深入了解 23 个会话钩子的实现细节
- 参见：[Tool Guard Hooks 详解](./03-Tool Guard Hooks.md) - 工具守卫钩子的权限检查机制
- 参见：[Hook 执行顺序](../09-集成工作流/02-Hook 执行顺序.md) - 完整的钩子调用链时序图
- 参见：[Boulder 机制](../04-continuation-system/boulder-mechanism.md) - Continuation Hooks 的核心原理
- 参见：[Context Injector](../05-context-system/context-injector.md) - Transform Hooks 的上下文注入实现
- 参见：[Agent 系统](../02-agent-syste./03-Agent 编排.md) - Session Hooks 如何与 Agent 协作
