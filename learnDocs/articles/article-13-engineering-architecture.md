# 如何搭建自己的领域插件工程架构

> 本系列第13篇 | 基于项目版本: 2026-03-28 | 源码位置: `package.json`, `tsconfig.json`, `bunfig.toml`, `.github/workflows/`, `script/build-binaries.ts`

---

## 这篇要回答的问题

1. **为什么选择 Bun？**
2. **如何保证交付稳定性与护栏？**
3. **如何进行自动化测试？**
4. **如何构建跨平台二进制？**
5. **如何搭建 CI/CD？**
6. **从零搭建全套工程需要什么？**

---

## 源码里这个问题出现在哪里

### 构建配置: `package.json`

```json
{
  "name": "oh-my-opencode",
  "version": "1.0.0",
  "type": "module",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "bin": {
    "oh-my-opencode": "bin/oh-my-opencode.js"
  },
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.js"
    },
    "./schema.json": "./dist/oh-my-opencode.schema.json"
  },
  "files": ["dist", "bin", "postinstall.mjs"],
  "scripts": {
    "build": "bun build src/index.ts --outdir dist --target bun --format esm --external @ast-grep/napi && tsc --emitDeclarationOnly",
    "build:binaries": "bun run script/build-binaries.ts",
    "build:all": "bun run build && bun run build:binaries",
    "test": "bun test",
    "typecheck": "tsc --noEmit",
    "clean": "rm -rf dist"
  },
  "dependencies": {
    "zod": "^3.22.0",
    "@ast-grep/napi": "^0.x.x"
  },
  "devDependencies": {
    "bun-types": "latest",
    "typescript": "^5.0.0"
  }
}
```

### TypeScript 配置: `tsconfig.json`

```json
{
  "compilerOptions": {
    // 严格模式
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "strictFunctionTypes": true,
    
    // 模块系统
    "module": "ESNext",
    "moduleResolution": "bundler",
    "target": "ESNext",
    
    // 类型声明
    "declaration": true,
    "declarationDir": "dist",
    
    // Bun 类型
    "types": ["bun-types"],
    "skipLibCheck": true,
    "esModuleInterop": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

### 测试配置: `bunfig.toml`

```toml
[test]
preload = ["./test-setup.ts"]

# 测试设置
coverage = true
coverageThreshold = 0.8

# 超时设置
timeout = 30000
```

```typescript
// test-setup.ts
import { beforeEach } from "bun:test"

// 重置状态
beforeEach(() => {
  // 清理测试状态
  resetTestState()
})

function resetTestState() {
  // 清理各种全局状态
}
```

### CI/CD: `.github/workflows/ci.yml`

```yaml
name: CI

on:
  push:
    branches: [master, dev]
  pull_request:
    branches: [master, dev]

