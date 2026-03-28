# OpenCode 底座与 OMO 编排层：边界在哪

> 本系列第2篇 | 基于项目版本: 2026-03-28 | 源码位置: `src/plugin-interface.ts`, `src/mcp/`, `src/plugin/tool-registry.ts`

---

## 这篇要回答的问题

1. **OpenCode 提供什么能力？**
2. **OMO 在此之上增强了什么？**
3. **哪些是继承，哪些是增强？**

---

## 源码里这个问题出现在哪里

### 插件接口: `src/plugin-interface.ts`

这是理解 OpenCode 底座与 OMO 增强的关键文件。它定义了插件需要实现的接口：

```typescript
// src/plugin-interface.ts 简化版
export interface PluginInterface {
  // OpenCode 底座提供的 hooks
  config?: ConfigHandler
  tool?: ToolHandler
  chat?: ChatHandlers
  event?: EventHandler
  toolExecute?: ToolExecuteHandlers
}
```

### 工具注册: `src/plugin/tool-registry.ts`

OpenCode 底座提供了26个工具，OMO 在此基础上可能添加或扩展：

```typescript
// 26个核心工具
const CORE_TOOLS = [
  // 文件操作
  'read', 'write', 'edit', 'glob', 'grep', 'look_at',
  // 任务执行
  'task', 'bash', 'interactive_bash',
  // 会话管理
  'session_list', 'session_read', 'session_search', 'session_info',
  'background_output', 'background_cancel',
  // LSP 工具
  'lsp_diagnostics', 'lsp_symbols', 'lsp_goto_definition',
  'lsp_find_references', 'lsp_prepare_rename', 'lsp_rename',
  // AST 工具
  'ast_grep_search', 'ast_grep_replace',
  // 网络工具
  'webfetch', 'websearch_web_search_exa',
  // 其他
  'skill', 'todowrite', 'question'
]
```

### 三层 MCP 系统: `src/mcp/`

```
Built-in MCP (src/mcp/)
    ├── websearch (Exa/Tavily)      ← OMO 内置
    ├── context7                     ← OMO 内置
    └── grep_app                     ← OMO 内置
    
Claude Code MCP (.mcp.json)         ← OpenCode 底座
    └── ${VAR} 环境变量扩展
    
Skill-embedded MCP (SKILL.md)       ← OMO 增强
    └── SkillMcpManager 管理
```

---

## 它当前的设计方案是什么

### 2.1 OpenCode 底座能力

*** 确定这个地方是 Open Core 的底座能力，而不是 OMO 增强的底座能力？

| 能力 | 说明 | 源码位置 |
|------|------|---------|
| **Tool 系统** | 26个工具：bash, read, write, edit... | `src/plugin/tool-registry.ts` |
| **Rule 系统** | AGENTS.md 规则注入 | `src/plugin/chat-message.ts` |
| **Plugin 系统** | 8个 hook handlers | `src/plugin/handlers/` |
| **MCP 系统** | 外部服务接入 | `.mcp.json` |
| **LSP 集成** | 代码诊断 | `src/tools/lsp/` |
| **SDK** | OpenCode SDK | `opencode` npm包 |

### 2.2 OMO 编排层增强

| 增强点 | 说明 | 源码位置 |
|--------|------|---------|
| **多 Agent 编排** | 11个 agent，角色分工 | `src/agents/` |
| **Intent Gate** | 复杂度判断与路由 | `src/hooks/intent-gate/` |
| **Task Delegation** | 任务委派机制 | `src/tools/delegate-task/` |
| **Context Management** | 上下文监控与压缩 | `src/hooks/context-window-monitor/` |
| **Verification** | 验收闭环 | `src/agents/momus/` |
| **Fallback** | 降级恢复 | `src/hooks/model-fallback/` |

### 2.3 三层 MCP 系统详解

```
┌─────────────────────────────────────────────────────┐
│ Layer 1: Built-in MCP (OMO 提供)                    │
├─────────────────────────────────────────────────────┤
│ • websearch (Exa/Tavily) - 网络搜索                 │
│ • context7 - 官方文档查询                           │
│ • grep_app - GitHub 代码搜索                        │
│ 位置: src/mcp/                                      │
└─────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────┐
│ Layer 2: Claude Code MCP (OpenCode 底座)            │
├─────────────────────────────────────────────────────┤
│ • 用户自定义 MCP 服务器                              │
│ • ${VAR} 环境变量扩展                               │
│ 位置: .mcp.json                                     │
└─────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────┐
│ Layer 3: Skill-embedded MCP (OMO 增强)              │
├─────────────────────────────────────────────────────┤
│ • Skills 自带 MCP 服务器                            │
│ • SkillMcpManager 动态管理                          │
│ 位置: SKILL.md 中的 mcp_servers 字段                │
└─────────────────────────────────────────────────────┘
```

