# Transform Hooks：4 个消息转换钩子

> 所属模块：03-hooks-system | 三层模型：Layer 3 | 优先级：P1

---

## 架构位置

```
┌─────────────────────────────────────────────────────────────────┐
│                        OpenCode Plugin                          │
├─────────────────────────────────────────────────────────────────┤
│  Layer 1: Session Hooks (23)                                    │
│  Layer 2: Tool Guard Hooks (12)                                 │
│  Layer 3: Transform Hooks (4)  ← 本文档重点                      │
│  Layer 4: Continuation Hooks (7)                                │
│  Layer 5: Skill Hooks (2)                                       │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│              experimental.chat.messages.transform               │
│  ┌──────────────┬──────────────┬──────────────┬──────────────┐  │
│  │claudeCodeHooks│keywordDetector│contextInjector│thinkingBlock │  │
│  │              │              │MessagesTransform│Validator   │  │
│  └──────────────┴──────────────┴──────────────┴──────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

Transform Hooks 位于 Hook 三层模型的第三层，专门处理消息转换和上下文注入。它们在消息被发送到 LLM API 之前执行，负责修改、增强和验证消息内容。

---

## 核心职责

Transform Hooks 是消息管道的最后一道处理关卡，承担以下核心职责：

**上下文注入**：自动将 AGENTS.md、README.md 等上下文文件注入到用户消息中，确保 Agent 始终掌握项目背景信息。这是通过 `contextInjectorMessagesTransform` 钩子实现的，它在消息数组中查找最后一条用户消息，并在其前面插入一个合成消息部分。

**模式检测与提示注入**：通过 `keywordDetector` 钩子检测用户输入中的特殊关键词（如 `ultrawork`、`search`、`analyze`），并注入对应的系统提示，激活特定的工作模式。

**兼容性处理**：`claudeCodeHooks` 钩子提供 Claude Code 的 settings.json 兼容性支持，将 Claude Code 的权限规则和钩子映射到 OpenCode 的事件系统。

**消息结构验证**：`thinkingBlockValidator` 钩子主动验证消息中的 thinking block 结构，防止 Anthropic API 返回 "Expected thinking/redacted_thinking but found tool_use" 错误。这是预防性的，在 API 调用前就修复问题。

这四个钩子共同确保消息在到达 LLM 之前是完整、合规且上下文丰富的。

---

## 代码索引 (Code Index)

| 功能点 | 文件路径 | 行号 | 函数/类 | 说明 |
|--------|----------|------|---------|------|
| Transform Hooks 组合 | `src/plugin/hooks/create-transform-hooks.ts` | 1-72 | `createTransformHooks` | 创建并组合 4 个 transform hooks |
| 上下文注入器 | `src/features/context-injector/injector.ts` | 82-167 | `createContextInjectorMessagesTransformHook` | 在消息中注入 AGENTS.md 等上下文 |
| Thinking Block 验证 | `src/hooks/thinking-block-validator/hook.ts` | 138-181 | `createThinkingBlockValidatorHook` | 验证并修复 thinking block 结构 |
| 关键词检测 | `src/hooks/keyword-detector/hook.ts` | 16-134 | `createKeywordDetectorHook` | 检测 ultrawork/search/analyze 模式 |
| Claude Code 兼容 | `src/hooks/claude-code-hooks/claude-code-hooks-hook.ts` | 10-22 | `createClaudeCodeHooksHook` | 提供 CC settings.json 兼容层 |
| 上下文收集器 | `src/features/context-injector/collector.ts` | - | `ContextCollector` | 收集并管理待注入的上下文 |
| 消息转换类型 | `src/features/context-injector/injector.ts` | 75-80 | `MessagesTransformHook` | 定义消息转换钩子接口 |
| Core Hooks 组合 | `src/plugin/hooks/create-core-hooks.ts` | 34-39 | `createTransformHooks` 调用 | 将 transform hooks 并入 core hooks |
| 关键词检测器 | `src/hooks/keyword-detector/detector.ts` | - | `detectKeywordsWithType` | 核心关键词检测逻辑 |
| Thinking Block 类型检查 | `src/hooks/thinking-block-validator/hook.ts` | 38-47 | `isSignedThinkingPart` | 检查是否为有效的 thinking part |

---

## 4 个 Transform Hooks

### 1. contextInjectorMessagesTransform (上下文注入)

**功能说明**：

`contextInjectorMessagesTransform` 是 Transform Hooks 中最关键的上下文注入器。它负责将 AGENTS.md、README.md 等项目上下文自动注入到用户消息中，确保 Agent 始终了解项目结构和规范。

**核心逻辑**（`src/features/context-injector/injector.ts` 第 86-166 行）：

```typescript
"experimental.chat.messages.transform": async (_input, output) => {
  const { messages } = output
  // 查找最后一条用户消息
  let lastUserMessageIndex = -1
  for (let i = messages.length - 1; i >= 0; i--) {
    if (messages[i].info.role === "user") {
      lastUserMessageIndex = i
      break
    }
  }
  // ... 省略中间逻辑
  
  // 创建合成消息部分并插入
  const syntheticPart = {
    id: `synthetic_hook_${sessionID}`,
    messageID: lastUserMessage.info.id,
    sessionID: (lastUserMessage.info as { sessionID?: string }).sessionID ?? "",
    type: "text" as const,
    text: pending.merged,
    synthetic: true,  // hidden in UI
  }
  lastUserMessage.parts.splice(textPartIndex, 0, syntheticPart as Part)
}
```

**转换机制**：

1. 遍历消息数组，找到最后一条用户消息
2. 检查 `ContextCollector` 是否有待注入的上下文
3. 创建一个 `synthetic: true` 的合成消息部分
4. 将合成部分插入到用户消息的文本部分之前
5. 合成部分在 UI 中隐藏，但会被 LLM 接收

---

### 2. thinkingBlockValidator (Thinking Block 验证)

**功能说明**：

`thinkingBlockValidator` 是一个预防性钩子，用于解决 Anthropic API 的 thinking block 验证问题。当使用 Claude 3.7 Sonnet 等支持 thinking 模式的模型时，如果消息历史包含 thinking block，后续所有 assistant 消息必须以 thinking block 开头，否则会报错。

**核心逻辑**（`src/hooks/thinking-block-validator/hook.ts` 第 138-181 行）：

```typescript
export function createThinkingBlockValidatorHook(): MessagesTransformHook {
  return {
    "experimental.chat.messages.transform": async (_input, output) => {
      const { messages } = output
      // 检查消息历史中是否有 Anthropic 签名的 thinking blocks
      if (!hasSignedThinkingBlocksInHistory(messages)) {
        return
      }
      // 处理所有 assistant 消息
      for (let i = 0; i < messages.length; i++) {
        const msg = messages[i]
        if (msg.info.role !== "assistant") continue
        // 检查消息是否有内容部分但没有以 thinking 开头
        if (hasContentParts(msg.parts) && !startsWithThinkingBlock(msg.parts)) {
          const previousThinkingPart = findPreviousThinkingPart(messages, i)
          if (previousThinkingPart) {
            prependThinkingBlock(msg, previousThinkingPart)
          }
        }
      }
    },
  }
}
```

**验证逻辑**：

1. 检查消息历史中是否存在 `type: "thinking"` 且带有有效 `signature` 的部分
2. 遍历所有 assistant 角色的消息
3. 如果消息包含内容部分（tool_use、text 等）但没有以 thinking block 开头
4. 从历史消息中查找最近的 thinking part 并前置到当前消息

**关键区别**：

- **预防性**：在 API 调用前修复问题
- **签名复用**：复用原始 thinking part 的签名，避免 API 拒绝
- **与 session-recovery 的区别**：session-recovery 是反应式的（错误发生后修复），而 thinkingBlockValidator 是预防式的

---

### 3. keywordDetector (关键词检测)

**功能说明**：

`keywordDetector` 检测用户输入中的特殊关键词，激活对应的工作模式。支持的关键词包括 `ultrawork`（全代理并行模式）、`search`（搜索模式）、`analyze`（分析模式）等。

**核心逻辑**（`src/hooks/keyword-detector/hook.ts` 第 26-133 行）：

```typescript
"chat.message": async (input, output): Promise<void> => {
  const promptText = extractPromptText(output.parts)
  // 跳过系统指令消息
  if (isSystemDirective(promptText)) {
    return
  }
  // 清理系统提醒内容
  const cleanText = removeSystemReminders(promptText)
  const modelID = input.model?.modelID
  let detectedKeywords = detectKeywordsWithType(cleanText, currentAgent, modelID)
  // 规划器 Agent 不接收 ultrawork 注入
  if (isPlannerAgent(currentAgent)) {
    detectedKeywords = detectedKeywords.filter((k) => k.type !== "ultrawork")
  }
  // 将检测到的关键词消息注入到输出
  const allMessages = detectedKeywords.map((k) => k.message).join("\n\n")
  const originalText = output.parts[textPartIndex].text ?? ""
  output.parts[textPartIndex].text = `${allMessages}\n\n---\n\n${originalText}`
}
```

**检测流程**：

1. 提取用户消息的文本内容
2. 移除系统提醒块（防止自动消息触发模式）
3. 使用正则表达式检测关键词
4. 根据 Agent 类型过滤（如规划器 Agent 不接收 ultrawork）
5. 将模式特定的系统提示注入到用户消息前

---

### 4. claudeCodeHooks (Claude Code 兼容)

**功能说明**：

`claudeCodeHooks` 提供 Claude Code 的完整兼容性支持，包括 settings.json 解析、权限规则映射和钩子事件转换。它让原本为 Claude Code 编写的配置和插件能够在 OpenCode 中无缝运行。

**核心逻辑**（`src/hooks/claude-code-hooks/claude-code-hooks-hook.ts` 第 10-22 行）：

```typescript
export function createClaudeCodeHooksHook(
  ctx: PluginInput,
  config: PluginConfig = {},
  contextCollector?: ContextCollector
) {
  return {
    "experimental.session.compacting": createPreCompactHandler(ctx, config),
    "chat.message": createChatMessageHandler(ctx, config, contextCollector),
    "tool.execute.before": createToolExecuteBeforeHandler(ctx, config),
    "tool.execute.after": createToolExecuteAfterHandler(ctx, config),
    event: createSessionEventHandler(ctx, config),
  }
}
```

**CC → OpenCode 映射**：

| Claude Code Hook | OpenCode Event | 处理函数 |
|------------------|----------------|----------|
| PreToolUse | `tool.execute.before` | `createToolExecuteBeforeHandler` |
| PostToolUse | `tool.execute.after` | `createToolExecuteAfterHandler` |
| Notification | `event` (session.idle) | `createSessionEventHandler` |
| Stop | `event` (session.idle) | `createSessionEventHandler` |

**权限系统**：

解析 Claude Code 的 permissions 格式，将 allow/deny 规则转换为 OpenCode 的工具限制：

```json
{
  "permissions": {
    "allow": ["Edit", "Write"],
    "deny": ["Bash(rm:*)"]
  }
}
```

---

## 消息转换机制

Transform Hooks 通过 `experimental.chat.messages.transform` 事件对消息数组进行转换。这个事件在消息被发送到 LLM API 之前触发，允许钩子修改消息内容。

**消息结构**：

```typescript
interface MessageWithParts {
  info: Message           // 消息元数据（id, role, timestamp 等）
  parts: Part[]           // 消息内容部分数组
}

