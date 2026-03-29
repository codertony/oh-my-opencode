# 主要 Agent：Sisyphus, Hephaestus, Prometheus

> 所属模块：02-core-agents | 三层模型：Layer 2 | 优先级：P1

---

## 架构位置

```
┌─────────────────────────────────────────────────────────────┐
│                    Layer 3: 用户界面层                        │
│              (OpenCode CLI / Claude Code / IDE)              │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Layer 2: 主要 Agent 层                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │  Sisyphus    │  │ Hephaestus   │  │ Prometheus   │       │
│  │  (编排主 Agent)│  │ (构建/部署专家) │  │ (实现 Agent)  │       │
│  └──────────────┘  └──────────────┘  └──────────────┘       │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Layer 1: 咨询 Agent 层                     │
│       (Oracle, Librarian, Explore, Multimodal-Looker)        │
└─────────────────────────────────────────────────────────────┘
```

主要 Agent 位于架构的第二层，直接面向用户请求，负责任务的规划、编排和执行。它们是 OhMyOpenCode 插件的核心智能体，每个 Agent 都有明确的职责分工和模型路由策略。

---

## 核心职责

主要 Agent 承担着 OhMyOpenCode 的核心工作流：

**Sisyphus** 是主编排 Agent，负责理解用户意图、制定执行计划、协调其他 Agent 并行工作。它采用 Intent Gate 机制分析用户请求的真实意图，通过动态提示词构建系统生成针对不同模型的优化指令。Sisyphus 不直接执行具体任务，而是通过 `task()` 工具将工作委派给专业 Agent。

**Hephaestus** 是自主深度工作 Agent，专注于端到端的复杂实现任务。它接收明确的目标而非详细的步骤指令，能够自主探索代码库、研究最佳实践、执行多文件修改直至任务完成。Hephaestus 的设计哲学是"给予目标，而非食谱"。

**Prometheus** 是战略规划 Agent，在执行任何代码修改前进行深度规划。它采用面试模式与用户交互，识别需求范围、澄清模糊点、构建详细的技术方案。Prometheus 的输出是一份经过验证的执行计划，供 Sisyphus 或 Hephaestus 后续执行。

---

## 代码索引 (Code Index)

| 功能点 | 文件路径 | 行号 | 函数/类 | 说明 |
|--------|----------|------|---------|------|
| Sisyphus Agent 创建 | `src/agents/sisyphus/index.ts` | 10-19 | `buildDefaultSisyphusPrompt` | 构建 Sisyphus 提示词 |
| Sisyphus 默认提示词 | `src/agents/sisyphus/default.ts` | 138-539 | `buildDefaultSisyphusPrompt` | 主提示词构建函数 |
| Sisyphus 配置生成 | `src/agents/builtin-agents/sisyphus-agent.ts` | 11-88 | `maybeCreateSisyphusConfig` | 条件化创建 Sisyphus 配置 |
| Hephaestus Agent 创建 | `src/agents/hephaestus/agent.ts` | 93-127 | `createHephaestusAgent` | Hephaestus Agent 工厂函数 |
| Hephaestus 提示词源 | `src/agents/hephaestus/agent.ts` | 20-30 | `getHephaestusPromptSource` | 根据模型选择提示词源 |
| Hephaestus 动态提示词 | `src/agents/hephaestus/agent.ts` | 48-91 | `buildDynamicHephaestusPrompt` | 动态构建提示词 |
| Prometheus 系统提示词 | `src/agents/prometheus/system-prompt.ts` | 15-20 | `PROMETHEUS_SYSTEM_PROMPT` | 组合式系统提示词 |
| Prometheus 提示词获取 | `src/agents/prometheus/system-prompt.ts` | 55-77 | `getPrometheusPrompt` | 模型适配的提示词获取 |
| 模型解析器 | `src/shared/model-resolver.ts` | 36-42 | `resolveModel` | 基础模型解析 |
| 带回退的模型解析 | `src/shared/model-resolver.ts` | 44-63 | `resolveModelWithFallback` | 带备用链的模型解析 |
| 内置 Agent 注册 | `src/agents/builtin-agents.ts` | 33-46 | `agentSources` | Agent 工厂函数映射表 |
| Agent 元数据 | `src/agents/builtin-agents.ts` | 52-60 | `agentMetadata` | Agent 提示词元数据 |