jobs:
  # Gate 1: 阻止直接 PR 到 master
  block-master-pr:
    if: github.event_name == 'pull_request' && github.base_ref == 'master'
    runs-on: ubuntu-latest
    steps:
      - run: |
          echo "PRs to master are not allowed. Please target dev branch."
          exit 1

  # Gate 2: 测试
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: oven-sh/setup-bun@v2
      - run: bun install
      
      # Mock-heavy tests（隔离运行）
      - name: Run mock-heavy tests
        run: |
          bun test src/plugin-handlers/
          bun test src/hooks/atlas/
      
      # 剩余测试（批量运行）
      - name: Run remaining tests
        run: bun test

  # Gate 3: 类型检查
  typecheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: oven-sh/setup-bun@v2
      - run: bun install
      - run: bun run typecheck

  # Gate 4: 构建
  build:
    needs: [test, typecheck]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: oven-sh/setup-bun@v2
      - run: bun install
      - run: bun run build
      
      # Schema 自动提交
      - name: Auto-commit schema
        if: github.ref == 'refs/heads/master'
        run: |
          git config user.name "github-actions[bot]"
          git add assets/*.schema.json
          git diff --staged --quiet || git commit -m "chore: update schema"
          git push
```

### 跨平台编译: `script/build-binaries.ts`

```typescript
#!/usr/bin/env bun
import { $ } from "bun"

const PLATFORMS = [
  { target: "bun-darwin-arm64", dir: "darwin-arm64", bin: "oh-my-opencode" },
  { target: "bun-darwin-x64", dir: "darwin-x64", bin: "oh-my-opencode" },
  { target: "bun-linux-x64", dir: "linux-x64", bin: "oh-my-opencode" },
  { target: "bun-linux-arm64", dir: "linux-arm64", bin: "oh-my-opencode" },
  { target: "bun-windows-x64", dir: "windows-x64", bin: "oh-my-opencode.exe" }
]

const ENTRY_POINT = "src/cli/index.ts"

for (const { target, dir, bin } of PLATFORMS) {
  const outfile = `packages/${dir}/bin/${bin}`
  
  console.log(`Building for ${target}...`)
  
  await $`bun build ${ENTRY_POINT} \
    --compile \
    --minify \
    --sourcemap \
    --bytecode \
    --target=${target} \
    --outfile=${outfile}`
    
  console.log(`✓ Built ${dir}`)
}

console.log('All platforms built successfully!')
```

---

## 它当前的设计方案是什么

### 13.1 为什么选择 Bun

```
Bun vs Node.js 对比：

| 特性 | Bun | Node.js |
|------|-----|---------|
| 启动速度 | ~2ms | ~30ms |
| 包管理 | 内置，并行 | npm/yarn，串行 |
| 测试 | 内置 (bun:test) | 需要 Jest |
| 打包 | 内置 (bun build) | 需要 webpack |
| 编译二进制 | 支持 (bun build --compile) | 需要 pkg |
| TypeScript | 原生支持 | 需要 ts-node |

选择 Bun 的原因：
1. 一站式：runtime + bundler + test + compile
2. 高性能：比 Node.js 快 3-10 倍
3. 原生 TS：无需额外配置
4. 跨平台编译：单命令生成多平台二进制
```

### 13.2 类型安全与护栏

```
TypeScript 严格模式：

✓ strict: true           # 启用所有严格检查
✓ noImplicitAny: true    # 禁止隐式 any
✓ strictNullChecks: true # 严格 null 检查
✓ strictFunctionTypes: true # 严格函数类型

反模式禁止（AGENTS.md）:
✗ NEVER use: as any
✗ NEVER use: @ts-ignore
✗ NEVER use: @ts-expect-error
✗ NEVER use: Empty catch blocks
✗ NEVER create: catch-all files (utils.ts)

模块化规则（.sisyphus/rules/）:
Rule 1: index.ts 只做入口，不放业务逻辑
Rule 2: 禁止 utils.ts / helpers.ts
Rule 3: 单一职责原则
Rule 4: 200 LOC 限制
```

### 13.3 自动化测试

```
测试模式：given/when/then

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

测试组织：
├── Co-located: *.test.ts 与源文件同目录
├── Preload: bunfig.toml 配置 test-setup.ts
└── CI 分割: mock-heavy 隔离运行

CI 测试分割：
├── Mock-heavy tests（隔离）
│   ├── src/plugin-handlers/
│   ├── src/hooks/atlas/
│   └── ...
└── Remaining tests（批量）
```

### 13.4 构建与跨平台二进制

```
构建流程（5步）：

Step 1: ESM Build
    bun build src/index.ts --outdir dist --target bun --format esm
    → dist/index.js

Step 2: TypeScript Declarations
    tsc --emitDeclarationOnly
    → dist/index.d.ts

Step 3: CLI Build
    bun build src/cli/index.ts --outdir dist/cli
    → dist/cli/index.js

Step 4: Schema Generation
    bun run script/build-schema.ts
    → dist/*.schema.json

Step 5: Platform Binaries
    bun run script/build-binaries.ts
    → packages/*/bin/oh-my-opencode

12平台支持：
├── macOS: arm64, x64, x64-baseline
├── Linux (glibc): x64, x64-baseline, arm64
├── Linux (musl): x64, x64-musl, arm64
└── Windows: x64, x64-baseline
```

### 13.5 CI/CD 架构

```
6个 Workflow：

├── ci.yml              # 主 CI（测试、类型检查、构建）
├── publish.yml         # 发布主包
├── publish-platform.yml # 发布平台二进制
├── sisyphus-agent.yml  # AI agent 工作流
├── cla.yml             # CLA 签署
└── lint-workflows.yml  # workflow 检查

CI 门禁：
├── block-master-pr     # 阻止直接 PR 到 master
├── test                # 测试
├── typecheck           # 类型检查
├── build               # 构建
└── draft-release       # 草稿发布

发布流程：
1. 版本计算
2. 检查是否已发布
3. 更新 package.json 版本
4. 构建主包
5. 发布到 npm
6. 触发平台二进制构建
7. 创建 GitHub Release
8. 合并到 master
```

---

## 这套方案解决了什么，牺牲了什么

### 解决的问题

**工程标准化**: 统一的构建、测试、发布流程

**跨平台**: 一套代码，多平台运行

**自动化**: CI/CD 自动化测试和发布

**类型安全**: 严格模式保证代码质量

### 牺牲的代价

**Bun 生态**: 比 Node.js 生态小

**学习成本**: 需要理解 Bun 特有的 API

**CI 复杂度**: 多平台构建配置复杂

**维护成本**: 需要维护多平台二进制

---

## 如果我要做自己的开发 Agent 套件，我会怎么迁移

### 从零搭建清单

```
Step 1: 初始化项目
    mkdir my-plugin && cd my-plugin
    bun init

Step 2: 安装依赖
    bun add zod @ast-grep/napi
    bun add -d bun-types typescript

