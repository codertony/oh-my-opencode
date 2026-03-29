# Tool Guard Hooks：12 个工具守卫钩子

> 所属模块：03-hooks-system | 三层模型：Layer 3 | 优先级：P1

---

## 架构位置

Tool Guard Hooks 位于三层钩子模型的第三层（Layer 3），专门负责在工具执行前后进行拦截、验证和增强。

```
┌─────────────────────────────────────────────────────────────┐
│                    OpenCode Plugin                          │
├─────────────────────────────────────────────────────────────┤
│  Layer 1: Session Hooks (23)                                │
│  Layer 2: Transform Hooks (4)                               │
│  Layer 3: Tool Guard Hooks (12) ◄── 你在这里                │
│  Layer 4: Continuation Hooks (7)                            │
│  Layer 5: Skill Hooks (2)                                   │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  tool.execute.before → [Guard Chain] → Execute →            │
│  tool.execute.after → [Guard Chain] → Return                │
└─────────────────────────────────────────────────────────────┘
```

## 核心职责

Tool Guard Hooks 是 oh-my-opencode 插件中专门负责工具执行安全与增强的守卫层。它们工作在工具调用的关键路径上，在 `tool.execute.before` 和 `tool.execute.after` 两个生命周期点进行拦截处理。

这些钩子的核心职责包括：

1. **安全防护**：防止危险操作，如覆盖未读取的现有文件（writeExistingFileGuard）
2. **内容检查**：检测并阻止 AI 生成的低质量注释模式（commentChecker）
3. **错误恢复**：自动检测 JSON 解析错误并提供修复提示（jsonErrorRecovery）
4. **上下文注入**：自动注入目录级别的 AGENTS.md 和 README.md（directoryAgentsInjector、directoryReadmeInjector）
5. **输出优化**：截断过长的工具输出，调整图像大小以节省 token（toolOutputTruncator、readImageResizer）
6. **功能增强**：为 Read 工具输出添加行哈希标识（hashlineReadEnhancer），支持 Hashline Edit 功能

每个钩子都可以独立启用或禁用，通过配置文件中的 `disabled_hooks` 数组进行控制。

## 代码索引 (Code Index)

| 功能点 | 文件路径 | 行号 | 函数/类 | 说明 |
|--------|----------|------|---------|------|
| 工具守卫钩子创建器 | `src/plugin/hooks/create-tool-guard-hooks.ts` | 44-141 | `createToolGuardHooks` | 创建所有 12 个工具守卫钩子 |
| 注释检查器钩子 | `src/hooks/comment-checker/hook.ts` | 43-184 | `createCommentCheckerHooks` | 检测 AI 生成的注释模式 |
| 工具输出截断器 | `src/hooks/tool-output-truncator.ts` | 37-66 | `createToolOutputTruncatorHook` | 截断过长工具输出 |
| 目录 AGENTS 注入器 | `src/hooks/directory-agents-injector/hook.ts` | 30-87 | `createDirectoryAgentsInjectorHook` | 自动注入 AGENTS.md |
| 目录 README 注入器 | `src/hooks/directory-readme-injector/hook.ts` | 30-87 | `createDirectoryReadmeInjectorHook` | 自动注入 README.md |
| 空任务响应检测器 | `src/hooks/empty-task-response-detector.ts` | 12-27 | `createEmptyTaskResponseDetectorHook` | 检测空任务响应 |
| 规则注入器 | `src/hooks/rules-injector/hook.ts` | 32-90 | `createRulesInjectorHook` | 条件规则注入 |
| 任务 TodoWrite 禁用器 | `src/hooks/tasks-todowrite-disabler/hook.ts` | 9-33 | `createTasksTodowriteDisablerHook` | 禁用 TodoWrite 工具 |
| 写入现有文件守卫 | `src/hooks/write-existing-file-guard/hook.ts` | 74-254 | `createWriteExistingFileGuardHook` | 防止覆盖未读文件 |
| Hashline 读取增强器 | `src/hooks/hashline-read-enhancer/hook.ts` | 169-193 | `createHashlineReadEnhancerHook` | 添加行哈希标识 |
| JSON 错误恢复 | `src/hooks/json-error-recovery/hook.ts` | 41-58 | `createJsonErrorRecoveryHook` | JSON 解析错误恢复 |
| 读取图像调整器 | `src/hooks/read-image-resizer/hook.ts` | 115-197 | `createReadImageResizerHook` | 调整图像大小 |
| Todo 描述覆盖器 | `src/hooks/todo-description-override/hook.ts` | 3-14 | `createTodoDescriptionOverrideHook` | 覆盖 TodoWrite 描述 |
| WebFetch 重定向守卫 | `src/hooks/webfetch-redirect-guard/hook.ts` | 68-123 | `createWebFetchRedirectGuardHook` | 处理重定向循环 |

