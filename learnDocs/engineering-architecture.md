# 工程架构深度解析

> 本文档详细解析 Oh My OpenAgent 的工程架构，帮助读者理解如何搭建自己的领域插件系统。

---

## 一、技术栈概览

### 1.1 核心技术选型

| 技术 | 版本 | 选型理由 |
|------|------|---------|
| **Bun** | latest | 高性能运行时、内置 bundler/test/compile |
| **TypeScript** | ESNext | 类型安全、严格模式 |
| **Zod** | v4 | Schema 验证、类型推导 |
| **ESM** | - | 现代 module 系统 |

### 1.2 为什么选择 Bun？

```
传统 Node.js 方案：
Node.js + npm/yarn + webpack/rollup + jest → 多工具链、配置复杂

Bun 方案：
Bun（runtime + bundler + test + compile）→ 一站式、配置简单
```

**Bun 核心优势**:

| 能力 | Bun | Node.js |
|------|-----|---------|
| 启动速度 | ~2ms | ~30ms |
| 包安装 | 并行、快 | 串行、慢 |
| 测试运行 | 内置 | 需要 Jest/Mocha |
| 打包 | 内置 | 需要 webpack/rollup |
| 编译二进制 | 支持 | 需要 pkg/nexe |
| TypeScript | 原生支持 | 需要 ts-node |

**Bun-specific APIs 使用示例**:
```typescript
// 1. Shebang - 直接用 bun 运行
#!/usr/bin/env bun

// 2. Shell helper - 在 TS 中执行 shell
import { $ } from "bun"
await $`bun build src/index.ts --outdir dist`

// 3. Runtime info - 获取版本
const version = Bun.version

// 4. File I/O - 高性能文件操作
const file = Bun.file("path/to/file")
const content = await file.text()

// 5. HTTP Server - 内置服务器
Bun.serve({
  fetch(req) { return new Response("Hello") }
})
```

### 1.3 禁止的反模式

**类型安全禁止**:
```typescript
// ❌ NEVER
const data: any = fetchData()
// @ts-ignore
someFunction(badType)
// @ts-expect-error
anotherBadCall()
```

**架构禁止**:
```typescript
// ❌ NEVER create catch-all files
// utils.ts, helpers.ts, service.ts, common.ts

// ❌ NEVER empty catch blocks
try { something() }
catch(e) {} // 必须处理错误

// ❌ NEVER business logic in index.ts
// index.ts 只用于 re-export 和 wiring
```

---

## 二、类型安全与护栏

### 2.1 TypeScript 配置详解

```json
// tsconfig.json
{
  "compilerOptions": {
    // 严格模式
    "strict": true,                    // 启用所有严格检查
    "noImplicitAny": true,             // 禁止隐式 any
    "strictNullChecks": true,          // 严格 null 检查
    "strictFunctionTypes": true,       // 严格函数类型
    
    // 模块系统
    "module": "ESNext",                // ESNext 模块
    "moduleResolution": "bundler",     // bundler 模式
    "target": "ESNext",                // ESNext 目标
    
    // 类型声明
    "declaration": true,               // 生成 .d.ts
    "declarationDir": "dist",          // 声明文件输出目录
    
    // 其他
    "skipLibCheck": true,              // 跳过 lib 检查
    "esModuleInterop": true,           // ESM 互操作
    "types": ["bun-types"]             // 使用 bun-types，不用 @types/node
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

### 2.2 模块化架构规则

**Rule 1: index.ts 只做入口**
```typescript
// ✅ CORRECT: index.ts 只用于 re-export
export { createAgent } from "./create-agent"
export { AgentConfig } from "./types"
export { agentRegistry } from "./registry"

// ❌ WRONG: index.ts 包含业务逻辑
export function doSomething() {  // 应该在独立文件
  // ...
}
```

**Rule 2: 禁止 catch-all 文件**
```typescript
// ❌ WRONG: utils.ts 包含所有东西
// utils.ts
export function formatDate() {}
export function slugify() {}
export function retry() {}
export function debounce() {}

// ✅ CORRECT: 按职责拆分
// date-formatter.ts
export function formatDate() {}

// slugify.ts
export function slugify() {}

// retry.ts
export function retry() {}
```

**Rule 3: 单一职责**
```typescript
// ✅ CORRECT: 一个文件一个职责
// task-create.ts - 只负责创建任务
export function createTask(config: TaskConfig): Task {
  // ...
}

