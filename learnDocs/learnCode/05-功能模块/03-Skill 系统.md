# Skill 系统：技能加载与 MCP 集成

> 所属模块：05-features-modules | 三层模型：Layer 3 | 优先级：P2

---

## 架构位置

```
┌─────────────────────────────────────────────────────────────────┐
│                        Layer 3: Features                         │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              Skill System (本模块)                       │   │
│  │  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐ │   │
│  │  │   Skill     │───▶│   Skill     │───▶│    MCP      │ │   │
│  │  │  Discovery  │    │   Loader    │    │  Manager    │ │   │
│  │  └─────────────┘    └─────────────┘    └─────────────┘ │   │
│  │         │                  │                  │         │   │
│  │         ▼                  ▼                  ▼         │   │
│  │  ┌─────────────────────────────────────────────────┐   │   │
│  │  │         Skill-MCP Bridge (集成层)                │   │   │
│  │  └─────────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              │                                   │
│                              ▼                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              Layer 2: Shared Utilities                   │   │
│  │         (Frontmatter Parser, File Utils, etc.)           │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

Skill 系统位于 Layer 3 Features 层，是连接用户配置与底层 MCP 协议的核心桥梁。它负责从多个作用域发现、加载、合并 Skill 定义，并管理 Skill 内嵌的 MCP 服务器生命周期。

---

## 核心职责

Skill 系统是 oh-my-opencode 插件的扩展机制核心，承担以下关键职责：

**1. 多作用域 Skill 发现**

系统支持四级作用域的 Skill 发现机制，按优先级从高到低依次为：Project (`.opencode/skills/`) > OpenCode Config (`~/.config/opencode/skills/`) > User (`~/.config/opencode/oh-my-opencode/skills/`) > Global (内置 Skills)。高优先级作用域的同名 Skill 会覆盖低优先级的定义，实现灵活的配置覆盖。

**2. YAML Frontmatter 解析**

每个 Skill 定义在 `SKILL.md` 文件中使用 YAML Frontmatter 描述元数据，包括名称、描述、适用工具、MCP 配置等。系统解析这些元数据并转换为内部 `LoadedSkill` 结构，供后续流程使用。

**3. MCP 服务器生命周期管理**

Skill 可以内嵌 MCP (Model Context Protocol) 服务器配置，系统通过 `SkillMcpManager` 统一管理这些服务器的连接、重连、断开和清理。支持 stdio (本地进程) 和 HTTP (远程服务) 两种连接类型，并实现了完善的错误重试和 OAuth 认证流程。

**4. Skill-MCP 桥接**

系统提供 Skill 内容与 MCP 工具的无缝集成，Agent 可以在执行 Skill 指令时透明地调用关联的 MCP 工具，无需关心底层连接细节。

---

## 代码索引 (Code Index)

| 功能点 | 文件路径 | 行号 | 函数/类 | 说明 |
|--------|----------|------|---------|------|
| Skill 发现 | `src/features/opencode-skill-loader/skill-discovery.ts` | 12-76 | `getAllSkills()` | 从四个作用域发现并合并所有 Skills |
| Skill 目录加载 | `src/features/opencode-skill-loader/skill-directory-loader.ts` | 7-106 | `loadSkillsFromDir()` | 递归加载目录中的 SKILL.md 文件 |
| Skill 文件解析 | `src/features/opencode-skill-loader/loaded-skill-from-path.ts` | 11-69 | `loadSkillFromPath()` | 解析单个 Skill 文件的 Frontmatter |
| MCP 配置解析 | `src/features/opencode-skill-loader/skill-mcp-config.ts` | 1-80 | `parseSkillMcpConfigFromFrontmatter()` | 从 YAML 提取 MCP 服务器配置 |
| MCP 管理器 | `src/features/skill-mcp-manager/manager.ts` | 9-154 | `SkillMcpManager` | 管理 MCP 客户端生命周期 |
| MCP 连接创建 | `src/features/skill-mcp-manager/connection.ts` | 17-89 | `getOrCreateClient()` | 获取或创建 MCP 客户端连接 |
| Stdio 客户端 | `src/features/skill-mcp-manager/stdio-client.ts` | 15-76 | `createStdioClient()` | 创建本地进程 MCP 连接 |
| HTTP 客户端 | `src/features/skill-mcp-manager/http-client.ts` | 10-65 | `createHttpClient()` | 创建远程 HTTP MCP 连接 |
| 内置 Skills | `src/features/builtin-skills/skills.ts` | 18-37 | `createBuiltinSkills()` | 创建内置 Skill 定义 |
| Skill 类型定义 | `src/features/skill-mcp-manager/types.ts` | 1-70 | `SkillMcpClientInfo` | MCP 客户端信息接口 |

---

## Skill 加载机制

### 1. Skill 发现

Skill 发现是系统的入口点，负责从四个作用域收集所有可用的 Skill 定义。

```typescript
// src/features/opencode-skill-loader/skill-discovery.ts:12-30
export async function getAllSkills(options?: SkillResolutionOptions): Promise<LoadedSkill[]> {
  const [discoveredSkills, builtinSkillDefinitions] = await Promise.all([
    discoverSkills({ includeClaudeCodePaths: true, directory: options?.directory }),
    Promise.resolve(
      createBuiltinSkills({
        browserProvider: options?.browserProvider,
        disabledSkills: options?.disabledSkills,
      })
    ),
  ])
  // ... 合并与过滤逻辑
}
```

发现过程采用并行策略，同时扫描文件系统 Skills 和生成内置 Skills。内置 Skills 包括 `git-master`、`playwright`、`frontend-ui-ux` 等，根据 `browserProvider` 配置选择对应的浏览器自动化 Skill。

### 2. Skill 加载

Skill 加载负责解析单个 `SKILL.md` 文件，提取 YAML Frontmatter 和 Markdown 内容。

```typescript
// src/features/opencode-skill-loader/loaded-skill-from-path.ts:11-35
export async function loadSkillFromPath(options: {
  skillPath: string
  resolvedPath: string
  defaultName: string
  scope: SkillScope
  namePrefix?: string
}): Promise<LoadedSkill | null> {
  const content = await fs.readFile(options.skillPath, "utf-8")
  const { data, body } = parseFrontmatter<SkillMetadata>(content)

  const frontmatterMcp = parseSkillMcpConfigFromFrontmatter(content)
  const mcpJsonMcp = await loadMcpJsonFromDir(options.resolvedPath)
  const mcpConfig = mcpJsonMcp || frontmatterMcp
  // ... 构建 LoadedSkill 对象
}
```

加载过程首先读取文件内容，使用 `parseFrontmatter` 分离 YAML 元数据和 Markdown 正文。然后尝试从两个来源获取 MCP 配置：Frontmatter 中的 `mcp` 字段，或同目录下的 `mcp.json` 文件。最终构建包含完整元数据、模板内容和 MCP 配置的 `LoadedSkill` 对象。

### 3. MCP 服务器集成

MCP 服务器通过 `SkillMcpManager` 进行统一管理，支持按需连接和自动重连。

```typescript
// src/features/skill-mcp-manager/manager.ts:46-62
async listTools(info: SkillMcpClientInfo, context: SkillMcpServerContext): Promise<Tool[]> {
  const client = await this.getOrCreateClientWithRetry(info, context.config)
  const result = await client.listTools()
  return result.tools
}

