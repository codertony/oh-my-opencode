# Skill Hooks：2 个技能激活钩子

> 所属模块：03-Hook 系统 | 三层模型：Layer 3 | 优先级：P1

---

## 架构位置

```
┌─────────────────────────────────────────────────────────────┐
│                    OpenCode Plugin Layer                    │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────┐   │
│  │              Layer 3: Hook System                    │   │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌──────────┐  │   │
│  │  │ Session │ │ Tool    │ │Transform│ │Continuation│  │   │
│  │  │ (23)    │ │ Guard   │ │ (4)     │ │ (7)      │  │   │
│  │  │         │ │ (12)    │ │         │ │          │  │   │
│  │  └─────────┘ └─────────┘ └─────────┘ └──────────┘  │   │
│  │                    ┌─────────┐                      │   │
│  │                    │ Skill   │                      │   │
│  │                    │ (2)     │                      │   │
│  │                    └─────────┘                      │   │
│  └─────────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────────┤
│  Layer 2: Tool Registry (26 tools)                          │
├─────────────────────────────────────────────────────────────┤
│  Layer 1: Agent System (11 agents)                          │
└─────────────────────────────────────────────────────────────┘
```

Skill Hooks 位于 Hook 系统的最顶层（Layer 3），是 48 个钩子中的最后 2 个。它们专注于技能的发现、提醒和自动激活，连接用户输入与 Skill 系统的强大能力。

---

## 核心职责

Skill Hooks 的核心职责是在适当的时机提醒和激活 Skill 功能，弥合用户意图与 Skill 能力之间的差距。

**技能提醒**：当 Agent 执行可委托任务时（如编辑文件、执行 Bash 命令），`categorySkillReminder` 钩子会检测并提醒 Agent 使用类别委托和 Skill 加载。这确保 Agent 充分利用可用的专业技能，而不是重复造轮子。

**命令自动检测**：`autoSlashCommand` 钩子自动检测用户输入中的 `/command` 模式，将其转换为对应的 Skill 模板。用户无需记忆复杂的 Skill 调用语法，只需输入自然的命令格式。

**Skill-MCP 桥接**：Skill Hooks 与 Skill 系统紧密协作，当 Skill 被激活时，自动处理其内嵌的 MCP 服务器连接，使 Agent 能够透明地使用 MCP 工具。

---

## 代码索引 (Code Index)

| 功能点 | 文件路径 | 行号 | 函数/类 | 说明 |
|--------|----------|------|---------|------|
| Skill Hooks 创建 | `src/plugin/hooks/create-skill-hooks.ts` | 14-49 | `createSkillHooks` | 创建 2 个 Skill 钩子 |
| 类别技能提醒钩子 | `src/hooks/category-skill-reminder/hook.ts` | 58-142 | `createCategorySkillReminderHook` | 检测可委托任务并提醒 |
| 提醒消息格式化 | `src/hooks/category-skill-reminder/formatter.ts` | 11-37 | `buildReminderMessage` | 构建技能提醒消息 |
| 自动斜杠命令钩子 | `src/hooks/auto-slash-command/hook.ts` | 73-236 | `createAutoSlashCommandHook` | 自动检测 /command 模式 |
| 命令检测器 | `src/hooks/auto-slash-command/detector.ts` | 37-56 | `detectSlashCommand` | 解析斜杠命令 |
| 命令执行器 | `src/hooks/auto-slash-command/executor.ts` | 133-164 | `executeSlashCommand` | 执行并替换命令模板 |
| Skill 加载器 | `src/features/opencode-skill-loader/loader.ts` | 70-90 | `discoverAllSkills` | 发现所有可用 Skills |
| MCP 管理器 | `src/features/skill-mcp-manager/manager.ts` | 9-154 | `SkillMcpManager` | 管理 MCP 客户端生命周期 |
| MCP 连接 | `src/features/skill-mcp-manager/connection.ts` | 17-89 | `getOrCreateClient` | 获取或创建 MCP 连接 |

---

## 2 个 Skill Hooks

### 1. Skill 加载钩子 (categorySkillReminder)

`categorySkillReminder` 钩子监控 Agent 的工具使用情况，当检测到 Agent 正在执行可委托的任务时，提醒其使用类别委托和 Skill 加载。

**工作原理**：

