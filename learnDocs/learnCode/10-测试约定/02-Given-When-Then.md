# Given-When-Then 测试模式

> 所属模块：10-测试约定 | 三层模型：Layer 3 | 优先级：P3

---

## 架构位置

```
┌─────────────────────────────────────────────────────────────┐
│                    Layer 3 - 测试层                          │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              测试约定 (Testing Conventions)          │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  │   │
│  │  │  测试设置   │  │Given-When-│  │  Mock 隔离   │  │   │
│  │  │             │  │   Then    │  │             │  │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│                    Layer 2 - 业务逻辑层                      │
└─────────────────────────────────────────────────────────────┘
```

Given-When-Then 模式位于测试层的核心位置，是项目测试编写的核心约定。它定义了测试用例的结构化组织方式，确保每个测试都清晰地表达前置条件、执行动作和预期结果。

---

## 核心职责

Given-When-Then 模式承担着以下关键职责：

**语义清晰**：通过明确的三个阶段划分，使测试用例的意图一目了然。Given 阶段描述测试的前置条件和初始状态，When 阶段描述被测系统的触发动作，Then 阶段描述预期的验证结果。这种结构让读者能够快速理解测试在验证什么行为。

**行为驱动**：该模式源自行为驱动开发（BDD），强调从用户行为角度描述测试，而非从实现细节角度。这有助于保持测试与业务需求的一致性，即使实现细节发生变化，测试的语义仍然有效。

**可维护性**：结构化的测试组织方式使得测试代码更易于维护。当测试失败时，开发者可以快速定位是前置条件设置问题、执行逻辑问题，还是预期结果断言问题。

**团队协作**：统一的测试编写风格降低了团队成员之间的认知负担。无论是编写新测试还是审查现有测试，团队成员都能遵循相同的模式和约定。

---

## 代码索引 (Code Index)

| 功能点 | 文件路径 | 行号 | 函数/类 | 说明 |
|--------|----------|------|---------|------|
| Given 注释标记 | `src/tools/task/task-create.test.ts` | 44-47 | `//#given` | 前置条件注释标记 |
| When 注释标记 | `src/tools/task/task-create.test.ts` | 49-51 | `//#when` | 执行动作注释标记 |
| Then 注释标记 | `src/tools/task/task-create.test.ts` | 53-56 | `//#then` | 验证结果注释标记 |
| Given 测试命名 | `src/hooks/todo-continuation-enforcer/session-state.test.ts` | 18 | `given repeated incomplete counts` | 测试用例命名模式 |
| When 测试命名 | `src/hooks/todo-continuation-enforcer/session-state.test.ts` | 36 | `given injection did not succeed` | 测试用例命名模式 |
| Then 验证断言 | `src/hooks/todo-continuation-enforcer/session-state.test.ts` | 30-33 | `expect().toBe()` | 结果验证断言 |
| 嵌套 describe | `src/plugin/chat-message.test.ts` | 58 | `describe("createChatMessageHandler")` | 测试套件组织 |
| 测试上下文设置 | `src/plugin/chat-message.test.ts` | 60-64 | `//#given` | 复杂前置条件设置 |

---

## Given-When-Then 模式详解

### 1. Given（前置条件）

Given 阶段负责设置测试执行前的初始状态。这包括准备输入数据、初始化被测对象、设置 Mock 返回值、配置环境变量等。

**代码示例**（来自 `src/tools/task/task-create.test.ts:44-47`）：

```typescript
//#given
const args = {
  subject: "Implement authentication",
}
```

**最佳实践**：
- 保持 Given 阶段简洁，只包含与当前测试直接相关的设置
- 使用描述性变量名，让数据的含义一目了然
- 对于复杂的前置条件，可以提取为辅助函数

**另一个示例**（来自 `src/hooks/todo-continuation-enforcer/session-state.test.ts:19-21`）：

```typescript
// given
const sessionID = "ses-stagnation"
const state = sessionStateStore.getState(sessionID)
```

### 2. When（执行动作）

When 阶段描述触发被测系统的动作。这通常是调用被测函数、触发事件、或者执行某个操作。

**代码示例**（来自 `src/tools/task/task-create.test.ts:49-51`）：

```typescript
//#when
const resultStr = await tool.execute(args, TEST_CONTEXT)
const result = JSON.parse(resultStr)
```

**最佳实践**：
- When 阶段应该只包含一个核心动作
- 如果动作返回 Promise，使用 await 等待完成
- 对于需要验证的返回值，在此阶段捕获

**另一个示例**（来自 `src/plugin/chat-message.test.ts:66-67`）：

```typescript
//#when
await handler(input, output)
```

### 3. Then（验证结果）

