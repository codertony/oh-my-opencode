# 做一个开发效率增强模块：从源码理解到实战改造

> 本系列第10篇 | 基于项目版本: 2026-03-28 | 源码位置: `src/features/builtin-skills/`, `src/features/builtin-commands/`

---

## 这篇要回答的问题

1. **如何选择落地场景？**
2. **如何设计和实现？**
3. **如何验证效果？**

---

## 源码里这个问题出现在哪里

### Skills 实现: `src/features/builtin-skills/`

```typescript
// src/features/builtin-skills/skill.ts
export interface Skill {
  name: string
  description: string
  systemPrompt: string
  mcpServers?: MCPServerConfig[]
  tools?: string[]
  scope?: 'global' | 'project'
}

// 内置 Skills
export const BUILTIN_SKILLS: Record<string, Skill> = {
  // Playwright Skill - 浏览器自动化
  playwright: {
    name: 'playwright',
    description: 'Browser automation via Playwright MCP',
    systemPrompt: `
      You are a browser automation expert.
      Use Playwright for all web interactions.
      
      Best practices:
      - Always wait for page load
      - Use specific selectors (id, data-testid)
      - Handle errors gracefully
      - Take screenshots for verification
      
      Available operations:
      - Navigate to URL
      - Click elements
      - Fill forms
      - Extract data
      - Take screenshots
    `,
    mcpServers: [{
      name: 'playwright',
      command: 'npx @anthropic-ai/playwright-mcp-server'
    }],
    scope: 'global'
  },
  
  // Git Master Skill - Git 操作
  'git-master': {
    name: 'git-master',
    description: 'Git operations and atomic commits',
    systemPrompt: `
      You are a Git expert focused on clean history.
      
      Rules:
      1. Use atomic commits (one logical change per commit)
      2. Write clear, conventional commit messages
      3. Rebase for clean history when appropriate
      4. Never force push to shared branches
      
      Commit message format:
      type(scope): description
      
      Types:
      - feat: new feature
      - fix: bug fix
      - docs: documentation
      - refactor: code change
      - test: tests
      - chore: build/tooling
    `,
    tools: ['bash', 'read', 'edit'],
    scope: 'global'
  }
}

// Skill 注册
export function registerSkill(skill: Skill) {
  // 1. 验证 Skill 配置
  validateSkill(skill)
  
  // 2. 注册到 SkillRegistry
  SkillRegistry.register(skill)
  
  // 3. 启动关联的 MCP servers
  if (skill.mcpServers) {
    skill.mcpServers.forEach(mcp => {
      SkillMcpManager.start(mcp)
    })
  }
  
  // 4. 记录到日志
  logger.info(`Skill registered: ${skill.name}`)
}
```

### Commands 实现: `src/features/builtin-commands/`

```typescript
// src/features/builtin-commands/command.ts
export interface Command {
  name: string
  description: string
  handler: (args: string[], ctx: CommandContext) => Promise<void>
}

// 内置 Commands
export const BUILTIN_COMMANDS: Record<string, Command> = {
  // /ultrawork 命令
  ultrawork: {
    name: 'ultrawork',
    description: 'Start ultrawork mode',
    async handler(args, ctx) {
      // 1. 分析当前任务
      const currentTask = await getCurrentTask(ctx)
      
      // 2. 激活所有 agents
      await activateAgents(['sisyphus', 'hephaestus', 'prometheus'])
      
      // 3. 设置 todo enforcer
      await enableTodoEnforcer()
      
      // 4. 启动 ralph loop
      await startRalphLoop({
        goal: currentTask,
        maxIterations: 100
      })
      
      ctx.output('Ultrawork mode activated. Working until done...')
    }
  },
  
  // /init-deep 命令
  'init-deep': {
    name: 'init-deep',
    description: 'Generate hierarchical AGENTS.md files',
    async handler(args, ctx) {
      const rootDir = ctx.workspaceDirectory
      
      // 1. 分析项目结构
      const structure = await analyzeProjectStructure(rootDir)
      
      // 2. 生成根级 AGENTS.md
      await generateRootAgentsMd(rootDir, structure)
      
      // 3. 为每个模块生成 AGENTS.md
      for (const module of structure.modules) {
        await generateModuleAgentsMd(module.path, module)
      }
      
      ctx.output('AGENTS.md files generated successfully!')
    }
  }
}
```

---

## 它当前的设计方案是什么

### 10.1 设计步骤

```
开发效率增强模块的设计流程（6步）：

Step 1: 定义目标场景
    ├── 痛点分析: 当前开发流程中哪里效率低？
    ├── 场景选择: PR评审？代码重构？Bug排查？
    └── 价值评估: 节省多少时间？使用频率？

Step 2: 分析所需能力
    ├── 需要什么工具？
    ├── 需要什么数据？
    ├── 需要什么权限？
    └── 需要什么集成？

Step 3: 选择扩展层
    ├── 纯规则能搞定？→ AGENTS.md
    ├── 需要拦截？→ Plugin Hook
    ├── 需要新能力？→ Custom Tool
    └── 需要外部服务？→ MCP

Step 4: 实现核心逻辑
    ├── 设计接口
    ├── 实现主体
    ├── 错误处理
    └── 日志记录

Step 5: 编写测试用例
    ├── 正常路径
    ├── 边界情况
    ├── 错误处理
    └── 集成测试

Step 6: 集成验证
    ├── 本地测试
    ├── 团队试用
    ├── 收集反馈
    └── 迭代优化
```

