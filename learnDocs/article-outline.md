# 《Oh My OpenAgent 源码解析：从复杂 Agent 学习到开发效率套件落地》

> 系列文章大纲 | 基于项目版本: 2026-03-28

---

## 系列总览

本系列共 **13 篇文章**，按照"由浅入深、问题驱动"的原则组织：

| 编号 | 标题 | 核心问题 | 代码焦点 |
|------|------|---------|---------|
| 第1篇 | 为什么 Oh My OpenAgent 值得作为复杂 Agent 学习样本 | 它解决的到底是什么问题？ | `AGENTS.md`, `src/index.ts` |
| 第2篇 | OpenCode 底座与 OMO 编排层：边界在哪 | 底座提供什么，编排层增强什么？ | `src/plugin-interface.ts`, 三层MCP |
| 第3篇 | 复杂任务为什么必须先规划再执行 | 任务如何从需求变成可执行工作流？ | `src/hooks/intent-gate/`, `src/agents/prometheus/` |
| 第4篇 | Planner / Reviewer / Executor 的角色分工设计 | 为什么不能合并成一个"超级Agent"？ | `src/agents/` 所有agent定义 |
| 第5篇 | 多 Agent 委派不是炫技，而是稳定性交换 | 为什么多Agent比单Agent更稳定？ | `src/tools/delegate-task/`, agent权限 |
| 第6篇 | AGENTS.md、rules 与目录上下文注入机制 | Agent的"外部脑"是如何构建的？ | `src/plugin/chat-message.ts`, 上下文注入 |
| 第7篇 | Context overload、压缩与会话恢复 | 长任务为什么会漂？如何控制？ | `src/hooks/context-window-monitor/`, compaction |
| 第8篇 | 复杂 Agent 的记忆体系：运行时、沉淀层、长期知识 | 记忆应该分成哪几层？ | `src/hooks/handoff/`, skills系统 |
| 第9篇 | Plugin、custom tool、MCP：能力扩展的三种路线 | 扩展能力应该放在哪一层？ | `src/plugin/tool-registry.ts`, `src/mcp/` |
| 第10篇 | 做一个开发效率增强模块：从源码理解到实战改造 | 如何落地自己的工具？ | 实战：PR助手/重构助手 |
| 第11篇 | Verification、fallback 与稳定交付闭环 | 为什么"能跑"不等于"能交付"？ | `src/agents/momus/`, verification |
| 第12篇 | 权限治理与产品化：让 Agent 真正进入日常开发 | 如何保证安全可控？ | `src/plugin-config.ts`, 权限系统 |
| 第13篇 | 如何搭建自己的领域插件工程架构 | 从零搭建全套工程需要什么？ | `package.json`, CI/CD, 测试架构 |

---

## 第一部分：建立认知框架

### 第1篇：为什么 Oh My OpenAgent 值得作为复杂 Agent 学习样本

**这篇要回答的问题**:
- 它解决的到底是什么问题？
- 它和 OpenCode 的关系是什么？
- 为什么它值得作为复杂 Agent 学习样本？

**源码位置**:
- `AGENTS.md` — 系统总览，定义了 runtime + orchestration + workflow policy 三层
- `src/index.ts` — 5步初始化流程

**核心内容**:

#### 1.1 复杂 Agent 的核心挑战
- Context overload（上下文过载）
- Cognitive drift（认知漂移）
- Verification gaps（验证缺口）
- Task decomposition（任务分解）
- Multi-agent coordination（多agent协调）

#### 1.2 OMO 的定位
```
OpenCode（底座）          →  tool substrate, agent runtime
OMO（编排增强层）         →  orchestration, workflow policy, role separation
```

#### 1.3 为什么不是"多几个 prompt"
- 角色分工：orchestrator, planner, executor, reviewer, specialist
- 分层架构：底座层 → 编排层 → 执行层 → 扩展层 → 治理层
- 稳定机制：verification, fallback, session recovery

**设计方案**:
- 解决了：复杂任务的分解、委派、验证、恢复
- 牺牲了：简单场景的轻量性（需要理解多层抽象）

**迁移指南**:
- 如果要做自己的开发 Agent 套件，先画分层图
- 明确哪些能力在底座，哪些在编排层

**最小实验**:
- 用 `bunx oh-my-opencode doctor` 检查环境
- 阅读 `AGENTS.md` 画出自己的理解图

---

### 第2篇：OpenCode 底座与 OMO 编排层：边界在哪

**这篇要回答的问题**:
- OpenCode 提供什么能力？
- OMO 在此之上增强了什么？
- 哪些是继承，哪些是增强？

**源码位置**:
- `src/plugin-interface.ts` — 插件接口定义
- `src/mcp/` — 三层MCP系统
- `src/plugin/tool-registry.ts` — 26个工具注册

**核心内容**:

#### 2.1 OpenCode 原生能力（底座）

OpenCode 作为底层平台，提供的是**会话基础设施**：

