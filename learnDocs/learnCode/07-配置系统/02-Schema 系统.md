# Schema 系统：Zod v4 验证与默认值填充

> 所属模块：07-configuration-system | 三层模型：Layer 3 | 优先级：P2

---

## 架构位置

Schema 系统位于配置加载管道的第三层，负责将原始配置数据转换为类型安全的结构化对象。

```
┌─────────────────────────────────────────────────────────────────┐
│                    Layer 1: 配置发现                             │
│         (detectPluginConfigFile - 查找 .jsonc/.json)            │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                    Layer 2: 配置解析                             │
│              (parseJsonc - JSONC 解析与注释处理)                 │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  ★ Layer 3: Schema 验证 (Zod v4)                                │
│     ├─ safeParse() 验证                                         │
│     ├─ 默认值填充 (Zod defaults)                                │
│     └─ 部分解析 Fallback                                        │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                    Layer 4: 配置合并                             │
│         (mergeConfigs - user + project 深度合并)                │
└─────────────────────────────────────────────────────────────────┘
```

---

## 核心职责

Schema 系统是 oh-my-opencode 配置体系的类型安全守门员。它通过 Zod v4 实现从原始 JSONC 数据到 TypeScript 类型的转换，确保配置在运行时符合预期结构。

系统承担四项核心职责：首先，Schema 定义使用 Zod 的链式 API 描述每个配置字段的类型、约束和默认值；其次，验证阶段通过 `safeParse()` 方法执行运行时类型检查，捕获类型不匹配、范围越界等错误；第三，对于验证通过但缺少的字段，系统自动填充 Zod schema 中定义的默认值；第四，当完整验证失败时，系统启用部分解析模式，保留有效的配置段，仅丢弃出错的部分，最大限度保证插件可用性。

这种设计使得配置错误不会导致整个插件崩溃，而是优雅降级，同时向用户提供清晰的错误反馈。

---

## 代码索引 (Code Index)

| 功能点 | 文件路径 | 行号 | 函数/类 | 说明 |
|--------|----------|------|---------|------|
| 根 Schema 定义 | `src/config/schema/oh-my-opencode-config.ts` | 26-71 | `OhMyOpenCodeConfigSchema` | 主配置 Schema，组合所有子模块 |
| Agent 覆盖配置 | `src/config/schema/agent-overrides.ts` | 5-56 | `AgentOverrideConfigSchema` | 单个 Agent 的 21 个可覆盖字段 |
| 分类配置 | `src/config/schema/categories.ts` | 4-28 | `CategoryConfigSchema` | 任务分类的模型与参数配置 |
| 回退模型 | `src/config/schema/fallback-models.ts` | 3-23 | `FallbackModelsSchema` | 支持字符串或对象数组的联合类型 |
| 后台任务 | `src/config/schema/background-task.ts` | 9-27 | `BackgroundTaskConfigSchema` | 并发限制与熔断器配置 |
| Hook 名称枚举 | `src/config/schema/hooks.ts` | 3-56 | `HookNameSchema` | 48 个内置 Hook 的枚举定义 |
| Skill 配置 | `src/config/schema/skills.ts` | 29-36 | `SkillsConfigSchema` | 支持多种格式的 Skill 配置 |
| 配置验证入口 | `src/plugin-config.ts` | 80-84 | `loadConfigFromPath` | 调用 Zod safeParse 进行验证 |
| 部分解析 | `src/plugin-config.ts` | 25-67 | `parseConfigPartially` | 验证失败时的降级解析逻辑 |
| 配置合并 | `src/plugin-config.ts` | 112-159 | `mergeConfigs` | user + project 配置深度合并 |

---

## Zod v4 Schema 系统

### 1. Schema 定义

oh-my-opencode 使用 Zod v4 的声明式 API 定义配置结构。每个模块拥有独立的 schema 文件，最终通过根 schema 组合。

```typescript
// src/config/schema/oh-my-opencode-config.ts (L26-44)
export const OhMyOpenCodeConfigSchema = z.object({
  $schema: z.string().optional(),
  new_task_system_enabled: z.boolean().optional(),
  default_run_agent: z.string().optional(),
  disabled_mcps: z.array(AnyMcpNameSchema).optional(),
  disabled_agents: z.array(z.string()).optional(),
  agents: AgentOverridesSchema.optional(),
  categories: CategoriesConfigSchema.optional(),
  claude_code: ClaudeCodeConfigSchema.optional(),
  // ... 更多字段
})
```

