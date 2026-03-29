# Hook 执行顺序：完整钩子链

> 所属模块：09-integration-workflow | 三层模型：Layer 3 | 优先级：P3

---

## 架构位置

```
┌─────────────────────────────────────────────────────────────────┐
│                        Layer 3: Integration                      │
│                    ┌─────────────────────┐                       │
│                    │   Hook Execution    │                       │
│                    │      Chain          │                       │
│                    └─────────────────────┘                       │
│                           │                                      │
│     ┌─────────────────────┼─────────────────────┐                │
│     │                     │                     │                │
│     ▼                     ▼                     ▼                │
│  Session              Tool Guard           Transform             │
│  (23 hooks)           (12 hooks)           (4 hooks)             │
│     │                     │                     │                │
│     └─────────────────────┼─────────────────────┘                │
│                           │                                      │
│                           ▼                                      │
│                    Continuation (7 hooks)                        │
│                           │                                      │
│                           ▼                                      │
│                      Skill (2 hooks)                             │
└─────────────────────────────────────────────────────────────────┘
```

## 核心职责

Hook 执行顺序是 oh-my-opencode 插件的核心编排机制，负责在 48 个生命周期钩子之间建立明确的执行优先级和依赖关系。这套机制确保从会话创建到工具执行、从消息转换到任务延续的完整流程中，每个钩子都能在正确的时机介入，既避免冲突又保证功能完整性。

执行顺序的设计遵循"由外向内、由通用到特定"的原则：Session Hooks 处理最外层的会话生命周期事件；Tool Guard Hooks 在工具执行前后提供保护和增强；Transform Hooks 负责消息内容的转换和验证；Continuation Hooks 管理任务的中断与恢复；Skill Hooks 则在最外层提供技能相关的辅助功能。这种分层架构让每个层级的钩子专注于特定职责，同时通过明确的顺序避免相互干扰。

## 代码索引 (Code Index)

| 功能点 | 文件路径 | 行号 | 函数/类 | 说明 |
|--------|----------|------|---------|------|
| Hook 创建入口 | `src/create-hooks.ts` | 28-87 | `createHooks()` | 主入口，组合所有钩子 |
| Core Hooks 组合 | `src/plugin/hooks/create-core-hooks.ts` | 9-46 | `createCoreHooks()` | 组合 Session + Tool Guard + Transform |
| Session Hooks 创建 | `src/plugin/hooks/create-session-hooks.ts` | 67-298 | `createSessionHooks()` | 创建 23 个会话钩子 |
| Tool Guard Hooks 创建 | `src/plugin/hooks/create-tool-guard-hooks.ts` | 44-141 | `createToolGuardHooks()` | 创建 12 个工具保护钩子 |
| Transform Hooks 创建 | `src/plugin/hooks/create-transform-hooks.ts` | 22-72 | `createTransformHooks()` | 创建 4 个转换钩子 |
| Continuation Hooks 创建 | `src/plugin/hooks/create-continuation-hooks.ts` | 31-128 | `createContinuationHooks()` | 创建 7 个延续钩子 |
| Skill Hooks 创建 | `src/plugin/hooks/create-skill-hooks.ts` | 14-49 | `createSkillHooks()` | 创建 2 个技能钩子 |
| 安全 Hook 包装 | `src/shared/safe-create-hook.ts` | 1-50 | `safeCreateHook()` | 错误隔离包装器 |
| Hook 类型定义 | `src/config/schema/hooks.ts` | 1-100 | `HookName` | 所有钩子名称枚举 |
| 会话恢复钩子 | `src/hooks/session-recovery/` | 1-50 | `createSessionRecoveryHook()` | 会话状态恢复机制 |
| Todo 延续执行器 | `src/hooks/todo-continuation-enforcer/` | 1-80 | `createTodoContinuationEnforcer()` | 任务延续核心逻辑 |
| 工具输出截断 | `src/hooks/tool-output-truncator.ts` | 1-60 | `createToolOutputTruncatorHook()` | 输出长度控制 |

