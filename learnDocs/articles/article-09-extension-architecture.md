# Plugin、custom tool、MCP：能力扩展的三种路线

> 本系列第9篇 | 基于项目版本: 2026-03-28 | 源码位置: `src/plugin/tool-registry.ts`, `src/hooks/`, `src/mcp/`, `src/features/builtin-skills/`

---

## 这篇要回答的问题

1. **规则层 vs plugin 层 vs tool 层 vs MCP 层的区别？**
2. **哪类需求适合放哪一层？**

---

## 源码里这个问题出现在哪里

### 四层扩展架构

```
用户请求
    │
    ├──→ Rules Layer (AGENTS.md)        ← 静态规则
    │       • 项目规范
    │       • 代码风格
    │       • 架构约定
    │
    ├──→ Plugin Layer (Hooks)           ← 生命周期拦截
    │       • 48个生命周期 hooks
    │       • 行为扩展
    │       • 权限控制
    │
    ├──→ Tool Layer (Tools)             ← 原子能力
    │       • 26个核心工具
    │       • 自定义工具
    │       • Skill 嵌入
    │
    └──→ MCP Layer (MCP)                ← 外部服务
            • Built-in MCP
            • Claude Code MCP
            • Skill-embedded MCP
```

### 工具注册: `src/plugin/tool-registry.ts`

```typescript
// src/plugin/tool-registry.ts 简化版
export class ToolRegistry {
  private tools: Map<string, Tool> = new Map()
  
  register(tool: Tool) {
    this.tools.set(tool.name, tool)
  }
  
  getAll(): Tool[] {
    return Array.from(this.tools.values())
  }
  
  get(name: string): Tool | undefined {
    return this.tools.get(name)
  }
}

// 26个核心工具
const CORE_TOOLS: Tool[] = [
  // 文件操作
  createReadTool(),
  createWriteTool(),
  createEditTool(),
  createGlobTool(),
  createGrepTool(),
  
  // 任务执行
  createTaskTool(),        // 委派任务
  createBashTool(),
  createInteractiveBashTool(),
  
  // 会话管理
  createSessionListTool(),
  createSessionReadTool(),
  createBackgroundOutputTool(),
  
  // LSP 工具
  createLspDiagnosticsTool(),
  createLspRenameTool(),
  createLspFindReferencesTool(),
  
  // AST 工具
  createAstGrepSearchTool(),
  createAstGrepReplaceTool(),
  
  // 网络工具
  createWebFetchTool(),
  createWebSearchTool(),
  createContext7Tool(),
  createGrepAppTool(),
  
  // 其他
  createSkillTool(),
  createTodowriteTool(),
  createQuestionTool()
]

// 初始化时注册
export function createTools(deps: Dependencies): ToolRegistry {
  const registry = new ToolRegistry()
  
  // 注册核心工具
  CORE_TOOLS.forEach(tool => registry.register(tool))
  
  // 注册自定义工具
  const customTools = loadCustomTools(deps)
  customTools.forEach(tool => registry.register(tool))
  
  return registry
}
```

### 48个 Hooks: `src/hooks/`