Schema 定义遵循模块化原则：基础类型（如 `z.string()`、`z.boolean()`）通过 `.optional()` 标记为可选，复杂类型通过组合子 schema（如 `AgentOverridesSchema`）实现复用。这种分层设计使得新增配置字段时只需修改对应模块的 schema 文件，无需触碰根 schema 的核心逻辑。

### 2. 验证

配置验证通过 Zod 的 `safeParse()` 方法执行，该方法返回包含 `success` 标志的结果对象，避免抛出异常中断流程。

```typescript
// src/plugin-config.ts (L80-84)
const result = OhMyOpenCodeConfigSchema.safeParse(rawConfig);

if (result.success) {
  log(`Config loaded from ${configPath}`, { agents: result.data.agents });
  return result.data;
}
```

当验证失败时，Zod 提供详细的错误信息，包括出错字段的路径和具体原因。系统将这些错误收集并记录，帮助用户快速定位配置问题。

```typescript
// src/plugin-config.ts (L87-94)
const errorMsg = result.error.issues
  .map((i) => `${i.path.join(".")}: ${i.message}`)
  .join(", ");
log(`Config validation error in ${configPath}:`, result.error.issues);
addConfigLoadError({
  path: configPath,
  error: `Partial config loaded — invalid sections skipped: ${errorMsg}`,
});
```

### 3. 默认值填充

Zod schema 通过 `.default()` 或 `.optional()` 结合类型推断实现默认值填充。oh-my-opencode 采用"可选字段 + 运行时合并"的策略：schema 中所有字段均为 optional，默认值在业务逻辑层或合并阶段填充。

```typescript
// src/config/schema/background-task.ts (L9-27)
export const BackgroundTaskConfigSchema = z.object({
  defaultConcurrency: z.number().min(1).optional(),
  staleTimeoutMs: z.number().min(60000).optional(),
  maxToolCalls: z.number().int().min(10).optional(),
  circuitBreaker: CircuitBreakerConfigSchema.optional(),
})
```

对于数值字段，使用 `.min()` 和 `.max()` 限定有效范围，确保即使配置文件中存在越界值，Zod 也会捕获并报告错误。

### 4. 部分解析 Fallback

当完整配置验证失败时，系统启用部分解析模式 `parseConfigPartially`，逐段验证配置，保留有效部分。

```typescript
// src/plugin-config.ts (L25-67)
export function parseConfigPartially(
  rawConfig: Record<string, unknown>
): OhMyOpenCodeConfig | null {
  const fullResult = OhMyOpenCodeConfigSchema.safeParse(rawConfig);
  if (fullResult.success) {
    return fullResult.data;
  }

  const partialConfig: Record<string, unknown> = {};
  const invalidSections: string[] = [];

  for (const key of Object.keys(rawConfig)) {
    if (PARTIAL_STRING_ARRAY_KEYS.has(key)) {
      // 特殊处理字符串数组字段
      const sectionValue = rawConfig[key];
      if (Array.isArray(sectionValue) && sectionValue.every((value) => typeof value === "string")) {
        partialConfig[key] = sectionValue;
      }
      continue;
    }

    const sectionResult = OhMyOpenCodeConfigSchema.safeParse({ [key]: rawConfig[key] });
    if (sectionResult.success) {
      const parsed = sectionResult.data as Record<string, unknown>;
      if (parsed[key] !== undefined) {
        partialConfig[key] = parsed[key];
      }
    } else {
      invalidSections.push(`${key}: ${sectionErrors}`);
    }
  }

  return partialConfig as OhMyOpenCodeConfig;
}
```

部分解析特别处理了 `disabled_*` 类字段（如 `disabled_agents`、`disabled_hooks`），这些字段即使整体验证失败，只要内容是字符串数组，也会被保留。这种设计确保用户可以通过禁用功能来临时绕过配置错误。

---

## Schema 验证流程

