# Tmux 工具：终端多路复用抽象层

> 所属模块：08-shared-utilities | 三层模型：Layer 3 | 优先级：P3

---

## 架构位置

```
┌─────────────────────────────────────────────────────────────┐
│                    Layer 4: Features                        │
│         ┌─────────────────────────────────────┐             │
│         │     Tmux Subagent Manager           │             │
│         │  (session lifecycle, grid planning) │             │
│         └─────────────────────────────────────┘             │
├─────────────────────────────────────────────────────────────┤
│                    Layer 3: Shared Utilities                │
│         ┌─────────────────────────────────────┐             │
│         │      Tmux Utils (本模块)            │             │
│         │  ┌─────────┐ ┌─────────┐ ┌────────┐ │             │
│         │  │ Session │ │ Command │ │ Error  │ │             │
│         │  │  Mgmt   │ │  Exec   │ │ Handle │ │             │
│         │  └─────────┘ └─────────┘ └────────┘ │             │
│         └─────────────────────────────────────┘             │
├─────────────────────────────────────────────────────────────┤
│                    Layer 2: Tools                           │
│         ┌─────────────────────────────────────┐             │
│         │   Interactive Bash / Tmux Path      │             │
│         └─────────────────────────────────────┘             │
├─────────────────────────────────────────────────────────────┤
│                    Layer 1: External                        │
│                      tmux binary                            │
└─────────────────────────────────────────────────────────────┘
```

Tmux 工具模块位于 Layer 3 (Shared Utilities)，为上层 Feature 模块提供统一的 tmux 操作抽象。它将底层的 tmux 命令调用封装为类型安全的 TypeScript API，处理环境检测、错误处理和状态管理。

---

## 核心职责

Tmux 工具模块承担以下核心职责：

**1. 环境抽象与检测**
模块提供 `isInsideTmux()` 函数检测当前是否运行在 tmux 会话中，通过检查 `TMUX` 环境变量实现。这是所有 tmux 操作的前置条件，确保命令只在有效环境中执行。

**2. 会话生命周期管理**
封装 pane 的创建、关闭和替换操作。`spawnTmuxPane` 负责在指定位置分割窗口并启动新的 OpenCode 会话；`closeTmuxPane` 实现优雅关闭（先发送 Ctrl+C，再 kill pane）；`replaceTmuxPane` 用于复用现有 pane 启动新会话。

**3. 布局与尺寸控制**
提供 `applyLayout` 和 `enforceMainPaneWidth` 函数管理窗口布局。支持 `main-vertical`、`main-horizontal` 等布局模式，并确保主 pane 保持最小宽度，为 Agent pane 分配合理空间。

**4. 服务器健康检查**
`isServerRunning` 函数通过访问 `/global/health` 端点验证 OpenCode 服务器状态，支持进程内标记和缓存机制，避免重复的网络请求。

**5. 错误处理与日志**
所有操作都包含详细的日志记录，通过 `log()` 函数写入 `/tmp/oh-my-opencode.log`。错误处理遵循"快速失败"原则，在操作前验证所有前置条件。

---

## 代码索引 (Code Index)

| 功能点 | 文件路径 | 行号 | 函数/类 | 说明 |
|--------|----------|------|---------|------|
| 环境检测 | `src/shared/tmux/tmux-utils/environment.ts` | 7-9 | `isInsideTmux()` | 检测是否在 tmux 会话中 |
| Pane ID 获取 | `src/shared/tmux/tmux-utils/environment.ts` | 11-13 | `getCurrentPaneId()` | 获取当前 pane ID |
| Pane 创建 | `src/shared/tmux/tmux-utils/pane-spawn.ts` | 10-94 | `spawnTmuxPane()` | 分割窗口创建新 pane |
| Pane 关闭 | `src/shared/tmux/tmux-utils/pane-close.ts` | 9-48 | `closeTmuxPane()` | 优雅关闭指定 pane |
| Pane 替换 | `src/shared/tmux/tmux-utils/pane-replace.ts` | 8-73 | `replaceTmuxPane()` | 复用 pane 启动新会话 |
| 尺寸查询 | `src/shared/tmux/tmux-utils/pane-dimensions.ts` | 9-28 | `getPaneDimensions()` | 获取 pane 和窗口尺寸 |
| 布局应用 | `src/shared/tmux/tmux-utils/layout.ts` | 43-65 | `applyLayout()` | 应用 tmux 布局 |
| 主 pane 宽度 | `src/shared/tmux/tmux-utils/layout.ts` | 67-96 | `enforceMainPaneWidth()` | 强制主 pane 宽度 |
| 服务器健康 | `src/shared/tmux/tmux-utils/server-health.ts` | 18-56 | `isServerRunning()` | 检查服务器运行状态 |
| 常量定义 | `src/shared/tmux/constants.ts` | 1-12 | `POLL_INTERVAL_BACKGROUND_MS` | 后台轮询间隔等常量 |