Then 阶段包含对预期结果的断言验证。这是测试的核心，验证被测系统是否按照预期行为执行。

**代码示例**（来自 `src/tools/task/task-create.test.ts:53-56`）：

```typescript
//#then
expect(result).toHaveProperty("task")
expect(result.task).toHaveProperty("id")
expect(result.task.subject).toBe("Implement authentication")
```

**最佳实践**：
- 每个 Then 阶段应该验证一个明确的预期
- 使用具体的断言方法（toBe, toEqual, toHaveProperty 等）
- 避免过于宽泛的断言，确保验证的是具体行为

**另一个示例**（来自 `src/hooks/todo-continuation-enforcer/session-state.test.ts:30-33`）：

```typescript
// then
expect(firstUpdate.stagnationCount).toBe(0)
expect(secondUpdate.stagnationCount).toBe(1)
expect(thirdUpdate.stagnationCount).toBe(2)
```

---

## 测试模式流程

Given-When-Then 模式的完整执行流程如下：

1. **测试套件加载**：Bun 测试运行器加载测试文件，解析 describe 和 test 定义
2. **前置钩子执行**：运行 beforeEach 钩子，重置全局状态
3. **Given 阶段**：设置测试的前置条件和初始状态
4. **When 阶段**：执行被测动作，触发系统行为
5. **Then 阶段**：验证执行结果是否符合预期
6. **断言评估**：Bun 评估所有 expect 断言，确定测试通过或失败
7. **后置清理**：运行 afterEach 钩子，准备下一个测试

---

