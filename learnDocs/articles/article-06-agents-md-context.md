# AGENTS.md、rules 与目录上下文注入机制

> 本系列第6篇 | 基于项目版本: 2026-03-28 | 源码位置: `AGENTS.md`, `src/plugin/chat-message.ts`, `src/plugin-config.ts`

---

## 这篇要回答的问题

1. **project / global 规则如何生效？**
2. **目录级上下文如何叠加？**
3. **规则优先级是什么？**
4. **为什么要 commit AGENTS.md？**

---

## 源码里这个问题出现在哪里

### AGENTS.md 项目规则

项目根目录的 `AGENTS.md` 是核心知识库：

```markdown
# oh-my-opencode — O P E N C O D E Plugin

**Generated:** 2026-03-06 | **Commit:** 7fe44024 | **Branch:** dev

## STRUCTURE
```
oh-my-opencode/
├── src/
│   ├── index.ts              # Plugin entry
│   ├── plugin-config.ts      # JSONC multi-level config
│   ├── agents/               # 11 agents
│   ├── hooks/                # 48 lifecycle hooks
│   ├── tools/                # 26 tools
│   └── ...
```

## INITIALIZATION FLOW
```
OhMyOpenCodePlugin(ctx)
  ├─→ loadPluginConfig()
  ├─→ createManagers()
  ├─→ createTools()
  ├─→ createHooks()
  └─→ createPluginInterface()
```

## 8 OPENCODE HOOK HANDLERS
| Handler | Purpose |
|---------|---------|
| `config` | 6-phase config loading |
| `tool` | 26 registered tools |
| `chat.message` | Context injection |

## CONVENTIONS
- **Runtime**: Bun only
- **TypeScript**: strict mode, ESNext
- **Test**: Bun test, given/when/then style
```

### 上下文注入: `src/plugin/chat-message.ts`

```typescript
// src/plugin/chat-message.ts 简化版
export function createChatMessageHandler(deps: Dependencies) {
  return {
    name: "chat-message",
    
    async before({ message, context }) {
      // 1. 加载多级 AGENTS.md
      const agentsMdContent = await loadAgentsMdHierarchy(context.directory)
      
      // 2. 合并各级规则
      const mergedRules = mergeRules([
        await loadGlobalAgentsMd(),      // ~/.config/opencode/AGENTS.md
        await loadProjectAgentsMd(),     // ./AGENTS.md
        await loadDirectoryAgentsMd(context.directory) // 子目录 AGENTS.md
      ])
      
      // 3. 注入到 prompt
      return {
        ...message,
        system: `${mergedRules}\n\n${message.system || ''}`
      }
    }
  }
}

// 多级 AGENTS.md 加载
async function loadAgentsMdHierarchy(currentDir: string): Promise<string> {
  const hierarchy = []
  
  // 1. Global 级别
  const globalAgents = await readFile(
    path.join(os.homedir(), '.config/opencode/AGENTS.md')
  )
  if (globalAgents) hierarchy.push({ level: 'global', content: globalAgents })
  
  // 2. Project 级别
  const projectRoot = await findProjectRoot(currentDir)
  const projectAgents = await readFile(
    path.join(projectRoot, 'AGENTS.md')
  )
  if (projectAgents) hierarchy.push({ level: 'project', content: projectAgents })
  
  // 3. Directory 级别（递归向上）
  let dir = currentDir
  while (dir.startsWith(projectRoot) && dir !== projectRoot) {
    const dirAgents = await readFile(path.join(dir, 'AGENTS.md'))
    if (dirAgents) {
      hierarchy.push({ level: 'directory', path: dir, content: dirAgents })
    }
    dir = path.dirname(dir)
  }
  
  return mergeHierarchy(hierarchy)
}
```

### 配置加载: `src/plugin-config.ts`

