# 多 Agent 委派不是炫技，而是稳定性交换

> 本系列第5篇 | 基于项目版本: 2026-03-28 | 源码位置: `src/tools/delegate-task/`, `src/tools/delegate-task/constants.ts`

---

## 这篇要回答的问题

1. **并行的价值是什么？**
2. **专职角色的价值是什么？**
3. **什么时候不该多 Agent？**

---

## 源码里这个问题出现在哪里

### 委派工具: `src/tools/delegate-task/`

```typescript
// src/tools/delegate-task/tool.ts 简化版
export const delegateTaskTool = createTool({
  name: "task",
  description: "Delegate task to subagent",
  
  parameters: z.object({
    category: z.enum(['quick', 'deep', 'ultrabrain', 'visual-engineering'])
      .describe("Task category for model matching"),
    description: z.string().describe("Task description"),
    prompt: z.string().describe("Full prompt for subagent"),
    load_skills: z.array(z.string()).optional(),
  }),
  
  async execute({ category, description, prompt, load_skills }, ctx) {
    // 1. 根据 category 匹配合适的模型
    const model = matchModelByCategory(category)
    
    // 2. 创建子 Agent 会话
    const subagent = await createSubagent({
      model,
      skills: load_skills,
      parentContext: ctx.session.id
    })
    
    // 3. 在后台执行（非阻塞）
    const taskId = await subagent.runInBackground(prompt)
    
    // 4. 返回任务句柄
    return {
      task_id: taskId,
      status: 'running',
      check_command: `/background_output ${taskId}`
    }
  }
})
```

### Category 配置: `src/tools/delegate-task/constants.ts`

```typescript
// Agent Category 到模型的映射
export const CATEGORY_MODEL_REQUIREMENTS = {
  // 快速任务 → 轻量模型
  'quick': {
    minTokens: 16_000,
    recommended: ['claude-haiku', 'gpt-3.5-turbo'],
    maxConcurrency: 10
  },
  
  // 深度研究 → 强力模型
  'deep': {
    minTokens: 100_000,
    recommended: ['claude-opus-4', 'kimi-k2.5'],
    maxConcurrency: 3
  },
  
  // 复杂逻辑 → 最强模型
  'ultrabrain': {
    minTokens: 200_000,
    recommended: ['gpt-4-turbo', 'claude-opus-4'],
    maxConcurrency: 1
  },
  
  // 视觉工程 → 视觉优化模型
  'visual-engineering': {
    minTokens: 50_000,
    recommended: ['claude-sonnet', 'gemini-pro-vision'],
    maxConcurrency: 5
  }
}

// 默认 Category
export const DEFAULT_CATEGORIES = [
  'quick',
  'deep', 
  'ultrabrain',
  'visual-engineering'
] as const
```

### 并行委派示例

```typescript
// 使用 task 工具并行委派
async function parallelDelegation() {
  // 任务1: 搜索文档（快速）
  const task1 = await task({
    category: 'quick',
    description: 'Search for auth patterns',
    prompt: 'Search codebase for authentication implementations'
  })
  
  // 任务2: 分析架构（深度）
  const task2 = await task({
    category: 'deep',
    description: 'Analyze system architecture',
    prompt: 'Analyze the plugin architecture and data flow'
  })
  
  // 任务3: 设计组件（视觉）
  const task3 = await task({
    category: 'visual-engineering',
    description: 'Design UI component',
    prompt: 'Design a login form component with validation'
  })
  
  // 并行等待所有结果
  const [result1, result2, result3] = await Promise.all([
    waitForTask(task1.task_id),
    waitForTask(task2.task_id),
    waitForTask(task3.task_id)
  ])
  
  return mergeResults(result1, result2, result3)
}
```

---

## 它当前的设计方案是什么

### 5.1 串行 vs 并行对比

```
串行执行：
Task1 ──→ Task2 ──→ Task3 ──→ Task4
 5min      5min      5min      5min
         总计: 20分钟

并行执行：
Task1 ┐
Task2 ├→ 合并结果
Task3 ┘
 5min     5min
         总计: 5分钟
```

**并行优势**:

| 优势 | 说明 |
|------|------|
| **时间效率** | 独立任务同时执行，节省时间 |
| **容错性** | 一个失败不影响其他 |
| **上下文隔离** | 各任务独立上下文，不互相污染 |
| **资源利用** | 充分利用多核/多模型能力 |

### 5.2 专职角色的价值

| 类型 | Agent | 优势 |
|------|-------|------|
| **只读顾问** | Oracle, Librarian, Explore | 不会误操作，专注分析 |
| **规划者** | Prometheus, Metis | 专注策略，不干扰执行 |
| **执行者** | Hephaestus | 有写权限，专注实现 |
| **审查者** | Momus | 独立视角，质量控制 |
| **编排者** | Sisyphus | 全局协调，调度资源 |

**为什么需要专职角色？**

```
通用 Agent（反模式）：
┌─────────────────────────────────────┐
│ 通用 Agent                          │
│ • 既会搜索又会写代码                │
│ • 既会规划又会审查                  │
│ • 一个 prompt 走天下                │
└─────────────────────────────────────┘
    ↓
问题：什么都会 = 什么都不精
     上下文混杂 = 容易出错

专职 Agent（正确模式）：
┌─────────────────────────────────────┐
│ Librarian │ 专注搜索               │
├─────────────────────────────────────┤
│ Hephaestus │ 专注实现              │
├─────────────────────────────────────┤
│ Momus     │ 专注审查               │
└─────────────────────────────────────┘
    ↓
优势：每个角色针对特定任务优化
     职责单一 = 质量更高
```

