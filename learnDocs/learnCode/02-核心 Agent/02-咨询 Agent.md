# 咨询 Agent：Oracle, Librarian, Explore, Metis, Momus

> 所属模块：02-core-agents | 三层模型：Layer 2 | 优先级：P1

---

## 架构位置

```
┌─────────────────────────────────────────────────────────────┐
│                    Layer 3: 主要 Agent                       │
│              (Sisyphus, Hephaestus, Atlas)                   │
└───────────────────────┬─────────────────────────────────────┘
                        │ 委托/咨询
┌───────────────────────▼─────────────────────────────────────┐
│                    Layer 2: 咨询 Agent                       │
│  ┌─────────┐ ┌───────────┐ ┌─────────┐ ┌────────┐ ┌────────┐ │
│  │ Oracle  │ │ Librarian │ │ Explore │ │ Metis  │ │ Momus  │ │
│  │ 架构顾问 │ │ 外部搜索  │ │ 代码搜索 │ │预规划  │ │ 审核员 │ │
│  └─────────┘ └───────────┘ └─────────┘ └────────┘ └────────┘ │
└───────────────────────┬─────────────────────────────────────┘
                        │ 并行调用
┌───────────────────────▼─────────────────────────────────────┐
│                    Layer 1: 工具层                           │
│     (LSP, AST-Grep, Git, WebSearch, Context7, etc.)         │
└─────────────────────────────────────────────────────────────┘
```

咨询 Agent 位于三层架构的中间层，作为"智囊团"为主要 Agent 提供专业领域的深度分析能力。它们都是只读 Agent，不直接修改代码，专注于信息收集、分析和建议。

---

## 核心职责

咨询 Agent 的核心职责是**在主要 Agent 执行工作之前或期间，提供专业领域的深度分析和建议**。它们不直接修改代码，而是通过以下方式支持主要 Agent：

1. **信息收集**：搜索代码库、外部文档、GitHub 仓库等，收集决策所需的信息
2. **模式识别**：发现代码库中的现有模式、最佳实践和潜在问题
3. **架构咨询**：为复杂的技术决策提供高智商的分析和建议
4. **规划支持**：在任务开始前分析需求、识别风险、制定执行策略
5. **质量审核**：审查工作计划，确保其可执行性和完整性

咨询 Agent 的设计理念是**专业化分工**——每个 Agent 专注于特定领域，通过并行调用提高效率，避免单个 Agent 承担过多职责导致的性能下降。

---

## 代码索引 (Code Index)

| 功能点 | 文件路径 | 行号 | 函数/类 | 说明 |
|--------|----------|------|---------|------|
| Oracle Agent 创建 | `src/agents/oracle.ts` | 245-276 | `createOracleAgent` | 高智商只读顾问工厂函数 |
| Oracle Prompt 元数据 | `src/agents/oracle.ts` | 8-38 | `ORACLE_PROMPT_METADATA` | Oracle 触发条件和用途定义 |
| Librarian Agent 创建 | `src/agents/librarian.ts` | 24-319 | `createLibrarianAgent` | 外部参考搜索器工厂函数 |
| Librarian Prompt 元数据 | `src/agents/librarian.ts` | 7-22 | `LIBRARIAN_PROMPT_METADATA` | Librarian 触发条件定义 |
| Explore Agent 创建 | `src/agents/explore.ts` | 27-121 | `createExploreAgent` | 代码库模式搜索器工厂函数 |
| Explore Prompt 元数据 | `src/agents/explore.ts` | 7-25 | `EXPLORE_PROMPT_METADATA` | Explore 触发条件定义 |
| Metis Agent 创建 | `src/agents/metis.ts` | 302-313 | `createMetisAgent` | 预规划分析师工厂函数 |
| Metis 系统 Prompt | `src/agents/metis.ts` | 22-293 | `METIS_SYSTEM_PROMPT` | 预规划分析完整指令 |
| Momus Agent 创建 | `src/agents/momus.ts` | 284-315 | `createMomusAgent` | 规划审核员工厂函数 |
| Momus Prompt 元数据 | `src/agents/momus.ts` | 318-346 | `momusPromptMetadata` | Momus 触发条件定义 |
| Agent 工具限制 | `src/shared/permission-compat.ts` | - | `createAgentToolRestrictions` | 创建 Agent 工具权限限制 |
| 内置 Agent 注册 | `src/agents/builtin-agents.ts` | 33-46 | `agentSources` | 所有内置 Agent 的注册表 |

