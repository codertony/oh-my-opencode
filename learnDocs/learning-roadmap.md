# Oh My OpenAgent 学习路线图

> **基于项目版本**: 2026-03-28 | **Commit**: current dev branch
> 
> 本路线图基于 `learnDocs/start.md` 中的指导文档，结合项目实际架构设计，旨在帮助读者从"源码理解"走向"Agent工程师"。

---

## 一、核心学习问题（8个关键问题）

根据项目架构分析，我们收敛出8个核心问题，所有学习动作围绕这些问题展开：

### 1. 一个复杂 Agent 系统到底在解决什么问题？

**问题本质**: OMO 不是"多几个 prompt"，而是把单个 coding agent 提升为一个可协作的开发团队。

**核心目标**: 让复杂任务能被**规划、委派、执行、验证和恢复**。

**关键文件**:
- `src/index.ts` — 插件初始化流程（5步）
- `AGENTS.md` — 系统总览，定义了 runtime + orchestration + workflow policy 三层

**学习产出**: 理解为什么复杂开发任务不能只靠一个 agent 一把梭

---

### 2. 任务是如何从"用户需求"变成"可执行工作流"的？

**问题本质**: 理解 orchestration 的核心链路

**关键子问题**:
- 什么情况下只需要直接 prompt？
- 什么情况下要先 plan？
- plan 如何变成待执行任务？
- 任务粒度如何控制？
- 谁负责验收标准？

**关键文件**:
- `src/hooks/intent-gate/` — Intent Gate 实现（复杂度判断、路由决策）
- `src/agents/prometheus/` — 规划 agent
- `src/agents/metis/` — 预规划分析
- `src/hooks/todo-continuation/` — Todo 管理与任务追踪

**学习产出**: 掌握任务编排的完整链路

---

### 3. 多 Agent 为什么能比单 Agent 更稳定？

**问题本质**: 角色边界和职责隔离

**关键子问题**:
- 这些 agent 的职责边界是什么？
- 哪些是只读顾问，哪些能写代码？
- 哪些能继续委派，哪些不能？
- 为什么 orchestrator 和 worker 不能混成一个？

**关键文件**:
- `src/agents/` — 11个 agent 定义
- `src/agents/builtin-agents/` — agent 工厂函数
- `src/tools/delegate-task/constants.ts` — agent categories 和 model requirements

**Agent 职责边界表**:

| Agent | 角色 | 工具权限 | 可委派 |
|-------|------|---------|--------|
| Sisyphus | 主编排者 | 全部 | 是 |
| Oracle | 只读顾问 | 读取类 | 否 |
| Librarian | 文档检索 | 读取类 | 否 |
| Explore | 代码搜索 | 读取类 | 否 |
| Atlas | 执行者 | 读取类 | 否 |
| Prometheus | 规划者 | 读取类 | 是 |
| Metis | 预规划分析 | 读取类 | 否 |
| Momus | 审查者 | 读取类 | 否 |
| Hephaestus | 实现者 | 全部 | 是 |
| Sisyphus-Junior | 子任务执行 | 部分 | 是 |

**学习产出**: 理解"复杂 Agent 不是堆人头，而是职责隔离"

---

### 4. 上下文为什么会失控，系统如何控制它？

**问题本质**: Context overload 是复杂 Agent 的核心挑战

**关键子问题**:
- AGENTS.md / README / rules 是怎么注入的？
- 目录级上下文如何叠加？
- context window 如何监控？
- compaction 如何做？
- 长会话如何恢复？

**关键文件**:
- `src/plugin/chat-message.ts` — 上下文注入
- `src/hooks/context-window-monitor/` — 上下文监控
- `src/hooks/preemptive-compaction/` — 预压缩机制
- `src/hooks/session-recovery/` — 会话恢复

**学习产出**: 掌握长任务不丢上下文的技术方案

---

### 5. "记忆"到底应该分成哪几层？

**问题本质**: 短期上下文 → 长期知识库 的闭环

**三层记忆模型**:

| 层级 | 内容 | 实现 |
|------|------|------|
| 运行时记忆 | 当前任务执行上下文 | 会话状态、压缩、恢复 |
| 阶段性沉淀 | learnings / decisions / issues / verification | notepad、verification记录 |
| 项目长期知识 | AGENTS.md、skills、规则、约定 | 层级AGENTS.md、skills系统 |

**关键文件**:
- `src/features/builtin-skills/` — skills 系统
- `AGENTS.md` — 项目规则注入
- `src/hooks/handoff/` — 会话交接

**学习产出**: 设计自己的 Agent 记忆体系

---

### 6. 工具体系是如何决定 Agent 上限的？

**问题本质**: 扩展边界决定系统能力天花板

**关键子问题**:
- 原生工具够不够？
- 什么能力适合做 custom tool？
- 什么能力适合做 plugin？
- 什么能力适合接 MCP？
- 哪些能力不该塞进主编排 prompt？

