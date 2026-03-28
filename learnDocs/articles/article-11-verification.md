# 为什么复杂 Agent 不能没有 verification

> 本系列第11篇 | 基于项目版本: 2026-03-28 | 源码位置: `src/agents/momus/`, `src/hooks/verification/`, `src/tools/lsp/diagnostics-tool.ts`

---

## 这篇要回答的问题

1. **Clarity / Verification / Context / Big Picture 各检查什么？**
2. **验收标准怎么定义？**
3. **自动验证和人工复核的边界？**

---

## 源码里这个问题出现在哪里

### Reviewer Agent: `src/agents/momus/`

```typescript
// src/agents/momus/agent.ts 简化版
export const momusAgent = createAgent({
  name: "momus",
  description: "Plan reviewer",
  
  systemPrompt: `
    You are Momus, the plan reviewer.
    Your job is to review plans before execution.
    
    ## Review Dimensions (四维验证)
    
    ### 1. Clarity (清晰度)
    - Is the task clearly defined?
    - Are requirements unambiguous?
    - Are acceptance criteria specific?
    - Verdict: APPROVE / NEEDS_CLARIFICATION
    
    ### 2. Verification (可验证性)
    - Are there clear verification methods?
    - Can completion be objectively measured?
    - Are there test cases?
    - Verdict: APPROVE / NEEDS_VERIFICATION
    
    ### 3. Context (上下文)
    - Is sufficient context provided?
    - Are dependencies identified?
    - Are risks acknowledged?
    - Verdict: APPROVE / NEEDS_CONTEXT
    
    ### 4. Big Picture (全局视角)
    - Does it align with project goals?
    - Are there side effects?
    - Is the approach sound?
    - Verdict: APPROVE / NEEDS_REVIEW
    
    ## Output Format
    For each dimension:
    - Verdict: APPROVE / REJECT
    - Issues: [if any]
    - Suggestions: [if any]
    
    Overall: APPROVE / REJECT
    If REJECT, provide specific feedback.
  `,
  
  tools: ['read', 'lsp_diagnostics'],  // 只读
  canDelegate: false,                  // 不能再委派
  canWriteCode: false                  // 不能写代码
})

// 四维验证实现
interface FourDimensionalReview {
  clarity: {
    verdict: 'APPROVE' | 'NEEDS_CLARIFICATION'
    issues?: string[]
    suggestions?: string[]
  }
  verification: {
    verdict: 'APPROVE' | 'NEEDS_VERIFICATION'
    issues?: string[]
    suggestions?: string[]
  }
  context: {
    verdict: 'APPROVE' | 'NEEDS_CONTEXT'
    issues?: string[]
    suggestions?: string[]
  }
  bigPicture: {
    verdict: 'APPROVE' | 'NEEDS_REVIEW'
    issues?: string[]
    suggestions?: string[]
  }
  overall: 'APPROVE' | 'REJECT'
}

export async function fourDimensionalReview(plan: Plan): Promise<FourDimensionalReview> {
  const review: FourDimensionalReview = {
    clarity: await reviewClarity(plan),
    verification: await reviewVerification(plan),
    context: await reviewContext(plan),
    bigPicture: await reviewBigPicture(plan),
    overall: 'APPROVE'
  }
  
  // 如果任一维度不通过，整体不通过
  if ([review.clarity, review.verification, review.context, review.bigPicture]
      .some(r => r.verdict !== 'APPROVE')) {
    review.overall = 'REJECT'
  }
  
  return review
}
```

### 验证机制: `src/hooks/verification/`

