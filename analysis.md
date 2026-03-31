# Claude Code 代码库架构分析报告

> 分析日期：2026-03-31  
> 源码目录：restored-src/src

---

## 1. 代码架构图

```
┌─────────────────────────────────────────────────────────────┐
│                        入口层                                │
│  cli.tsx (快速路径) → main.tsx (全量初始化) → setup.ts    │
└──────────────────────────┬──────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────┐
│                       核心引擎                               │
│  QueryEngine (执行循环) → query.ts (消息处理) → context.ts │
└──────────────────────────┬──────────────────────────────────┘
                           ↓
        ┌─────────────────┼─────────────────┐
        ↓                 ↓                 ↓
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│   状态层      │  │   工具层      │  │   命令层      │
│ store.ts     │  │ Tool.ts      │  │ commands.ts  │
│ AppStateStore│  │ 30+工具实现   │  │ 80+命令实现   │
└──────────────┘  └──────────────┘  └──────────────┘
        ↓
┌─────────────────────────────────────────────────────────────┐
│                       UI 渲染层                             │
│  Ink (React终端) → App.tsx → REPL.tsx → 80+组件            │
└─────────────────────────────────────────────────────────────┘
```

### Mermaid 版本

```mermaid
flowchart TB
    subgraph "入口层 Entrypoints"
        CLI["entrypoints/cli.tsx<br/>CLI主入口 - 快速路径分发"]
        MAIN["main.tsx<br/>主程序 - 完整初始化"]
        SETUP["setup.ts<br/>会话初始化"]
    end

    subgraph "核心引擎 Core Engine"
        QE["QueryEngine.ts<br/>查询引擎 - 执行循环"]
        QUERY["query.ts<br/>消息处理核心"]
        QCONTEXT["context.ts<br/>上下文构建器"]
    end

    subgraph "状态管理 State"
        STATE["bootstrap/state.ts<br/>全局单例状态"]
        APPSTORE["state/AppStateStore.ts<br/>React响应式状态"]
        STORE["state/store.ts<br/>极简发布订阅Store"]
    end

    subgraph "命令系统 Commands"
        CMDREG["commands.ts<br/>命令注册表"]
        CMDS["commands/*/<br/>80+命令实现"]
    end

    subgraph "工具系统 Tools"
        TOOLS["tools.ts<br/>工具注册与构建"]
        BASETOOL["Tool.ts<br/>工具基类定义"]
        TOOLIMPL["tools/*Tool/<br/>30+工具实现"]
    end

    subgraph "UI渲染层 UI Rendering (Ink/React)"
        INK["ink.ts<br/>Ink渲染器封装"]
        APP["components/App.tsx<br/>根组件"]
        REPL["screens/REPL.tsx<br/>主交互屏幕"]
        COMPONENTS["components/<br/>80+UI组件"]
    end

    subgraph "上下文 Contexts"
        MAILBOX["context/mailbox.tsx<br/>进程间消息"]
        NOTIF["context/notifications.tsx<br/>通知队列"]
        STATS["context/stats.tsx<br/>统计信息"]
        VOICE["context/voice.tsx<br/>语音上下文"]
    end

    subgraph "协调模式 Coordinator Mode"
        COORD["coordinator/coordinatorMode.ts<br/>多Worker编排"]
        TEAMMATE["tasks/InProcessTeammateTask<br/>进程内Agent"]
        SWARM["utils/swarm/<br/>多Agent通信"]
    end

    subgraph "服务层 Services"
        API["services/api/<br/>Claude API调用"]
        MCP["services/mcp/<br/>MCP服务器管理"]
        ANALYTICS["services/analytics/<br/>遥测与增长"]
        COMPACT["services/compact/<br/>上下文压缩"]
        PLUGINS["services/plugins/<br/>插件系统"]
    end

    subgraph "权限系统 Permissions"
        PERMS["utils/permissions/<br/>权限判断"]
        HOOKS["hooks/toolPermission/<br/>权限Hook处理"]
    end

    CLI --> MAIN
    MAIN --> SETUP
    SETUP --> QE
    QE --> QUERY
    QUERY --> QCONTEXT
    QUERY --> TOOLS
    QE --> APPSTORE
    
    APPSTORE <--> STATE
    APPSTORE --> INK
    INK --> APP
    APP --> REPL
    REPL --> COMPONENTS
    
    QE --> TOOLIMPL
    QE --> CMDS
    QE --> COORD
    COORD --> TEAMMATE
    COORD --> SWARM
    
    QE --> API
    QE --> MCP
    QE --> PERMS
    PERMS --> HOOKS
```