**三层 MCP 系统**:

| 层级 | 来源 | 机制 |
|------|------|------|
| Built-in | `src/mcp/` | 3个远程HTTP: websearch, context7, grep_app |
| Claude Code | `.mcp.json` | `${VAR}` 环境变量扩展 |
| Skill-embedded | SKILL.md YAML | SkillMcpManager 管理 |

**关键文件**:
- `src/plugin/tool-registry.ts` — 26个工具注册
- `src/mcp/` — 内置 MCP
- `src/features/builtin-skills/` — skills 定义
- `src/features/builtin-commands/` — commands 定义

**学习产出**: 掌握扩展能力的选择原则

---

### 7. Agent 为什么"能跑起来"不等于"能交付"？

**问题本质**: 验证闭环是生产化的核心

**关键子问题**:
- 验收标准怎么定义？
- verification 怎么嵌进流程？
- review 在哪一步做？
- fallback 何时触发？
- 会话坏了怎么恢复？

**验证机制**:
- **Clarity检查**: 任务是否清晰
- **Verification检查**: 是否有验证手段
- **Context检查**: 上下文是否充分
- **Big Picture检查**: 是否理解全局

**关键文件**:
- `src/agents/momus/` — reviewer agent
- `src/hooks/verification/` — 验证机制
- `src/hooks/model-fallback/` — 模型降级
- `src/hooks/session-recovery/` — 会话恢复
- `src/tools/lsp/diagnostics-tool.ts` — LSP诊断

**学习产出**: 建立稳定交付的验收闭环

---

### 8. 安全和权限为什么是 Agent 工程的硬门槛？

**问题本质**: 能 demo ≠ 能日常使用

**关键子问题**:
- 哪些动作可以自动执行？
- 哪些必须 ask？
- 哪些必须 deny？
- 哪些 agent 只能读不能写？
- 如何防止越权修改？

**权限级别**:
- `allow` — 自动执行
- `ask` — 需要确认
- `deny` — 禁止执行

**关键文件**:
- `src/plugin-config.ts` — 权限配置
- `src/hooks/write-existing-file-guard/` — 写文件守卫
- `src/hooks/tool-output-truncator/` — 输出截断

**学习产出**: 设计安全可控的 Agent 系统

---

## 二、学习研究路线图（8个阶段）

### 阶段 0：建立坐标系（1-2天）

**目标**: 画出系统地图，理解分层架构

**任务清单**:
- [ ] 理解 OpenCode 是底座，OMO 是编排增强层
- [ ] 画出系统分层图：底座能力层、编排层、执行层、扩展层、治理层
- [ ] 固定研究基线：记录当前 repo 分支/commit

**阶段产出**:
- 1 张系统架构图
- 1 篇总览文章
- 1 份术语表

**关键文件**:
- `AGENTS.md` — 系统总览
- `src/index.ts` — 初始化流程
- `src/plugin-interface.ts` — 插件接口

---

### 阶段 1：学习"编排"（3-5天）

**目标**: 理解任务如何流动

**核心问题**:
- 用户请求如何进入系统？
- 谁判断复杂度？
- 谁做规划？
- 谁做审查？
- 谁做执行？
- 谁做收尾验证？

**关键文件**:
- `src/hooks/intent-gate/` — Intent Gate
- `src/agents/prometheus/` — 规划 agent
- `src/agents/metis/` — 预规划分析
- `src/agents/momus/` — 审查 agent

**阶段产出**:
- 任务流转图
- 一篇"复杂 Agent 为什么必须分层"的文章
- 一个最小版任务分派流程图

---

### 阶段 2：研究"角色边界"（2-3天）

**目标**: 理解 agent 职责隔离

**核心问题**:
- 为什么有些 agent 只读，有些可写？
- 为什么有些 agent 不能再委派？

**关键文件**:
- `src/agents/` — 所有 agent 定义
- `src/tools/delegate-task/constants.ts` — agent categories

**阶段产出**:
- 一张"agent 职责边界表"
- 一篇"复杂 Agent 不是堆人头，而是职责隔离"的文章
- 一个开发效率套件角色设计草案

---

### 阶段 3：研究"上下文系统"（3-4天）

**目标**: 掌握上下文管理机制

**核心问题**:
- AGENTS.md 为什么重要？
- 项目规则和个人规则怎么叠加？
- README 与目录上下文的注入时机是什么？
- 长任务怎么不丢上下文？
- compaction 会损失什么？

**关键文件**:
- `src/plugin/chat-message.ts` — 上下文注入
- `src/hooks/context-window-monitor/` — 监控
- `src/hooks/preemptive-compaction/` — 压缩

**阶段产出**:
- 一篇"Agent 的上下文不是 prompt，而是分层知识系统"
- 一个示例项目的 AGENTS.md 设计模板
- 一个"长任务上下文漂移排查清单"