```typescript
// src/hooks/verification/hook.ts 简化版
export function createVerificationHook() {
  return {
    name: "verification",
    
    // 任务完成后验证
    async afterTaskComplete({ task, result }) {
      // 1. 获取验收标准
      const criteria = task.acceptanceCriteria
      
      // 2. 执行验证
      const verification = await verifyTask(task, result, criteria)
      
      // 3. 记录结果
      await saveVerificationResult({
        taskId: task.id,
        result: verification.passed ? 'PASS' : 'FAIL',
        details: verification.details,
        timestamp: Date.now()
      })
      
      // 4. 如果不通过，触发重试或人工介入
      if (!verification.passed) {
        return {
          proceed: false,
          reason: 'Verification failed',
          retry: verification.retryable
        }
      }
      
      return { proceed: true }
    }
  }
}

// 任务验证
interface VerificationResult {
  passed: boolean
  details: VerificationDetail[]
  retryable: boolean
}

async function verifyTask(
  task: Task,
  result: TaskResult,
  criteria: AcceptanceCriteria
): Promise<VerificationResult> {
  const details: VerificationDetail[] = []
  let allPassed = true
  
  // 验证每个标准
  for (const criterion of criteria.items) {
    const detail = await verifyCriterion(task, result, criterion)
    details.push(detail)
    if (!detail.passed) {
      allPassed = false
    }
  }
  
  return {
    passed: allPassed,
    details,
    retryable: details.some(d => d.retryable)
  }
}

async function verifyCriterion(
  task: Task,
  result: TaskResult,
  criterion: Criterion
): Promise<VerificationDetail> {
  switch (criterion.type) {
    case 'file_exists':
      return await verifyFileExists(criterion.path)
      
    case 'code_compiles':
      return await verifyCompilation(task.workingDirectory)
      
    case 'tests_pass':
      return await verifyTests(task.workingDirectory, criterion.pattern)
      
    case 'lint_clean':
      return await verifyLint(task.workingDirectory)
      
    case 'custom':
      return await criterion.verify(result)
      
    default:
      return { passed: false, message: `Unknown criterion type: ${criterion.type}`, retryable: false }
  }
}
```

### LSP 诊断: `src/tools/lsp/diagnostics-tool.ts`

```typescript
// src/tools/lsp/diagnostics-tool.ts 简化版
export const lspDiagnosticsTool = createTool({
  name: "lsp_diagnostics",
  description: "Get LSP diagnostics for files",
  
  parameters: z.object({
    filePath: z.string().describe("File or directory to check"),
    severity: z.enum(['error', 'warning', 'information', 'hint', 'all']).optional(),
    extension: z.string().optional().describe("File extension filter")
  }),
  
  async execute({ filePath, severity = 'all', extension }, ctx) {
    // 1. 获取 LSP 客户端
    const lspClient = await getLspClient(filePath)
    
    // 2. 请求诊断
    const diagnostics = await lspClient.getDiagnostics(filePath, { extension })
    
    // 3. 过滤严重程度
    const filtered = severity === 'all' 
      ? diagnostics 
      : diagnostics.filter(d => d.severity === severity)
    
    // 4. 限制返回数量
    const limited = filtered.slice(0, DEFAULT_MAX_DIAGNOSTICS)
    
    return {
      diagnostics: limited,
      total: filtered.length,
      truncated: filtered.length > DEFAULT_MAX_DIAGNOSTICS
    }
  }
})

// 诊断结果处理
interface Diagnostic {
  file: string
  line: number
  character: number
  severity: 'error' | 'warning' | 'information' | 'hint'
  message: string
  code?: string
  source?: string
}
```

---

## 它当前的设计方案是什么

### 11.1 四维验证

| 维度 | 检查内容 | 不通过的后果 | 示例 |
|------|---------|-------------|------|
| **Clarity** | 任务是否清晰 | 返回澄清请求 | "优化代码" → "需要明确优化什么" |
| **Verification** | 是否有验证手段 | 要求补充验收标准 | 没有测试计划 → 无法确认完成 |
| **Context** | 上下文是否充分 | 提示需要更多信息 | 缺少依赖说明 → 可能遗漏文件 |
| **Big Picture** | 是否理解全局 | 提供背景说明 | 修改影响未知 → 可能引入 bug |

### 11.2 验收标准定义