```typescript
// src/plugin-config.ts 简化版
export async function loadPluginConfig(
  directory: string,
  ctx: PluginContext
): Promise<OhMyOpenCodeConfig> {
  
  // 1. 加载默认配置
  const defaults = getDefaultConfig()
  
  // 2. 加载用户级配置
  const userConfig = await loadUserConfig()
  
  // 3. 加载项目级配置
  const projectConfig = await loadProjectConfig(directory)
  
  // 4. 深度合并（注意 disabled_* 是数组合并）
  const merged = deepMerge(defaults, userConfig, projectConfig, {
    arrayMerge: (target, source, key) => {
      // disabled_* 字段取并集
      if (key.startsWith('disabled_')) {
        return [...new Set([...target, ...source])]
      }
      return source
    }
  })
  
  // 5. Zod 验证
  return OhMyOpenCodeConfigSchema.parse(merged)
}
```

---

## 它当前的设计方案是什么

### 6.1 AGENTS.md 层级

```
AGENTS.md 层级结构：

~/.config/opencode/AGENTS.md     ← Global（用户级）
    │
    └── 个人偏好设置
        ├── 喜欢的代码风格
        ├── 个人 aliases
        └── 自定义规则
        
            ↓ 覆盖
            
项目根目录/AGENTS.md              ← Project（项目级）
    │
    └── 项目特定规范
        ├── 项目架构说明
        ├── 编码规范
        └── 团队约定
        
            ↓ 合并（叠加）
            
    src/components/AGENTS.md      ← Directory（目录级）
        └── 组件特定规则
        
    src/utils/AGENTS.md           ← Directory（目录级）
        └── 工具函数规则
```

**叠加规则**:
- Global → Project 是**覆盖**（Project 优先）
- Project → Directory 是**合并**（Directory 补充）
- `disabled_*` 数组取**并集**

### 6.2 注入时机

```
注入时机（两个时间点）：

1. 初始化阶段：
   loadPluginConfig()
       ↓
   读取 AGENTS.md
       ↓
   缓存到内存
   
2. 每条消息：
   chat.message hook
       ↓
   重新加载（支持热更新）
       ↓
   注入到 prompt
       ↓
   发送给 LLM
```

### 6.3 规则优先级

```
优先级（从高到低）：

┌────────────────────────────────────────┐
│ 1. Directory AGENTS.md                  │  ← 最高
│    (子目录规则，补充性)                 │
├────────────────────────────────────────┤
│ 2. Project AGENTS.md                    │
│    (项目规则，覆盖 global)              │
├────────────────────────────────────────┤
│ 3. Global AGENTS.md                     │
│    (用户级默认规则)                     │
├────────────────────────────────────────┤
│ 4. 系统默认                              │  ← 最低
└────────────────────────────────────────┘

特殊规则：
- disabled_* 数组：取并集（不是覆盖）
  ├── disabled_agents: global + project
  ├── disabled_hooks: global + project
  └── disabled_tools: global + project
```

### 6.4 为什么 commit AGENTS.md

| 原因 | 说明 |
|------|------|
| **团队共享** | 确保所有团队成员使用相同的规则 |
| **版本控制** | 规则变更可追溯，可回滚 |
| **环境一致性** | CI/CD 和本地开发使用相同规则 |
| **知识沉淀** | 项目知识保存在代码库中 |
| **新人友好** | 新成员通过 AGENTS.md 快速了解项目 |

---

## 这套方案解决了什么，牺牲了什么

### 解决的问题

**项目知识持久化**: AGENTS.md 作为代码的一部分，不会丢失

**团队协作**: 团队共享相同的规则和约定

**环境一致**: 不同环境（本地/CI）使用相同的配置

**层级细化**: 支持全局 → 项目 → 目录的多级规则

### 牺牲的代价

**规则文件维护成本**: 需要持续更新 AGENTS.md

**版本控制冲突**: 多人同时修改 AGENTS.md 可能产生冲突

**加载性能**: 每次消息都要读取和合并规则

**理解成本**: 需要理解多级规则的叠加逻辑

---

## 如果我要做自己的开发 Agent 套件，我会怎么迁移

### AGENTS.md 模板设计

