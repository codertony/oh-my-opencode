# Harness vs Framework 区别

> 所属模块：01-harness-architecture | 三层模型：Layer 3 | 优先级：P0

---

## 架构位置

```
┌─────────────────────────────────────────────────────────────────┐
│                        Layer 3: 执行层                           │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    Harness (执行平台)                    │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐  │   │
│  │  │ Claude Code │  │  Codex CLI  │  │   Gemini CLI    │  │   │
│  │  └─────────────┘  └─────────────┘  └─────────────────┘  │   │
│  │  ┌─────────────────────────────────────────────────────┐│   │
│  │  │        Oh My OpenAgent (Harness 增强层)              ││   │
│  │  │   • 多模型编排  • 48个生命周期钩子  • 26个工具        ││   │
│  │  └─────────────────────────────────────────────────────┘│   │
│  └─────────────────────────────────────────────────────────┘   │
│                              ↕                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                   Framework (构建工具包)                  │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐  │   │
│  │  │  LangGraph  │  │   CrewAI    │  │   AutoGen       │  │   │
│  │  └─────────────┘  └─────────────┘  └─────────────────┘  │   │
│  │  需要开发者自行组装组件，编写编排逻辑                      │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 核心职责

Harness 和 Framework 是 AI 编程工具生态中两个截然不同的概念，但常被混淆。

**Harness（执行平台）** 是开箱即用的完整产品。用户安装后即可直接使用，无需编写代码或配置复杂的组件关系。Claude Code、Codex CLI、Gemini CLI 都属于 Harness。它们提供从对话到代码编辑的完整闭环，包含预设的 Agent 行为、工具集和交互模式。

**Framework（构建工具包）** 是供开发者组装自己 Agent 系统的组件库。LangGraph、CrewAI、AutoGen 都属于 Framework。它们提供节点、边、Agent 基类、记忆模块等基础构件，但需要开发者自行编写编排逻辑、定义工作流、集成工具。

**OMO 的定位** 是 Harness 增强层。它不改变 OpenCode 作为底层 Harness 的本质，而是在其之上通过插件机制注入 48 个生命周期钩子、26 个工具和 11 个专业化 Agent，将单一 Agent 转变为协调工作的开发团队。

---

## 代码索引 (Code Index)

| 功能点 | 文件路径 | 行号 | 函数/类 | 说明 |
|--------|----------|------|---------|------|
| 插件入口 | `src/index.ts` | 20-35 | `OhMyOpenCodePlugin` | 插件初始化入口，加载配置并组装组件 |
| 工具注册 | `src/plugin/tool-registry.ts` | 43-97 | `createToolRegistry` | 注册26个工具到 OpenCode 工具系统 |
| Agent 定义 | `src/agents/sisyphus.ts` | 44-80 | `buildDynamicSisyphusPrompt` | Sisyphus 主 Agent 的动态提示构建 |
| 类别配置 | `src/tools/delegate-task/constants.ts` | 288-297 | `DEFAULT_CATEGORIES` | 8个内置任务类别及其默认模型映射 |
| 委托任务 | `src/tools/delegate-task/constants.ts` | 574-623 | `buildPlanAgentSystemPrepend` | 子 Agent 委托任务的系统提示构建 |
| 钩子创建 | `src/create-hooks.ts` | 1-50 | `createHooks` | 组合48个生命周期钩子 |

---

## Harness vs Framework 对比

### Harness (执行平台)

Harness 是面向终端用户的完整产品：

- **Claude Code**: Anthropic 官方 CLI 工具，提供对话式编程体验，内置文件编辑、命令执行、代码搜索
- **Codex CLI**: OpenAI 的编程 Agent，支持多文件编辑和代码库理解
- **Gemini CLI**: Google 的编程助手，集成 Gemini 模型能力

Harness 的特点：
- 安装即用，零配置
- 预设 Agent 行为和工具集
- 厂商锁定特定模型
- 有限的自定义能力

### Framework (构建工具包)

Framework 是面向开发者的组装工具：

- **LangGraph**: LangChain 的图编排框架，用节点和边定义 Agent 工作流
- **CrewAI**: 多 Agent 协作框架，支持角色定义和任务分配
- **AutoGen**: Microsoft 的 Agent 对话框架，支持多 Agent 对话模式

Framework 的特点：
- 需要编写代码组装
- 高度可定制
- 学习曲线陡峭
- 需要自行处理模型集成、工具调用、错误恢复

### 对比表格

| 维度 | Harness | Framework |
|------|---------|-----------|
| 使用方式 | 安装即用，命令行交互 | 导入库，编写代码组装 |
| 目标用户 | 终端开发者 | Agent 系统开发者 |
| 配置复杂度 | 低（预设配置） | 高（需自行配置） |
| 自定义能力 | 有限（插件/配置） | 极高（代码级控制） |
| 学习曲线 | 平缓 | 陡峭 |
| 典型代表 | Claude Code, Codex CLI, Gemini CLI | LangGraph, CrewAI, AutoGen |
| 模型选择 | 厂商锁定 | 灵活选择 |
| 多 Agent 编排 | 通常单 Agent | 原生支持多 Agent |
| 工具集成 | 内置固定工具集 | 需自行集成 |
| 部署方式 | 本地 CLI | 需部署为服务 |
| 社区生态 | 围绕具体产品 | 围绕框架本身 |
| 升级维护 | 厂商负责 | 开发者负责 |

---

## OMO 的定位：Harness 增强层

### 为什么选择 Harness 增强而非 Framework

OMO 选择基于 OpenCode（Claude Code 的开源分支）进行增强，而非从头基于 Framework 构建，基于以下核心判断：

**1. 生产就绪的基础设施**

OpenCode 作为 Claude Code 的开源分支，已经解决了 Harness 层面的基础问题：文件编辑的稳定性、工具调用的可靠性、会话状态管理、终端交互体验。这些看似简单的功能，实际需要大量工程投入才能做好。OMO 直接继承这些成熟能力，专注于在其上构建编排层。

**2. 兼容性与迁移成本**

OMO 保持与 Claude Code 的完全兼容。用户的 hooks、commands、skills、MCPs、plugins 都可以无缝迁移。如果选择 Framework 路线，用户需要完全重写工作流，迁移成本极高。

**3. 模型中立性**

虽然基于 OpenCode，但 OMO 通过插件机制打破了厂商锁定。通过类别系统（category），任务自动路由到最适合的模型：前端任务用 Gemini，深度推理用 GPT-5.4，快速任务用 GPT-5.4-mini。这是 Framework 的优势，但 OMO 在 Harness 层实现了同样的灵活性。

**4. 工程效率**

Framework 提供了灵活性，但也意味着需要处理大量底层问题：Agent 生命周期管理、并发控制、错误恢复、上下文窗口管理。OMO 利用 OpenCode 的插件架构，通过 48 个钩子点注入增强逻辑，在保持 Harness 稳定性的同时获得 Framework 的编排能力。

**5. 用户心智模型**

开发者已经习惯 Claude Code / OpenCode 的交互模式。OMO 不改变这种心智模型，只是让它更强大。用户仍然使用相同的命令、相同的界面，但背后是多 Agent 协调工作。

---

## 流程/架构图

```
┌────────────────────────────────────────────────────────────────────┐
│                         用户交互层                                  │
│                    (OpenCode CLI / IDE 插件)                       │
└────────────────────────────────────────────────────────────────────┘
                                    ↓
