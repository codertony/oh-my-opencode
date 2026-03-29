# Session Hooks：23 个会话生命周期钩子

> 所属模块：03-hooks-system | 三层模型：Layer 3 | 优先级：P1

---

## 架构位置

Session Hooks 位于 Hook 三层模型的第一层（Layer 3），直接处理 OpenCode 的会话生命周期事件。

```
┌─────────────────────────────────────────────────────────────┐
│                    OpenCode Core                            │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │ chat.created│  │ chat.deleted│  │    chat.idle        │  │
│  └──────┬──────┘  └──────┬──────┘  └──────────┬──────────┘  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │ chat.error  │  │chat.message │  │    chat.params      │  │
│  └──────┬──────┘  └──────┬──────┘  └──────────┬──────────┘  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │chat.headers │  │   event.*   │  │    tool.*           │  │
│  └──────┬──────┘  └──────┬──────┘  └──────────┬──────────┘  │
└─────────┼────────────────┼────────────────────┼─────────────┘
          │                │                    │
          ▼                ▼                    ▼
┌─────────────────────────────────────────────────────────────┐
│              Session Hooks (23 hooks)                       │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────────────┐ │
│  │contextWindow │ │sessionRecovery│ │  sessionNotification │ │
│  │   Monitor    │ │              │ │                      │ │
│  └──────────────┘ └──────────────┘ └──────────────────────┘ │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────────────┐ │
│  │  thinkMode   │ │ modelFallback │ │  anthropicEffort     │ │
│  └──────────────┘ └──────────────┘ └──────────────────────┘ │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────────────┐ │
│  │  ralphLoop   │ │  startWork   │ │   autoUpdateChecker  │ │
│  └──────────────┘ └──────────────┘ └──────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

---

## 核心职责

Session Hooks 是 oh-my-opencode 插件中最核心的钩子层，负责处理与 OpenCode 会话生命周期直接相关的所有事件。这一层包含 23 个独立的钩子，每个钩子专注于特定的会话管理职责。

这些钩子的核心职责包括：

1. **会话生命周期管理**：监听 `session.created`、`session.deleted`、`session.idle`、`session.error` 等事件，在会话的各个阶段执行相应的逻辑。

2. **模型与参数控制**：通过 `chat.params` 和 `chat.headers` 钩子动态调整模型参数、推理力度（reasoning effort）、温度等设置。

3. **消息处理与路由**：`chat.message` 钩子处理用户输入，支持 first-message 变体检测、会话设置、关键词检测等功能。

4. **错误恢复与降级**：`sessionRecovery`、`modelFallback`、`runtimeFallback` 等钩子提供多层次的错误恢复机制。

5. **状态监控与通知**：`contextWindowMonitor` 监控上下文窗口使用率，`sessionNotification` 提供系统通知功能。

---

## 代码索引 (Code Index)

| 功能点 | 文件路径 | 行号 | 函数/类 | 说明 |
|--------|----------|------|---------|------|
| Session Hooks 创建 | `src/plugin/hooks/create-session-hooks.ts` | 67-298 | `createSessionHooks()` | 创建所有 23 个 Session Hooks |
| Event 处理器 | `src/plugin/event.ts` | 128-607 | `createEventHandler()` | 处理 session.created/deleted/idle/error |
| Chat Message 处理器 | `src/plugin/chat-message.ts` | 87-230 | `createChatMessageHandler()` | 处理 chat.message 事件 |
| Chat Params 处理器 | `src/plugin/chat-params.ts` | 85-182 | `createChatParamsHandler()` | 处理 chat.params 事件 |
| Chat Headers 处理器 | `src/plugin/chat-headers.ts` | 117-141 | `createChatHeadersHandler()` | 处理 chat.headers 事件 |
| 上下文窗口监控 | `src/hooks/context-window-monitor.ts` | 1-200 | `createContextWindowMonitorHook()` | 监控上下文使用率 |
| 会话恢复 | `src/hooks/session-recovery/hook.ts` | 1-150 | `createSessionRecoveryHook()` | 自动恢复会话错误 |
| 模型降级 | `src/hooks/model-fallback/hook.ts` | 1-180 | `createModelFallbackHook()` | 模型故障自动切换 |
| 思考模式 | `src/hooks/think-mode/hook.ts` | 1-120 | `createThinkModeHook()` | 动态思考预算控制 |
| Ralph Loop | `src/hooks/ralph-loop/hook.ts` | 1-200 | `createRalphLoopHook()` | 自引用开发循环 |
| 运行时降级 | `src/hooks/runtime-fallback/hook.ts` | 1-220 | `createRuntimeFallbackHook()` | API 错误自动切换模型 |
| Anthropic 力度 | `src/hooks/anthropic-effort/hook.ts` | 1-80 | `createAnthropicEffortHook()` | 推理力度调整 |

---

## 23 个 Session Hooks

### 1. chat.created

**事件类型**：`session.created`

当新会话被创建时触发。用于初始化会话状态、设置主会话 ID、触发自动更新检查等。

```typescript
// src/plugin/event.ts:335-351
if (event.type === "session.created") {
  const sessionInfo = props?.info as { id?: string; title?: string; parentID?: string } | undefined;

  if (!sessionInfo?.parentID) {
    setMainSession(sessionInfo?.id);  // 设置主会话
  }

  firstMessageVariantGate.markSessionCreated(sessionInfo);

  await managers.tmuxSessionManager.onSessionCreated(event);
}
```

**主要功能**：
- 区分主会话和子会话（通过 parentID 判断）
- 初始化 Tmux 会话管理
- 触发 `autoUpdateChecker` 检查插件更新

---

### 2. chat.deleted

**事件类型**：`session.deleted`

当会话被删除时触发。用于清理会话相关的状态和资源。

```typescript
// src/plugin/event.ts:354-383
if (event.type === "session.deleted") {
  const sessionInfo = props?.info as { id?: string } | undefined;
  if (sessionInfo?.id === getMainSessionID()) {
    setMainSession(undefined);  // 清除主会话
  }

  if (sessionInfo?.id) {
    clearSessionAgent(sessionInfo.id);
    clearPendingModelFallback(sessionInfo.id);
    clearSessionFallbackChain(sessionInfo.id);
    clearSessionModel(sessionInfo.id);
    deleteSessionTools(sessionInfo.id);
    await managers.skillMcpManager.disconnectSession(sessionInfo.id);
    await managers.tmuxSessionManager.onSessionDeleted({ sessionID: sessionInfo.id });
  }
}
```

**主要功能**：
- 清理会话代理状态
- 清除模型降级链
- 断开 MCP 连接
- 清理 Tmux 会话

---

### 3. chat.idle

**事件类型**：`session.idle`

当会话进入空闲状态时触发。这是最重要的会话事件之一，许多钩子在此事件上执行。

```typescript
// src/plugin/event.ts:304-316
if (input.event.type === "session.idle") {
  const sessionID = (input.event.properties as Record<string, unknown> | undefined)?.sessionID as string | undefined;
  if (sessionID) {
    const emittedAt = recentSyntheticIdles.get(sessionID);
    if (emittedAt && Date.now() - emittedAt < DEDUP_WINDOW_MS) {
      recentSyntheticIdles.delete(sessionID);
      return;  // 去重处理
    }
    recentRealIdles.set(sessionID, Date.now());
  }
}
```

**主要功能**：
- 触发 `contextWindowMonitor` 监控上下文窗口
- 触发 `todoContinuationEnforcer` 检查待办事项
- 触发 `sessionNotification` 发送完成通知
- 触发 `ralphLoop` 继续循环

---

### 4. chat.error

**事件类型**：`session.error`

当会话发生错误时触发。用于错误恢复和降级处理。

```typescript
// src/plugin/event.ts:521-605
if (event.type === "session.error") {
  const sessionID = props?.sessionID as string | undefined;
  const error = props?.error;

  // 首先尝试会话恢复
  if (hooks.sessionRecovery?.isRecoverableError(error)) {
    const recovered = await hooks.sessionRecovery.handleSessionRecovery(messageInfo);
    if (recovered) {
      await pluginContext.client.session.prompt({
        path: { id: sessionID },
        body: { parts: [{ type: "text", text: "continue" }] },
      });
    }
  }
  // 然后尝试模型降级
  else if (shouldRetryError(errorInfo) && isModelFallbackEnabled) {
    const setFallback = setPendingModelFallback(sessionID, agentName, currentProvider, currentModel);
    if (setFallback) {
      await autoContinueAfterFallback(sessionID, "session.error");
    }
  }
}
```

**主要功能**：
- 尝试 `sessionRecovery` 自动恢复
- 触发 `modelFallback` 模型降级
- 触发 `runtimeFallback` 运行时降级

---

### 5. chat.message (first-message)

**事件类型**：`chat.message`

处理用户消息，支持 first-message 变体检测。

```typescript
// src/plugin/chat-message.ts:126-129
const isFirstMessage = firstMessageVariantGate.shouldOverride(input.sessionID);
if (isFirstMessage) {
  firstMessageVariantGate.markApplied(input.sessionID);
}
```

**主要功能**：
- 检测会话的第一条消息
- 应用 first-message 变体覆盖
- 初始化会话模型状态

---

### 6. chat.message (session-setup)

**事件类型**：`chat.message`

在消息处理时设置会话状态和模型。

```typescript
// src/plugin/chat-message.ts:122-158
if (input.agent) {
  setSessionAgent(input.sessionID, input.agent);  // 设置会话代理
}