---

## 五大咨询 Agent

### 1. Oracle - 高智商只读顾问

Oracle 是咨询 Agent 中的"大脑"，专门处理需要深度推理的复杂问题。

**核心能力**：
- 架构决策分析
- 复杂调试问题诊断
- 代码审查和重构建议
- 多系统权衡分析

**模型配置**：
- 默认模型：`gpt-5.4 high`
- 回退链：`gemini-3.1-pro high → claude-opus-4-6 max`
- 温度：0.1（低随机性，高确定性）

**工具限制**：
Oracle 是只读 Agent，禁止以下工具：
```typescript
const restrictions = createAgentToolRestrictions([
  "write",
  "edit", 
  "apply_patch",
  "task",
]);
```

**使用场景**：
- 复杂架构设计决策
- 完成重要工作后自我审查
- 2次以上修复尝试失败后的调试
- 不熟悉的代码模式分析
- 安全/性能问题评估

**触发条件**（来自 `ORACLE_PROMPT_METADATA`）：
```typescript
triggers: [
  { domain: "Architecture decisions", trigger: "Multi-system tradeoffs, unfamiliar patterns" },
  { domain: "Self-review", trigger: "After completing significant implementation" },
  { domain: "Hard debugging", trigger: "After 2+ failed fix attempts" },
]
```

---

### 2. Librarian - 外部参考搜索器

Librarian 专门搜索外部资源，包括官方文档、GitHub 仓库、开源代码示例等。

**核心能力**：
- 查找官方文档和 API 参考
- 搜索 GitHub 上的开源实现示例
- 分析外部依赖的内部实现
- 获取库的版本历史和变更记录

**模型配置**：
- 默认模型：`minimax-m2.7`
- 回退链：`minimax-m2.7-highspeed → claude-haiku-4-5 → gpt-5-nano`
- 成本：CHEAP（低成本）

**请求分类**（Librarian 的核心工作流程）：

| 类型 | 触发条件 | 执行策略 |
|------|----------|----------|
| TYPE A: 概念性 | "如何使用 X？" | 文档发现 → Context7 + WebSearch |
| TYPE B: 实现性 | "X 如何实现 Y？" | GitHub 克隆 + 源码阅读 + blame |
| TYPE C: 上下文 | "为什么这样修改？" | Issues/PRs + git log/blame |
| TYPE D: 综合性 | 复杂/模糊请求 | 文档发现 → 所有工具并行 |

**使用场景**：
- 如何使用某个库？
- 框架特性的最佳实践是什么？
- 外部依赖为何表现出某种行为？
- 查找库的使用示例
- 处理不熟悉的 npm/pip/cargo 包

**代码示例**（来自 `src/agents/librarian.ts` 第 40-317 行）：
```typescript
export function createLibrarianAgent(model: string): AgentConfig {
  const restrictions = createAgentToolRestrictions([
    "write",
    "edit",
    "apply_patch",
    "task",
    "call_omo_agent",
  ])

  return {
    description:
      "Specialized codebase understanding agent for multi-repository analysis...",
    mode: MODE,
    model,
    temperature: 0.1,
    ...restrictions,
    prompt: `# THE LIBRARIAN

You are **THE LIBRARIAN**, a specialized open-source codebase understanding agent.

Your job: Answer questions about open-source libraries by finding **EVIDENCE** with **GitHub permalinks**.
`,
  }
}
```

---

### 3. Explore - 代码库模式搜索器

