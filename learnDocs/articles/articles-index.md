# Oh My OpenAgent 源码解析系列文章索引

> 《Oh My OpenAgent 源码解析：从复杂 Agent 学习到开发效率套件落地》
> 
> 基于项目版本: 2026-03-28 | 共13篇

---

## 系列总览

本系列共 **13篇文章**，按照"由浅入深、问题驱动"的原则组织，帮助读者从"源码理解"走向"Agent 工程师"。

---

## 第一部分：建立认知框架

### 第1篇: 为什么 Oh My OpenAgent 值得作为复杂 Agent 学习样本
**文件**: `article-01-why-learn-omo.md`

**要回答的问题**:
- 它解决的到底是什么问题？
- 它和 OpenCode 的关系是什么？
- 为什么它值得作为复杂 Agent 学习样本？

**核心内容**:
- 五大核心挑战：Context overload、Cognitive drift、Verification gaps、Task decomposition、Multi-agent coordination
- 分层架构：底座层 vs 编排层
- 角色分工：Orchestrator、Planner、Executor、Reviewer、Specialist
- 稳定机制：Verification、Fallback、Session Recovery

**源码焦点**: `AGENTS.md`, `src/index.ts`

---

### 第2篇: OpenCode 底座与 OMO 编排层：边界在哪
**文件**: `article-02-opencode-vs-omo.md`

**要回答的问题**:
- OpenCode 提供什么能力？
- OMO 在此之上增强了什么？
- 哪些是继承，哪些是增强？

**核心内容**:
- OpenCode 底座：Tool 系统、Rule 系统、Plugin 系统、MCP、LSP、SDK
- OMO 增强：多 Agent 编排、Intent Gate、Task Delegation、Context Management、Verification、Fallback
- 三层 MCP 系统：Built-in → Claude Code → Skill-embedded

**源码焦点**: `src/plugin-interface.ts`, `src/mcp/`, `src/plugin/tool-registry.ts`

---

### 第3篇: 复杂任务为什么必须先规划再执行
**文件**: `article-03-task-planning.md`

**要回答的问题**:
- 复杂度判断如何做？
- 规划入口在哪里？
- 任务拆分粒度如何控制？
- 验收条件如何定义？

**核心内容**:
- Intent Gate：简单任务直接执行，复杂任务进入规划
- 规划流程：Metis → Prometheus → Momus → 执行
- 任务拆分四原则：单一职责、可验证、可委派、粒度适中

**源码焦点**: `src/hooks/intent-gate/`, `src/agents/prometheus/`, `src/agents/metis/`

---

## 第二部分：吃透任务编排

### 第4篇: Planner / Reviewer / Executor 的角色分工设计
**文件**: `article-04-role-separation.md`

**要回答的问题**:
- Planner 负责什么？
- Reviewer 为什么必要？
- Executor 为什么必须受约束？
- Intelligence 在系统里，而不是在某个 Agent 里？

**核心内容**:
- 角色职责矩阵：Metis(预规划)、Prometheus(规划)、Momus(审查)、Hephaestus(执行)、Sisyphus(编排)
- 为什么不能合并成"超级 Agent"：职责不清、缺乏制衡、上下文膨胀、难以调试
- Intelligence 分布：系统智能 > 单个 Agent 智能之和

**源码焦点**: `src/agents/` 所有 agent 定义

---

### 第5篇: 多 Agent 委派不是炫技，而是稳定性交换
**文件**: `article-05-multi-agent-delegation.md`

**要回答的问题**:
- 并行的价值是什么？
- 专职角色的价值是什么？
- 什么时候不该多 agent？

**核心内容**:
- 并行价值：时间效率、容错性、上下文隔离
- 专职角色：只读顾问、规划者、执行者、审查者
- 什么时候不该多 Agent：任务简单、依赖强、上下文共享需求高、成本敏感

**源码焦点**: `src/tools/delegate-task/`, `src/tools/delegate-task/constants.ts`

---

## 第三部分：吃透上下文、记忆与恢复

### 第6篇: AGENTS.md、rules 与目录上下文注入机制
**文件**: `article-06-agents-md-context.md`

**要回答的问题**:
- project / global 规则如何生效？
- 目录级上下文如何叠加？
- 规则优先级是什么？
- 为什么要 commit AGENTS.md？

**核心内容**:
- AGENTS.md 层级：Global → Project → Directory
- 注入时机：初始化阶段 + 每条消息
- 规则优先级：Project 覆盖 Global，Directory 补充 Project
- 为什么要 commit：团队共享、版本控制、环境一致性