| 能力 | API/机制 | 说明 |
|------|---------|------|
| **Session 管理** | `client.session.create()`, `get()`, `list()` | 创建、获取、列出会话 |
| **父子 Session** | `parentID` 字段 | 通过 parentID 建立会话层级 |
| **异步消息** | `client.session.promptAsync()` | 发送消息立即返回，不等待完成 |
| **同步消息** | `client.session.prompt()` | 发送消息并等待响应 |
| **子会话查询** | `client.session.children()` | 获取某会话的子会话 |
| **事件系统** | `session.created`, `session.idle`, `session.compacted` | 会话生命周期事件 |
| **插件钩子** | `hooks.tool`, `hooks.event`, `hooks.chat.*` | 扩展点机制 |

**OpenCode 不提供的能力**（这些正是 OMO 补充的）：
- ❌ `task` 工具/函数 — 没有内置的子 Agent 委派
- ❌ Task ID 生成与管理 — 没有任务标识系统
- ❌ 后台任务队列 — 没有并发控制或任务调度
- ❌ Agent Category 路由 — 没有按 category 选择模型
- ❌ 任务完成检测/轮询 — 没有内置的任务状态轮询
- ❌ 上下文压缩状态恢复 — 没有 agent/model 配置检查点

#### 2.2 OMO 编排层增强

OMO 在 OpenCode 之上构建了**完整的多 Agent 编排系统**：

```
┌─────────────────────────────────────────────────────────────┐
│                    OpenCode（底座层）                        │
├─────────────────────────────────────────────────────────────┤
│ • Session CRUD（create, get, list, delete）                 │
│ • Parent-child 关系（parentID）                              │
│ • Async/sync 消息发送                                        │
│ • 事件通知（idle, compacted, error, etc.）                  │
│ • 插件钩子系统                                               │
└─────────────────────────────────────────────────────────────┘
                            ▲
                            │ 扩展
┌─────────────────────────────────────────────────────────────┐
│              OMO（oh-my-opencode 编排层）                   │
├─────────────────────────────────────────────────────────────┤
│ • Task 系统（`task()` 工具）— 创建、跟踪、管理工作单元      │
│ • Task ID 生成（`bg_xxx`, `sync_xxx`）— 唯一任务标识        │
│ • 并发控制 — 按 model/provider 限制并发（默认5）            │
│ • 后台执行 — 异步任务 + 轮询通知                           │
│ • 同步执行 — 等待完成再返回                                │
│ • Category 路由 — 8 个内置 category 映射到不同模型          │
│ • Agent 委派 — explore/librarian/oracle/metis/momus        │
│ • 压缩状态恢复 — agent/model/todos/task history 保持       │
│ • Todo 强制执行 — 未完成任务自动续接                       │
│ • 多 Agent 状态管理 — Atlas/Boulder 协调机制               │
└─────────────────────────────────────────────────────────────┘
```

**详细能力对照**：

| 场景 | 纯 OpenCode | 使用 OMO |
|------|-------------|---------|
| **边coding边research** | 手动：复制粘贴研究结果 | 自动：`task(subagent_type="explore", run_in_background=true)` — 结果准备好时自动通知 |
| **多文件重构** | 单 Agent 顺序处理 | 并行：后台 Agent 同时处理不同文件 |
| **上下文溢出** | Session 停止；工作丢失 | 自动压缩恢复；Agent 配置保持；任务续接 |
| **模型选择** | 每请求手动切换 | 自动：`category="visual-engineering"` → Gemini；`"ultrabrain"` → GPT-5.4 |
| **复杂任务规划** | 即兴提示 | Prometheus 访谈 → 创建可验证计划 → Atlas 执行 |
| **未完成工作** | Agent 空闲；任务遗忘 | Todo 强制执行器强制续接；Boulder 状态持久化 |

**核心源码位置**：
- `src/tools/delegate-task/tools.ts` — `task()` 工具实现
- `src/features/background-agent/manager.ts` — BackgroundManager 核心类
- `src/features/background-agent/spawner.ts` — Task ID 生成、session 创建
- `src/tools/delegate-task/sync-task.ts` — 同步任务执行
- `src/tools/delegate-task/background-task.ts` — 后台任务执行
- `src/hooks/compaction-context-injector/hook.ts` — 上下文压缩时的状态注入
- `src/hooks/compaction-todo-preserver/hook.ts` — 压缩时保持 todo 状态
- `src/hooks/todo-continuation-enforcer/` — Boulder 机制强制任务继续
- `src/hooks/atlas/` — 多 session 的 orchestration

#### 2.3 关键机制详解：Task ID 与 Session ID

当调用 `task()` 时，OMO 如何管理任务标识和通信：

**Task 调用流程**：
```typescript
task({
  category: "visual-engineering",
  description: "创建响应式头部",
  prompt: "添加响应式导航头部...",
  run_in_background: true,
})
```