const storedMainSessionModel = getStoredMainSessionModel(input, pluginConfig, isFirstMessage, output);
if (storedMainSessionModel) {
  output.message["model"] = storedMainSessionModel;  // 恢复存储的模型
}

// 设置会话模型状态
if (modelOverride && "providerID" in modelOverride && "modelID" in modelOverride) {
  setSessionModel(input.sessionID, { providerID, modelID });
}
```

**主要功能**：
- 设置会话代理
- 恢复存储的模型配置
- 应用模型覆盖

---

### 7. chat.message (keyword-detection)

**事件类型**：`chat.message`

检测用户输入中的关键词，触发特定模式。

```typescript
// src/plugin/chat-message.ts:162
await hooks.keywordDetector?.["chat.message"]?.(input, output)
```

**支持的模式**：
- `ultrawork` / `ulw`：Ultrawork 模式
- `search`：搜索模式
- `analyze`：分析模式
- `prove-yourself`：验证模式

---

### 8. chat.params

**事件类型**：`chat.params`

动态调整聊天参数，如温度、topP、推理力度等。

```typescript
// src/plugin/chat-params.ts:89-181
export function createChatParamsHandler(args: {
  anthropicEffort: { "chat.params"?: (input: ChatParamsHookInput, output: ChatParamsOutput) => Promise<void> } | null
}): (input: unknown, output: unknown) => Promise<void> {
  return async (input, output): Promise<void> {
    const normalizedInput = buildChatParamsInput(input);
    if (!normalizedInput) return;
    if (!isChatParamsOutput(output)) return;

    // 应用存储的参数
    const storedPromptParams = getSessionPromptParams(normalizedInput.sessionID);
    if (storedPromptParams) {
      if (storedPromptParams.temperature !== undefined) {
        output.temperature = storedPromptParams.temperature;
      }
    }

    // 解析兼容的模型设置
    const compatibility = resolveCompatibleModelSettings({
      providerID: normalizedInput.model.providerID,
      modelID: normalizedInput.model.modelID,
      desired: { variant, reasoningEffort, temperature, topP, maxTokens, thinking },
      capabilities,
    });

    // 应用 Anthropic 推理力度
    await args.anthropicEffort?.["chat.params"]?.(normalizedInput, output);
  };
}
```

**主要功能**：
- 应用存储的会话参数
- 解析模型能力并调整参数
- 支持 Anthropic reasoning effort 设置

---

### 9. chat.headers

**事件类型**：`chat.headers`

设置聊天请求的 HTTP 头，主要用于 Copilot 提供者的 `x-initiator` 头注入。

```typescript
// src/plugin/chat-headers.ts:117-141
export function createChatHeadersHandler(args: { ctx: PluginContext }): (input: unknown, output: unknown) => Promise<void> {
  return async (input, output): Promise<void> {
    const normalizedInput = buildChatHeadersInput(input);
    if (!normalizedInput) return;
    if (!isChatHeadersOutput(output)) return;

    if (!isCopilotProvider(normalizedInput.provider.id)) return;

    // 检查是否为内部消息
    if (!(await isOmoInternalMessage(normalizedInput, ctx.client))) return;

    output.headers["x-initiator"] = "agent";  // 注入 x-initiator 头
  };
}
```

**主要功能**：
- 为 Copilot 提供者注入 `x-initiator: agent` 头
- 识别内部发起的消息（通过 `OMO_INTERNAL_INITIATOR_MARKER`）

---

### 10. event.* (消息更新)

**事件类型**：`message.updated`

当消息更新时触发，用于跟踪模型和代理状态。

```typescript
// src/plugin/event.ts:385-402
if (event.type === "message.updated") {
  const info = props?.info as Record<string, unknown> | undefined;
  const sessionID = info?.sessionID as string | undefined;
  const agent = info?.agent as string | undefined;
  const role = info?.role as string | undefined;

  if (sessionID && role === "user") {
    if (agent) {
      updateSessionAgent(sessionID, agent);  // 更新会话代理
    }
    const providerID = info?.providerID as string | undefined;
    const modelID = info?.modelID as string | undefined;
    if (providerID && modelID) {
      setSessionModel(sessionID, { providerID, modelID });  // 更新会话模型
    }
  }
}
```

**主要功能**：
- 跟踪用户消息的代理和模型
- 为模型降级提供当前状态

---

### 11. contextWindowMonitor

**触发事件**：`session.idle`

监控上下文窗口使用率，在接近限制时触发预警。

```typescript
// src/hooks/context-window-monitor.ts
export function createContextWindowMonitorHook(ctx: PluginContext, modelCacheState: ModelCacheState) {
  return {
    event: async (input: EventInput) => {
      if (input.event.type !== "session.idle") return;

      const sessionID = getEventSessionID(input);
      if (!sessionID) return;

      // 监控上下文窗口使用率
      const usage = await getContextWindowUsage(ctx, sessionID, modelCacheState);
      if (usage.percentage > 80) {
        // 触发预警或自动压缩
      }
    },
  };
}
```

---

### 12. sessionRecovery

**触发事件**：`session.error`

自动检测可恢复的错误并尝试恢复会话。

```typescript
// src/hooks/session-recovery/hook.ts
export function createSessionRecoveryHook(ctx: PluginContext, options: SessionRecoveryOptions) {
  return {
    isRecoverableError: (error: unknown): boolean => {
      const errorType = detectErrorType(error);
      return errorType !== null;
    },

    handleSessionRecovery: async (info: MessageInfo): Promise<boolean> => {
      const errorType = detectErrorType(info.error);
      if (!errorType) return false;

      // 根据错误类型选择恢复策略
      switch (errorType) {
        case "tool_result_missing":
          return await recoverToolResultMissing(ctx, info);
        case "thinking_block_order":
          return await recoverThinkingBlockOrder(ctx, info);
        case "thinking_disabled_violation":
          return await recoverThinkingDisabledViolation(ctx, info);
        default:
          return false;
      }
    },
  };
}
```

---

### 13. modelFallback

**触发事件**：`chat.message`, `session.error`

当模型调用失败时自动降级到备用模型。

```typescript
// src/hooks/model-fallback/hook.ts
export function createModelFallbackHook(args: {
  toast: ToastFn;
  onApplied?: (input: FallbackAppliedInput) => Promise<void>;
}) {
  return {
    "chat.message": async (input: ChatMessageInput, output: ChatMessageHandlerOutput) => {
      // 检查是否有待处理的降级
      const pendingFallback = getPendingModelFallback(input.sessionID);
      if (pendingFallback) {
        output.message["model"] = {
          providerID: pendingFallback.providerID,
          modelID: pendingFallback.modelID,
        };
        clearPendingModelFallback(input.sessionID);
      }
    },
  };
}
```

---

### 14. thinkMode

**触发事件**：`chat.message`, `chat.params`

动态切换模型的思考模式（如 Claude 的 extended thinking）。

```typescript
// src/hooks/think-mode/hook.ts
export function createThinkModeHook() {
  return {
    "chat.message": async (input: ChatMessageInput, output: ChatMessageHandlerOutput) => {
      // 检测是否需要思考模式
      if (shouldEnableThinkMode(input)) {
        output.message["variant"] = "thinking";
      }
    },

    "chat.params": async (input: ChatParamsHookInput, output: ChatParamsOutput) => {
      // 调整思考预算
      if (input.message.variant === "thinking") {
        output.options.thinking = {
          type: "enabled",
          budget_tokens: 16000,
        };
      }
    },
  };
}
```

---

### 15. ralphLoop

**触发事件**：`event`

自引用开发循环，持续迭代直到任务完成。

```typescript
// src/hooks/ralph-loop/hook.ts
export function createRalphLoopHook(ctx: PluginContext, options: RalphLoopOptions) {
  const loopState = new Map<string, LoopState>();

  return {
    event: async (input: EventInput) => {
      if (input.event.type !== "session.idle") return;

      const sessionID = getEventSessionID(input);
      if (!sessionID) return;

      const state = loopState.get(sessionID);
      if (!state || !state.active) return;

      // 检查是否完成
      if (isLoopComplete(input)) {
        stopLoop(sessionID);
        return;
      }

      // 继续循环
      await continueLoop(ctx, sessionID, state);
    },

    startLoop: (sessionID: string, task: string, options: LoopOptions) => {
      loopState.set(sessionID, { active: true, task, iteration: 0, ...options });
    },

    cancelLoop: (sessionID: string) => {
      stopLoop(sessionID);
      loopState.delete(sessionID);
    },
  };
}
```

---

### 16. runtimeFallback

**触发事件**：`event`

运行时模型降级，在 API 错误时自动切换模型。

```typescript
// src/hooks/runtime-fallback/hook.ts
export function createRuntimeFallbackHook(ctx: PluginContext, options: RuntimeFallbackOptions) {
  return {
    event: async (input: EventInput) => {
      if (input.event.type !== "session.error") return;

      const sessionID = getEventSessionID(input);
      if (!sessionID) return;

      const error = extractError(input.event);
      if (!isRuntimeError(error)) return;

      // 获取降级链
      const fallbackChain = getRuntimeFallbackChain(options.pluginConfig);
      const currentModel = getSessionModel(sessionID);

      // 找到下一个可用模型
      const nextModel = findNextFallbackModel(fallbackChain, currentModel);
      if (nextModel) {
        setSessionModel(sessionID, nextModel);
        await notifyModelSwitched(ctx, sessionID, nextModel);
      }
    },

    "chat.message": async (input: ChatMessageInput, output: ChatMessageHandlerOutput) => {
      // 应用降级后的模型
      const fallbackModel = getSessionModel(input.sessionID);
      if (fallbackModel) {
        output.message["model"] = fallbackModel;
      }
    },
  };
}
```

---

### 17. anthropicEffort

**触发事件**：`chat.params`

调整 Anthropic 模型的推理力度（reasoning effort）。

```typescript
// src/hooks/anthropic-effort/hook.ts
export function createAnthropicEffortHook() {
  return {
    "chat.params": async (input: ChatParamsHookInput, output: ChatParamsOutput) => {
      if (input.model.providerID !== "anthropic") return;

      // 根据代理和任务类型调整 effort
      const effort = resolveEffortLevel(input);
      if (effort) {
        output.options.reasoningEffort = effort;
      }
    },
  };
}
```

---

### 18. startWork

**触发事件**：`chat.message`

处理 `/start-work` 命令，启动工作会话。

```typescript
// src/hooks/start-work/start-work-hook.ts
export function createStartWorkHook(ctx: PluginContext) {
  return {
    "chat.message": async (input: ChatMessageInput, output: ChatMessageHandlerOutput) => {
      const parts = output.parts;
      const promptText = extractPromptText(parts);

      if (isStartWorkCommand(promptText)) {
        const workRequest = parseUserRequest(promptText);
        const worktreeInfo = detectWorktree(ctx.directory);

        // 注入工作启动提示
        output.parts = buildStartWorkParts(workRequest, worktreeInfo);
      }
    },
  };
}
```

---

### 19. sessionNotification

**触发事件**：`session.idle`

会话完成时发送系统通知。

```typescript
// src/hooks/session-notification.ts
export function createSessionNotification(ctx: PluginContext) {
  return {
    event: async (input: EventInput) => {
      if (input.event.type !== "session.idle") return;

      const sessionID = getEventSessionID(input);
      if (!sessionID) return;

      // 检查会话是否完成
      if (await isSessionComplete(ctx, sessionID)) {
        await sendNotification(ctx, {
          title: "Session Complete",
          message: `Session ${sessionID} has completed`,
        });
      }
    },
  };
}
```

---

### 20. autoUpdateChecker

**触发事件**：`session.created`

检查插件是否有新版本可用。

```typescript
// src/hooks/auto-update-checker.ts
export function createAutoUpdateCheckerHook(ctx: PluginContext, options: AutoUpdateOptions) {
  let checked = false;

  return {
    event: async (input: EventInput) => {
      if (input.event.type !== "session.created") return;
      if (checked) return;  // 只检查一次

      checked = true;
      const latestVersion = await checkNpmVersion("oh-my-opencode");
      const currentVersion = getCurrentVersion();

      if (latestVersion && latestVersion !== currentVersion) {
        await showUpdateToast(ctx, latestVersion, currentVersion);
      }
    },
  };
}
```

---

### 21. agentUsageReminder

**触发事件**：`chat.message`

提醒用户可用代理的功能。

```typescript
// src/hooks/agent-usage-reminder.ts
export function createAgentUsageReminderHook(ctx: PluginContext) {
  return {
    "chat.message": async (input: ChatMessageInput, output: ChatMessageHandlerOutput) => {
      // 根据用户输入提醒可用代理
      if (shouldRemindAgents(input)) {
        const reminder = buildAgentReminder();
        output.parts.push({ type: "text", text: reminder });
      }
    },
  };
}
```

---

### 22. taskResumeInfo

**触发事件**：`chat.message`

在恢复任务时注入上下文信息。

```typescript
// src/hooks/task-resume-info/hook.ts
export function createTaskResumeInfoHook() {
  return {
    "chat.message": async (input: ChatMessageInput, output: ChatMessageHandlerOutput) => {
      const resumeInfo = getTaskResumeInfo(input.sessionID);
      if (resumeInfo) {
        output.parts.unshift({
          type: "text",
          text: formatResumeContext(resumeInfo),
        });
      }
    },
  };
}
```

---

### 23. preemptiveCompaction

**触发事件**：`session.idle`

在达到上下文限制前主动触发压缩。

```typescript
// src/hooks/preemptive-compaction.ts
export function createPreemptiveCompactionHook(ctx: PluginContext, config: OhMyOpenCodeConfig, modelCacheState: ModelCacheState) {
  return {
    event: async (input: EventInput) => {
      if (input.event.type !== "session.idle") return;

      const sessionID = getEventSessionID(input);
      if (!sessionID) return;

      const usage = await getContextWindowUsage(ctx, sessionID, modelCacheState);
      // 在达到 90% 前主动压缩
      if (usage.percentage > 90 && config.experimental?.preemptive_compaction) {
        await triggerCompaction(ctx, sessionID);
      }
    },
  };
}
```

---

## 会话生命周期

Session Hooks 处理完整的会话生命周期：

```
┌─────────────────────────────────────────────────────────────────┐
│                        会话生命周期流程                          │
└─────────────────────────────────────────────────────────────────┘

  用户创建会话
       │
       ▼
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ session.     │────▶│ autoUpdate   │────▶│ 初始化状态   │
│ created      │     │ Checker      │     │ (Tmux等)     │
└──────────────┘     └──────────────┘     └──────────────┘
       │
       ▼
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ chat.message │────▶│ keyword      │────▶│ modelFallback│
│ (first)      │     │ Detector     │     │ thinkMode    │
└──────────────┘     └──────────────┘     └──────────────┘
       │
       ▼
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ 用户交互中   │◀───▶│ chat.params  │────▶│ anthropic    │
│ (多轮对话)   │     │ chat.headers │     │ Effort       │
└──────────────┘     └──────────────┘     └──────────────┘
       │
       ▼
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ session.idle │────▶│ contextWindow│────▶│ ralphLoop    │
│ (空闲)       │     │ Monitor      │     │ todoEnforcer │
└──────────────┘     └──────────────┘     └──────────────┘
       │
       ▼
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ session.error│────▶│ session      │────▶│ modelFallback│
│ (错误)       │     │ Recovery     │     │ runtimeFallback│
└──────────────┘     └──────────────┘     └──────────────┘
       │
       ▼
