# OpenCode 兼容层

> 所属模块：05-功能模块 | 三层模型：Layer 3 | 优先级：P2

---

## 架构位置

```
┌─────────────────────────────────────────────────────────────┐
│                    Layer 3: 功能模块                         │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────┐   │
│  │              OpenCode 兼容层                         │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐ │   │
│  │  │ 插件接口    │  │ 钩子处理器  │  │ 工具兼容    │ │   │
│  │  │ (8 hooks)   │  │ (48 hooks)  │  │ (26 tools)  │ │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘ │   │
│  └─────────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────────┤
│  依赖：Hook 系统 | 配置系统 | 工具系统 | Agent 系统          │
└─────────────────────────────────────────────────────────────┘
```

## 核心职责

OpenCode 兼容层是 oh-my-opencode 插件与 OpenCode 编辑器之间的桥梁，负责将插件功能无缝集成到 OpenCode 的运行时环境中。该层实现了 8 个核心 OpenCode 钩子处理器，管理 48 个内部钩子的生命周期，并确保 26 个自定义工具与 OpenCode 的工具系统完全兼容。

兼容层的核心职责包括：

1. **插件接口适配**：将 oh-my-opencode 的功能包装成 OpenCode 可识别的插件接口
2. **钩子生命周期管理**：协调 48 个内部钩子在 8 个 OpenCode 钩子点的执行
3. **工具注册与过滤**：动态注册工具并根据配置进行权限控制
4. **会话状态同步**：维护会话创建、删除、空闲等状态与 OpenCode 的同步
5. **配置管道执行**：驱动 6 阶段配置加载流程，完成 Agent、工具、MCP 的初始化

## 代码索引 (Code Index)

| 功能点 | 文件路径 | 行号 | 函数/类 | 说明 |
|--------|----------|------|---------|------|
| 插件接口创建 | `src/plugin-interface.ts` | 16-28 | `createPluginInterface` | 组装 8 个 OpenCode 钩子处理器 |
| 工具注册表 | `src/plugin/tool-registry.ts` | 43-50 | `createToolRegistry` | 创建并过滤 26 个工具 |
| 配置处理器 | `src/plugin-handlers/config-handler.ts` | 20-50 | `createConfigHandler` | 6 阶段配置加载主入口 |
| 聊天消息处理 | `src/plugin/chat-message.ts` | 87-96 | `createChatMessageHandler` | 处理 chat.message 钩子 |
| 事件处理 | `src/plugin/event.ts` | 129-136 | `createEventHandler` | 处理 session 生命周期事件 |
| 工具执行前 | `src/plugin/tool-execute-before.ts` | 14-22 | `createToolExecuteBeforeHandler` | 工具调用前置钩子 |
| 核心钩子组装 | `src/plugin/hooks/create-core-hooks.ts` | 9-46 | `createCoreHooks` | 组合 39 个核心钩子 |
| 类型定义 | `src/plugin/types.ts` | 10-16 | `PluginInterface` | OpenCode 插件接口类型 |

## OpenCode 兼容机制

### 1. 插件接口

OpenCode 兼容层实现了标准的 OpenCode 插件接口，包含 8 个核心钩子点：

```typescript
// src/plugin-interface.ts:32-77
return {
  tool: tools,                                    // 工具注册
  "chat.params": async (input, output) => {      // 聊天参数调整
    const handler = createChatParamsHandler({...})
    await handler(input, output)
  },
  "chat.headers": createChatHeadersHandler({ctx}), // HTTP 头注入
  "chat.message": createChatMessageHandler({...}), // 消息处理
  "experimental.chat.messages.transform": createMessagesTransformHandler({...}),
  "experimental.chat.system.transform": createSystemTransformHandler(),
  config: managers.configHandler,                  // 配置钩子
  event: createEventHandler({...}),               // 事件处理
  "tool.execute.before": createToolExecuteBeforeHandler({...}),
  "tool.execute.after": createToolExecuteAfterHandler({...}),
}
```

