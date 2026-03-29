# Agent 推理循环

> 所属模块：02-core-agents | 三层模型：Layer 2 | 优先级：P1

---

## 架构位置

```
┌─────────────────────────────────────────────────────────────┐
│                    Layer 3: 用户交互层                        │
│         (Claude Code / OpenCode / 其他客户端)                │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Layer 2: Agent 编排层                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐     │
│  │ Sisyphus │  │  Oracle  │  │Librarian │  │ Explore  │     │
│  │ 主循环   │  │ 深度分析 │  │外部搜索  │  │代码搜索  │     │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘     │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              ReAct 推理循环引擎                      │   │
│  │   Reason → Act → Observe → Reason → Act → ...      │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Layer 1: 工具执行层                        │
│         (LSP, AST-Grep, Tmux, MCP, Git, etc.)              │
└─────────────────────────────────────────────────────────────┘
```

## 核心职责

Agent 推理循环是 OhMyOpenCode 的核心编排机制，负责将用户的自然语言请求转化为结构化的执行流程。每个 Agent 都遵循特定的推理策略，通过多阶段循环实现任务的理解、分解、执行和验证。

推理循环的核心价值在于：

1. **意图识别**：将用户的表面请求映射到真实的底层需求
2. **任务分解**：将复杂任务拆分为可并行执行的子任务
3. **动态调度**：根据任务类型选择最优的 Agent 组合
4. **结果验证**：确保每个执行步骤都有明确的完成标准
5. **失败恢复**：在遇到困难时能够回退、重试或升级处理

---

## 代码索引 (Code Index)

| 功能点 | 文件路径 | 行号 | 函数/类 | 说明 |
|--------|----------|------|---------|------|
| Sisyphus 主循环 | `src/agents/sisyphus.ts` | 95-384 | `buildDynamicSisyphusPrompt` | 四阶段推理循环实现 |
| Intent Gate | `src/agents/sisyphus/default.ts` | 189-214 | `intent_verbalization` | 意图识别与路由映射 |
| Oracle 分析循环 | `src/agents/oracle.ts` | 44-154 | `ORACLE_DEFAULT_PROMPT` | 深度分析框架 |
| Librarian 搜索循环 | `src/agents/librarian.ts` | 56-278 | `createLibrarianAgent` | 外部文档搜索策略 |
| Explore 代码搜索 | `src/agents/explore.ts` | 43-120 | `createExploreAgent` | 代码库模式识别 |
| Boulder State | `src/features/boulder-state/types.ts` | 8-60 | `BoulderState` | 任务状态持久化 |
| 任务引用解析 | `src/features/boulder-state/top-level-task.ts` | 14-77 | `readCurrentTopLevelTask` | 计划任务解析 |
| Agent 模式定义 | `src/agents/types.ts` | 1-50 | `AgentMode` | Agent 运行模式枚举 |

---

## ReAct 循环实现

ReAct (Reason + Act) 是 OhMyOpenCode 的核心推理模式，每个 Agent 都遵循"思考-行动-观察"的循环：

```
┌─────────────────────────────────────────────────────────────┐
│                      ReAct 循环流程                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   ┌──────────┐    ┌──────────┐    ┌──────────┐             │
│   │ Reason   │───▶│   Act    │───▶│ Observe  │             │
│   │  推理    │    │  执行    │    │  观察    │             │
│   └──────────┘    └──────────┘    └──────────┘             │
│        ▲                                    │               │
│        └────────────────────────────────────┘               │
│                                                             │
│   循环终止条件：                                             │
│   - 任务完成 (所有检查项通过)                                 │
│   - 达到最大迭代次数                                         │
│   - 用户明确终止                                             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Reason 阶段**：分析当前状态，决定下一步行动
**Act 阶段**：执行工具调用或 Agent 委派
**Observe 阶段**：收集执行结果，更新状态

---

## 主要 Agent 的推理策略

### 1. Sisyphus: 编排主循环

Sisyphus 是主编排 Agent，采用四阶段推理循环：

#### Phase 0: Intent Gate (意图门)

```typescript
// src/agents/sisyphus/default.ts:189-214
<intent_verbalization>
### Step 0: Verbalize Intent (BEFORE Classification)

Before classifying the task, identify what the user actually wants 
from you as an orchestrator. Map the surface form to the true intent.