### 10.2 PR Review 助手示例

```typescript
// PR Review Skill 设计
interface PRReviewSkill {
  // 获取 PR 变更
  fetchChanges(prUrl: string): Promise<PRChanges>
  
  // 分析变更
  analyzeChanges(changes: PRChanges): Analysis
  
  // 生成建议
  generateSuggestions(analysis: Analysis): Suggestion[]
  
  // 输出报告
  outputReport(suggestions: Suggestion[]): void
}

// 实现
export const prReviewSkill: Skill = {
  name: 'pr-review',
  description: 'Automated PR review assistant',
  
  systemPrompt: `
    You are a PR review assistant.
    
    Review checklist:
    1. Code quality (readability, maintainability)
    2. Security (injection, XSS, etc.)
    3. Performance (inefficient algorithms)
    4. Tests (coverage, quality)
    5. Documentation (comments, README)
    
    Output format:
    - Severity: critical/warning/suggestion
    - Location: file:line
    - Issue: description
    - Suggestion: how to fix
    
    Be constructive, not critical.
  `,
  
  tools: [
    'read',           // 读取文件
    'grep',           // 搜索代码
    'lsp_diagnostics', // 类型检查
    'bash'            // 运行测试
  ]
}

// 使用示例
async function reviewPR(prUrl: string) {
  // 1. 获取变更
  const changes = await fetchPRChanges(prUrl)
  
  // 2. 并行分析
  const analyses = await Promise.all([
    // 代码质量分析
    task({
      category: 'deep',
      description: 'Code quality analysis',
      prompt: `Review code quality for: ${changes.files.join(', ')}`
    }),
    
    // 安全分析
    task({
      category: 'deep',
      description: 'Security analysis',
      prompt: `Check for security issues in: ${changes.files.join(', ')}`
    }),
    
    // 测试分析
    task({
      category: 'quick',
      description: 'Test coverage analysis',
      prompt: `Analyze test coverage for changes`
    })
  ])
  
  // 3. 合并结果
  const report = mergeAnalyses(analyses)
  
  // 4. 输出报告
  console.log(formatReport(report))
}
```

### 10.3 实现清单

```
PR Review 助手实现清单：

Phase 1: 基础功能
- [ ] 实现 PR 变更获取
- [ ] 实现代码读取和分析
- [ ] 实现基本报告生成

Phase 2: 高级分析
- [ ] 集成 LSP 进行类型检查
- [ ] 集成 AST-grep 进行模式匹配
- [ ] 实现安全规则检查

Phase 3: 自动化
- [ ] 实现 GitHub webhook 触发
- [ ] 实现自动评论 PR
- [ ] 实现报告存储

Phase 4: 优化
- [ ] 优化分析速度（并行化）
- [ ] 添加自定义规则支持
- [ ] 添加学习机制
```

---

## 这套方案解决了什么，牺牲了什么

### 解决的问题

**从理解到落地**: 把源码理解转化为实际可用的工具

**效率提升**: 自动化重复性工作

**质量保证**: 标准化审查流程

**可复用性**: 一次开发，多次使用

### 牺牲的代价

**开发时间**: 需要投入时间开发和测试

**维护成本**: 需要持续维护

**学习成本**: 团队需要学习使用新工具

**边界情况**: 自动化可能无法处理所有场景

---

## 如果我要做自己的开发 Agent 套件，我会怎么迁移

### 场景选择指南

| 场景 | 适合的扩展层 | 复杂度 | 预期收益 |
|------|------------|--------|---------|
| PR Review | Skill + Tool | 中 | 高 |
| 代码重构 | Skill | 高 | 高 |
| Bug 排查 | Skill + MCP | 中 | 中 |
| 文档生成 | Tool | 低 | 中 |
| 测试生成 | Skill | 中 | 高 |
| 性能分析 | MCP | 高 | 中 |

### 完整实现示例