async callTool(
  info: SkillMcpClientInfo,
  context: SkillMcpServerContext,
  name: string,
  args: Record<string, unknown>
): Promise<unknown> {
  return await this.withOperationRetry(info, context.config, async (client) => {
    const result = await client.callTool({ name, arguments: args })
    return result.content
  })
}
```

`SkillMcpManager` 提供完整的 MCP 操作接口，包括列出工具、调用工具、读取资源和获取提示词。所有操作都内置了重试机制，当连接断开时会自动尝试重连，确保 Skill 执行的稳定性。

---

## Skill 加载流程

Skill 加载遵循以下流程：

1. **作用域扫描**：按 Project → OpenCode → User → Global 顺序扫描各作用域的 Skills 目录
2. **文件发现**：在每个目录中递归查找 `SKILL.md` 或 `{dirname}.md` 文件
3. **Frontmatter 解析**：解析 YAML 元数据，提取名称、描述、工具限制、MCP 配置等
4. **MCP 配置合并**：合并 Frontmatter 和 `mcp.json` 中的 MCP 服务器配置
5. **作用域合并**：按优先级合并各作用域的 Skills，高优先级覆盖低优先级
6. **Provider 过滤**：根据 `browserProvider` 配置过滤浏览器相关的 Skills
7. **禁用过滤**：移除被 `disabledSkills` 禁用的 Skills
8. **缓存存储**：将结果缓存，避免重复加载

---

## 流程/架构图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Skill 加载与 MCP 集成流程                           │
└─────────────────────────────────────────────────────────────────────────────┘

  ┌──────────────┐
  │   开始加载    │
  └──────┬───────┘
         │
         ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  1. 四作用域发现 (Project > OpenCode > User > Global)            │
  │     ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌──────────┐│
  │     │  .opencode/ │ │ ~/.config/  │ │ ~/.config/  │ │ Built-in ││
  │     │   skills/   │ │ opencode/   │ │ oh-my-opencode/ │ Skills ││
  │     │             │ │  skills/    │ │   skills/   │ │          ││
  │     └──────┬──────┘ └──────┬──────┘ └──────┬──────┘ └────┬─────┘│
  └────────────┼───────────────┼───────────────┼─────────────┼──────┘
               │               │               │             │
               └───────────────┴───────┬───────┴─────────────┘
                                       ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  2. 目录递归扫描                                                  │
  │     - 查找 SKILL.md                                              │
  │     - 查找 {dirname}.md                                          │
  │     - 递归子目录 (maxDepth=2)                                     │
  └────────────────────────┬────────────────────────────────────────┘
                           ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  3. 文件解析 (loadSkillFromPath)                                  │
  │     ┌──────────────────────────────────────────────────────┐   │
  │     │  parseFrontmatter()                                   │   │
  │     │  ├── YAML 元数据 (name, description, tools, mcp)      │   │
  │     │  └── Markdown 正文 (Skill 指令内容)                    │   │
  │     └──────────────────────────────────────────────────────┘   │
  │     ┌──────────────────────────────────────────────────────┐   │
  │     │  MCP 配置来源                                          │   │
  │     │  ├── Frontmatter: mcp: {...}                          │   │
  │     │  └── mcp.json (同目录)                                │   │
  │     └──────────────────────────────────────────────────────┘   │
  └────────────────────────┬────────────────────────────────────────┘
                           ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  4. 作用域合并 (merger)                                           │
  │     - 高优先级覆盖低优先级                                         │
  │     - MCP 配置智能合并                                             │
  └────────────────────────┬────────────────────────────────────────┘
                           ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  5. MCP 客户端管理 (SkillMcpManager)                              │
  │     ┌──────────────────────────────────────────────────────┐   │
  │     │  getOrCreateClient()                                  │   │
  │     │  ├── 检查现有连接                                      │   │
  │     │  ├── 防止并发竞争                                      │   │
  │     │  ├── 创建新连接 (stdio/http)                           │   │
  │     │  └── 缓存客户端                                        │   │
  │     └──────────────────────────────────────────────────────┘   │
  │     ┌──────────────────────────────────────────────────────┐   │
  │     │  连接类型                                              │   │
  │     │  ├── stdio: 本地进程 (command + args)                  │   │
  │     │  └── http: 远程服务 (url + headers)                    │   │
  │     └──────────────────────────────────────────────────────┘   │
  └────────────────────────┬────────────────────────────────────────┘
                           ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  6. Skill 执行                                                    │
  │     - Agent 接收 Skill 模板                                       │
  │     - 透明调用 MCP 工具                                           │
  │     - 自动重连和错误恢复                                           │
  └────────────────────────┬────────────────────────────────────────┘
                           ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  7. 清理与断开                                                    │
  │     - 会话结束断开 MCP 连接                                        │
  │     - 空闲超时自动清理                                             │
  │     - 进程退出清理                                                 │
  └─────────────────────────────────────────────────────────────────┘
```