type MessagesTransformHook = {
  "experimental.chat.messages.transform"?: (
    input: Record<string, never>,  // 空输入
    output: { messages: MessageWithParts[] }  // 可修改的消息数组
  ) => Promise<void>
}
```

**转换流程**：

```
用户输入 → OpenCode 组装消息 → Transform Hooks 处理 → LLM API
                │
                ▼
    ┌───────────────────────┐
    │ 1. claudeCodeHooks    │ ← CC 兼容性处理
    │ 2. keywordDetector    │ ← 模式检测与提示注入
    │ 3. contextInjector    │ ← 上下文注入
    │ 4. thinkingValidator  │ ← 结构验证与修复
    └───────────────────────┘
```

**关键特性**：

- **顺序执行**：Transform Hooks 按照注册顺序依次执行
- **原地修改**：直接修改 `output.messages` 数组
- **合成消息**：使用 `synthetic: true` 标记在 UI 中隐藏的消息部分
- **类型安全**：TypeScript 类型确保消息结构正确

---

## 流程/架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Transform Hooks 执行流程                         │
└─────────────────────────────────────────────────────────────────────┘

  ┌─────────────┐
  │  用户输入    │
  └──────┬──────┘
         │
         ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │                    OpenCode 消息组装                             │
  │  - 收集历史消息                                                  │
  │  - 添加当前用户消息                                              │
  │  - 准备消息数组 (MessageWithParts[])                             │
  └─────────────────────────────┬───────────────────────────────────┘
                                │
                                ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │              experimental.chat.messages.transform               │
  │                                                                 │
  │  ┌─────────────────────────────────────────────────────────┐   │
  │  │ Step 1: claudeCodeHooks                                 │   │
  │  │ - 解析 CC settings.json                                 │   │
  │  │ - 应用权限规则                                          │   │
  │  │ - 映射 CC 钩子到 OpenCode 事件                          │   │
  │  └─────────────────────────────────────────────────────────┘   │
  │                              │                                  │
  │                              ▼                                  │
  │  ┌─────────────────────────────────────────────────────────┐   │
  │  │ Step 2: keywordDetector                                 │   │
  │  │ - 扫描用户输入关键词                                    │   │
  │  │ - 检测 ultrawork/search/analyze                         │   │
  │  │ - 注入模式特定提示                                      │   │
  │  └─────────────────────────────────────────────────────────┘   │
  │                              │                                  │
  │                              ▼                                  │
  │  ┌─────────────────────────────────────────────────────────┐   │
  │  │ Step 3: contextInjectorMessagesTransform                │   │
  │  │ - 检查 ContextCollector                                 │   │
  │  │ - 获取待注入的 AGENTS.md/README.md                      │   │
  │  │ - 创建合成消息部分并插入                                │   │
  │  └─────────────────────────────────────────────────────────┘   │
  │                              │                                  │
  │                              ▼                                  │
  │  ┌─────────────────────────────────────────────────────────┐   │
  │  │ Step 4: thinkingBlockValidator                          │   │
  │  │ - 检查消息历史中的 thinking blocks                      │   │
  │  │ - 验证 assistant 消息结构                               │   │
  │  │ - 修复缺失的 thinking blocks                            │   │
  │  └─────────────────────────────────────────────────────────┘   │
  │                              │                                  │
  └──────────────────────────────┼──────────────────────────────────┘
                                 │
                                 ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │                      发送到 LLM API                              │
  │  - 消息已包含上下文                                              │
  │  - 结构已验证修复                                                │
  │  - 模式提示已注入                                                │
  └─────────────────────────────────────────────────────────────────┘
```

