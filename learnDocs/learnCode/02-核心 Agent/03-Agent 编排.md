# Agent 编排系统

> 所属模块：02-core-agents | 三层模型：Layer 2 | 优先级：P1

---

## 架构位置

```
┌─────────────────────────────────────────────────────────────────┐
│                        Layer 3: 工具层                           │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │
│  │  LSP Tools  │  │  AST-Grep   │  │  Session Manager        │  │
│  └─────────────┘  └─────────────┘  └─────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              ▲
                              │ 调用
┌─────────────────────────────────────────────────────────────────┐
│                     Layer 2: Agent 编排层                        │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              task() 工具 / Agent 委派链                   │    │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐  │    │
│  │  │  Category   │  │   Model     │  │   Fallback      │  │    │
│  │  │  路由系统    │  │  解析器      │  │   流水线        │  │    │
│  │  └─────────────┘  └─────────────┘  └─────────────────┘  │    │
│  └─────────────────────────────────────────────────────────┘    │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐           │
│  │ Sisyphus │ │Hephaestus│ │  Oracle  │ │Librarian │ ...       │
│  │ (主Agent) │ │(深度工作) │ │(架构咨询) │ │(文档搜索) │           │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘           │
└─────────────────────────────────────────────────────────────────┘
                              ▲
                              │ 委派
┌─────────────────────────────────────────────────────────────────┐
│                     Layer 1: 核心 Agent                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │
│  │   Sisyphus  │  │  Prometheus │  │    Sisyphus-Junior      │  │
│  │  (主协调器)  │  │  (规划器)    │  │   (Category 执行器)      │  │
│  └─────────────┘  └─────────────┘  └─────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

Agent 编排系统位于 Layer 2，是连接主 Agent 与具体工具执行的桥梁。它负责将任务按类别路由到合适的模型，并管理模型降级和 Agent 委派链。

---

## 核心职责

Agent 编排系统是 oh-my-opencode 的多模型调度中枢，承担以下核心职责：

**1. 任务分类与路由**
系统通过 8 个内置 Category 将任务按领域分类，每个 Category 映射到特定的模型配置。例如 `visual-engineering` 任务自动路由到 Gemini Pro，`ultrabrain` 任务路由到 GPT-5.4。这种设计让调用者只需声明任务类型，无需关心具体模型选择。

**2. 模型解析与 Fallback**
当首选模型不可用时，系统按 4 级流水线进行降级：用户覆盖配置 → Category 默认模型 → Provider Fallback 链 → 系统默认模型。这确保了即使部分模型服务中断，任务仍能完成。

**3. Agent 委派链管理**
主 Agent (Sisyphus) 通过 `task()` 工具将子任务委派给专业 Agent。编排系统维护 Agent 注册表、权限控制和执行模式（同步/异步），实现真正的多 Agent 协作。

**4. 动态 Prompt 构建**
根据 Category 自动注入上下文提示，如 `visual-engineering` 会附加设计系统工作流要求，`writing` 会附加反 AI 腔调规则。

---

## 代码索引 (Code Index)

| 功能点 | 文件路径 | 行号 | 函数/类 | 说明 |
|--------|----------|------|---------|------|
| task 工具创建 | `src/tools/delegate-task/tools.ts` | 28-259 | `createDelegateTask()` | 主入口，处理参数验证和执行路由 |
| 8 大 Category 定义 | `src/tools/delegate-task/constants.ts` | 288-297 | `DEFAULT_CATEGORIES` | 内置 Category 到模型的映射配置 |
| Category 描述 | `src/tools/delegate-task/constants.ts` | 310-319 | `CATEGORY_DESCRIPTIONS` | 各 Category 的用途说明 |
| Category Prompt 附加 | `src/tools/delegate-task/constants.ts` | 299-308 | `CATEGORY_PROMPT_APPENDS` | 各 Category 的上下文提示 |
| 模型解析入口 | `src/shared/model-resolver.ts` | 36-42 | `resolveModel()` | 基础模型解析函数 |
| 带 Fallback 的解析 | `src/shared/model-resolver.ts` | 44-63 | `resolveModelWithFallback()` | 完整 4 步解析流水线 |
| Agent 注册表 | `src/agents/builtin-agents.ts` | 33-46 | `agentSources` | 11 个内置 Agent 的工厂映射 |
| Agent 元数据 | `src/agents/builtin-agents.ts` | 52-60 | `agentMetadata` | Agent 提示元数据 |
| 创建内置 Agents | `src/agents/builtin-agents.ts` | 62-200 | `createBuiltinAgents()` | 批量创建 Agent 配置 |
| 参数验证 | `src/tools/delegate-task/tools.ts` | 112-144 | execute 内验证逻辑 | category/subagent_type 互斥检查 |
| Category 执行解析 | `src/tools/delegate-task/tools.ts` | 192-230 | `resolveCategoryExecution()` | Category 任务的模型解析 |
| Subagent 执行解析 | `src/tools/delegate-task/tools.ts` | 232-239 | `resolveSubagentExecution()` | 直接 Agent 调用的解析 |
| 同步任务执行 | `src/tools/delegate-task/tools.ts` | 256 | `executeSyncTask()` | 同步模式执行 |
| 后台任务执行 | `src/tools/delegate-task/tools.ts` | 253 | `executeBackgroundTask()` | 异步模式执行 |

---

## task() 工具实现

`task()` 是 Agent 编排的核心工具，定义在 `src/tools/delegate-task/tools.ts` 第 28-259 行。

### 参数定义

```typescript
// src/tools/delegate-task/tools.ts:99-108
args: {
  load_skills: tool.schema.array(tool.schema.string())
    .describe("Skill names to inject. REQUIRED - pass [] if no skills needed."),
  description: tool.schema.string()
    .describe("Short task description (3-5 words)"),
  prompt: tool.schema.string()
    .describe("Full detailed prompt for the agent"),
  run_in_background: tool.schema.boolean()
    .describe("REQUIRED. true=async (returns task_id), false=sync (waits)"),
  category: tool.schema.string().optional()
    .describe("REQUIRED if subagent_type not provided"),
  subagent_type: tool.schema.string().optional()
    .describe("REQUIRED if category not provided. Valid: explore, librarian, oracle, metis, momus"),
  session_id: tool.schema.string().optional()
    .describe("Existing Task session to continue"),
  command: tool.schema.string().optional()
    .describe("The command that triggered this task"),
}
```

### 执行流程

```
┌─────────────────┐
│   参数验证       │ ← 检查 category/subagent_type 互斥性
│  (112-144行)    │   验证 run_in_background, load_skills
└────────┬────────┘
         ▼
