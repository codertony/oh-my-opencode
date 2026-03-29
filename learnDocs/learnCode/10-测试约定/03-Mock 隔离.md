# Mock 隔离与 CI 测试分割

> 所属模块：10-测试约定 | 三层模型：Layer 3 | 优先级：P3

---

## 架构位置

```
┌─────────────────────────────────────────────────────────────┐
│                    Layer 3 - 测试执行层                      │
├─────────────────────────────────────────────────────────────┤
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │ 测试设置      │  │ Given-When-  │  │ Mock 隔离    │      │
│  │ (test-setup) │  │ Then 模式    │  │ (本文档)     │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│                                              │              │
│                    ┌─────────────────────────┘              │
│                    ▼                                        │
│         ┌─────────────────────┐                            │
│         │   CI 测试分割策略    │                            │
│         │  (独立进程 vs 批量)  │                            │
│         └─────────────────────┘                            │
└─────────────────────────────────────────────────────────────┘
```

Mock 隔离位于测试执行层的最顶层，负责确保 Mock 重型测试不会污染模块缓存，同时协调 CI 环境中的测试并行化策略。

---

## 核心职责

Mock 隔离模块的核心职责是管理测试执行环境中的模块污染问题。在 Bun 测试框架中，`mock.module()` 会修改全局模块缓存，导致不同测试文件之间的状态泄漏。如果不加以隔离，一个测试文件中的 Mock 可能会意外影响其他测试的执行结果，造成难以调试的 flaky tests。

本模块通过两种策略解决这一问题：首先，识别使用 `mock.module()` 的 Mock 重型测试文件；其次，在 CI 环境中将这些测试与常规测试分离，使用独立的进程执行。这种分割策略既保证了测试的可靠性，又最大化了并行执行的效率。此外，模块还负责配置测试预加载脚本，确保每个测试文件在执行前都能重置共享状态。

---

## 代码索引 (Code Index)

| 功能点 | 文件路径 | 行号 | 函数/类 | 说明 |
|--------|----------|------|---------|------|
| Mock 模块污染防护 | `src/shared/model-capabilities.test.ts` | 6-11 | `mock.module()` | Mock 模块缓存防止本地磁盘缓存污染测试结果 |
| 测试预加载配置 | `bunfig.toml` | 1-2 | `preload` | 配置测试全局预加载脚本 |
| 状态重置逻辑 | `test-setup.ts` | 5-8 | `beforeEach` | 每个测试前重置 Claude Session 和模型回退状态 |
| Mock 恢复机制 | `src/shared/model-capabilities.test.ts` | 13-15 | `afterAll` + `mock.restore()` | 测试结束后恢复原始模块 |
| CI 隔离执行 | `.github/workflows/ci.yml` | 47-72 | `bun test` (独立进程) | Mock 重型测试单独进程执行 |
| 批量测试执行 | `.github/workflows/ci.yml` | 73-120 | `bun test` (批量) | 常规测试批量执行 |
| 依赖注入模式 | `src/tools/call-omo-agent/sync-executor.test.ts` | 35-44 | `createDependencies()` | 使用 Mock 依赖注入实现可测试性 |
| 异步模块导入 | `src/tools/call-omo-agent/sync-executor.test.ts` | 30-33 | `importExecuteSync()` | 动态导入避免模块缓存污染 |

---

## Mock 隔离策略

### 1. Mock 重型测试

Mock 重型测试是指使用 `mock.module()` 进行模块级 Mock 的测试文件。这类测试会修改全局模块缓存，必须在独立进程中执行。

```typescript
// src/shared/model-capabilities.test.ts:1-15
import { afterAll, describe, expect, test, mock } from "bun:test"

// Mock connected-providers-cache to prevent local disk cache from polluting test results.
// Without this, findProviderModelMetadata reads real cached model metadata (e.g., from opencode serve)
// which causes the "prefers runtime models.dev cache" test to get different values than expected.
mock.module("./connected-providers-cache", () => ({
  findProviderModelMetadata: () => undefined,
  readConnectedProvidersCache: () => null,
  hasConnectedProvidersCache: () => false,
  hasProviderModelsCache: () => false,
}))

afterAll(() => {
  mock.restore()
})
```