Explore 是专门的代码库搜索 Agent，用于在本地代码库中查找文件、模式和实现。

**核心能力**：
- 语义搜索（定义、引用）
- 结构模式匹配（函数形状、类结构）
- 文本模式搜索（字符串、注释、日志）
- 文件模式查找（按名称/扩展名）

**模型配置**：
- 默认模型：`grok-code-fast-1`
- 回退链：`minimax-m2.7-highspeed → minimax-m2.7 → claude-haiku-4-5 → gpt-5-nano`
- 成本：FREE（极低成本）

**输出格式要求**（来自 `src/agents/explore.ts` 第 69-86 行）：
```typescript
<results>
<files>
- /absolute/path/to/file1.ts — [why this file is relevant]
- /absolute/path/to/file2.ts — [why this file is relevant]
</files>

<answer>
[Direct answer to their actual need, not just file list]
</answer>

<next_steps>
[What they should do with this information]
</next_steps>
</results>
```

**使用场景**：
- 需要多个搜索角度时
- 不熟悉的模块结构探索
- 跨层模式发现
- "X 在哪里实现？"
- "哪个文件包含 Y？"

**工具策略**：
- **语义搜索**：LSP 工具（`lsp_goto_definition`, `lsp_find_references`）
- **结构模式**：AST-Grep（`ast_grep_search`）
- **文本模式**：Grep（`grep`）
- **文件模式**：Glob（`glob`）
- **历史/演进**：Git 命令

---

### 4. Metis - 预规划分析师

Metis 以希腊智慧女神命名，在规划阶段分析用户请求，识别隐藏意图和潜在风险。

**核心能力**：
- 识别隐藏意图和未明确说明的需求
- 检测可能导致实施失败的模糊性
- 标记潜在的 AI-slop 模式（过度工程、范围蔓延）
- 生成澄清问题
- 为规划 Agent 准备指令

**模型配置**：
- 默认模型：`claude-opus-4-6 max`
- 回退链：`gpt-5.4 high → gemini-3.1-pro high`
- 温度：0.3（略高于其他 Agent，允许一定创造性）
- 思考预算：32000 tokens

**意图分类**（来自 `src/agents/metis.ts` 第 37-44 行）：
```typescript
// Step 1: Identify Intent Type
- **Refactoring**: "refactor", "restructure", "clean up" — SAFETY: 回归预防
- **Build from Scratch**: "create new", "add feature" — DISCOVERY: 先探索模式
- **Mid-sized Task**: 有范围的功能 — GUARDRAILS: 明确交付物
- **Collaborative**: "help me plan" — INTERACTIVE: 对话式澄清
- **Architecture**: "how should we structure" — STRATEGIC: 长期影响
- **Research**: 需要调查 — INVESTIGATION: 退出标准
```

**使用场景**：
- 规划非平凡任务之前
- 用户请求模糊或开放式时
- 防止 AI 过度工程模式
- 需要明确需求边界时

**输出格式**：
```markdown
## Intent Classification
**Type**: [Refactoring | Build | Mid-sized | Collaborative | Architecture | Research]
**Confidence**: [High | Medium | Low]

## Questions for User
1. [最关键的问题]
2. [第二优先级]

## Directives for Prometheus
- MUST: [Required action]
- MUST NOT: [Forbidden action]
```

---

### 5. Momus - 规划审核员

Momus 以希腊讽刺之神命名，以挑剔的眼光审查工作计划，发现每一个遗漏和模糊之处。

**核心能力**：
- 验证引用的文件是否存在
- 确保核心任务有足够的上下文
- 捕获阻塞性问题（会完全停止工作的问题）
- 检查 QA 场景的可执行性

**模型配置**：
- 默认模型：`gpt-5.4 xhigh`
- 回退链：`claude-opus-4-6 max → gemini-3.1-pro high`
- 温度：0.1

