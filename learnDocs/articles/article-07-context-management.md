# Context overload、压缩与会话恢复

> 本系列第7篇 | 基于项目版本: 2026-03-28 | 源码位置: `src/hooks/context-window-monitor/`, `src/hooks/preemptive-compaction/`, `src/hooks/session-recovery/`

---

## 这篇要回答的问题

1. **Context monitor 如何工作？**
2. **Preemptive compaction 何时触发？**
3. **Compaction 时保留什么、丢弃什么？**
4. **什么信息不该靠上下文硬扛？**

---

## 源码里这个问题出现在哪里

### 上下文监控: `src/hooks/context-window-monitor/`

```typescript
// src/hooks/context-window-monitor/hook.ts 简化版
export function createContextWindowMonitorHook() {
  return {
    name: "context-window-monitor",
    
    async beforeChatMessage({ context }) {
      // 计算当前上下文指标
      const metrics = calculateContextMetrics(context)
      
      // 如果超过阈值，触发压缩
      if (metrics.usageRatio > 0.8) {
        await triggerCompaction(context, metrics)
      }
      
      return { proceed: true }
    }
  }
}

// 上下文指标计算
interface ContextMetrics {
  currentTokens: number    // 当前 token 数
  maxTokens: number        // 最大 token 数
  usageRatio: number       // 使用率 (current / max)
}

function calculateContextMetrics(context: SessionContext): ContextMetrics {
  const currentTokens = estimateTokens(context.messages)
  const maxTokens = context.modelConfig.contextWindow
  
  return {
    currentTokens,
    maxTokens,
    usageRatio: currentTokens / maxTokens
  }
}

// Token 估算（简化版）
function estimateTokens(messages: Message[]): number {
  return messages.reduce((total, msg) => {
    // 英文 ~4 chars/token, 中文 ~1.5 chars/token
    const charCount = msg.content.length
    return total + Math.ceil(charCount / 3)
  }, 0)
}
```

### 预压缩: `src/hooks/preemptive-compaction/`

```typescript
// src/hooks/preemptive-compaction/hook.ts 简化版
export function createPreemptiveCompactionHook() {
  return {
    name: "preemptive-compaction",
    
    async compact(context: SessionContext, metrics: ContextMetrics) {
      // 1. 分析消息重要性
      const analyzed = analyzeMessageImportance(context.messages)
      
      // 2. 分类处理
      const { keep, compress, discard } = categorizeMessages(analyzed)
      
      // 3. 执行压缩
      const compacted = await Promise.all([
        // 保留的消息（原样）
        ...keep,
        
        // 压缩的消息（生成摘要）
        ...compress.map(m => summarizeMessage(m)),
        
        // 丢弃的消息（直接删除）
        // (discarded 被忽略)
      ])
      
      // 4. 更新上下文
      context.messages = compacted
      
      // 5. 记录压缩日志
      logCompaction({
        beforeTokens: metrics.currentTokens,
        afterTokens: estimateTokens(compacted),
        kept: keep.length,
        compressed: compress.length,
        discarded: discard.length
      })
    }
  }
}

// 消息重要性分析
function analyzeMessageImportance(messages: Message[]): AnalyzedMessage[] {
  return messages.map(msg => ({
    ...msg,
    importance: calculateImportance(msg),
    category: categorizeMessage(msg)
  }))
}

function calculateImportance(msg: Message): number {
  let score = 0
  
  // 系统消息：高优先级
  if (msg.role === 'system') score += 100
  
  // 包含关键决策：高优先级
  if (msg.content.includes('决策') || msg.content.includes('plan')) score += 80
  
  // 用户最近消息：中高优先级
  if (msg.role === 'user') score += 60
  
  // 包含代码/配置：中优先级
  if (msg.content.includes('```')) score += 40
  
  // 普通对话：低优先级
  score += 10
  
  // 时间衰减（越旧越低）
  const age = Date.now() - msg.timestamp
  score -= age / (1000 * 60 * 60)  // 每小时减1分
  
  return Math.max(0, score)
}
```

### 会话恢复: `src/hooks/session-recovery/`

```typescript
// src/hooks/session-recovery/hook.ts 简化版
export function createSessionRecoveryHook() {
  return {
    name: "session-recovery",
    
    async onError({ error, context }) {
      // 判断错误类型
      if (isRecoverableError(error)) {
        // 保存当前状态
        await saveSessionState(context)
        
        // 尝试恢复
        const recovered = await attemptRecovery(context, error)
        
        if (recovered) {
          return { recovered: true, context: recovered }
        } else {
          // 恢复失败，提示用户
          return { recovered: false, error: 'Session recovery failed' }
        }
      }
      
      return { recovered: false }
    },
    
    async onStartup() {
      // 检查是否有需要恢复的会话
      const savedState = await loadSavedSessionState()
      if (savedState) {
        return {
          shouldRecover: true,
          savedState
        }
      }
      return { shouldRecover: false }
    }
  }
}

