# 权限、回退与恢复，Agent 系统怎么才能日常可用

> 本系列第12篇 | 基于项目版本: 2026-03-28 | 源码位置: `src/plugin-config.ts`, `src/hooks/model-fallback/`, `src/hooks/session-recovery/`, `src/hooks/write-existing-file-guard/`

---

## 这篇要回答的问题

1. **Permissions 如何配置？**
2. **Tool restriction 如何生效？**
3. **Fallback 何时触发？**
4. **Session recovery 如何工作？**

---

## 源码里这个问题出现在哪里

### 权限配置: `src/plugin-config.ts`

```typescript
// src/plugin-config.ts 简化版
export interface OhMyOpenCodeConfig {
  // 全局权限配置
  permissions: {
    bash: PermissionLevel      // 'allow' | 'ask' | 'deny'
    edit: PermissionLevel
    write: PermissionLevel
    webfetch: PermissionLevel
  }
  
  // Agent 级权限覆盖
  agent_permissions: {
    [agentName: string]: {
      bash?: PermissionLevel
      edit?: PermissionLevel
      write?: PermissionLevel
    }
  }
  
  // 工具级权限
  tool_permissions: {
    [toolName: string]: PermissionLevel
  }
}

type PermissionLevel = 'allow' | 'ask' | 'deny'

// 权限级别
const PERMISSION_LEVELS = {
  allow: 2,  // 自动执行
  ask: 1,    // 需要确认
  deny: 0    // 禁止执行
}

// 权限解析
function resolvePermission(
  global: PermissionLevel,
  agent?: PermissionLevel,
  tool?: PermissionLevel
): PermissionLevel {
  // 优先级: tool > agent > global
  if (tool) return tool
  if (agent) return agent
  return global
}

// 配置示例 (oh-my-opencode.jsonc)
const EXAMPLE_CONFIG = {
  permissions: {
    bash: "ask",      // 全局：bash 需要确认
    edit: "allow",    // 全局：edit 自动执行
    write: "deny"     // 全局：write 禁止
  },
  
  agent_permissions: {
    prometheus: {
      bash: "deny",   // Prometheus 不能执行 bash
      write: "deny"   // Prometheus 不能写文件
    },
    hephaestus: {
      bash: "ask",    // Hephaestus 执行 bash 需要确认
      write: "allow"  // Hephaestus 可以写文件
    },
    sisyphus: {
      bash: "allow",  // Sisyphus 自动执行 bash
      write: "allow"  // Sisyphus 可以写文件
    }
  }
}
```

### 写文件守卫: `src/hooks/write-existing-file-guard/`

```typescript
// src/hooks/write-existing-file-guard/hook.ts
export function createWriteExistingFileGuardHook() {
  return {
    name: "write-existing-file-guard",
    
    async beforeToolExecute({ tool, args, context }) {
      if (tool !== 'write') return { proceed: true }
      
      const { path: filePath } = args
      
      // 1. 检查文件是否存在
      const exists = await fileExists(filePath)
      if (!exists) return { proceed: true }
      
      // 2. 检查是否有 read-before-write 权限
      const hasRead = await context.hasReadFile(filePath)
      
      if (!hasRead) {
        return {
          proceed: false,
          error: `
            File ${filePath} already exists and you haven't read it.
            
            This guard prevents accidental overwrites.
            
            Options:
            1. Read the file first: read("${filePath}")
            2. Use edit tool for modifications
            3. Explicitly delete and rewrite if intended
          `
        }
      }
      
      // 3. 检查文件是否被外部修改
      const fileHash = await getFileHash(filePath)
      const cachedHash = context.getCachedHash(filePath)
      
      if (cachedHash && fileHash !== cachedHash) {
        return {
          proceed: false,
          error: `
            File ${filePath} has been modified externally.
            
            Please re-read the file to get the latest version.
          `
        }
      }
      
      return { proceed: true }
    }
  }
}
```

### 模型降级: `src/hooks/model-fallback/`