**底层执行流程**：
```
OMO task() 工具
    ↓
1. 解析 category → model（如 "visual-engineering" → Gemini 3.1 Pro）
2. 生成 Task ID（如 "bg_a1b2c3d4"）
3. 调用 OpenCode: client.session.create({ parentID: 当前Session })
4. 调用 OpenCode: client.session.promptAsync({ sessionID: 新Session, ... })
    ↓
如果 run_in_background=true:
   - 立即返回 Task ID
   - 每3秒轮询 client.session.get()
   - 完成后向父 Session 注入通知

如果 run_in_background=false:
   - 阻塞直到 session.idle 事件
   - 返回结果给调用者
```

**ID 体系对照**：

| OMO 概念 | OpenCode 概念 | 用途 |
|---------|---------------|------|
| `task_id` (`bg_xxx` 或 `sync_xxx`) | 无 | OMO 生成的任务跟踪标识 |
| `session_id` | Session `id` 字段 | OpenCode 的会话标识 |
| `parentSessionID` | Session `parentID` | 子 Session 关联父 Session |

**结果通信机制**：
- 后台任务：完成后 OMO 的 `result-handler.ts` 调用 OpenCode 向父 Session 注入系统消息
- 同步任务：结果直接返回在工具响应中

#### 2.4 上下文压缩与状态保持

当 OpenCode 触发 `session.compacted`（上下文窗口满）时，OMO 的钩子捕获并恢复：

**保持的状态**：
1. **Agent 配置**（当前运行哪个 agent）
2. **模型设置**（provider、model ID、variant）
3. **工具限制**（该 agent 能/不能做什么）
4. **活动 todos**（还有哪些工作未完成）
5. **后台任务历史**（哪些 sub-agent 正在运行）

**实现文件**：
- `src/hooks/compaction-context-injector/hook.ts` — 捕获 agent/model/tools
- `src/hooks/compaction-todo-preserver/hook.ts` — 保持 todo 状态
- `src/features/background-agent/manager.ts` — 跟踪 sub-agent 任务

**关键点**：没有 OMO，上下文压缩会重置 agent 到默认设置并丢失委派的工作。

#### 2.5 三层 MCP 系统
```
Built-in MCP (src/mcp/)
    ├── websearch (Exa/Tavily)
    ├── context7
    └── grep_app
    
Claude Code MCP (.mcp.json)
    └── ${VAR} 环境变量扩展
    
Skill-embedded MCP (SKILL.md)
    └── SkillMcpManager 管理
```

**设计方案**:
- 解决了：能力边界清晰，可独立演进
- 牺牲了：需要理解两层架构

**迁移指南**:
- 底座能力直接用，编排能力根据场景定制
- MCP 优先用 Built-in，复杂场景用 Skill-embedded

**最小实验**:
- 查看 `src/plugin/tool-registry.ts` 中的工具列表
- 理解 `createPluginInterface` 的组装逻辑
- 用 `background_output(task_id="...")` 查看后台任务结果

---

## 第二部分：吃透任务编排

### 第3篇：从用户需求到执行计划，任务是怎么被拆开的

**这篇要回答的问题**:
- 复杂度判断如何做？
- 规划入口在哪里？
- 任务拆分粒度如何控制？
- 验收条件如何定义？

**源码位置**:
- `src/hooks/intent-gate/` — Intent Gate 实现
- `src/agents/prometheus/` — 规划 agent
- `src/agents/metis/` — 预规划分析
- `src/hooks/todo-continuation/` — Todo 管理

**核心内容**:

#### 3.1 Intent Gate（意图门）
```
用户请求 → Intent Gate → 简单任务？ → 直接执行
                        → 复杂任务？ → 进入规划流程
```

**判断维度**:
- 任务涉及模块数
- 是否需要外部资源
- 是否需要多步骤
- 是否有明确验收标准

#### 3.2 规划流程
```
Metis（预规划分析）
    ↓
Prometheus（规划agent）
    ↓
Momus（审查规划）
    ↓
执行
```

#### 3.3 任务拆分原则
- 单一职责：每个任务一个明确目标
- 可验证：每个任务有验收标准
- 可委派：明确谁能执行
- 粒度适中：不过大也不过小

**设计方案**:
- 解决了：复杂任务的可控执行
- 牺牲了：简单场景的延迟增加

**迁移指南**:
- 设计自己的 Intent Gate 规则
- 定义任务拆分模板

**最小实验**:
- 阅读 `src/hooks/intent-gate/hook.ts` 理解判断逻辑
- 尝试设计一个简单的复杂度判断规则

---

### 第4篇：Planner / Reviewer / Executor 的角色分工设计

**这篇要回答的问题**:
- planner 负责什么？
- reviewer 为什么必要？
- executor 为什么必须受约束？
- intelligence 在系统里，而不是在某个 agent 里？

**源码位置**:
- `src/agents/prometheus/` — Planner
- `src/agents/metis/` — 预规划顾问
- `src/agents/momus/` — Reviewer
- `src/agents/hephaestus/` — Executor

**核心内容**:

