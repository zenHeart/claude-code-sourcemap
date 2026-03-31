# Claude Code 代码库架构分析报告

> 分析日期：2026-03-31  
> 源码目录：restored-src/src  
> 目标：让新人能照此文档复刻一个完整的 Claude Code

---

## 目录

1. [整体架构](#1-整体架构)
2. [核心技术栈](#2-核心技术栈)
3. [用户交互完整流程](#3-用户交互完整流程)
4. [Agent 工程化](#4-agent-工程化)
5. [扩展机制](#5-扩展机制)
6. [其他工程经验](#6-其他工程经验)

---

## 1. 整体架构

### 1.1 分层架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                         用户层                                    │
│                   命令行界面（Terminal）                           │
└─────────────────────────┬───────────────────────────────────────┘
                            │ 用户输入
┌─────────────────────────▼───────────────────────────────────────┐
│                        入口层                                      │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐          │
│  │  cli.tsx   │───▶│  main.tsx  │───▶│ setup.ts    │          │
│  │ 快速路径    │    │ 全量初始化  │    │ 会话初始化  │          │
│  └─────────────┘    └─────────────┘    └─────────────┘          │
└─────────────────────────┬───────────────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────────────┐
│                       核心引擎层                                   │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐   │
│  │  QueryEngine    │  │   query.ts     │  │   context.ts   │   │
│  │   执行循环       │◀─▶│  消息处理     │◀─▶│  上下文构建    │   │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘   │
└─────────────────────────┬───────────────────────────────────────┘
                          │
        ┌─────────────────┼─────────────────┐
        ▼                 ▼                 ▼
┌───────────────┐  ┌───────────────┐  ┌───────────────┐
│   状态管理层   │  │   工具层      │  │   命令层      │
│  store.ts    │  │   Tool.ts    │  │ commands.ts │
│  AppState    │  │  30+ 工具    │  │  80+ 命令    │
└───────────────┘  └───────────────┘  └───────────────┘
        │
┌─────────────────────────▼───────────────────────────────────────┐
│                       服务层                                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │ API 服务  │  │ MCP 服务  │  │ Compact  │  │ Analytics│       │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘       │
└─────────────────────────┬───────────────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────────────┐
│                       UI 渲染层                                  │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │   Ink    │  │ App.tsx  │  │ REPL.tsx │  │Components│       │
│  │ React终端│  │  根组件   │  │ 主交互   │  │  80+ 组件 │       │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘       │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 模块依赖关系

```
cli.tsx (入口)
    │
    ▼
main.tsx (初始化)
    │
    ▼
setup.ts (会话初始化)
    │
    ▼
QueryEngine.ts (核心引擎)
    │
    ├──▶ query.ts (消息处理)
    │         │
    │         ▼
    │    context.ts (上下文构建)
    │
    ├──▶ tools.ts (工具注册)
    │         │
    │         ▼
    │    Tool.ts (工具基类)
    │
    ├──▶ commands.ts (命令注册)
    │
    ├──▶ services/
    │    ├── api/ (Claude API)
    │    ├── compact/ (上下文压缩)
    │    └── mcp/ (MCP 服务器)
    │
    └──▶ state/
         ├── store.ts (极简 Store)
         └── AppStateStore.ts (React 状态)
```

---

## 2. 核心技术栈

### 2.1 技术选型

| 类别 | 技术 | 用途 |
|------|------|------|
| **运行时** | Bun | 启动速度比 Node 快 |
| **UI** | Ink | React 的终端实现，用 React 写 TUI |
| **状态** | 自研 Pub/Sub Store | 约 30 行，比 Redux 简洁 |
| **CLI** | Commander.js | 命令行参数解析 |
| **API** | Anthropic Claude API | LLM 调用 |
| **协议** | MCP (Model Context Protocol) | 工具服务器协议 |
| **类型** | TypeScript + Zod | 端到端类型安全 |

### 2.2 关键库

```json
{
  "dependencies": {
    "@anthropic-ai/sdk": "^0.40.0",
    "@modelcontextprotocol/sdk": "^0.5.0",
    "ink": "^5.0.0",
    "react": "^18.0.0",
    "commander": "^12.0.0",
    "zod": "^3.23.0",
    "lodash-es": "^4.17.0"
  }
}
```

### 2.3 Feature Flag + DCE

**Bun 特有的编译时消除**：

```typescript
import { feature } from 'bun:bundle'

// 编译时内联检查，启用则包含代码，否则完全消除
if (feature('COORDINATOR_MODE')) {
  // 只有开启 COORDINATOR_MODE 编译标志才会包含这段代码
}

// 常用 Flags
const COORDINATOR_MODE = feature('COORDINATOR_MODE')  // 多 Agent 协调
const KAIROS = feature('KAIROS')                     // 助手模式
const VOICE_MODE = feature('VOICE_MODE')              // 语音模式
const BG_SESSIONS = feature('BG_SESSIONS')            // 后台会话
```

---

## 3. 用户交互完整流程

### 3.1 启动流程

**Step 1: cli.tsx 快速路径**

```typescript
// src/entrypoints/cli.tsx
async function main(): Promise<void> {
  const args = process.argv.slice(2);

  // ⚡ 快速路径：--version 零导入
  if (args.length === 1 && (args[0] === '--version' || args[0] === '-v')) {
    console.log(`${MACRO.VERSION} (Claude Code)`);
    return;  // 直接返回，不加载任何模块
  }

  // 其他命令才加载完整模块
  const { profileCheckpoint } = await import('../utils/startupProfiler.js');
  profileCheckpoint('cli_entry');

  // 动态导入主程序
  const { cliMain } = await import('./main.js');
  return cliMain();
}
```

**Step 2: main.tsx 全量初始化**

```typescript
// src/main.tsx
export async function cliMain(): Promise<void> {
  profileCheckpoint('main_tsx_entry');

  // 并行预读配置（减少等待时间）
  const [mdmConfig, keychain] = await Promise.all([
    startMdmRawRead(),           // MDM 配置
    startKeychainPrefetch(),     // Keychain
  ]);

  // 导入大量模块（~135ms）
  const [
    { init },
    { getCommands },
    { launchRepl },
  ] = await Promise.all([
    import('./init.js'),
    import('./commands.js'),
    import('./screens/REPL.js'),
  ]);

  // 初始化环境
  await init();

  // 获取命令
  const commands = await getCommands();

  // 启动 TUI
  await launchRepl(root, { commands }, replProps);
}
```

**Step 3: setup.ts 会话初始化**

```typescript
// src/setup.ts
export async function setup(): Promise<SetupResult> {
  // 定位项目根目录
  const gitRoot = await findCanonicalGitRoot();

  // 快照 Hook 配置
  const hooksConfig = captureHooksConfigSnapshot();

  // 初始化文件监控
  const fileWatcher = initializeFileChangedWatcher();

  // 启动进程间通信
  const udsServer = startUdsMessaging();

  // 快照 Swarm 模式配置
  const teammateSnapshot = captureTeammateModeSnapshot();

  return { gitRoot, hooksConfig, fileWatcher, udsServer, teammateSnapshot };
}
```

### 3.2 查询执行流程

**核心查询循环**：

```typescript
// src/query.ts (核心查询循环)
export async function* query(params: QueryParams): AsyncGenerator<...> {
  const consumedCommandUuids: string[] = [];
  const terminal = yield* queryLoop(params, consumedCommandUuids);

  // 通知命令完成
  for (const uuid of consumedCommandUuids) {
    notifyCommandLifecycle(uuid, 'completed');
  }

  return terminal;
}

async function* queryLoop(params: QueryParams, ...): AsyncGenerator<...> {
  while (true) {
    // 1. 检查是否需要压缩上下文
    if (shouldCompact(params.messages)) {
      yield* compactMessages(params);
    }

    // 2. 构建 System Prompt
    const systemPrompt = buildSystemPrompt(params);

    // 3. 调用 Claude API（流式）
    const stream = await callClaudeAPI(systemPrompt, params.messages);

    // 4. 处理流式响应
    for await (const event of stream) {
      if (event.type === 'content_block') {
        if (event.toolUse) {
          yield* handleToolUse(event.toolUse);
        }
      }
      yield event;
    }

    // 5. 检查是否结束
    if (isComplete(response)) break;
  }
}

// 工具执行循环
async function* handleToolUse(toolUse: ToolUseBlock): AsyncGenerator<...> {
  // 权限检查
  const decision = await canUseTool(toolUse.name, toolUse.input);
  if (decision === 'deny') {
    yield createToolResult(toolUse.id, 'Permission denied');
    return;
  }

  // 获取工具实例并执行
  const tool = findToolByName(tools, toolUse.name);
  const result = await tool.call(toolUse.input, context);

  yield createToolResultMessage(toolUse.id, result);
}
```

---

## 4. Agent 工程化

### 4.1 核心循环实现

**QueryEngine 是 Agent 的心脏**：

```typescript
// 概念性实现
export class QueryEngine {
  private tools: Tools;
  private messages: Message[] = [];
  private context: ToolUseContext;

  async *handleNextMessage(userInput: string): AsyncGenerator<Message | StreamEvent> {
    // 1. 追加用户消息
    this.messages.push(createUserMessage(userInput));

    // 2. 主循环
    while (true) {
      // 2.1 构建请求
      const request = this.buildRequest();

      // 2.2 调用 API
      const response = await this.callAPI(request);

      // 2.3 处理响应
      for (const block of response.content) {
        if (block.type === 'tool_use') {
          const result = await this.executeTool(block);
          this.messages.push(createToolResult(block.id, result));
        } else if (block.type === 'text') {
          yield createAssistantMessage(block.text);
        }
      }

      // 2.4 检查是否完成
      if (!response.hasMore) break;
    }
  }
}
```

### 4.2 提示词工程

**System Prompt 构建**：

```typescript
// src/context.ts
export function buildSystemPrompt(params: QueryParams): SystemPrompt {
  const sections: string[] = [];

  // 1. 核心指令
  sections.push(`你是一个 AI 编程助手，名为 Claude Code。`);
  sections.push(`当前项目根目录: ${params.gitRoot}`);

  // 2. 可用工具
  sections.push(`## 可用工具
${params.tools.map(t => `- ${t.name}: ${t.description}`).join('\n')}`);

  // 3. 工作目录规则
  sections.push(`## 工作目录规则
- 只在项目根目录下操作
- 不要修改 .gitignore 中的文件`);

  // 4. 安全规则
  sections.push(`## 安全规则
- 删除文件前确认
- 危险操作需要用户确认`);

  // 5. 上下文信息
  if (params.attachedFiles.length > 0) {
    sections.push(`## 附加文件
${params.attachedFiles.map(f => `// ${f.path}\n${f.content}`).join('\n\n')}`);
  }

  return sections.join('\n\n');
}
```

### 4.3 上下文工程

**消息规范化**：

```typescript
// src/utils/messages.ts
export function normalizeMessagesForAPI(messages: Message[]): Message[] {
  const result: Message[] = [];

  for (const msg of messages) {
    switch (msg.type) {
      case 'user':
        result.push({
          ...msg,
          content: transformContent(msg.content),
        });
        break;
      case 'assistant':
        result.push({
          ...msg,
          content: msg.content.filter(b => b.type !== 'thinking'),
        });
        break;
      case 'tool_result':
        result.push({
          ...msg,
          content: truncateIfNeeded(msg.content, MAX_RESULT_LENGTH),
        });
        break;
    }
  }

  return result;
}
```

**上下文压缩（Compact）**：

```typescript
// src/services/compact/compact.ts
export async function compactMessages(params: QueryParams): Promise<void> {
  const messages = params.messages;

  // 1. 计算压缩点
  const compactPoint = findCompactPoint(messages);

  // 2. 提取要压缩的消息
  const messagesToCompact = messages.slice(0, compactPoint);

  // 3. 生成摘要
  const summary = await generateSummary(messagesToCompact);

  // 4. 替换为摘要消息
  params.messages = [
    createSummaryMessage(summary),
    ...messages.slice(compactPoint),
  ];
}
```

---

## 5. 扩展机制

### 5.1 工具系统架构

**工具基类**：

```typescript
// src/Tool.ts
export type Tool<
  Input extends AnyObject = AnyObject,
  Output = unknown,
  P extends ToolProgressData = ToolProgressData,
> = {
  // 核心方法
  call(
    args: z.infer<Input>,
    context: ToolUseContext,
    canUseTool: CanUseToolFn,
    parentMessage: AssistantMessage,
    onProgress?: ToolCallProgress<P>,
  ): Promise<ToolResult<Output>>;

  description(args: z.infer<Input>, options: DescriptionOptions): Promise<string>;

  // 必填属性
  readonly name: string;
  readonly inputSchema: Input;
  readonly maxResultSizeChars: number;

  // 可选方法（有默认值）
  isEnabled(): boolean;
  isConcurrencySafe(input: z.infer<Input>): boolean;
  isReadOnly(input: z.infer<Input>): boolean;
}

// 工具工厂函数
const TOOL_DEFAULTS = {
  isEnabled: () => true,
  isConcurrencySafe: (_input?: unknown) => false,
  isReadOnly: (_input?: unknown) => false,
  isDestructive: (_input?: unknown) => false,
  checkPermissions: async () => ({ behavior: 'allow' as const }),
};

export function buildTool<D extends ToolDef>(def: D): Tool {
  return {
    ...TOOL_DEFAULTS,
    userFacingName: () => def.name,
    ...def,
  } as Tool;
}
```

**自定义工具示例**：

```typescript
// 自定义 Hello World 工具
export const HelloWorldTool = buildTool({
  name: 'hello_world',
  description: '打印 Hello World',

  inputSchema: z.object({
    name: z.string().optional().describe('要问候的名字'),
  }),

  async call(args, context) {
    const greeting = args.name
      ? `Hello, ${args.name}!`
      : 'Hello, World!';

    return {
      data: { message: greeting, timestamp: Date.now() },
    };
  },
});
```

### 5.2 命令系统架构

**命令注册表**：

```typescript
// src/commands.ts
export async function getCommands(): Promise<Command[]> {
  return [
    new HelpCommand(),
    new ExitCommand(),
    new ModelCommand(),
    new PermissionsCommand(),
  ];
}

export class Command {
  name: string;
  description: string;
  alias?: string[];

  async run(args: string[], context: CommandContext): Promise<void> {
    throw new Error('Not implemented');
  }
}
```

**斜杠命令解析**：

```typescript
// src/utils/messageQueueManager.ts
export function parseSlashCommand(input: string): { command: string; args: string } | null {
  const match = input.match(/^\/(\w+)(?:\s+(.*))?$/);
  if (!match) return null;

  return { command: match[1], args: match[2] || '' };
}
```

### 5.3 Hook 系统架构

**Hook 类型**：

```typescript
// src/types/hooks.ts
export type HookType =
  | 'pre_tool_use'      // 工具执行前
  | 'post_tool_use'     // 工具执行后
  | 'pre_query'          // 查询前
  | 'post_query'         // 查询后
  | 'pre_compact'        // 压缩前
  | 'post_compact';      // 压缩后

export type Hook = {
  id: string;
  type: HookType;
  if?: string;           // 条件表达式
  run: (context: HookContext) => Promise<void>;
};
```

**权限 Hook**：

```typescript
// src/hooks/useCanUseTool.tsx
export function useCanUseTool() {
  const canUseTool = useCallback(async (
    tool: Tool,
    input: Record<string, unknown>,
    context: ToolUseContext,
  ): Promise<PermissionDecision> => {
    // 1. 检查配置规则
    const configDecision = checkConfigRules(tool.name, input);
    if (configDecision) return configDecision;

    // 2. 检查 Always Allow
    if (isAlwaysAllowed(tool.name)) {
      return { behavior: 'allow' };
    }

    // 3. 检查 Always Deny
    if (isAlwaysDenied(tool.name)) {
      return { behavior: 'deny' };
    }

    // 4. 交互式确认
    return { behavior: 'ask' };
  }, []);

  return canUseTool;
}
```

---

## 6. 其他工程经验

### 6.1 极简响应式 Store

**约 30 行的完整实现**：

```typescript
// src/state/store.ts
type Listener = () => void;

export function createStore<T>(initialState: T, onChange?) {
  let state = initialState;
  const listeners = new Set<Listener>();  // Set 自动去重

  return {
    getState: () => state,

    setState: (updater) => {
      const prev = state;
      const next = updater(prev);
      if (Object.is(next, prev)) return;  // 新旧相同则跳过
      state = next;
      onChange?.({ newState: next, oldState: prev });
      for (const listener of listeners) listener();
    },

    subscribe: (listener) => {
      listeners.add(listener);
      return () => listeners.delete(listener);  // 返回取消订阅
    },
  };
}
```

**借鉴意义**：
- `Object.is()` 检查避免无效渲染
- `Set` 存储 listeners 自动去重
- 返回取消订阅函数，内存安全
- 约 30 行 vs Redux 100+ 行

### 6.2 依赖注入

**QueryDeps 依赖注入**：

```typescript
// src/query/deps.ts
export type QueryDeps = {
  // API 调用
  callClaudeAPI: typeof callClaudeAPI;

  // 工具
  tools: Tools;

  // 权限
  canUseTool: CanUseToolFn;

  // 压缩
  compact: typeof compactMessages;

  // ... 更多依赖
};

// 使用
export async function* query(params: QueryParams): AsyncGenerator<...> {
  const deps = params.deps ?? productionDeps;  // 默认生产依赖

  while (true) {
    // 使用注入的依赖
    const response = await deps.callClaudeAPI(request);
    // ...
  }
}
```

### 6.3 Coordinator 模式

**多 Agent 协调**：

```typescript
// src/coordinator/coordinatorMode.ts
export function isCoordinatorMode(): boolean {
  if (feature('COORDINATOR_MODE')) {
    return isEnvTruthy(process.env.CLAUCE_CODE_COORDINATOR_MODE);
  }
  return false;
}

// Worker 可用的工具
const workerTools = isEnvTruthy(process.env.CLAUCE_CODE_SIMPLE)
  ? [BASH_TOOL_NAME, FILE_READ_TOOL_NAME, FILE_EDIT_TOOL_NAME]
  : Array.from(ASYNC_AGENT_ALLOWED_TOOLS);
```

**主 Agent 派发 Worker**：

```typescript
// src/tools/AgentTool/AgentTool.ts
export const AgentTool = buildTool({
  name: 'agent',
  description: '派发子 Agent 执行任务',

  inputSchema: z.object({
    task: z.string().describe('任务描述'),
    mode: z.enum(['coordinator', 'worker']).optional(),
  }),

  async call(args, context) {
    if (args.mode === 'coordinator') {
      // 主模式：分解任务，派发 worker
      const workers = await spawnWorkers(args.task);
      const results = await Promise.all(workers.map(w => w.execute()));
      return { data: aggregateResults(results) };
    } else {
      // Worker 模式：执行具体任务
      return await executeTask(args.task);
    }
  },
});
```

---

## 总结

Claude Code 核心设计经验：

| 经验 | 说明 |
|------|------|
| **快速路径优化** | 零导入 `--version`，动态 `import()` |
| **极简 Store** | 30 行实现响应式 |
| **Feature Flag + DCE** | 编译时消除废弃代码 |
| **工具插件化** | `Tool` 基类 + `buildTool()` 工厂 |
| **Hook 注入权限** | 权限逻辑可替换 |
| **AsyncGenerator 流式** | `yield*` 委托实现流式处理 |
| **依赖注入** | 核心逻辑与实现分离 |
| **Coordinator 模式** | 多 Agent 编排 |

**核心原则**：性能优先、简洁、可扩展、可测试