**审核原则**（来自 `src/agents/momus.ts` 第 25-199 行）：
```typescript
// 核心原则： blocker-finder, not perfectionist
// 只检查会完全阻塞工作的问题，不追求完美

// 检查的四件事：
1. Reference verification: 引用的文件是否存在？
2. Executability: 开发者能否开始每个任务？
3. Critical blockers: 会完全停止工作的信息缺失
4. QA scenario executability: 每个任务是否有可执行的 QA 场景？
```

**决策框架**：
- **OKAY**（默认）：引用文件存在且相关，任务有足够上下文开始，无矛盾
- **REJECT**（仅针对真正阻塞）：引用文件不存在，任务完全无法开始，计划内部矛盾

**使用场景**：
- Prometheus 创建工作计划后
- 执行复杂待办列表前
- 委托给执行者前验证计划质量
- 需要严格审查 ADHD 导致的遗漏时

**触发条件**：
```typescript
keyTrigger: "Work plan saved to `.sisyphus/plans/*.md` → invoke Momus with the file path"
```

---

## 何时使用哪个咨询 Agent

```
用户请求
    │
    ▼
是否需要外部信息？
    │    ├─ 是 → Librarian (搜索文档/GitHub)
    │    └─ 否
    ▼
是否需要在代码库中查找？
    │    ├─ 是 → Explore (代码库搜索)
    │    └─ 否
    ▼
是否需要架构/复杂决策建议？
    │    ├─ 是 → Oracle (高智商咨询)
    │    └─ 否
    ▼
是否在规划阶段？
    │    ├─ 是 → Metis (预规划分析)
    │    └─ 否
    ▼
是否有工作计划需要审核？
         ├─ 是 → Momus (规划审核)
         └─ 否 → 直接执行
```

| 场景 | 推荐 Agent | 原因 |
|------|-----------|------|
| "如何使用 React Query？" | Librarian | 需要外部文档和示例 |
| "找到所有使用 useEffect 的地方" | Explore | 代码库内搜索 |
| "这个架构设计合理吗？" | Oracle | 需要深度架构分析 |
| "帮我规划这个功能" | Metis → Prometheus | 先分析需求再规划 |
| "审核这个计划" | Momus | 验证计划可执行性 |
| "调试这个复杂 bug" | Oracle | 需要深度推理 |
| "查找类似实现" | Explore | 代码库模式发现 |

---

## 流程/架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                        主要 Agent (Sisyphus)                     │
│                         接收用户请求                              │
└────────────────────────────┬────────────────────────────────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
    ┌─────────────────┐ ┌──────────┐ ┌──────────────┐
    │   需要外部信息？  │ │需要代码搜索│ │ 需要架构建议？ │
    │  → Librarian    │ │→ Explore │ │  → Oracle    │
    └─────────────────┘ └──────────┘ └──────────────┘
              │              │              │
              └──────────────┼──────────────┘
                             ▼
              ┌──────────────────────────────┐
              │      收集到足够信息？          │
              │   否 → 继续并行咨询 Agent      │
              │   是 → 进入规划阶段           │
              └──────────────────────────────┘
                             │
                             ▼
              ┌──────────────────────────────┐
              │      Metis 预规划分析         │
              │   - 意图分类                  │
              │   - 风险识别                  │
              │   - 生成澄清问题              │
              └──────────────────────────────┘
                             │
                             ▼
              ┌──────────────────────────────┐
              │      Prometheus 制定计划      │
              │   - 创建详细工作计划          │
              │   - 定义任务和验收标准        │
              └──────────────────────────────┘
                             │
                             ▼
              ┌──────────────────────────────┐
              │      Momus 计划审核           │
              │   - 验证引用有效性            │
              │   - 检查可执行性              │
              │   - [OKAY] 或 [REJECT]        │
              └──────────────────────────────┘
                             │
                             ▼
              ┌──────────────────────────────┐
              │      Hephaestus 执行          │
              │   - 按计划实施                │
              │   - 完成任务                  │
              └──────────────────────────────┘
```

---

## 关键代码片段

### 片段 1: Oracle Agent 创建与模型适配

```typescript
// src/agents/oracle.ts:245-276
export function createOracleAgent(model: string): AgentConfig {
  const restrictions = createAgentToolRestrictions([
    "write",
    "edit",
    "apply_patch",
    "task",
  ]);

  const base = {
    description:
      "Read-only consultation agent. High-IQ reasoning specialist...",
    mode: MODE,
    model,
    temperature: 0.1,
    ...restrictions,
    prompt: ORACLE_DEFAULT_PROMPT,
  } as AgentConfig;

  // GPT 模型使用 reasoningEffort，其他模型使用 thinking
  if (isGptModel(model)) {
    return {
      ...base,
      prompt: ORACLE_GPT_PROMPT,
      reasoningEffort: "medium",
      textVerbosity: "high",
    } as AgentConfig;
  }

  return {
    ...base,
    thinking: { type: "enabled", budgetTokens: 32000 },
  } as AgentConfig;
}
```

### 片段 2: Explore Agent 的并行执行要求

```typescript
// src/agents/explore.ts:43-119
export function createExploreAgent(model: string): AgentConfig {
  // ...
  prompt: `You are a codebase search specialist. 

## CRITICAL: What You Must Deliver

Every response MUST include:

### 1. Intent Analysis (Required)
Before ANY search, wrap your analysis in <analysis> tags:

<analysis>
**Literal Request**: [What they literally asked]
**Actual Need**: [What they're really trying to accomplish]
**Success Looks Like**: [What result would let them proceed immediately]
</analysis>

### 2. Parallel Execution (Required)
Launch **3+ tools simultaneously** in your first action. 
Never sequential unless output depends on prior result.

### 3. Structured Results (Required)
Always end with this exact format:

<results>
<files>
- /absolute/path/to/file1.ts — [why this file is relevant]
</files>
<answer>
[Direct answer to their actual need]
</answer>
<next_steps>
[What they should do with this information]
</next_steps>
</results>
`,
}
```

### 片段 3: Metis 的意图分类逻辑

```typescript
// src/agents/metis.ts:33-44
## PHASE 0: INTENT CLASSIFICATION (MANDATORY FIRST STEP)

Before ANY analysis, classify the work intent. 
This determines your entire strategy.

### Step 1: Identify Intent Type

- **Refactoring**: "refactor", "restructure", "clean up" — 
  SAFETY: regression prevention, behavior preservation
- **Build from Scratch**: "create new", "add feature", greenfield — 
  DISCOVERY: explore patterns first, informed questions
- **Mid-sized Task**: Scoped feature, specific deliverable — 
  GUARDRAILS: exact deliverables, explicit exclusions
- **Collaborative**: "help me plan", "let's figure out" — 
  INTERACTIVE: incremental clarity through dialogue
- **Architecture**: "how should we structure", system design — 
  STRATEGIC: long-term impact, Oracle recommendation
- **Research**: Investigation needed, goal exists but path unclear — 
  INVESTIGATION: exit criteria, parallel probes
```

---

## 依赖关系

```
consultant-agents
    │
    ├─── 依赖 ───► agent-builder.ts      (Agent 构建工具)
    │
    ├─── 依赖 ───► types.ts              (Agent 类型定义)
    │
    ├─── 依赖 ───► permission-compat.ts  (工具权限限制)
    │
    ├─── 依赖 ───► dynamic-agent-prompt-builder.ts (Prompt 构建)
    │
    └─── 被依赖 ◄── builtin-agents.ts    (Agent 注册)
```

- **Agent Builder**: 所有咨询 Agent 都通过 `buildAgent()` 模式创建
- **Permission Compat**: 使用 `createAgentToolRestrictions()` 限制只读 Agent 的工具访问
- **Dynamic Prompt Builder**: Metis 使用 `buildAntiDuplicationSection()` 构建反重复规则
- **Builtin Agents**: 在 `createBuiltinAgents()` 中注册所有咨询 Agent

---

## 实战示例

### 示例 1: Oracle 架构咨询

**场景**：用户询问是否应该将现有的 REST API 迁移到 GraphQL

**调用方式**：
```typescript
call_omo_agent(
  subagent_type="oracle",
  prompt="""
我正在考虑将我们的 REST API 迁移到 GraphQL。

当前状态：
- 50+ REST endpoints
- 使用 Express.js
- 前端是 React + TanStack Query
- 团队有 5 名开发者

请分析：
1. 迁移的收益和成本
2. 推荐的迁移策略（大爆炸 vs 渐进式）
3. 主要风险和缓解措施
4. 工作量估算
"""
)
```

**Oracle 输出示例**：
```markdown
**Bottom line**: 渐进式迁移推荐。从高频查询开始，保留现有 REST 作为兼容层。

**Action plan**:
1. 选择 3-5 个最复杂的查询作为试点
2. 使用 Apollo Server 的 RESTDataSource 作为过渡
3. 建立 schema 设计规范（参考现有类型定义）
4. 为前端创建 GraphQL 客户端封装
5. 逐步迁移，每个 sprint 1-2 个 endpoint

**Effort estimate**: Medium (1-2 weeks for pilot, 2-3 months full migration)

**Watch out for**:
- N+1 查询问题（使用 DataLoader）
- 缓存策略重新设计
- 团队学习曲线
```

---

### 示例 2: Librarian 文档搜索

**场景**：用户想了解如何使用 TanStack Query 的乐观更新功能

**调用方式**：
```typescript
call_omo_agent(
  subagent_type="librarian",
  prompt="""
如何在 TanStack Query 中实现乐观更新（Optimistic Updates）？

需要：
1. 官方文档的相关部分
2. 真实的代码示例
3. 常见陷阱和最佳实践
4. TypeScript 类型定义
"""
)
```

**Librarian 执行流程**：

1. **Phase 0**: 分类为 TYPE A（概念性问题）
2. **Phase 0.5**: 文档发现
   - 搜索 TanStack Query 官方文档
   - 找到 sitemap.xml 理解文档结构
   - 定位到 optimistic-updates 相关页面
3. **Phase 1**: 并行执行
   - `context7_resolve-library-id("@tanstack/react-query")`
   - `webfetch("https://tanstack.com/query/latest/docs/react/guides/optimistic-updates")`
   - `grep_app_searchGitHub(query: "optimisticUpdate", language: ["TypeScript"])`

**Librarian 输出示例**：
```markdown
**Claim**: 乐观更新通过 `onMutate` 回调实现，在请求发送前更新缓存

**Evidence** ([source](https://github.com/TanStack/query/blob/abc123/packages/react-query/src/types.ts#L45-L52)):
```typescript
onMutate?: (variables: TVariables) => Promise<TContext> | TContext
```

**官方文档** ([Optimistic Updates](https://tanstack.com/query/latest/docs/react/guides/optimistic-updates)):
- 使用 `onMutate` 在 mutation 开始前更新缓存
- 使用 `onError` 在失败时回滚
- 使用 `onSettled` 在完成后重新获取

**常见陷阱**:
1. 忘记处理并发 mutation 的竞态条件
2. 没有正确回滚导致数据不一致
3. 乐观更新过于激进，用户体验差
```

---

## 交叉引用

- 参见：[主要 Agent](./01-主要 Agent.md) - 了解 Sisyphus、Hephaestus、Atlas 等主要 Agent 如何调用咨询 Agent
- 参见：[Agent 编排](./03-Agent 编排.md) - 深入了解 Agent 之间的协作机制和委托模式
- 参见：[Hook 系统](../03-Hook 系统/01-Hook 分层.md) - 了解咨询 Agent 如何与 Hook 系统集成
- 参见：[工具系统](../04-工具系统/01-工具注册.md) - 了解咨询 Agent 可用的工具集
- 参见：[配置系统](../05-config-system/agent-config.md) - 了解如何自定义咨询 Agent 的模型和参数