```typescript
// src/hooks/category-skill-reminder/hook.ts:87-117
const toolExecuteAfter = async (input: ToolExecuteInput, output: ToolExecuteOutput) => {
  const { tool, sessionID } = input
  const toolLower = tool.toLowerCase()

  if (!isTargetAgent(sessionID, input.agent)) {
    return
  }

  const state = getOrCreateState(sessionID)

  if (DELEGATION_TOOLS.has(toolLower)) {
    state.delegationUsed = true
    return
  }

  if (!DELEGATABLE_WORK_TOOLS.has(toolLower)) {
    return
  }

  state.toolCallCount++

  if (state.toolCallCount >= 3 && !state.delegationUsed && !state.reminderShown) {
    output.output += reminderMessage
    state.reminderShown = true
  }
}
```

**目标 Agent**：Sisyphus、Sisyphus-Junior、Atlas 等编排型 Agent。

**可委托工具**：edit、write、bash、read、grep、glob 等表示 Agent 正在执行具体工作的工具。

**提醒触发条件**：
1. Agent 连续执行 3 次以上可委托工具
2. 尚未使用 task/call_omo_agent 等委托工具
3. 当前会话尚未显示过提醒

**提醒消息示例**：

```
[Category+Skill Reminder]

**Built-in**: git-master, playwright, frontend-ui-ux
**⚡ YOUR SKILLS (PRIORITY)**: github-triage, pre-publish-review

> User-installed skills OVERRIDE built-in defaults. ALWAYS prefer YOUR SKILLS when domain matches.

```typescript
task(category="visual-engineering", load_skills=["github-triage"], run_in_background=true)
```
```

### 2. Skill-MCP 桥接 (autoSlashCommand)

`autoSlashCommand` 钩子自动检测用户输入中的 `/command` 模式，将其转换为对应的 Skill 模板内容，实现命令式 Skill 激活。

**工作原理**：

```typescript
// src/hooks/auto-slash-command/hook.ts:88-156
"chat.message": async (input, output): Promise<void> => {
  const promptText = extractPromptText(output.parts)
  const parsed = detectSlashCommand(promptText)

  if (!parsed) {
    return
  }

  const result = await executeSlashCommand(parsed, executionOptions)

  if (!result.success || !result.replacementText) {
    return
  }

  const taggedContent = `${AUTO_SLASH_COMMAND_TAG_OPEN}\n${result.replacementText}\n${AUTO_SLASH_COMMAND_TAG_CLOSE}`
  output.parts[idx].text = taggedContent
}
```

**命令检测流程**：
1. 从消息中提取文本内容
2. 使用正则表达式匹配 `/command` 模式
3. 排除代码块中的命令（避免误触发）
4. 查找匹配的 Skill 或命令定义
5. 将命令替换为完整的 Skill 模板

**命令执行流程**：

```typescript
// src/hooks/auto-slash-command/executor.ts:133-164
export async function executeSlashCommand(parsed, options): Promise<ExecuteResult> {
  const command = await findCommand(parsed.command, options)

  if (!command) {
    return { success: false, error: `Command "/${parsed.command}" not found` }
  }

  if (command.scope === "skill" && command.metadata.agent) {
    if (!options?.agent || command.metadata.agent !== options.agent) {
      return { success: false, error: `Skill restricted to agent "${command.metadata.agent}"` }
    }
  }

  const template = await formatCommandTemplate(command, parsed.args)
  return { success: true, replacementText: template }
}
```

---

## Skill 激活流程

当用户输入包含 `/command` 或 Agent 被提醒使用 Skill 时，完整的激活流程如下：

1. **命令检测**：`autoSlashCommand` 钩子检测 `/command` 模式
2. **Skill 查找**：在已加载的 Skills 中查找匹配的命令
3. **模板加载**：加载 Skill 的 Markdown 模板内容
4. **变量替换**：替换 `${user_message}` 和 `$ARGUMENTS` 等变量
5. **MCP 初始化**：如果 Skill 包含 MCP 配置，初始化对应的服务器连接
6. **内容注入**：将处理后的 Skill 内容注入到对话中
7. **Agent 执行**：Agent 根据 Skill 指令执行任务
8. **工具调用**：Agent 通过 `SkillMcpManager` 调用 MCP 工具
9. **清理断开**：会话结束时断开 MCP 连接

---