---

## 2. 核心设计理念

### 2.1 分层架构

Claude Code 采用**分层+模块化**架构，从上到下：

- **入口层**：`cli.tsx` 做快速路径分流，避免不必要的模块加载
- **核心引擎**：`QueryEngine` 驱动主循环，`query.ts` 处理消息
- **状态层**：双轨状态（全局单例 `state.ts` + React响应式 `AppStateStore`）
- **UI层**：Ink（React的终端实现）+ 自研组件库
- **工具/命令层**：插件化设计，工具和命令独立注册

### 2.2 快速路径优化

`cli.tsx` 的核心思想——**零导入快速路径**：

```
--version        → 直接输出版本号，无任何import
--daemon         → 动态import daemon模块
--bg/--background → 动态import bg.js
```

这使得基本命令（如 `--version`）启动极快。

### 2.3 Feature Flag 驱动的 DCE

使用 `feature('FLAG_NAME')` 从 `bun:bundle` 内联检查，实现**编译时消除**不需要的代码路径：

- `COORDINATOR_MODE` - 多Agent协调模式
- `KAIROS` - 助手模式
- `BRIDGE_MODE` - 远程控制模式
- `VOICE_MODE` - 语音模式
- `TEMPLATES` - 模板任务

### 2.4 权限系统架构

权限判断通过 **Hook 模式**注入到 `QueryEngine`，而非硬编码：

- `useCanUseTool.tsx` 是核心权限判断Hook
- 支持多种模式：Interactive、Coordinator、Swarm Worker、Bypass
- 每个工具执行前通过 `canUseTool` 函数判断

### 2.5 极简响应式 Store

```typescript
// state/store.ts - 约30行的发布订阅Store
export function createStore<T>(initialState: T, onChange?) {
  let state = initialState
  const listeners = new Set<Listener>()
  return {
    getState: () => state,
    setState: (updater) => { /* 更新 + 通知 */ },
    subscribe: (listener) => { /* 返回取消订阅 */ }
  }
}
```

整个应用围绕这个模式构建，所有状态变更都通过 `setState`。

### 2.6 Coordinator 模式（多Agent编排）

当 `COORDINATOR_MODE` 开启时：

- 主Agent扮演**协调者**角色
- 通过 `AgentTool` 派发 `worker` 子Agent
- 子Agent的结果通过 `<task-notification>` XML标签异步返回
- 协调者负责任务分解、结果聚合、用户通信

---

## 3. 从启动到执行命令的完整链路

```
1. 用户执行 `claude` 命令
   └─> entrypoints/cli.tsx:main()
       ├─> --version → 直接打印版本（零导入）
       └─> 其他命令 → 动态import main.tsx

2. main.tsx 初始化（模块导入阶段 ~135ms）
   ├─> profileCheckpoint('main_tsx_entry')
   ├─> startMdmRawRead() - MDM配置并行读取
   ├─> startKeychainPrefetch() - Keychain并行预读
   └─> 导入大量模块（免疫分析、配置、API等）

3. init() 函数（enableConfigs + 环境准备）
   ├─> applySafeConfigEnvironmentVariables()
   ├─> applyExtraCACertsFromConfig()
   ├─> waitForPolicyLimitsToLoad()
   └─> waitForRemoteManagedSettingsToLoad()

4. setup() 函数（会话初始化）
   ├─> findCanonicalGitRoot() - 定位项目根目录
   ├─> captureHooksConfigSnapshot() - 快照钩子配置
   ├─> initializeFileChangedWatcher() - 初始化文件监控
   ├─> startUdsMessaging() - 进程间UDS通信
   └─> captureTeammateModeSnapshot() - Swarm快照

5. getCommands() - 挂载所有命令到Commander

6. 渲染Ink TUI
   └─> launchRepl(root, appProps, replProps)
       └─> App > REPL 组件树

7. 用户输入 → processUserInput()
   ├─> 解析斜杠命令 (/help, /commit等)
   ├─> 解析 @agent 提及
   ├─> 处理附件（图片、文件）
   └─> 返回消息数组 + shouldQuery标志

8. QueryEngine.handleNextMessage()
   ├─> 追加消息到历史
   ├─> 检查是否需要compact
   ├─> 构建system prompt
   ├─> 调用Claude API (withRetry)
   ├─> 解析tool_use块
   └─> 循环执行直到完成

9. 工具执行循环
   for each tool_use:
       ├─> canUseTool() → 权限检查
       ├─> tool.description() → 获取描述
       ├─> tool.validate() → 参数校验
       ├─> tool.execute() → 执行
       └─> 追加tool_result消息

10. Compact（上下文压缩）
    ├─> calculateTokenWarningState()
    ├─> buildPostCompactMessages()
    └─> 替换对话历史为摘要
```