┌─────────────────┐
│   Skill 解析     │ ← resolveSkillContent() 加载技能内容
│  (148-156行)    │
└────────┬────────┘
         ▼
┌─────────────────┐     ┌─────────────────┐
│  有 session_id?  │────→│  继续现有会话    │ ← executeSyncContinuation()
│   (160-165行)    │     │  或后台继续      │   executeBackgroundContinuation()
└────────┬────────┘     └─────────────────┘
         │ 无
         ▼
┌─────────────────┐
│  按 Category    │ ← resolveCategoryExecution()
│  或 Subagent    │   解析模型和 Agent
│  路由 (192-239) │
└────────┬────────┘
         ▼
┌─────────────────┐     ┌─────────────────┐
│ run_in_background│────→│ executeBackgroundTask()
│    === true?     │     │ 返回 task_id
│   (252-254行)    │     │ 异步轮询结果
└────────┬────────┘     └─────────────────┘
         │ false
         ▼
┌─────────────────┐
│ executeSyncTask │ ← 同步执行，等待完成返回结果
│   (256行)       │
└─────────────────┘
```

### 关键验证规则

```typescript
// src/tools/delegate-task/tools.ts:112-120
if (args.category && args.subagent_type) {
  throw new Error(
    `Invalid arguments: 'category' and 'subagent_type' are mutually exclusive. ` +
    `Provide EXACTLY ONE.`
  )
}