---

## 三大主要 Agent

### 1. Sisyphus - 编排主 Agent

Sisyphus 是 OhMyOpenCode 的主 Agent，名称源自希腊神话中推石上山的西西弗斯，象征着持续不懈的工作精神。

**核心职责：**
- **意图分析**：通过 Intent Gate 机制识别用户请求的真实意图
- **任务规划**：将复杂任务分解为可执行的子任务
- **Agent 委派**：根据任务类型选择合适的子 Agent
- **并行协调**：同时触发多个背景 Agent 提高效率
- **结果验证**：验证子 Agent 的工作成果

**代码示例 - Sisyphus 提示词构建：**

```typescript
// src/agents/sisyphus/default.ts:138-186
export function buildDefaultSisyphusPrompt(
  model: string,
  availableAgents: AvailableAgent[],
  availableTools: AvailableTool[] = [],
  availableSkills: AvailableSkill[] = [],
  availableCategories: AvailableCategory[] = [],
  useTaskSystem = false,
): string {
  const keyTriggers = buildKeyTriggersSection(availableAgents, availableSkills);
  const toolSelection = buildToolSelectionTable(
    availableAgents,
    availableTools,
    availableSkills,
  );
  // ... 构建各个提示词段落
  
  return `<Role>
You are "Sisyphus" - Powerful AI Agent with orchestration capabilities from OhMyOpenCode.

**Why Sisyphus?**: Humans roll their boulder every day. So do you. 
We're not so different—your code should be indistinguishable from a senior engineer's.

**Identity**: SF Bay Area engineer. Work, delegate, verify, ship. No AI slop.

**Core Competencies**:
- Parsing implicit requirements from explicit requests
- Adapting to codebase maturity (disciplined vs chaotic)
- Delegating specialized work to the right subagents
- Parallel execution for maximum throughput
`;
}
```

**工作模式：**

Sisyphus 采用四阶段工作流：
1. **Phase 0 - Intent Gate**：分析用户意图，确定路由策略
2. **Phase 1 - Codebase Assessment**：评估代码库状态（规范/混乱/遗留）
3. **Phase 2 - Execution**：执行探索、研究或实现
4. **Phase 3 - Completion**：验证结果并完成任务

### 2. Hephaestus - 构建/部署专家

Hephaestus 是自主深度工作 Agent，名称源自希腊神话中的火神与工匠之神，象征着精湛的工艺技术。

**核心职责：**
- **自主探索**：在代码库中自主搜索相关模式和实现
- **端到端执行**：从需求理解到代码实现的完整流程
- **深度研究**：使用 explore/librarian Agent 进行全面的上下文收集
- **目标导向**：接收目标而非步骤，自主决定实现路径

**代码示例 - Hephaestus Agent 创建：**

```typescript
// src/agents/hephaestus/agent.ts:93-127
export function createHephaestusAgent(
  model: string,
  availableAgents?: AvailableAgent[],
  availableToolNames?: string[],
  availableSkills?: AvailableSkill[],
  availableCategories?: AvailableCategory[],
  useTaskSystem = false,
): AgentConfig {
  const tools = availableToolNames ? categorizeTools(availableToolNames) : [];

  const prompt = buildDynamicHephaestusPrompt({
    model,
    availableAgents,
    availableTools: tools,
    availableSkills,
    availableCategories,
    useTaskSystem,
  });

  return {
    description:
      "Autonomous Deep Worker - goal-oriented execution with GPT Codex. " +
      "Explores thoroughly before acting, uses explore/librarian agents " +
      "for comprehensive context, completes tasks end-to-end.",
    mode: MODE,  // "all"
    model,
    maxTokens: 32000,
    prompt,
    color: "#D97706",
    permission: {
      question: "allow",
      call_omo_agent: "deny",
    },
    reasoningEffort: "medium",
  };
}
```

**提示词元数据：**

