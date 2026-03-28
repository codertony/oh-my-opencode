# 复杂 Agent 的记忆体系：运行时、沉淀层、长期知识

> 本系列第8篇 | 基于项目版本: 2026-03-28 | 源码位置: `src/hooks/handoff/`, `src/hooks/session-recovery/`, `src/features/builtin-skills/`

---

## 这篇要回答的问题

1. **哪些信息只应留在会话里？**
2. **哪些应该写入 notepad / verification / decisions？**
3. **哪些应该回写项目知识？**
4. **会话中断后如何恢复？**

---

## 源码里这个问题出现在哪里

### 会话交接: `src/hooks/handoff/`

```typescript
// src/hooks/handoff/hook.ts 简化版
export function createHandoffHook() {
  return {
    name: "handoff",
    
    async onSessionEnd({ context, reason }) {
      // 提取会话中的关键信息
      const handoffData = extractHandoffData(context)
      
      // 保存到阶段性沉淀
      await saveToNotepad({
        type: 'session_summary',
        data: handoffData,
        timestamp: Date.now()
      })
      
      // 如果有重要决策，保存到 decisions
      if (handoffData.decisions.length > 0) {
        await saveToDecisions(handoffData.decisions)
      }
      
      // 如果有发现的问题，保存到 issues
      if (handoffData.issues.length > 0) {
        await saveToIssues(handoffData.issues)
      }
    },
    
    async onSessionStart({ sessionId }) {
      // 检查是否有 handoff 数据
      const previousHandoff = await loadHandoffData(sessionId)
      
      if (previousHandoff) {
        // 恢复关键上下文
        return {
          context: {
            previousDecisions: previousHandoff.decisions,
            pendingTasks: previousHandoff.pendingTasks,
            keyLearnings: previousHandoff.learnings
          }
        }
      }
    }
  }
}

// 提取交接数据
interface HandoffData {
  sessionContext: SessionState
  pendingTasks: Task[]
  decisions: Decision[]
  issues: Issue[]
  learnings: Learning[]
  verificationResults: VerificationResult[]
}

function extractHandoffData(context: SessionContext): HandoffData {
  return {
    sessionContext: extractSessionState(context),
    pendingTasks: extractPendingTasks(context),
    decisions: extractDecisions(context),
    issues: extractIssues(context),
    learnings: extractLearnings(context),
    verificationResults: extractVerificationResults(context)
  }
}
```

### Skills 系统: `src/features/builtin-skills/`

```typescript
// src/features/builtin-skills/skill.ts 简化版
export interface Skill {
  name: string
  description: string
  systemPrompt: string
  mcpServers?: MCPServerConfig[]
  tools?: string[]
}

// Skill 作为长期知识的载体
export const skills: Record<string, Skill> = {
  'playwright': {
    name: 'playwright',
    description: 'Browser automation via Playwright',
    systemPrompt: `
      You are a browser automation expert.
      Use Playwright MCP for all browser operations.
      
      Best practices:
      - Always wait for page load
      - Use specific selectors
      - Handle errors gracefully
    `,
    mcpServers: [{
      name: 'playwright',
      command: 'npx @anthropic-ai/playwright-mcp-server'
    }]
  },
  
  'git-master': {
    name: 'git-master',
    description: 'Git operations and atomic commits',
    systemPrompt: `
      You are a Git expert.
      
      Rules:
      - Use atomic commits
      - Write clear commit messages
      - Rebase for clean history
      
      Commands:
      - /commit: Create atomic commit
      - /rebase: Interactive rebase
    `,
    tools: ['bash', 'read', 'edit']
  }
}

// Skill 知识回写
export async function syncSkillToAgentsMd(
  skill: Skill,
  projectRoot: string
) {
  const agentsMdPath = path.join(projectRoot, 'AGENTS.md')
  
  // 读取现有内容
  const existing = await readFile(agentsMdPath)
  
  // 添加 Skill 知识
  const skillSection = `
## Skill: ${skill.name}

${skill.description}

### Guidelines
${skill.systemPrompt}