```typescript
// src/hooks/model-fallback/hook.ts
export function createModelFallbackHook() {
  return {
    name: "model-fallback",
    
    async onModelError({ error, context, attempt }) {
      // 判断错误类型
      if (!isRetryableError(error)) {
        return { retry: false }
      }
      
      // 获取 fallback chain
      const fallbackChain = context.config.fallback_models || [
        'claude-opus-4',
        'gpt-4-turbo',
        'claude-sonnet-4',
        'kimi-k2.5'
      ]
      
      // 尝试下一个模型
      const nextModel = fallbackChain[attempt]
      if (!nextModel) {
        return { 
          retry: false, 
          error: 'All fallback models exhausted' 
        }
      }
      
      // 降级到下一个模型
      return {
        retry: true,
        model: nextModel,
        message: `Falling back to ${nextModel} due to: ${error.message}`
      }
    }
  }
}

// 可重试错误类型
function isRetryableError(error: Error): boolean {
  return [
    'RateLimitError',
    'ModelUnavailableError', 
    'TimeoutError',
    'ContextWindowExceededError',
    'TemporaryError'
  ].includes(error.name)
}

// Fallback Chain 示例
const FALLBACK_CHAIN = {
  'claude-opus-4': ['gpt-4-turbo', 'claude-sonnet-4', 'kimi-k2.5'],
  'gpt-4-turbo': ['claude-opus-4', 'claude-sonnet-4'],
  'deep-category': ['claude-opus-4', 'kimi-k2.5', 'gpt-4-turbo'],
  'quick-category': ['claude-haiku', 'gpt-3.5-turbo', 'kimi-k2.5']
}
```

### 会话恢复: `src/hooks/session-recovery/`

```typescript
// src/hooks/session-recovery/hook.ts
export function createSessionRecoveryHook() {
  return {
    name: "session-recovery",
    
    // 错误时触发
    async onError({ error, context, state }) {
      // 保存当前状态
      await saveSessionState({
        sessionId: context.sessionId,
        currentTask: context.currentTask,
        todoList: context.todoList,
        decisions: context.decisions,
        timestamp: Date.now(),
        error: {
          name: error.name,
          message: error.message,
          stack: error.stack
        }
      })
      
      // 尝试恢复策略
      const recovery = await attemptRecovery(error, context)
      
      if (recovery.success) {
        return {
          recovered: true,
          context: recovery.context,
          message: 'Session recovered'
        }
      }
      
      return {
        recovered: false,
        message: `Recovery failed: ${recovery.reason}`
      }
    },
    
    // 启动时检查
    async onStartup() {
      const savedState = await loadSessionState()
      
      if (savedState && savedState.error) {
        return {
          hasRecovery: true,
          state: savedState,
          options: [
            { label: 'Resume', action: 'resume' },
            { label: 'Restart', action: 'restart' },
            { label: 'Inspect', action: 'inspect' }
          ]
        }
      }
      
      return { hasRecovery: false }
    }
  }
}

// 恢复策略
async function attemptRecovery(
  error: Error, 
  context: SessionContext
): Promise<RecoveryResult> {
  switch (error.name) {
    case 'ContextWindowExceededError':
      // 策略1: 压缩上下文
      return await recoverByCompaction(context)
      
    case 'RateLimitError':
      // 策略2: 等待后重试
      await delay(60000)
      return { success: true, context }
      
    case 'ModelUnavailableError':
      // 策略3: 切换模型
      return await recoverByModelSwitch(context)
      
    case 'TimeoutError':
      // 策略4: 降低任务复杂度
      return await recoverBySimplification(context)
      
    case 'ToolExecutionError':
      // 策略5: 跳过失败工具
      return await recoverBySkipTool(context, error)
      
    default:
      return { success: false, reason: 'Unknown error type' }
  }
}
```

---

## 它当前的设计方案是什么

### 12.1 权限级别

```
权限三级别：

┌────────────────────────────────────────┐
│ allow (自动执行)                       │
│                                         │
│ • 无需确认                             │
│ • 适合低风险操作                       │
│ • 示例: read, glob, grep               │
└────────────────────────────────────────┘

┌────────────────────────────────────────┐
│ ask (需要确认)                         │
│                                         │
│ • 执行前询问用户                       │
│ • 适合中等风险操作                     │
│ • 示例: bash, write, edit              │
└────────────────────────────────────────┘

┌────────────────────────────────────────┐
│ deny (禁止执行)                        │
│                                         │
│ • 完全禁止                             │
│ • 适合高风险操作                       │
│ • 示例: rm -rf /, 写入系统目录         │
└────────────────────────────────────────┘
```