```typescript
// src/agents/hephaestus/agent.ts:129-154
export const hephaestusPromptMetadata: AgentPromptMetadata = {
  category: "specialist",
  cost: "EXPENSIVE",
  promptAlias: "Hephaestus",
  triggers: [
    {
      domain: "Autonomous deep work",
      trigger: "End-to-end task completion without premature stopping",
    },
  ],
  useWhen: [
    "Task requires deep exploration before implementation",
    "User wants autonomous end-to-end completion",
    "Complex multi-file changes needed",
  ],
  avoidWhen: [
    "Simple single-step tasks",
    "Tasks requiring user confirmation at each step",
  ],
};
```

### 3. Prometheus - 实现 Agent

Prometheus 是战略规划 Agent，名称源自希腊神话中的先知之神，象征着前瞻性的规划能力。

**核心职责：**
- **需求访谈**：通过提问澄清用户需求的范围和细节
- **方案规划**：制定详细的技术实现方案
- **风险评估**：识别潜在的技术风险和挑战
- **计划生成**：输出结构化的执行计划文档

**代码示例 - Prometheus 系统提示词：**

```typescript
// src/agents/prometheus/system-prompt.ts:15-32
export const PROMETHEUS_SYSTEM_PROMPT = `${PROMETHEUS_IDENTITY_CONSTRAINTS}
${PROMETHEUS_INTERVIEW_MODE}
${PROMETHEUS_PLAN_GENERATION}
${PROMETHEUS_HIGH_ACCURACY_MODE}
${PROMETHEUS_PLAN_TEMPLATE}
${PROMETHEUS_BEHAVIORAL_SUMMARY}`;

/**
 * Prometheus planner permission configuration.
 * Allows write/edit for plan files (.md only, enforced by prometheus-md-only hook).
 */
export const PROMETHEUS_PERMISSION = {
  edit: "allow" as const,
  bash: "allow" as const,
  webfetch: "allow" as const,
  question: "allow" as const,
};
```

**模型适配策略：**

```typescript
// src/agents/prometheus/system-prompt.ts:39-77
export function getPrometheusPromptSource(model?: string): PrometheusPromptSource {
  if (model && isGptModel(model)) {
    return "gpt";
  }
  if (model && isGeminiModel(model)) {
    return "gemini";
  }
  return "default";
}

export function getPrometheusPrompt(model?: string, disabledTools?: readonly string[]): string {
  const source = getPrometheusPromptSource(model);
  const isQuestionDisabled = disabledTools?.includes("question") ?? false;

  let prompt: string;
  switch (source) {
    case "gpt":
      prompt = getGptPrometheusPrompt();
      break;
    case "gemini":
      prompt = getGeminiPrometheusPrompt();
      break;
    case "default":
    default:
      prompt = PROMETHEUS_SYSTEM_PROMPT;
  }
  // ...
  return prompt;
}
```

---

## Agent 模型路由

模型路由是 OhMyOpenCode 的核心机制，决定每个 Agent 使用哪个 AI 模型。

**四步解析流程：**

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│  1. Override    │ -> │ 2. Category     │ -> │ 3. Provider     │ -> │ 4. System       │
│  (用户覆盖)      │    │ Default         │    │ Fallback        │    │ Default         │
│                 │    │ (类别默认)       │    │ (供应商回退)     │    │ (系统默认)       │
└─────────────────┘    └─────────────────┘    └─────────────────┘    └─────────────────┘
```

**代码实现：**

```typescript
// src/shared/model-resolver.ts:36-63
export function resolveModel(input: ModelResolutionInput): string | undefined {
  return (
    normalizeModel(input.userModel) ??      // 1. 用户覆盖
    normalizeModel(input.inheritedModel) ?? // 2. 继承模型
    input.systemDefault                     // 3. 系统默认
  );
}