// src/tools/delegate-task/tools.ts:121-123
if (args.category) {
  args.subagent_type = SISYPHUS_JUNIOR_AGENT  // "sisyphus-junior"
}
```

当提供 `category` 时，系统自动将执行目标设置为 `sisyphus-junior` Agent，这是 Category 任务的默认执行器。

---

## 8 大内置 Category

Category 系统定义在 `src/tools/delegate-task/constants.ts` 第 288-319 行。

### 1. visual-engineering

```typescript
// src/tools/delegate-task/constants.ts:289
"visual-engineering": { model: "google/gemini-3.1-pro", variant: "high" }
```

**用途**：前端开发、UI/UX 实现、样式调整、动画效果

**特点**：
- 使用 Gemini Pro 模型，擅长视觉理解
- 自动附加设计系统工作流要求（第 8-95 行）
- 强制要求先分析现有设计系统再编码
- 禁止硬编码颜色、间距等魔法数字

**使用场景**：
```typescript
task(
  category="visual-engineering",
  load_skills=["frontend-ui-ux"],
  description="Implement login page",
  prompt="Create a responsive login page with...",
  run_in_background=false
)
```

### 2. ultrabrain

```typescript
// src/tools/delegate-task/constants.ts:290
ultrabrain: { model: "openai/gpt-5.4", variant: "xhigh" }
```

**用途**：复杂逻辑推理、架构决策、算法设计

**特点**：
- 使用 GPT-5.4 xhigh，最强推理能力
- 附加代码风格要求（第 97-117 行）
- 要求先搜索现有代码模式再编写
- 偏向简单方案，强调可维护性

**使用场景**：重构核心模块、设计复杂算法、评估技术方案

### 3. deep

```typescript
// src/tools/delegate-task/constants.ts:291
deep: { model: "openai/gpt-5.3-codex", variant: "medium" }
```

**用途**：目标导向的自主任务执行

**特点**：
- 自主探索模式，无需逐步指令
- 要求静默探索代码库 5-15 分钟
- 适合需要深度理解的问题
- 不询问澄清问题，自主决策

**使用场景**：端到端功能实现、复杂 Bug 修复、跨模块重构

### 4. artistry

```typescript
// src/tools/delegate-task/constants.ts:292
artistry: { model: "google/gemini-3.1-pro", variant: "high" }
```

**用途**：高度创造性任务

**特点**：
- 突破常规边界，探索非传统方案
- 生成多样化、大胆的选择
- 平衡新颖性与一致性
- 适合需要艺术感的任务

**使用场景**：创意编程、独特交互设计、品牌视觉开发

### 5. quick

```typescript
// src/tools/delegate-task/constants.ts:293
quick: { model: "openai/gpt-5.4-mini" }
```

**用途**：简单快速任务

**特点**：
- 使用轻量级模型，响应快速
- 要求提示必须包含 MUST DO/MUST NOT DO/EXPECTED OUTPUT 结构
- 低成本，适合小修改

**使用场景**：拼写修正、简单格式化、单行修改

### 6. unspecified-low

```typescript
// src/tools/delegate-task/constants.ts:294
"unspecified-low": { model: "anthropic/claude-sonnet-4-6" }
```

**用途**：不适合其他 Category 的中等复杂度任务

**特点**：
- 使用 Claude Sonnet 4.6
- 要求先验证是否确实不适合其他 Category
- 范围限于少数文件/模块

**使用场景**：通用开发任务、配置调整、工具脚本

### 7. unspecified-high

```typescript
// src/tools/delegate-task/constants.ts:295
"unspecified-high": { model: "anthropic/claude-opus-4-6", variant: "max" }
```

**用途**：不适合其他 Category 的高复杂度任务

**特点**：
- 使用 Claude Opus 4.6 max
- 跨多个系统/模块的广泛影响
- 需要仔细协调的复杂变更

**使用场景**：大型重构、系统集成、复杂迁移

### 8. writing

```typescript
// src/tools/delegate-task/constants.ts:296
writing: { model: "kimi-for-coding/k2p5" }
```

**用途**：文档编写、技术写作

**特点**：
- 使用 Kimi K2.5，擅长中文和英文写作
- 强制反 AI 腔调规则（第 225-249 行）
- 禁止使用 em dash、AI 填充词
- 要求像人类一样写作

**使用场景**：README 编写、文档撰写、博客文章

---

## 模型解析与 Fallback

模型解析系统定义在 `src/shared/model-resolver.ts`。

### 4 步 Fallback 流水线

```typescript
// src/shared/model-resolver.ts:44-63
export function resolveModelWithFallback(
  input: ExtendedModelResolutionInput,
): ModelResolutionResult | undefined {
  const resolved = resolveModelPipeline({
    intent: { uiSelectedModel, userModel, userFallbackModels, categoryDefaultModel },
    constraints: { availableModels },
    policy: { fallbackChain, systemDefaultModel },
  })
  // ...
}
```

解析优先级（从高到低）：

| 优先级 | 来源 | 说明 |
|--------|------|------|
| 1 | UI 选择模型 | 用户在前端界面选择的模型（仅主 Agent） |
| 2 | 用户覆盖配置 | `agentOverrides` 中指定的模型 |
| 3 | Category 默认 | Category 配置中的 model 字段 |
| 4 | Provider Fallback | `AGENT_MODEL_REQUIREMENTS` 定义的降级链 |
| 5 | 系统默认 | OpenCode 配置中的默认模型 |

### Fallback 链示例

```typescript
// src/shared/model-requirements.ts (概念示例)
const fallbackChains = {
  oracle: ["gpt-5.4 high", "gemini-3.1-pro high", "claude-opus-4-6 max"],
  librarian: ["minimax-m2.7", "minimax-m2.7-highspeed", "claude-haiku-4-5"],
  explore: ["grok-code-fast-1", "minimax-m2.7-highspeed", "gpt-5-nano"],
}
```

当首选模型不可用时，系统按链顺序尝试下一个模型，直到找到可用模型或耗尽选项。

---

## Agent 委派链

Agent 委派通过 `task()` 或 `call_omo_agent()` 工具实现。

### Agent 类型与模式

| Agent | 模式 | 用途 | 禁止工具 |
|-------|------|------|----------|
| Sisyphus | primary | 主协调器，规划与委派 | 无 |
| Hephaestus | primary | 自主深度工作 | 无 |
| Oracle | subagent | 只读架构咨询 | write, edit, task |
| Librarian | subagent | 外部文档搜索 | write, edit, task |
| Explore | subagent | 代码库搜索 | write, edit, task |
| Multimodal-Looker | subagent | PDF/图像分析 | 除 read 外全部 |
| Metis | subagent | 预规划咨询 | 无 |
| Momus | subagent | 计划审查 | write, edit, task |
| Atlas | primary | Todo 列表协调 | task, call_omo_agent |
| Sisyphus-Junior | all | Category 任务执行 | 无 |

### 委派流程

```
┌─────────────┐     task()      ┌─────────────────┐
│  Sisyphus   │ ───────────────→│  编排系统        │
│  (主Agent)   │   category=X    │  (tools.ts)      │
└─────────────┘                 └────────┬────────┘
                                         │
                    ┌────────────────────┼────────────────────┐
                    ▼                    ▼                    ▼
            ┌─────────────┐      ┌─────────────┐      ┌─────────────┐
            │Sisyphus-Junior│     │   Oracle    │      │  Librarian  │
            │ (Category执行)│     │  (架构咨询)  │      │  (文档搜索)  │
            └─────────────┘      └─────────────┘      └─────────────┘