**Intent → Routing Map:**
| Surface Form | True Intent | Your Routing |
|---|---|---|
| "explain X" | Research/understanding | explore/librarian → synthesize |
| "implement X" | Implementation | plan → delegate or execute |
| "look into X" | Investigation | explore → report findings |
| "I'm seeing error X" | Fix needed | diagnose → fix minimally |
</intent_verbalization>
```

#### Phase 1: Codebase Assessment (代码库评估)

- 检查配置文件 (linter, formatter, type config)
- 采样 2-3 个相似文件评估一致性
- 分类项目状态：Disciplined / Transitional / Legacy / Greenfield

#### Phase 2A: Exploration & Research (探索研究)

并行执行策略：
- Fire 2-5 个 explore/librarian Agent 并行搜索
- 使用 `run_in_background=true` 异步收集结果
- 通过 `background_output(task_id="...")` 获取结果

#### Phase 2B: Implementation (实现执行)

- 创建详细的 Todo 列表 (2+ 步骤必须)
- 标记 `in_progress` 后开始执行
- 完成后立即标记 `completed`

#### Phase 2C: Failure Recovery (失败恢复)

```typescript
// src/agents/sisyphus/default.ts:356-364
After 3 Consecutive Failures:
1. STOP all further edits immediately
2. REVERT to last known working state
3. DOCUMENT what was attempted
4. CONSULT Oracle with full failure context
5. If Oracle cannot resolve → ASK USER
```

#### Phase 3: Completion (完成验证)

任务完成检查清单：
- [ ] 所有计划 Todo 项标记完成
- [ ] 修改文件的诊断信息干净
- [ ] 构建通过 (如适用)
- [ ] 用户原始请求完全满足

### 2. Oracle: 深度分析循环

Oracle 是只读咨询 Agent，专注于复杂架构决策和深度调试：

```typescript
// src/agents/oracle.ts:60-69
<decision_framework>
Apply pragmatic minimalism in all recommendations:
- **Bias toward simplicity**: The right solution is typically 
  the least complex one that fulfills the actual requirements.
- **Leverage what exists**: Favor modifications to current code 
  over introducing new components.
- **Prioritize developer experience**: Optimize for readability 
  and maintainability.
</decision_framework>
```

**触发条件**：
- 复杂架构设计
- 2+ 次修复尝试失败后
- 不熟悉的代码模式
- 安全/性能问题

**输出结构** (三层)：
1. **Essential**: Bottom line (2-3 句) + Action plan (≤7 步) + Effort estimate
2. **Expanded**: Why this approach + Watch out for
3. **Edge cases**: Escalation triggers + Alternative sketch

### 3. Librarian: 外部搜索循环

Librarian 专注于外部库和文档的搜索，采用分类驱动的搜索策略：

```typescript
// src/agents/librarian.ts:56-64
## PHASE 0: REQUEST CLASSIFICATION (MANDATORY FIRST STEP)