```typescript
// Hooks 分类
const HOOK_CATEGORIES = {
  // Session Hooks (23个) - 会话生命周期
  SESSION: [
    'chat.message',         // 消息处理前
    'chat.params',          // 参数调整
    'chat.headers',         // 请求头注入
    'chat.response',        // 响应处理
    'session.created',      // 会话创建
    'session.ended',        // 会话结束
    'session.error',        // 会话错误
    // ... 更多
  ],
  
  // Tool-Guard Hooks (12个) - 工具执行拦截
  TOOL_GUARD: [
    'write-existing-file-guard',  // 写文件守卫
    'file-guard',                 // 文件操作守卫
    'tool-output-truncator',      // 输出截断
    'bash-guard',                 // Bash 命令守卫
    // ... 更多
  ],
  
  // Transform Hooks (4个) - 数据转换
  TRANSFORM: [
    'messages-transform',         // 消息转换
    'tool-result-transform',      // 工具结果转换
    // ... 更多
  ],
  
  // Continuation Hooks (7个) - 会话延续
  CONTINUATION: [
    'todo-continuation',          // Todo 延续
    'session-recovery',           // 会话恢复
    'ralph-loop',                 // Ralph 循环
    // ... 更多
  ],
  
  // Skill Hooks (2个) - Skill 系统
  SKILL: [
    'skill.before-execute',
    'skill.after-execute'
  ]
}

// Hook 实现示例
export function createWriteExistingFileGuardHook() {
  return {
    name: 'write-existing-file-guard',
    
    async beforeToolExecute({ tool, args }) {
      if (tool === 'write') {
        const { path } = args
        
        // 检查文件是否存在
        if (await fileExists(path)) {
          // 检查是否有 read-before-write 权限
          if (!hasReadPermission(path)) {
            return {
              proceed: false,
              error: 'File already exists. Use edit tool instead.'
            }
          }
        }
      }
      
      return { proceed: true }
    }
  }
}
```

### 三层 MCP: `src/mcp/`

```typescript
// src/mcp/ 目录结构
src/mcp/
├── websearch/          # Exa/Tavily 搜索
│   └── mcp-server.ts
├── context7/           # 官方文档查询
│   └── mcp-server.ts
├── grep_app/           # GitHub 代码搜索
│   └── mcp-server.ts
└── factory.ts          # MCP 工厂

// MCP 配置
interface MCPServerConfig {
  name: string
  type: 'built-in' | 'claude-code' | 'skill-embedded'
  command?: string        // stdio 模式
  url?: string           // HTTP 模式
  env?: Record<string, string>
}

// 三层 MCP
const MCP_LAYERS = {
  // Layer 1: Built-in (OMO 提供)
  BUILT_IN: [
    { name: 'websearch', command: 'npx @exa-ai/mcp-server' },
    { name: 'context7', command: 'npx @context7/mcp-server' },
    { name: 'grep_app', command: 'npx @grep-app/mcp-server' }
  ],
  
  // Layer 2: Claude Code (用户配置)
  CLAUDE_CODE: [
    // 从 .mcp.json 加载
    // 支持 ${VAR} 环境变量
  ],
  
  // Layer 3: Skill-embedded (Skill 自带)
  SKILL_EMBEDDED: [
    // 从 SKILL.md 加载
    // SkillMcpManager 管理
  ]
}
```

---

## 它当前的设计方案是什么

### 9.1 四层扩展架构详解

```
┌─────────────────────────────────────────────────────┐
│ Layer 1: Rules Layer (AGENTS.md)                    │
├─────────────────────────────────────────────────────┤
│ • 静态规则、项目知识                                │
│ • 注入到每个 prompt                                 │
│ • 无需代码，纯文本                                  │
├─────────────────────────────────────────────────────┤
│ 适合: 编码规范、项目结构、API约定                    │
│ 成本: 低 (纯文本)                                   │
│ 灵活: 低 (只能改文本)                               │
└─────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────┐
│ Layer 2: Plugin Layer (Hooks)                       │
├─────────────────────────────────────────────────────┤
│ • 生命周期拦截                                      │
│ • 行为扩展                                          │
│ • 权限控制                                          │
├─────────────────────────────────────────────────────┤
│ 适合: 权限守卫、输出处理、会话管理                   │
│ 成本: 中 (需要写 hook)                              │
│ 灵活: 中 (拦截点固定)                               │
└─────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────┐
│ Layer 3: Tool Layer (Tools)                         │
├─────────────────────────────────────────────────────┤
│ • 可调用能力、原子操作                              │
│ • Agent 直接调用                                    │
│ • 返回结构化数据                                    │
├─────────────────────────────────────────────────────┤
│ 适合: 新功能、领域特定操作                           │
│ 成本: 中 (需要实现 tool)                            │
│ 灵活: 高 (Agent 自由调用)                           │
└─────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────┐
│ Layer 4: MCP Layer (MCP)                            │
├─────────────────────────────────────────────────────┤
│ • 外部服务集成                                      │
│ • 独立进程运行                                      │
│ • 标准化协议                                        │
├─────────────────────────────────────────────────────┤
│ 适合: 外部 API、第三方服务                           │
│ 成本: 高 (需要实现 MCP server)                      │
│ 灵活: 高 (任意服务可接入)                           │
└─────────────────────────────────────────────────────┘
```