**源码焦点**: `AGENTS.md`, `src/plugin/chat-message.ts`, `src/plugin-config.ts`

---

### 第7篇: Context overload、压缩与会话恢复
**文件**: `article-07-context-management.md`

**要回答的问题**:
- context monitor 如何工作？
- preemptive compaction 何时触发？
- compaction 时保留什么、丢弃什么？
- 什么信息不该靠上下文硬扛？

**核心内容**:
- Context Window Monitor：监控指标 currentTokens/maxTokens/usageRatio
- Preemptive Compaction：阈值 0.8，保留目标/决策/验证标准，压缩历史对话，丢弃冗余信息
- Session Recovery：多种恢复策略（压缩、等待、切换模型、降低复杂度）
- 不该硬扛的信息：项目知识 → AGENTS.md，历史决策 → notepad，团队规范 → rules

**源码焦点**: `src/hooks/context-window-monitor/`, `src/hooks/preemptive-compaction/`, `src/hooks/session-recovery/`

---

### 第8篇: 复杂 Agent 的记忆体系：运行时、沉淀层、长期知识
**文件**: `article-08-memory-system.md`

**要回答的问题**:
- 哪些信息只应留在会话里？
- 哪些应该写入 notepad / verification / decisions？
- 哪些应该回写项目知识？
- 会话中断后如何恢复？

**核心内容**:
- 三层记忆模型：运行时记忆(Session) → 阶段性沉淀(Learnings) → 项目长期知识(Knowledge)
- 信息流转：任务执行 → 生成 learnings → 沉淀 → 回写
- Session Handoff：会话交接时提取关键信息
- 记忆边界表：类型、存储位置、生命周期

**源码焦点**: `src/hooks/handoff/`, `src/hooks/session-recovery/`, `src/features/builtin-skills/`

---

## 第四部分：吃透工具与扩展

### 第9篇: Plugin、custom tool、MCP：能力扩展的三种路线
**文件**: `article-09-extension-architecture.md`

**要回答的问题**:
- 规则层 vs plugin 层 vs tool 层 vs MCP 层的区别？
- 哪类需求适合放哪一层？

**核心内容**:
- 四层扩展架构：Rules(AGENTS.md) → Plugin(Hooks) → Tool(Tools) → MCP(MCP)
- 选择决策树：静态规则 → AGENTS.md，生命周期拦截 → Plugin，原子操作 → Tool，外部服务 → MCP
- 48个 Hooks 分类：Session(23)、Tool-Guard(12)、Transform(4)、Continuation(7)、Skill(2)

**源码焦点**: `src/plugin/tool-registry.ts`, `src/hooks/`, `src/mcp/`, `src/features/builtin-skills/`

---

### 第10篇: 做一个开发效率增强模块：从源码理解到实战改造
**文件**: `article-10-practical-module.md`

**要回答的问题**:
- 如何选择落地场景？
- 如何设计和实现？
- 如何验证效果？

**核心内容**:
- 设计步骤6步：定义场景 → 分析能力 → 选择扩展层 → 实现逻辑 → 编写测试 → 集成验证
- PR Review 助手示例：获取变更 → 分析变更 → 生成建议 → 输出报告
- 实现清单：创建 skill → 实现核心逻辑 → 编写测试 → 编写文档 → 集成测试

**源码焦点**: `src/features/builtin-skills/`, `src/features/builtin-commands/`

---

## 第五部分：吃透工程化交付

### 第11篇: 为什么复杂 Agent 不能没有 verification
**文件**: `article-11-verification.md`

**要回答的问题**:
- clarity / verification / context / big picture 各检查什么？
- 验收标准怎么定义？
- 自动验证和人工复核的边界？

**核心内容**:
- 四维验证：Clarity(清晰度)、Verification(可验证性)、Context(上下文)、Big Picture(全局视角)
- Acceptance Criteria 接口：功能完成、测试通过、代码质量、文档完整
- 自动验证 vs 人工复核：LSP/测试/构建 vs 设计/业务/边界/UX

**源码焦点**: `src/agents/momus/`, `src/hooks/verification/`, `src/tools/lsp/diagnostics-tool.ts`

---

### 第12篇: 权限、回退与恢复，Agent 系统怎么才能日常可用
**文件**: `article-12-permissions-governance.md`

**要回答的问题**:
- permissions 如何配置？
- tool restriction 如何生效？
- fallback 何时触发？
- session recovery 如何工作？

**核心内容**:
- 权限三级：allow(自动)、ask(需确认)、deny(禁止)
- Agent 级权限：不同 Agent 不同权限（Oracle只读、Hephaestus可写）
- Fallback 机制：模型不可用时的降级策略
- Recovery 流程：会话中断后的恢复机制