### 2.4 继承 vs 增强对比表

| 功能 | OpenCode 底座 | OMO 增强 | 关系 |
|------|--------------|---------|------|
| Tools | 26个基础工具 | 可能扩展 | 继承+扩展 |
| Hooks | 8个基础 handler | 48个具体 hook | 继承+细化 |
| MCP | 基础接入能力 | 3个内置 + Skill嵌入 | 增强 |
| Agents | 无 | 11个角色 | 新增 |
| Rules | AGENTS.md 支持 | 目录级注入 | 增强 |
| Verification | 基础检查 | 四维验证 | 增强 |

---

## 这套方案解决了什么，牺牲了什么

### 解决的问题

**能力边界清晰**: 底座能力 vs 编排增强明确分离，可独立演进

**可扩展性**: 三层 MCP 让不同场景可以选择合适的扩展方式

**兼容性**: 完全兼容 OpenCode 底座，现有插件可用

### 牺牲的代价

**理解成本**: 需要理解两层架构，增加了学习负担

**配置复杂度**: 需要同时理解 OpenCode 和 OMO 的配置

**调试难度**: 问题可能出现在底座层或编排层，需要分层排查

---

## 如果我要做自己的开发 Agent 套件，我会怎么迁移

### 迁移策略

```
1. 底座能力直接用
   └── OpenCode 提供的工具、hooks、MCP → 直接用

2. 编排能力按场景定制
   └── 参考 OMO 的设计，但不一定全部照搬

3. MCP 分层使用
   ├── 通用能力 → Built-in MCP
   ├── 项目特定 → Claude Code MCP
   └── Skill 特定 → Skill-embedded MCP
```

### 选择指南

| 需求类型 | 推荐方案 | 原因 |
|---------|---------|------|
| 通用搜索 | Built-in MCP | 开箱即用 |
| 团队私有服务 | Claude Code MCP | 项目级配置 |
| Skill 专属能力 | Skill-embedded MCP | 按需加载，上下文隔离 |

---




## 一个最小实验

### 实验1: 查看工具列表

```bash
# 查看所有注册的工具
grep -r "createTool" src/plugin/tool-registry.ts
```

理解：
- 哪些是 OpenCode 底座提供的？
- 哪些是 OMO 添加的？

### 实验2: 理解插件接口组装

```typescript
// src/index.ts 中的 createPluginInterface
export function createPluginInterface(deps: Dependencies): PluginInterface {
  return {
    // Tool hook - 继承自 OpenCode
    tool: async () => deps.tools.getAll(),
    
    // Chat hooks - OMO 增强
    chat: {
      message: createChatMessageHandler(deps),  // 添加 AGENTS.md 注入
      params: createChatParamsHandler(deps),     // 模型参数调整
      headers: createChatHeadersHandler(deps),   // 请求头注入
    },
    
    // Event hook - OMO 增强
    event: createEventHandler(deps),             // 会话生命周期管理
    
    // Config hook - OMO 增强
    config: createConfigHandler(deps),           // 多级配置加载
    
    // Tool execution hooks - OMO 增强
    toolExecute: {
      before: createToolExecuteBeforeHandler(deps),  // 权限检查
      after: createToolExecuteAfterHandler(deps),    // 输出处理
    },
  }
}
```

**观察重点**:
- 哪些是纯继承？（如 `tool`）
- 哪些是增强？（如 `chat.message` 添加了 AGENTS.md 注入）

---

## 总结

OpenCode 底座与 OMO 编排层的边界清晰：

1. **OpenCode 提供**: Tool 系统、基础 Hooks、MCP 接入、LSP、SDK
2. **OMO 增强**: 多 Agent 编排、Intent Gate、任务委派、上下文管理、验证闭环
3. **三层 MCP**: Built-in（OMO）→ Claude Code（底座）→ Skill-embedded（OMO）

**关键认知**: 好的插件不是替代底座，而是在底座之上做有意义的增强。

---

## 下一步

阅读下一篇：**《复杂任务为什么必须先规划再执行》**，理解任务如何从用户需求变成可执行工作流。

---

**系列文章**: 《Oh My OpenAgent 源码解析：从复杂 Agent 学习到开发效率套件落地》
- 第1篇: 为什么 Oh My OpenAgent 值得作为复杂 Agent 学习样本
- 第2篇: OpenCode 底座与 OMO 编排层：边界在哪（本文）
- 第3篇: 复杂任务为什么必须先规划再执行
- ...（共13篇）