### 9.2 选择决策树

```
需求分类决策树：

你的需求是什么？
    │
    ├── 静态规则/项目知识？
    │   └── Rules Layer → AGENTS.md
    │       示例: "使用 2 空格缩进"
    │
    ├── 需要拦截生命周期？
    │   └── Plugin Layer → Hook
    │       示例: "写文件前检查权限"
    │
    ├── 需要原子操作能力？
    │   └── Tool Layer → Custom Tool
    │       示例: "查询数据库"
    │
    └── 需要外部服务？
        └── MCP Layer → MCP Server
            示例: "搜索网络"

复杂度递增：
Rules (1) → Plugin (2) → Tool (3) → MCP (4)
```

### 9.3 各层实现方式对比

| 层级 | 实现方式 | 示例 | 开发成本 | 运行时成本 |
|------|---------|------|---------|-----------|
| Rules | AGENTS.md 文件 | 项目规范、代码风格 | 低 | 无 |
| Plugin | hook.ts 文件 | write-existing-file-guard | 中 | 低 |
| Tool | tool-definition.ts | task, session-manager | 中 | 中 |
| MCP | mcp-server | websearch, context7 | 高 | 高 |

### 9.4 48个 Hooks 分类

```
Session Hooks (23个):
├── chat.message           # 消息处理
├── chat.params            # 参数调整
├── chat.headers           # 请求头
├── chat.response          # 响应处理
├── session.created        # 会话创建
├── session.ended          # 会话结束
├── session.error          # 会话错误
├── session.idle           # 会话空闲
├── context.injected       # 上下文注入
└── ... (14个更多)

Tool-Guard Hooks (12个):
├── write-existing-file-guard
├── file-guard
├── bash-guard
├── edit-guard
├── tool-output-truncator
└── ... (7个更多)

Transform Hooks (4个):
├── messages-transform
├── tool-result-transform
├── error-transform
└── output-transform

Continuation Hooks (7个):
├── todo-continuation
├── session-recovery
├── ralph-loop
├── handoff
└── ... (3个更多)

Skill Hooks (2个):
├── skill.before-execute
└── skill.after-execute
```

---

## 这套方案解决了什么，牺牲了什么

### 解决的问题

**扩展边界清晰**: 四层架构明确每种扩展的适用范围

**渐进增强**: 从简单规则到复杂 MCP，按需选择

**可组合性**: 各层可以组合使用

**生态兼容**: MCP 标准化协议，兼容外部服务

### 牺牲的代价

**学习成本**: 需要理解四层区别和选择

**架构复杂度**: 多层架构增加理解难度

**性能开销**: 每层都有额外的处理开销

**维护成本**: 多层需要分别维护

---

## 如果我要做自己的开发 Agent 套件，我会怎么迁移

### 扩展策略

```
从简单开始，逐步深入：

Step 1: Rules
    └── 用 AGENTS.md 定义项目规范
    └── 成本最低，效果立竿见影
    
Step 2: Plugin (如果需要)
    └── 添加权限守卫 Hook
    └── 添加输出处理 Hook
    
Step 3: Tool (如果需要)
    └── 实现自定义工具
    └── 比如数据库查询工具
    
Step 4: MCP (如果需要)
    └── 接入外部服务
    └── 比如私有 API
```