export function resolveModelWithFallback(
  input: ExtendedModelResolutionInput,
): ModelResolutionResult | undefined {
  const resolved = resolveModelPipeline({
    intent: { uiSelectedModel, userModel, userFallbackModels, categoryDefaultModel },
    constraints: { availableModels },
    policy: { fallbackChain, systemDefaultModel },
  });

  return {
    model: resolved.model,
    source: resolved.provenance,  // "override" | "category-default" | "provider-fallback" | "system-default"
    variant: resolved.variant,
  };
}
```

**主要 Agent 的模型配置：**

| Agent | 主模型 | 温度 | 模式 | 回退链 |
|-------|--------|------|------|--------|
| Sisyphus | claude-opus-4-6 max | 0.1 | all | k2p5 → kimi-k2.5 → gpt-5.4 medium → glm-5 |
| Hephaestus | gpt-5.4 medium | 0.1 | all | — |
| Prometheus | claude-opus-4-6 max | 0.1 | — | gpt-5.4 high → gemini-3.1-pro |

---

## 流程/架构图

```
用户请求
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│  Sisyphus (主编排 Agent)                                      │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ Phase 0: Intent Gate (意图分析)                        │  │
│  │ - 识别真实意图 (研究/实现/修复/评估)                    │  │
│  │ - 确定路由策略                                         │  │
│  └───────────────────────────────────────────────────────┘  │
│                          │                                  │
│          ┌───────────────┼───────────────┐                  │
│          ▼               ▼               ▼                  │
│    ┌──────────┐    ┌──────────┐    ┌──────────┐            │
│    │  研究类   │    │  实现类   │    │  规划类   │            │
│    │  请求    │    │  请求    │    │  请求    │            │
│    └────┬─────┘    └────┬─────┘    └────┬─────┘            │
│         │               │               │                   │
│         ▼               ▼               ▼                   │
│    ┌──────────┐    ┌──────────┐    ┌──────────┐            │
│    │ Explore  │    │Hephaestus│    │Prometheus│            │
│    │Librarian │    │ (执行)   │    │ (规划)   │            │
│    │ (搜索)   │    │          │    │          │            │
│    └──────────┘    └──────────┘    └──────────┘            │
│                          │                                  │
│                          ▼                                  │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ Phase 3: Verification (结果验证)                       │  │
│  │ - 验证输出质量                                         │  │
│  │ - 确保符合预期                                         │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
    │
    ▼
  完成/交付
```

---

## 关键代码片段

### 片段 1：Sisyphus 任务管理提示词

```typescript
// src/agents/sisyphus/default.ts:28-136
export function buildTaskManagementSection(useTaskSystem: boolean): string {
  if (useTaskSystem) {
    return `<Task_Management>
## Task Management (CRITICAL)

**DEFAULT BEHAVIOR**: Create tasks BEFORE starting any non-trivial task. 
This is your PRIMARY coordination mechanism.

### When to Create Tasks (MANDATORY)

- Multi-step task (2+ steps) → ALWAYS \`TaskCreate\` first
- Uncertain scope → ALWAYS (tasks clarify thinking)
- User request with multiple items → ALWAYS
- Complex single task → \`TaskCreate\` to break down

### Workflow (NON-NEGOTIABLE)

1. **IMMEDIATELY on receiving request**: \`TaskCreate\` to plan atomic steps.
2. **Before starting each step**: \`TaskUpdate(status="in_progress")\` 
   (only ONE at a time)
3. **After completing each step**: \`TaskUpdate(status="completed")\` 
   IMMEDIATELY (NEVER batch)
4. **If scope changes**: Update tasks before proceeding

**FAILURE TO USE TASKS ON NON-TRIVIAL TASKS = INCOMPLETE WORK.**
</Task_Management>`;
  }
  // ... Todo 系统版本
}
```

### 片段 2：Agent 源注册表

```typescript
// src/agents/builtin-agents.ts:33-46
const agentSources: Record<BuiltinAgentName, AgentSource> = {
  sisyphus: createSisyphusAgent,
  hephaestus: createHephaestusAgent,
  oracle: createOracleAgent,
  librarian: createLibrarianAgent,
  explore: createExploreAgent,
  "multimodal-looker": createMultimodalLookerAgent,
  metis: createMetisAgent,
  momus: createMomusAgent,
  atlas: createAtlasAgent as AgentFactory,
  "sisyphus-junior": createSisyphusJuniorAgentWithOverrides as unknown as AgentFactory,
};
```

---

## 依赖关系

主要 Agent 依赖以下核心组件：

**配置系统：**
- `src/config/schema/` - Agent 配置模式定义
- `src/shared/model-requirements.ts` - 模型需求与回退链

**工具系统：**
- `src/tools/delegate-task/` - Agent 委派任务工具
- `src/tools/background-agent/` - 背景 Agent 管理

**提示词构建：**
- `src/agents/dynamic-agent-prompt-builder.ts` - 动态提示词构建

**共享工具：**
- `src/shared/model-resolver.ts` - 模型解析
- `src/shared/model-normalization.ts` - 模型名称规范化

---

## 实战示例

### 示例：使用 Sisyphus 委派 Hephaestus 实现功能

**场景：** 用户请求实现一个用户认证系统

**Sisyphus 的处理流程：**

```typescript
// 1. 意图分析
// Sisyphus 识别出这是"实现类"请求，需要端到端开发