### 5.3 什么时候不该多 Agent

不是所有场景都适合多 Agent。以下情况**不应该**使用多 Agent：

| 情况 | 原因 | 推荐方案 |
|------|------|---------|
| **任务简单** | 单 Agent 可完成 | 直接执行 |
| **依赖强** | 无法并行，必须串行 | 顺序执行 |
| **上下文共享需求高** | 需要频繁信息交换 | 单 Agent |
| **成本敏感** | 多 Agent = 多 token | 单 Agent |
| **实时性要求高** | 并行有协调开销 | 单 Agent |

**决策树**:

```
是否使用多 Agent？
    │
    ├── 任务简单？ → 单 Agent
    │
    ├── 任务依赖强？ → 单 Agent
    │
    ├── 成本敏感？ → 单 Agent
    │
    └── 任务独立且复杂？ → 多 Agent
            │
            ├── 需要不同类型工作？ → 不同 Category
            │
            └── 同类型工作？ → 同 Category 并行
```

---

## 这套方案解决了什么，牺牲了什么

### 解决的问题

**稳定性**: 通过分工实现稳定性，单个 Agent 失败不影响整体

**效率**: 独立任务并行执行，节省时间

**质量**: 专职角色针对特定任务优化

**可扩展性**: 可以根据任务类型动态分配资源

### 牺牲的代价

**复杂度**: 需要理解 Category 和模型匹配

**成本**: 多 Agent = 多次模型调用 = 更多 token

**协调开销**: 需要等待和合并结果

**上下文割裂**: 子 Agent 之间不共享上下文

---

## 如果我要做自己的开发 Agent 套件，我会怎么迁移

### 委派策略设计

```
1. 评估任务复杂度
   └── 简单？→ 单 Agent
   └── 复杂？→ 多 Agent

2. 分析任务依赖
   └── 独立？→ 并行执行
   └── 依赖？→ 顺序执行

3. 选择 Category
   └── 代码搜索 → quick
   └── 架构设计 → deep
   └── 算法难题 → ultrabrain
   └── UI设计 → visual-engineering

4. 设置并发限制
   └── quick: 10并发
   └── deep: 3并发
   └── ultrabrain: 1并发
```

### 最小实现

```typescript
// 简单版委派系统
interface TaskConfig {
  type: 'quick' | 'deep' | 'parallel'
  description: string
  dependencies?: string[]
}

async function executeTasks(tasks: TaskConfig[]) {
  // 1. 分组：有依赖 vs 无依赖
  const independent = tasks.filter(t => !t.dependencies?.length)
  const dependent = tasks.filter(t => t.dependencies?.length)
  
  // 2. 并行执行独立任务
  const parallelResults = await Promise.all(
    independent.map(t => executeTask(t))
  )
  
  // 3. 顺序执行依赖任务
  const sequentialResults = []
  for (const task of dependent) {
    // 等待依赖完成
    await waitForDependencies(task.dependencies!)
    const result = await executeTask(task)
    sequentialResults.push(result)
  }
  
  return [...parallelResults, ...sequentialResults]
}
```

---

## 一个最小实验

### 实验1: 分析 delegate-task 逻辑

```bash
# 查看委派工具实现
cat src/tools/delegate-task/tool.ts

# 查看 Category 配置
cat src/tools/delegate-task/constants.ts
```

**观察重点**:
- Category 如何映射到模型？
- 并发限制如何配置？
- 子 Agent 的上下文如何隔离？

### 实验2: 设计自己的委派策略

创建一个简单的委派策略：

```typescript
// my-delegation-strategy.ts

// 任务分类器
function classifyTask(description: string): TaskCategory {
  if (description.includes('搜索') || description.includes('查找')) {
    return 'quick'
  }
  if (description.includes('设计') || description.includes('架构')) {
    return 'deep'
  }
  if (description.includes('UI') || description.includes('界面')) {
    return 'visual-engineering'
  }
  return 'quick'  // 默认
}

// 并发控制器
class ConcurrencyController {
  private running = new Map<string, number>()
  
  private limits = {
    quick: 5,
    deep: 2,
    'visual-engineering': 3
  }
  
  canStart(category: string): boolean {
    const current = this.running.get(category) || 0
    return current < this.limits[category]
  }
  
  start(category: string) {
    this.running.set(category, (this.running.get(category) || 0) + 1)
  }
  
  finish(category: string) {
    this.running.set(category, (this.running.get(category) || 0) - 1)
  }
}
```

---

## 总结

多 Agent 委派的价值：

1. **并行价值**: 独立任务同时执行，节省时间，提高容错性
2. **专职角色**: 每个 Agent 针对特定任务优化，质量更高
3. **合理选择**: 不是所有场景都适合多 Agent，需要权衡

**关键认知**: 多 Agent 不是"炫技"，而是**用分工换取稳定性**。

---

## 下一步

阅读下一篇：**《AGENTS.md、rules 与目录上下文注入机制》**，理解 Agent 的"外部脑"是如何构建的。

---

**系列文章**: 《Oh My OpenAgent 源码解析：从复杂 Agent 学习到开发效率套件落地》
- ...
- 第4篇: Planner / Reviewer / Executor 的角色分工设计
- 第5篇: 多 Agent 委派不是炫技，而是稳定性交换（本文）
- 第6篇: AGENTS.md、rules 与目录上下文注入机制
- ...（共13篇）