---

### 阶段 4：研究"记忆、沉淀与恢复"（2-3天）

**目标**: 理解三层记忆模型

**核心问题**:
- 哪些信息只应留在会话里？
- 哪些应该写入 notepad / verification / decisions？
- 哪些应该回写项目知识？
- 会话中断后如何恢复？

**关键文件**:
- `src/hooks/handoff/` — 会话交接
- `src/hooks/session-recovery/` — 会话恢复
- `src/features/builtin-skills/` — skills 系统

**阶段产出**:
- 一篇"Agent 记忆不等于向量库"的文章
- 一个"短期记忆 / 长期知识"分层模型
- 一个 session handoff 模板

---

### 阶段 5：研究"扩展能力"（3-5天）

**目标**: 开始做自己的开发套件

**需求分类**:
1. 通过规则能解决的 → AGENTS.md
2. 通过 plugin hook 能解决的 → hooks/
3. 通过 custom tool / MCP 能解决的 → tools/, mcp/

**关键文件**:
- `src/plugin/tool-registry.ts` — 工具注册
- `src/hooks/` — 所有 hooks
- `src/mcp/` — MCP 集成
- `src/features/builtin-skills/` — skills

**建议实现**:
- PR 评审助手
- 代码库检索与模式归纳助手
- 任务分解与验收清单生成器

**阶段产出**:
- 一篇"哪些能力该放在 plugin，哪些该放在 tool"
- 1-2 个最小可用增强模块
- 一个 Agent 套件 v0.1

---

### 阶段 6：研究"验证、评测与稳定性"（2-3天）

**目标**: 把"能跑"变成"能用"

**核心问题**:
- 一个任务什么时候算完成？
- 如何自动检查"不完整完成"？
- fallback 的代价和收益是什么？
- 稳定性如何观测？

**评测指标**:
- 任务完成率
- 首轮成功率
- 平均修正轮次
- 失败原因分类
- token / latency / cost
- 人工接管率

**关键文件**:
- `src/agents/momus/` — reviewer
- `src/hooks/verification/` — 验证
- `src/hooks/model-fallback/` — 降级

**阶段产出**:
- 一篇"Agent 系统为什么必须有验收闭环"
- 一套评测表
- Agent 套件 v0.2 评测报告

---

### 阶段 7：研究"权限治理与产品化落地"（2-3天）

**目标**: 让 Agent 真正进入日常开发

**核心问题**:
- 哪些任务可自动执行？
- 哪些任务必须审批？
- 哪些任务只能建议不能落地？
- 团队协作时如何共享规则和能力？

**关键文件**:
- `src/plugin-config.ts` — 权限配置
- `src/hooks/write-existing-file-guard/` — 写守卫

**阶段产出**:
- 一篇"开发 Agent 从 demo 到生产，差在权限和治理"
- Agent 套件 v1.0 使用规范

---

### 阶段 8：研究"工程架构"（3-5天）

**目标**: 掌握如何搭建自己的插件工程

**核心问题**:
- 为什么选择 Bun？
- 如何保证交付稳定性与护栏？
- 如何进行自动化测试？
- 如何构建跨平台二进制？
- 如何搭建 CI/CD？

**关键文件**:
- `package.json` — 构建配置
- `tsconfig.json` — TypeScript 配置
- `bunfig.toml` — Bun 测试配置
- `.github/workflows/` — CI/CD
- `script/build-binaries.ts` — 跨平台编译

**阶段产出**:
- 一篇"如何搭建自己的领域插件工程架构"
- 工程脚手架模板

---

## 三、学习优先级建议

```
先学 任务编排 → 再学 上下文系统 → 再学 角色边界
     ↓                ↓                ↓
然后进入 扩展能力 → 最后补 验证、权限和稳定性
     ↓
最终掌握 工程架构
```

**原因**: 如果先看插件和工具，很容易陷入"会接东西"；但如果没理解编排、上下文和验证，做出来的只是能调用工具的 prompt 工程，不是复杂 Agent 系统。

---

## 四、版本漂移控制

项目正在快速演进，建议：

1. **固定分析基线**: 每篇文章注明基于哪个 tag / release / commit
2. **区分稳定机制和易变实现**:
   - 稳定机制：分层编排、上下文压缩、fallback、权限隔离
   - 易变实现：具体 agent 名称、默认模型、某个 hook 名称、某条 fallback chain

---

## 五、文章模板（每篇固定结构）

每篇文章都按这 6 段写：

1. **这篇要回答什么问题**
2. **源码里这个问题出现在哪里**
3. **它当前的设计方案是什么**
4. **这套方案解决了什么，牺牲了什么**
5. **如果我要做自己的开发 Agent 套件，我会怎么迁移**
6. **一个最小实验 / 小改造**

这样写，文章就不会沦为"源码导览"，而会变成"工程理解 + 方法迁移"。