### 最小实现

```typescript
// 1. Rules Layer
// AGENTS.md
/*
# My Project Rules

## 代码规范
- 使用 TypeScript
- 使用 functional components
- 命名: PascalCase for components

## 常用命令
- npm test
- npm run build
*/

// 2. Plugin Layer
// src/hooks/my-guard/hook.ts
export function createMyGuardHook() {
  return {
    name: 'my-guard',
    async beforeToolExecute({ tool, args }) {
      if (tool === 'bash' && args.command.includes('rm -rf')) {
        return {
          proceed: false,
          error: 'Dangerous command blocked'
        }
      }
      return { proceed: true }
    }
  }
}

// 3. Tool Layer
// src/tools/my-query/tool.ts
export const myQueryTool = createTool({
  name: 'my_query',
  description: 'Query my database',
  parameters: z.object({
    table: z.string(),
    where: z.string().optional()
  }),
  async execute({ table, where }) {
    const result = await db.query(`SELECT * FROM ${table} ${where ? 'WHERE ' + where : ''}`)
    return { rows: result }
  }
})

// 4. MCP Layer
// mcp-servers/my-server.ts
import { Server } from '@modelcontextprotocol/sdk/server'

const server = new Server({
  name: 'my-mcp-server',
  version: '1.0.0'
}, {
  capabilities: { tools: {} }
})

server.setRequestHandler('tools/list', async () => ({
  tools: [{
    name: 'my_api',
    description: 'Call my API',
    inputSchema: { /* ... */ }
  }]
}))
```

---

## 一个最小实验

### 实验1: 创建一个简单 Hook

```typescript
// my-hook.ts
export function createMyHook() {
  return {
    name: 'my-hook',
    
    // 在工具执行前拦截
    async beforeToolExecute({ tool, args }) {
      console.log(`Tool ${tool} called with:`, args)
      return { proceed: true }
    },
    
    // 在工具执行后处理结果
    async afterToolExecute({ tool, result }) {
      console.log(`Tool ${tool} returned:`, result)
      return result
    }
  }
}

// 注册到 create-hooks.ts
export function createHooks(deps: Dependencies) {
  return [
    // ... 其他 hooks
    createMyHook()
  ]
}
```

### 实验2: 实现一个 Basic Tool

```typescript
// my-tool.ts
import { createTool } from '../factory'
import { z } from 'zod'

export const myTool = createTool({
  name: 'my_tool',
  description: 'My custom tool',
  
  parameters: z.object({
    input: z.string().describe('Input to process')
  }),
  
  async execute({ input }, ctx) {
    // 工具逻辑
    const processed = input.toUpperCase()
    
    return {
      result: processed,
      length: processed.length
    }
  }
})

// 注册到 tool-registry.ts
const CUSTOM_TOOLS = [
  myTool
  // ... 其他自定义工具
]
```

---

## 总结

能力扩展的四层架构：

1. **Rules**: AGENTS.md - 静态规则，成本最低
2. **Plugin**: Hooks - 生命周期拦截，权限控制
3. **Tool**: Tools - 原子能力，Agent 直接调用
4. **MCP**: MCP - 外部服务，标准化协议

**关键认知**: 不是每一层都需要，按需选择，从简单开始。

---

## 下一步

阅读下一篇：**《做一个开发效率增强模块：从源码理解到实战改造》**，完整实战示例。

---

**系列文章**: 《Oh My OpenAgent 源码解析：从复杂 Agent 学习到开发效率套件落地》
- ...
- 第8篇: 复杂 Agent 的记忆体系：运行时、沉淀层、长期知识
- 第9篇: Plugin、custom tool、MCP：能力扩展的三种路线（本文）
- 第10篇: 做一个开发效率增强模块：从源码理解到实战改造
- ...（共13篇）