// ❌ WRONG: 一个文件多个职责
// task.ts
export function createTask() {}
export function listTasks() {}
export function deleteTask() {}
export function updateTask() {}
```

**Rule 4: 200 LOC 限制**
```typescript
// 文件超过 200 行代码 = 需要拆分信号

// 如何计数：
// ✓ COUNT: import, function, class, interface, if, for, return
// ✗ SKIP: blank lines, comments, prompt strings

// 示例：
const prompt = `
  You are an assistant.     // ← 不计数（prompt 内容）
  Follow these rules:
`;
```

### 2.3 运行时守卫

**Write-Existing-File-Guard**:
```typescript
// src/hooks/write-existing-file-guard/hook.ts
export function createWriteExistingFileGuardHook() {
  return {
    name: "write-existing-file-guard",
    async execute({ path, content }: WriteArgs) {
      // 检查文件是否存在
      if (await fileExists(path)) {
        // 检查是否有 read-before-write 权限
        if (!hasReadPermission(path)) {
          throw new Error(
            "File already exists. Use edit tool instead."
          )
        }
      }
      // 允许写入
      return proceed()
    }
  }
}
```

**Timeout Guard**:
```typescript
// src/hooks/ralph-loop/with-timeout.ts
export function withTimeout<T>(
  promise: Promise<T>,
  timeoutMs: number
): Promise<T> {
  return Promise.race([
    promise,
    new Promise<never>((_, reject) =>
      setTimeout(() => reject(new Error("Timeout")), timeoutMs)
    )
  ])
}
```

**Output Truncation**:
```typescript
// src/tools/lsp/constants.ts
export const DEFAULT_MAX_DIAGNOSTICS = 200

// src/tools/lsp/diagnostics-tool.ts
if (diagnostics.length > maxDiagnostics) {
  diagnostics = diagnostics.slice(0, maxDiagnostics)
  // 添加截断提示
}
```

### 2.4 Comment Checker（AI 生成注释检测）

```typescript
// src/hooks/comment-checker/hook.ts
export function createCommentCheckerHook() {
  return {
    name: "comment-checker",
    async afterEdit({ content }: EditArgs) {
      // 检测 AI 生成的注释模式
      const patterns = [
        /This function (does|performs|handles)/i,
        /Helper function for/i,
        /Utility to/i,
        // ... 更多模式
      ]
      
      for (const pattern of patterns) {
        if (pattern.test(content)) {
          warn("AI-generated comment detected. Consider rewriting.")
        }
      }
    }
  }
}
```

---

## 三、自动化测试架构

### 3.1 测试模式：Given/When/Then

```typescript
// bin/platform.test.ts
import { describe, test, expect } from "bun:test"

describe("getPlatformPackage", () => {
  describe("#given macOS ARM64 platform", () => {
    test("returns correct package name", () => {
      // Given
      const platform = "darwin-arm64"
      
      // When
      const result = getPlatformPackage(platform)
      
      // Then
      expect(result).toBe("oh-my-opencode-darwin-arm64")
    })
  })
  
  describe("#when getting platform package for unknown platform", () => {
    test("throws error", () => {
      // Given
      const platform = "unknown"
      
      // When & Then
      expect(() => getPlatformPackage(platform)).toThrow()
    })
  })
})
```

### 3.2 测试组织

**Co-located 模式**:
```
src/
├── tools/
│   ├── task/
│   │   ├── task-create.ts       # 实现
│   │   └── task-create.test.ts  # 测试（同目录）
│   └── session/
│       ├── session-manager.ts
│       └── session-manager.test.ts
```

**Preload 配置**:
```toml
# bunfig.toml
[test]
preload = ["./test-setup.ts"]
```

```typescript
// test-setup.ts
import { beforeEach } from "bun:test"
import { _resetForTesting as resetClaudeSessionState } from "./src/features/claude-code-session-state/state"
import { _resetForTesting as resetModelFallbackState } from "./src/hooks/model-fallback/hook"

beforeEach(() => {
  resetClaudeSessionState()
  resetModelFallbackState()
})
```

### 3.3 CI 测试分割策略

**为什么需要分割**:
```
问题：Mock-heavy tests 会污染 module cache
      导致后续测试失败

解决：将 Mock-heavy tests 隔离运行
      每个 test 文件独立进程