Classify EVERY request into one of these categories:
- **TYPE A: CONCEPTUAL**: "How do I use X?" → Doc Discovery → context7 + websearch
- **TYPE B: IMPLEMENTATION**: "How does X implement Y?" → gh clone + read + blame
- **TYPE C: CONTEXT**: "Why was this changed?" → gh issues/prs + git log/blame
- **TYPE D: COMPREHENSIVE**: Complex requests → Doc Discovery → ALL tools
```

**文档发现流程** (Phase 0.5)：
1. `websearch("library-name official documentation")` 查找官方文档
2. 版本检查 (如指定版本)
3. `webfetch(docs_url + "/sitemap.xml")` 站点地图发现
4. 针对性调查相关页面

**证据合成要求**：
每个声明必须包含 GitHub 永久链接作为证据：
```markdown
**Claim**: [断言内容]
**Evidence** ([source](https://github.com/.../blob/<sha>/path#L10-L20)):
```typescript
// 实际代码
```
```

### 4. Explore: 代码库搜索循环

Explore 是代码库搜索专家，专注于内部代码的模式识别：

```typescript
// src/agents/explore.ts:56-86
## CRITICAL: What You Must Deliver

Every response MUST include:

### 1. Intent Analysis (Required)
<analysis>
**Literal Request**: [字面请求]
**Actual Need**: [真实需求]
**Success Looks Like**: [成功标准]
</analysis>

### 2. Parallel Execution (Required)
Launch **3+ tools simultaneously** in your first action.

### 3. Structured Results (Required)
<results>
<files>
- /absolute/path/to/file1.ts — [相关性说明]
</files>
<answer>[直接回答]</answer>
<next_steps>[后续步骤]</next_steps>
</results>
```

**工具策略**：
- **语义搜索** (定义、引用): LSP 工具
- **结构模式** (函数形状、类结构): `ast_grep_search`
- **文本模式** (字符串、注释): `grep`
- **文件模式** (按名称/扩展名查找): `glob`

---

## 流程/架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Sisyphus 主循环                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   ┌──────────────┐                                                  │
│   │  Phase 0     │  Intent Gate (意图门)                            │
│   │  意图识别    │  - 表面形式 → 真实意图映射                        │
│   └──────┬───────┘  - 路由决策声明                                   │
│          │                                                          │
│          ▼                                                          │
│   ┌──────────────┐                                                  │
│   │  Step 1      │  Classify Request Type                           │
│   │  请求分类    │  - Trivial / Explicit / Exploratory              │
│   └──────┬───────┘  - Open-ended / Ambiguous                        │
│          │                                                          │
│          ▼                                                          │
│   ┌──────────────┐                                                  │
│   │  Step 2      │  Check for Ambiguity                             │
│   │  歧义检查    │  - 多解释 → 必须询问                             │
│   └──────┬───────┘  - 关键信息缺失 → 必须询问                       │
│          │                                                          │
│          ▼                                                          │
│   ┌──────────────┐                                                  │
│   │  Step 3      │  Validate Before Acting                          │
│   │  行动验证    │  - 委派检查 (MANDATORY)                          │
│   └──────┬───────┘  - 默认偏向: DELEGATE                           │
│          │                                                          │
│          ▼                                                          │
│   ┌──────────────┐     ┌──────────────┐     ┌──────────────┐       │
│   │  Phase 1     │────▶│  Phase 2A    │────▶│  Phase 2B    │       │
│   │  代码库评估  │     │  探索研究    │     │  实现执行    │       │
│   └──────────────┘     └──────────────┘     └──────┬───────┘       │
│                                                    │                │
│                              ┌─────────────────────┘                │
│                              │                                      │
│                              ▼                                      │
│   ┌──────────────┐     ┌──────────────┐     ┌──────────────┐       │
│   │  Phase 2C    │◀────│  Failure     │────▶│  Phase 3     │       │
│   │  失败恢复    │     │  (if needed) │     │  完成验证    │       │
│   └──────────────┘     └──────────────┘     └──────────────┘       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 关键代码片段

### 片段 1: Intent Gate 实现

```typescript
// src/agents/sisyphus/default.ts:189-214
## Phase 0 - Intent Gate (EVERY message)

<intent_verbalization>
### Step 0: Verbalize Intent (BEFORE Classification)

Before classifying the task, identify what the user actually wants 
from you as an orchestrator. Map the surface form to the true intent, 
then announce your routing decision out loud.

**Intent → Routing Map:**

| Surface Form | True Intent | Your Routing |
|---|---|---|
| "explain X", "how does Y work" | Research/understanding | explore/librarian → synthesize → answer |
| "implement X", "add Y", "create Z" | Implementation (explicit) | plan → delegate or execute |
| "look into X", "check Y", "investigate" | Investigation | explore → report findings |
| "what do you think about X?" | Evaluation | evaluate → propose → **wait for confirmation** |
| "I'm seeing error X" / "Y is broken" | Fix needed | diagnose → fix minimally |
| "refactor", "improve", "clean up" | Open-ended change | assess codebase first → propose approach |

**Verbalize before proceeding:**

> "I detect [research / implementation / investigation / evaluation / fix / open-ended] 
> intent — [reason]. My approach: [explore → answer / plan → delegate / clarify first / etc.]."
```

### 片段 2: Boulder State 任务状态管理

```typescript
// src/features/boulder-state/types.ts:8-60
/**
 * Boulder State Types
 *
 * Manages the active work plan state for Sisyphus orchestrator.
 * Named after Sisyphus's boulder - the eternal task that must be rolled.
 */

export interface BoulderState {
  /** Absolute path to the active plan file */
  active_plan: string
  /** ISO timestamp when work started */
  started_at: string
  /** Session IDs that have worked on this plan */
  session_ids: string[]
  /** Plan name derived from filename */
  plan_name: string
  /** Agent type to use when resuming (e.g., 'atlas') */
  agent?: string
  /** Absolute path to the git worktree root where work happens */
  worktree_path?: string
  /** Preferred reusable subagent sessions keyed by current top-level plan task */
  task_sessions?: Record<string, TaskSessionState>
}

export interface TaskSessionState {
  /** Stable identifier for the current top-level plan task */
  task_key: string
  /** Original task label from the plan file */
  task_label: string
  /** Full task title from the plan file */
  task_title: string
  /** Preferred reusable subagent session */
  session_id: string
  /** Agent associated with the task session, when known */
  agent?: string
  /** Category associated with the task session, when known */
  category?: string
  /** Last update timestamp */
  updated_at: string
}
```

---

## 依赖关系

```
Agent 推理循环依赖关系图:

┌─────────────────────────────────────────────────────────────┐
│                    Agent 推理循环                             │
│              (src/agents/sisyphus.ts)                        │
└─────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
        ▼                     ▼                     ▼
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│ Boulder State│    │ Background   │    │   Hooks      │
│ 任务状态持久化 │    │ Agent 管理   │    │ 生命周期钩子 │
│              │    │              │    │              │
│ types.ts     │    │ spawner/     │    │ intent-gate/ │
│ top-level-   │    │ polling.ts   │    │ todo-enforcer│
│ task.ts      │    │              │    │              │
└──────────────┘    └──────────────┘    └──────────────┘
        │                     │                     │
        └─────────────────────┼─────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    工具层 (Tools)                            │
│   LSP, AST-Grep, Tmux, MCP, Git, WebSearch, Context7        │
└─────────────────────────────────────────────────────────────┘
```

---

## 实战示例

### 场景：用户请求 "帮我优化这个 API 的性能"

**推理流程演示**：

#### Step 1: Intent Gate 识别

```
Sisyphus 内部推理:
"I detect [open-ended] intent — user said 'optimize' without specific 
metrics or targets. My approach: assess codebase first → propose approach."
```

#### Step 2: Phase 1 代码库评估

```typescript
// 并行执行探索
// 1. 检查配置文件
read("tsconfig.json")
read("package.json") 

// 2. 采样 API 相关文件
glob("src/api/**/*.ts")
read("src/api/routes.ts")

// 3. 分类项目状态
// → Disciplined (consistent patterns, configs present, tests exist)
```

#### Step 3: Phase 2A 探索研究

```typescript
// 并行委派多个 Explore Agent
task(
  subagent_type="explore",
  run_in_background=true,
  description="Find API performance patterns",
  prompt="I'm optimizing API performance. Find: current API routes, 
          middleware usage, database query patterns, caching implementation. 
          Focus on src/api/ — skip tests."
)

task(
  subagent_type="librarian", 
  run_in_background=true,
  description="Find API optimization best practices",
  prompt="I need current best practices for REST API performance optimization. 
          Find: caching strategies, pagination patterns, N+1 query solutions, 
          response compression. 2026+ guidance only."
)
```

#### Step 4: Phase 2B 实现执行

```typescript
// 创建详细 Todo
todowrite({
  todos: [
    { content: "分析当前 API 性能瓶颈", status: "completed", priority: "high" },
    { content: "添加 Redis 缓存层", status: "in_progress", priority: "high" },
    { content: "优化数据库查询 (N+1)", status: "pending", priority: "high" },
    { content: "实现响应压缩", status: "pending", priority: "medium" },
    { content: "运行性能测试验证", status: "pending", priority: "high" }
  ]
})

// 执行实现...
// 完成后立即标记
// taskUpdate({ ... })
```

#### Step 5: Phase 3 完成验证

```
检查清单:
✓ 所有计划 Todo 项标记完成
✓ 修改文件的诊断信息干净 (lsp_diagnostics 无错误)
✓ 构建通过 (bun run build 退出码 0)
✓ 性能测试显示 40% 延迟改善
✓ 用户原始请求完全满足
```

---

## 交叉引用

- 参见：[主要 Agent](./01-主要 Agent.md) - 了解所有内置 Agent 的详细配置和能力
- 参见：[Agent 编排](./03-Agent 编排.md) - 深入了解 Agent 委派和并行执行机制
- 参见：[意图门](../00-架构概览/01-三层架构模型.md) - 三层架构中的意图识别层设计
- 参见：[Boulder State](../01-core-feature./04-Boulder 状态.md) - 任务状态持久化和恢复机制
- 参见：[背景 Agent](../01-core-feature./01-后台 Agent.md) - 异步 Agent 执行和结果收集
- 参见：[ReAct 模式](../03-advanced-patterns/react-pattern.md) - Reason-Act-Observe 循环的深入解析

---

*文档版本: 1.0 | 最后更新: 2026-03-28 | 作者: OhMyOpenCode Team*