配置加载时的完整验证流程如下：

1. **文件读取**：从 `~/.config/opencode/` 和项目 `.opencode/` 目录读取 `.jsonc` 或 `.json` 文件
2. **JSONC 解析**：使用 `parseJsonc` 处理带注释的 JSON
3. **迁移检查**：调用 `migrateConfigFile` 处理旧版本配置的自动升级
4. **完整验证**：使用 `OhMyOpenCodeConfigSchema.safeParse()` 验证整个配置对象
5. **成功路径**：验证通过，返回完整配置数据
6. **失败降级**：验证失败时，调用 `parseConfigPartially` 进行分段验证
7. **部分合并**：将有效配置段合并为部分配置对象
8. **错误报告**：记录无效字段，提示用户修复

---

## 流程/架构图

```
┌──────────────────────────────────────────────────────────────────────┐
│                        Schema 验证流程                                │
└──────────────────────────────────────────────────────────────────────┘

  ┌──────────────┐
  │  读取配置文件  │◄────────────────────────────────────────┐
  │  (.jsonc)    │                                         │
  └──────┬───────┘                                         │
         │                                                  │
         ▼                                                  │
  ┌──────────────┐     ┌──────────────┐                    │
  │  JSONC 解析   │────►│  配置迁移     │                    │
  │ parseJsonc   │     │ migrateConfig │                   │
  └──────┬───────┘     └──────┬───────┘                    │
         │                    │                            │
         ▼                    ▼                            │
  ┌──────────────────────────────────────┐                 │
  │      Zod safeParse() 完整验证         │                 │
  │  OhMyOpenCodeConfigSchema.safeParse() │                 │
  └──────────────────┬───────────────────┘                 │
                     │                                     │
           ┌─────────┴─────────┐                          │
           │                   │                          │
           ▼                   ▼                          │
    ┌────────────┐      ┌────────────┐                    │
    │  验证成功   │      │  验证失败   │                    │
    │ success=true│      │ success=false│                   │
    └─────┬──────┘      └─────┬──────┘                    │
          │                   │                           │
          ▼                   ▼                           │
    ┌────────────┐      ┌────────────────────┐            │
    │ 返回完整配置 │      │ parseConfigPartially│           │
    │ result.data │      │    部分解析         │            │
    └────────────┘      └─────────┬──────────┘            │
                                  │                       │
                    ┌─────────────┼─────────────┐         │
                    ▼             ▼             ▼         │
              ┌────────┐    ┌────────┐    ┌────────┐      │
              │ 有效段1 │    │ 有效段2 │    │ 无效段  │──────┘
              └───┬────┘    └───┬────┘    └────────┘
                  │             │
                  └──────┬──────┘
                         ▼
                  ┌────────────┐
                  │ 合并部分配置 │
                  │ partialConfig│
                  └─────┬──────┘
                        ▼
                  ┌────────────┐
                  │ 记录错误信息 │
                  │ addConfigLoadError
                  └────────────┘
```

---

## 关键代码片段

### 片段 1：Agent 覆盖配置的完整定义

```typescript
// src/config/schema/agent-overrides.ts (L5-56)
export const AgentOverrideConfigSchema = z.object({
  model: z.string().optional(),
  fallback_models: FallbackModelsSchema.optional(),
  variant: z.string().optional(),
  category: z.string().optional(),
  skills: z.array(z.string()).optional(),
  temperature: z.number().min(0).max(2).optional(),
  top_p: z.number().min(0).max(1).optional(),
  prompt: z.string().optional(),
  prompt_append: z.string().optional(),
  tools: z.record(z.string(), z.boolean()).optional(),
  disable: z.boolean().optional(),
  description: z.string().optional(),
  mode: z.enum(["subagent", "primary", "all"]).optional(),
  color: z.string().regex(/^#[0-9A-Fa-f]{6}$/).optional(),
  permission: AgentPermissionSchema.optional(),
  maxTokens: z.number().optional(),
  thinking: z.object({
    type: z.enum(["enabled", "disabled"]),
    budgetTokens: z.number().optional(),
  }).optional(),
  reasoningEffort: z.enum(["none", "minimal", "low", "medium", "high", "xhigh"]).optional(),
  textVerbosity: z.enum(["low", "medium", "high"]).optional(),
  providerOptions: z.record(z.string(), z.unknown()).optional(),
})
```