---

## Tmux 工具函数

### 1. 会话管理

会话管理是 Tmux 工具的核心功能，负责 pane 的生命周期控制。

**Pane 创建流程** (`spawnTmuxPane`，第 10-94 行)：

```typescript
// src/shared/tmux/tmux-utils/pane-spawn.ts
export async function spawnTmuxPane(
  sessionId: string,
  description: string,
  config: TmuxConfig,
  serverUrl: string,
  targetPaneId?: string,
  splitDirection: SplitDirection = "-h",
): Promise<SpawnPaneResult>
```

创建流程包含以下步骤：
1. **前置检查**：验证 `config.enabled`、是否在 tmux 环境、服务器是否运行
2. **命令构建**：使用 `split-window` 命令，指定分割方向 (`-h` 水平/`-v` 垂直)
3. **会话启动**：在新 pane 中执行 `opencode attach <url> --session <id>`
4. **标题设置**：使用 `select-pane -T` 设置 pane 标题便于识别

**Pane 关闭流程** (`closeTmuxPane`，第 9-48 行)：

关闭操作采用"优雅关闭"策略：
1. 首先发送 `Ctrl+C` 信号给目标 pane，允许进程自行清理
2. 等待 250ms 让进程响应
3. 执行 `kill-pane` 强制终止

这种设计避免了数据丢失，同时确保 pane 最终被关闭。

### 2. 命令执行

Tmux 工具使用 Bun 的 `spawn` API 执行 tmux 命令，所有命令都通过 `getTmuxPath()` 解析的 tmux 二进制文件执行。

**命令执行模式** (第 67-71 行)：

```typescript
// src/shared/tmux/tmux-utils/pane-spawn.ts
const proc = spawn([tmux, ...args], { stdout: "pipe", stderr: "pipe" })
const exitCode = await proc.exited
const stdout = await new Response(proc.stdout).text()
```

关键特点：
- 使用 `"pipe"` 模式捕获输出，便于错误诊断
- 通过 `exitCode` 判断命令是否成功
- 异步执行，不阻塞主线程

**布局命令执行** (`applyLayout`，第 43-65 行)：

```typescript
export async function applyLayout(
  tmux: string,
  layout: TmuxLayout,
  mainPaneSize: number,
): Promise<void>
```

支持 `even-horizontal`、`even-vertical`、`main-horizontal`、`main-vertical`、`tiled` 等标准 tmux 布局。对于 `main-*` 布局，还会设置 `main-pane-height` 或 `main-pane-width` 选项。

### 3. 错误处理

Tmux 工具采用多层错误处理策略：

**前置条件验证** (第 29-42 行)：

```typescript
// src/shared/tmux/tmux-utils/pane-spawn.ts
if (!config.enabled) {
  log("[spawnTmuxPane] SKIP: config.enabled is false")
  return { success: false }
}
if (!isInsideTmux()) {
  log("[spawnTmuxPane] SKIP: not inside tmux")
  return { success: false }
}
```

每个操作在执行前检查：
- 配置是否启用
- 是否在 tmux 环境中
- tmux 二进制是否可用
- 服务器是否运行

**进程级错误捕获** (第 72-74 行)：

```typescript
if (exitCode !== 0 || !paneId) {
  return { success: false }
}
```

命令执行后检查退出码，非零退出码视为失败。

**日志记录** (第 85-91 行)：

所有操作都记录到 `/tmp/oh-my-opencode.log`，包含：
- 操作类型和参数
- 成功/失败状态
- 错误信息和 stderr 输出

---

## Tmux 工具流程

Tmux 工具的使用遵循"查询-决策-执行"模式：