```markdown
# Project AGENTS.md 模板

## 项目概述
- 项目名称: xxx
- 技术栈: xxx
- 主要模块: xxx

## 编码规范
### 命名规范
- 文件: kebab-case
- 组件: PascalCase
- 函数: camelCase

### 代码风格
- 缩进: 2 spaces
- 引号: single quote
- 分号: required

## 项目结构
```
src/
├── components/    # UI 组件
├── utils/         # 工具函数
├── hooks/         # 自定义 hooks
└── types/         # 类型定义
```

## 常用命令
- 测试: npm test
- 构建: npm run build
- 开发: npm run dev

## 注意事项
- xxx 模块是核心，修改需谨慎
- xxx 依赖外部 API，需要配置 API key
```

### 目录级规则示例

```markdown
# src/components/AGENTS.md

## 本目录规则
- 所有组件必须使用 TypeScript
- 必须包含单元测试
- Props 必须定义 interface

## 示例
```tsx
interface ButtonProps {
  label: string
  onClick: () => void
  disabled?: boolean
}

export const Button: React.FC<ButtonProps> = (props) => {
  // ...
}
```
```

### 配置加载代码

```typescript
// 多级配置加载器
export class ConfigLoader {
  private cache: Map<string, Config> = new Map()
  
  async load(directory: string): Promise<Config> {
    // 检查缓存
    if (this.cache.has(directory)) {
      return this.cache.get(directory)!
    }
    
    // 加载各级配置
    const global = await this.loadGlobalConfig()
    const project = await this.loadProjectConfig(directory)
    const dir = await this.loadDirectoryConfig(directory)
    
    // 合并
    const merged = this.mergeConfigs(global, project, dir)
    
    // 缓存
    this.cache.set(directory, merged)
    
    return merged
  }
  
  private mergeConfigs(...configs: Config[]): Config {
    return configs.reduce((acc, curr) => ({
      ...acc,
      ...curr,
      // 特殊处理 disabled_ 数组
      disabled_tools: [...acc.disabled_tools, ...curr.disabled_tools],
      disabled_hooks: [...acc.disabled_hooks, ...curr.disabled_hooks],
    }))
  }
}
```

---

## 一个最小实验

### 实验1: 创建测试 AGENTS.md

```bash
# 1. 创建项目级 AGENTS.md
cat > AGENTS.md << 'EOF'
# My Project

## 规范
- 使用 TypeScript
- 使用 functional components

## 结构
src/
├── components/
├── utils/
└── hooks/
EOF

# 2. 创建目录级 AGENTS.md
mkdir -p src/components
cat > src/components/AGENTS.md << 'EOF'
# Components 规则

- 必须包含 PropTypes 或 interface
- 必须包含单元测试
EOF
```

### 实验2: 观察规则注入

```bash
# 开启调试模式，观察 prompt 变化
export OMO_DEBUG=true

# 发起一个请求
# 观察日志中 AGENTS.md 的加载顺序和内容
```

**观察重点**:
- 哪一级 AGENTS.md 被加载了？
- 各级内容是如何合并的？
- 最终注入到 prompt 的是什么？

---

## 总结

AGENTS.md 机制的核心价值：

1. **层级规则**: Global → Project → Directory 的多级规则体系
2. **动态注入**: 每条消息自动加载和注入
3. **团队协作**: 通过版本控制共享规则
4. **知识沉淀**: 项目知识保存在代码库中

**关键认知**: AGENTS.md 不是"提示词模板"，而是**项目的"外部大脑"**。

---

## 下一步

阅读下一篇：**《Context overload、压缩与会话恢复》**，理解长任务如何不丢上下文。

---

**系列文章**: 《Oh My OpenAgent 源码解析：从复杂 Agent 学习到开发效率套件落地》
- ...
- 第5篇: 多 Agent 委派不是炫技，而是稳定性交换
- 第6篇: AGENTS.md、rules 与目录上下文注入机制（本文）
- 第7篇: Context overload、压缩与会话恢复
- ...（共13篇）