// 可恢复的错误类型
function isRecoverableError(error: Error): boolean {
  return [
    'ContextWindowExceeded',      // 上下文超限
    'RateLimitExceeded',          // 速率限制
    'ModelUnavailable',           // 模型不可用
    'TimeoutError',               // 超时
    'ConnectionError'             // 连接错误
  ].includes(error.name)
}

// 恢复策略
async function attemptRecovery(
  context: SessionContext,
  error: Error
): Promise<SessionContext | null> {
  switch (error.name) {
    case 'ContextWindowExceeded':
      // 策略1: 压缩上下文
      return await compactAndRetry(context)
      
    case 'RateLimitExceeded':
      // 策略2: 等待后重试
      await delay(60000)
      return context
      
    case 'ModelUnavailable':
      // 策略3: 切换到 fallback 模型
      return await switchToFallbackModel(context)
      
    case 'TimeoutError':
      // 策略4: 减少请求复杂度
      return await reduceComplexity(context)
      
    default:
      return null
  }
}
```

---

## 它当前的设计方案是什么

### 7.1 Context Window Monitor

```
监控指标：
┌────────────────────────────────────────┐
│ ContextMetrics                         │
├────────────────────────────────────────┤
│ currentTokens: 45,000                  │
│ maxTokens:    100,000                  │
│ usageRatio:   0.45  ← 45%              │
│                                        │
│ Status: ✅ Healthy (ratio < 0.8)       │
└────────────────────────────────────────┘

触发阈值：
├── usageRatio > 0.8  → 警告，准备压缩
├── usageRatio > 0.9  → 强制压缩
└── usageRatio > 0.95 → 拒绝新请求
```

### 7.2 Preemptive Compaction 策略

```
压缩决策树：

消息分类：
├── 保留 (Keep)        ← 最高优先级
│   ├── 系统消息
│   ├── 关键决策
│   ├── 任务目标
│   ├── 验证标准
│   └── 最近3轮对话
│
├── 压缩 (Compress)    ← 中优先级
│   ├── 历史对话 → 生成摘要
│   ├── 搜索结果 → 保留关键发现
│   ├── 代码片段 → 保留签名
│   └── 中间思考 → 保留结论
│
└── 丢弃 (Discard)     ← 最低优先级
    ├── 过时的分析
    ├── 已执行的命令输出
    ├── 临时变量
    └── 重复信息

压缩后结构：
保留消息 (原样) + 摘要消息 (压缩) + 新消息
```

### 7.3 Session Recovery 流程

```
会话中断 → Session Recovery Hook
    │
    ├── 保存状态
    │   ├── 会话上下文
    │   ├── 待办任务
    │   ├── 决策记录
    │   └── 验证结果
    │
    ├── 恢复策略
    │   ├── ContextWindowExceeded → 压缩上下文
    │   ├── RateLimitExceeded    → 延迟重试
    │   ├── ModelUnavailable     → Fallback模型
    │   └── TimeoutError         → 降低复杂度
    │
    └── 重建上下文
        ├── 加载保存的状态
        ├── 恢复关键决策
        └── 继续执行任务
```

### 7.4 不该靠上下文硬扛的信息

| 信息类型 | 应该放在哪 | 原因 |
|---------|----------|------|
| 项目长期知识 | `AGENTS.md` | 持久化，版本控制 |
| 历史决策 | `notepad/` 或 `decisions/` | 跨会话可用 |
| 团队规范 | `rules/` 或 `AGENTS.md` | 共享给团队 |
| 验证结果 | `verification/` | 可追溯 |
| 临时分析 | Session 内 | 用完即弃 |

---

## 这套方案解决了什么，牺牲了什么

### 解决的问题

**长任务稳定**: 通过监控和压缩，长任务不会丢上下文

**自动恢复**: 会话中断后可以恢复，不用从头开始

**资源控制**: 防止 token 无限增长导致的成本问题

**容错性**: 多种恢复策略应对不同错误

### 牺牲的代价

**信息损失**: 压缩可能丢失细节，特别是中间推理过程

**恢复延迟**: 恢复会话需要时间

**复杂度**: 需要理解多级策略

**不完美恢复**: 有些信息恢复后可能丢失上下文

---

## 如果我要做自己的开发 Agent 套件，我会怎么迁移

### 压缩策略设计

```typescript
// 自定义压缩策略
interface CompactionStrategy {
  name: string
  threshold: number        // 触发阈值 (0-1)
  keepRules: KeepRule[]    // 保留规则
  compressRules: CompressRule[]  // 压缩规则
}

