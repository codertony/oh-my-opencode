# Tmux 集成：Subagent 终端多路复用

> 所属模块：05-features-modules | 三层模型：Layer 3 | 优先级：P2

---

## 架构位置

```
┌─────────────────────────────────────────────────────────────┐
│                    Layer 3: Features                         │
│  ┌───────────────────────────────────────────────────────┐  │
│  │              Tmux Subagent Module                      │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌───────────────┐  │  │
│  │  │   Manager   │  │   Decision  │  │   Executor    │  │  │
│  │  │  (manager)  │  │   Engine    │  │  (action)     │  │  │
│  │  └─────────────┘  └─────────────┘  └───────────────┘  │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌───────────────┐  │  │
│  │  │Grid Planning│  │   Polling   │  │ Event Handler │  │  │
│  │  │(grid-plan)  │  │   Manager   │  │  (lifecycle)  │  │  │
│  │  └─────────────┘  └─────────────┘  └───────────────┘  │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────┐
│                    Layer 2: Tools                            │
│  ┌───────────────────────────────────────────────────────┐  │
│  │           Interactive Bash Tool                        │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌───────────────┐  │  │
│  │  │   Tokenize  │  │   Execute   │  │   Timeout     │  │  │
│  │  │   Command   │  │   Tmux Cmd  │  │   Handler     │  │  │
│  │  └─────────────┘  └─────────────┘  └───────────────┘  │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

## 核心职责

Tmux 集成模块是 oh-my-opencode 的核心特性之一，它实现了 Subagent 在终端多路复用器中的无缝运行。该模块采用"状态优先"架构，将 tmux 窗格作为 Subagent 会话的物理载体，实现了以下核心能力：

首先，它提供了智能的窗格生命周期管理。当后台 Agent 被创建时，系统会自动在 tmux 中分割窗格并启动对应的 Subagent 会话；当会话结束时，窗格会被自动清理。这种自动化管理让用户无需手动操作 tmux 即可并行运行多个 Agent。

其次，模块实现了网格布局规划算法。根据窗口大小和配置参数，系统会计算最优的窗格布局，确保主编辑区（main pane）始终保留足够的空间，同时为 Agent 窗格分配合适的区域。布局支持垂直和水平两种模式，并能动态调整。

最后，Interactive Bash 工具提供了与 TUI 应用（如 vim、htop、pudb 等）的完整交互能力。不同于普通的 Bash 工具，它通过 tmux 命令直接操作窗格，支持持续交互而非一次性命令执行。

## 代码索引 (Code Index)

| 功能点 | 文件路径 | 行号 | 函数/类 | 说明 |
|--------|----------|------|---------|------|
| 会话管理器 | `src/features/tmux-subagent/manager.ts` | 56-718 | `TmuxSessionManager` | 核心管理类，处理窗格生命周期 |
| 类型定义 | `src/features/tmux-subagent/types.ts` | 1-52 | `TrackedSession`, `WindowState` | 核心数据结构定义 |
| 决策引擎 | `src/features/tmux-subagent/decision-engine.ts` | 1-22 | `decideSpawnActions` | 窗格创建决策逻辑入口 |
| 动作执行器 | `src/features/tmux-subagent/action-executor.ts` | 66-137 | `executeAction`, `executeActions` | 执行窗格操作 |
| 网格规划 | `src/features/tmux-subagent/grid-planning.ts` | 46-116 | `calculateCapacity`, `computeGridPlan` | 计算窗格容量和布局 |
| 生成决策 | `src/features/tmux-subagent/spawn-action-decider.ts` | 25-133 | `decideSpawnActions` | 决定是否创建新窗格 |
| 轮询管理 | `src/features/tmux-subagent/polling-manager.ts` | 12-145 | `TmuxPollingManager` | 监控会话健康状态 |
| 交互式 Bash | `src/tools/interactive-bash/tools.ts` | 51-136 | `interactive_bash` | TUI 应用交互工具 |

## Tmux 集成机制

### 1. Tmux 会话管理

Tmux 会话管理采用"状态优先"架构，确保内部缓存与实际的 tmux 状态保持一致。核心流程遵循 QUERY → DECIDE → EXECUTE → UPDATE 四个阶段。

**查询阶段 (QUERY)**：系统首先通过 `queryWindowState` 获取当前窗口的实际状态，包括主窗格和 Agent 窗格的位置、大小等信息。这是唯一的真实数据源。

```typescript
// src/features/tmux-subagent/manager.ts:143-154
private async queryWindowStateSafely(): Promise<WindowState | null> {
  if (!this.sourcePaneId) return null
  try {
    return await queryWindowState(this.sourcePaneId)
  } catch (error) {
    log("[tmux-session-manager] failed to query window state for close", {
      error: String(error),
    })
    return null
  }
}
```

**决策阶段 (DECIDE)**：`decideSpawnActions` 函数根据窗口状态和容量配置，决定是否需要创建、关闭或替换窗格。决策是纯粹的函数，没有副作用。

**执行阶段 (EXECUTE)**：`executeActions` 函数执行决策产生的动作序列，包括 spawn（创建）、close（关闭）、replace（替换）三种操作。

**更新阶段 (UPDATE)**：只有在 tmux 确认操作成功后，内部缓存才会更新。这种设计确保了系统状态的一致性。

### 2. Subagent 执行

Subagent 在 tmux 窗格中的执行流程如下：

当 `session.created` 事件触发时，`TmuxSessionManager.onSessionCreated` 方法会被调用。该方法首先检查是否启用了 tmux 集成以及是否在 tmux 环境中运行。然后，它会查询当前窗口状态，调用决策引擎决定如何创建窗格，最后执行相应的动作。

```typescript
// src/features/tmux-subagent/manager.ts:438-461
async onSessionCreated(event: SessionCreatedEvent): Promise<void> {
  const enabled = this.isEnabled()
  log("[tmux-session-manager] onSessionCreated called", {
    enabled,
    tmuxConfigEnabled: this.tmuxConfig.enabled,
    isInsideTmux: this.deps.isInsideTmux(),
    eventType: event.type,
    infoId: event.properties?.info?.id,
    infoParentID: event.properties?.info?.parentID,
  })

  if (!enabled) return
  if (event.type !== "session.created") return

  const info = event.properties?.info
  if (!info?.id || !info?.parentID) return

  const sessionId = info.id
  const title = info.title ?? "Subagent"
```

窗格创建使用 `spawnTmuxPane` 函数，它会执行 tmux 的 `split-window` 命令，在新的窗格中启动 Subagent 进程。Subagent 通过 `--server-url` 参数连接到主会话，实现双向通信。

### 3. Interactive Bash

Interactive Bash 工具专门用于与 TUI 应用进行持续交互。与普通 Bash 工具不同，它直接操作 tmux 窗格，支持需要用户输入的应用。

```typescript
// src/tools/interactive-bash/tools.ts:51-136
export const interactive_bash: ToolDefinition = tool({
  description: INTERACTIVE_BASH_DESCRIPTION,
  args: {
    tmux_command: tool.schema.string().describe("The tmux command to execute (without 'tmux' prefix)"),
  },
  execute: async (args) => {
    try {
      const tmuxPath = getCachedTmuxPath() ?? "tmux"
      const parts = tokenizeCommand(args.tmux_command)
      // ... 命令执行逻辑
      const proc = spawnWithWindowsHide([tmuxPath, ...parts], {
        stdout: "pipe",
        stderr: "pipe",
      })
      // ... 超时处理和结果返回
    }
  }
})
```

该工具会阻止某些敏感命令（如 `capture-pane`、`save-buffer` 等），引导用户使用普通 Bash 工具执行这些操作。命令执行支持超时机制，默认超时时间为 60 秒。

## Tmux 工作流程

Tmux 集成的工作流程分为会话创建和会话销毁两个主要路径：

**会话创建流程**：
1. 后台 Agent 被创建，触发 `session.created` 事件
2. `TmuxSessionManager` 接收事件，查询当前窗口状态
3. 决策引擎评估是否可以创建新窗格（考虑窗口大小、现有窗格数量等）
4. 如果可以创建，执行 `split-window` 命令创建新窗格
5. 在新窗格中启动 Subagent 进程
6. 轮询管理器开始监控会话健康状态

**会话销毁流程**：
1. 后台 Agent 结束，触发 `session.deleted` 事件
2. 系统查询窗口状态，找到对应的窗格
3. 执行 `kill-pane` 命令关闭窗格
4. 如果关闭失败，标记为 pending 状态，稍后重试
5. 清理内部缓存

**延迟附加机制**：当窗口空间不足时，新的 Subagent 会被放入延迟队列。轮询器会定期检查窗口状态，一旦空间可用就自动创建窗格。

## 流程/架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Tmux 工作流程                                 │
└─────────────────────────────────────────────────────────────────────┘

  Session Created Event                    Session Deleted Event
           │                                        │
           ▼                                        ▼
  ┌─────────────────┐                     ┌─────────────────┐
  │  Check Enabled  │                     │  Remove Deferred│
  │  & In Tmux      │                     │  Session        │
  └────────┬────────┘                     └────────┬────────┘
           │                                        │
           ▼                                        ▼
  ┌─────────────────┐                     ┌─────────────────┐
  │ Query Window    │                     │ Query Window    │
  │ State           │                     │ State           │
  └────────┬────────┘                     └────────┬────────┘
           │                                        │
           ▼                                        ▼
  ┌─────────────────┐                     ┌─────────────────┐
  │ Decide Spawn    │                     │ Decide Close    │
  │ Actions         │                     │ Action          │
  └────────┬────────┘                     └────────┬────────┘
           │                                        │
     ┌─────┴─────┐                                  │
     │           │                                  │
     ▼           ▼                                  ▼
┌────────┐  ┌────────┐                      ┌─────────────────┐
│ Can    │  │ Cannot │                      │ Execute Close   │
│ Spawn  │  │ Spawn  │                      │ Action          │
└───┬────┘  └───┬────┘                      └────────┬────────┘
    │           │                                    │
    ▼           ▼                                    ▼
┌────────┐  ┌────────┐                      ┌─────────────────┐
│Execute │  │Enqueue │                      │ Remove Tracked  │
│Actions │  │Deferred│                      │ Session         │
└───┬────┘  └────────┘                      └─────────────────┘
    │
    ▼
┌────────┐
│ Track  │
│ Session│
└───┬────┘
    │
    ▼
┌────────┐
│ Start  │
│ Polling│
└────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                      延迟附加循环                                    │
└─────────────────────────────────────────────────────────────────────┘

  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
  │   Check     │────▶│   Window    │────▶│   Decide    │
  │   Queue     │     │   State     │     │   Spawn     │
  └─────────────┘     └─────────────┘     └──────┬──────┘
       ▲                                         │
       │                                         │
       └─────────────────────────────────────────┘
                    (If cannot spawn, retry later)
```

## 关键代码片段

### 片段 1：窗格动作类型定义

```typescript
// src/features/tmux-subagent/types.ts:36-39
export type PaneAction =
  | { type: "close"; paneId: string; sessionId: string }
  | { type: "spawn"; sessionId: string; description: string; targetPaneId: string; splitDirection: SplitDirection }
  | { type: "replace"; paneId: string; oldSessionId: string; newSessionId: string; description: string }
```

这个类型定义了三种窗格操作：关闭现有窗格、创建新窗格、替换窗格中的会话。每种操作都包含了执行所需的全部信息。

### 片段 2：网格容量计算

```typescript
// src/features/tmux-subagent/grid-planning.ts:46-76
export function calculateCapacity(
  windowWidth: number,
  windowHeight: number,
  options?: CapacityOptions,
  mainPaneWidth?: number,
): GridCapacity {
  const availableWidth =
    typeof mainPaneWidth === "number"
      ? Math.max(0, windowWidth - mainPaneWidth - DIVIDER_SIZE)
      : resolveAgentAreaWidth(windowWidth, options)
  const minPaneWidth = resolveMinPaneWidth(options)
  const cols = Math.min(
    MAX_GRID_SIZE,
    Math.max(
      0,
      Math.floor(
        (availableWidth + DIVIDER_SIZE) / (minPaneWidth + DIVIDER_SIZE),
      ),
    ),
  )
  const rows = Math.min(
    MAX_GRID_SIZE,
    Math.max(
      0,
      Math.floor(
        (windowHeight + DIVIDER_SIZE) / (MIN_PANE_HEIGHT + DIVIDER_SIZE),
      ),
    ),
  )
  return { cols, rows, total: cols * rows }
}
```

该函数计算窗口可以容纳的窗格数量，考虑了分隔线宽度、最小窗格尺寸等约束条件。

### 片段 3：轮询稳定性检测

```typescript
// src/features/tmux-subagent/polling-manager.ts:75-115
if (isIdle && elapsedMs >= MIN_STABILITY_TIME_MS) {
  try {
    const messagesResult = await this.client.session.messages({ 
      path: { id: sessionId } 
    })
    const currentMsgCount = Array.isArray(messagesResult.data) 
      ? messagesResult.data.length 
      : 0

    if (tracked.lastMessageCount === currentMsgCount) {
      tracked.stableIdlePolls = (tracked.stableIdlePolls ?? 0) + 1
      
      if (tracked.stableIdlePolls >= STABLE_POLLS_REQUIRED) {
        const recheckResult = await this.client.session.status({ path: undefined })
        const recheckStatuses = normalizeSDKResponse(recheckResult, {} as Record<string, { type: string }>)
        const recheckStatus = recheckStatuses[sessionId]
        
        if (recheckStatus?.type === "idle") {
          shouldCloseViaStability = true
        }
      }
    }
  }
}
```

稳定性检测确保会话在空闲状态持续一段时间且消息数量不再变化后才关闭，避免过早关闭正在处理任务的会话。

## 依赖关系

Tmux 集成模块依赖以下组件：

1. **Shared Tmux 模块** (`src/shared/tmux/`)：提供底层的 tmux 命令执行函数，如 `spawnTmuxPane`、`closeTmuxPane`、`queryWindowState` 等。

2. **Config Schema** (`src/config/schema`)：提供 `TmuxConfig` 类型定义，包含布局模式、窗格大小等配置项。

3. **Background Agent** (`src/features/background-agent/`)：Tmux 集成是后台 Agent 的可选执行环境，为其提供终端多路复用能力。

4. **Plugin Interface** (`src/plugin/`)：通过事件钩子接收会话创建和销毁事件。

5. **Interactive Bash Tool** (`src/tools/interactive-bash/`)：提供与 TUI 应用交互的能力。

## 实战示例

### 示例 1: TUI 应用交互

使用 Interactive Bash 工具在 tmux 窗格中运行 vim 并执行编辑操作：

```typescript
// 创建新的 tmux 会话并在其中启动 vim
interactive_bash({
  tmux_command: 'new-session -d -s omo-dev "vim src/index.ts"'
})

// 向 vim 发送命令（进入插入模式并输入文本）
interactive_bash({
  tmux_command: 'send-keys -t omo-dev "i" "const x = 42;" Escape ":wq" Enter'
})

// 捕获窗格输出查看结果
// 注意：capture-pane 被阻止，需要使用普通 Bash 工具
bash({
  command: 'tmux capture-pane -p -t omo-dev'
})
```

这个示例展示了如何在 tmux 窗格中启动 vim，向其发送按键命令，最后保存并退出。适用于需要与编辑器、调试器等 TUI 应用交互的场景。

### 示例 2: 长期运行任务

在后台窗格中运行测试套件，主会话可以继续其他工作：

```typescript
// 在后台窗格中启动测试（使用 & 让命令在后台运行）
bash({
  command: 'tmux split-window -d -h "npm run test:watch"'
})

// 主会话继续执行其他任务
// ...

// 稍后检查测试窗格的状态
interactive_bash({
  tmux_command: 'display-message -p "#{pane_title}"'
})

// 当需要查看测试结果时，切换到该窗格
interactive_bash({
  tmux_command: 'select-pane -t 1'
})
```

这个示例展示了如何利用 tmux 的多窗格特性，在后台运行长期任务（如测试监视模式），同时主会话保持可用状态。

## 交叉引用

- 参见：[后台 Agent](./01-后台 Agent.md) - 了解 Subagent 的生命周期管理和任务调度机制
- 参见：[会话持久化](../01-Harness 架构/06-会话持久化.md) - 了解会话状态如何在主会话和 Subagent 之间同步
- 参见：[Boulder 机制](./04-Boulder 状态.md) - 了解多步操作的状态持久化
- 参见：[工具注册](../02-tools-syste./01-工具注册.md) - 了解 Interactive Bash 工具的注册和调用方式
- 参见：[配置系统](../04-config-system/tmux-config.md) - 了解 Tmux 集成的配置选项