### 片段 2：分类配置的枚举与结构

```typescript
// src/config/schema/categories.ts (L4-44)
export const CategoryConfigSchema = z.object({
  description: z.string().optional(),
  model: z.string().optional(),
  fallback_models: FallbackModelsSchema.optional(),
  variant: z.string().optional(),
  temperature: z.number().min(0).max(2).optional(),
  top_p: z.number().min(0).max(1).optional(),
  maxTokens: z.number().optional(),
  reasoningEffort: z.enum(["none", "minimal", "low", "medium", "high", "xhigh"]).optional(),
  textVerbosity: z.enum(["low", "medium", "high"]).optional(),
  is_unstable_agent: z.boolean().optional(),
  disable: z.boolean().optional(),
})

export const BuiltinCategoryNameSchema = z.enum([
  "visual-engineering",
  "ultrabrain",
  "deep",
  "artistry",
  "quick",
  "unspecified-low",
  "unspecified-high",
  "writing",
])
```

### 片段 3：回退模型的联合类型定义

```typescript
// src/config/schema/fallback-models.ts (L20-23)
export const FallbackModelsSchema = z.union([
  z.string(),
  z.array(z.union([z.string(), FallbackModelObjectSchema])),
])
```

此定义展示了 Zod 联合类型的强大：fallback_models 可以是单个模型字符串，也可以是字符串和配置对象的混合数组，为配置提供极大的灵活性。

---

## 依赖关系

Schema 系统与以下组件存在依赖关系：

- **配置加载器** (`plugin-config.ts`)：调用 Schema 进行验证，处理验证结果
- **类型系统** (`z.infer`)：从 Schema 推导 TypeScript 类型，实现类型安全
- **迁移系统** (`migrateConfigFile`)：在验证前执行配置版本升级
- **日志系统** (`shared/log`)：记录验证错误和部分解析结果
- **Agent 系统**：通过 `AgentOverridesSchema` 定义 Agent 的可配置参数
- **Hook 系统**：通过 `HookNameSchema` 枚举所有可用 Hook

---

## 实战示例

### 示例 1：Schema 验证失败

假设用户在配置文件中设置了无效的 temperature 值：

```jsonc
// oh-my-opencode.jsonc
{
  "agents": {
    "sisyphus": {
      "temperature": 3.5  // 错误：超出 0-2 范围
    }
  }
}
```

验证流程：

1. `safeParse()` 检测到 `temperature` 值 3.5 超出 `.max(2)` 限制
2. 返回 `success: false` 和错误详情
3. 系统调用 `parseConfigPartially` 尝试部分解析
4. 由于 `agents` 段整体无效，该段被跳过
5. 其他有效配置（如 `disabled_hooks`）被保留
6. 用户收到错误提示：`agents.sisyphus.temperature: Number must be less than or equal to 2`

### 示例 2：默认值填充

考虑以下最小配置：

```jsonc
// oh-my-opencode.jsonc
{
  "background_task": {
    "defaultConcurrency": 3
  }
}
```

验证与填充过程：

1. `BackgroundTaskConfigSchema.safeParse()` 验证通过
2. 返回对象仅包含 `defaultConcurrency: 3`
3. 业务逻辑层使用默认值填充缺失字段：
   - `staleTimeoutMs` → 180000 (3分钟)
   - `maxToolCalls` → 200
   - `circuitBreaker` → 默认熔断器配置
4. 最终配置对象包含完整的功能配置

---

## 交叉引用

- 参见：[配置加载](./01-配置加载.md) - 了解配置文件的发现、解析和合并流程
- 参见：[模型解析](./03-模型解析.md) - 了解 Agent 模型如何通过 category 和 fallback 机制解析
- 参见：[配置迁移](./config-migration.md) - 了解旧版本配置的自动升级机制
- 参见：[Agent 系统](../04-agent-system/agent-overrides.md) - 了解 Agent 覆盖配置的详细用法
- 参见：[Hook 系统](../06-hook-system/hook-registration.md) - 了解 Hook 名称枚举的使用场景