---

## 4. 核心设计模式

| 模式 | 应用位置 | 说明 |
|------|----------|------|
| **Feature Flag + DCE** | `feature('FLAG')` | 编译时消除废弃代码 |
| **Fast Path Dispatch** | `cli.tsx` | 快速路径优先，避免不必要导入 |
| **Dependency Injection** | `useCanUseTool`, `QueryEngine` | 权限和处理函数注入 |
| **Pub/Sub Store** | `state/store.ts` | 极简响应式状态管理 |
| **Context + Reducer** | `AppStateStore.ts` | React的setState驱动UI更新 |
| **Command Registry** | `commands.ts` | 动态命令加载和注册 |
| **Tool Registry** | `tools.ts` | 工具的集中注册和构建 |
| **Worker Pool** | `coordinatorMode.ts` | 多Agent任务派发和协调 |
| **Async Iterator** | `query.ts` 流式处理 | API流式响应处理 |

---

## 5. 关键文件速查

| 文件 | 职责 |
|------|------|
| `entrypoints/cli.tsx` | CLI主入口，版本检查，快速路径分发 |
| `main.tsx` | 主程序，全量模块导入和初始化编排 |
| `setup.ts` | 会话初始化（Git根目录、Hooks、Worktree） |
| `state/store.ts` | 核心响应式Store实现 |
| `state/AppStateStore.ts` | App级别状态类型定义 |
| `QueryEngine.ts` | 查询执行引擎，主消息循环 |
| `query.ts` | processUserInput + API调用 + 工具循环 |
| `context.ts` | System/User Context构建器 |
| `commands.ts` | 80+命令的注册表 |
| `tools.ts` | 30+工具的注册表 |
| `Tool.ts` | 工具基类，定义execute/description/validate |
| `components/App.tsx` | React根组件，Provider组合 |
| `screens/REPL.tsx` | 主交互界面（4926行庞然大物） |
| `coordinator/coordinatorMode.ts` | 多Agent协调模式 |
| `hooks/useCanUseTool.tsx` | 权限判断Hook |
| `context/notifications.tsx` | 通知队列管理 |
| `services/compact/` | 上下文压缩服务 |
| `services/mcp/` | MCP服务器管理 |
| `utils/messages.ts` | 消息构建工具函数 |

---

## 6. 核心目录结构

```
src/
├── entrypoints/          # 程序入口点
│   ├── cli.tsx          # CLI快速路径分发
│   ├── init.ts          # 初始化函数
│   └── sdk/             # Agent SDK类型
├── bootstrap/           # 启动状态
│   └── state.ts         # 全局单例状态
├── coordinator/         # 协调模式
│   └── coordinatorMode.ts
├── commands/            # 80+命令实现
│   ├── help/
│   ├── commit.ts
│   ├── review.ts
│   └── ...
├── tools/              # 30+工具实现
│   ├── BashTool/
│   ├── AgentTool/
│   ├── FileEditTool/
│   └── ...
├── state/              # React状态管理
│   ├── store.ts        # 极简Store
│   └── AppStateStore.ts
├── context/            # React Context
│   ├── mailbox.tsx     # 进程间消息
│   └── notifications.tsx
├── components/        # 80+UI组件
│   ├── App.tsx
│   ├── PromptInput/
│   ├── permissions/
│   └── ...
├── screens/            # 顶层屏幕
│   └── REPL.tsx       # 主交互界面
├── services/          # 后台服务
│   ├── api/           # Claude API
│   ├── compact/       # 上下文压缩
│   └── mcp/           # MCP服务器
├── hooks/             # React Hooks
│   └── useCanUseTool.tsx
├── utils/             # 工具函数
│   ├── messages.ts
│   ├── permissions/
│   └── ...
└── ink/               # Ink渲染器
    ├── root.ts
    ├── components/
    └── hooks/
```