关键要点：
- 使用 `mock.module()` 在文件顶部进行模块级 Mock
- 在 `afterAll` 钩子中调用 `mock.restore()` 恢复原始模块
- 注释说明 Mock 的原因和防止的问题

### 2. CI 测试分割

CI 工作流将测试分为两个阶段：Mock 重型测试独立执行，常规测试批量执行。

```yaml
# .github/workflows/ci.yml:47-72
- name: Run mock-heavy tests (isolated)
  run: |
    # These files use mock.module() which pollutes module cache
    # Run them in separate processes to prevent cross-file contamination
    bun test src/plugin-handlers
    bun test src/hooks/atlas
    bun test src/hooks/compaction-context-injector
    bun test src/features/tmux-subagent
    bun test src/cli/doctor/formatter.test.ts
    bun test src/cli/doctor/format-default.test.ts
    bun test src/tools/call-omo-agent/sync-executor.test.ts
    bun test src/tools/call-omo-agent/session-creator.test.ts
    bun test src/tools/session-manager
    # ... 更多 Mock 重型测试
```

分割策略说明：
- Mock 重型测试：每个目录/文件独立 `bun test` 进程
- 常规测试：使用 `find` 排除已执行的 Mock 重型文件后批量执行
- 通过注释明确标记哪些文件使用 `mock.module()`

### 3. 测试并行化

Bun 测试框架天然支持并行执行，但 Mock 隔离要求特定的执行顺序：

```yaml
# .github/workflows/ci.yml:73-120
- name: Run remaining tests
  run: |
    # Enumerate subdirectories/files explicitly to EXCLUDE mock-heavy files
    # that were already run in isolation above.
    SHARED_FILES=$(find src/shared -name '*.test.ts' \
      ! -name 'model-capabilities.test.ts' \
      ! -name 'log-legacy-plugin-startup-warning.test.ts' \
      ! -name 'model-error-classifier.test.ts' \
      ! -name 'opencode-message-dir.test.ts' \
      | sort | tr '\n' ' ')
    bun test bin script src/config src/mcp src/index.test.ts \
      src/agents $SHARED_FILES \
      src/cli/run src/cli/config-manager src/cli/mcp-oauth \
      # ... 更多目录
```

并行化原则：
- Mock 重型测试串行执行（避免并行时的模块缓存冲突）
- 常规测试在单个 `bun test` 调用中批量执行（最大化并行效率）
- 使用 `find` 命令精确排除已隔离执行的文件

---

## 测试分割流程

测试分割遵循以下流程：

1. **识别阶段**：扫描所有 `*.test.ts` 文件，检测是否使用 `mock.module()`
2. **分类阶段**：将测试文件分为 Mock 重型和常规两类
3. **隔离执行**：Mock 重型测试使用独立进程逐个执行
4. **批量执行**：常规测试合并为单次 `bun test` 调用
5. **验证阶段**：确保所有测试文件都被覆盖，无遗漏

---

## 流程/架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                        测试分割流程                                  │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│  1. 扫描测试文件                                                     │
│     find src -name "*.test.ts"                                      │
└─────────────────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┴───────────────┐
              ▼                               ▼
┌─────────────────────────┐     ┌─────────────────────────────┐
│  2a. Mock 重型测试       │     │  2b. 常规测试                │
│  (包含 mock.module())    │     │  (无模块级 Mock)             │
│                         │     │                             │
│  - model-capabilities   │     │  - 工具测试                  │
│  - sync-executor        │     │  - 配置测试                  │
│  - session-manager      │     │  - 钩子测试                  │
└─────────────────────────┘     └─────────────────────────────┘
              │                               │
              ▼                               ▼
┌─────────────────────────┐     ┌─────────────────────────────┐
│  3a. 独立进程执行        │     │  3b. 批量执行                │
│                         │     │                             │
│  bun test file1.test.ts │     │  bun test src/tools         │
│  bun test file2.test.ts │     │  bun test src/config        │
│  (串行，避免污染)        │     │  (并行，最大化效率)          │
└─────────────────────────┘     └─────────────────────────────┘
              │                               │
              └───────────────┬───────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│  4. 合并结果                                                         │
│     - 所有测试通过 → 继续构建流程                                     │
│     - 任一测试失败 → 终止 CI 流程                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 关键代码片段

### 片段 1: 测试预加载配置

```toml
# bunfig.toml:1-2
[test]
preload = ["./test-setup.ts"]
```