## 流程/架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                    Given-When-Then 测试流程                      │
└─────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  1. 测试套件加载                                                 │
│     describe() → test() 定义收集                                │
└─────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  2. 前置钩子 (beforeEach)                                        │
│     重置全局状态 → 清理 Mock → 准备环境                          │
└─────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  3. Given 阶段（前置条件）                                       │
│     ┌───────────────────────────────────────────────────────┐  │
│     │  • 准备输入数据                                        │  │
│     │  • 初始化被测对象                                      │  │
│     │  • 设置 Mock 返回值                                    │  │
│     │  • 配置环境变量                                        │  │
│     └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  4. When 阶段（执行动作）                                        │
│     ┌───────────────────────────────────────────────────────┐  │
│     │  • 调用被测函数/方法                                   │  │
│     │  • 触发事件                                            │  │
│     │  • 执行操作                                            │  │
│     │  • 捕获返回值                                          │  │
│     └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  5. Then 阶段（验证结果）                                        │
│     ┌───────────────────────────────────────────────────────┐  │
│     │  • expect(result).toBe(expected)                      │  │
│     │  • expect(result).toHaveProperty(key)                 │  │
│     │  • expect(fn).toThrow()                               │  │
│     │  • expect(array).toContain(item)                      │  │
│     └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  6. 后置钩子 (afterEach)                                         │
│     清理资源 → 恢复 Mock → 准备下一个测试                        │
└─────────────────────────────────────────────────────────────────┘
```

---

## 关键代码片段

### 片段 1：基本 Given-When-Then 结构

```typescript
// src/tools/task/task-create.test.ts:43-57
test("creates task with required subject field", async () => {
  //#given
  const args = {
    subject: "Implement authentication",
  }

  //#when
  const resultStr = await tool.execute(args, TEST_CONTEXT)
  const result = JSON.parse(resultStr)

  //#then
  expect(result).toHaveProperty("task")
  expect(result.task).toHaveProperty("id")
  expect(result.task.subject).toBe("Implement authentication")
})
```

### 片段 2：使用 given 前缀的测试命名

```typescript
// src/hooks/todo-continuation-enforcer/session-state.test.ts:18-34
test("given repeated incomplete counts after a continuation, tracks stagnation", () => {
  // given
  const sessionID = "ses-stagnation"
  const state = sessionStateStore.getState(sessionID)

  // when
  const firstUpdate = sessionStateStore.trackContinuationProgress(sessionID, 2)
  state.awaitingPostInjectionProgressCheck = true
  const secondUpdate = sessionStateStore.trackContinuationProgress(sessionID, 2)
  state.awaitingPostInjectionProgressCheck = true
  const thirdUpdate = sessionStateStore.trackContinuationProgress(sessionID, 2)

  // then
  expect(firstUpdate.stagnationCount).toBe(0)
  expect(secondUpdate.stagnationCount).toBe(1)
  expect(thirdUpdate.stagnationCount).toBe(2)
})
```

### 片段 3：复杂前置条件和验证

```typescript
// src/plugin/chat-message.test.ts:59-71
test("first message: does not override TUI variant when user has no selection", async () => {
  //#given - first message, no user-selected variant
  const args = createMockHandlerArgs({ shouldOverride: true })
  const handler = createChatMessageHandler(args)
  const input = createMockInput("hephaestus", { providerID: "openai", modelID: "gpt-5.3-codex" })
  const output = createMockOutput() // no variant set

  //#when
  await handler(input, output)

  //#then - TUI sent undefined, should stay undefined (no config override)
  expect(output.message["variant"]).toBeUndefined()
})
```

---

## 依赖关系

Given-When-Then 模式与以下组件存在依赖关系：

**向上依赖**：
- Bun 测试框架：提供 `describe`, `test`, `expect` 等测试 API
- 测试设置模块：提供 `beforeEach` 和 `afterEach` 钩子支持

**向下依赖**：
- 被测代码：测试的目标对象
- Mock 隔离策略：确保测试环境的独立性

**同级依赖**：
- 测试设置：提供全局状态重置
- Mock 隔离：确保测试间不相互污染

---

## 实战示例

### 示例 1: Agent 测试

以下是一个测试 Agent 工具创建任务的完整示例：

```typescript
// src/tools/task/task-create.test.ts
describe("task_create tool", () => {
  let tool: ReturnType<typeof createTaskCreateTool>

  beforeEach(() => {
    // 设置测试目录
    if (existsSync(TEST_STORAGE)) {
      rmSync(TEST_STORAGE, { recursive: true, force: true })
    }
    mkdirSync(TEST_DIR, { recursive: true })
    tool = createTaskCreateTool(TEST_CONFIG)
  })

  describe("create action", () => {
    test("creates task with required subject field", async () => {
      //#given
      const args = {
        subject: "Implement authentication",
      }

      //#when
      const resultStr = await tool.execute(args, TEST_CONTEXT)
      const result = JSON.parse(resultStr)

      //#then
      expect(result).toHaveProperty("task")
      expect(result.task).toHaveProperty("id")
      expect(result.task.subject).toBe("Implement authentication")
    })

    test("auto-generates T-{uuid} format ID", async () => {
      //#given
      const args = {
        subject: "Test task",
      }

      //#when
      const resultStr = await tool.execute(args, TEST_CONTEXT)
      const result = JSON.parse(resultStr)

      //#then
      expect(result.task.id).toMatch(/^T-[a-f0-9-]+$/)
    })
  })
})
```

### 示例 2: Hook 测试

以下是一个测试 Hook 状态管理的完整示例：

```typescript
// src/hooks/todo-continuation-enforcer/session-state.test.ts
describe("createSessionStateStore", () => {
  let sessionStateStore: SessionStateStore

  beforeEach(() => {
    sessionStateStore = createSessionStateStore()
  })

  afterEach(() => {
    sessionStateStore.shutdown()
  })

  test("given repeated incomplete counts after a continuation, tracks stagnation", () => {
    // given
    const sessionID = "ses-stagnation"
    const state = sessionStateStore.getState(sessionID)

    // when
    const firstUpdate = sessionStateStore.trackContinuationProgress(sessionID, 2)
    state.awaitingPostInjectionProgressCheck = true
    const secondUpdate = sessionStateStore.trackContinuationProgress(sessionID, 2)
    state.awaitingPostInjectionProgressCheck = true
    const thirdUpdate = sessionStateStore.trackContinuationProgress(sessionID, 2)

    // then
    expect(firstUpdate.stagnationCount).toBe(0)
    expect(secondUpdate.stagnationCount).toBe(1)
    expect(thirdUpdate.stagnationCount).toBe(2)
  })

  test("given incomplete count decreases, resets stagnation tracking", () => {
    // given
    const sessionID = "ses-progress-reset"
    const state = sessionStateStore.getState(sessionID)
    state.lastInjectedAt = Date.now()
    sessionStateStore.trackContinuationProgress(sessionID, 3)
    sessionStateStore.trackContinuationProgress(sessionID, 3)

    // when
    const progressUpdate = sessionStateStore.trackContinuationProgress(sessionID, 2)

    // then
    expect(progressUpdate.hasProgressed).toBe(true)
    expect(progressUpdate.stagnationCount).toBe(0)
  })
})
```

---

## 交叉引用

- 参见：[测试设置](./01-测试设置.md) - 了解全局测试配置和预加载脚本
- 参见：[Mock 隔离](./03-Mock 隔离.md) - 了解 Mock 隔离策略和 CI 测试分割
- 参见：[Hook 分层](../03-Hook 系统/01-Hook 分层.md) - 了解 Hook 系统的测试方法
- 参见：[Bun 测试文档](https://bun.sh/docs/cli/test) - 官方测试框架文档
- 参见：[BDD 风格指南](https://cucumber.io/docs/bdd/) - 行为驱动开发最佳实践