┌──────────────┐     ┌──────────────┐
│ session.     │────▶│ 清理所有状态 │
│ deleted      │     │ (模型、代理) │
└──────────────┘     └──────────────┘
```

---

## 流程/架构图

Session Hooks 的执行流程：

```
┌─────────────────────────────────────────────────────────────────┐
│                     OpenCode Event System                        │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Plugin Interface Layer                        │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │
│  │   event.ts  │  │chat-message │  │ chat-params.ts          │  │
│  │  (处理器)   │  │   .ts       │  │ chat-headers.ts         │  │
│  └──────┬──────┘  └──────┬──────┘  └───────────┬─────────────┘  │
└─────────┼────────────────┼─────────────────────┼────────────────┘
          │                │                     │
          ▼                ▼                     ▼
┌─────────────────────────────────────────────────────────────────┐
│                   Session Hooks (23 hooks)                       │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  Core Session: contextWindowMonitor, sessionRecovery,       ││
│  │  sessionNotification, thinkMode, modelFallback,             ││
│  │  anthropicEffort, ralphLoop, startWork, autoUpdateChecker   ││
│  └─────────────────────────────────────────────────────────────┘│
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  Fallback: runtimeFallback, modelFallback,                  ││
│  │  anthropicContextWindowLimitRecovery                        ││
│  └─────────────────────────────────────────────────────────────┘│
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  Utility: agentUsageReminder, taskResumeInfo,               ││
│  │  preemptiveCompaction, interactiveBashSession               ││
│  └─────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Shared State Layer                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │
│  │SessionModel │  │SessionAgent │  │ FallbackChain           │  │
│  │   State     │  │   State     │  │   State                 │  │
│  └─────────────┘  └─────────────┘  └─────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 关键代码片段

