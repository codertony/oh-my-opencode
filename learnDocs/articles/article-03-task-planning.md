# 复杂任务为什么必须先规划再执行

> 本系列第3篇 | 基于项目版本: 2026-03-28 | 源码位置: `src/hooks/intent-gate/`, `src/agents/prometheus/`, `src/agents/metis/`

---

## 这篇要回答的问题

1. **复杂度判断如何做？**
2. **规划入口在哪里？**
3. **任务拆分粒度如何控制？**
4. **验收条件如何定义？**

---

## 源码里这个问题出现在哪里

### Intent Gate: `src/hooks/intent-gate/`

这是任务进入系统的第一道关卡：

```typescript
// src/hooks/intent-gate/hook.ts 简化版
export function createIntentGateHook() {
  return {
    name: "intent-gate",
    
    async beforeChatMessage({ message, context }) {
      // 分析用户请求复杂度
      const complexity = analyzeComplexity(message.content)
      
      if (complexity.level === 'simple') {
        // 简单任务 → 直接执行
        return { proceed: true, strategy: 'direct' }
      } else {
        // 复杂任务 → 进入规划流程
        return { proceed: true, strategy: 'plan', invoke: 'prometheus' }
      }
    }
  }
}

// 复杂度分析
function analyzeComplexity(request: string): ComplexityResult {
  const indicators = {
    // 涉及模块数
    modules: countModules(request),
    // 是否需要外部资源
    external: needsExternalResources(request),
    // 是否需要多步骤
    multiStep: impliesMultiStep(request),
    // 是否有明确验收标准
    hasCriteria: hasAcceptanceCriteria(request)
  }
  
  // 综合评分
  return calculateComplexity(indicators)
}
```

### 规划 Agent: `src/agents/prometheus/`

Prometheus 是规划 Agent，负责制定执行计划：

```typescript
// src/agents/prometheus/agent.ts 简化版
export const prometheusAgent = createAgent({
  name: "prometheus",
  description: "Strategic planning consultant",
  
  systemPrompt: `
    You are Prometheus, the strategic planning consultant.
    
    ## Role
    - Interview the user to understand requirements
    - Research via explore/librarian agents
    - Make informed suggestions
    - Generate work plans
    
    ## Constraints
    - DO NOT write code
    - DO NOT execute tasks
    - ONLY plan and delegate
    
    ## Output Format
    Every plan must include:
    1. TL;DR (quick summary)
    2. Context (what we discussed)
    3. Work Objectives (concrete deliverables)
    4. Verification Strategy (how to verify)
    5. Execution Strategy (parallelization plan)
    6. TODOs (specific, actionable tasks)
  `,
  
  tools: ['read', 'glob', 'grep'], // 只读工具
  canDelegate: true,              // 可以委派研究任务
})
```

### 预规划分析: `src/agents/metis/`

Metis 在 Prometheus 之前运行，提供预规划分析：

```typescript
// src/agents/metis/agent.ts 简化版
export const metisAgent = createAgent({
  name: "metis",
  description: "Pre-planning consultant",
  
  systemPrompt: `
    You are Metis, catching gaps before planning.
    
    ## Role
    - Review planning session before work plan generation
    - Identify missing questions
    - Flag scope creep areas
    - Validate assumptions
    
    ## Output
    List of gaps classified as:
    - CRITICAL: Requires user input
    - MINOR: Can self-resolve
    - AMBIGUOUS: Has reasonable default
  `,
  
  tools: ['read'],  // 只读
  canDelegate: false // 不可再委派
})
```

---

## 它当前的设计方案是什么

### 3.1 Intent Gate（意图门）

```
用户请求 → Intent Gate
                │
                ├── 简单任务？ → 直接执行
                │
                └── 复杂任务？ → 进入规划流程
                                  │
                                  ↓
                         Metis（预规划分析）
                                  │
                                  ↓
                         Prometheus（制定计划）
                                  │
                                  ↓
                         Momus（审查计划）
                                  │
                                  ↓
                              执行
```

### 3.2 复杂度判断维度

| 维度 | 简单任务 | 复杂任务 |
|------|---------|---------|
| **涉及模块数** | 1-2个文件 | 3+个文件或跨模块 |
| **外部资源** | 不需要 | 需要搜索/文档/API |
| **步骤数量** | 单步骤 | 多步骤有依赖 |
| **验收标准** | 明确具体 | 模糊需澄清 |

### 3.3 任务拆分原则

```
任务拆分四原则：

1. 单一职责
   └── 每个任务一个明确目标
   
2. 可验证
   └── 每个任务有明确的验收标准
   
3. 可委派
   └── 明确谁能执行（哪个 agent）
   
4. 粒度适中
   └── 不过大（难以验证）
   └── 不过小（overhead太高）
```