每个钩子处理器都遵循 OpenCode 的 `(input, output) => Promise<void>` 签名规范，通过修改 `output` 对象来实现功能注入。

### 2. 钩子处理器

兼容层将 48 个内部钩子映射到 8 个 OpenCode 钩子点：

**钩子分层架构：**

| OpenCode 钩子 | 内部钩子数量 | 主要功能 |
|---------------|--------------|----------|
| `chat.message` | 12 | 首消息变体、模型回退、关键词检测、思考模式 |
| `event` | 23 | 会话监控、自动更新、上下文注入、Ralph Loop |
| `tool.execute.before` | 12 | 文件保护、标签截断、规则注入、注释检查 |
| `tool.execute.after` | 4 | 输出截断、元数据存储 |
| `config` | 1 | 6 阶段配置管道 |

```typescript
// src/plugin/hooks/create-core-hooks.ts:18-45
const session = createSessionHooks({...})      // 23 个会话钩子
const tool = createToolGuardHooks({...})       // 12 个工具守卫钩子
const transform = createTransformHooks({...})  // 4 个转换钩子

return {
  ...session,
  ...tool,
  ...transform,
}
```

### 3. 工具兼容性

工具注册表负责将 oh-my-opencode 的 26 个工具适配到 OpenCode 的工具系统：

```typescript
// src/plugin/tool-registry.ts:136-151
const allTools: Record<string, ToolDefinition> = {
  ...builtinTools,                    // 内置工具
  ...createGrepTools(ctx),            // Grep 搜索
  ...createGlobTools(ctx),            // 文件匹配
  ...createAstGrepTools(ctx),         // AST 搜索
  ...createSessionManagerTools(ctx),  // 会话管理
  ...backgroundTools,                 // 后台任务
  call_omo_agent: callOmoAgent,       // Agent 调用
  task: delegateTask,                 // 任务委托
  skill_mcp: skillMcpTool,            // Skill MCP
  skill: skillTool,                   // Skill 执行
  interactive_bash,                   // 交互式终端
  ...taskToolsRecord,                 // 任务系统
  ...hashlineToolsRecord,             // Hashline 编辑
}
```

工具兼容性处理包括：
- **参数模式归一化**：`normalizeToolArgSchemas` 统一参数定义格式
- **工具过滤**：`filterDisabledTools` 根据配置禁用指定工具
- **权限控制**：不同 Agent 拥有不同的工具访问权限

## 兼容层架构

OpenCode 兼容层采用分层设计，确保与 OpenCode 的松耦合：

1. **接口层** (`src/plugin-interface.ts`)：实现 OpenCode 插件契约
2. **处理器层** (`src/plugin/`)：8 个钩子处理器的具体实现
3. **钩子层** (`src/plugin/hooks/`)：48 个内部钩子的定义与组合
4. **配置层** (`src/plugin-handlers/`)：6 阶段配置加载管道

## 流程/架构图

```
┌────────────────────────────────────────────────────────────────┐
│                     OpenCode 编辑器                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐ │
│  │ chat.message│  │    event    │  │ tool.execute.before     │ │
│  └──────┬──────┘  └──────┬──────┘  └───────────┬─────────────┘ │
└─────────┼────────────────┼─────────────────────┼───────────────┘
          │                │                     │
          ▼                ▼                     ▼
┌────────────────────────────────────────────────────────────────┐
│                   OpenCode 兼容层                               │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              createPluginInterface                        │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐  │  │
│  │  │chat-message │  │   event     │  │tool-execute-    │  │  │
│  │  │  handler    │  │  handler    │  │ before handler  │  │  │
│  │  └──────┬──────┘  └──────┬──────┘  └────────┬────────┘  │  │
│  └─────────┼────────────────┼──────────────────┼───────────┘  │
│            │                │                  │              │
│            ▼                ▼                  ▼              │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                    48 个内部钩子                          │  │
│  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────────┐  │  │
│  │  │ Session (23) │ │ ToolGuard(12)│ │ Transform (4)    │  │  │
│  │  │ - modelFallback│ │ - writeGuard │ │ - contextInjector│  │  │
│  │  │ - ralphLoop  │ │ - commentChk │ │ - keywordDetect  │  │  │
│  │  │ - thinkMode  │ │ - rulesInj   │ │ - thinkingBlkVal │  │  │
│  │  └──────────────┘ └──────────────┘ └──────────────────┘  │  │
│  └──────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────┘
          │                │                  │
          ▼                ▼                  ▼
┌────────────────────────────────────────────────────────────────┐
│                     功能模块                                    │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────────────┐  │
│  │ Agent系统 │  │ 工具系统  │  │ MCP系统   │  │ 后台任务系统    │  │
│  └──────────┘  └──────────┘  └──────────┘  └────────────────┘  │
└────────────────────────────────────────────────────────────────┘
```