## 12 个 Tool Guard Hooks

### 1. commentChecker（注释检查器）

**功能**：检测并阻止 AI 生成的低质量注释模式，确保代码注释符合人类编写标准。

**守卫逻辑**：
- 在 `tool.execute.before` 阶段注册待处理的写操作
- 在 `tool.execute.after` 阶段使用 CLI 工具检查文件内容
- 支持 Write、Edit、MultiEdit 和 ApplyPatch 工具
- 检测 AI 常见的注释模式（如过度详细的函数注释、模板化注释等）

```typescript
// src/hooks/comment-checker/hook.ts:43-96
export function createCommentCheckerHooks(config?: CommentCheckerConfig) {
  return {
    "tool.execute.before": async (input, output) => {
      const toolLower = input.tool.toLowerCase()
      if (toolLower !== "write" && toolLower !== "edit" && toolLower !== "multiedit") {
        return
      }
      // 注册待处理调用，保存文件路径和内容
      registerPendingCall(input.callID, { filePath, content, ... })
    },
    "tool.execute.after": async (input, output) => {
      const pendingCall = takePendingCall(input.callID)
      if (!pendingCall) return
      // 使用 CLI 检查注释质量
      await processWithCli(input, pendingCall, output, cliPath, config?.custom_prompt)
    }
  }
}
```

### 2. writeExistingFileGuard（写入现有文件守卫）

**功能**：防止 AI 在未先读取文件的情况下覆盖现有文件，避免意外数据丢失。

**守卫逻辑**：
- 跟踪每个会话中已读取的文件路径
- 在 Write 工具执行前检查目标文件是否已存在
- 如果文件存在且未被读取过，抛出错误阻止写入
- 支持 `overwrite` 参数绕过守卫（仅限明确声明）

```typescript
// src/hooks/write-existing-file-guard/hook.ts:160-238
"tool.execute.before": async (input, output) => {
  const toolName = input.tool?.toLowerCase()
  if (toolName === "read") {
    // 记录已读取的文件路径
    registerReadPermission(input.sessionID, canonicalPath)
    return
  }
  if (toolName === "write") {
    // 检查文件是否存在且未被读取
    if (input.sessionID && consumeReadPermission(input.sessionID, canonicalPath)) {
      // 允许写入（已读取过）
      return
    }
    // 阻止写入并抛出错误
    throw new Error("File already exists. Use edit tool instead.")
  }
}
```

### 3. rulesInjector（规则注入器）

**功能**：在读取、写入或编辑文件时，自动注入相关的 AGENTS.md 规则文件内容。

**守卫逻辑**：
- 监听 Read、Write、Edit、MultiEdit 工具的执行
- 从工具输出中提取文件路径
- 查找该路径附近的 AGENTS.md 文件（按距离优先级）
- 使用会话缓存避免重复注入
- 支持动态截断以适应不同模型的上下文窗口

```typescript
// src/hooks/rules-injector/hook.ts:44-56
const toolExecuteAfter = async (input, output) => {
  const toolName = input.tool.toLowerCase()
  if (TRACKED_TOOLS.includes(toolName)) {
    const filePath = getRuleInjectionFilePath(output)
    if (!filePath) return
    await processFilePathForInjection(filePath, input.sessionID, output)
  }
}
```

### 4. jsonErrorRecovery（JSON 错误恢复）

**功能**：检测工具调用中的 JSON 解析错误，并注入修复提示。

**守卫逻辑**：
- 在 `tool.execute.after` 阶段检查输出内容
- 使用正则表达式匹配常见的 JSON 错误模式
- 排除特定工具（如 bash、read、glob 等非结构化输出工具）
- 注入详细的错误修复指南

```typescript
// src/hooks/json-error-recovery/hook.ts:14-40
export const JSON_ERROR_PATTERNS = [
  /json parse error/i,
  /failed to parse json/i,
  /invalid json/i,
  /malformed json/i,
  /unexpected end of json input/i,
]

export const JSON_ERROR_REMINDER = `
[JSON PARSE ERROR - IMMEDIATE ACTION REQUIRED]
You sent invalid JSON arguments. The system could not parse your tool call.
STOP and do this NOW:
1. LOOK at the error message above to see what was expected vs what you sent.
2. CORRECT your JSON syntax (missing braces, unescaped quotes, trailing commas, etc).
3. RETRY the tool call with valid JSON.
`
```