┌────────────────────────────────────────────────────────────────────┐
│                      Oh My OpenAgent 插件层                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌───────────┐ │
│  │   48 Hooks  │  │  26 Tools   │  │  11 Agents  │  │ 8 Categories│ │
│  │  生命周期   │  │  工具集     │  │  Agent 集群 │  │ 任务类别  │ │
│  └─────────────┘  └─────────────┘  └─────────────┘  └───────────┘ │
│                                                                    │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │                    编排核心 (Sisyphus)                       │  │
│  │         意图识别 → 任务分解 → Agent 委托 → 结果验证          │  │
│  └─────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────┘
                                    ↓
┌────────────────────────────────────────────────────────────────────┐
│                      OpenCode Harness 层                           │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌───────────┐ │
│  │  文件编辑   │  │  命令执行   │  │  代码搜索   │  │  LSP 集成 │ │
│  │  (Edit)     │  │  (Bash)     │  │  (Grep)     │  │           │ │
│  └─────────────┘  └─────────────┘  └─────────────┘  └───────────┘ │
└────────────────────────────────────────────────────────────────────┘
                                    ↓
┌────────────────────────────────────────────────────────────────────┐
│                        模型提供商层                                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐           │
│  │ Anthropic│  │  OpenAI  │  │  Google  │  │   其他   │           │
│  │ (Claude) │  │  (GPT)   │  │ (Gemini) │  │          │           │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘           │
└────────────────────────────────────────────────────────────────────┘
```

---

## 关键代码片段

### 1. 插件初始化与组件组装

`src/index.ts` 第 20-95 行展示了 OMO 如何将各个组件组装到 OpenCode 插件架构中：

```typescript
const OhMyOpenCodePlugin: Plugin = async (ctx) => {
  // 加载配置
  const pluginConfig = loadPluginConfig(ctx.directory, ctx)
  
  // 创建管理器（Tmux、Background、Skill MCP）
  const managers = createManagers({
    ctx,
    pluginConfig,
    tmuxConfig,
    modelCacheState,
    backgroundNotificationHookEnabled: isHookEnabled("background-notification"),
  })

  // 创建工具（26个工具）
  const toolsResult = await createTools({
    ctx,
    pluginConfig,
    managers,
  })

  // 创建钩子（48个生命周期钩子）
  const hooks = createHooks({
    ctx,
    pluginConfig,
    modelCacheState,
    backgroundManager: managers.backgroundManager,
    isHookEnabled,
    safeHookEnabled,
    mergedSkills: toolsResult.mergedSkills,
    availableSkills: toolsResult.availableSkills,
  })

  // 组装插件接口
  const pluginInterface = createPluginInterface({
    ctx,
    pluginConfig,
    firstMessageVariantGate,
    managers,
    hooks,
    tools: toolsResult.filteredTools,
  })

  return {
    name: "oh-my-openagent",
    ...pluginInterface,
    // ...
  }
}
```

### 2. 类别系统与模型路由

`src/tools/delegate-task/constants.ts` 第 288-319 行定义了 8 个内置类别及其默认模型映射：

```typescript
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