## 关键代码片段

### 片段 1: 配置处理器主入口

```typescript
// src/plugin-handlers/config-handler.ts:20-50
export function createConfigHandler(deps: ConfigHandlerDeps) {
  const { ctx, pluginConfig, modelCacheState } = deps;

  return async (config: Record<string, unknown>) => {
    const formatterConfig = config.formatter;

    applyProviderConfig({ config, modelCacheState });
    clearFormatterCache()

    const pluginComponents = await loadPluginComponents({ pluginConfig });

    const agentResult = await applyAgentConfig({
      config,
      pluginConfig,
      ctx,
      pluginComponents,
    });

    applyToolConfig({ config, pluginConfig, agentResult });
    await applyMcpConfig({ config, pluginConfig, pluginComponents });
    await applyCommandConfig({ config, pluginConfig, ctx, pluginComponents });

    config.formatter = formatterConfig;
  };
}
```

### 片段 2: 事件处理器中的模型回退逻辑

```typescript
// src/plugin/event.ts:405-457
if (sessionID && role === "assistant" && !isRuntimeFallbackEnabled && isModelFallbackEnabled) {
  try {
    const assistantMessageID = info?.id as string | undefined;
    const assistantError = info?.error;
    if (assistantMessageID && assistantError) {
      const lastHandled = lastHandledModelErrorMessageID.get(sessionID);
      if (lastHandled === assistantMessageID) {
        return;
      }

      const errorName = extractErrorName(assistantError);
      const errorMessage = extractErrorMessage(assistantError);
      const errorInfo = { name: errorName, message: errorMessage };

      if (shouldRetryError(errorInfo)) {
        let agentName = agent ?? getSessionAgent(sessionID);
        if (!agentName && sessionID === getMainSessionID()) {
          if (errorMessage.includes("claude-opus") || errorMessage.includes("opus")) {
            agentName = "sisyphus";
          } else if (errorMessage.includes("gpt-5")) {
            agentName = "hephaestus";
          } else {
            agentName = "sisyphus";
          }
        }

        if (agentName) {
          const currentProvider = resolveFallbackProviderID(
            sessionID,
            info?.providerID as string | undefined,
          );
          const rawModel = (info?.modelID as string | undefined) ?? "claude-opus-4-6";
          const currentModel = normalizeFallbackModelID(rawModel);
          applyUserConfiguredFallbackChain(sessionID, agentName, currentProvider, args.pluginConfig);

          const setFallback = setPendingModelFallback(sessionID, agentName, currentProvider, currentModel);

          if (setFallback && shouldAutoRetrySession(sessionID)) {
            lastHandledModelErrorMessageID.set(sessionID, assistantMessageID);
            await autoContinueAfterFallback(sessionID, "message.updated");
          }
        }
      }
    }
  } catch (err) {
    log("[event] model-fallback error in message.updated:", { sessionID, error: err });
  }
}
```

### 片段 3: 工具执行前的 Agent 解析