### 12.2 Agent 级权限

```
不同 Agent 不同权限：

┌────────────────────────────────────────┐
│ Oracle (只读顾问)                      │
│                                         │
│ read: allow                            │
│ write: deny                            │
│ bash: deny                             │
│                                         │
│ 只能分析，不能修改                     │
└────────────────────────────────────────┘

┌────────────────────────────────────────┐
│ Prometheus (规划者)                    │
│                                         │
│ read: allow                            │
│ write: deny                            │
│ bash: deny                             │
│                                         │
│ 只能规划，不能执行                     │
└────────────────────────────────────────┘

┌────────────────────────────────────────┐
│ Hephaestus (实现者)                    │
│                                         │
│ read: allow                            │
│ write: allow                           │
│ bash: ask                              │
│                                         │
│ 可以写代码，bash 需确认                │
└────────────────────────────────────────┘

┌────────────────────────────────────────┐
│ Sisyphus (编排者)                      │
│                                         │
│ read: allow                            │
│ write: allow                           │
│ bash: allow                            │
│                                         │
│ 全部权限，但需要谨慎                   │
└────────────────────────────────────────┘
```

### 12.3 Fallback 机制

```
模型调用失败 → Fallback Hook
    │
    ├── 错误类型判断
    │   ├── 可重试？→ 等待/降级
    │   └── 不可重试？→ 报错
    │
    └── 降级策略
        ├── RateLimit → 等待60秒
        ├── ModelUnavailable → 切换模型
        ├── ContextWindow → 压缩上下文
        └── Timeout → 降低复杂度

Fallback Chain:
Primary Model → Fallback 1 → Fallback 2 → Fallback 3
(claude-opus)   (gpt-4)      (kimi)       (sonnet)
     │              │            │            │
     └──────────────┴────────────┴────────────┘
                    ↓
              全部失败 → 报错
```

### 12.4 Recovery 流程

```
会话中断 → Recovery Hook
    │
    ├── 保存状态
    │   ├── 会话上下文
    │   ├── 当前任务
    │   ├── 待办列表
    │   └── 决策记录
    │
    ├── 恢复策略
    │   ├── ContextWindowExceeded → 压缩
    │   ├── RateLimit → 等待
    │   ├── ModelUnavailable → 切换
    │   └── Timeout → 简化
    │
    ├── 重建上下文
    │   ├── 加载保存的状态
    │   ├── 恢复关键决策
    │   └── 继续任务
    │
    └── 用户选择
        ├── Resume → 继续
        ├── Restart → 重新开始
        └── Inspect → 查看状态
```

---

## 这套方案解决了什么，牺牲了什么

### 解决的问题

**系统可用性**: 错误时自动恢复，不会直接崩溃

**安全可控**: 权限分级，防止误操作

**降级能力**: 模型不可用时自动切换

**数据不丢失**: 会话中断后可以恢复

### 牺牲的代价

**配置复杂度**: 需要理解和配置多层权限

**恢复延迟**: 恢复需要时间

**权限限制**: 过度限制可能影响效率

**维护成本**: 需要维护 fallback chain

---

## 如果我要做自己的开发 Agent 套件，我会怎么迁移

### 权限配置策略

```typescript
// 权限配置生成器
class PermissionConfigBuilder {
  private config: PermissionConfig = {
    permissions: {},
    agent_permissions: {},
    tool_permissions: {}
  }
  
  // 全局权限
  global(permissions: Partial<GlobalPermissions>) {
    this.config.permissions = { ...this.config.permissions, ...permissions }
    return this
  }
  
  // Agent 权限
  agent(agentName: string, permissions: Partial<AgentPermissions>) {
    this.config.agent_permissions[agentName] = {
      ...this.config.agent_permissions[agentName],
      ...permissions
    }
    return this
  }
  
  // 工具权限
  tool(toolName: string, level: PermissionLevel) {
    this.config.tool_permissions[toolName] = level
    return this
  }
  
  build(): PermissionConfig {
    return this.config
  }
}

// 使用
const config = new PermissionConfigBuilder()
  // 全局：保守设置
  .global({ bash: 'ask', write: 'ask', edit: 'allow' })
  
  // 规划者：只读
  .agent('planner', { bash: 'deny', write: 'deny', edit: 'deny' })
  
  // 执行者：可写
  .agent('executor', { bash: 'ask', write: 'allow', edit: 'allow' })
  
  // 危险工具：禁止
  .tool('dangerous_bash', 'deny')
  
  .build()
```