#### 4.1 角色职责矩阵
| 角色 | 职责 | 工具权限 | 可委派 |
|------|------|---------|--------|
| Metis | 分析需求，识别风险 | 只读 | 否 |
| Prometheus | 制定计划，分解任务 | 只读 | 是 |
| Momus | 审查计划，验证完整性 | 只读 | 否 |
| Hephaestus | 执行任务，落地代码 | 全部 | 是 |
| Sisyphus | 主编排，协调整体流程 | 全部 | 是 |

#### 4.2 为什么不能合并
```
如果合并成"超级Agent"：
├── 职责不清：既规划又执行，容易自我合理化
├── 缺乏制衡：没有独立审查环节
├── 上下文膨胀：所有信息堆在一个上下文
└── 难以调试：问题定位困难
```

#### 4.3 Intelligence 分布
- 不在单个 agent，而在系统协作
- 规划质量 = 系统设计质量
- 执行稳定性 = 角色约束强度

**设计方案**:
- 解决了：职责清晰，问题可追溯
- 牺牲了：需要理解多角色协作

**迁移指南**:
- 根据任务复杂度选择角色分工
- 简单任务可简化，复杂任务必须分层

**最小实验**:
- 分析一个实际请求如何流转
- 画出角色交互时序图

---

### 第5篇：多 Agent 委派不是炫技，而是稳定性交换

**这篇要回答的问题**:
- 并行的价值是什么？
- 专职角色的价值是什么？
- 什么时候不该多 agent？

**源码位置**:
- `src/tools/delegate-task/` — 委派工具
- `src/tools/delegate-task/constants.ts` — agent categories

**核心内容**:

#### 5.1 并行的价值
```
串行执行：Task1 → Task2 → Task3 → Task4
并行执行：Task1 ┐
              Task2 ├→ 合并结果
              Task3 ┘
```

**优势**:
- 时间效率：独立任务同时执行
- 容错性：一个失败不影响其他
- 上下文隔离：各任务独立上下文

#### 5.2 专职角色的价值
| 类型 | Agent | 优势 |
|------|-------|------|
| 只读顾问 | Oracle, Librarian, Explore | 不会误操作，专注分析 |
| 执行者 | Hephaestus | 有写权限，专注实现 |
| 审查者 | Momus | 独立视角，质量控制 |

#### 5.3 什么时候不该多 agent
- 任务简单，单 agent 可完成
- 任务依赖强，无法并行
- 上下文共享需求高
- 成本敏感（多 agent = 多 token）

**设计方案**:
- 解决了：稳定性通过分工实现
- 牺牲了：复杂度和成本增加

**迁移指南**:
- 评估任务复杂度决定是否多 agent
- 设计自己的委派策略

**最小实验**:
- 分析 `delegate-task` 的实现逻辑
- 理解 `DEFAULT_CATEGORIES` 的分类

---

## 第三部分：吃透上下文、记忆与恢复

### 第6篇：AGENTS.md、rules 与目录上下文注入机制

**这篇要回答的问题**:
- project / global 规则如何生效？
- 目录级上下文如何叠加？
- 规则优先级是什么？
- 为什么要 commit AGENTS.md？

**源码位置**:
- `AGENTS.md` — 项目规则
- `src/plugin/chat-message.ts` — 上下文注入
- `src/plugin-config.ts` — 配置加载

**核心内容**:

#### 6.1 AGENTS.md 层级
```
~/.config/opencode/AGENTS.md     ← Global（用户级）
    ↓ 覆盖
项目根目录/AGENTS.md              ← Project（项目级）
    ↓ 合并
子目录/AGENTS.md                  ← Directory（目录级）
```

#### 6.2 注入时机
```
初始化阶段：
    loadPluginConfig() → 读取 AGENTS.md
    
每条消息：
    chat.message hook → 注入到 prompt
```

#### 6.3 规则优先级
1. Project 规则覆盖 Global 规则
2. Directory 规则补充 Project 规则
3. `disabled_*` 数组取并集

#### 6.4 为什么 commit AGENTS.md
- 团队共享规则
- 版本控制规则变更
- 环境一致性

**设计方案**:
- 解决了：项目知识持久化
- 牺牲了：规则文件维护成本

**迁移指南**:
- 设计项目 AGENTS.md 模板
- 规划目录级规则

**最小实验**:
- 创建一个测试 AGENTS.md
- 观察规则如何影响 agent 行为

---

### 第7篇：Context overload、压缩与会话恢复

**这篇要回答的问题**:
- context monitor 如何工作？
- preemptive compaction 何时触发？
- compaction 时保留什么、丢弃什么？
- 什么信息不该靠上下文硬扛？

**源码位置**:
- `src/hooks/context-window-monitor/` — 上下文监控
- `src/hooks/preemptive-compaction/` — 预压缩
- `src/hooks/session-recovery/` — 会话恢复

**核心内容**:

#### 7.1 Context Window Monitor
```typescript
// 监控指标
interface ContextMetrics {
  currentTokens: number
  maxTokens: number
  usageRatio: number  // 触发阈值：0.8
}
```

#### 7.2 Preemptive Compaction
```
触发条件：usageRatio > 0.8
    ↓
压缩策略：
    ├── 保留：任务目标、关键决策、验证标准
    ├── 压缩：历史对话摘要
    └── 丢弃：冗余信息
```

#### 7.3 Session Recovery
```
会话中断 → recovery hook
    ↓
恢复策略：
    ├── 读取保存的状态
    ├── 重建上下文
    └── 继续任务
```

#### 7.4 不该靠上下文硬扛的信息
- 项目长期知识 → AGENTS.md
- 历史决策 → notepad/decisions
- 团队规范 → rules

**设计方案**:
- 解决了：长任务不丢上下文
- 牺牲了：压缩可能丢失细节

**迁移指南**:
- 设计自己的压缩策略
- 规划信息持久化方案

**最小实验**:
- 分析 context-window-monitor 的阈值设置
- 理解 compaction 的保留逻辑

---

### 第8篇：复杂 Agent 的记忆体系：运行时、沉淀层、长期知识

**这篇要回答的问题**:
- 哪些信息只应留在会话里？
- 哪些应该写入 notepad / verification / decisions？
- 哪些应该回写项目知识？
- 会话中断后如何恢复？

**源码位置**:
- `src/hooks/handoff/` — 会话交接
- `src/hooks/session-recovery/` — 会话恢复
- `src/features/builtin-skills/` — skills 系统

**核心内容**:

#### 8.1 三层记忆模型
```
┌─────────────────────────────────────┐
│ 运行时记忆（Session）               │  ← 会话状态、当前上下文
├─────────────────────────────────────┤
│ 阶段性沉淀（Learnings）             │  ← notepad, decisions, issues
├─────────────────────────────────────┤
│ 项目长期知识（Knowledge）           │  ← AGENTS.md, skills, rules
└─────────────────────────────────────┘
```

#### 8.2 信息流转
```
任务执行 → 生成 learnings
    ↓
沉淀到 decisions/issues
    ↓
回写到 AGENTS.md/skills
```

#### 8.3 Session Handoff
```typescript
interface HandoffData {
  sessionContext: SessionState
  pendingTasks: Task[]
  decisions: Decision[]
  issues: Issue[]
}
```

#### 8.4 记忆边界
| 类型 | 存储位置 | 生命周期 |
|------|---------|---------|
| 运行时 | 内存 | 会话结束 |
| 沉淀 | notepad | 跨会话 |
| 长期 | AGENTS.md | 永久 |

**设计方案**:
- 解决了：知识不丢失
- 牺牲了：维护复杂度

**迁移指南**:
- 设计自己的记忆分层
- 实现知识回写机制

**最小实验**:
- 分析 handoff hook 的实现
- 设计一个简单的知识沉淀流程

---

## 第四部分：吃透工具与扩展

### 第9篇：从源码阅读走到能力扩展，plugin、tool、MCP 该怎么选

**这篇要回答的问题**:
- 规则层 vs plugin 层 vs tool 层 vs MCP 层的区别？
- 哪类需求适合放哪一层？

**源码位置**:
- `src/plugin/tool-registry.ts` — 工具注册
- `src/hooks/` — 所有 hooks
- `src/mcp/` — MCP 集成
- `src/features/builtin-skills/` — skills

**核心内容**:

#### 9.1 四层扩展架构
```
┌─────────────────────────────────────┐
│ Rules Layer (AGENTS.md)             │  ← 静态规则、项目知识
├─────────────────────────────────────┤
│ Plugin Layer (hooks)                │  ← 生命周期拦截、行为扩展
├─────────────────────────────────────┤
│ Tool Layer (tools)                  │  ← 可调用能力、原子操作
├─────────────────────────────────────┤
│ MCP Layer (mcp/)                    │  ← 外部服务集成
└─────────────────────────────────────┘
```

#### 9.2 选择决策树
```
需求类型
├── 静态规则/项目知识？ → AGENTS.md
├── 生命周期拦截？ → Plugin Hook
├── 原子操作能力？ → Custom Tool
└── 外部服务？ → MCP
```

#### 9.3 各层实现方式
| 层级 | 实现方式 | 示例 |
|------|---------|------|
| Rules | AGENTS.md 文件 | 项目规范、代码风格 |
| Plugin | hook.ts 文件 | write-existing-file-guard |
| Tool | tool-definition.ts | task, session-manager |
| MCP | mcp server | websearch, context7 |

#### 9.4 48个 Hooks 分类
```
Session Hooks (23):
    ├── chat.message
    ├── chat.params
    ├── chat.headers
    └── ...
    
Tool-Guard Hooks (12):
    ├── write-existing-file-guard
    ├── file-guard
    └── ...
    
Transform Hooks (4):
    ├── messages-transform
    └── ...
    
Continuation Hooks (7):
    ├── todo-continuation
    ├── session-recovery
    └── ...
    
Skill Hooks (2):
    └── ...
```