### 5. hashlineReadEnhancer（Hashline 读取增强器）

**功能**：为 Read 工具的输出添加行哈希标识，支持 Hashline Edit 功能。

**守卫逻辑**：
- 仅在 `hashline_edit.enabled` 为 true 时启用
- 解析 Read 工具的输出格式（行号 + 内容）
- 为每行内容计算哈希值并附加到行号后
- 输出格式：`行号#哈希|内容`

```typescript
// src/hooks/hashline-read-enhancer/hook.ts:55-65
function transformLine(line: string): string {
  const parsed = parseReadLine(line)
  if (!parsed) return line
  const hash = computeLineHash(parsed.lineNumber, parsed.content)
  return `${parsed.lineNumber}#${hash}|${parsed.content}`
}

// 示例输出：
// 11#VK| function hello() {
// 22#XJ|   return "world";
// 33#MB| }
```

### 6. toolOutputTruncator（工具输出截断器）

**功能**：截断过长的工具输出，防止上下文窗口溢出。

**守卫逻辑**：
- 针对特定工具（grep、glob、lsp_diagnostics、webfetch 等）进行截断
- 默认最大 50,000 tokens（约 200k 字符）
- WebFetch 工具使用更激进的截断（10,000 tokens）
- 使用动态截断器根据模型上下文窗口自适应调整

```typescript
// src/hooks/tool-output-truncator.ts:37-66
export function createToolOutputTruncatorHook(ctx, options) {
  const truncator = createDynamicTruncator(ctx, options?.modelCacheState)
  return {
    "tool.execute.after": async (input, output) => {
      if (!TRUNCATABLE_TOOLS.includes(input.tool)) return
      const targetMaxTokens = TOOL_SPECIFIC_MAX_TOKENS[input.tool] ?? DEFAULT_MAX_TOKENS
      const { result, truncated } = await truncator.truncate(
        input.sessionID,
        output.output,
        { targetMaxTokens }
      )
      if (truncated) output.output = result
    }
  }
}
```

### 7. directoryAgentsInjector（目录 AGENTS 注入器）

**功能**：在读取文件时，自动注入该目录下的 AGENTS.md 文件内容。

**守卫逻辑**：
- 监听 Read 工具的执行
- 从输出标题中提取文件路径
- 查找并注入该文件所在目录的 AGENTS.md
- 使用会话缓存避免重复注入
- 在 session.deleted 和 session.compacted 事件时清理缓存

### 8. directoryReadmeInjector（目录 README 注入器）

**功能**：与 directoryAgentsInjector 类似，但注入 README.md 文件。

**守卫逻辑**：
- 与 AGENTS 注入器相同的机制
- 优先注入目录级别的 README.md
- 支持动态截断以适应不同模型

### 9. emptyTaskResponseDetector（空任务响应检测器）

**功能**：检测 Task 工具返回空响应的情况，并注入警告信息。

**守卫逻辑**：
- 监听 Task 工具的执行结果
- 检查输出内容是否为空或仅包含空白字符
- 注入警告信息提醒用户任务可能未正确执行

```typescript
// src/hooks/empty-task-response-detector.ts:12-27
export function createEmptyTaskResponseDetectorHook(_ctx) {
  return {
    "tool.execute.after": async (input, output) => {
      if (input.tool !== "Task" && input.tool !== "task") return
      const responseText = output.output?.trim() ?? ""
      if (responseText === "") {
        output.output = EMPTY_RESPONSE_WARNING
      }
    }
  }
}
```

### 10. tasksTodowriteDisabler（任务 TodoWrite 禁用器）

**功能**：当任务系统启用时，禁用 TodoWrite 工具以防止冲突。

**守卫逻辑**：
- 检查配置中的 `experimental.task_system` 是否启用
- 在 `tool.execute.before` 阶段拦截 TodoWrite 工具调用
- 抛出错误阻止执行，提示使用任务系统替代

```typescript
// src/hooks/tasks-todowrite-disabler/hook.ts:9-33
export function createTasksTodowriteDisablerHook(config) {
  const isTaskSystemEnabled = config.experimental?.task_system ?? false
  return {
    "tool.execute.before": async (input, _output) => {
      if (!isTaskSystemEnabled) return
      if (BLOCKED_TOOLS.some(blocked => blocked.toLowerCase() === input.tool.toLowerCase())) {
        throw new Error(REPLACEMENT_MESSAGE)
      }
    }
  }
}
```

### 11. readImageResizer（读取图像调整器）

**功能**：在读取图像文件时自动调整图像大小，减少 token 消耗。

**守卫逻辑**：
- 仅对 Anthropic 提供商的会话生效
- 支持 PNG、JPEG、GIF、WebP 格式
- 计算图像 token 消耗（宽 × 高 / 750）
- 自动调整超过限制的大图像
- 在输出中附加调整信息

### 12. todoDescriptionOverride（Todo 描述覆盖器）

**功能**：覆盖 TodoWrite 工具的描述，提供更详细的用法说明。

**守卫逻辑**：
- 在 `tool.definition` 阶段拦截 TodoWrite 工具定义
- 替换为标准化的详细描述
- 帮助 AI 更好地理解和使用待办事项系统

```typescript
// src/hooks/todo-description-override/hook.ts:3-14
export function createTodoDescriptionOverrideHook() {
  return {
    "tool.definition": async (input, output) => {
      if (input.toolID === "todowrite") {
        output.description = TODOWRITE_DESCRIPTION
      }
    }
  }
}
```

### 13. webfetchRedirectGuard（WebFetch 重定向守卫）

**功能**：处理 WebFetch 工具的重定向循环问题，预先解析重定向链。

**守卫逻辑**：
- 在 `tool.execute.before` 阶段预解析 URL 重定向
- 如果重定向次数超过限制，记录失败信息
- 在 `tool.execute.after` 阶段检查是否发生重定向错误
- 提供更清晰的错误信息

```typescript
// src/hooks/webfetch-redirect-guard/hook.ts:72-104
"tool.execute.before": async (input, output) => {
  if (!isWebFetchTool(input.tool)) return
  const url = getWebFetchUrl(output.args)
  const resolution = await resolveWebFetchRedirects({ url, format, timeoutSeconds })
  if (resolution.type === "resolved") {
    output.args.url = resolution.url  // 使用解析后的最终 URL
  } else {
    pendingFailures.set(key, { originalUrl: url, storedAt: Date.now() })
  }
}
```

## 工具执行守卫机制

Tool Guard Hooks 采用双向守卫机制，在工具执行的前后两个关键点进行拦截：

```
用户请求 → tool.execute.before → [Guard Chain] → 实际执行 → tool.execute.after → [Guard Chain] → 返回结果
                │                                          │
                ▼                                          ▼
         ┌─────────────┐                           ┌─────────────┐
         │ Before 守卫  │                           │ After 守卫   │
         ├─────────────┤                           ├─────────────┤
         │ • 权限检查   │                           │ • 结果处理   │
         │ • 参数验证   │                           │ • 错误恢复   │
         │ • 前置注入   │                           │ • 内容增强   │
         │ • 拦截阻止   │                           │ • 后处理    │
         └─────────────┘                           └─────────────┘