```

---

## 流程/架构图

### Category 任务完整流程

```
┌─────────────────────────────────────────────────────────────────────┐
│                         调用者 (Sisyphus)                            │
│  task(category="visual-engineering", prompt="...", run_in_background=false) │
└─────────────────────────────────┬───────────────────────────────────┘
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      task() 工具 (tools.ts)                          │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │ 1. 参数验证                                                   │  │
│  │    - category 和 subagent_type 互斥检查                        │  │
│  │    - run_in_background 必填                                   │  │
│  │    - load_skills 必填 (可为 [])                                │  │
│  └───────────────────────────────────────────────────────────────┘  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │ 2. Skill 解析                                                 │  │
│  │    - 加载指定技能内容                                          │  │
│  │    - 注入到系统提示                                            │  │
│  └───────────────────────────────────────────────────────────────┘  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │ 3. Category 解析 (resolveCategoryExecution)                    │  │
│  │    - 查找 Category 配置 (DEFAULT_CATEGORIES)                   │  │
│  │    - 获取默认模型 (gemini-3.1-pro high)                        │  │
│  │    - 检查模型可用性                                            │  │
│  │    - 执行 Fallback 链                                          │  │
│  └───────────────────────────────────────────────────────────────┘  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │ 4. Prompt 构建 (buildSystemContent)                            │  │
│  │    - 基础系统提示                                              │  │
│  │    + Skill 内容                                                │  │
│  │    + Category 附加提示 (CATEGORY_PROMPT_APPENDS)               │  │
│  │    = 完整系统提示                                              │  │
│  └───────────────────────────────────────────────────────────────┘  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │ 5. 执行分发                                                   │  │
│  │    ├─ run_in_background=true  → executeBackgroundTask()       │  │
│  │    │                            返回 task_id，异步执行         │  │
│  │    │                                                          │  │
│  │    └─ run_in_background=false → executeSyncTask()             │  │
│  │                                 同步等待，返回完整结果          │  │
│  └───────────────────────────────────────────────────────────────┘  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      Sisyphus-Junior Agent                           │
│              (使用解析后的模型执行具体任务)                           │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 关键代码片段