**设计方案**:
- 解决了：扩展边界清晰
- 牺牲了：需要理解多层架构

**迁移指南**:
- 按需求类型选择扩展层
- 从简单 Rules 开始，逐步深入

**最小实验**:
- 创建一个简单的 hook
- 实现一个 basic tool

---

### 第10篇：做一个开发效率增强模块：从源码理解到实战改造

**这篇要回答的问题**:
- 如何选择落地场景？
- 如何设计和实现？
- 如何验证效果？

**实战场景选择**:
- PR review 助手
- 重构辅助
- Bug 排查
- 文档生成

**源码参考**:
- `src/features/builtin-skills/` — skills 实现
- `src/features/builtin-commands/` — commands 实现

**核心内容**:

#### 10.1 设计步骤
```
1. 定义目标场景
2. 分析所需能力
3. 选择扩展层
4. 实现核心逻辑
5. 编写测试
6. 集成验证
```

#### 10.2 PR Review 助手示例
```typescript
// 设计思路
interface PRCReviewSkill {
  // 获取 PR 变更
  fetchChanges: () => Promise<Changes>
  // 分析变更
  analyzeChanges: (changes: Changes) => Analysis
  // 生成建议
  generateSuggestions: (analysis: Analysis) => Suggestion[]
  // 输出报告
  outputReport: (suggestions: Suggestion[]) => void
}
```

#### 10.3 实现清单
- [ ] 创建 skill 定义文件
- [ ] 实现核心逻辑
- [ ] 编写测试用例
- [ ] 编写使用文档
- [ ] 集成测试

**设计方案**:
- 解决了：从理解到落地
- 牺牲了：需要实际编码

**迁移指南**:
- 选择自己熟悉的场景
- 参考现有 skill 实现

**最小实验**:
- 实现一个简单的 PR 检查工具
- 验证基本功能

---

## 第五部分：吃透工程化交付

### 第11篇：为什么复杂 Agent 不能没有 verification

**这篇要回答的问题**:
- clarity / verification / context / big picture 各检查什么？
- 验收标准怎么定义？
- 自动验证和人工复核的边界？

**源码位置**:
- `src/agents/momus/` — reviewer agent
- `src/hooks/verification/` — 验证机制
- `src/tools/lsp/diagnostics-tool.ts` — LSP 诊断

**核心内容**:

#### 11.1 四维验证
| 维度 | 检查内容 | 不通过的后果 |
|------|---------|-------------|
| Clarity | 任务是否清晰 | 返回澄清请求 |
| Verification | 是否有验证手段 | 要求补充验收标准 |
| Context | 上下文是否充分 | 提示需要更多信息 |
| Big Picture | 是否理解全局 | 提供背景说明 |

#### 11.2 验收标准定义
```typescript
interface AcceptanceCriteria {
  // 功能完成
  functional: string[]
  // 测试通过
  tests: string[]
  // 代码质量
  quality: string[]
  // 文档完整
  documentation: string[]
}
```

#### 11.3 自动验证 vs 人工复核
```
自动验证：
├── LSP diagnostics（类型错误、语法错误）
├── 测试运行
├── Lint 检查
└── 构建验证

人工复核：
├── 设计决策
├── 业务逻辑
├── 边界情况
└── 用户体验
```

**设计方案**:
- 解决了：交付质量可控
- 牺牲了：流程复杂度增加

**迁移指南**:
- 定义自己的验收标准模板
- 设置自动验证门槛

**最小实验**:
- 分析 momus agent 的审查逻辑
- 设计一个简单的验收清单

---

### 第12篇：权限、回退与恢复，Agent 系统怎么才能日常可用

**这篇要回答的问题**:
- permissions 如何配置？
- tool restriction 如何生效？
- fallback 何时触发？
- session recovery 如何工作？

**源码位置**:
- `src/plugin-config.ts` — 权限配置
- `src/hooks/model-fallback/` — 模型降级
- `src/hooks/session-recovery/` — 会话恢复
- `src/hooks/write-existing-file-guard/` — 写守卫

**核心内容**:

#### 12.1 权限级别
```jsonc
{
  "permissions": {
    "bash": "ask",      // 需要确认
    "edit": "allow",    // 自动执行
    "write": "deny"     // 禁止执行
  }
}
```

#### 12.2 Agent 级权限
```typescript
// 不同 agent 有不同的工具限制
const agentPermissions = {
  oracle: { write: 'deny', bash: 'deny' },
  hephaestus: { write: 'allow', bash: 'ask' },
  sisyphus: { write: 'allow', bash: 'allow' }
}
```

#### 12.3 Fallback 机制
```
模型调用失败
    ↓
Model Fallback Hook
    ↓
尝试备选模型
    ↓
成功？继续 / 失败？报错
```