```typescript
// Acceptance Criteria 接口
interface AcceptanceCriteria {
  // 功能完成
  functional: {
    description: string
    verify: () => Promise<boolean>
  }[]
  
  // 测试通过
  tests: {
    command: string
    expectedOutput: string
    expectedExitCode: number
  }[]
  
  // 代码质量
  quality: {
    type: 'lint' | 'typecheck' | 'format'
    command: string
    mustPass: boolean
  }[]
  
  // 文档完整
  documentation: {
    required: string[]
    optional: string[]
  }
}

// 示例
const exampleCriteria: AcceptanceCriteria = {
  functional: [
    {
      description: 'Login API returns JWT token',
      verify: async () => {
        const response = await fetch('/api/login', { /* ... */ })
        return response.token !== undefined
      }
    }
  ],
  
  tests: [
    {
      command: 'npm test auth',
      expectedOutput: 'all tests passed',
      expectedExitCode: 0
    }
  ],
  
  quality: [
    { type: 'lint', command: 'npm run lint', mustPass: true },
    { type: 'typecheck', command: 'tsc --noEmit', mustPass: true }
  ],
  
  documentation: {
    required: ['README.md', 'API.md'],
    optional: ['CHANGELOG.md']
  }
}
```

### 11.3 自动验证 vs 人工复核

```
自动验证：
├── LSP diagnostics（类型错误、语法错误）
├── 测试运行（npm test）
├── Lint 检查（eslint, prettier）
├── 构建验证（npm run build）
└── 代码覆盖率（coverage report）

人工复核：
├── 设计决策（架构是否合理）
├── 业务逻辑（是否符合需求）
├── 边界情况（是否都处理了）
├── 用户体验（是否易用）
└── 安全性（是否有漏洞）

分界线：
├── 能自动化的 → 自动验证
│   └── 语法、类型、测试、构建
├── 需要判断的 → 人工复核
│   └── 设计、业务、UX
└── 两者结合 → 先自动后人工
    └── 自动通过后再人工复核
```

---

## 这套方案解决了什么，牺牲了什么

### 解决的问题

**交付质量可控**: 每个任务都有明确的验收标准

**自动化检查**: 减少人工检查的重复工作

**可追溯性**: 每个验收都有记录

**失败可恢复**: 不通过时知道原因，可以重试

### 牺牲的代价

**流程复杂度**: 增加了验证环节

**时间成本**: 验证需要时间

**假阴性**: 自动验证可能误判

**维护成本**: 验收标准需要维护

---

## 如果我要做自己的开发 Agent 套件，我会怎么迁移

### 验收标准模板

```typescript
// 验收标准模板
interface TaskTemplate {
  name: string
  criteria: AcceptanceCriteria
}

const TASK_TEMPLATES: Record<string, TaskTemplate> = {
  'feature': {
    name: 'Feature Implementation',
    criteria: {
      functional: ['Feature works as specified'],
      tests: ['Unit tests pass', 'Integration tests pass'],
      quality: ['Lint clean', 'Type check pass'],
      documentation: ['README updated', 'API docs updated']
    }
  },
  
  'bugfix': {
    name: 'Bug Fix',
    criteria: {
      functional: ['Bug is fixed', 'No regression'],
      tests: ['Regression test added', 'All tests pass'],
      quality: ['Lint clean'],
      documentation: ['CHANGELOG updated']
    }
  },
  
  'refactor': {
    name: 'Code Refactoring',
    criteria: {
      functional: ['Behavior unchanged'],
      tests: ['All existing tests pass'],
      quality: ['Code quality improved', 'Lint clean'],
      documentation: ['Comments updated']
    }
  }
}

// 使用模板
function getCriteriaForTask(type: string): AcceptanceCriteria {
  return TASK_TEMPLATES[type]?.criteria || TASK_TEMPLATES['feature'].criteria
}
```

### 自动验证流程