## 5 层钩子执行顺序

### 1. Session Hooks (23 个)

Session Hooks 是会话层级的生命周期钩子，在会话创建、消息发送、状态变化等关键节点触发。这些钩子构成了插件与 OpenCode 交互的第一道防线。

**执行顺序：**

```
1. context-window-monitor      # 上下文窗口监控
2. preemptive-compaction       # 预压缩
3. session-recovery            # 会话恢复
4. session-notification        # 会话通知
5. think-mode                  # 思考模式
6. model-fallback              # 模型回退
7. anthropic-context-window-limit-recovery  # Anthropic 窗口限制恢复
8. auto-update-checker         # 自动更新检查
9. agent-usage-reminder        # Agent 使用提醒
10. non-interactive-env        # 非交互环境
11. interactive-bash-session   # 交互式 Bash 会话
12. ralph-loop                 # Ralph 循环
13. edit-error-recovery        # 编辑错误恢复
14. delegate-task-retry        # 委托任务重试
15. start-work                 # 开始工作
16. prometheus-md-only        # Prometheus MD 专用
17. sisyphus-junior-notepad   # Sisyphus Junior 记事本
18. no-sisyphus-gpt           # 禁用 Sisyphus GPT
19. no-hephaestus-non-gpt     # 禁用 Hephaestus 非 GPT
20. question-label-truncator  # 问题标签截断
21. task-resume-info          # 任务恢复信息
22. anthropic-effort          # Anthropic 努力级别
23. runtime-fallback          # 运行时回退
24. legacy-plugin-toast       # 遗留插件提示
```

**代码引用：**

```typescript
// src/plugin/hooks/create-session-hooks.ts:67-298
export function createSessionHooks(args: {
  ctx: PluginContext
  pluginConfig: OhMyOpenCodeConfig
  modelCacheState: ModelCacheState
  isHookEnabled: (hookName: HookName) => boolean
  safeHookEnabled: boolean
}): SessionHooks {
  const { ctx, pluginConfig, modelCacheState, isHookEnabled, safeHookEnabled } = args
  const safeHook = <T>(hookName: HookName, factory: () => T): T | null =>
    safeCreateHook(hookName, factory, { enabled: safeHookEnabled })

  const contextWindowMonitor = isHookEnabled("context-window-monitor")
    ? safeHook("context-window-monitor", () =>
        createContextWindowMonitorHook(ctx, modelCacheState))
    : null
  // ... 其他 22 个钩子
}
```

### 2. Tool Guard Hooks (12 个)

Tool Guard Hooks 在工具执行前后提供保护和增强功能，包括输出截断、规则注入、错误恢复等。这些钩子确保工具调用的安全性和可靠性。

**执行顺序：**

```
1. comment-checker            # 注释检查器
2. tool-output-truncator      # 工具输出截断
3. directory-agents-injector  # 目录 Agents 注入
4. directory-readme-injector  # 目录 README 注入
5. empty-task-response-detector  # 空任务响应检测
6. rules-injector             # 规则注入器
7. tasks-todowrite-disabler   # 任务 Todo 写入禁用
8. write-existing-file-guard  # 写入现有文件保护
9. hashline-read-enhancer     # Hashline 读取增强
10. json-error-recovery       # JSON 错误恢复
11. read-image-resizer        # 读取图片调整大小
12. todo-description-override # Todo 描述覆盖
13. webfetch-redirect-guard   # Webfetch 重定向保护
```

**代码引用：**

