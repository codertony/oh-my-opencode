# Oh My OpenAgent 学习文档集

> 生成日期：2026-03-28 | 基于 dev 分支 | 开发者架构文档

---

## 文档分类

### 📘 用户文档
- **[用户指南](../docs/guide/)** - 安装、快速开始、功能参考
- **[配置参考](../docs/reference/configuration.md)** - 配置选项详解

### 📗 架构文档 (learnCode/)
- **[00-架构概览](./learnCode/00-架构概览/)** - 三层架构、OpenCode vs OMO、初始化流程
- **[01-Harness 架构](./learnCode/01-Harness 架构/)** - Harness 职责、工具层、权限层
- **[02-核心 Agent](./learnCode/02-核心 Agent/)** - 主要 Agent、咨询 Agent、Agent 编排
- **[03-Hook 系统](./learnCode/03-Hook 系统/)** - Hook 分层、Session/Tool Guard/Transform Hooks
- **[04-工具系统](./learnCode/04-工具系统/)** - 工具注册、文件操作、代码智能
- **[05-功能模块](./learnCode/05-功能模块/)** - 后台 Agent、Tmux 集成、Skill 系统
- **[06-上下文管理](./learnCode/06-上下文管理/)** - 会话上下文、上下文注入、压缩
- **[07-配置系统](./learnCode/07-配置系统/)** - 配置加载、Schema 系统、模型解析
- **[08-共享工具](./learnCode/08-共享工具/)** - 模型解析工具、Tmux 工具、配置工具
- **[09-集成工作流](./learnCode/09-集成工作流/)** - 初始化序列、Hook 执行顺序、工具生命周期
- **[10-测试约定](./learnCode/10-测试约定/)** - 测试设置、Given-When-Then、Mock 隔离

**总计**: 49 个架构文档

### 📙 学习路线
| 文件 | 说明 |
|------|------|
| `start.md` | 原始学习路线指导文档（用户提供） |
| `learning-roadmap.md` | **学习路线图** - 8 个核心问题 + 8 个学习阶段 |
| `article-outline.md` | **系列文章大纲** - 13 篇文章详细大纲 |
| `engineering-architecture.md` | **工程架构深度解析** - 技术栈、测试、构建、CI/CD |
| `README.md` | 本文件 - 文档索引 |

---

## 学习路径建议

### 快速开始

```
1. 阅读 start.md 了解学习理念
2. 阅读 learning-roadmap.md 建立整体认知
3. 按顺序阅读 article-outline.md 中的文章大纲
4. 参考 engineering-architecture.md 了解工程实现
```

### 学习时间规划

| 阶段 | 内容 | 建议时间 |
|------|------|---------|
| 阶段 0 | 建立坐标系 | 1-2 天 |
| 阶段 1 | 学习编排 | 3-5 天 |
| 阶段 2 | 研究角色边界 | 2-3 天 |
| 阶段 3 | 研究上下文系统 | 3-4 天 |
| 阶段 4 | 研究记忆与恢复 | 2-3 天 |
| 阶段 5 | 研究扩展能力 | 3-5 天 |
| 阶段 6 | 研究验证与稳定性 | 2-3 天 |
| 阶段 7 | 研究权限治理 | 2-3 天 |
| 阶段 8 | 研究工程架构 | 3-5 天 |
| **总计** | | **21-33 天** |

---

## 核心知识点速查

### 8 个核心问题

1. 一个复杂 Agent 系统到底在解决什么问题？
2. 任务是如何从"用户需求"变成"可执行工作流"的？
3. 多 Agent 为什么能比单 Agent 更稳定？
4. 上下文为什么会失控，系统如何控制它？
5. "记忆"到底应该分成哪几层？
6. 工具体系是如何决定 Agent 上限的？
7. Agent 为什么"能跑起来"不等于"能交付"？
8. 安全和权限为什么是 Agent 工程的硬门槛？

### OpenCode vs OMO 能力边界

**关键理解**：OpenCode 只提供 Session 基础设施，OMO 构建了完整的多 Agent 编排层

| OpenCode 原生 | OMO 编排增强 |
|---------------|-------------|
| Session 创建/管理 | Task 系统 (`task()` 工具) |
| Parent-child 关系 | Task ID 生成与管理 |
| Async/sync 消息 | 并发控制 (5/model) |
| 事件系统 | 后台/同步任务执行 |
| 插件钩子 | Category 路由 (8 个内置) |
| — | Agent 委派机制 |
| — | 上下文压缩状态恢复 |
| — | Todo 强制执行 |

**详细说明**：参见 `article-outline.md` 第2篇「OpenCode 底座与 OMO 编排层：边界在哪」

### 11 个 Agent 角色

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

### 48 个 Hooks 分类