```
┌─────────────────────────────────────────────────────────────┐
│                      使用流程                                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. 环境检测                                                │
│     └─> isInsideTmux()                                      │
│         └─> 检查 TMUX 环境变量                              │
│                                                             │
│  2. 状态查询                                                │
│     └─> queryWindowState()                                  │
│         └─> 获取 pane 布局、尺寸信息                        │
│                                                             │
│  3. 决策计算                                                │
│     └─> decideSpawnActions()                                │
│         └─> 基于容量配置决定 spawn/close/replace            │
│                                                             │
│  4. 动作执行                                                │
│     └─> executeActions()                                    │
│         ├─> spawnTmuxPane()  创建新 pane                    │
│         ├─> closeTmuxPane()  关闭旧 pane                    │
│         └─> replaceTmuxPane() 替换 pane 内容                │
│                                                             │
│  5. 布局调整                                                │
│     └─> enforceMainPaneWidth()                              │
│         └─> 确保主 pane 保持配置宽度                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 流程/架构图

```
                    Tmux 工具架构
    ┌──────────────────────────────────────────────┐
    │                                              │
    │   ┌────────────────────────────────────┐    │
    │   │     TmuxSessionManager             │    │
    │   │  (features/tmux-subagent)          │    │
    │   └────────────────────────────────────┘    │
    │                    │                         │
    │                    ▼                         │
    │   ┌────────────────────────────────────┐    │
    │   │      ActionExecutor                │    │
    │   │  (decision-engine, action-executor)│    │
    │   └────────────────────────────────────┘    │
    │                    │                         │
    │         ┌──────────┴──────────┐             │
    │         ▼                     ▼             │
    │   ┌──────────┐          ┌──────────┐       │
    │   │  spawn   │          │  close   │       │
    │   │  pane    │          │  pane    │       │
    │   └──────────┘          └──────────┘       │
    │         │                     │             │
    │         └──────────┬──────────┘             │
    │                    ▼                         │
    │   ┌────────────────────────────────────┐    │
    │   │      Tmux Utils (Layer 3)          │    │
    │   │  ┌────────┐ ┌────────┐ ┌────────┐  │    │
    │   │  │ pane-  │ │ pane-  │ │ pane-  │  │    │
    │   │  │ spawn  │ │ close  │ │replace │  │    │
    │   │  └────────┘ └────────┘ └────────┘  │    │
    │   │  ┌────────┐ ┌────────┐ ┌────────┐  │    │
    │   │  │ layout │ │ server │ │ pane-  │  │    │
    │   │  │        │ │ health │ │dimensions│  │    │
    │   │  └────────┘ └────────┘ └────────┘  │    │
    │   └────────────────────────────────────┘    │
    │                    │                         │
    │                    ▼                         │
    │   ┌────────────────────────────────────┐    │
    │   │      tmux binary (system)          │    │
    │   └────────────────────────────────────┘    │
    │                                              │
    └──────────────────────────────────────────────┘
```

---

## 关键代码片段

### 片段 1: Pane 创建与 OpenCode 会话启动

```typescript
// src/shared/tmux/tmux-utils/pane-spawn.ts:52-65
const shell = process.env.SHELL || "/bin/sh"
const escapedUrl = shellEscapeForDoubleQuotedCommand(serverUrl)
const opencodeCmd = `${shell} -c "opencode attach ${escapedUrl} --session ${sessionId}"`

const args = [
  "split-window",
  splitDirection,  // "-h" 水平分割 或 "-v" 垂直分割
  "-d",            // 后台运行
  "-P",            // 打印 pane ID
  "-F",
  "#{pane_id}",    // 输出格式
  ...(targetPaneId ? ["-t", targetPaneId] : []),  // 目标 pane
  opencodeCmd,     // 要执行的命令
]

const proc = spawn([tmux, ...args], { stdout: "pipe", stderr: "pipe" })
```

### 片段 2: 优雅关闭 Pane

```typescript
// src/shared/tmux/tmux-utils/pane-close.ts:23-37
log("[closeTmuxPane] sending Ctrl+C for graceful shutdown", { paneId })
const ctrlCProc = spawn([tmux, "send-keys", "-t", paneId, "C-c"], {
  stdout: "pipe",
  stderr: "pipe",
})
await ctrlCProc.exited

await delay(250)  // 等待进程响应

log("[closeTmuxPane] killing pane", { paneId })
const proc = spawn([tmux, "kill-pane", "-t", paneId], {
  stdout: "pipe",
  stderr: "pipe",
})
```

### 片段 3: 服务器健康检查与缓存

```typescript
// src/shared/tmux/tmux-utils/server-health.ts:18-45
export async function isServerRunning(serverUrl: string): Promise<boolean> {
  // 进程内标记检查（同一进程已确认运行）
  if (isMarkedRunningInProcess()) {
    return true
  }

  // URL 级缓存（避免重复请求同一服务器）
  if (serverCheckUrl === serverUrl && serverAvailable === true) {
    return true
  }

  const healthUrl = new URL("/global/health", serverUrl).toString()
  const timeoutMs = 3000
  const maxAttempts = 2

  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    const controller = new AbortController()
    const timeout = setTimeout(() => controller.abort(), timeoutMs)

    try {
      const response = await fetch(healthUrl, {
        signal: controller.signal,
      }).catch(() => null)

      if (response?.ok) {
        serverCheckUrl = serverUrl
        serverAvailable = true
        return true
      }
    } finally {
      clearTimeout(timeout)
    }

    if (attempt < maxAttempts) {
      await delay(250)
    }
  }

  return false
}
```

---

## 依赖关系

Tmux 工具模块的依赖关系：

**向上依赖 (被上层使用)**：
- `features/tmux-subagent/manager.ts` - TmuxSessionManager 使用所有工具函数
- `features/tmux-subagent/action-executor.ts` - 执行 pane 操作
- `features/tmux-subagent/decision-engine.ts` - 决策时查询 pane 状态

**向下依赖 (使用下层)**：
- `tools/interactive-bash/tmux-path-resolver.ts` - 解析 tmux 二进制路径
- `shared/logger.ts` - 日志记录
- `shared/shell-env.ts` - Shell 命令转义
- `config/schema.ts` - TmuxConfig 类型定义

**外部依赖**：
- `tmux` 二进制 (系统安装)
- `opencode` CLI (用于 attach 会话)

---

## 实战示例

### 示例 1: 会话创建与清理

场景：在后台创建一个新的 Agent 会话 pane，并在任务完成后清理。

```typescript
import { spawnTmuxPane, closeTmuxPane, isInsideTmux } from "./shared/tmux"
import type { TmuxConfig } from "./config/schema"