```

### Before 守卫（执行前）

Before 守卫主要用于：
- **权限验证**：如 writeExistingFileGuard 检查文件读取权限
- **参数修改**：如 webfetchRedirectGuard 修改 URL 参数
- **拦截阻止**：如 tasksTodowriteDisabler 阻止特定工具调用
- **前置注册**：如 commentChecker 注册待处理的写操作

### After 守卫（执行后）

After 守卫主要用于：
- **结果处理**：如 toolOutputTruncator 截断过长输出
- **错误恢复**：如 jsonErrorRecovery 检测并修复 JSON 错误
- **内容增强**：如 hashlineReadEnhancer 添加行哈希标识
- **上下文注入**：如 rulesInjector 注入规则文件内容

## 流程/架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Tool Guard Hooks 执行流程                      │
└─────────────────────────────────────────────────────────────────────┘

  ┌──────────────┐
  │ 工具调用请求  │
  └──────┬───────┘
         │
         ▼
  ┌──────────────────────────────────────────────────────────────┐
  │                    tool.execute.before                        │
  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐             │
  │  │writeExisting│ │webfetch     │ │comment      │             │
  │  │FileGuard    │ │RedirectGuard│ │Checker      │             │
  │  │(权限检查)    │ │(URL解析)    │ │(注册待处理)  │             │
  │  └─────────────┘ └─────────────┘ └─────────────┘             │
  │  ┌─────────────┐ ┌─────────────┐                              │
  │  │tasksTodo    │ │rulesInjector│                              │
  │  │writeDisabler│ │(前置处理)    │                              │
  │  │(拦截禁用)    │ │             │                              │
  │  └─────────────┘ └─────────────┘                              │
  └────────────────────────┬─────────────────────────────────────┘
                           │
                           ▼
                    ┌──────────────┐
                    │   实际执行    │
                    │   工具调用    │
                    └──────┬───────┘
                           │
                           ▼
  ┌──────────────────────────────────────────────────────────────┐
  │                    tool.execute.after                         │
  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐             │
  │  │toolOutput   │ │jsonError    │ │hashline     │             │
  │  │Truncator    │ │Recovery     │ │ReadEnhancer │             │
  │  │(截断输出)    │ │(错误恢复)    │ │(添加哈希)    │             │
  │  └─────────────┘ └─────────────┘ └─────────────┘             │
  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐             │
  │  │directory    │ │directory    │ │comment      │             │
  │  │AgentsInject │ │ReadmeInject │ │Checker      │             │
  │  │(注入AGENTS) │ │(注入README) │ │(检查注释)    │             │
  │  └─────────────┘ └─────────────┘ └─────────────┘             │
  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐             │
  │  │emptyTask    │ │readImage    │ │rulesInjector│             │
  │  │Detector     │ │Resizer      │ │(注入规则)    │             │
  │  │(检测空响应)  │ │(调整图像)    │ │             │             │
  │  └─────────────┘ └─────────────┘ └─────────────┘             │
  └────────────────────────┬─────────────────────────────────────┘
                           │
                           ▼
                    ┌──────────────┐
                    │   返回结果    │
                    │  给 AI 模型   │
                    └──────────────┘
```