### 降级策略设计

```typescript
// Fallback 策略
interface FallbackStrategy {
  name: string
  condition: (error: Error) => boolean
  action: (context: SessionContext) => Promise<RecoveryResult>
}

const FALLBACK_STRATEGIES: FallbackStrategy[] = [
  {
    name: 'rate-limit',
    condition: e => e.name === 'RateLimitError',
    action: async () => {
      await delay(60000)
      return { success: true }
    }
  },
  {
    name: 'model-unavailable',
    condition: e => e.name === 'ModelUnavailableError',
    action: async (ctx) => {
      const next = ctx.config.fallback_models[ctx.currentAttempt]
      if (next) {
        ctx.switchModel(next)
        return { success: true }
      }
      return { success: false, reason: 'No fallback available' }
    }
  },
  {
    name: 'context-overflow',
    condition: e => e.name === 'ContextWindowExceededError',
    action: async (ctx) => {
      await ctx.compact()
      return { success: true }
    }
  }
]

// 应用策略
async function applyFallback(
  error: Error, 
  context: SessionContext
): Promise<RecoveryResult> {
  for (const strategy of FALLBACK_STRATEGIES) {
    if (strategy.condition(error)) {
      console.log(`Applying fallback: ${strategy.name}`)
      return await strategy.action(context)
    }
  }
  return { success: false, reason: 'No matching strategy' }
}
```

---

## 一个最小实验

### 实验1: 配置不同的权限级别

```jsonc
// my-permissions.jsonc
{
  // 全局权限
  permissions: {
    bash: "ask",      // 需要确认
    write: "ask",     // 需要确认
    edit: "allow"     // 自动执行
  },
  
  // Agent 权限
  agent_permissions: {
    // 只读 Agent
    reader: {
      bash: "deny",
      write: "deny",
      edit: "deny"
    },
    
    // 写作者 Agent
    writer: {
      bash: "ask",
      write: "allow",
      edit: "allow"
    }
  },
  
  // 工具权限
  tool_permissions: {
    "rm": "deny",           // 禁止删除
    "sudo": "deny",         // 禁止 sudo
    "npm_install": "ask"    // 安装需确认
  }
}
```

### 实验2: 测试 Fallback 触发

```typescript
// test-fallback.ts

// 模拟模型不可用
async function testModelFallback() {
  const models = ['primary', 'fallback1', 'fallback2']
  let attempt = 0
  
  while (attempt < models.length) {
    try {
      console.log(`Trying model: ${models[attempt]}`)
      await callModel(models[attempt])
      console.log('Success!')
      break
    } catch (error) {
      console.log(`Failed: ${error.message}`)
      attempt++
      
      if (attempt < models.length) {
        console.log(`Falling back to: ${models[attempt]}`)
        await delay(1000) // 模拟等待
      } else {
        console.log('All models exhausted')
        throw error
      }
    }
  }
}

// 模拟调用
async function callModel(model: string): Promise<void> {
  if (model === 'primary') {
    throw new Error('Model unavailable')
  }
  // fallback 成功
}

// 延迟
function delay(ms: number): Promise<void> {
  return new Promise(resolve => setTimeout(resolve, ms))
}

// 运行
testModelFallback()
```

---

## 总结

权限、回退与恢复的核心机制：

1. **权限三级**: allow / ask / deny，全局 → Agent → 工具
2. **Agent 隔离**: 不同角色不同权限，防止越权
3. **Fallback**: 模型不可用时自动降级
4. **Recovery**: 会话中断后可恢复

**关键认知**: 能日常使用的 Agent 系统，必须有完善的权限和恢复机制。

---

## 下一步

阅读下一篇：**《如何搭建自己的领域插件工程架构》**，完整工程化指南。

---

**系列文章**: 《Oh My OpenAgent 源码解析：从复杂 Agent 学习到开发效率套件落地》
- ...
- 第11篇: 为什么复杂 Agent 不能没有 verification
- 第12篇: 权限、回退与恢复，Agent 系统怎么才能日常可用（本文）
- 第13篇: 如何搭建自己的领域插件工程架构