async function createAgentSession(
  sessionId: string,
  description: string,
  serverUrl: string
): Promise<string | null> {
  // 检查是否在 tmux 环境中
  if (!isInsideTmux()) {
    console.log("Not in tmux environment, skipping pane creation")
    return null
  }

  const config: TmuxConfig = {
    enabled: true,
    layout: "main-vertical",
    main_pane_size: 60,
    main_pane_min_width: 80,
    agent_pane_min_width: 52,
  }

  // 创建 pane
  const result = await spawnTmuxPane(
    sessionId,
    description,
    config,
    serverUrl,
    undefined,  // 使用当前 pane 作为目标
    "-h"        // 水平分割
  )

  if (!result.success) {
    console.error("Failed to spawn pane")
    return null
  }

  console.log(`Pane created: ${result.paneId}`)
  return result.paneId
}

async function cleanupAgentSession(paneId: string): Promise<boolean> {
  const success = await closeTmuxPane(paneId)
  if (success) {
    console.log(`Pane ${paneId} closed successfully`)
  } else {
    console.error(`Failed to close pane ${paneId}`)
  }
  return success
}

// 使用示例
const paneId = await createAgentSession("agent-123", "Code Review", "http://localhost:4096")
if (paneId) {
  // ... 执行任务 ...
  await cleanupAgentSession(paneId)
}
```

### 示例 2: 命令执行与输出捕获

场景：查询当前 pane 的尺寸信息，用于决策是否分割。

```typescript
import { getPaneDimensions, isInsideTmux, getCurrentPaneId } from "./shared/tmux"
import { MIN_PANE_WIDTH } from "./features/tmux-subagent/types"

async function checkPaneCapacity(): Promise<{
  canSplit: boolean
  currentWidth: number
  windowWidth: number
}> {
  if (!isInsideTmux()) {
    return { canSplit: false, currentWidth: 0, windowWidth: 0 }
  }

  const currentPaneId = getCurrentPaneId()
  if (!currentPaneId) {
    return { canSplit: false, currentWidth: 0, windowWidth: 0 }
  }

  const dimensions = await getPaneDimensions(currentPaneId)
  if (!dimensions) {
    return { canSplit: false, currentWidth: 0, windowWidth: 0 }
  }

  const { paneWidth, windowWidth } = dimensions

  // 计算分割后每个 pane 的宽度
  const dividerWidth = 1
  const newPaneWidth = Math.floor((windowWidth - dividerWidth) / 2)

  // 检查是否满足最小宽度要求
  const canSplit = newPaneWidth >= MIN_PANE_WIDTH

  console.log(`Current pane: ${paneWidth}, Window: ${windowWidth}`)
  console.log(`After split: ${newPaneWidth}, Can split: ${canSplit}`)

  return {
    canSplit,
    currentWidth: paneWidth,
    windowWidth,
  }
}

// 使用示例
const capacity = await checkPaneCapacity()
if (capacity.canSplit) {
  console.log("Window has enough space for new agent pane")
} else {
  console.log("Window too narrow, consider closing existing panes first")
}
```

---

## 交叉引用

- 参见：[Tmux 集成](../05-功能模块/02-Tmux 集成.md) - Tmux Subagent 功能模块的完整说明
- 参见：[后台 Agent](../05-功能模块/01-后台 Agent.md) - 后台任务管理与 Tmux 集成的关系
- 参见：[配置系统](../06-config-system/tmux-config.md) - Tmux 配置选项详解
- 参见：[交互式 Bash](../04-tools/interactive-bash.md) - Tmux 路径解析工具说明
- 参见：[Shared 模块概览](./00-shared-overview.md) - Shared Utilities 模块整体架构

---

*文档生成时间：2026-03-28 | 基于 oh-my-opencode 源码*
