# Planner / Reviewer / Executor 的角色分工设计

> 本系列第4篇 | 基于项目版本: 2026-03-28 | 源码位置: `src/agents/prometheus/`, `src/agents/metis/`, `src/agents/momus/`, `src/agents/hephaestus/`

---

## 这篇要回答的问题

1. **Planner 负责什么？**
2. **Reviewer 为什么必要？**
3. **Executor 为什么必须受约束？**
4. **Intelligence 在系统里，而不是在某个 Agent 里？**

---

## 源码里这个问题出现在哪里

### Agent 定义: `src/agents/`

所有 Agent 的定义都体现了明确的角色边界：

```typescript
// src/agents/builtin-agents/ 中的角色定义

// Prometheus - 规划者 (Planner)
export const prometheusAgent = {
  name: "prometheus",
  role: "planner",
  description: "Strategic planning consultant",
  
  // 只读权限
  tools: ['read', 'glob', 'grep', 'webfetch'],
  canDelegate: true,      // 可以委派研究任务
  canWriteCode: false,    // 不能写代码
  
  systemPrompt: `
    You are Prometheus, the planner.
    DO NOT write code. DO NOT execute tasks.
    ONLY plan and delegate.
  `
}

// Momus - 审查者 (Reviewer)
export const momusAgent = {
  name: "momus",
  role: "reviewer",
  description: "Plan reviewer",
  
  // 只读权限
  tools: ['read', 'lsp_diagnostics'],
  canDelegate: false,     // 不可再委派
  canWriteCode: false,    // 不能写代码
  
  systemPrompt: `
    You are Momus, the reviewer.
    Check plans for: clarity, verification, context, big picture.
    Reject if quality gates not met.
  `
}

// Hephaestus - 实现者 (Executor)
export const hephaestusAgent = {
  name: "hephaestus",
  role: "executor",
  description: "Autonomous deep worker",
  
  // 全部权限
  tools: ['*'],           // 所有工具
  canDelegate: true,      // 可以委派子任务
  canWriteCode: true,     // 可以写代码
  
  systemPrompt: `
    You are Hephaestus, the executor.
    You implement plans. You write code.
    Follow plans, verify completion.
  `
}

// Sisyphus - 编排者 (Orchestrator)
export const sisyphusAgent = {
  name: "sisyphus",
  role: "orchestrator",
  description: "Main orchestrator",
  
  // 全部权限
  tools: ['*'],
  canDelegate: true,      // 可以委派给任何 agent
  canWriteCode: true,     // 可以写代码
  
  systemPrompt: `
    You are Sisyphus, the orchestrator.
    You drive tasks to completion.
    Plan, delegate, verify, iterate.
  `
}
```

### Agent Categories: `src/tools/delegate-task/constants.ts`

```typescript
// Agent 分类和权限矩阵
export const AGENT_CATEGORIES = {
  // 只读顾问 - 不能写代码，不能再委派
  READONLY_ADVISOR: {
    agents: ['oracle', 'librarian', 'explore'],
    permissions: { read: 'allow', write: 'deny', bash: 'deny' },
    canDelegate: false
  },
  
  // 规划者 - 只读，但可以委派研究
  PLANNER: {
    agents: ['prometheus', 'metis'],
    permissions: { read: 'allow', write: 'deny', bash: 'deny' },
    canDelegate: true
  },
  
  // 审查者 - 只读，不能委派
  REVIEWER: {
    agents: ['momus'],
    permissions: { read: 'allow', write: 'deny', bash: 'deny' },
    canDelegate: false
  },
  
  // 执行者 - 全部权限，可以委派
  EXECUTOR: {
    agents: ['hephaestus', 'sisyphus'],
    permissions: { read: 'allow', write: 'allow', bash: 'ask' },
    canDelegate: true
  },
  
  // 子任务执行者 - 部分权限
  WORKER: {
    agents: ['sisyphus-junior'],
    permissions: { read: 'allow', write: 'allow', bash: 'ask' },
    canDelegate: true
  }
}
```

---

## 它当前的设计方案是什么

### 4.1 角色职责矩阵

| 角色 | Agent | 职责 | 工具权限 | 可委派 |
|------|-------|------|---------|--------|
| **Metis** | 预规划顾问 | 分析需求，识别风险 | 只读 | 否 |
| **Prometheus** | 规划者 | 制定计划，分解任务 | 只读 | 是 |
| **Momus** | 审查者 | 审查计划，验证完整性 | 只读 | 否 |
| **Hephaestus** | 执行者 | 执行任务，落地代码 | 全部 | 是 |
| **Sisyphus** | 编排者 | 主编排，协调整体流程 | 全部 | 是 |
| **Oracle** | 架构顾问 | 架构咨询，调试 | 只读 | 否 |
| **Librarian** | 文档检索 | 文档/代码搜索 | 只读 | 否 |
| **Explore** | 代码探索 | 快速代码搜索 | 只读 | 否 |

### 4.2 为什么不能合并成"超级 Agent"

```
如果合并成"超级 Agent"：

┌─────────────────────────────────────┐
│ 超级 Agent                          │
├─────────────────────────────────────┤
│ • 既规划又执行                      │
│ • 既审查又实现                      │
│ • 既分析又落地                      │
└─────────────────────────────────────┘
        ↓
问题 1: 职责不清
    └── 既规划又执行，容易自我合理化
    └── "我觉得这样实现符合规划"
    
问题 2: 缺乏制衡
    └── 没有独立的审查环节
    └── 自己检查自己的工作
    
问题 3: 上下文膨胀
    └── 所有信息堆在一个上下文
    └── Context overload
    
问题 4: 难以调试
    └── 出问题不知道在哪个环节
    └── 规划错？执行错？还是理解错？
```