## 流程/架构图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Skill Hooks 激活流程                                  │
└─────────────────────────────────────────────────────────────────────────────┘

  用户输入: "/github-triage my-repo"
         │
         ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  1. autoSlashCommand 钩子 (chat.message)                         │
  │     ┌──────────────────────────────────────────────────────┐   │
  │     │  detectSlashCommand("/github-triage my-repo")         │   │
  │     │  └── 解析出: { command: "github-triage", args: "my-repo" }│   │
  │     └──────────────────────────────────────────────────────┘   │
  └────────────────────────┬────────────────────────────────────────┘
                           ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  2. executeSlashCommand                                          │
  │     ┌──────────────────────────────────────────────────────┐   │
  │     │  findCommand("github-triage")                         │   │
  │     │  └── 返回 LoadedSkill 对象                             │   │
  │     └──────────────────────────────────────────────────────┘   │
  │     ┌──────────────────────────────────────────────────────┐   │
  │     │  formatCommandTemplate()                              │   │
  │     │  ├── 加载 SKILL.md 内容                               │   │
  │     │  ├── 解析 file:// 引用                                │   │
  │     │  └── 替换 $ARGUMENTS -> "my-repo"                     │   │
  │     └──────────────────────────────────────────────────────┘   │
  └────────────────────────┬────────────────────────────────────────┘
                           ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  3. 内容注入到对话                                                │
  │     [AUTO_SLASH_COMMAND]                                        │
  │     # /github-triage Command                                    │
  │     **Description**: Read-only GitHub triage...                 │
  │     ...                                                         │
  │     [/AUTO_SLASH_COMMAND]                                       │
  └────────────────────────┬────────────────────────────────────────┘
                           ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  4. Agent 执行 Skill 指令                                         │
  │     └── 识别需要使用 github-triage Skill                        │
  └────────────────────────┬────────────────────────────────────────┘
                           ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  5. MCP 服务器初始化 (如需要)                                      │
  │     ┌──────────────────────────────────────────────────────┐   │
  │     │  SkillMcpManager.getOrCreateClient()                  │   │
  │     │  ├── 检查现有连接                                      │   │
  │     │  ├── 创建 stdio/http 连接                              │   │
  │     │  └── 缓存客户端                                        │   │
  │     └──────────────────────────────────────────────────────┘   │
  └────────────────────────┬────────────────────────────────────────┘
                           ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  6. 工具执行                                                      │
  │     └── SkillMcpManager.callTool() -> MCP Server -> GitHub API  │
  └────────────────────────┬────────────────────────────────────────┘
                           ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  7. 清理与断开                                                    │
  │     └── session.deleted 事件 -> disconnectSession()             │
  └─────────────────────────────────────────────────────────────────┘
```

---

## 关键代码片段

### 片段 1：Skill Hooks 创建入口

```typescript
// src/plugin/hooks/create-skill-hooks.ts:14-49
export function createSkillHooks(args: {
  ctx: PluginContext
  pluginConfig: OhMyOpenCodeConfig
  isHookEnabled: (hookName: HookName) => boolean
  safeHookEnabled: boolean
  mergedSkills: LoadedSkill[]
  availableSkills: AvailableSkill[]
}): SkillHooks {
  const { ctx, pluginConfig, isHookEnabled, safeHookEnabled, mergedSkills, availableSkills } = args

  const safeHook = <T>(hookName: HookName, factory: () => T): T | null =>
    safeCreateHook(hookName, factory, { enabled: safeHookEnabled })

  const categorySkillReminder = isHookEnabled("category-skill-reminder")
    ? safeHook("category-skill-reminder", () =>
        createCategorySkillReminderHook(ctx, availableSkills))
    : null

  const autoSlashCommand = isHookEnabled("auto-slash-command")
    ? safeHook("auto-slash-command", () =>
        createAutoSlashCommandHook({
          skills: mergedSkills,
          pluginsEnabled: pluginConfig.claude_code?.plugins ?? true,
          enabledPluginsOverride: pluginConfig.claude_code?.plugins_override,
        }))
    : null

  return { categorySkillReminder, autoSlashCommand }
}
```

### 片段 2：斜杠命令检测

```typescript
// src/hooks/auto-slash-command/detector.ts:37-56
export function detectSlashCommand(text: string): ParsedSlashCommand | null {
  const textWithoutCodeBlocks = removeCodeBlocks(text)
  const trimmed = textWithoutCodeBlocks.trim()

  if (!trimmed.startsWith("/")) {
    return null
  }

  const parsed = parseSlashCommand(trimmed)

  if (!parsed) {
    return null
  }

  if (isExcludedCommand(parsed.command)) {
    return null
  }

  return parsed
}
```

### 片段 3：MCP 客户端获取与重试

```typescript
// src/features/skill-mcp-manager/manager.ts:95-134
private async withOperationRetry<T>(
  info: SkillMcpClientInfo,
  config: ClaudeCodeMcpServer,
  operation: (client: Client) => Promise<T>
): Promise<T> {
  const maxRetries = 3
  let lastError: Error | null = null

  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      const client = await this.getOrCreateClientWithRetry(info, config)
      return await operation(client)
    } catch (error) {
      lastError = error instanceof Error ? error : new Error(String(error))
      
      const stepUpHandled = await handleStepUpIfNeeded({
        error: lastError,
        config,
        authProviders: this.state.authProviders,
      })
      if (stepUpHandled) {
        await forceReconnect(this.state, this.getClientKey(info))
        continue
      }

      if (attempt === maxRetries) {
        throw new Error(`Failed after ${maxRetries} reconnection attempts`)
      }

      await forceReconnect(this.state, this.getClientKey(info))
    }
  }

  throw lastError ?? new Error("Operation failed with unknown error")
}
```

---

## 依赖关系

Skill Hooks 与以下组件存在依赖关系：

| 依赖组件 | 关系类型 | 说明 |
|----------|----------|------|
| `opencode-skill-loader` | 强依赖 | Skill 发现和加载 |
| `skill-mcp-manager` | 强依赖 | MCP 客户端生命周期管理 |
| `dynamic-agent-prompt-builder` | 弱依赖 | 可用技能列表构建 |
| `claude-code-session-state` | 弱依赖 | 会话 Agent 信息获取 |
| `slashcommand` | 弱依赖 | 命令发现和解析 |

---

## 实战示例

### 示例 1: GitHub Triage Skill 激活

**用户输入**：
```
/github-triage my-project
```

**系统处理流程**：

1. **命令检测**：`autoSlashCommand` 钩子检测到 `/github-triage` 命令
2. **Skill 查找**：在 `mergedSkills` 中找到 `github-triage` Skill
3. **模板加载**：读取 `.opencode/skills/github-triage/SKILL.md` 内容
4. **变量替换**：将 `$ARGUMENTS` 替换为 `"my-project"`
5. **内容注入**：将完整 Skill 指令注入到对话中

**生成的 Skill 内容**（部分）：

```markdown
# GitHub Triage - Read-Only Analyzer