```typescript
// src/plugin/tool-execute-before.ts:87-99
if (input.tool === "task") {
  const argsObject = output.args
  const category = typeof argsObject.category === "string" ? argsObject.category : undefined
  const subagentType = typeof argsObject.subagent_type === "string" ? argsObject.subagent_type : undefined
  const sessionId = typeof argsObject.session_id === "string" ? argsObject.session_id : undefined

  if (category) {
    argsObject.subagent_type = "sisyphus-junior"
  } else if (!subagentType && sessionId) {
    const resolvedAgent = await resolveSessionAgent(ctx.client, sessionId)
    argsObject.subagent_type = resolvedAgent ?? "continue"
  }
  // ...
}
```

## 依赖关系

OpenCode 兼容层依赖以下组件：

| 依赖组件 | 用途 |
|----------|------|
| Hook 系统 | 48 个内部钩子的定义与执行 |
| 配置系统 | 多级配置加载与合并 |
| 工具系统 | 26 个工具的工厂函数 |
| Agent 系统 | Agent 配置与模型解析 |
| MCP 系统 | 内置 MCP 服务器的生命周期 |
| 会话状态 | 跨会话的状态管理 |

## 实战示例

### 示例 1: OpenCode 钩子注册

以下展示如何在兼容层中注册一个新的 chat.message 钩子：

```typescript
// 在 src/plugin/hooks/create-session-hooks.ts 中添加新钩子
export function createSessionHooks(args: {
  ctx: PluginContext
  pluginConfig: OhMyOpenCodeConfig
  isHookEnabled: (hookName: HookName) => boolean
}) {
  const { ctx, pluginConfig, isHookEnabled } = args

  return {
    // 现有钩子...
    
    // 新钩子：自定义模型选择器
    customModelSelector: isHookEnabled("customModelSelector") 
      ? {
          "chat.message": async (input: ChatMessageInput, output: ChatMessageHandlerOutput) => {
            // 根据输入内容动态选择模型
            if (shouldUseHeavyModel(output.parts)) {
              output.message["model"] = { 
                providerID: "anthropic", 
                modelID: "claude-opus-4-6" 
              }
            }
          }
        }
      : null,
  }
}
```

然后在 `src/plugin/chat-message.ts` 中调用该钩子：

```typescript
await hooks.customModelSelector?.["chat.message"]?.(input, output)
```

### 示例 2: 工具兼容性适配

为 OpenCode 添加一个自定义工具并确保兼容性：

```typescript
// src/tools/custom-analysis/index.ts
import type { ToolDefinition } from "@opencode-ai/plugin"

export function createCustomAnalysisTool(ctx: PluginContext): ToolDefinition {
  return {
    name: "custom_analysis",
    description: "Perform custom code analysis on the workspace",
    parameters: {
      type: "object",
      properties: {
        pattern: {
          type: "string",
          description: "Analysis pattern to search for",
        },
        scope: {
          type: "string",
          enum: ["file", "directory", "workspace"],
          description: "Scope of the analysis",
        },
      },
      required: ["pattern"],
    },
    async execute(args: { pattern: string; scope?: string }) {
      // 工具实现...
      return { results: [] }
    },
  }
}
```

在工具注册表中添加：

```typescript
// src/plugin/tool-registry.ts
import { createCustomAnalysisTool } from "../tools/custom-analysis"

// 在 allTools 中添加
const allTools: Record<string, ToolDefinition> = {
  ...builtinTools,
  // ... 其他工具
  custom_analysis: createCustomAnalysisTool(ctx),
}
```

## 交叉引用

- 参见：[初始化流程](../00-架构概览/03-初始化流程.md)
- 参见：[Hook 分层](../03-Hook 系统/01-Hook 分层.md)
- 参见：[配置处理器](../07-配置系统/04-插件处理器.md)
- 参见：[工具注册](../04-工具系统/01-工具注册.md)
- 参见：[Agent 配置](../02-Agent 系统/01-Agent 配置.md)