---

## 关键代码片段

### 片段 1：Skill 目录递归加载

```typescript
// src/features/opencode-skill-loader/skill-directory-loader.ts:32-88
for (const entry of directories) {
  const entryPath = join(options.skillsDir, entry.name)
  const resolvedPath = await resolveSymlinkAsync(entryPath)
  const dirName = entry.name

  const skillMdPath = join(resolvedPath, "SKILL.md")
  try {
    await fs.access(skillMdPath)
    const skill = await loadSkillFromPath({
      skillPath: skillMdPath,
      resolvedPath,
      defaultName: dirName,
      scope: options.scope,
      namePrefix,
    })
    if (skill && !skillMap.has(skill.name)) {
      skillMap.set(skill.name, skill)
    }
    continue
  } catch {
    // no SKILL.md
  }

  // 递归处理子目录
  if (depth < maxDepth) {
    const newPrefix = namePrefix ? `${namePrefix}/${dirName}` : dirName
    const nestedSkills = await loadSkillsFromDir({
      skillsDir: resolvedPath,
      scope: options.scope,
      namePrefix: newPrefix,
      depth: depth + 1,
      maxDepth,
    })
    // ...
  }
}
```

### 片段 2：MCP 客户端连接管理

```typescript
// src/features/skill-mcp-manager/connection.ts:17-45
export async function getOrCreateClient(params: {
  state: SkillMcpManagerState
  clientKey: string
  info: SkillMcpClientInfo
  config: ClaudeCodeMcpServer
}): Promise<Client> {
  const { state, clientKey, info, config } = params

  if (state.disposed) {
    throw new Error(`MCP manager for "${info.sessionID}" has been shut down`)
  }

  const existing = state.clients.get(clientKey)
  if (existing) {
    existing.lastUsedAt = Date.now()
    return existing.client
  }

  // 防止并发竞争：如果连接已在进行中，等待它完成
  const pending = state.pendingConnections.get(clientKey)
  if (pending) {
    return pending
  }
  // ... 创建新连接
}
```