### Tools
${skill.tools?.join(', ') || 'None'}
`
  
  // 写入（如果不存在）
  if (!existing.includes(`## Skill: ${skill.name}`)) {
    await appendFile(agentsMdPath, skillSection)
  }
}
```

### 会话恢复: `src/hooks/session-recovery/`

```typescript
// src/hooks/session-recovery/hook.ts
export function createSessionRecoveryHook() {
  return {
    name: "session-recovery",
    
    async saveState(context: SessionContext) {
      const state: SessionState = {
        id: context.sessionId,
        timestamp: Date.now(),
        
        // 运行时记忆
        currentTask: context.currentTask,
        messageHistory: context.messages.slice(-10), // 最近10条
        
        // 阶段性沉淀引用
        notepadRef: context.notepad?.lastEntry,
        decisionsRef: context.decisions?.map(d => d.id),
        
        // 长期知识引用
        agentsMdVersion: context.config?.agentsMdHash,
        skillsLoaded: context.skills?.map(s => s.name)
      }
      
      await saveToSessionStore(state)
    },
    
    async recoverState(sessionId: string): Promise<SessionContext | null> {
      // 1. 加载保存的状态
      const saved = await loadFromSessionStore(sessionId)
      if (!saved) return null
      
      // 2. 恢复阶段性沉淀
      const notepad = saved.notepadRef 
        ? await loadNotepadEntry(saved.notepadRef)
        : null
        
      const decisions = saved.decisionsRef
        ? await Promise.all(saved.decisionsRef.map(loadDecision))
        : []
      
      // 3. 检查长期知识版本
      const currentAgentsMd = await loadProjectAgentsMd()
      if (hash(currentAgentsMd) !== saved.agentsMdVersion) {
        console.warn('AGENTS.md has changed since last session')
      }
      
      // 4. 重建上下文
      return {
        sessionId: saved.id,
        currentTask: saved.currentTask,
        messages: saved.messageHistory,
        notepad,
        decisions,
        // ...
      }
    }
  }
}
```

---

## 它当前的设计方案是什么

### 8.1 三层记忆模型

```
┌─────────────────────────────────────────────────────┐
│ Layer 3: 项目长期知识 (Knowledge)                    │
│                                                     │
│ • AGENTS.md          - 项目规则和架构               │
│ • Skills             - 领域特定能力                 │
│ • Rules              - 代码规范和约定               │
│                                                     │
│ 生命周期: 永久                                      │
│ 存储: 文件系统 (Git)                                │
│ 访问: 每次会话加载                                  │
└─────────────────────────────────────────────────────┘
                          ↑ 回写 (异步)
┌─────────────────────────────────────────────────────┐
│ Layer 2: 阶段性沉淀 (Learnings)                      │
│                                                     │
│ • notepad/           - 学习笔记                     │
│ • decisions/         - 决策记录                     │
│ • issues/            - 问题跟踪                     │
│ • verification/      - 验证结果                     │
│                                                     │
│ 生命周期: 跨会话                                    │
│ 存储: 文件系统                                      │
│ 访问: Session handoff                               │
└─────────────────────────────────────────────────────┘
                          ↑ 提取 (Session end)
┌─────────────────────────────────────────────────────┐
│ Layer 1: 运行时记忆 (Session)                        │
│                                                     │
│ • 当前任务上下文                                    │
│ • 对话历史 (最近N条)                                │
│ • 临时变量                                          │
│ • 中间计算结果                                      │
│                                                     │
│ 生命周期: 当前会话                                  │
│ 存储: 内存                                          │
│ 访问: 实时                                          │
└─────────────────────────────────────────────────────┘
```

### 8.2 信息流转

```
任务执行 → 生成 learnings
    │
    ├── 即时洞察 → Session 内存 (临时)
    │
    ├── 重要决策 → decisions/ (沉淀)
    │
    ├── 问题发现 → issues/ (沉淀)
    │
    ├── 学习总结 → notepad/ (沉淀)
    │
    └── 验证结果 → verification/ (沉淀)
                   │
                   ↓ (定期回写)
                   │
            AGENTS.md (长期知识)
            Skills (长期能力)
```

### 8.3 Session Handoff 数据结构

```typescript
// 会话交接数据
interface HandoffData {
  // 运行时记忆（摘要）
  sessionContext: {
    currentGoal: string
    progress: number
    blockers: string[]
  }
  
  // 待办任务（引用）
  pendingTasks: {
    id: string
    description: string
    status: 'pending' | 'in_progress'
  }[]
  
  // 关键决策（完整）
  decisions: {
    id: string
    timestamp: number
    context: string
    decision: string
    rationale: string
    alternatives: string[]
  }[]
  
  // 发现的问题（完整）
  issues: {
    id: string
    severity: 'critical' | 'high' | 'medium' | 'low'
    description: string
    impact: string
    proposedSolution?: string
  }[]
  
  // 学习收获（完整）
  learnings: {
    id: string
    category: 'technical' | 'process' | 'domain'
    insight: string
    evidence: string
    applicability: string
  }[]
}
```

### 8.4 记忆边界

| 类型 | 存储位置 | 生命周期 | 访问方式 | 示例 |
|------|---------|---------|---------|------|
| 运行时 | 内存 | 会话结束 | 实时访问 | 当前变量、临时计算 |
| 沉淀 | `notepad/`, `decisions/` | 跨会话 | Handoff | 决策记录、问题跟踪 |
| 长期 | `AGENTS.md`, Skills | 永久 | 每次加载 | 项目规则、领域知识 |

---

## 这套方案解决了什么，牺牲了什么

### 解决的问题

**知识不丢失**: 关键决策和学习被持久化，不会因会话结束而丢失

**上下文恢复**: 会话中断后可以从上次状态恢复

**知识积累**: 长期知识库不断增长，Agent 越来越"懂"项目

**团队协作**: 沉淀的知识可以在团队中共享

### 牺牲的代价

**维护复杂度**: 需要管理多层存储

**存储成本**: 沉淀层和长期知识占用磁盘空间

**一致性挑战**: 长期知识可能被多人同时修改

**回写延迟**: 从沉淀到长期知识需要人工确认

---

## 如果我要做自己的开发 Agent 套件，我会怎么迁移

### 记忆分层设计

```typescript
// 记忆管理器
class MemoryManager {
  // Layer 1: 运行时
  private sessionMemory: Map<string, any> = new Map()
  