const myStrategy: CompactionStrategy = {
  name: 'default',
  threshold: 0.8,
  
  keepRules: [
    { type: 'system', priority: 100 },
    { type: 'decision', priority: 90 },
    { type: 'user', priority: 80, maxCount: 5 },
    { type: 'code', priority: 70 }
  ],
  
  compressRules: [
    { type: 'chat', action: 'summarize' },
    { type: 'search', action: 'extract_key_findings' },
    { type: 'thinking', action: 'keep_conclusion_only' }
  ]
}

// 压缩实现
async function compactSession(
  session: Session,
  strategy: CompactionStrategy
): Promise<Session> {
  const messages = [...session.messages]
  
  // 1. 评分
  const scored = messages.map(m => ({
    ...m,
    score: calculateScore(m, strategy.keepRules)
  }))
  
  // 2. 排序
  scored.sort((a, b) => b.score - a.score)
  
  // 3. 保留高分消息
  const keepCount = Math.min(
    messages.length * 0.6,  // 保留60%
    strategy.keepRules.find(r => r.type === 'user')?.maxCount || 5
  )
  const keep = scored.slice(0, keepCount)
  
  // 4. 压缩剩余消息
  const toCompress = scored.slice(keepCount)
  const compressed = await Promise.all(
    toCompress.map(m => compressMessage(m, strategy.compressRules))
  )
  
  return {
    ...session,
    messages: [...keep, ...compressed.filter(Boolean)]
  }
}
```

### 信息持久化方案

```
信息分层持久化：

运行时 (Session):
├── 当前任务上下文
├── 最近对话历史
└── 临时计算结果

阶段性 (Cross-Session):
├── notepad/          # 学习笔记
├── decisions/        # 决策记录
├── issues/           # 问题跟踪
└── verification/     # 验证结果

长期知识 (Project):
├── AGENTS.md         # 项目规则
├── README.md         # 项目说明
└── docs/             # 详细文档
```

---

## 一个最小实验

### 实验1: 分析阈值设置

```bash
# 查看 context-window-monitor 的默认阈值
grep -r "threshold" src/hooks/context-window-monitor/
grep -r "usageRatio" src/hooks/context-window-monitor/

# 查看支持的配置项
cat src/config/schema/context-window-monitor.ts
```

**思考**:
- 0.8 的阈值是否合理？
- 不同模型的上下文窗口不同，如何适配？
- 阈值调低会怎样？调高会怎样？

### 实验2: 理解 Compaction 保留逻辑

```typescript
// 测试压缩逻辑
import { compactSession } from './compaction'

const testMessages = [
  { role: 'system', content: 'You are a helpful assistant' },
  { role: 'user', content: '帮我写个函数' },
  { role: 'assistant', content: '好的，让我分析需求...' },
  // ... 更多消息
]

const compacted = await compactSession(
  { messages: testMessages },
  { threshold: 0.8, keepRules: [...], compressRules: [...] }
)

console.log('Before:', testMessages.length)
console.log('After:', compacted.messages.length)
console.log('Kept:', compacted.messages.filter(m => !m.compressed).length)
console.log('Compressed:', compacted.messages.filter(m => m.compressed).length)
```

---

## 总结

Context 管理的核心机制：

1. **监控**: Context Window Monitor 实时监控 token 使用率
2. **压缩**: Preemptive Compaction 在达到阈值前主动压缩
3. **恢复**: Session Recovery 提供多种恢复策略
4. **分层**: 不同信息放在合适的层级，不该硬扛

**关键认知**: Context 管理不是"能存多少"，而是"什么该存、存多久、怎么恢复"。

---

## 下一步

阅读下一篇：**《复杂 Agent 的记忆体系：运行时、沉淀层、长期知识》**，深入理解三层记忆模型。

---

**系列文章**: 《Oh My OpenAgent 源码解析：从复杂 Agent 学习到开发效率套件落地》
- ...
- 第6篇: AGENTS.md、rules 与目录上下文注入机制
- 第7篇: Context overload、压缩与会话恢复（本文）
- 第8篇: 复杂 Agent 的记忆体系：运行时、沉淀层、长期知识
- ...（共13篇）