---

## 关键代码片段

### 片段 1：上下文注入核心逻辑

```typescript
// src/features/context-injector/injector.ts:86-166
export function createContextInjectorMessagesTransformHook(
  collector: ContextCollector
): MessagesTransformHook {
  return {
    "experimental.chat.messages.transform": async (_input, output) => {
      const { messages } = output
      if (messages.length === 0) return

      // 查找最后一条用户消息
      let lastUserMessageIndex = -1
      for (let i = messages.length - 1; i >= 0; i--) {
        if (messages[i].info.role === "user") {
          lastUserMessageIndex = i
          break
        }
      }

      if (lastUserMessageIndex === -1) return

      const lastUserMessage = messages[lastUserMessageIndex]
      const sessionID = (lastUserMessage.info as unknown as { sessionID?: string }).sessionID
        ?? getMainSessionID()

      if (!collector.hasPending(sessionID)) return

      const pending = collector.consume(sessionID)
      if (!pending.hasContent) return

      // 创建合成消息部分
      const syntheticPart = {
        id: `synthetic_hook_${sessionID}`,
        messageID: lastUserMessage.info.id,
        sessionID: (lastUserMessage.info as { sessionID?: string }).sessionID ?? "",
        type: "text" as const,
        text: pending.merged,
        synthetic: true,  // 在 UI 中隐藏
      }

      // 插入到用户消息中
      const textPartIndex = lastUserMessage.parts.findIndex(
        (p) => p.type === "text" && (p as { text?: string }).text
      )
      lastUserMessage.parts.splice(textPartIndex, 0, syntheticPart as Part)
    },
  }
}
```