### 3.4 规划流程详解

```
完整规划流程（5步）：

Step 1: Metis 预规划分析
    ├── 识别缺失的问题
    ├── 标记潜在的范围蔓延
    └── 验证关键假设
    
Step 2: Prometheus 制定计划
    ├── 采访用户明确需求
    ├── 研究现有代码/文档
    └── 生成详细工作计划
    
Step 3: Momus 审查计划
    ├── 检查计划完整性
    ├── 验证可执行性
    └── 提出改进建议
    
Step 4: 计划确认
    ├── 用户确认（如有需要）
    └── 最终定稿
    
Step 5: 执行
    └── Sisyphus 按计划执行
```

---

## 这套方案解决了什么，牺牲了什么

### 解决的问题

**任务可控执行**: 复杂任务被拆分成可管理的子任务，每步可追踪

**需求澄清**: 通过 Metis 和 Prometheus 的采访模式，避免理解偏差

**质量保障**: Momus 审查确保计划的可执行性和完整性

### 牺牲的代价

**延迟增加**: 简单任务也要经过复杂度判断，增加了响应时间

**token 消耗**: 规划阶段需要额外的模型调用

**用户等待**: 复杂任务需要等待规划完成才能开始执行

---

## 如果我要做自己的开发 Agent 套件，我会怎么迁移

### 迁移步骤

```
1. 设计 Intent Gate 规则
   └── 定义什么算"简单"，什么算"复杂"
   
2. 定义任务拆分模板
   └── 每种任务类型的标准拆分模式
   
3. 创建规划 Agent
   └── 可以参考 Prometheus 的 prompt 设计
   
4. 建立审查机制
   └── 计划执行前的质量检查点
```

### 简化版实现

```typescript
// 最小版 Intent Gate
function shouldPlan(request: string): boolean {
  // 简单规则：超过100字符或包含"实现"/"重构"等关键词 → 需要规划
  return request.length > 100 || 
         /实现|重构|添加功能|修改架构/.test(request)
}

// 最小版任务拆分
function createSimplePlan(goal: string): Task[] {
  return [
    { step: 1, action: '分析需求', verify: '需求文档完成' },
    { step: 2, action: '设计方案', verify: '设计评审通过' },
    { step: 3, action: '实现代码', verify: '测试通过' },
    { step: 4, action: '验证交付', verify: '验收完成' }
  ]
}
```

---

## 一个最小实验

### 实验1: 阅读 Intent Gate 逻辑

```bash
# 查看 Intent Gate 实现
cat src/hooks/intent-gate/hook.ts
```

**观察重点**:
- 如何判断复杂度？
- 简单任务和复杂任务的 routing 有何不同？

### 实验2: 设计简单复杂度规则

创建一个你自己的复杂度判断规则：

```typescript
// my-intent-gate.ts
interface ComplexityRule {
  name: string
  check: (request: string) => boolean
  weight: number
}

const rules: ComplexityRule[] = [
  { name: '长度', check: r => r.length > 100, weight: 1 },
  { name: '多文件', check: r => /多个文件|模块|架构/.test(r), weight: 2 },
  { name: '外部依赖', check: r => /搜索|查找|调研/.test(r), weight: 1 },
]

function calculateComplexity(request: string): 'simple' | 'complex' {
  const score = rules.reduce((sum, rule) => 
    sum + (rule.check(request) ? rule.weight : 0), 0)
  return score >= 2 ? 'complex' : 'simple'
}
```

---

## 总结

复杂任务必须先规划再执行，因为：

1. **复杂度判断**: Intent Gate 识别任务复杂度，决定执行策略
2. **预规划分析**: Metis 在正式规划前识别风险和缺口
3. **规划流程**: Prometheus 制定详细可执行的计划
4. **审查机制**: Momus 确保计划质量

**关键认知**: 规划不是 overhead，而是确保复杂任务能够正确完成的必要投资。

---

## 下一步

阅读下一篇：**《Planner / Reviewer / Executor 的角色分工设计》**，深入理解为什么这些角色不能合并成"超级 Agent"。

---

**系列文章**: 《Oh My OpenAgent 源码解析：从复杂 Agent 学习到开发效率套件落地》
- 第1篇: 为什么 Oh My OpenAgent 值得作为复杂 Agent 学习样本
- 第2篇: OpenCode 底座与 OMO 编排层：边界在哪
- 第3篇: 复杂任务为什么必须先规划再执行（本文）
- ...（共13篇）