```typescript
// src/plugin/hooks/create-tool-guard-hooks.ts:44-141
export function createToolGuardHooks(args: {
  ctx: PluginContext
  pluginConfig: OhMyOpenCodeConfig
  modelCacheState: ModelCacheState
  isHookEnabled: (hookName: HookName) => boolean
  safeHookEnabled: boolean
}): ToolGuardHooks {
  const { ctx, pluginConfig, modelCacheState, isHookEnabled, safeHookEnabled } = args
  const safeHook = <T>(hookName: HookName, factory: () => T): T | null =>
    safeCreateHook(hookName, factory, { enabled: safeHookEnabled })

  const commentChecker = isHookEnabled("comment-checker")
    ? safeHook("comment-checker", () => createCommentCheckerHooks(pluginConfig.comment_checker))
    : null
  // ... 其他 11 个钩子
}
```

### 3. Transform Hooks (4 个)

Transform Hooks 负责消息内容的转换和验证，在消息发送到模型之前或从模型返回之后进行处理。

**执行顺序：**

```
1. claude-code-hooks          # Claude Code 钩子
2. keyword-detector           # 关键词检测器
3. context-injector-messages-transform  # 上下文注入消息转换
4. thinking-block-validator   # 思考块验证器
```

**代码引用：**

```typescript
// src/plugin/hooks/create-transform-hooks.ts:22-72
export function createTransformHooks(args: {
  ctx: PluginContext
  pluginConfig: OhMyOpenCodeConfig
  isHookEnabled: (hookName: string) => boolean
  safeHookEnabled?: boolean
}): TransformHooks {
  const { ctx, pluginConfig, isHookEnabled } = args
  const safeHookEnabled = args.safeHookEnabled ?? true

  const claudeCodeHooks = isHookEnabled("claude-code-hooks")
    ? safeCreateHook("claude-code-hooks", () =>
        createClaudeCodeHooksHook(ctx, { ... }, contextCollector),
      { enabled: safeHookEnabled })
    : null
  // ... 其他 3 个钩子
}
```

### 4. Continuation Hooks (7 个)

Continuation Hooks 管理任务的中断、恢复和延续，确保长时间运行的任务能够在各种异常情况下正确恢复。

**执行顺序：**

```
1. stop-continuation-guard    # 停止延续保护
2. compaction-context-injector  # 压缩上下文注入
3. compaction-todo-preserver  # 压缩 Todo 保留
4. todo-continuation-enforcer # Todo 延续执行器
5. unstable-agent-babysitter  # 不稳定 Agent 看护
6. background-notification    # 后台通知
7. atlas                      # Atlas 钩子
```

**代码引用：**

```typescript
// src/plugin/hooks/create-continuation-hooks.ts:31-128
export function createContinuationHooks(args: {
  ctx: PluginContext
  pluginConfig: OhMyOpenCodeConfig
  isHookEnabled: (hookName: HookName) => boolean
  safeHookEnabled: boolean
  backgroundManager: BackgroundManager
  sessionRecovery: SessionRecovery
}): ContinuationHooks {
  const stopContinuationGuard = isHookEnabled("stop-continuation-guard")
    ? safeHook("stop-continuation-guard", () =>
        createStopContinuationGuardHook(ctx, { backgroundManager }))
    : null
  // ... 其他 6 个钩子
}
```

### 5. Skill Hooks (2 个)

Skill Hooks 提供与技能系统相关的辅助功能，在会话中自动注入技能相关的上下文和命令支持。

**执行顺序：**

```
1. category-skill-reminder    # 分类技能提醒
2. auto-slash-command         # 自动斜杠命令
```

**代码引用：**

```typescript
// src/plugin/hooks/create-skill-hooks.ts:14-49
export function createSkillHooks(args: {
  ctx: PluginContext
  pluginConfig: OhMyOpenCodeConfig
  isHookEnabled: (hookName: HookName) => boolean
  safeHookEnabled: boolean
  mergedSkills: LoadedSkill[]
  availableSkills: AvailableSkill[]
}): SkillHooks {
  const categorySkillReminder = isHookEnabled("category-skill-reminder")
    ? safeHook("category-skill-reminder", () =>
        createCategorySkillReminderHook(ctx, availableSkills))
    : null

  const autoSlashCommand = isHookEnabled("auto-slash-command")
    ? safeHook("auto-slash-command", () =>
        createAutoSlashCommandHook({ skills: mergedSkills, ... }))
    : null

  return { categorySkillReminder, autoSlashCommand }
}
```