## 关键代码片段

### 片段 1：工具守卫钩子创建器

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

  // 创建所有 12 个工具守卫钩子
  const commentChecker = isHookEnabled("comment-checker")
    ? safeHook("comment-checker", () => createCommentCheckerHooks(pluginConfig.comment_checker))
    : null

  const toolOutputTruncator = isHookEnabled("tool-output-truncator")
    ? safeHook("tool-output-truncator", () =>
        createToolOutputTruncatorHook(ctx, { modelCacheState, experimental: pluginConfig.experimental }))
    : null

  // ... 其他钩子的创建

  return {
    commentChecker,
    toolOutputTruncator,
    directoryAgentsInjector,
    directoryReadmeInjector,
    emptyTaskResponseDetector,
    rulesInjector,
    tasksTodowriteDisabler,
    writeExistingFileGuard,
    hashlineReadEnhancer,
    jsonErrorRecovery,
    readImageResizer,
    todoDescriptionOverride,
    webfetchRedirectGuard,
  }
}
```

### 片段 2：写入现有文件守卫核心逻辑

```typescript
// src/hooks/write-existing-file-guard/hook.ts:74-147
export function createWriteExistingFileGuardHook(ctx: PluginInput): Hooks {
  const readPermissionsBySession = new Map<string, Set<string>>()
  const MAX_TRACKED_PATHS_PER_SESSION = 1024

  const registerReadPermission = (sessionID: string, canonicalPath: string): void => {
    const readSet = ensureSessionReadSet(sessionID)
    if (readSet.has(canonicalPath)) {
      readSet.delete(canonicalPath)
    }
    readSet.add(canonicalPath)
    trimSessionReadSet(readSet)
  }

  const consumeReadPermission = (sessionID: string, canonicalPath: string): boolean => {
    const readSet = readPermissionsBySession.get(sessionID)
    if (!readSet || !readSet.has(canonicalPath)) {
      return false
    }
    readSet.delete(canonicalPath)
    touchSession(sessionID)
    return true
  }

  return {
    "tool.execute.before": async (input, output) => {
      const toolName = input.tool?.toLowerCase()
      if (toolName === "read") {
        registerReadPermission(input.sessionID, canonicalPath)
        return
      }
      if (toolName === "write" && existsSync(resolvedPath)) {
        if (!consumeReadPermission(input.sessionID, canonicalPath)) {
          throw new Error("File already exists. Use edit tool instead.")
        }
      }
    }
  }
}
```

### 片段 3：Hashline 读取增强器

```typescript
// src/hooks/hashline-read-enhancer/hook.ts:55-65, 169-193
function transformLine(line: string): string {
  const parsed = parseReadLine(line)
  if (!parsed) return line
  if (parsed.content.endsWith(OPENCODE_LINE_TRUNCATION_SUFFIX)) {
    return line
  }
  const hash = computeLineHash(parsed.lineNumber, parsed.content)
  return `${parsed.lineNumber}#${hash}|${parsed.content}`
}