### 片段 2：Thinking Block 验证与修复

```typescript
// src/hooks/thinking-block-validator/hook.ts:138-181
export function createThinkingBlockValidatorHook(): MessagesTransformHook {
  return {
    "experimental.chat.messages.transform": async (_input, output) => {
      const { messages } = output
      if (!messages || messages.length === 0) return

      // 跳过：如果没有 Anthropic 签名的 thinking blocks
      if (!hasSignedThinkingBlocksInHistory(messages)) return

      // 处理所有 assistant 消息
      for (let i = 0; i < messages.length; i++) {
        const msg = messages[i]
        if (msg.info.role !== "assistant") continue

        // 检查：有内容部分但没有以 thinking 开头
        if (hasContentParts(msg.parts) && !startsWithThinkingBlock(msg.parts)) {
          // 查找最近的 thinking part
          const previousThinkingPart = findPreviousThinkingPart(messages, i)

          if (previousThinkingPart) {
            // 前置 thinking block（复用原始签名）
            prependThinkingBlock(msg, previousThinkingPart)
          }
          // 如果没有可用的 thinking part，跳过注入
          // 避免创建无签名的合成 thinking block 导致 API 拒绝
        }
      }
    },
  }
}
```

---

## 依赖关系

```
Transform Hooks
     │
     ├─── 依赖 ───► ContextCollector (上下文收集)
     │               src/features/context-injector/collector.ts
     │
     ├─── 依赖 ───► PluginContext (插件上下文)
     │               src/plugin/types.ts
     │
     ├─── 依赖 ───► Session State (会话状态)
     │               src/features/claude-code-session-state
     │
     ├─── 依赖 ───► Claude Code 配置解析
     │               src/hooks/claude-code-hooks/settings-loader.ts
     │
     └─── 被依赖 ◄─── Core Hooks 组合
                     src/plugin/hooks/create-core-hooks.ts
```