### 片段 1: Category 配置定义

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

### 片段 2: 模型解析流水线

```typescript
// src/shared/model-resolver.ts:36-42
export function resolveModel(input: ModelResolutionInput): string | undefined {
  return (
    normalizeModel(input.userModel) ??      // 1. 用户覆盖
    normalizeModel(input.inheritedModel) ?? // 2. 继承模型
    input.systemDefault                     // 3. 系统默认
  )
}

// src/shared/model-resolver.ts:44-63
export function resolveModelWithFallback(
  input: ExtendedModelResolutionInput,
): ModelResolutionResult | undefined {
  const resolved = resolveModelPipeline({
    intent: { uiSelectedModel, userModel, userFallbackModels, categoryDefaultModel },
    constraints: { availableModels },
    policy: { fallbackChain, systemDefaultModel },
  })
  // 返回解析结果和来源
  return {
    model: resolved.model,
    source: resolved.provenance,  // "override" | "category-default" | "provider-fallback" | "system-default"
    variant: resolved.variant,
  }
}
```

### 片段 3: Agent 注册表

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
}
```

---

## 依赖关系

```
Agent 编排系统
    │
    ├── 依赖 ──→ 配置系统 (src/config/schema)
    │              - CategoryConfig 定义
    │              - AgentOverrides 配置
    │
    ├── 依赖 ──→ 模型解析 (src/shared/model-*.ts)
    │              - model-resolver.ts
    │              - model-requirements.ts (Fallback 链)
    │              - model-availability.ts (可用性检查)
    │
    ├── 依赖 ──→ Agent 定义 (src/agents/)
    │              - builtin-agents.ts (Agent 注册表)
    │              - sisyphus-junior.ts (Category 执行器)
    │
    ├── 依赖 ──→ Skill 系统 (src/features/opencode-skill-loader/)
    │              - 技能内容解析与注入
    │
    └── 被依赖 ──→ 工具注册表 (src/plugin/tool-registry.ts)
                   - task() 工具注册