#### 12.4 Recovery 流程
```
会话中断
    ↓
保存状态
    ↓
恢复触发
    ↓
重建上下文
    ↓
继续执行
```

**设计方案**:
- 解决了：系统可用性
- 牺牲了：配置复杂度

**迁移指南**:
- 根据风险级别设置权限
- 设计降级策略

**最小实验**:
- 配置不同的权限级别
- 测试 fallback 触发

---

## 第六部分：工程架构（新增）

### 第13篇：如何搭建自己的领域插件工程架构

**这篇要回答的问题**:
- 为什么选择 Bun？
- 如何保证交付稳定性与护栏？
- 如何进行自动化测试？
- 如何构建跨平台二进制？
- 如何搭建 CI/CD？
- 从零搭建全套工程需要什么？

**源码位置**:
- `package.json` — 构建配置
- `tsconfig.json` — TypeScript 配置
- `bunfig.toml` — Bun 测试配置
- `.github/workflows/` — CI/CD
- `script/build-binaries.ts` — 跨平台编译

**核心内容**:

#### 13.1 为什么选择 Bun

**Bun 特性**:
| 特性 | 说明 |
|------|------|
| Runtime | Bun 运行时，替代 Node.js |
| Bundler | 内置打包器，支持 ESM |
| Test Runner | 内置测试框架 |
| Binary Compile | 编译为平台二进制 |
| Performance | 比 Node.js 快 3-10 倍 |

**为什么不用 npm/yarn**:
```typescript
// AGENTS.md 约定
"Runtime: Bun only — never use npm/yarn"
"TypeScript: strict mode, ESNext, bundler moduleResolution, bun-types (never @types/node)"
```

**Bun-specific APIs**:
```typescript
// shebang
#!/usr/bin/env bun

// shell helper
import { $ } from "bun"
await $`bun build ...`

// runtime info
Bun.version
```

**关键文件**:
- `package.json` — bun-types devDependency
- `script/build-binaries.ts` — bun build --compile

#### 13.2 类型安全与护栏

**TypeScript 严格模式**:
```json
// tsconfig.json
{
  "compilerOptions": {
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true
  }
}
```

**反模式禁止**（AGENTS.md）:
```
Never use:
├── as any
├── @ts-ignore
├── @ts-expect-error
├── Empty catch blocks: catch(e) {}
└── Catch-all files: utils.ts, helpers.ts
```

**模块化规则**（`.sisyphus/rules/modular-code-enforcement.md`）:
```
Rule 1: index.ts is ENTRY POINT, not dumping ground
Rule 2: No catch-all files (utils.ts/service.ts banned)
Rule 3: Single Responsibility Principle
Rule 4: 200 LOC hard limit
```

**运行时守卫**:
- `write-existing-file-guard` — 防止意外覆盖
- `timeout` limits — 防止无限等待
- `output truncation` — 防止输出爆炸

#### 13.3 自动化测试架构

**测试模式**:
```typescript
// given/when/then 风格
describe("MyModule", () => {
  describe("#given initial state", () => {
    // 准备
  })
  describe("#when action performed", () => {
    // 执行
  })
  describe("#then expected result", () => {
    // 断言
  })
})
```

**测试组织**:
- Co-located: `*.test.ts` 与源文件同目录
- Preload: `bunfig.toml` 配置 `test-setup.ts`

**CI 测试分割**:
```yaml
# ci.yml
jobs:
  test:
    steps:
      # 1. Mock-heavy tests（隔离运行）
      - run: bun test src/plugin-handlers/
      - run: bun test src/hooks/atlas/
      
      # 2. 剩余测试（批量运行）
      - run: bun test
```

**测试覆盖**:
- 1268 TypeScript 文件
- 100+ 测试文件
- 覆盖：48 hooks, 26 tools, 11 agents

#### 13.4 构建与跨平台二进制

**构建流程**:
```bash
# Step 1: ESM build
bun build src/index.ts --outdir dist --target bun --format esm --external @ast-grep/napi

# Step 2: TypeScript declarations
tsc --emitDeclarationOnly

# Step 3: CLI build
bun build src/cli/index.ts --outdir dist/cli --target bun --format esm --external @ast-grep/napi

# Step 4: Schema generation
bun run script/build-schema.ts

# Step 5: Platform binaries
bun run script/build-binaries.ts
```

**12 平台支持**:
```
darwin-arm64, darwin-x64, darwin-x64-baseline
linux-x64, linux-x64-baseline, linux-arm64
linux-x64-musl, linux-x64-musl-baseline, linux-arm64-musl
windows-x64, windows-x64-baseline
```

**二进制编译**:
```typescript
// script/build-binaries.ts
bun build src/cli/index.ts \
  --compile \
  --minify \
  --sourcemap \
  --bytecode \
  --target=$TARGET \
  --outfile=$OUTPUT
```