此配置确保每个测试文件在执行前都加载 `test-setup.ts`，实现全局状态重置。

### 片段 2: 全局状态重置

```typescript
// test-setup.ts:1-8
import { beforeEach } from "bun:test"
import { _resetForTesting as resetClaudeSessionState } from "./src/features/claude-code-session-state/state"
import { _resetForTesting as resetModelFallbackState } from "./src/hooks/model-fallback/hook"

beforeEach(() => {
  resetClaudeSessionState()
  resetModelFallbackState()
})
```

每个测试前重置共享状态，防止测试间的状态泄漏。

### 片段 3: 依赖注入与 Mock

```typescript
// src/tools/call-omo-agent/sync-executor.test.ts:35-44
function createDependencies(overrides?: Partial<Dependencies>): Dependencies {
  return {
    createOrGetSession: mock(async () => ({ sessionID: "ses-test-123", isNew: true })),
    waitForCompletion: mock(async () => {}),
    processMessages: mock(async () => "agent response"),
    setSessionFallbackChain: mock(() => {}),
    clearSessionFallbackChain: mock(() => {}),
    ...overrides,
  }
}
```

使用依赖注入模式，将外部依赖作为参数传入，便于在测试中注入 Mock 实现。

---

## 依赖关系

Mock 隔离模块依赖以下组件：

- **Bun 测试框架**：提供 `mock.module()` 和 `mock()` 函数
- **CI 工作流**：`.github/workflows/ci.yml` 实现测试分割执行
- **测试设置**：`test-setup.ts` 提供全局状态重置
- **被测模块**：需要识别哪些模块使用模块级 Mock

---

## 实战示例

### 示例 1: Mock 重型测试隔离

场景：测试模型能力解析功能，需要 Mock 本地缓存模块以防止读取真实缓存数据。

```typescript
// src/shared/model-capabilities.test.ts
import { afterAll, describe, expect, test, mock } from "bun:test"

// 在文件顶部进行模块级 Mock
mock.module("./connected-providers-cache", () => ({
  findProviderModelMetadata: () => undefined,
  readConnectedProvidersCache: () => null,
  hasConnectedProvidersCache: () => false,
  hasProviderModelsCache: () => false,
}))

afterAll(() => {
  mock.restore()  // 测试结束后恢复
})

// 正常编写测试...
describe("getModelCapabilities", () => {
  test("uses runtime metadata before snapshot data", () => {
    // 测试实现...
  })
})
```

CI 配置中需要单独执行：

```yaml
- name: Run mock-heavy tests (isolated)
  run: |
    bun test src/shared/model-capabilities.test.ts
```

### 示例 2: CI 测试分割配置

完整的 CI 测试分割配置示例：

```yaml
# .github/workflows/ci.yml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: oven-sh/setup-bun@v2
        with:
          bun-version: latest

      - name: Install dependencies
        run: bun install

      # 阶段 1: Mock 重型测试独立执行
      - name: Run mock-heavy tests (isolated)
        run: |
          # 这些文件使用 mock.module()，会污染模块缓存
          # 使用独立进程执行，防止交叉污染
          bun test src/shared/model-capabilities.test.ts
          bun test src/shared/log-legacy-plugin-startup-warning.test.ts
          bun test src/tools/call-omo-agent/sync-executor.test.ts
          bun test src/tools/session-manager

      # 阶段 2: 常规测试批量执行
      - name: Run remaining tests
        run: |
          # 排除已执行的 Mock 重型文件
          SHARED_FILES=$(find src/shared -name '*.test.ts' \
            ! -name 'model-capabilities.test.ts' \
            ! -name 'log-legacy-plugin-startup-warning.test.ts' \
            | sort | tr '\n' ' ')
          bun test src/tools/ast-grep src/tools/grep src/tools/glob \
            src/config src/agents $SHARED_FILES
```

---

## 交叉引用

- 参见：[测试设置](./01-测试设置.md) - 了解全局测试配置和预加载脚本
- 参见：[Given-When-Then](./02-Given-When-Then.md) - 了解测试命名和结构约定
- 参见：[CI 工作流](../09-集成工作流/01-初始化序列.md) - 了解完整的 CI/CD 流程
- 参见：[Bun 测试文档](https://bun.sh/docs/cli/test) - 官方测试框架文档