```

---

## 实战示例

### 示例 1: 前端任务委派

场景：需要实现一个新的登录页面组件。

```typescript
// 主 Agent (Sisyphus) 决定委派给 visual-engineering Category
task(
  category="visual-engineering",
  load_skills=["frontend-ui-ux"],
  description="Implement login page",
  prompt=`Create a responsive login page with the following requirements:

MUST DO:
1. Analyze the existing design system first (check theme.ts, tailwind config)
2. Read at least 5 existing UI components to understand patterns
3. Use design tokens for all colors, spacing, typography
4. Implement form validation with proper error states
5. Add loading states for submit button

MUST NOT DO:
- Use hardcoded colors like #3b82f6
- Use arbitrary spacing like margin: 13px
- Skip design system analysis

EXPECTED OUTPUT:
- A new LoginPage component in src/pages/Login.tsx
- Uses existing Button, Input components from design system
- Follows project's naming conventions and file structure
- Includes basic unit tests`,
  run_in_background=false
)
```

执行流程：
1. 编排系统识别 `category="visual-engineering"`
2. 解析模型为 `google/gemini-3.1-pro` (variant: high)
3. 附加 `VISUAL_CATEGORY_PROMPT_APPEND` 到系统提示
4. 加载 `frontend-ui-ux` skill 内容
5. 创建 Sisyphus-Junior 会话，同步执行
6. 返回执行结果给主 Agent

### 示例 2: 复杂逻辑委派

场景：需要重构一个核心模块的架构。

```typescript
// 使用 ultrabrain Category 处理复杂逻辑
task(
  category="ultrabrain",
  load_skills=["git-master"],
  description="Refactor auth module architecture",
  prompt=`Refactor the authentication module to support OAuth2 + JWT hybrid flow.

Current issues:
- Auth logic is scattered across 5 files
- No clear separation between OAuth and JWT handling
- Token refresh logic has race conditions

MUST DO:
1. Search existing codebase for auth-related patterns
2. Design a clean architecture with clear interfaces
3. Implement OAuth2 flow in src/auth/oauth/
4. Implement JWT handling in src/auth/jwt/
5. Create unified AuthService facade
6. Add comprehensive error handling
7. Write migration guide for existing code

MUST NOT DO:
- Break existing API contracts
- Skip writing tests
- Use clever tricks that reduce readability

EXPECTED OUTPUT:
- Architecture diagram (ASCII or markdown)
- Refactored code with clear interfaces
- Unit tests for new components
- Migration guide for team`,
  run_in_background=false
)
```

执行流程：
1. 编排系统识别 `category="ultrabrain"`
2. 解析模型为 `openai/gpt-5.4` (variant: xhigh)
3. 附加 `ULTRABRAIN_CATEGORY_PROMPT_APPEND` 到系统提示
4. 强调代码风格一致性和简单性偏好
5. 同步执行，等待详细架构方案

---

## 交叉引用

- 参见：[主要 Agent](./01-主要 Agent.md) - 了解 Sisyphus、Hephaestus 等主 Agent 的详细配置
- 参见：[咨询 Agent](./02-咨询 Agent.md) - 了解 Oracle、Librarian、Metis 等咨询型 Agent
- 参见：[工具注册](../04-工具系统/01-工具注册.md) - 了解 task() 工具的注册机制
- 参见：[模型解析](../03-share./03-模型解析.md) - 深入了解 4 步 Fallback 流水线实现
- 参见：[Category 配置](../05-config/categories.md) - 了解如何自定义 Category 和模型映射
- 参见：[Skill 系统](../06-features/skill-loader.md) - 了解 load_skills 参数的工作原理