```
Session Hooks (23)
├── chat.message
├── chat.params
├── chat.headers
└── ...

Tool-Guard Hooks (12)
├── write-existing-file-guard
├── file-guard
└── ...

Transform Hooks (4)
├── messages-transform
└── ...

Continuation Hooks (7)
├── todo-continuation
├── session-recovery
└── ...

Skill Hooks (2)
└── ...
```

### 26 个 Tools

```
文件操作
├── read, write, edit
├── glob, grep
└── look_at

任务执行
├── task (delegate)
├── bash
└── interactive_bash

会话管理
├── session_list, session_read
├── session_search, session_info
└── background_output, background_cancel

LSP 工具
├── lsp_diagnostics, lsp_symbols
├── lsp_goto_definition, lsp_find_references
├── lsp_prepare_rename, lsp_rename
└── ast_grep_search, ast_grep_replace

网络工具
├── webfetch
├── websearch_web_search_exa
├── context7_resolve-library-id
├── context7_query-docs
└── grep_app_searchGitHub

其他
├── skill, skill_mcp
├── todowrite
├── question
└── chrome-devtools_*
```

---

## 工程架构关键点

### 技术栈

- **Runtime**: Bun（不用 Node.js）
- **Language**: TypeScript strict mode
- **Schema**: Zod v4
- **Test**: Bun test + given/when/then
- **Build**: Bun build + tsc declarations
- **CI/CD**: GitHub Actions

### 测试策略

```
CI 测试分割：
├── Mock-heavy tests（隔离运行）
│   ├── src/plugin-handlers/
│   ├── src/hooks/atlas/
│   ├── src/features/tmux-subagent/
│   ├── src/tools/ast-grep/
│   └── src/tools/lsp/
└── Remaining tests（批量运行）
```

### 构建流程

```
1. ESM Build (bun build)
2. TypeScript Declarations (tsc)
3. CLI Build
4. Schema Generation
5. Platform Binaries (12 platforms)
```

### CI/CD 门禁

```
ci.yml:
├── block-master-pr    # 阻止直接 PR 到 master
├── test               # 测试
├── typecheck          # 类型检查
├── build              # 构建 + schema 自动提交
└── draft-release      # 草稿发布

publish.yml:
├── calculate-version  # 版本计算
├── publish-main       # 发布主包
└── trigger-platform   # 触发平台二进制构建
```

---

## 关键源码文件索引

### 核心入口

| 文件 | 说明 |
|------|------|
| `src/index.ts` | 插件入口（5步初始化） |
| `AGENTS.md` | 项目规则与总览 |
| `package.json` | 依赖与构建配置 |

### Agent 定义

| 文件 | 说明 |
|------|------|
| `src/agents/` | 所有 agent 目录 |
| `src/tools/delegate-task/constants.ts` | agent categories |

### 编排系统

| 文件 | 说明 |
|------|------|
| `src/hooks/intent-gate/` | Intent Gate |
| `src/agents/prometheus/` | 规划 agent |
| `src/agents/metis/` | 预规划分析 |

### 上下文管理

| 文件 | 说明 |
|------|------|
| `src/hooks/context-window-monitor/` | 上下文监控 |
| `src/hooks/preemptive-compaction/` | 压缩 |
| `src/hooks/session-recovery/` | 恢复 |

### 工具与扩展

| 文件 | 说明 |
|------|------|
| `src/plugin/tool-registry.ts` | 工具注册 |
| `src/hooks/` | 所有 hooks |
| `src/mcp/` | MCP 集成 |
| `src/features/builtin-skills/` | skills |

### 配置与权限

| 文件 | 说明 |
|------|------|
| `src/plugin-config.ts` | 配置加载 |
| `src/config/schema/` | Zod schemas |

### 测试与构建

| 文件 | 说明 |
|------|------|
| `bunfig.toml` | Bun 测试配置 |
| `test-setup.ts` | 测试初始化 |
| `script/build-binaries.ts` | 二进制构建 |
| `.github/workflows/` | CI/CD |

---

## 从零搭建自己的插件

### 最小步骤

```bash
# 1. 初始化
mkdir my-plugin && cd my-plugin
bun init

# 2. 安装依赖
bun add zod @ast-grep/napi
bun add -d bun-types typescript

# 3. 创建结构
mkdir -p src/{agents,hooks,tools,config}

# 4. 复制配置
# - tsconfig.json
# - bunfig.toml
# - package.json

# 5. 实现入口
# src/index.ts

# 6. 测试
bun test

# 7. 构建
bun run build

# 8. 发布
npm publish --access public
```

### 详细指南

请参考 `engineering-architecture.md` 第六章"从零搭建自己的插件工程"。

---

## 学习资源

- **项目文档**: `AGENTS.md`
- **Bun 文档**: https://bun.sh/docs
- **Zod 文档**: https://zod.dev
- **TypeScript 文档**: https://www.typescriptlang.org/docs

---

**祝学习愉快！**