<role>
Read-only GitHub triage orchestrator. Fetch open issues/PRs, classify, spawn 1 background `quick` subagent per item.
</role>

## Architecture

**1 ISSUE/PR = 1 `task_create` = 1 `quick` SUBAGENT (background).**

## Phase 1: Fetch All Open Items
...
```

**Agent 执行**：
- Agent 读取 Skill 指令
- 识别需要执行 GitHub Triage 流程
- 使用 `task()` 工具启动后台子 Agent
- 每个 Issue/PR 分配一个独立的分析子 Agent

### 示例 2: Pre-publish Review Skill 激活

**用户输入**：
```
pre-publish review
```

**系统处理流程**：

1. **关键词匹配**：`autoSlashCommand` 检测到触发词 "pre-publish review"
2. **Skill 激活**：匹配到 `pre-publish-review` Skill
3. **模板加载**：加载 16-Agent 发布审查流程的完整指令
4. **MCP 准备**：准备 npm registry MCP 连接（如需要）

**Skill 执行流程**：

```
Phase 0: Detect Unpublished Changes
  └── skill(name="get-unpublished-changes")

Phase 1: Parse Changes into Groups
  └── 将变更分组（最多 10 组）

Phase 2: Spawn All Agents
  ├── Layer 1: 10 个 ultrabrain agents（并行）
  │   └── 每组变更一个深度分析 Agent
  ├── Layer 2: 5 个 review-work agents（并行）
  │   └── 目标验证、QA 执行、代码质量、安全、上下文挖掘
  └── Layer 3: 1 个 oracle agent
      └── 整体发布综合评估

Phase 3: Collect Results
  └── 收集所有 Agent 的分析结果

Phase 4: Final Verdict
  └── 生成最终发布审查报告
```

**输出结果**：包含版本升级建议、破坏性变更列表、部署风险评估和发布后的监控建议。

---

## 交叉引用

- 参见：[Hook 分层](./01-Hook 分层.md) - 了解 Skill Hooks 在 5 层钩子架构中的位置
- 参见：[Skill 系统](../05-功能模块/03-Skill 系统.md) - 深入了解 Skill 加载和 MCP 集成的完整机制
- 参见：[MCP 集成](../05-功能模块/03-Skill 系统.md) - 了解 Skill-MCP 桥接的底层实现
- 参见：[Session Hooks](./02-Session Hooks.md) - 了解与 Skill Hooks 协作的会话级钩子
- 参见：[后台 Agent](../05-功能模块/01-后台 Agent.md) - 了解 Skill 如何与后台 Agent 协作执行复杂任务
- 参见：[Agent 系统](../04-Agent 系统/01-Agent 概述.md) - 了解 Agent 如何使用 Skill 进行任务委派