Step 3: 配置文件
    ├── tsconfig.json      # TypeScript 配置
    ├── bunfig.toml        # Bun 配置
    ├── package.json       # 包配置
    └── test-setup.ts      # 测试设置

Step 4: 项目结构
    mkdir -p src/{agents,hooks,tools,config}
    mkdir -p script
    mkdir -p packages

Step 5: 实现入口
    src/index.ts
    src/plugin-interface.ts

Step 6: 测试
    bun test

Step 7: 构建
    bun run build

Step 8: 发布
    npm publish --access public
```

### 核心实现模板

```typescript
// src/index.ts
export async function createMyPlugin(ctx: PluginContext): Promise<PluginInterface> {
  // Step 1: 加载配置
  const config = await loadPluginConfig(ctx.directory, ctx)
  
  // Step 2: 创建 managers
  const managers = createManagers(config, ctx)
  
  // Step 3: 创建 tools
  const tools = createTools({ config, managers, ctx })
  
  // Step 4: 创建 hooks
  const hooks = createHooks({ config, tools, managers, ctx })
  
  // Step 5: 创建插件接口
  return createPluginInterface({ tools, hooks, config })
}

// src/plugin-interface.ts
export function createPluginInterface(deps: Dependencies): PluginInterface {
  return {
    tool: async () => deps.tools.getAll(),
    chat: {
      message: createChatMessageHandler(deps),
      params: createChatParamsHandler(deps),
      headers: createChatHeadersHandler(deps)
    },
    event: createEventHandler(deps),
    config: createConfigHandler(deps),
    toolExecute: {
      before: createToolExecuteBeforeHandler(deps),
      after: createToolExecuteAfterHandler(deps)
    }
  }
}
```

---

## 一个最小实验

### 实验1: 创建最小可运行插件

```typescript
// my-minimal-plugin.ts

// 1. 定义插件接口
interface MyPlugin {
  name: string
  version: string
  init: () => Promise<void>
}

// 2. 实现插件
const myPlugin: MyPlugin = {
  name: 'my-plugin',
  version: '1.0.0',
  
  async init() {
    console.log('My plugin initialized!')
    
    // 注册工具
    registerTool({
      name: 'my_hello',
      execute: async () => ({ message: 'Hello from my plugin!' })
    })
    
    // 注册 hook
    registerHook({
      name: 'my-hook',
      beforeChatMessage: async (msg) => {
        console.log('Intercepted:', msg)
        return msg
      }
    })
  }
}

// 3. 导出
export default myPlugin
```

### 实验2: 配置测试和构建

```json
// package.json
{
  "name": "my-minimal-plugin",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "build": "bun build src/index.ts --outdir dist",
    "test": "bun test",
    "typecheck": "tsc --noEmit"
  },
  "devDependencies": {
    "bun-types": "latest",
    "typescript": "^5.0.0"
  }
}
```

```typescript
// my-plugin.test.ts
import { describe, test, expect } from "bun:test"
import myPlugin from "./my-plugin"

describe("MyPlugin", () => {
  test("should have correct name", () => {
    expect(myPlugin.name).toBe("my-plugin")
  })
  
  test("should initialize", async () => {
    await expect(myPlugin.init()).resolves.toBeUndefined()
  })
})
```

---

## 总结

工程架构的核心要点：

1. **Bun**: 一站式 runtime + bundler + test + compile
2. **TypeScript**: 严格模式，类型安全
3. **测试**: given/when/then，CI 分割
4. **构建**: 5步流程，12平台支持
5. **CI/CD**: 6个 workflow，自动化门禁

**关键认知**: 好的工程架构让开发可持续，让发布可信赖。

---

## 系列完结

恭喜完成《Oh My OpenAgent 源码解析》系列学习！

### 回顾：8个核心问题

1. ✅ 一个复杂 Agent 系统到底在解决什么问题？
2. ✅ 任务是如何从"用户需求"变成"可执行工作流"的？
3. ✅ 多 Agent 为什么能比单 Agent 更稳定？
4. ✅ 上下文为什么会失控，系统如何控制它？
5. ✅ "记忆"到底应该分成哪几层？
6. ✅ 工具体系是如何决定 Agent 上限的？
7. ✅ Agent 为什么"能跑起来"不等于"能交付"？
8. ✅ 安全和权限为什么是 Agent 工程的硬门槛？

### 下一步

- **实践**: 基于学到的知识，搭建自己的 Agent 套件
- **深入**: 阅读 OMO 源码，理解更多细节
- **贡献**: 参与 OMO 开源项目，贡献代码
- **分享**: 把学到的知识分享给团队

### 资源

- **项目地址**: https://github.com/code-yeongyu/oh-my-openagent
- **文档**: `AGENTS.md`, `README.md`
- **社区**: Discord, GitHub Issues

**祝学习愉快，编码顺利！**

---

**系列文章**: 《Oh My OpenAgent 源码解析：从复杂 Agent 学习到开发效率套件落地》
- 共13篇，已全部完结
- 本文是系列收官之作