```typescript
// 1. 定义 Skill
export const codeReviewSkill: Skill = {
  name: 'code-review',
  description: 'Automated code review',
  systemPrompt: `
    You are a code reviewer.
    Review code for quality, security, and performance.
  `,
  tools: ['read', 'grep', 'lsp_diagnostics']
}

// 2. 实现核心逻辑
// src/skills/code-review/reviewer.ts
export class CodeReviewer {
  async review(files: string[]): Promise<ReviewResult> {
    const results: ReviewIssue[] = []
    
    for (const file of files) {
      // 读取文件
      const content = await readFile(file)
      
      // 静态分析
      const staticIssues = await this.staticAnalysis(file, content)
      results.push(...staticIssues)
      
      // LSP 检查
      const lspIssues = await this.lspCheck(file)
      results.push(...lspIssues)
      
      // AI 分析
      const aiIssues = await this.aiAnalysis(file, content)
      results.push(...aiIssues)
    }
    
    return { issues: results }
  }
  
  private async staticAnalysis(file: string, content: string): Promise<ReviewIssue[]> {
    const issues: ReviewIssue[] = []
    
    // 检查常见模式
    const patterns = [
      { pattern: /console\.log/, severity: 'warning', message: 'Remove console.log' },
      { pattern: /TODO|FIXME/, severity: 'info', message: 'Check TODO/FIXME' },
      { pattern: /eval\(/, severity: 'critical', message: 'Avoid eval()' }
    ]
    
    for (const { pattern, severity, message } of patterns) {
      if (pattern.test(content)) {
        issues.push({ file, severity, message, line: this.findLine(content, pattern) })
      }
    }
    
    return issues
  }
  
  private async lspCheck(file: string): Promise<ReviewIssue[]> {
    const diagnostics = await lsp_diagnostics({ filePath: file })
    return diagnostics.map(d => ({
      file,
      severity: d.severity === 'error' ? 'critical' : 'warning',
      message: d.message,
      line: d.line
    }))
  }
  
  private async aiAnalysis(file: string, content: string): Promise<ReviewIssue[]> {
    // 使用 AI 进行深度分析
    const result = await task({
      category: 'deep',
      description: `Analyze ${file}`,
      prompt: `Review this code for issues:\n${content}`
    })
    
    return result.issues || []
  }
}

// 3. 注册 Skill
export function registerCodeReviewSkill() {
  registerSkill(codeReviewSkill)
  
  // 注册命令
  registerCommand({
    name: 'review',
    description: 'Review code changes',
    handler: async (args) => {
      const files = args.length > 0 ? args : await getChangedFiles()
      const reviewer = new CodeReviewer()
      const result = await reviewer.review(files)
      console.log(formatReviewResult(result))
    }
  })
}
```

---

## 一个最小实验

### 实验: 实现一个简单的 PR 检查工具

```typescript
// simple-pr-check.ts

// 1. 获取变更文件
async function getChangedFiles(): Promise<string[]> {
  const result = await $`git diff --name-only HEAD~1`
  return result.stdout.split('\n').filter(f => f.endsWith('.ts'))
}

// 2. 检查每个文件
async function checkFile(file: string): Promise<CheckResult> {
  const content = await readFile(file)
  const issues: Issue[] = []
  
  // 检查 1: 是否有 console.log
  if (content.includes('console.log')) {
    issues.push({
      file,
      severity: 'warning',
      message: 'Found console.log, consider removing'
    })
  }
  
  // 检查 2: 是否有 TODO
  if (content.includes('TODO')) {
    issues.push({
      file,
      severity: 'info',
      message: 'Found TODO comment'
    })
  }
  
  // 检查 3: 文件是否过大
  const lines = content.split('\n').length
  if (lines > 200) {
    issues.push({
      file,
      severity: 'warning',
      message: `File has ${lines} lines, consider splitting`
    })
  }
  
  return { file, issues }
}

// 3. 主函数
async function main() {
  const files = await getChangedFiles()
  console.log(`Checking ${files.length} files...`)
  
  const results = await Promise.all(files.map(checkFile))
  
  // 输出报告
  let hasIssues = false
  for (const result of results) {
    if (result.issues.length > 0) {
      hasIssues = true
      console.log(`\n📁 ${result.file}`)
      for (const issue of result.issues) {
        const emoji = issue.severity === 'critical' ? '🔴' : 
                     issue.severity === 'warning' ? '🟡' : '🔵'
        console.log(`  ${emoji} ${issue.message}`)
      }
    }
  }
  
  if (!hasIssues) {
    console.log('✅ No issues found!')
  }
}

// 运行
main().catch(console.error)
```

**验证**:
```bash
# 在 git 仓库中运行
bun run simple-pr-check.ts

# 预期输出
Checking 3 files...

📁 src/index.ts
  🟡 Found console.log, consider removing
  🔵 Found TODO comment

📁 src/utils.ts
  🟡 File has 250 lines, consider splitting
```

---

## 总结

开发效率增强模块的实战要点：

1. **场景选择**: 从痛点出发，选择高频、高价值的场景
2. **分层设计**: 根据需求选择合适的扩展层
3. **分阶段实现**: 从简单开始，逐步迭代
4. **验证闭环**: 测试 → 试用 → 反馈 → 优化

**关键认知**: 好的开发效率工具不是"大而全"，而是"小而精、解决具体问题"。

---

## 下一步

阅读下一篇：**《为什么复杂 Agent 不能没有 verification》**，理解验收闭环的重要性。

---

**系列文章**: 《Oh My OpenAgent 源码解析：从复杂 Agent 学习到开发效率套件落地》
- ...
- 第9篇: Plugin、custom tool、MCP：能力扩展的三种路线
- 第10篇: 做一个开发效率增强模块：从源码理解到实战改造（本文）
- 第11篇: 为什么复杂 Agent 不能没有 verification
- ...（共13篇）