```typescript
// 验证管道
class VerificationPipeline {
  private checks: VerificationCheck[] = []
  
  addCheck(check: VerificationCheck) {
    this.checks.push(check)
    return this
  }
  
  async run(context: VerificationContext): Promise<VerificationResult> {
    const results: CheckResult[] = []
    
    for (const check of this.checks) {
      console.log(`Running: ${check.name}`)
      const result = await check.run(context)
      results.push(result)
      
      // 如果有检查失败且是 required，停止
      if (!result.passed && check.required) {
        return {
          passed: false,
          failedAt: check.name,
          results
        }
      }
    }
    
    const allPassed = results.every(r => r.passed)
    return {
      passed: allPassed,
      results
    }
  }
}

// 使用
const pipeline = new VerificationPipeline()
  .addCheck({ name: 'Lint', run: runLint, required: true })
  .addCheck({ name: 'TypeCheck', run: runTypeCheck, required: true })
  .addCheck({ name: 'Tests', run: runTests, required: true })
  .addCheck({ name: 'Build', run: runBuild, required: false })
  .addCheck({ name: 'Coverage', run: checkCoverage, required: false })

const result = await pipeline.run({ workingDirectory: '.' })
```

---

## 一个最小实验

### 实验1: 分析 Momus 审查逻辑

```bash
# 查看 Momus Agent 实现
cat src/agents/momus/agent.ts

# 查看四维验证的具体检查点
grep -r "reviewClarity\|reviewVerification\|reviewContext\|reviewBigPicture" src/agents/momus/

# 查看验证 hook 实现
cat src/hooks/verification/hook.ts
```

**观察重点**:
- 每个维度检查什么？
- 如何判断通过/不通过？
- 反馈如何组织？

### 实验2: 设计简单验收清单

```typescript
// my-verification-checklist.ts

interface ChecklistItem {
  id: string
  description: string
  verify: () => Promise<boolean>
  required: boolean
}

const MY_CHECKLIST: ChecklistItem[] = [
  {
    id: '1',
    description: '代码编译通过',
    verify: async () => {
      try {
        await $`tsc --noEmit`
        return true
      } catch {
        return false
      }
    },
    required: true
  },
  {
    id: '2',
    description: 'Lint 检查通过',
    verify: async () => {
      try {
        await $`eslint src/`
        return true
      } catch {
        return false
      }
    },
    required: true
  },
  {
    id: '3',
    description: '测试通过',
    verify: async () => {
      try {
        await $`npm test`
        return true
      } catch {
        return false
      }
    },
    required: true
  },
  {
    id: '4',
    description: '代码覆盖率 > 80%',
    verify: async () => {
      const result = await $`npm run coverage -- --json`
      const coverage = JSON.parse(result.stdout)
      return coverage.total.lines.pct >= 80
    },
    required: false
  }
]

// 运行检查
async function runVerification() {
  console.log('Running verification checklist...\n')
  
  let allRequiredPassed = true
  
  for (const item of MY_CHECKLIST) {
    const status = item.required ? '[Required]' : '[Optional]'
    process.stdout.write(`${status} ${item.description}... `)
    
    const passed = await item.verify()
    console.log(passed ? '✅' : '❌')
    
    if (!passed && item.required) {
      allRequiredPassed = false
    }
  }
  
  console.log(`\n${allRequiredPassed ? '✅ All required checks passed!' : '❌ Some required checks failed'}`)
  return allRequiredPassed
}

runVerification()
```

---

## 总结

Verification 机制的核心价值：

1. **四维验证**: Clarity、Verification、Context、Big Picture
2. **验收标准**: 功能、测试、质量、文档四个维度
3. **自动+人工**: 能自动的自动，需要判断的人工
4. **可追溯**: 每个验收都有记录

**关键认知**: Verification 不是"锦上添花"，而是"能交付"的保障。

---

## 下一步

阅读下一篇：**《权限、回退与恢复，Agent 系统怎么才能日常可用》**，理解安全和稳定性。

---

**系列文章**: 《Oh My OpenAgent 源码解析：从复杂 Agent 学习到开发效率套件落地》
- ...
- 第10篇: 做一个开发效率增强模块：从源码理解到实战改造
- 第11篇: 为什么复杂 Agent 不能没有 verification（本文）
- 第12篇: 权限、回退与恢复，Agent 系统怎么才能日常可用
- ...（共13篇）