### 4.3 Intelligence 分布

```
传统思维（错误）：
┌─────────────────────────────────────┐
│ 超级智能 Agent                      │
│ "模型越强，单 Agent 越能搞定一切"   │
└─────────────────────────────────────┘

OMO 思维（正确）：
┌─────────────────────────────────────┐
│ Intelligence 在系统协作中           │
├─────────────────────────────────────┤
│ Prometheus: 规划智能                │
│ Momus:     审查智能                 │
│ Hephaestus: 实现智能                │
│ Sisyphus:  编排智能                 │
├─────────────────────────────────────┤
│ 系统智能 > 单个 Agent 智能之和      │
└─────────────────────────────────────┘
```

**关键洞察**:
- 规划质量 = 系统设计质量
- 执行稳定性 = 角色约束强度
- 审查有效性 = 独立性保障

---

## 这套方案解决了什么，牺牲了什么

### 解决的问题

**职责清晰**: 每个角色有明确的边界，出了问题知道找谁

**相互制衡**: 规划 → 审查 → 执行，每个环节都有人把关

**可调试性**: 问题可以定位到具体角色和环节

**专业化**: 每个 Agent 可以针对特定任务调优

### 牺牲的代价

**协作成本**: 角色间需要通信和协调

**延迟增加**: 必须经过规划和审查才能执行

**资源消耗**: 多个 Agent = 多次模型调用

**配置复杂度**: 需要理解和配置多个 Agent

---

## 如果我要做自己的开发 Agent 套件，我会怎么迁移

### 迁移策略

```
根据任务复杂度选择角色分工：

简单任务 (1-2个文件修改):
├── Executor 直接执行
└── 无需规划和审查

中等任务 (3-5个文件):
├── Planner 制定计划
└── Executor 执行

复杂任务 (6+文件或架构变更):
├── Planner 制定计划
├── Reviewer 审查
└── Executor 执行
```

### 最小实现

```typescript
// 简单版角色系统
interface Role {
  name: string
  canPlan: boolean
  canReview: boolean
  canExecute: boolean
  permissions: string[]
}

const ROLES: Record<string, Role> = {
  planner: {
    name: 'planner',
    canPlan: true,
    canReview: false,
    canExecute: false,
    permissions: ['read', 'research']
  },
  
  reviewer: {
    name: 'reviewer',
    canPlan: false,
    canReview: true,
    canExecute: false,
    permissions: ['read', 'diagnostics']
  },
  
  executor: {
    name: 'executor',
    canPlan: false,
    canReview: false,
    canExecute: true,
    permissions: ['*']  // 全部权限
  }
}

// 任务流转
async function executeWithRoles(task: Task) {
  // 1. 规划
  const plan = await planner.createPlan(task)
  
  // 2. 审查
  const review = await reviewer.review(plan)
  if (!review.approved) {
    throw new Error('Plan rejected: ' + review.reason)
  }
  
  // 3. 执行
  const result = await executor.execute(plan)
  
  return result
}
```

---

## 一个最小实验

### 实验1: 分析请求流转

观察一个实际请求在系统中的流转：

```
用户: "帮我重构这个模块"
    ↓
Intent Gate 判断: 复杂任务
    ↓
Prometheus (Planner): 制定重构计划
    ├── 分析当前代码
    ├── 识别重构点
    └── 生成任务列表
    ↓
Momus (Reviewer): 审查计划
    ├── 检查完整性
    ├── 验证可行性
    └── 批准/拒绝
    ↓
Hephaestus (Executor): 执行重构
    ├── 修改代码
    ├── 运行测试
    └── 验证结果
    ↓
Sisyphus (Orchestrator): 整体协调
    └── 确认完成
```

### 实验2: 画出角色交互时序图

```
用户     IntentGate   Prometheus   Momus   Hephaestus
 |           |           |          |          |
 |──请求────→|           |          |          |
 |           |──分析────→|          |          |
 |           |           |──规划───→|          |
 |           |           |          |──审查───→|
 |           |           |          |          |──执行──→
 |           |           |          |          |
```

**思考**:
- 如果合并成超级 Agent，会少了哪些检查点？
- 每个角色的输出是什么？输入是什么？

---

## 总结

Planner / Reviewer / Executor 分离的设计：

1. **职责分离**: 规划、审查、执行由不同角色负责
2. **权限隔离**: 规划者只读，执行者可写，审查者独立
3. **系统智能**: Intelligence 分布在协作中，而非集中在单个 Agent
4. **可追踪性**: 每个环节都有明确的责任人和输出

**关键认知**: 复杂 Agent 系统不是"更聪明的单个 Agent"，而是"分工明确、相互制衡的 Agent 团队"。

---

## 下一步

阅读下一篇：**《多 Agent 委派不是炫技，而是稳定性交换》**，理解并行的价值和专职角色的意义。

---

**系列文章**: 《Oh My OpenAgent 源码解析：从复杂 Agent 学习到开发效率套件落地》
- ...
- 第3篇: 复杂任务为什么必须先规划再执行
- 第4篇: Planner / Reviewer / Executor 的角色分工设计（本文）
- 第5篇: 多 Agent 委派不是炫技，而是稳定性交换
- ...（共13篇）