export const CATEGORY_DESCRIPTIONS: Record<string, string> = {
  "visual-engineering": "Frontend, UI/UX, design, styling, animation",
  ultrabrain: "Use ONLY for genuinely hard, logic-heavy tasks",
  deep: "Goal-oriented autonomous problem-solving",
  artistry: "Complex problem-solving with unconventional, creative approaches",
  quick: "Trivial tasks - single file changes, typo fixes",
  "unspecified-low": "Tasks that don't fit other categories, low effort",
  "unspecified-high": "Tasks that don't fit other categories, high effort",
  writing: "Documentation, prose, technical writing",
}
```

---

## 依赖关系

OMO 对 OpenCode Harness 的依赖关系：

| OMO 组件 | 依赖的 OpenCode 能力 | 说明 |
|----------|---------------------|------|
| 工具系统 | `ToolDefinition` 接口 | 26个工具通过 OpenCode 工具接口注册 |
| 钩子系统 | 8个 OpenCode Hook 处理器 | 48个钩子通过 `chat.message`、`tool.execute.before` 等钩子点注入 |
| Agent 委托 | `subagent` 机制 | `call_omo_agent` 工具调用 OpenCode 的子 Agent 能力 |
| 会话管理 | Session API | `session_list`、`session_read` 等工具基于 OpenCode 会话系统 |
| 文件编辑 | `Edit` 工具增强 | Hashline 编辑工具包装 OpenCode 的原生编辑能力 |
| LSP 集成 | LSP Client | `lsp_diagnostics`、`lsp_rename` 等工具复用 OpenCode 的 LSP 客户端 |

---

## 实战示例

### 场景：实现一个前端组件

**使用 Framework（如 LangGraph）的方式：**

```python
# 需要开发者编写代码组装
from langgraph import Graph
from langchain import Agent

# 1. 定义各个 Agent
frontend_agent = Agent(model="gemini-3.1-pro", tools=[file_edit, bash])
review_agent = Agent(model="claude-opus-4-6", tools=[code_review])

# 2. 定义工作流
graph = Graph()
graph.add_node("design", frontend_agent)
graph.add_node("implement", frontend_agent)
graph.add_node("review", review_agent)
graph.add_edge("design", "implement")
graph.add_edge("implement", "review")

# 3. 执行
result = graph.run("实现一个登录表单组件")
```

**使用 Harness + OMO 的方式：**

```bash
# 用户直接输入
ultrawork 实现一个登录表单组件，使用 React 和 Tailwind
```

OMO 自动处理：
1. Sisyphus 识别意图为前端任务
2. 自动选择 `visual-engineering` 类别 → Gemini 模型
3. 委托给前端 Agent 完成实现
4. 自动验证代码质量
5. 无需编写任何编排代码

---

## 交叉引用

- 参见：[三层架构模型](../00-架构概览/01-三层架构模型.md) - 了解 OMO 在整个架构中的位置
- 参见：[Harness 职责](./01-Harness 职责.md) - 深入了解 Harness 层的具体职责
- 参见：[OpenCode vs OMO 分层关系](../00-架构概览/02-OpenCode 与 OMO.md) - 理解 OpenCode 和 OMO 的分层关系
- 参见：[Agent 编排系统](../02-agent-orchestration/agent-orchestration-overview.md) - 了解 OMO 的多 Agent 编排机制
- 参见：[类别系统详解](../02-agent-orchestration/category-system.md) - 深入了解类别路由系统

---

**文档版本**: 1.0 | **最后更新**: 2026-03-28 | **作者**: OMO 文档团队