## Hook 执行流程

完整的 Hook 执行流程遵循以下顺序：

1. **初始化阶段**：`createHooks()` 被调用，依次创建 Core、Continuation 和 Skill 三层钩子
2. **Core Hooks 组合**：`createCoreHooks()` 组合 Session、Tool Guard 和 Transform 三层
3. **条件创建**：每个钩子通过 `isHookEnabled()` 检查是否启用，通过 `safeCreateHook()` 包装以隔离错误
4. **依赖注入**：Continuation Hooks 接收 Session Recovery 回调，建立跨层依赖
5. **运行时执行**：OpenCode 在特定生命周期点调用对应的钩子处理函数

## 流程/架构图

```
┌────────────────────────────────────────────────────────────────────┐
│                        OpenCode 生命周期                            │
└────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌────────────────────────────────────────────────────────────────────┐
│  Phase 1: Session Hooks (23)                                       │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                │
│  │ context-    │  │ session-    │  │ think-mode  │  ...           │
│  │ window-     │  │ recovery    │  │             │                │
│  │ monitor     │  │             │  │             │                │
│  └─────────────┘  └─────────────┘  └─────────────┘                │
└────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌────────────────────────────────────────────────────────────────────┐
│  Phase 2: Tool Guard Hooks (12)                                    │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                │
│  │ comment-    │  │ tool-output-│  │ rules-      │  ...           │
│  │ checker     │  │ truncator   │  │ injector    │                │
│  └─────────────┘  └─────────────┘  └─────────────┘                │
└────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌────────────────────────────────────────────────────────────────────┐
│  Phase 3: Transform Hooks (4)                                      │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                │
│  │ claude-code │  │ keyword-    │  │ context-    │                │
│  │ -hooks      │  │ detector    │  │ injector    │                │
│  └─────────────┘  └─────────────┘  └─────────────┘                │
└────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌────────────────────────────────────────────────────────────────────┐
│  Phase 4: Continuation Hooks (7)                                   │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                │
│  │ stop-       │  │ todo-       │  │ atlas       │  ...           │
│  │ continuation│  │ continuation│  │             │                │
│  │ -guard      │  │ -enforcer   │  │             │                │
│  └─────────────┘  └─────────────┘  └─────────────┘                │
└────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌────────────────────────────────────────────────────────────────────┐
│  Phase 5: Skill Hooks (2)                                          │
│  ┌─────────────┐  ┌─────────────┐                                  │
│  │ category-   │  │ auto-slash  │                                  │
│  │ skill-      │  │ -command    │                                  │
│  │ reminder    │  │             │                                  │
│  └─────────────┘  └─────────────┘                                  │
└────────────────────────────────────────────────────────────────────┘
```

## 关键代码片段

### 片段 1: Hook 主入口组合

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

### 片段 2: 安全 Hook 包装器

```typescript
// src/shared/safe-create-hook.ts
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
    log(`Failed to create hook ${hookName}:`, error)
    return null
  }
}
```

### 片段 3: Core Hooks 聚合

```typescript
// src/plugin/hooks/create-core-hooks.ts:9-46
export function createCoreHooks(args: {
  ctx: PluginContext
  pluginConfig: OhMyOpenCodeConfig
  modelCacheState: ModelCacheState
  isHookEnabled: (hookName: HookName) => boolean
  safeHookEnabled: boolean
}) {
  const { ctx, pluginConfig, modelCacheState, isHookEnabled, safeHookEnabled } = args

  const session = createSessionHooks({
    ctx,
    pluginConfig,
    modelCacheState,
    isHookEnabled,
    safeHookEnabled,
  })

  const tool = createToolGuardHooks({
    ctx,
    pluginConfig,
    modelCacheState,
    isHookEnabled,
    safeHookEnabled,
  })

  const transform = createTransformHooks({
    ctx,
    pluginConfig,
    isHookEnabled: (name) => isHookEnabled(name as HookName),
    safeHookEnabled,
  })

  return {
    ...session,
    ...tool,
    ...transform,
  }
}
```