```

**CI 配置**:
```yaml
# .github/workflows/ci.yml
jobs:
  test:
    steps:
      # 1. Mock-heavy tests（隔离运行）
      - name: Run mock-heavy tests (isolated)
        run: |
          bun test src/plugin-handlers/
          bun test src/hooks/atlas/
          bun test src/features/tmux-subagent/
          bun test src/tools/ast-grep/
          bun test src/tools/lsp/
      
      # 2. 构建剩余测试列表
      - name: Build remaining tests list
        id: build-list
        run: |
          # 排除 mock-heavy tests
          SHARED_FILES=$(find src -name "*.test.ts" | grep -v "plugin-handlers\|atlas\|tmux\|ast-grep\|lsp")
          echo "files=$SHARED_FILES" >> $GITHUB_OUTPUT
      
      # 3. 剩余测试（批量运行）
      - name: Run remaining tests
        run: bun test ${{ steps.build-list.outputs.files }}
```

### 3.4 测试覆盖范围

```
覆盖统计：
├── 48 lifecycle hooks
├── 26 tools
├── 11 agents
├── 配置加载
├── MCP 集成
└── CLI 命令

测试类型：
├── 单元测试（大部分）
├── 集成测试（CLI, hooks）
└── E2E 测试（少数）
```

---

## 四、构建系统

### 4.1 构建流程详解

```
完整构建流程：

Step 1: ESM Build
    bun build src/index.ts \
      --outdir dist \
      --target bun \
      --format esm \
      --external @ast-grep/napi
    → 输出: dist/index.js

Step 2: TypeScript Declarations
    tsc --emitDeclarationOnly
    → 输出: dist/index.d.ts, dist/**/*.d.ts

Step 3: CLI Build
    bun build src/cli/index.ts \
      --outdir dist/cli \
      --target bun \
      --format esm \
      --external @ast-grep/napi
    → 输出: dist/cli/index.js

Step 4: Schema Generation
    bun run script/build-schema.ts
    → 输出: assets/oh-my-opencode.schema.json
             dist/oh-my-opencode.schema.json