export function createHashlineReadEnhancerHook(_ctx, config) {
  return {
    "tool.execute.after": async (input, output) => {
      if (!isReadTool(input.tool)) return
      if (typeof output.output !== "string") return
      if (!shouldProcess(config)) return
      
      // 转换输出，添加行哈希
      output.output = transformOutput(output.output)
    }
  }
}

// 转换前：
// 1: function hello() {
// 2:   return "world";
// 3: }

// 转换后：
// 1#VK| function hello() {
// 2#XJ|   return "world";
// 3#MB| }
```

## 依赖关系

Tool Guard Hooks 依赖以下组件：

| 依赖组件 | 用途 |
|----------|------|
| `PluginContext` | 获取工作目录、配置等上下文信息 |
| `ModelCacheState` | 获取模型上下文窗口大小，用于动态截断 |
| `DynamicTruncator` | 动态截断内容以适应模型上下文限制 |
| `safeCreateHook` | 安全包装器，防止单个钩子错误影响整个链 |
| `computeLineHash` | 计算行内容哈希，用于 Hashline Edit |

依赖关系图：

```
Tool Guard Hooks
    │
    ├─── PluginContext (ctx.directory, ctx.logger)
    │
    ├─── ModelCacheState (anthropicContext1MEnabled)
    │
    ├─── DynamicTruncator (truncate, adaptive sizing)
    │
    ├─── safeCreateHook (error isolation)
    │
    └─── computeLineHash (hashline support)
```

## 实战示例

### 示例：writeExistingFileGuard 防止数据丢失

**场景**：AI 尝试写入一个已存在的配置文件，但未先读取其内容。

**执行流程**：

1. **用户请求**：AI 调用 Write 工具写入 `config/database.json`

2. **Before 守卫拦截**：
   ```typescript
   // writeExistingFileGuard 检查
   const toolName = "write"
   const filePath = "config/database.json"
   const canonicalPath = "/project/config/database.json"
   
   // 检查文件是否存在
   if (existsSync(resolvedPath)) {  // true
     // 检查是否已读取过
     const hasReadPermission = consumeReadPermission(sessionID, canonicalPath)  // false
     if (!hasReadPermission) {
       throw new Error("File already exists. Use edit tool instead.")
     }
   }
   ```

3. **错误返回**：
   ```
   Error: File already exists. Use edit tool instead.
   ```

4. **AI 修正**：AI 收到错误后，先使用 Read 工具读取文件：
   ```
   Read file: config/database.json
   ```

5. **读取权限记录**：
   ```typescript
   // writeExistingFileGuard 记录读取
   registerReadPermission(sessionID, "/project/config/database.json")
   ```

6. **再次写入**：AI 再次调用 Write 工具

7. **守卫通过**：
   ```typescript
   // 这次 consumeReadPermission 返回 true
   if (consumeReadPermission(sessionID, canonicalPath)) {
     // 允许写入，同时使其他会话的缓存失效
     invalidateOtherSessions(canonicalPath, sessionID)
     return  // 不抛出错误
   }
   ```

8. **写入成功**：文件被安全覆盖

这个示例展示了 writeExistingFileGuard 如何通过权限跟踪机制，确保 AI 在覆盖现有文件前已经了解其内容，有效防止意外数据丢失。

## 交叉引用

- 参见：[Hook 分层架构](./01-Hook 分层.md) - 了解三层钩子模型的整体设计
- 参见：[Session Hooks](./02-Session Hooks.md) - 第一层会话生命周期钩子
- 参见：[Transform Hooks](./04-Transform Hooks.md) - 第二层消息转换钩子
- 参见：[权限层](../01-Harness 架构/04-权限层.md) - 了解权限系统如何与工具守卫配合
- 参见：[Hashline Edit 工具](../02-tools-system/hashline-edit.md) - Hashline Read Enhancer 支持的核心功能
- 参见：[AGENTS.md 注入机制](../04-context-system/agents-injection.md) - Rules Injector 和 Directory Injectors 的详细说明
- 参见：[配置系统](../05-config-system/hook-config.md) - 如何启用/禁用特定工具守卫钩子