---

## 7. 模块详解

### 7.1 入口层 (entrypoints/)

**核心文件**: `cli.tsx`

职责：
- 程序主入口点
- 快速路径分发：处理 `--version`、`--daemon`、`--bg` 等命令
- 版本信息输出

关键代码模式：
```typescript
// 零导入快速路径
if (process.argv.includes('--version')) {
  console.log(version)
  process.exit(0)
}
```

### 7.2 核心引擎 (QueryEngine, query.ts)

**QueryEngine** 是整个应用的核心驱动引擎，负责：
- 管理消息循环
- 调用 Claude API
- 执行工具循环
- 处理上下文压缩

**query.ts** 负责：
- 用户输入处理 (`processUserInput`)
- API 调用封装
- 工具执行循环

### 7.3 状态管理 (state/)

**store.ts** - 约 30 行的极简发布订阅 Store：
- `getState()` - 获取当前状态
- `setState(updater)` - 更新状态并通知所有监听者
- `subscribe(listener)` - 订阅状态变化，返回取消订阅函数

**AppStateStore.ts** - React 响应式状态：
- 基于 `store.ts` 构建
- 提供类型安全的状态操作
- 驱动 React UI 更新

### 7.4 命令系统 (commands/)

80+ 命令，包括：
- `/help` - 帮助命令
- `/commit` - Git 提交
- `/review` - 代码审查
- `/model` - 模型切换
- 等等...

命令注册表 (`commands.ts`) 统一管理所有命令的注册和执行。

### 7.5 工具系统 (tools/)

30+ 工具，包括：
- `BashTool` - 执行 Bash 命令
- `AgentTool` - 派发子 Agent（Coordinator 模式核心）
- `FileEditTool` - 文件编辑
- `GrepTool` - 代码搜索
- `WebFetchTool` - 网页获取
- 等等...

工具基类定义：
```typescript
class Tool {
  description(): string    // 工具描述
  validate(params): void   // 参数校验
  execute(params): Promise<Result>  // 执行
}
```

### 7.6 UI 渲染层 (Ink + React)

**Ink** 是 React 的终端实现，允许使用 React 组件构建 TUI。

**组件树**：
```
App
└── REPL
    ├── Header
    ├── Messages
    ├── PromptInput
    ├── Permissions
    └── ...
```

### 7.7 协调模式 (coordinator/)

**Coordinator Mode** 是 Claude Code 的多 Agent 编排机制：

- 主 Agent 扮演协调者角色
- 通过 `AgentTool` 派发 worker 子 Agent
- 子 Agent 结果通过 `<task-notification>` 异步返回
- 支持 Swarm 模式的多 Agent 协作

### 7.8 服务层 (services/)

| 服务 | 职责 |
|------|------|
| `api/` | Claude API 调用、重试、流式处理 |
| `compact/` | 上下文压缩，当 token 超限时压缩对话历史 |
| `mcp/` | MCP (Model Context Protocol) 服务器管理 |
| `analytics/` | 遥测和增长分析 |
| `plugins/` | 插件系统 |

---

## 8. 技术栈

| 类别 | 技术 |
|------|------|
| **运行时** | Bun |
| **UI** | Ink (React for Terminal) |
| **状态** | 自研 Pub/Sub Store + React Hooks |
| **CLI** | Commander.js |
| **API** | Claude API (流式) |
| **协议** | MCP (Model Context Protocol) |
| **测试** | 测试框架 |

---

## 9. 总结

Claude Code 是一个设计精良的 CLI 应用：

1. **分层+模块化**：清晰的分层架构，每个模块职责单一
2. **性能优先**：快速路径优化、Feature Flag DCE、极简状态管理
3. **可扩展**：插件化的工具和命令系统
4. **多 Agent**：Coordinator 模式支持复杂任务分解
5. **TypeScript**：完整的类型安全