Step 5: Platform Binaries
    bun run script/build-binaries.ts
    → 输出: packages/*/bin/oh-my-opencode(.exe)
```

### 4.2 外部依赖处理

```typescript
// 为什么需要 --external @ast-grep/napi？
// @ast-grep/napi 是 native 模块，不能打包

// package.json
{
  "dependencies": {
    "@ast-grep/napi": "^0.x.x"  // native 依赖
  }
}

// 构建时排除
bun build src/index.ts --external @ast-grep/napi
```

### 4.3 Schema 生成

```typescript
// script/build-schema.ts
import { buildSchemaDocument } from "./build-schema-document"
import { OhMyOpenCodeConfigSchema } from "../src/config/schema"

// 使用 zod-to-json-schema 转换
const schema = buildSchemaDocument(OhMyOpenCodeConfigSchema)

// 写入两个位置
await Bun.write("assets/oh-my-opencode.schema.json", JSON.stringify(schema))
await Bun.write("dist/oh-my-opencode.schema.json", JSON.stringify(schema))
```

### 4.4 跨平台二进制编译

**平台支持矩阵**:
```
├── macOS
│   ├── darwin-arm64        (Apple Silicon)
│   ├── darwin-x64          (Intel Mac)
│   └── darwin-x64-baseline (旧版 Intel)
├── Linux (glibc)
│   ├── linux-x64
│   ├── linux-x64-baseline
│   └── linux-arm64
├── Linux (musl/Alpine)
│   ├── linux-x64-musl
│   ├── linux-x64-musl-baseline
│   └── linux-arm64-musl
└── Windows
    ├── windows-x64
    └── windows-x64-baseline
```

**编译脚本**:
```typescript
// script/build-binaries.ts
#!/usr/bin/env bun
import { $ } from "bun"

const PLATFORMS = [
  { target: "bun-darwin-arm64", dir: "darwin-arm64", bin: "oh-my-opencode" },
  { target: "bun-darwin-x64", dir: "darwin-x64", bin: "oh-my-opencode" },
  // ... 更多平台
]

const ENTRY_POINT = "src/cli/index.ts"

for (const { target, dir, bin } of PLATFORMS) {
  const outfile = `packages/${dir}/bin/${bin}`
  
  await $`bun build ${ENTRY_POINT} \
    --compile \
    --minify \
    --sourcemap \
    --bytecode \
    --target=${target} \
    --outfile=${outfile}`
    
  console.log(`✓ Built ${dir}`)
}
```

### 4.5 包结构

```
发布到 npm 的内容：

oh-my-opencode（主包）:
├── dist/
│   ├── index.js          # ESM 入口
│   ├── index.d.ts        # 类型声明
│   ├── cli/
│   │   └── index.js      # CLI 入口
│   └── oh-my-opencode.schema.json
├── bin/
│   └── oh-my-opencode.js # 平台检测 wrapper
├── assets/
│   └── oh-my-opencode.schema.json
└── package.json

oh-my-opencode-darwin-arm64（平台包）:
├── bin/
│   └── oh-my-opencode    # 二进制
└── package.json

oh-my-opencode-linux-x64（平台包）:
├── bin/
│   └── oh-my-opencode
└── package.json

... 其他平台包
```

**Wrapper 脚本**:
```javascript
// bin/oh-my-opencode.js
#!/usr/bin/env node
const { platform, arch } = process
const { execSync } = require('child_process')
const path = require('path')

// 检测平台
const platformMap = {
  'darwin-arm64': 'darwin-arm64',
  'darwin-x64': 'darwin-x64',
  'linux-x64': 'linux-x64',
  'linux-arm64': 'linux-arm64',
  'win32-x64': 'windows-x64'
}

const key = `${platform}-${arch}`
const pkgName = `oh-my-opencode-${platformMap[key]}`

// 运行对应平台的二进制
const binPath = path.resolve(__dirname, `../node_modules/${pkgName}/bin/oh-my-opencode${platform === 'win32' ? '.exe' : ''}`)
execSync(`"${binPath}" ${process.argv.slice(2).join(' ')}`, { stdio: 'inherit' })
```

---

## 五、CI/CD 架构

### 5.1 Workflow 概览

```
.github/workflows/
├── ci.yml                  # 主 CI（测试、构建）
├── publish.yml             # 发布主包
├── publish-platform.yml    # 发布平台二进制
├── sisyphus-agent.yml      # AI agent 工作流
├── cla.yml                 # CLA 签署
└── lint-workflows.yml      # workflow 检查
```

### 5.2 CI Workflow 详解

```yaml
# ci.yml 完整流程

name: CI

on:
  push:
    branches: [master, dev]
  pull_request:
    branches: [master, dev]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  # Gate 1: 阻止直接 PR 到 master
  block-master-pr:
    if: github.event_name == 'pull_request' && github.base_ref == 'master'
    runs-on: ubuntu-latest
    steps:
      - run: |
          echo "❌ PRs to master are not allowed. Please target dev branch."
          exit 1

  # Gate 2: 测试
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: oven-sh/setup-bun@v2
      
      - run: bun install
      
      # Mock-heavy tests（隔离运行）
      - name: Run mock-heavy tests (isolated)
        run: |
          bun test src/plugin-handlers/
          bun test src/hooks/atlas/
          bun test src/features/tmux-subagent/
      
      # 剩余测试
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
      - name: Auto-commit schema changes
        if: github.ref == 'refs/heads/master'
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add assets/oh-my-opencode.schema.json
          git diff --staged --quiet || git commit -m "chore: update schema"
          git push

  # Gate 5: 草稿发布
  draft-release:
    needs: build
    if: github.ref == 'refs/heads/dev'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Create draft release
        run: |
          # 生成 release notes
          # 创建/更新 draft release
```

### 5.3 Publish Workflow 详解

```yaml
# publish.yml 核心步骤

jobs:
  test:
    # 同 ci.yml 的测试步骤

  calculate-version:
    needs: test
    outputs:
      version: ${{ steps.calc.outputs.version }}
    steps:
      - name: Calculate next version
        id: calc
        run: |
          # 根据输入或自动计算版本号
          VERSION="${{ github.event.inputs.version }}"
          if [ -z "$VERSION" ]; then
            # 自动计算
            VERSION=$(npm view oh-my-opencode version | awk -F. '{$3++; print $1"."$2"."$3}')
          fi
          echo "version=$VERSION" >> $GITHUB_OUTPUT

  publish-main:
    needs: calculate-version
    steps:
      - name: Check if already published
        run: |
          STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
            "https://registry.npmjs.org/oh-my-opencode/${{ needs.calculate-version.outputs.version }}")
          if [ "$STATUS" = "200" ]; then
            echo "Already published, skipping"
            exit 0
          fi
      
      - name: Update version
        run: |
          VERSION="${{ needs.calculate-version.outputs.version }}"
          jq --arg v "$VERSION" '.version = $v' package.json > tmp.json && mv tmp.json package.json
      
      - name: Build
        run: bun run build
      
      - name: Publish to npm
        run: npm publish --access public --provenance
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NODE_AUTH_TOKEN }}
      
      - name: Commit version bump
        run: |
          git config user.name "github-actions[bot]"
          git add package.json
          git commit -m "chore: bump version to ${{ needs.calculate-version.outputs.version }}"
          git push

  trigger-platform:
    needs: [calculate-version, publish-main]
    uses: ./.github/workflows/publish-platform.yml
    with:
      version: ${{ needs.calculate-version.outputs.version }}
```

### 5.4 Platform Binary Workflow

```yaml
# publish-platform.yml

jobs:
  build:
    strategy:
      matrix:
        platform:
          - darwin-arm64
          - darwin-x64
          - linux-x64
          - linux-arm64
          - windows-x64
          # ... 更多平台
    runs-on: ${{ matrix.platform == 'windows-x64' && 'windows-latest' || 'ubuntu-latest' }}
    steps:
      - name: Build binary
        run: |
          PLATFORM="${{ matrix.platform }}"
          TARGET="bun-${PLATFORM}"
          OUTPUT="packages/${PLATFORM}/bin/oh-my-opencode"
          
          bun build src/cli/index.ts \
            --compile \
            --minify \
            --sourcemap \
            --bytecode \
            --target=$TARGET \
            --outfile=$OUTPUT
      
      - name: Compress binary
        run: |
          if [[ "$PLATFORM" == windows-* ]]; then
            7z a binary-${PLATFORM}.zip packages/${PLATFORM}
          else
            tar -czvf binary-${PLATFORM}.tar.gz packages/${PLATFORM}
          fi
      
      - uses: actions/upload-artifact@v4
        with:
          name: binary-${{ matrix.platform }}
          path: binary-${{ matrix.platform }}.*

  publish:
    needs: build
    strategy:
      matrix:
        platform: [...] # 同上
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: binary-${{ matrix.platform }}
      
      - name: Publish to npm
        run: |
          cd packages/${{ matrix.platform }}
          npm publish --access public --provenance
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NODE_AUTH_TOKEN }}
```

### 5.5 Quality Gates

```yaml
# lint-workflows.yml
# 确保 workflow 文件语法正确

on:
  push:
    paths: ['.github/workflows/**']

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Install actionlint
        run: |
          curl -sL https://github.com/rhysd/actionlint/releases/latest/download/actionlint_linux_amd64.tar.gz | tar xz
          sudo mv actionlint /usr/local/bin/
      
      - name: Run actionlint
        run: actionlint -color -shellcheck=.github/workflows/*.yml
```

---

## 六、从零搭建自己的插件工程

### 6.1 初始化步骤

```bash
# 1. 创建项目
mkdir my-domain-plugin && cd my-domain-plugin
bun init

# 2. 项目结构
mkdir -p src/{agents,hooks,tools,config,mcp}
mkdir -p script
mkdir -p packages

# 3. 安装依赖
bun add zod @ast-grep/napi
bun add -d bun-types typescript

# 4. 复制配置模板
# tsconfig.json, bunfig.toml, package.json 模板见下文
```

### 6.2 配置模板

**package.json**:
```json
{
  "name": "my-domain-plugin",
  "version": "0.1.0",
  "type": "module",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "bin": {
    "my-plugin": "bin/my-plugin.js"
  },
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.js"
    },
    "./schema.json": "./dist/my-plugin.schema.json"
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

**tsconfig.json**:
```json
{
  "compilerOptions": {
    "strict": true,
    "module": "ESNext",
    "moduleResolution": "bundler",
    "target": "ESNext",
    "declaration": true,
    "declarationDir": "dist",
    "outDir": "dist",
    "rootDir": "src",
    "skipLibCheck": true,
    "esModuleInterop": true,
    "types": ["bun-types"]
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

**bunfig.toml**:
```toml
[test]
preload = ["./test-setup.ts"]
```

### 6.3 核心实现模板

**src/index.ts**:
```typescript
import type { PluginContext, PluginInterface } from "opencode"
import { loadPluginConfig } from "./plugin-config"
import { createManagers } from "./managers"
import { createTools } from "./plugin/tool-registry"
import { createHooks } from "./create-hooks"
import { createPluginInterface } from "./plugin-interface"

export async function createMyPlugin(ctx: PluginContext): Promise<PluginInterface> {
  // Step 1: Load config
  const config = await loadPluginConfig(ctx.directory, ctx)
  
  // Step 2: Create managers
  const managers = createManagers(config, ctx)
  
  // Step 3: Create tools
  const tools = createTools({ config, managers, ctx })
  
  // Step 4: Create hooks
  const hooks = createHooks({ config, tools, managers, ctx })
  
  // Step 5: Create plugin interface
  return createPluginInterface({ tools, hooks, config })
}
```

**src/plugin-interface.ts**:
```typescript
import type { PluginInterface } from "opencode"

export function createPluginInterface(deps: Dependencies): PluginInterface {
  return {
    // Tool hook
    tool: async () => deps.tools.getAll(),
    
    // Chat hooks
    chat: {
      message: createChatMessageHandler(deps),
      params: createChatParamsHandler(deps),
      headers: createChatHeadersHandler(deps),
    },
    
    // Event hook
    event: createEventHandler(deps),
    
    // Config hook
    config: createConfigHandler(deps),
    
    // Tool execution hooks
    toolExecute: {
      before: createToolExecuteBeforeHandler(deps),
      after: createToolExecuteAfterHandler(deps),
    },
  }
}
```

### 6.4 添加 Agent/Hook/Tool

**添加 Agent**:
```typescript
// src/agents/my-agent/agent.ts
import { createAgent } from "../factory"

export const myAgent = createAgent({
  name: "my-agent",
  description: "My domain-specific agent",
  mode: "subagent",
  
  systemPrompt: `
    You are a domain-specific agent for...
  `,
  
  tools: ["read", "grep", "glob"], // 只读工具
  canDelegate: false,
})
```

**添加 Hook**:
```typescript
// src/hooks/my-hook/hook.ts
export function createMyHook(deps: Dependencies) {
  return {
    name: "my-hook",
    
    async beforeToolExecute(args: ToolArgs) {
      // 拦截工具执行
      if (args.tool === "write") {
        // 自定义逻辑
      }
      return { proceed: true }
    },
  }
}

// 注册到 src/create-hooks.ts
```

**添加 Tool**:
```typescript
// src/tools/my-tool/tool.ts
import { createTool } from "../factory"

export const myTool = createTool({
  name: "my_tool",
  description: "My domain-specific tool",
  
  parameters: z.object({
    input: z.string().describe("Input parameter"),
  }),
  
  async execute({ input }, ctx) {
    // 工具逻辑
    return { result: "..." }
  },
})

// 注册到 src/plugin/tool-registry.ts
```

### 6.5 发布 Checklist

- [ ] 代码通过 `bun run typecheck`
- [ ] 测试通过 `bun test`
- [ ] 构建成功 `bun run build`
- [ ] 版本号已更新
- [ ] CHANGELOG 已更新
- [ ] README 已更新
- [ ] 发布到 npm: `npm publish --access public`
- [ ] 创建 GitHub Release

---

## 七、最佳实践总结

### 7.1 架构原则

1. **单一职责**: 一个文件一个职责
2. **依赖注入**: 通过 deps 传递依赖
3. **工厂模式**: createXXX 创建所有组件
4. **类型安全**: strict mode, no any
5. **测试覆盖**: given/when/then 风格

### 7.2 性能优化

1. **并行执行**: 独立任务并行运行
2. **懒加载**: 按需加载模块
3. **缓存**: Bun 内置缓存机制
4. **Compaction**: 上下文压缩

### 7.3 安全实践

1. **权限分离**: 不同 agent 不同权限
2. **Guard hooks**: 关键操作拦截
3. **Timeout**: 防止无限等待
4. **Output truncation**: 防止输出爆炸

### 7.4 可维护性

1. **文档**: AGENTS.md 作为项目知识库
2. **测试**: 覆盖核心逻辑
3. **CI/CD**: 自动化质量门禁
4. **Versioning**: 语义化版本控制

---

**文档完结** | 希望这篇文档能帮助你搭建自己的领域插件系统！