### 片段 1：Event Handler 中的 Hook 分发

```typescript
// src/plugin/event.ts:228-258
const dispatchToHooks = async (input: EventInput): Promise<void> => {
  await runEventHookSafely("autoUpdateChecker", hooks.autoUpdateChecker?.event, input);
  await runEventHookSafely("sessionNotification", hooks.sessionNotification, input);
  await runEventHookSafely("todoContinuationEnforcer", hooks.todoContinuationEnforcer?.handler, input);
  await runEventHookSafely("contextWindowMonitor", hooks.contextWindowMonitor?.event, input);
  await runEventHookSafely("preemptiveCompaction", hooks.preemptiveCompaction?.event, input);
  await runEventHookSafely("thinkMode", hooks.thinkMode?.event, input);
  await runEventHookSafely("ralphLoop", hooks.ralphLoop?.event, input);
  await runEventHookSafely("runtimeFallback", hooks.runtimeFallback?.event, input);
  // ... 更多钩子
};
```

### 片段 2：Session Hooks 创建工厂

```typescript
// src/plugin/hooks/create-session-hooks.ts:67-100
export function createSessionHooks(args: {
  ctx: PluginContext
  pluginConfig: OhMyOpenCodeConfig
  modelCacheState: ModelCacheState
  isHookEnabled: (hookName: HookName) => boolean
  safeHookEnabled: boolean
}): SessionHooks {
  const { ctx, pluginConfig, modelCacheState, isHookEnabled, safeHookEnabled } = args;
  const safeHook = <T>(hookName: HookName, factory: () => T): T | null =>
    safeCreateHook(hookName, factory, { enabled: safeHookEnabled });

  const contextWindowMonitor = isHookEnabled("context-window-monitor")
    ? safeHook("context-window-monitor", () =>
        createContextWindowMonitorHook(ctx, modelCacheState))
    : null;

  const sessionRecovery = isHookEnabled("session-recovery")
    ? safeHook("session-recovery", () =>
        createSessionRecoveryHook(ctx, { experimental: pluginConfig.experimental }))
    : null;

  // ... 其他钩子创建
}
```