  // Layer 2: 阶段性
  private notepadPath: string
  private decisionsPath: string
  private issuesPath: string
  
  // Layer 3: 长期
  private agentsMdPath: string
  private skillsPath: string
  
  // 写入运行时
  setSession(key: string, value: any) {
    this.sessionMemory.set(key, value)
  }
  
  getSession(key: string): any {
    return this.sessionMemory.get(key)
  }
  
  // 写入沉淀
  async addDecision(decision: Decision) {
    await appendToFile(
      path.join(this.decisionsPath, `${Date.now()}.md`),
      formatDecision(decision)
    )
  }
  
  async addLearning(learning: Learning) {
    await appendToFile(
      path.join(this.notepadPath, 'learnings.md'),
      formatLearning(learning)
    )
  }
  
  // 回写长期
  async syncToAgentsMd(insight: string) {
    const agentsMd = await readFile(this.agentsMdPath)
    if (!agentsMd.includes(insight)) {
      await appendFile(this.agentsMdPath, `\n## Learning\n${insight}\n`)
    }
  }
}
```

### 知识沉淀流程

```
任务执行中：
    │
    ├── 发现重要决策？
    │   └── 立即写入 decisions/
    │
    ├── 发现新问题？
    │   └── 立即写入 issues/
    │
    └── 获得新洞察？
        └── 暂存 Session 内存
            │
Session 结束：
    │
    ├── 提取所有学习 → notepad/
    │
    └── Handoff 数据保存
        │
定期（每周）：
    │
    ├── 审核 notepad/
    │
    ├── 有价值的？→ 回写 AGENTS.md
    │
    └── 形成 Skill？→ 创建 Skill 文件
```

---

## 一个最小实验

### 实验1: 分析 Handoff Hook 实现

```bash
# 查看 handoff hook 实现
cat src/hooks/handoff/hook.ts

# 查看数据结构定义
grep -r "HandoffData" src/

# 查看保存位置
ls -la ~/.config/opencode/handoff/
```

**观察重点**:
- 哪些信息被保存？
- 保存的格式是什么？
- 如何加载和恢复？

### 实验2: 设计简单的知识沉淀流程

```typescript
// 简单版知识沉淀
class SimpleKnowledgeSystem {
  // Session 中捕获决策
  captureDecision(context: string, decision: string) {
    const entry = {
      timestamp: new Date().toISOString(),
      context,
      decision
    }
    
    // 保存到 decisions/
    fs.appendFileSync(
      'decisions/decisions.jsonl',
      JSON.stringify(entry) + '\n'
    )
  }
  
  // 定期回顾
  async reviewDecisions() {
    const decisions = await readJsonl('decisions/decisions.jsonl')
    
    // 有价值的决策？
    const valuable = decisions.filter(d => this.isValuable(d))
    
    // 回写到 AGENTS.md
    for (const d of valuable) {
      await appendToAgentsMd(`
## Decision: ${d.decision}
Context: ${d.context}
Date: ${d.timestamp}
`)
    }
  }
  
  private isValuable(decision: Decision): boolean {
    // 简单规则：涉及架构、设计模式的决策
    return /架构|设计模式|重构|API/.test(decision.context)
  }
}
```

---

## 总结

复杂 Agent 的记忆体系核心：

1. **三层记忆**: 运行时 → 沉淀 → 长期知识
2. **信息流转**: Session 生成 learnings → 沉淀保存 → 定期回写长期知识
3. **Session Handoff**: 会话交接时提取关键信息
4. **知识积累**: 长期知识库不断增长

**关键认知**: Agent 记忆不是"记住一切"，而是"分层存储、按需提取、定期沉淀"。

---

## 下一步

阅读下一篇：**《Plugin、custom tool、MCP：能力扩展的三种路线》**，理解如何扩展 Agent 能力。

---

**系列文章**: 《Oh My OpenAgent 源码解析：从复杂 Agent 学习到开发效率套件落地》
- ...
- 第7篇: Context overload、压缩与会话恢复
- 第8篇: 复杂 Agent 的记忆体系：运行时、沉淀层、长期知识（本文）
- 第9篇: Plugin、custom tool、MCP：能力扩展的三种路线
- ...（共13篇）