// 2. 创建任务列表
TaskCreate({
  tasks: [
    { id: "1", content: "分析现有代码库中的认证模式", status: "pending" },
    { id: "2", content: "设计用户认证系统架构", status: "pending" },
    { id: "3", content: "实现用户模型和数据库迁移", status: "pending" },
    { id: "4", content: "实现登录/注册 API", status: "pending" },
    { id: "5", content: "实现 JWT 中间件", status: "pending" },
    { id: "6", content: "编写测试用例", status: "pending" },
  ]
});

// 3. 委派 Hephaestus 执行
TaskUpdate({ taskId: "2", status: "in_progress" });

task({
  subagent_type: "hephaestus",
  run_in_background: false,
  description: "设计并实现用户认证系统",
  prompt: `
1. TASK: 设计并实现完整的用户认证系统
2. EXPECTED OUTCOME: 
   - 用户模型 (User model)
   - 登录/注册 API 端点
   - JWT 认证中间件
   - 密码加密处理
3. REQUIRED TOOLS: read, write, edit, bash
4. MUST DO:
   - 使用 bcrypt 进行密码加密
   - 使用 JWT 进行令牌生成
   - 遵循现有代码风格
   - 编写单元测试
5. MUST NOT DO:
   - 使用明文存储密码
   - 修改不相关的文件
6. CONTEXT: 项目使用 Express + TypeScript，数据库使用 PostgreSQL
  `
});

// 4. 验证结果
// 检查 Hephaestus 的输出是否符合预期
// 运行测试验证功能正确性
lsp_diagnostics({ filePath: "src/auth/" });
```

**Prometheus 的规划介入：**

如果任务特别复杂，Sisyphus 可能先调用 Prometheus：

```typescript
// 在实现前进行规划
task({
  subagent_type: "prometheus",
  run_in_background: false,
  description: "规划用户认证系统",
  prompt: `
我需要实现一个用户认证系统。请：
1. 通过提问澄清需求范围（是否需要刷新令牌？多设备登录？）
2. 评估技术方案（JWT vs Session）
3. 生成详细的实施计划
4. 识别潜在风险

当前技术栈：Express + TypeScript + PostgreSQL
  `
});

// Prometheus 输出规划文档后，Sisyphus 再委派 Hephaestus 执行
```

---

## 交叉引用

- 参见：[咨询 Agent](./02-咨询 Agent.md) - 了解 Explore、Librarian、Oracle 等咨询型 Agent 的详细说明
- 参见：[Agent 编排](./03-Agent 编排.md) - 深入了解 Agent 之间的协作机制和委派协议
- 参见：[模型解析](../07-配置系统/03-模型解析.md) - 详细了解模型路由的四步解析流程
- 参见：[动态提示词构建](../03-prompt-system/dynamic-prompt-builder.md) - 了解如何根据可用 Agent 和工具动态构建提示词
- 参见：[任务系统](../04-tools/delegate-task.md) - 了解 `task()` 工具的完整使用指南
- 参见：[背景 Agent](../04-tool./01-后台 Agent.md) - 了解如何并行运行多个背景 Agent

---

## 总结

Sisyphus、Hephaestus 和 Prometheus 构成了 OhMyOpenCode 的核心 Agent  trio：

- **Sisyphus** 负责"思考"和"协调"，是用户的主要接口
- **Hephaestus** 负责"执行"和"实现"，是深度工作的专家
- **Prometheus** 负责"规划"和"设计"，是战略思考的顾问

三者通过模型路由系统分配到最适合的 AI 模型，通过动态提示词系统获得针对特定模型的优化指令，共同协作完成复杂的软件开发任务。