## 依赖关系

Hook 执行顺序依赖以下核心组件：

1. **PluginContext**：提供 OpenCode SDK 客户端和目录信息
2. **OhMyOpenCodeConfig**：插件配置，控制哪些钩子启用
3. **ModelCacheState**：模型缓存状态，用于上下文窗口监控
4. **BackgroundManager**：后台任务管理器，Continuation Hooks 依赖
5. **SessionRecovery**：会话恢复机制，跨层共享

## 实战示例

### 示例 1: Tool 执行钩子链

当用户调用 `edit` 工具修改文件时，完整的钩子执行链如下：

```
1. Session Hooks:
   - context-window-monitor: 检查上下文窗口使用率
   - think-mode: 如果启用思考模式，调整参数

2. Tool Guard Hooks (tool.execute.before):
   - write-existing-file-guard: 检查是否写入已存在文件
   - rules-injector: 注入当前目录的 .sisyphus/rules
   - hashline-read-enhancer: 如果启用 hashline，增强读取

3. Tool 执行: OpenCode 执行实际的 edit 操作

4. Tool Guard Hooks (tool.execute.after):
   - comment-checker: 检查生成的注释质量
   - tool-output-truncator: 截断过长的输出
   - json-error-recovery: 如果返回 JSON 错误，尝试恢复

5. Transform Hooks:
   - thinking-block-validator: 验证思考块格式
   - context-injector-messages-transform: 注入 AGENTS.md 上下文

6. Continuation Hooks:
   - todo-continuation-enforcer: 检查是否有未完成的 todo
   - stop-continuation-guard: 检查是否应该停止延续
```

### 示例 2: 会话事件钩子链

当新会话创建时，钩子执行顺序：

```
1. Session Hooks:
   - session-recovery: 检查是否有可恢复的会话状态
   - auto-update-checker: 检查插件是否有更新
   - agent-usage-reminder: 显示 Agent 使用提示
   - non-interactive-env: 检测非交互环境并调整行为
   - ralph-loop: 初始化 Ralph 循环状态

2. Skill Hooks:
   - category-skill-reminder: 根据分类注入技能提示
   - auto-slash-command: 注册技能斜杠命令

3. Transform Hooks:
   - keyword-detector: 检测消息中的关键词
   - claude-code-hooks: 应用 Claude Code 兼容处理

4. Continuation Hooks:
   - atlas: 初始化 Atlas 状态跟踪
   - background-notification: 设置后台任务通知
```

## 交叉引用

- 参见：[Hook 分层](../03-Hook 系统/01-Hook 分层.md) - 详细了解五层钩子的设计原理和职责划分
- 参见：[Session Hooks](../03-Hook 系统/02-Session Hooks.md) - 23 个会话钩子的详细说明
- 参见：[Tool Guard Hooks](../03-Hook 系统/03-Tool Guard Hooks.md) - 12 个工具保护钩子的工作机制
- 参见：[Transform Hooks](../03-Hook 系统/04-Transform Hooks.md) - 消息转换钩子的实现细节
- 参见：[Continuation Hooks](../03-hooks-system/continuation-hooks.md) - 任务延续钩子的恢复机制
- 参见：[Skill Hooks](../03-hooks-system/skill-hooks.md) - 技能系统钩子的集成方式
- 参见：[Plugin Interface](../02-core-architecture/plugin-interface.md) - 8 个 OpenCode Hook Handler 的实现