### 片段 3：Chat Message Handler 中的 Hook 链

```typescript
// src/plugin/chat-message.ts:159-170
await hooks.stopContinuationGuard?.["chat.message"]?.(input);
await hooks.backgroundNotificationHook?.["chat.message"]?.(input, output);
await hooks.runtimeFallback?.["chat.message"]?.(input, output);
await hooks.keywordDetector?.["chat.message"]?.(input, output);
await hooks.thinkMode?.["chat.message"]?.(input, output);
await hooks.claudeCodeHooks?.["chat.message"]?.(input, output);
await hooks.autoSlashCommand?.["chat.message"]?.(input, output);
await hooks.noSisyphusGpt?.["chat.message"]?.(input, output);
await hooks.noHephaestusNonGpt?.["chat.message"]?.(input, output);
if (hooks.startWork && isStartWorkHookOutput(output)) {
  await hooks.startWork["chat.message"]?.(input, output);
}
```

---

## 依赖关系

Session Hooks 依赖以下组件：

| 依赖组件 | 用途 | 文件 |
|----------|------|------|
| PluginContext | 访问 OpenCode 客户端 API | `src/plugin/types.ts` |
| ModelCacheState | 模型能力缓存 | `src/plugin-state.ts` |
| SessionModelState | 会话模型状态管理 | `src/shared/session-model-state.ts` |
| SessionAgentState | 会话代理状态管理 | `src/features/claude-code-session-state.ts` |
| TmuxSessionManager | Tmux 会话管理 | `src/features/tmux-session-manager.ts` |
| SkillMcpManager | MCP 连接管理 | `src/features/skill-mcp-manager.ts` |
| LSPManager | LSP 客户端管理 | `src/tools/lsp-manager.ts` |