**源码焦点**: `src/plugin-config.ts`, `src/hooks/model-fallback/`, `src/hooks/session-recovery/`, `src/hooks/write-existing-file-guard/`

---

### 第13篇: 如何搭建自己的领域插件工程架构
**文件**: `article-13-engineering-architecture.md`

**要回答的问题**:
- 为什么选择 Bun？
- 如何保证交付稳定性与护栏？
- 如何进行自动化测试？
- 如何构建跨平台二进制？
- 如何搭建 CI/CD？
- 从零搭建全套工程需要什么？

**核心内容**:
- Bun 特性：runtime + bundler + test + compile，比 Node.js 快 3-10 倍
- TypeScript 严格模式：strict, noImplicitAny, strictNullChecks
- 测试架构：given/when/then 风格，CI 测试分割
- 构建流程：5步构建，12平台支持
- CI/CD：6个 workflow，自动化门禁
- 从零搭建清单：8步搭建完整工程

**源码焦点**: `package.json`, `tsconfig.json`, `bunfig.toml`, `.github/workflows/`, `script/build-binaries.ts`

---

## 学习路径建议

### 阅读顺序

```
第一阶段：建立认知（3篇）
├── 第1篇: 为什么学习
├── 第2篇: 底座与编排层边界
└── 第3篇: 任务规划

第二阶段：任务编排（2篇）
├── 第4篇: 角色分工
└── 第5篇: 多Agent委派

第三阶段：上下文与记忆（3篇）
├── 第6篇: AGENTS.md机制
├── 第7篇: Context管理
└── 第8篇: 记忆体系

第四阶段：工具与扩展（2篇）
├── 第9篇: 扩展架构
└── 第10篇: 实战模块

第五阶段：工程化（3篇）
├── 第11篇: Verification
├── 第12篇: 权限治理
└── 第13篇: 工程架构
```

### 时间规划

| 阶段 | 文章数 | 建议时间 |
|------|--------|---------|
| 第一阶段 | 3篇 | 2-3天 |
| 第二阶段 | 2篇 | 2天 |
| 第三阶段 | 3篇 | 3-4天 |
| 第四阶段 | 2篇 | 2-3天 |
| 第五阶段 | 3篇 | 3-4天 |
| **总计** | **13篇** | **12-16天** |

---

## 核心知识点速查

### 8个核心问题

1. 一个复杂 Agent 系统到底在解决什么问题？
2. 任务是如何从"用户需求"变成"可执行工作流"的？
3. 多 Agent 为什么能比单 Agent 更稳定？
4. 上下文为什么会失控，系统如何控制它？
5. "记忆"到底应该分成哪几层？
6. 工具体系是如何决定 Agent 上限的？
7. Agent 为什么"能跑起来"不等于"能交付"？
8. 安全和权限为什么是 Agent 工程的硬门槛？

### 11个 Agent 角色

| Agent | 角色 | 工具权限 |
|-------|------|---------|
| Sisyphus | 主编排者 | 全部 |
| Oracle | 只读顾问 | 读取类 |
| Librarian | 文档检索 | 读取类 |
| Explore | 代码搜索 | 读取类 |
| Atlas | 执行者 | 读取类 |
| Prometheus | 规划者 | 读取类 |
| Metis | 预规划分析 | 读取类 |
| Momus | 审查者 | 读取类 |
| Hephaestus | 实现者 | 全部 |
| Sisyphus-Junior | 子任务执行 | 部分 |
| Multimodal-Looker | 多模态分析 | 读取类 |

### 48个 Hooks 分类

- Session Hooks (23): chat.message, chat.params, session.created...
- Tool-Guard Hooks (12): write-existing-file-guard, file-guard...
- Transform Hooks (4): messages-transform, tool-result-transform...
- Continuation Hooks (7): todo-continuation, session-recovery...
- Skill Hooks (2): skill.before-execute, skill.after-execute

---

## 关键文件索引

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

---

## 下一步

完成本系列学习后，建议：

1. **实践**: 基于学到的知识，搭建自己的 Agent 套件
2. **深入**: 阅读 OMO 源码，理解更多细节
3. **贡献**: 参与 OMO 开源项目，贡献代码
4. **分享**: 把学到的知识分享给团队

### 资源

- **项目地址**: https://github.com/code-yeongyu/oh-my-openagent
- **文档**: `AGENTS.md`, `README.md`
- **社区**: Discord, GitHub Issues

---

**系列完结** | 祝学习愉快！