### 片段 3：Stdio MCP 客户端创建

```typescript
// src/features/skill-mcp-manager/stdio-client.ts:15-45
export async function createStdioClient(params: SkillMcpClientConnectionParams): Promise<Client> {
  const { state, clientKey, info, config } = params

  const command = getStdioCommand(config, info.serverName)
  const args = config.args ?? []
  const mergedEnv = createCleanMcpEnvironment(config.env)

  registerProcessCleanup(state)

  const transport = new StdioClientTransport({
    command,
    args,
    env: mergedEnv,
    stderr: "ignore",
  })

  const client = new Client(
    { name: `skill-mcp-${info.skillName}-${info.serverName}`, version: "1.0.0" },
    { capabilities: {} }
  )

  await client.connect(transport)
  // ... 缓存客户端
}
```

---

## 依赖关系

Skill 系统与以下组件存在依赖关系：

| 依赖组件 | 关系类型 | 说明 |
|----------|----------|------|
| `shared/frontmatter` | 强依赖 | YAML Frontmatter 解析 |
| `shared/file-utils` | 强依赖 | 文件系统操作和符号链接解析 |
| `claude-code-mcp-loader` | 强依赖 | MCP 服务器配置类型定义 |
| `mcp-oauth` | 弱依赖 | OAuth 认证流程处理 |
| `builtin-skills` | 强依赖 | 内置 Skill 定义 |
| `config/schema` | 弱依赖 | 浏览器 Provider 配置类型 |

---

## 实战示例

### 示例 1: GitHub Triage Skill

GitHub Triage Skill 是一个典型的内置 Skill，展示了 Skill 系统的完整使用流程。

**SKILL.md 结构：**

```markdown
---
name: github-triage
description: Read-only GitHub triage for issues AND PRs
tools: [Bash, Read, Write]
mcp:
  github:
    type: stdio
    command: npx
    args: [-y, @github/mcp-server]
---

Analyze GitHub issues and PRs, write evidence-backed reports...
```

**系统处理流程：**

1. **发现阶段**：`skill-discovery.ts` 从内置 Skills 中发现 `github-triage`
2. **加载阶段**：`loaded-skill-from-path.ts` 解析 Frontmatter，提取 MCP 配置
3. **MCP 初始化**：当 Agent 执行该 Skill 时，`SkillMcpManager` 按需启动 GitHub MCP 服务器
4. **工具调用**：Agent 通过 `callTool()` 透明调用 GitHub MCP 提供的工具
5. **清理阶段**：会话结束时自动断开 MCP 连接

### 示例 2: Pre-publish Review Skill

Pre-publish Review Skill 展示了复杂 Skill 的 MCP 集成和 Agent 协作。

**SKILL.md 结构：**

```markdown
---
name: pre-publish-review
description: Nuclear-grade 16-agent pre-publish release gate
tools: [Bash, Read, Write, delegate_task]
mcp:
  npm-registry:
    type: http
    url: https://registry.npmjs.org
---

Run /get-unpublished-changes to detect all changes since last npm release...
```

**系统处理流程：**

1. **Skill 解析**：系统解析出该 Skill 需要 `delegate_task` 工具和 HTTP 类型的 MCP 服务器
2. **Agent 委派**：主 Agent 使用 `delegate_task` 启动多个子 Agent 进行并行审查
3. **MCP 调用**：通过 HTTP MCP 获取 npm 包信息和发布状态
4. **结果聚合**：子 Agent 完成后，主 Agent 整合结果生成发布报告
5. **资源释放**：所有子 Agent 会话和 MCP 连接被正确清理

---

## 交叉引用

- 参见：[后台 Agent](./01-后台 Agent.md) - 了解 Skill 如何与后台 Agent 协作执行复杂任务
- 参见：[配置系统](../07-配置系统/01-配置加载.md) - 了解 Skill 配置的多级合并机制
- 参见：[MCP 系统](../08-shared-utilities/mcp-utils.md) - 了解 MCP 协议的底层实现细节
- 参见：[Agent 系统](../04-agent-system/builtin-agents.md) - 了解 Agent 如何使用 Skill 进行任务执行
- 参见：[Tmux 集成](./tmux-subagent.md) - 了解 Skill 如何与 Tmux 集成实现交互式会话