---

## 实战示例

### 示例：自定义 Session Hook 实现模型降级

以下示例展示如何创建一个自定义的 Session Hook，在特定错误发生时自动降级模型：

```typescript
// src/hooks/custom-model-fallback/hook.ts
import type { PluginContext } from "../../plugin/types";
import type { EventInput } from "../../plugin/event";

interface FallbackConfig {
  primaryModel: string;
  fallbackModels: string[];
  errorPatterns: RegExp[];
}

export function createCustomModelFallbackHook(
  ctx: PluginContext,
  config: FallbackConfig
) {
  const fallbackState = new Map<string, number>(); // sessionID -> current fallback index

  return {
    event: async (input: EventInput) => {
      if (input.event.type !== "session.error") return;

      const props = input.event.properties as Record<string, unknown> | undefined;
      const sessionID = props?.sessionID as string | undefined;
      const error = props?.error;

      if (!sessionID || !error) return;

      // 检查错误是否匹配降级模式
      const errorMessage = extractErrorMessage(error);
      const shouldFallback = config.errorPatterns.some(pattern =>
        pattern.test(errorMessage)
      );

      if (!shouldFallback) return;

      // 获取当前降级索引
      const currentIndex = fallbackState.get(sessionID) ?? -1;
      const nextIndex = currentIndex + 1;

      if (nextIndex >= config.fallbackModels.length) {
        console.log(`[CustomFallback] No more fallback models for session ${sessionID}`);
        return;
      }

      const fallbackModel = config.fallbackModels[nextIndex];
      fallbackState.set(sessionID, nextIndex);

      // 应用降级模型
      console.log(`[CustomFallback] Switching to ${fallbackModel} for session ${sessionID}`);

      await ctx.client.session.update({
        path: { id: sessionID },
        body: { model: fallbackModel },
        query: { directory: ctx.directory },
      });

      // 发送继续提示
      await ctx.client.session.prompt({
        path: { id: sessionID },
        body: { parts: [{ type: "text", text: "continue" }] },
        query: { directory: ctx.directory },
      });
    },

    // 清理会话状态
    clear: (sessionID: string) => {
      fallbackState.delete(sessionID);
    },
  };
}

// 使用示例
const fallbackHook = createCustomModelFallbackHook(ctx, {
  primaryModel: "claude-opus-4-6",
  fallbackModels: ["claude-opus-4", "gpt-5.4", "kimi-k2.5"],
  errorPatterns: [
    /rate limit exceeded/i,
    /context window exceeded/i,
    /model unavailable/i,
  ],
});
```

### 配置启用自定义 Hook

```jsonc
// .opencode/oh-my-opencode.jsonc
{
  "agents": {
    "sisyphus": {
      "model": "claude-opus-4-6"
    }
  },
  "experimental": {
    "custom_fallback": true
  }
}
```

---

## 交叉引用

- 参见：[Hook 分层架构](./01-Hook 分层.md) - 了解三层 Hook 模型的完整架构
- 参见：[Tool Guard Hooks](./03-Tool Guard Hooks.md) - 了解工具执行前后的保护机制
- 参见：[Continuation Hooks](./continuation-hooks.md) - 了解会话延续相关的钩子
- 参见：[会话持久化](../01-Harness 架构/06-会话持久化.md) - 了解会话状态的持久化机制
- 参见：[模型降级策略](../02-agent-orchestration/model-fallback.md) - 深入了解模型降级的工作原理
- 参见：[Tmux 集成](../04-tools-syste./02-Tmux 集成.md) - 了解 interactiveBashSession 钩子的 Tmux 集成