**包结构**:
```
dist/
├── index.js          # ESM 入口
├── index.d.ts        # 类型声明
├── cli/
│   └── index.js      # CLI 入口
└── *.schema.json     # JSON Schema

packages/
├── darwin-x64/
│   └── bin/oh-my-opencode
├── linux-x64/
│   └── bin/oh-my-opencode
└── windows-x64/
    └── bin/oh-my-opencode.exe

bin/
└── oh-my-opencode.js  # 平台检测 wrapper
```

#### 13.5 CI/CD 架构

**6 个 Workflow**:
| Workflow | 触发 | 作用 |
|----------|------|------|
| ci.yml | push/PR to master/dev | 测试、类型检查、构建 |
| publish.yml | manual dispatch | 版本发布、npm 发布 |
| publish-platform.yml | called by publish | 平台二进制构建 |
| sisyphus-agent.yml | @mention / dispatch | AI agent 处理 |
| cla.yml | issue_comment/PR | CLA 签署 |
| lint-workflows.yml | push to .github/ | workflow 检查 |

**CI 门禁**:
```yaml
# ci.yml gates
jobs:
  block-master-pr:   # 阻止直接 PR 到 master
  test:              # 测试（隔离 + 批量）
  typecheck:         # 类型检查
  build:             # 构建 + schema 自动提交
  draft-release:     # dev 推送时创建草稿发布
```

**发布流程**:
```
1. 版本计算
2. 检查是否已发布
3. 更新 package.json 版本
4. 构建主包
5. 发布到 npm
6. 触发平台二进制构建
7. 创建 GitHub Release
8. 合并到 master
```

#### 13.6 从零搭建清单

**初始化**:
```bash
# 1. 创建项目
mkdir my-plugin && cd my-plugin
bun init

# 2. 安装依赖
bun add zod @ast-grep/napi
bun add -d bun-types typescript

# 3. 配置 TypeScript
# 复制 tsconfig.json 模板

# 4. 配置 Bun
# 创建 bunfig.toml
```

**项目结构**:
```
my-plugin/
├── src/
│   ├── index.ts           # 插件入口
│   ├── plugin-interface.ts # 插件接口
│   ├── agents/            # agent 定义
│   ├── hooks/             # hook 定义
│   ├── tools/             # tool 定义
│   └── config/            # 配置 schema
├── script/
│   ├── build-binaries.ts  # 二进制构建
│   └── build-schema.ts    # schema 生成
├── test-setup.ts          # 测试配置
├── bunfig.toml            # Bun 配置
├── tsconfig.json          # TypeScript 配置
├── package.json           # 包配置
└── AGENTS.md              # 项目规则
```

**核心实现**:
```typescript
// src/index.ts
export async function createMyPlugin(ctx: PluginContext) {
  // 1. 加载配置
  const config = await loadPluginConfig(ctx.directory, ctx)
  
  // 2. 创建 managers
  const managers = createManagers(config)
  
  // 3. 创建 tools
  const tools = createTools(managers)
  
  // 4. 创建 hooks
  const hooks = createHooks(tools)
  
  // 5. 创建插件接口
  return createPluginInterface({ tools, hooks })
}
```

**设计方案**:
- 解决了：可复制的工程模板
- 牺牲了：需要理解 Bun 生态

**迁移指南**:
- 从简单结构开始
- 逐步添加 agents/hooks/tools
- 配置 CI/CD

**最小实验**:
- 创建最小可运行插件
- 配置测试和构建
- 发布到 npm

---

## 附录

### A. 关键文件速查表

| 功能 | 关键文件 |
|------|---------|
| 初始化流程 | `src/index.ts`, `AGENTS.md` |
| Agent 定义 | `src/agents/`, `src/tools/delegate-task/constants.ts` |
| 任务编排 | `src/hooks/intent-gate/`, `src/agents/prometheus/` |
| 上下文管理 | `src/hooks/context-window-monitor/`, `src/hooks/preemptive-compaction/` |
| 工具注册 | `src/plugin/tool-registry.ts` |
| MCP 集成 | `src/mcp/` |
| Hooks | `src/hooks/` (48个) |
| 权限配置 | `src/plugin-config.ts` |
| 测试配置 | `bunfig.toml`, `test-setup.ts` |
| 构建配置 | `package.json`, `tsconfig.json` |
| CI/CD | `.github/workflows/` |
| 二进制构建 | `script/build-binaries.ts` |

### B. 术语表

| 术语 | 定义 |
|------|------|
| Agent | 执行特定角色的 AI 实体 |
| Hook | 生命周期拦截点 |
| Tool | 可调用的原子能力 |
| MCP | Model Context Protocol，外部服务集成 |
| Intent Gate | 意图门，判断任务复杂度 |
| Compaction | 上下文压缩 |
| Fallback | 降级策略 |
| Session Recovery | 会话恢复 |
| Verification | 验收验证 |

### C. 学习资源

- 项目 AGENTS.md — 系统总览
- OpenCode 文档 — 底座能力
- Bun 文档 — https://bun.sh/docs
- Zod 文档 — https://zod.dev

---

**系列完结** | 祝学习愉快！