**关键依赖说明**：

- **ContextCollector**：负责收集和管理待注入的上下文内容（AGENTS.md、README.md 等）
- **Session State**：跟踪会话和 Agent 状态，用于关键词检测的过滤逻辑
- **PluginContext**：提供 TUI、客户端等插件级功能
- **CC Settings Loader**：解析 Claude Code 的配置文件

---

## 实战示例

### 示例：Ultrawork 模式激活流程

当用户输入包含 "ultrawork" 或 "ulw" 时，Transform Hooks 的完整处理流程：

**1. 用户输入**：

```
ultrawork 帮我重构这个项目的代码
```

**2. keywordDetector 处理**（`src/hooks/keyword-detector/hook.ts`）：

```typescript
// 检测关键词
const detectedKeywords = detectKeywordsWithType(cleanText, currentAgent, modelID)
// 结果: [{ type: "ultrawork", message: "..." }]

// 注入模式提示
output.parts[textPartIndex].text = `${allMessages}\n\n---\n\n${originalText}`
```

**3. contextInjectorMessagesTransform 处理**（`src/features/context-injector/injector.ts`）：

```typescript
// 检查是否有待注入的上下文
if (collector.hasPending(sessionID)) {
  const pending = collector.consume(sessionID)
  // 将 AGENTS.md 内容注入到用户消息中
  const syntheticPart = {
    type: "text",
    text: pending.merged,  // AGENTS.md 内容
    synthetic: true,
  }
  lastUserMessage.parts.splice(textPartIndex, 0, syntheticPart)
}
```

**4. 最终消息结构**：

```
[系统消息]
[历史消息...]
[用户消息]
  ├─ [合成部分: AGENTS.md 内容]  ← contextInjector 注入
  ├─ [合成部分: Ultrawork 模式提示]  ← keywordDetector 注入
  └─ [原始文本: "ultrawork 帮我重构这个项目的代码"]
```

**5. TUI 反馈**：

```typescript
ctx.client.tui.showToast({
  body: {
    title: "Ultrawork Mode Activated",
    message: "All agents at your disposal.",
    variant: "success",
    duration: 3000,
  },
})
```

这个示例展示了 Transform Hooks 如何协同工作，将简单的用户输入转换为包含完整上下文和模式提示的丰富消息。

---

## 交叉引用

- 参见：[Hook 分层架构](./01-Hook 分层.md) - 了解 Transform Hooks 在三层模型中的位置
- 参见：[Session Hooks](./02-Session Hooks.md) - 了解 Layer 1 的会话生命周期钩子
- 参见：[Tool Guard Hooks](./03-Tool Guard Hooks.md) - 了解 Layer 2 的工具执行保护钩子
- 参见：[上下文组装](../01-Harness 架构/05-上下文组装.md) - 了解 AGENTS.md 和 README.md 的收集机制
- 参见：[Context Injector 模块](../02-features/context-injector.md) - 深入了解上下文注入器的实现细节
- 参见：[Claude Code 兼容性](../04-compatibility/claude-code-hooks.md) - 了解 CC 兼容性层的完整实现
- 参见：[Keyword Detector 详解](../02-features/keyword-detector.md) - 了解关键词检测的完整逻辑

---

## 配置选项

Transform Hooks 可以通过 `disabled_hooks` 配置项禁用：

```jsonc
// .opencode/oh-my-opencode.jsonc
{
  "disabled_hooks": [
    "thinking-block-validator",  // 禁用 thinking block 验证
    "keyword-detector",          // 禁用关键词检测
    "claude-code-hooks"          // 禁用 CC 兼容层
  ]
}
```

注意：`contextInjectorMessagesTransform` 是核心功能，不可禁用。
