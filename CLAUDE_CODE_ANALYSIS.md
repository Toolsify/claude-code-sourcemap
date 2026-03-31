# Claude Code 技术架构深度解析

> **版本**: 2.1.88 | **分析日期**: 2026-04-01 | **源码来源**: npm package source map 还原

---

## 摘要

Claude Code 是 Anthropic 推出的终端 AI 编程助手，通过深度分析其还原源码（4756 个文件，含 1906 个 TypeScript/TSX 源文件），本文揭示了其强大的技术架构设计。Claude Code 的核心竞争力在于：**多层安全防御体系**、**丰富的工具生态系统**、**先进的多 Agent 协作机制**、**灵活的扩展架构**以及**智能的上下文管理**。

---

## 目录

1. [项目概述与技术栈](#一项目概述与技术栈)
2. [核心架构设计](#二核心架构设计)
3. [安全执行体系](#三安全执行体系)
4. [工具系统架构](#四工具系统架构)
5. [多 Agent 协作系统](#五多-agent-协作系统)
6. [扩展性设计](#六扩展性设计)
7. [上下文与记忆系统](#七上下文与记忆系统)
8. [性能优化策略](#八性能优化策略)
9. [UI/UX 架构](#九uiux-架构)
10. [企业级特性](#十企业级特性)
11. [核心竞争力总结](#核心竞争力总结)

---

## 一、项目概述与技术栈

### 1.1 技术栈总览

| 类别 | 技术选型 | 说明 |
|------|----------|------|
| **运行时** | Node.js / Bun | 支持双运行时，Bun 提供更快启动 |
| **UI 框架** | React + Ink | 终端 UI 渲染，声明式组件模型 |
| **语言** | TypeScript/TSX | 完整类型系统，编译时安全 |
| **状态管理** | 自定义 Store | 类 Zustand 的外部状态管理 |
| **包管理** | npm/bun | 199 个依赖包 |
| **构建工具** | Bun Bundle | 支持 Dead Code Elimination |

### 1.2 项目规模

```
总文件数:     4756 个
源码文件:     1906 个 (.ts/.tsx)
工具实现:     43 个
命令实现:     101 个
UI 组件:      144 个
工具函数:     329 个
依赖包:       199 个
```

### 1.3 目录结构概览

```
restored-src/src/
├── main.tsx                 # CLI 入口，初始化协调
├── QueryEngine.ts           # 查询引擎，会话生命周期
├── Tool.ts                  # 工具抽象层
├── tools/                   # 43 个工具实现
│   ├── BashTool/           # Shell 命令执行
│   ├── FileEditTool/       # 文件编辑
│   ├── AgentTool/          # 多 Agent 协作
│   └── MCPTool/            # MCP 协议集成
├── commands/                # 101 个 CLI 命令
├── services/                # 服务层（API、MCP、分析）
├── utils/                   # 329 个工具函数
│   ├── permissions/        # 24 个权限模块
│   ├── sandbox/            # 沙箱执行
│   └── hooks/              # 17 个 Hook 模块
├── state/                   # 状态管理
├── components/              # 144 个 React 组件
└── screens/                 # 屏幕（REPL、Doctor）
```

---

## 二、核心架构设计

### 2.1 系统架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                         CLI Entry (main.tsx)                    │
├─────────────────────────────────────────────────────────────────┤
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │  Commander   │  │   Init()     │  │  Migrations  │          │
│  │  (命令解析)   │  │  (初始化)     │  │  (数据迁移)   │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
├─────────────────────────────────────────────────────────────────┤
│                      QueryEngine (查询引擎)                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │   Query      │  │   Tools      │  │   Messages   │          │
│  │  (查询处理)   │  │  (工具调用)   │  │  (消息管理)   │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
├─────────────────────────────────────────────────────────────────┤
│                     State Management (状态管理)                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │  AppState    │  │   Store      │  │  Selectors   │          │
│  │  (应用状态)   │  │  (状态存储)   │  │  (状态选择)   │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
├─────────────────────────────────────────────────────────────────┤
│                      Security Layer (安全层)                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │  Permissions │  │   Sandbox    │  │  Classifier  │          │
│  │  (权限控制)   │  │  (沙箱执行)   │  │  (风险分类)   │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
├─────────────────────────────────────────────────────────────────┤
│                       Tool System (工具系统)                     │
│  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐       │
│  │  Bash  │ │  File  │ │ Agent  │ │  MCP   │ │  Web   │ ...   │
│  └────────┘ └────────┘ └────────┘ └────────┘ └────────┘       │
├─────────────────────────────────────────────────────────────────┤
│                    Extension System (扩展系统)                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │    Hooks     │  │   Plugins    │  │    Skills    │          │
│  │  (事件钩子)   │  │  (插件系统)   │  │  (技能系统)   │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 入口系统 (main.tsx)

Claude Code 的入口设计极为精巧，采用**并行预取**策略优化启动性能：

```typescript
// main.tsx - 启动时的并行优化
// 1. 性能分析检查点
profileCheckpoint('main_tsx_entry');

// 2. 并行启动 MDM 子进程（与导入并行）
startMdmRawRead();

// 3. 并行预取 Keychain（macOS 安全存储）
startKeychainPrefetch();

// 4. 然后才开始重量级模块导入
import { Command as CommanderCommand } from '@commander-js/extra-typings';
import React from 'react';
// ... 135ms 的导入时间
```

**设计亮点**：
- **Side-effect 优先**：在模块导入前启动异步任务
- **并行预取**：MDM、Keychain 与模块导入并行执行
- **性能监控**：`profileCheckpoint` 追踪每个阶段耗时

### 2.3 查询引擎 (QueryEngine)

查询引擎是 Claude Code 的核心，负责处理用户查询和工具调用：

```typescript
// QueryEngine.ts - 核心查询处理
export async function query(
  options: QueryOptions,
): Promise<QueryResult> {
  // 1. 构建系统提示词
  const systemPrompt = await buildSystemPrompt(options);
  
  // 2. 加载工具集
  const tools = await assembleToolPool(options);
  
  // 3. 处理用户输入
  const processedInput = await processUserInput(
    options.input,
    options.context,
  );
  
  // 4. 执行 API 调用
  const response = await callAPI({
    system: systemPrompt,
    messages: options.messages,
    tools: tools,
  });
  
  // 5. 处理工具调用
  return handleToolCalls(response, options);
}
```

### 2.4 状态管理

Claude Code 使用自定义的外部状态存储，类似 Zustand：

```typescript
// state/AppStateStore.ts
export type AppState = {
  // 会话状态
  sessionId: string;
  messages: Message[];
  
  // 工具状态
  toolPermissionContext: ToolPermissionContext;
  inProgressToolUseIDs: Set<string>;
  
  // UI 状态
  isWaitingForInput: boolean;
  currentToolUse: ToolUse | null;
  
  // 成本状态
  totalCost: number;
  costThreshold: number;
};

// 创建 Store
export function createStore(
  initialState: AppState,
  onChange?: (args: { newState: AppState; oldState: AppState }) => void,
): AppStateStore {
  let state = initialState;
  
  return {
    getState: () => state,
    setState: (updater) => {
      const oldState = state;
      state = typeof updater === 'function' ? updater(state) : updater;
      onChange?.({ newState: state, oldState });
    },
  };
}
```

---

## 三、安全执行体系

### 3.1 多层权限系统

Claude Code 实现了业界领先的 **24 个权限模块**，形成纵深防御：

```
┌─────────────────────────────────────────────────────────────┐
│                    Permission Architecture                   │
├─────────────────────────────────────────────────────────────┤
│  Layer 1: Permission Modes                                  │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐          │
│  │ default │ │  plan   │ │ accept  │ │ bypass  │          │
│  │ (询问)  │ │ (计划)  │ │ (自动)  │ │ (绕过)  │          │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘          │
├─────────────────────────────────────────────────────────────┤
│  Layer 2: Permission Rules                                  │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐          │
│  │ AlwaysAllow │ │ AlwaysDeny  │ │  AlwaysAsk  │          │
│  │ (白名单)    │ │ (黑名单)    │ │ (强制询问)  │          │
│  └─────────────┘ └─────────────┘ └─────────────┘          │
├─────────────────────────────────────────────────────────────┤
│  Layer 3: Intelligent Classifier                            │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐          │
│  │   Bash      │ │  Filesystem │ │  Network    │          │
│  │ Classifier  │ │  Validator  │ │  Filter     │          │
│  └─────────────┘ └─────────────┘ └─────────────┘          │
├─────────────────────────────────────────────────────────────┤
│  Layer 4: Sandbox Isolation                                 │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐          │
│  │ Filesystem  │ │   Network   │ │  Process    │          │
│  │ Restrictions│ │  Restrictions│ │  Isolation  │          │
│  └─────────────┘ └─────────────┘ └─────────────┘          │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 权限模式详解

```typescript
// utils/permissions/PermissionMode.ts
export const PERMISSION_MODE_CONFIG = {
  default: {
    title: 'Default',
    shortTitle: 'Default',
    symbol: '',
    color: 'text',
    external: 'default',
  },
  plan: {
    title: 'Plan Mode',
    shortTitle: 'Plan',
    symbol: PAUSE_ICON,  // ⏸
    color: 'planMode',
    external: 'plan',
  },
  acceptEdits: {
    title: 'Accept edits',
    shortTitle: 'Accept',
    symbol: '⏵⏵',
    color: 'autoAccept',
    external: 'acceptEdits',
  },
  bypassPermissions: {
    title: 'Bypass Permissions',
    shortTitle: 'Bypass',
    symbol: '⏵⏵',
    color: 'error',
    external: 'bypassPermissions',
  },
  auto: {
    title: 'Auto mode',
    shortTitle: 'Auto',
    symbol: '⏵⏵',
    color: 'warning',
    external: 'default',
  },
};
```

### 3.3 权限上下文

```typescript
// Tool.ts - 工具权限上下文
export type ToolPermissionContext = {
  // 当前权限模式
  mode: PermissionMode;
  
  // 额外工作目录（多目录支持）
  additionalWorkingDirectories: Map<string, AdditionalWorkingDirectory>;
  
  // 权限规则
  alwaysAllowRules: ToolPermissionRulesBySource;
  alwaysDenyRules: ToolPermissionRulesBySource;
  alwaysAskRules: ToolPermissionRulesBySource;
  
  // 绕过权限模式
  isBypassPermissionsModeAvailable: boolean;
  
  // 自动模式（AI 分类器）
  isAutoModeAvailable?: boolean;
  
  // 危险规则过滤
  strippedDangerousRules?: ToolPermissionRulesBySource;
  
  // 避免权限提示（后台 Agent）
  shouldAvoidPermissionPrompts?: boolean;
};
```

### 3.4 沙箱执行机制

```typescript
// utils/sandbox/sandbox-adapter.ts
export class SandboxManager {
  // 检查沙箱是否启用
  static isSandboxingEnabled(): boolean {
    return getSettingsForSource('policySettings')
      ?.sandbox?.enabled ?? false;
  }
  
  // 文件系统限制
  static getFsRestrictions(): {
    allowRead: string[];
    allowWrite: string[];
    denyRead: string[];
    denyWrite: string[];
  } {
    const settings = getSettingsForSource('policySettings');
    return {
      allowRead: settings?.sandbox?.filesystem?.allowRead ?? [],
      allowWrite: settings?.sandbox?.filesystem?.allowWrite ?? [],
      denyRead: settings?.sandbox?.filesystem?.denyRead ?? [],
      denyWrite: settings?.sandbox?.filesystem?.denyWrite ?? [],
    };
  }
  
  // 网络限制
  static getNetworkRestrictions(): {
    allowHosts: NetworkHostPattern[];
    denyHosts: NetworkHostPattern[];
  } {
    const settings = getSettingsForSource('policySettings');
    return {
      allowHosts: settings?.sandbox?.network?.allowHosts ?? [],
      denyHosts: settings?.sandbox?.network?.denyHosts ?? [],
    };
  }
}
```

### 3.5 智能风险评估

```typescript
// utils/permissions/bashClassifier.ts
export type ClassifierResult = {
  matches: boolean;
  matchedDescription?: string;
  confidence: 'high' | 'medium' | 'low';
  reason: string;
};

export type ClassifierBehavior = 'deny' | 'ask' | 'allow';

// AI 驱动的命令分类
export async function classifyBashCommand(
  command: string,
  cwd: string,
  descriptions: string[],
  behavior: ClassifierBehavior,
  signal: AbortSignal,
  isNonInteractiveSession: boolean,
): Promise<ClassifierResult> {
  // 1. 检查危险模式
  if (matchesDangerousPattern(command)) {
    return {
      matches: true,
      confidence: 'high',
      reason: 'Command matches dangerous pattern',
    };
  }
  
  // 2. AI 分类（ANT-ONLY）
  if (isClassifierPermissionsEnabled()) {
    return await aiClassify(command, cwd, descriptions, signal);
  }
  
  return { matches: false, confidence: 'high', reason: 'Safe' };
}
```

---

## 四、工具系统架构

### 4.1 工具抽象层

Claude Code 的工具系统采用统一的抽象接口，所有工具都实现 `Tool` 接口：

```typescript
// Tool.ts - 工具接口定义
export type Tool<
  Input extends AnyObject = AnyObject,
  Output = unknown,
  P extends ToolProgressData = ToolProgressData,
> = {
  // 基本信息
  name: string;
  aliases?: string[];  // 向后兼容的别名
  searchHint?: string;  // 关键词搜索提示
  
  // 核心方法
  call(
    args: z.infer<Input>,
    context: ToolUseContext,
    canUseTool: CanUseToolFn,
    parentMessage: AssistantMessage,
    onProgress?: ToolCallProgress<P>,
  ): Promise<ToolResult<Output>>;
  
  description(
    input: z.infer<Input>,
    options: {
      isNonInteractiveSession: boolean;
      toolPermissionContext: ToolPermissionContext;
      tools: Tools;
    },
  ): Promise<string>;
  
  // Schema
  readonly inputSchema: Input;
  outputSchema?: z.ZodType<unknown>;
  
  // 安全检查
  isConcurrencySafe(input: z.infer<Input>): boolean;
  isEnabled(): boolean;
  isReadOnly(input: z.infer<Input>): boolean;
  isDestructive?(input: z.infer<Input>): boolean;
  
  // 权限检查
  checkPermissions(
    input: z.infer<Input>,
    context: ToolUseContext,
  ): Promise<PermissionResult>;
  
  validateInput?(
    input: z.infer<Input>,
    context: ToolUseContext,
  ): Promise<ValidationResult>;
  
  // UI 渲染
  prompt(options: {...}): Promise<string>;
  userFacingName(input: Partial<z.infer<Input>>): string;
  getToolUseSummary?(input: Partial<z.infer<Input>>): string | null;
  getActivityDescription?(input: Partial<z.infer<Input>>): string | null;
};
```

### 4.2 工具分类与实现

| 类别 | 工具 | 功能 |
|------|------|------|
| **文件操作** | FileReadTool, FileWriteTool, FileEditTool | 读写编辑文件 |
| **代码搜索** | GrepTool, GlobTool | 代码搜索和模式匹配 |
| **命令执行** | BashTool, PowerShellTool | Shell 命令执行 |
| **AI Agent** | AgentTool | 多 Agent 协作 |
| **MCP 集成** | MCPTool, ReadMcpResourceTool | MCP 协议工具 |
| **Web 操作** | WebFetchTool, WebSearchTool | 网页抓取和搜索 |
| **任务管理** | TaskCreateTool, TodoWriteTool | 任务和待办管理 |
| **其他** | NotebookEditTool, REPLTool | Jupyter、交互式环境 |

### 4.3 BashTool 深度分析

BashTool 是最复杂的工具之一，实现了智能命令分析：

```typescript
// tools/BashTool/BashTool.tsx
export function isSearchOrReadBashCommand(command: string): {
  isSearch: boolean;
  isRead: boolean;
  isList: boolean;
} {
  // 解析命令和操作符
  const partsWithOperators = splitCommandWithOperators(command);
  
  // 搜索命令集
  const BASH_SEARCH_COMMANDS = new Set([
    'find', 'grep', 'rg', 'ag', 'ack', 'locate', 'which', 'whereis'
  ]);
  
  // 读取命令集
  const BASH_READ_COMMANDS = new Set([
    'cat', 'head', 'tail', 'less', 'more',
    'wc', 'stat', 'file', 'strings',
    'jq', 'awk', 'cut', 'sort', 'uniq', 'tr'
  ]);
  
  // 列表命令集
  const BASH_LIST_COMMANDS = new Set(['ls', 'tree', 'du']);
  
  // 语义中性命令（不影响管道性质）
  const BASH_SEMANTIC_NEUTRAL_COMMANDS = new Set([
    'echo', 'printf', 'true', 'false', ':'
  ]);
  
  // 分析每个管道部分
  for (const part of partsWithOperators) {
    if (BASH_SEMANTIC_NEUTRAL_COMMANDS.has(baseCommand)) {
      continue;  // 跳过中性命令
    }
    
    if (!BASH_SEARCH_COMMANDS.has(baseCommand) &&
        !BASH_READ_COMMANDS.has(baseCommand) &&
        !BASH_LIST_COMMANDS.has(baseCommand)) {
      return { isSearch: false, isRead: false, isList: false };
    }
  }
  
  return { isSearch: hasSearch, isRead: hasRead, isList: hasList };
}
```

### 4.4 工具协作机制

```typescript
// Tool.ts - ToolUseContext
export type ToolUseContext = {
  options: {
    commands: Command[];
    debug: boolean;
    mainLoopModel: string;
    tools: Tools;
    verbose: boolean;
    thinkingConfig: ThinkingConfig;
    mcpClients: MCPServerConnection[];
    mcpResources: Record<string, ServerResource[]>;
    isNonInteractiveSession: boolean;
    agentDefinitions: AgentDefinitionsResult;
    maxBudgetUsd?: number;
    customSystemPrompt?: string;
    appendSystemPrompt?: string;
  };
  
  abortController: AbortController;
  readFileState: FileStateCache;
  getAppState(): AppState;
  setAppState(f: (prev: AppState) => AppState): void;
  
  // UI 回调
  setToolJSX?: SetToolJSXFn;
  addNotification?: (notif: Notification) => void;
  appendSystemMessage?: (msg: SystemMessage) => void;
  
  // 进度跟踪
  setInProgressToolUseIDs: (f: (prev: Set<string>) => Set<string>) => void;
  setResponseLength: (f: (prev: number) => number) => void;
  
  // 消息历史
  messages: Message[];
};
```

---

## 五、多 Agent 协作系统

### 5.1 Agent 架构设计

Claude Code 实现了先进的多 Agent 系统，支持任务委派和并行执行：

```
┌─────────────────────────────────────────────────────────────┐
│                    Multi-Agent Architecture                  │
├─────────────────────────────────────────────────────────────┤
│                     Main Thread (主 Agent)                   │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  User Query → QueryEngine → Tool Selection → Execute │  │
│  └───────────────────────────────────────────────────────┘  │
├─────────────────────────────────────────────────────────────┤
│                      Agent Tool (Agent 工具)                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │  Foreground │  │  Background │  │   Remote    │        │
│  │  (前台执行)  │  │  (后台执行)  │  │  (远程执行)  │        │
│  └─────────────┘  └─────────────┘  └─────────────┘        │
├─────────────────────────────────────────────────────────────┤
│                    Agent Isolation (隔离机制)                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │  Worktree   │  │   Remote    │  │    Team     │        │
│  │  (工作树)   │  │  (远程环境)  │  │  (团队协作)  │        │
│  └─────────────┘  └─────────────┘  └─────────────┘        │
└─────────────────────────────────────────────────────────────┘
```

### 5.2 AgentTool 实现

```typescript
// tools/AgentTool/AgentTool.tsx
const baseInputSchema = lazySchema(() => z.object({
  description: z.string().describe('A short (3-5 word) description of the task'),
  prompt: z.string().describe('The task for the agent to perform'),
  subagent_type: z.string().optional().describe('The type of specialized agent'),
  model: z.enum(['sonnet', 'opus', 'haiku']).optional().describe('Model override'),
  run_in_background: z.boolean().optional().describe('Run in background'),
}));

const fullInputSchema = lazySchema(() => {
  const multiAgentInputSchema = z.object({
    name: z.string().optional().describe('Name for the spawned agent'),
    team_name: z.string().optional().describe('Team name for spawning'),
    mode: permissionModeSchema().optional().describe('Permission mode'),
  });
  
  return baseInputSchema().merge(multiAgentInputSchema).extend({
    isolation: z.enum(['worktree', 'remote']).optional().describe('Isolation mode'),
    cwd: z.string().optional().describe('Working directory override'),
  });
});
```

### 5.3 任务委派机制

```typescript
// tools/AgentTool/runAgent.ts
export async function runAgent(
  input: AgentInput,
  context: ToolUseContext,
  canUseTool: CanUseToolFn,
): Promise<ToolResult<AgentOutput>> {
  // 1. 创建 Agent ID
  const agentId = createAgentId();
  
  // 2. 选择执行模式
  if (input.isolation === 'worktree') {
    // 工作树隔离模式
    const worktree = await createAgentWorktree(agentId, input.cwd);
    return runInWorktree(agentId, input, worktree, context);
  }
  
  if (input.run_in_background) {
    // 后台执行模式
    return runInBackground(agentId, input, context);
  }
  
  // 3. 前台执行
  return runInForeground(agentId, input, context);
}
```

### 5.4 工作树隔离

```typescript
// utils/worktree.ts
export async function createAgentWorktree(
  agentId: AgentId,
  baseBranch?: string,
): Promise<WorktreeInfo> {
  const gitRoot = await findGitRoot();
  const worktreePath = join(gitRoot, '.claude', 'worktrees', agentId);
  
  // 创建 Git 工作树
  await exec(`git worktree add ${worktreePath} ${baseBranch || 'HEAD'}`);
  
  return {
    path: worktreePath,
    agentId,
    createdAt: Date.now(),
  };
}

export async function removeAgentWorktree(
  worktree: WorktreeInfo,
): Promise<void> {
  // 清理工作树
  await exec(`git worktree remove ${worktree.path} --force`);
}
```

---

## 六、扩展性设计

### 6.1 Hooks 系统

Claude Code 实现了 **17 种 Hook 事件**，支持深度定制：

```
┌─────────────────────────────────────────────────────────────┐
│                      Hook Events                             │
├─────────────────────────────────────────────────────────────┤
│  Tool Lifecycle                                             │
│  ├── PreToolUse      (工具执行前)                           │
│  ├── PostToolUse     (工具执行后)                           │
│  └── PostToolUseFailure (工具执行失败)                      │
├─────────────────────────────────────────────────────────────┤
│  Session Lifecycle                                          │
│  ├── SessionStart    (会话开始)                             │
│  ├── Stop            (响应结束)                             │
│  └── StopFailure     (响应失败)                             │
├─────────────────────────────────────────────────────────────┤
│  User Interaction                                           │
│  ├── UserPromptSubmit (用户提交输入)                        │
│  ├── Notification     (通知发送)                            │
│  └── PermissionDenied (权限拒绝)                            │
├─────────────────────────────────────────────────────────────┤
│  Compact & Resume                                           │
│  ├── PreCompact       (压缩前)                              │
│  └── PostCompact      (压缩后)                              │
└─────────────────────────────────────────────────────────────┘
```

### 6.2 Hook 配置示例

```typescript
// utils/hooks/hooksConfigManager.ts
export const getHookEventMetadata = memoize(
  function (toolNames: string[]): Record<HookEvent, HookEventMetadata> {
    return {
      PreToolUse: {
        summary: 'Before tool execution',
        description: `Input to command is JSON of tool call arguments.
Exit code 0 - stdout/stderr not shown
Exit code 2 - show stderr to model and block tool call
Other exit codes - show stderr to user only but continue`,
        matcherMetadata: {
          fieldToMatch: 'tool_name',
          values: toolNames,
        },
      },
      PostToolUse: {
        summary: 'After tool execution',
        description: `Input to command is JSON with fields "inputs" and "response".
Exit code 0 - stdout shown in transcript mode (ctrl+o)
Exit code 2 - show stderr to model immediately
Other exit codes - show stderr to user only`,
        matcherMetadata: {
          fieldToMatch: 'tool_name',
          values: toolNames,
        },
      },
      UserPromptSubmit: {
        summary: 'When the user submits a prompt',
        description: `Input to command is JSON with original user prompt text.
Exit code 0 - stdout shown to Claude
Exit code 2 - block processing, erase original prompt
Other exit codes - show stderr to user only`,
      },
      // ... 更多事件
    };
  },
);
```

### 6.3 插件系统

```typescript
// plugins/bundled/index.ts
export function initBuiltinPlugins(): void {
  // 加载内置插件
  const plugins = getBuiltinPlugins();
  
  for (const plugin of plugins) {
    registerPlugin({
      name: plugin.name,
      version: plugin.version,
      tools: plugin.tools,
      commands: plugin.commands,
      hooks: plugin.hooks,
    });
  }
}

// 插件生命周期
export type Plugin = {
  name: string;
  version: string;
  tools?: ToolDefinition[];
  commands?: CommandDefinition[];
  hooks?: HookDefinition[];
  init?(): Promise<void>;
  destroy?(): Promise<void>;
};
```

### 6.4 MCP 协议集成

```
┌─────────────────────────────────────────────────────────────┐
│                    MCP Integration                           │
├─────────────────────────────────────────────────────────────┤
│  Claude Code                                                │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                  MCP Client                         │   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐            │   │
│  │  │  Auth   │  │  Tools  │  │Resources│            │   │
│  │  └─────────┘  └─────────┘  └─────────┘            │   │
│  └─────────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────────┤
│  MCP Servers                                                │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │   Server 1  │  │   Server 2  │  │   Server N  │        │
│  │  (本地/远程) │  │  (本地/远程) │  │  (本地/远程) │        │
│  └─────────────┘  └─────────────┘  └─────────────┘        │
└─────────────────────────────────────────────────────────────┘
```

```typescript
// services/mcp/client.ts
export async function getMcpToolsCommandsAndResources(
  connections: MCPServerConnection[],
): Promise<McpToolsAndResources> {
  const tools: MCPTool[] = [];
  const resources: ServerResource[] = [];
  
  for (const connection of connections) {
    // 获取服务器能力
    const capabilities = await connection.getServerCapabilities();
    
    // 加载工具
    if (capabilities.tools) {
      const serverTools = await connection.listTools();
      tools.push(...serverTools.map(tool => ({
        ...tool,
        serverName: connection.name,
      })));
    }
    
    // 加载资源
    if (capabilities.resources) {
      const serverResources = await connection.listResources();
      resources.push(...serverResources);
    }
  }
  
  return { tools, resources };
}
```

---

## 七、上下文与记忆系统

### 7.1 上下文窗口管理

Claude Code 支持从 200K 到 1M token 的动态上下文窗口：

```typescript
// utils/context.ts
export const MODEL_CONTEXT_WINDOW_DEFAULT = 200_000;
export const COMPACT_MAX_OUTPUT_TOKENS = 20_000;
export const MAX_OUTPUT_TOKENS_DEFAULT = 32_000;
export const MAX_OUTPUT_TOKENS_UPPER_LIMIT = 64_000;

export function getContextWindowForModel(
  model: string,
  betas?: string[],
): number {
  // 1. 环境变量覆盖（ANT-ONLY）
  if (process.env.USER_TYPE === 'ant' && 
      process.env.CLAUDE_CODE_MAX_CONTEXT_TOKENS) {
    const override = parseInt(process.env.CLAUDE_CODE_MAX_CONTEXT_TOKENS, 10);
    if (!isNaN(override) && override > 0) {
      return override;
    }
  }
  
  // 2. [1m] 后缀 - 显式 opt-in
  if (has1mContext(model)) {
    return 1_000_000;
  }
  
  // 3. 模型能力检查
  const cap = getModelCapability(model);
  if (cap?.max_input_tokens && cap.max_input_tokens >= 100_000) {
    return cap.max_input_tokens;
  }
  
  // 4. Beta 标签检查
  if (betas?.includes(CONTEXT_1M_BETA_HEADER) && modelSupports1M(model)) {
    return 1_000_000;
  }
  
  // 5. 默认值
  return MODEL_CONTEXT_WINDOW_DEFAULT;
}
```

### 7.2 长期记忆系统

Claude Code 实现了基于文件的长期记忆系统：

```
~/.claude/
├── MEMORY.md                    # 全局记忆（自动维护）
├── projects/
│   └── {project-hash}/
│       ├── MEMORY.md            # 项目记忆
│       └── memory/
│           ├── preferences.md   # 偏好设置
│           └── learnings.md     # 学习记录
└── team-memory/                 # 团队记忆
    ├── patterns.md
    └── conventions.md
```

```typescript
// memdir/memdir.ts
export const ENTRYPOINT_NAME = 'MEMORY.md';
export const MAX_ENTRYPOINT_LINES = 200;
export const MAX_ENTRYPOINT_BYTES = 25_000;

export function truncateEntrypointContent(raw: string): EntrypointTruncation {
  const trimmed = raw.trim();
  const contentLines = trimmed.split('\n');
  const lineCount = contentLines.length;
  const byteCount = trimmed.length;
  
  const wasLineTruncated = lineCount > MAX_ENTRYPOINT_LINES;
  const wasByteTruncated = byteCount > MAX_ENTRYPOINT_BYTES;
  
  if (!wasLineTruncated && !wasByteTruncated) {
    return {
      content: trimmed,
      lineCount,
      byteCount,
      wasLineTruncated,
      wasByteTruncated,
    };
  }
  
  // 智能截断
  let truncated = wasLineTruncated
    ? contentLines.slice(0, MAX_ENTRYPOINT_LINES).join('\n')
    : trimmed;
  
  if (truncated.length > MAX_ENTRYPOINT_BYTES) {
    const cutAt = truncated.lastIndexOf('\n', MAX_ENTRYPOINT_BYTES);
    truncated = truncated.slice(0, cutAt > 0 ? cutAt : MAX_ENTRYPOINT_BYTES);
  }
  
  // 添加警告
  const reason = wasByteTruncated && !wasLineTruncated
    ? `${formatFileSize(byteCount)} (limit: ${formatFileSize(MAX_ENTRYPOINT_BYTES)})`
    : wasLineTruncated && !wasByteTruncated
      ? `${lineCount} lines (limit: ${MAX_ENTRYPOINT_LINES})`
      : `${lineCount} lines and ${formatFileSize(byteCount)}`;
  
  return {
    content: truncated + `\n\n> WARNING: ${ENTRYPOINT_NAME} is ${reason}.`,
    lineCount,
    byteCount,
    wasLineTruncated,
    wasByteTruncated,
  };
}
```

### 7.3 Token 预算管理

```typescript
// utils/tokenBudget.ts
export type TokenBudget = {
  totalBudget: number;
  systemPrompt: number;
  messages: number;
  tools: number;
  output: number;
  buffer: number;
};

export function calculateTokenBudget(
  contextWindow: number,
  maxOutputTokens: number,
): TokenBudget {
  const totalBudget = contextWindow;
  const buffer = Math.floor(totalBudget * 0.05);  // 5% 缓冲
  
  return {
    totalBudget,
    systemPrompt: Math.floor(totalBudget * 0.15),  // 15% 系统提示
    messages: Math.floor(totalBudget * 0.50),       // 50% 消息历史
    tools: Math.floor(totalBudget * 0.20),          // 20% 工具定义
    output: maxOutputTokens,
    buffer,
  };
}
```

---

## 八、性能优化策略

### 8.1 启动优化

```typescript
// main.tsx - 启动时并行预取
export function startDeferredPrefetches(): void {
  // 跳过 bare 模式
  if (isBareMode()) return;
  
  // 并行启动所有预取任务
  void initUser();                    // 用户信息
  void getUserContext();              // 用户上下文
  prefetchSystemContextIfSafe();      // 系统上下文（Git 状态）
  void getRelevantTips();             // 提示信息
  
  // 云服务凭证预取
  if (isEnvTruthy(process.env.CLAUDE_CODE_USE_BEDROCK)) {
    void prefetchAwsCredentialsAndBedRockInfoIfSafe();
  }
  if (isEnvTruthy(process.env.CLAUDE_CODE_USE_VERTEX)) {
    void prefetchGcpCredentialsIfSafe();
  }
  
  // 文件统计
  void countFilesRoundedRg(getCwd(), AbortSignal.timeout(3000), []);
  
  // 分析和特性标志
  void initializeAnalyticsGates();
  void prefetchOfficialMcpUrls();
  void refreshModelCapabilities();
  
  // 文件变更检测器
  void settingsChangeDetector.initialize();
  void skillChangeDetector.initialize();
}
```

### 8.2 运行时优化

```typescript
// 工具函数 - 记忆化
export function memoize<T extends (...args: any[]) => any>(
  fn: T,
  resolver?: (...args: Parameters<T>) => string,
): T {
  const cache = new Map<string, ReturnType<T>>();
  
  return ((...args: Parameters<T>) => {
    const key = resolver ? resolver(...args) : JSON.stringify(args);
    
    if (cache.has(key)) {
      return cache.get(key);
    }
    
    const result = fn(...args);
    cache.set(key, result);
    return result;
  }) as T;
}

// 延迟 Schema 加载
export function lazySchema<T>(factory: () => T): () => T {
  let instance: T | undefined;
  
  return () => {
    if (!instance) {
      instance = factory();
    }
    return instance;
  };
}
```

### 8.3 缓存策略

```
┌─────────────────────────────────────────────────────────────┐
│                      Caching Layers                         │
├─────────────────────────────────────────────────────────────┤
│  Level 1: In-Memory Cache                                   │
│  ├── FileStateCache (文件状态)                              │
│  ├── CompletionCache (补全结果)                             │
│  └── ModelCapabilityCache (模型能力)                        │
├─────────────────────────────────────────────────────────────┤
│  Level 2: Disk Cache                                        │
│  ├── Plugin Cache (~/.claude/plugins/cache/)                │
│  ├── Session Storage (~/.claude/projects/)                  │
│  └── Tool Results (~/.claude/tool-results/)                 │
├─────────────────────────────────────────────────────────────┤
│  Level 3: API Cache                                         │
│  ├── Prompt Cache (API 提示缓存)                            │
│  └── Model Strings (模型名称缓存)                           │
└─────────────────────────────────────────────────────────────┘
```

---

## 九、UI/UX 架构

### 9.1 终端 UI 框架

Claude Code 使用 **Ink**（React for CLI）构建终端界面：

```typescript
// components/App.tsx
export function App({
  getFpsMetrics,
  stats,
  initialState,
  children,
}: Props): React.ReactElement {
  return (
    <FpsMetricsProvider getFpsMetrics={getFpsMetrics}>
      <StatsProvider store={stats}>
        <AppStateProvider
          initialState={initialState}
          onChangeAppState={onChangeAppState}
        >
          {children}
        </AppStateProvider>
      </StatsProvider>
    </FpsMetricsProvider>
  );
}
```

### 9.2 组件层次

```
┌─────────────────────────────────────────────────────────────┐
│                        App (根组件)                          │
├─────────────────────────────────────────────────────────────┤
│  Providers                                                  │
│  ├── FpsMetricsProvider (性能指标)                          │
│  ├── StatsProvider (统计数据)                               │
│  ├── AppStateProvider (应用状态)                            │
│  ├── MailboxProvider (消息队列)                             │
│  └── VoiceProvider (语音集成)                               │
├─────────────────────────────────────────────────────────────┤
│  Screens                                                    │
│  ├── REPL (交互式界面)                                      │
│  ├── Doctor (诊断工具)                                      │
│  └── ResumeConversation (恢复会话)                          │
├─────────────────────────────────────────────────────────────┤
│  Components (144 个)                                        │
│  ├── PromptInput (输入组件)                                 │
│  ├── Messages (消息列表)                                    │
│  ├── ToolUseLoader (工具使用加载)                           │
│  ├── PermissionRequest (权限请求)                           │
│  └── ...                                                    │
└─────────────────────────────────────────────────────────────┘
```

### 9.3 虚拟消息列表

```typescript
// components/VirtualMessageList.tsx
export function VirtualMessageList({
  messages,
  jumpHandle,
  onScroll,
}: VirtualMessageListProps) {
  // 虚拟滚动 - 只渲染可见消息
  const visibleRange = useMemo(() => {
    return calculateVisibleRange(
      scrollTop,
      viewportHeight,
      itemHeights,
    );
  }, [scrollTop, viewportHeight, itemHeights]);
  
  return (
    <Box flexDirection="column">
      {messages.slice(visibleRange.start, visibleRange.end).map(msg => (
        <MessageRow key={msg.id} message={msg} />
      ))}
    </Box>
  );
}
```

---

## 十、企业级特性

### 10.1 策略管理

```typescript
// services/policyLimits/index.ts
export type PolicyLimits = {
  // 成本限制
  maxCostPerSession?: number;
  maxCostPerDay?: number;
  
  // 使用限制
  maxSessionsPerDay?: number;
  maxTokensPerSession?: number;
  
  // 功能限制
  allowedModels?: string[];
  blockedTools?: string[];
  
  // 沙箱策略
  sandbox?: SandboxPolicy;
};

export async function loadPolicyLimits(): Promise<void> {
  // 从远程加载策略
  const remoteLimits = await fetchRemotePolicyLimits();
  
  // 合并本地策略
  const localLimits = getLocalPolicyLimits();
  
  // 应用策略
  applyPolicyLimits(mergeLimits(remoteLimits, localLimits));
}
```

### 10.2 远程管理设置

```typescript
// services/remoteManagedSettings/index.ts
export async function loadRemoteManagedSettings(): Promise<void> {
  // 企业客户远程配置
  const settings = await fetchRemoteManagedSettings({
    organizationId: getOrganizationId(),
    userId: getUserId(),
  });
  
  if (settings) {
    // 应用远程设置
    applyRemoteSettings(settings);
    
    // 注册热更新
    registerSettingsHotReload(settings.version);
  }
}
```

### 10.3 分析与遥测

```typescript
// services/analytics/index.ts
export type AnalyticsMetadata = {
  event_name: string;
  timestamp: number;
  session_id: string;
  user_id?: string;
  organization_id?: string;
};

export function logEvent(
  eventName: string,
  metadata: Record<string, unknown>,
): void {
  // 脱敏处理
  const sanitized = sanitizeMetadata(metadata);
  
  // 发送遥测
  enqueueTelemetryEvent({
    event_name: eventName,
    timestamp: Date.now(),
    session_id: getSessionId(),
    ...sanitized,
  });
}
```

---

## 十一、API 服务层深度分析

### 11.1 API 调用架构图

```
                          调用方/CLI UI
                               |
                               v
                      Claude Code API 服务层
            (claude.ts / client.ts / withRetry.ts / errors.ts)
                               |
                               | getAnthropicClient() 构造统一客户端
                               v
                        Anthropic SDK 客户端
                               |
          +--------------------+--------------------+
          |                    |                    |
          v                    v                    v
       Bedrock             Foundry/Azure        Vertex/Google
       (AWS SDK)            (Foundry SDK)        (Vertex SDK)
          |                    |                    |
          +--------------------+--------------------+
                               |
                               v
                        远端后端服务 (Claude API)
                               |
                               v
                          响应回传 + 日志/指标记录
```

### 11.2 核心 API 客户端设计

```typescript
// services/api/client.ts - 统一客户端构造
export async function getAnthropicClient(options: {
  apiKey: string;
  maxRetries: number;
  model: string;
  fetchOverride?: typeof fetch;
  source: string;
}): Promise<Anthropic> {
  // 根据部署场景切换客户端
  if (isBedrockMode()) {
    return createBedrockClient(options);
  }
  if (isVertexMode()) {
    return createVertexClient(options);
  }
  if (isFoundryMode()) {
    return createFoundryClient(options);
  }
  // 默认 First-Party API
  return createAnthropicClient(options);
}
```

### 11.3 流式/非流式调用路径

```typescript
// services/api/claude.ts - 流式调用
export async function* queryModelWithStreaming({
  messages,
  systemPrompt,
  thinkingConfig,
  tools,
  signal,
  options,
}: QueryOptions): AsyncGenerator<
  StreamEvent | AssistantMessage | SystemAPIErrorMessage,
  void
> {
  return yield* withStreamingVCR(messages, async function* () {
    yield* queryModel(
      messages,
      systemPrompt,
      thinkingConfig,
      tools,
      signal,
      options,
    );
  });
}

// 调用示例
for await (const event of queryModelWithStreaming(options)) {
  if (event.type === 'assistant') {
    // 处理助手输出
  } else if (isSystemEvent(event)) {
    // 处理系统事件
  }
}
```

### 11.4 错误处理机制

```typescript
// services/api/errors.ts - 错误分类与映射
export type ErrorClassification =
  | 'aborted'
  | 'api_timeout'
  | 'rate_limit'
  | 'server_error'
  | 'client_error'
  | 'unknown';

export function classifyAPIError(error: Error): ErrorClassification {
  if (error instanceof APIUserAbortError) return 'aborted';
  if (error instanceof APIConnectionTimeoutError) return 'api_timeout';
  if (isRateLimitError(error)) return 'rate_limit';
  if (isServerError(error)) return 'server_error';
  if (isClientError(error)) return 'client_error';
  return 'unknown';
}

// 错误转用户可读消息
export function getAssistantMessageFromError(
  error: Error,
  model: string,
  options?: ErrorOptions,
): AssistantMessage {
  const classification = classifyAPIError(error);
  return createAssistantAPIErrorMessage({
    content: formatErrorMessage(error, classification),
    error: classification,
    model,
  });
}
```

### 11.5 重试策略

```typescript
// services/api/withRetry.ts - 重试机制
export async function* withRetry<T>(
  clientFactory: () => Promise<Anthropic>,
  action: (client: Anthropic, attempt: number, ctx: RetryContext) => Promise<T>,
  options: RetryOptions,
): AsyncGenerator<SystemAPIErrorMessage, T> {
  const { maxRetries, model, fallbackModel, signal } = options;
  
  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      const client = await clientFactory();
      return await action(client, attempt, { signal });
    } catch (error) {
      // 529/429 容量错误 - 重试
      if (isCapacityError(error) && attempt < maxRetries) {
        yield createRetryMessage(error, attempt);
        await sleep(calculateBackoff(attempt));
        continue;
      }
      
      // 401/403 鉴权错误 - 刷新令牌
      if (isAuthError(error)) {
        await refreshAuthToken();
        continue;
      }
      
      // 连续 529 - 触发降级
      if (shouldFallback(error, attempt, fallbackModel)) {
        throw new FallbackTriggeredError(fallbackModel!);
      }
      
      throw new CannotRetryError(error);
    }
  }
}
```

### 11.6 Prompt 缓存机制

```typescript
// services/api/promptCacheBreakDetection.ts
export function getPromptCachingEnabled(model: string): boolean {
  // 全局禁用检查
  if (isEnvTruthy(process.env.CLAUDE_CODE_DISABLE_PROMPT_CACHING)) {
    return false;
  }
  
  // 模型能力检查
  const capability = getModelCapability(model);
  return capability?.supportsPromptCaching ?? false;
}

// 缓存控制对象
export function getCacheControl(options: {
  scope: CacheScope;
  querySource: QuerySource;
}): CacheControl {
  const enableCaching = getPromptCachingEnabled(options.scope.model);
  
  if (!enableCaching) return { type: 'none' };
  
  return {
    type: 'ephemeral',
    ttl: shouldUseOneHourTTL(options) ? '1h' : '5m',
    scope: options.scope,
  };
}

// 缓存断路检测
export function checkResponseForCacheBreak(
  querySource: string,
  cacheReadTokens: number,
  cacheCreationTokens: number,
  messages: Message[],
  agentId?: string,
  requestId?: string,
): void {
  const prev = getLastPromptState();
  const current = buildCurrentState(messages);
  
  if (detectCacheBreak(prev, current, cacheReadTokens)) {
    logCacheBreakEvent({
      reason: diffStates(prev, current),
      querySource,
      agentId,
      requestId,
    });
    writeCacheBreakDiff(prev, current);
  }
}
```

---

## 十二、全局状态机分析

### 12.1 全局状态架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                    STATE (全局单例状态)                          │
├─────────────────────────────────────────────────────────────────┤
│  会话与项目                                                     │
│  ├── originalCwd, projectRoot, cwd (工作目录)                   │
│  ├── sessionId, sessionProjectDir (会话标识)                    │
│  └── parentSessionId, sessionSource (会话关系)                  │
├─────────────────────────────────────────────────────────────────┤
│  成本与模型                                                     │
│  ├── totalCostUSD, totalAPIDuration (累计成本)                  │
│  ├── modelUsage: { [model]: ModelUsage } (模型使用统计)         │
│  └── initialMainLoopModel, mainLoopModelOverride (模型配置)     │
├─────────────────────────────────────────────────────────────────┤
│  运行时信息                                                     │
│  ├── startTime, lastInteractionTime (时间追踪)                  │
│  ├── turnToolDurationMs, turnHookDurationMs (耗时统计)          │
│  └── promptId, lastEmittedDate (提示追踪)                       │
├─────────────────────────────────────────────────────────────────┤
│  IO/UI 设置                                                     │
│  ├── isInteractive, clientType (交互模式)                       │
│  ├── questionPreviewFormat (输出格式)                           │
│  └── flagSettingsPath, chromeFlagOverride (配置路径)            │
├─────────────────────────────────────────────────────────────────┤
│  指标与遥测                                                     │
│  ├── meter, sessionCounter, costCounter (OpenTelemetry)         │
│  ├── loggerProvider, eventLogger, tracerProvider (日志)         │
│  └── statsStore, agentColorMap (统计与颜色)                     │
├─────────────────────────────────────────────────────────────────┤
│  缓存与扩展                                                     │
│  ├── planSlugCache, systemPromptSectionCache (缓存)             │
│  ├── invokedSkills, inlinePlugins (技能/插件)                   │
│  └── sdkBetas, allowedChannels (Beta 与通道)                   │
└─────────────────────────────────────────────────────────────────┘
```

### 12.2 关键状态变量详解

```typescript
// bootstrap/state.ts - 状态定义
type State = {
  // === 会话与项目 ===
  originalCwd: string;           // 原始工作目录
  projectRoot: string;           // 稳定项目根目录（用于身份识别）
  cwd: string;                   // 当前工作目录
  sessionId: SessionId;          // 当前会话 ID (UUID)
  sessionProjectDir: string | null;  // 会话所属项目目录
  
  // === 成本与模型 ===
  totalCostUSD: number;          // 累计成本
  totalAPIDuration: number;      // API 调用总时长
  totalToolDuration: number;     // 工具执行总时长
  modelUsage: { [model: string]: ModelUsage };  // 模型使用统计
  
  // === 运行时 ===
  startTime: number;             // 会话开始时间
  lastInteractionTime: number;   // 最后交互时间
  turnToolDurationMs: number;    // 当轮工具耗时
  turnHookDurationMs: number;    // 当轮 Hook 耗时
  
  // === UI/配置 ===
  isInteractive: boolean;        // 是否交互模式
  clientType: string;            // 客户端类型
  questionPreviewFormat: string; // 问题预览格式
  
  // === 缓存 ===
  planSlugCache: Map<string, string>;           // Plan slug 缓存
  systemPromptSectionCache: Map<string, string | null>;  // 系统提示缓存
  invokedSkills: Map<string, InvokedSkill>;     // 已调用技能
};
```

### 12.3 状态初始化流程

```typescript
// bootstrap/state.ts - 初始化
function getInitialState(): State {
  // 1. 解析并标准化工作目录
  const rawCwd = cwd();
  let resolvedCwd: string;
  try {
    resolvedCwd = realpathSync(rawCwd).normalize('NFC');
  } catch {
    resolvedCwd = rawCwd.normalize('NFC');
  }
  
  // 2. 构建初始状态
  return {
    originalCwd: resolvedCwd,
    projectRoot: resolvedCwd,
    cwd: resolvedCwd,
    sessionId: randomUUID() as SessionId,
    
    totalCostUSD: 0,
    totalAPIDuration: 0,
    totalToolDuration: 0,
    modelUsage: {},
    
    startTime: Date.now(),
    lastInteractionTime: Date.now(),
    
    isInteractive: process.stdout.isTTY,
    clientType: 'cli',
    
    planSlugCache: new Map(),
    systemPromptSectionCache: new Map(),
    invokedSkills: new Map(),
    
    // ... 其他字段
  };
}

// 全局单例
const STATE: State = getInitialState();
```

### 12.4 状态变更模式

```typescript
// 成本累积 - 原子性多字段更新
export function addToTotalCostState(
  cost: number,
  modelUsage: ModelUsage,
  model: string,
): void {
  STATE.modelUsage[model] = modelUsage;
  STATE.totalCostUSD += cost;
}

// 会话切换 - 原子性切换 + 清理 + 通知
export function switchSession(
  sessionId: SessionId,
  projectDir: string | null,
): void {
  STATE.planSlugCache.delete(STATE.sessionId);
  STATE.sessionId = sessionId;
  STATE.sessionProjectDir = projectDir;
  sessionSwitched.emit(sessionId);
}

// 延迟刷新 - 批量更新策略
export function updateLastInteractionTime(immediate?: boolean): void {
  if (immediate) {
    STATE.lastInteractionTime = Date.now();
    flushInteractionTime();
  } else {
    interactionTimeDirty = true;
  }
}
```

---

## 十三、任务调度系统

### 13.1 任务调度架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                    Task Framework (任务框架)                     │
├─────────────────────────────────────────────────────────────────┤
│  AppState.tasks (全局任务状态字典)                               │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  registerTask()  →  updateTaskState()  →  evictTask()    │  │
│  └───────────────────────────────────────────────────────────┘  │
├─────────────────────────────────────────────────────────────────┤
│  任务类型                                                       │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐            │
│  │ LocalShell  │  │ LocalAgent  │  │ RemoteAgent │            │
│  │ (Bash执行)  │  │ (本地代理)  │  │ (远程代理)  │            │
│  └─────────────┘  └─────────────┘  └─────────────┘            │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐            │
│  │ InProcess   │  │  DreamTask  │  │  Monitor    │            │
│  │ (进程内)    │  │  (梦境UI)   │  │  (监控)     │            │
│  └─────────────┘  └─────────────┘  └─────────────┘            │
├─────────────────────────────────────────────────────────────────┤
│  生命周期                                                       │
│  pending → running → completed/failed/killed                   │
├─────────────────────────────────────────────────────────────────┤
│  前台/后台切换                                                  │
│  ┌─────────────┐  ← isBackgrounded →  ┌─────────────┐         │
│  │  Foreground │  ←── background() ── │  Background │         │
│  │  (实时显示) │  ── foreground() ──→  │  (静默执行) │         │
│  └─────────────┘                      └─────────────┘         │
├─────────────────────────────────────────────────────────────────┤
│  并发控制                                                       │
│  ├── AbortController (取消信号)                                 │
│  ├── Stall Watchdog (阻塞检测)                                  │
│  └── Polling + Event-driven (轮询 + 事件驱动)                  │
└─────────────────────────────────────────────────────────────────┘
```

### 13.2 任务类型分类

| 任务类型 | 实现文件 | 用途 |
|---------|---------|------|
| `local_bash` | LocalShellTask.tsx | Bash 命令执行与监控 |
| `local_agent` | LocalAgentTask.tsx | 本地 Agent 后台执行 |
| `remote_agent` | RemoteAgentTask.tsx | 远程 CCR 任务轮询 |
| `in_process_teammate` | InProcessTeammateTask.tsx | 进程内队友协作 |
| `dream` | DreamTask.ts | UI 梦境任务展示 |

### 13.3 任务生命周期管理

```typescript
// tasks/types.ts - 任务状态
export type TaskStatus = 'pending' | 'running' | 'completed' | 'failed' | 'killed';

export type TaskState = {
  taskId: string;
  status: TaskStatus;
  startTime: number;
  endTime?: number;
  notified: boolean;        // 防重复通知
  isBackgrounded: boolean;  // 前后台标记
  progress?: AgentProgress; // 进度追踪
  error?: string;           // 错误信息
};

// 任务注册
export function registerTask(task: TaskState): void {
  AppState.tasks[task.taskId] = task;
}

// 任务状态更新
export function updateTaskState(
  taskId: string,
  updater: (prev: TaskState) => TaskState,
): void {
  const prev = AppState.tasks[taskId];
  if (prev) {
    AppState.tasks[taskId] = updater(prev);
  }
}

// 任务清理
export function evictTask(taskId: string): void {
  delete AppState.tasks[taskId];
}
```

### 13.4 并发控制机制

```typescript
// AbortController 取消信号
export async function spawnShellTask(
  command: string,
  options: SpawnOptions,
  parentAbortController?: AbortController,
): Promise<TaskResult> {
  const abortController = new AbortController();
  
  // 父取消联动
  if (parentAbortController) {
    parentAbortController.signal.addEventListener('abort', () => {
      abortController.abort();
    });
  }
  
  // 执行任务
  const process = spawn(command, { signal: abortController.signal });
  // ...
}

// Stall 阻塞检测
const STALL_WATCHDOG_INTERVAL = 5000;  // 5 秒检查
const STALL_THRESHOLD = 30000;         // 30 秒无输出视为阻塞

function startStallWatchdog(taskId: string): void {
  setInterval(() => {
    const task = getTask(taskId);
    const outputSize = getOutputSize(taskId);
    
    if (outputSize === lastOutputSize[taskId]) {
      stallDuration[taskId] += STALL_WATCHDOG_INTERVAL;
      
      if (stallDuration[taskId] >= STALL_THRESHOLD) {
        // 发送阻塞通知
        notifyStall(taskId, stallDuration[taskId]);
      }
    } else {
      stallDuration[taskId] = 0;
      lastOutputSize[taskId] = outputSize;
    }
  }, STALL_WATCHDOG_INTERVAL);
}
```

### 13.5 前台/后台任务切换

```typescript
// 前台 → 后台
export function backgroundTask(taskId: string): void {
  updateTaskState(taskId, (prev) => ({
    ...prev,
    isBackgrounded: true,
  }));
  
  // 启动后台监控
  startBackgroundMonitoring(taskId);
}

// 后台 → 前台
export function foregroundTask(taskId: string): void {
  updateTaskState(taskId, (prev) => ({
    ...prev,
    isBackgrounded: false,
  }));
  
  // 回显输出到当前 UI
  replayTaskOutput(taskId);
}

// 自动后台化（超时后）
const AUTO_BACKGROUND_MS = 120_000;  // 2 分钟

export function registerForeground(task: TaskState): void {
  registerTask(task);
  
  setTimeout(() => {
    if (getTask(task.taskId)?.status === 'running') {
      backgroundTask(task.taskId);
    }
  }, AUTO_BACKGROUND_MS);
}
```

---

## 十四、远程连接与通信

### 14.1 远程连接架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                    Claude Code Frontend (CLI/UI)                 │
├─────────────────────────────────────────────────────────────────┤
│                          │ cc:// 协议                           │
│                          v                                      │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              Direct Connect Server                       │   │
│  │  POST /sessions → { sessionId, wsUrl, authToken }       │   │
│  └─────────────────────────────────────────────────────────┘   │
│                          │                                      │
│                          v                                      │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              WebSocket Connection                         │   │
│  │  wss://api.anthropic.com/v1/sessions/ws/{id}/subscribe  │   │
│  └─────────────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────────────┤
│                         Bridge Layer                            │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  - 环境注册与会话管理                                     │   │
│  │  - 令牌轮换与心跳                                        │   │
│  │  - 多会话调度                                            │   │
│  └─────────────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────────────┤
│                      Remote CCR Layer                           │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  RemoteSessionManager                                   │   │
│  │  ├── SessionsWebSocket (订阅与认证)                      │   │
│  │  ├── handleMessage (消息分发)                            │   │
│  │  ├── handleControlRequest (权限请求)                     │   │
│  │  └── sdkMessageAdapter (消息转译)                        │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### 14.2 直接连接管理 (cc://)

```typescript
// server/directConnectManager.ts
export type DirectConnectConfig = {
  serverUrl: string;   // REST API 根 URL
  sessionId: string;   // 会话 ID
  wsUrl: string;       // WebSocket 端点
  authToken: string;   // 认证令牌
};

export class DirectConnectSessionManager {
  private ws: WebSocket | null = null;
  
  async connect(): Promise<void> {
    // 建立 WebSocket 连接
    this.ws = new WebSocket(this.config.wsUrl, {
      headers: {
        Authorization: `Bearer ${this.config.authToken}`,
      },
    });
    
    this.ws.on('open', () => this.callbacks.onConnected());
    this.ws.on('message', (data) => this.handleMessage(data));
    this.ws.on('close', () => this.callbacks.onDisconnected());
    this.ws.on('error', (err) => this.callbacks.onError(err));
  }
  
  private handleMessage(data: string): void {
    const message = JSON.parse(data);
    
    // 控制请求 - 权限
    if (message.type === 'control_request') {
      if (message.request?.subtype === 'can_use_tool') {
        this.callbacks.onPermissionRequest(
          message.request,
          message.request_id,
        );
      }
      return;
    }
    
    // SDK 消息 - 转发给上层
    this.callbacks.onMessage(message);
  }
  
  // 发送用户消息
  sendMessage(content: string): void {
    this.ws?.send(JSON.stringify({
      type: 'user',
      message: { role: 'user', content },
    }));
  }
  
  // 响应权限请求
  respondToPermissionRequest(
    requestId: string,
    result: PermissionResult,
  ): void {
    this.ws?.send(JSON.stringify({
      type: 'control_response',
      subtype: 'success',
      request_id: requestId,
      response: result,
    }));
  }
}
```

### 14.3 WebSocket 通信协议

```typescript
// remote/SessionsWebSocket.ts
export class SessionsWebSocket {
  // 连接 URL
  // wss://api.anthropic.com/v1/sessions/ws/{sessionId}/subscribe
  //   ?organization_uuid={orgUuid}
  
  // 认证消息
  // { type: 'auth', credential: { type: 'oauth', token: '...' } }
  
  // 消息类型
  // - SDKMessage: 模型输出
  // - control_request: 权限/中断请求
  // - control_response: 权限响应
  // - control_cancel_request: 取消请求
  // - keep_alive: 心跳
  // - auth_status: 认证状态
  
  // 重连策略
  private reconnect(): void {
    if (this.permanentCloseCodes.has(this.lastCloseCode)) {
      return;  // 永久关闭，不重连
    }
    
    const delay = Math.min(
      this.baseDelay * Math.pow(2, this.reconnectAttempts),
      this.maxDelay,
    );
    
    setTimeout(() => this.connect(), delay);
    this.reconnectAttempts++;
  }
}
```

### 14.4 权限桥接机制

```typescript
// remote/remotePermissionBridge.ts
export function createSyntheticAssistantMessage(
  request: SDKControlPermissionRequest,
  requestId: string,
): AssistantMessage {
  return {
    type: 'assistant',
    message: {
      role: 'assistant',
      content: [{
        type: 'tool_use',
        id: request.tool_use_id,
        name: request.tool_name,
        input: request.input,
      }],
    },
    // 用于权限对话展示
  };
}

// 工具占位符（本地无实现时）
export function createToolStub(toolName: string): Tool {
  return {
    name: toolName,
    renderToolUseMessage: (input) => ({
      jsx: <Text>Tool: {toolName}</Text>,
    }),
    // 最小实现，仅用于权限对话
  };
}
```

---

## 十五、AI 伴侣系统 (Buddy)

### 15.1 Buddy 系统架构

```
┌─────────────────────────────────────────────────────────────────┐
│                    Buddy AI Companion System                     │
├─────────────────────────────────────────────────────────────────┤
│  数据模型                                                       │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐            │
│  │ Companion   │  │   Bones     │  │    Soul     │            │
│  │ (完整伴侣)  │  │ (外观骨架)  │  │ (个性灵魂)  │            │
│  └─────────────┘  └─────────────┘  └─────────────┘            │
├─────────────────────────────────────────────────────────────────┤
│  生成机制                                                       │
│  userId + SALT → hashString() → mulberry32() → roll()          │
│  ├── rollRarity()   → common/uncommon/rare/epic/legendary      │
│  ├── rollSpecies()  → 鸭子/猫/狗等                             │
│  ├── rollStats()    → 智力/魅力/活力等属性                     │
│  └── rollAppearance() → 眼睛/帽子/闪亮                         │
├─────────────────────────────────────────────────────────────────┤
│  UI 渲染                                                       │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐            │
│  │  Companion  │  │  Floating   │  │  Sprite     │            │
│  │  Sprite     │  │  Bubble     │  │  Animation  │            │
│  │  (ASCII画)  │  │  (对话气泡) │  │  (表情动画) │            │
│  └─────────────┘  └─────────────┘  └─────────────┘            │
├─────────────────────────────────────────────────────────────────┤
│  触发机制                                                       │
│  ├── /buddy 命令触发                                           │
│  ├── 启动时 teaser 通知                                        │
│  └── companionIntroText 对话注入                               │
└─────────────────────────────────────────────────────────────────┘
```

### 15.2 伴侣生成算法

```typescript
// buddy/companion.ts
const SALT = 'friend-2026-401';

export function roll(userId: string): Roll {
  // 确定性哈希
  const seed = hashString(userId + SALT);
  const rng = mulberry32(seed);
  
  // 稀有度（加权随机）
  const rarity = rollRarity(rng);  // common → legendary
  
  // 骨架生成
  const bones: CompanionBones = {
    rarity,
    species: pick(rng, SPECIES),    // 鸭子、猫、狗等
    eye: pick(rng, EYES),           // 眼睛样式
    hat: rarity === 'common' ? 'none' : pick(rng, HATS),
    shiny: rng() < 0.01,            // 1% 闪亮几率
    stats: rollStats(rng, rarity),  // 属性生成
  };
  
  return { bones, inspirationSeed: seed };
}

// 属性生成（一高一低，其余随机）
function rollStats(
  rng: () => number,
  rarity: Rarity,
): Record<StatName, number> {
  const floor = RARITY_FLOOR[rarity];  // 稀有度下限
  const peak = pick(rng, STAT_NAMES);  // 高属性
  const dump = pick(rng, STAT_NAMES);  // 低属性
  
  const stats = {} as Record<StatName, number>;
  for (const name of STAT_NAMES) {
    if (name === peak) {
      stats[name] = Math.min(100, floor + 50 + Math.floor(rng() * 30));
    } else if (name === dump) {
      stats[name] = Math.max(1, floor - 10 + Math.floor(rng() * 15));
    } else {
      stats[name] = floor + Math.floor(rng() * 40);
    }
  }
  return stats;
}
```

### 15.3 UI 渲染与交互

```typescript
// buddy/CompanionSprite.tsx
export function CompanionSprite({ companion, state }: Props) {
  const cols = useTerminalSize().columns;
  
  // 终端宽度自适应
  if (cols < MIN_COLS_FOR_FULL_SPRITE) {
    return <CompactView companion={companion} />;
  }
  
  // 完整 ASCII 精灵
  return (
    <Box flexDirection="column">
      <Text color={RARITY_COLORS[companion.rarity]}>
        {renderSprite(SPRITES[companion.species], state.frame)}
      </Text>
      {state.showBubble && (
        <CompanionFloatingBubble
          text={state.bubbleText}
          emotion={state.emotion}
        />
      )}
    </Box>
  );
}

// 5 帧动画
const SPRITES = {
  duck: [
    '    __\n   (  >\n    )/\n   (_',
    // ... 其他帧
  ],
  // ...
};
```

---

## 十六、查询配置与 Token 预算

### 16.1 查询配置

```typescript
// query/config.ts
export type QueryConfig = {
  sessionId: string;
  gates: {
    streamingToolExecution: boolean;    // 流式工具执行
    emitToolUseSummaries: boolean;      // 工具使用摘要
    isAnt: boolean;                     // Anthropic 内部
    fastModeEnabled: boolean;           // 快速模式
  };
};

export function buildQueryConfig(): QueryConfig {
  return {
    sessionId: getSessionId(),
    gates: {
      streamingToolExecution: getFeatureGate('streamingToolExecution'),
      emitToolUseSummaries: getFeatureGate('emitToolUseSummaries'),
      isAnt: process.env.USER_TYPE === 'ant',
      fastModeEnabled: isFastModeEnabled(),
    },
  };
}
```

### 16.2 Token 预算管理

```typescript
// query/tokenBudget.ts
const COMPLETION_THRESHOLD = 0.9;   // 完成阈值
const DIMINISHING_THRESHOLD = 500;  // 收益递减阈值

export type BudgetTracker = {
  turnCount: number;
  recentTokens: number[];
  startTime: number;
};

export function checkTokenBudget(
  tracker: BudgetTracker,
  currentTokens: number,
): BudgetDecision {
  const ratio = currentTokens / getCurrentTurnTokens();
  
  // 达到完成阈值
  if (ratio >= COMPLETION_THRESHOLD) {
    return {
      action: 'stop',
      event: {
        type: 'budget_completion',
        continuationCount: tracker.turnCount,
        percentComplete: ratio * 100,
        durationMs: Date.now() - tracker.startTime,
      },
    };
  }
  
  // 收益递减检测
  const recentDelta = getRecentTokenDelta(tracker);
  if (recentDelta < DIMINISHING_THRESHOLD) {
    return {
      action: 'stop',
      event: {
        type: 'diminishing_returns',
        recentDelta,
      },
    };
  }
  
  // 继续执行
  return {
    action: 'continue',
    nudge: getBudgetContinuationMessage(tracker),
  };
}
```

---

## 十七、输出样式系统

### 17.1 输出样式加载

```typescript
// outputStyles/loadOutputStylesDir.ts
export type OutputStyleConfig = {
  name: string;
  description: string;
  prompt: string;
  source: string;
  keepCodingInstructions: boolean;
};

export function loadOutputStyles(): OutputStyleConfig[] {
  const styles: OutputStyleConfig[] = [];
  
  // 项目级样式
  const projectDir = join(getProjectDir(), '.claude', 'output-styles');
  styles.push(...loadMarkdownFilesForSubdir(projectDir, 'project'));
  
  // 用户级样式
  const userDir = join(getHomeDir(), '.claude', 'output-styles');
  styles.push(...loadMarkdownFilesForSubdir(userDir, 'user'));
  
  return styles;
}

// Markdown 解析
function parseOutputStyle(
  filePath: string,
  content: string,
  source: 'project' | 'user',
): OutputStyleConfig {
  const { frontmatter, body } = parseFrontmatter(content);
  
  return {
    name: frontmatter.name ?? basename(filePath, '.md'),
    description: frontmatter.description ??
      extractDescriptionFromMarkdown(body),
    prompt: body,
    source: filePath,
    keepCodingInstructions: frontmatter['keep-coding-instructions'] !== 'false',
  };
}

// 缓存管理
const loadOutputStylesDirStyles = memoize(loadOutputStyles);

export function clearOutputStyleCaches(): void {
  loadOutputStylesDirStyles.cache.clear?.();
}
```

---

## 核心竞争力总结

### 技术创新

| 维度 | 创新点 | 竞争优势 |
|------|--------|----------|
| **安全** | 24 个权限模块 + AI 分类器 + 沙箱执行 | 业界最完善的终端 AI 安全体系 |
| **工具** | 43 个专业工具 + 统一抽象层 | 覆盖完整开发工作流 |
| **Agent** | 多 Agent 协作 + 工作树隔离 | 支持复杂任务分解和并行执行 |
| **扩展** | Hooks + Plugins + Skills + MCP | 高度可定制的生态系统 |
| **上下文** | 200K-1M token + 长期记忆 | 强大的上下文理解能力 |

### 架构优势

1. **分层设计**：清晰的关注点分离，易于维护和扩展
2. **类型安全**：完整的 TypeScript 类型系统，编译时发现问题
3. **并行优化**：启动时并行预取，运行时并行工具调用
4. **可扩展性**：Hooks、Plugins、Skills 三层扩展机制
5. **企业就绪**：策略管理、审计日志、多租户支持

### 产品体验

1. **智能权限**：AI 驱动的风险评估，减少用户干预
2. **上下文感知**：200K-1M token 理解大型代码库
3. **长期记忆**：跨会话的知识积累和偏好学习
4. **多 Agent**：复杂任务自动分解和并行执行
5. **丰富工具**：43 个工具覆盖开发全流程

---

## 十八、核心架构 COMPLETE 补充

### 18.1 AppState 完整状态变量清单

| 字段 | 类型 | 用途 |
|------|------|------|
| `settings` | object | 运行时配置与用户设置 |
| `tasks` | TaskState[] | 任务队列/状态集合 |
| `agentNameRegistry` | Map | 代理名称与元数据映射 |
| `verbose` | boolean | 详细日志开关 |
| `mainLoopModel` | string | 当前主循环模型 |
| `toolPermissionContext` | ToolPermissionContext | 工具权限上下文 |
| `agent` | string | 当前代理类型 |
| `agentDefinitions` | AgentDefinitions | 代理定义集合 |
| `mcp` | MCPState | MCP 客户端/工具/资源 |
| `plugins` | PluginState | 插件启用/禁用/错误状态 |
| `notifications` | NotificationQueue | 通知队列 |
| `todos` | TodoState[] | 待办任务集合 |
| `fileHistory` | FileHistoryState | 文件历史/快照 |
| `attribution` | AttributionState | 资源归属状态 |
| `thinkingEnabled` | boolean | Thinking 模式开关 |
| `speculation` | SpeculationState | 推理状态 |
| `effortValue` | number | 努力程度/预算 |
| `fastMode` | FastModeState | 快速模式状态 |
| `teamContext` | TeamContext | 团队协作上下文 |

### 18.2 QueryEngine 完整流程

```typescript
// QueryEngine 核心流程
export class QueryEngine {
  // 1. 初始化
  constructor(config: QueryEngineConfig) {
    this.config = config;
    this.mutableMessages = [...config.initialMessages];
    this.abortController = new AbortController();
    this.totalUsage = EMPTY_USAGE;
  }
  
  // 2. 提交消息 (Generator)
  async *submitMessage(prompt, options) {
    // a) 构建系统提示
    const { defaultSystemPrompt, userContext, systemContext } =
      await fetchSystemPromptParts(this.config);
    
    // b) 处理用户输入
    const { messagesFromUserInput, shouldQuery, allowedTools } =
      await processUserInput(prompt, context);
    
    // c) 进入核心查询循环
    for await (const message of query({
      messages: this.mutableMessages,
      systemPrompt,
      tools: allowedTools,
      signal: this.abortController.signal,
    })) {
      yield message;
      this.mutableMessages.push(message);
    }
    
    // d) 持久化转录
    await flushSessionStorage(this.mutableMessages);
  }
  
  // 3. 中断
  interrupt(): void {
    this.abortController.abort();
  }
}
```

### 18.3 Store 实现细节

```typescript
// state/store.ts
export function createStore<T>(
  initialState: T,
  onChange?: (args: { newState: T; oldState: T }) => void,
): Store<T> {
  let state = initialState;
  const listeners = new Set<() => void>();
  
  return {
    getState: () => state,
    
    setState: (updater) => {
      const oldState = state;
      const newState = typeof updater === 'function'
        ? (updater as (s: T) => T)(state)
        : updater;
      
      if (Object.is(oldState, newState)) return;
      
      state = newState;
      onChange?.({ newState, oldState });
      listeners.forEach(l => l());
    },
    
    subscribe: (listener) => {
      listeners.add(listener);
      return () => listeners.delete(listener);
    },
  };
}
```

### 18.4 Selectors 状态派生

```typescript
// state/selectors.ts
export function getViewedTeammateTask(
  appState: Pick<AppState, 'viewingAgentTaskId' | 'tasks'>,
): InProcessTeammateTaskState | undefined {
  if (!appState.viewingAgentTaskId) return undefined;
  const task = appState.tasks[appState.viewingAgentTaskId];
  if (!task || !isInProcessTeammateTask(task)) return undefined;
  return task;
}

export function getActiveAgentForInput(
  appState: AppState,
): ActiveAgentForInput {
  const viewedTask = getViewedTeammateTask(appState);
  if (viewedTask) return { type: 'viewed', task: viewedTask };
  
  if (appState.viewingAgentTaskId) {
    const task = appState.tasks[appState.viewingAgentTaskId];
    if (task && isLocalAgentTask(task)) {
      return { type: 'named_agent', task };
    }
  }
  
  return { type: 'leader' };
}
```

### 18.5 onChangeAppState 副作用处理

```typescript
// state/onChangeAppState.ts
export function onChangeAppState({ newState, oldState }: {
  newState: AppState;
  oldState: AppState;
}): void {
  // 权限模式变更
  if (newState.toolPermissionContext.mode !== oldState.toolPermissionContext.mode) {
    const externalMode = toExternalPermissionMode(newState.toolPermissionContext.mode);
    notifySessionMetadataChanged({ permission_mode: externalMode });
    notifyPermissionModeChanged(newState.toolPermissionContext.mode);
  }
  
  // 模型变更
  if (newState.mainLoopModel !== oldState.mainLoopModel) {
    if (newState.mainLoopModel === null) {
      setMainLoopModelOverride(null);
    } else {
      updateSettingsForSource('userSettings', {
        model: newState.mainLoopModel,
      });
      setMainLoopModelOverride(newState.mainLoopModel);
    }
  }
  
  // 视图模式变更
  if (newState.expandedView !== oldState.expandedView) {
    const showExpanded = newState.expandedView !== 'none';
    saveGlobalConfig(prev => ({
      ...prev,
      showExpandedTodos: showExpanded,
      showSpinnerTree: showExpanded,
    }));
  }
  
  // Settings 变更 - 清除缓存
  if (newState.settings !== oldState.settings) {
    clearApiKeyHelperCache();
    clearAwsCredentialsCache();
    clearGcpCredentialsCache();
    applyConfigEnvironmentVariables(newState.settings.env);
  }
}
```

---

## 十九、Beta Headers 完整列表

| Beta Header | 值 | 说明 |
|-------------|-----|------|
| `CLAUDE_CODE_20250219_BETA_HEADER` | `claude-code-20250219` | Claude Code 基础 Beta |
| `INTERLEAVED_THINKING_BETA_HEADER` | `interleaved-thinking-2025-05-14` | 交错思考 |
| `CONTEXT_1M_BETA_HEADER` | `context-1m-2025-08-07` | 1M 上下文窗口 |
| `CONTEXT_MANAGEMENT_BETA_HEADER` | `context-management-2025-06-27` | 上下文管理 |
| `STRUCTURED_OUTPUTS_BETA_HEADER` | `structured-outputs-2025-12-15` | 结构化输出 |
| `WEB_SEARCH_BETA_HEADER` | `web-search-2025-03-05` | Web 搜索 |
| `TOOL_SEARCH_BETA_HEADER_1P` | `advanced-tool-use-2025-11-20` | 工具搜索 (1P) |
| `TOOL_SEARCH_BETA_HEADER_3P` | `tool-search-tool-2025-10-19` | 工具搜索 (3P) |
| `EFFORT_BETA_HEADER` | `effort-2025-11-24` | Effort 模式 |
| `TASK_BUDGETS_BETA_HEADER` | `task-budgets-2026-03-13` | 任务预算 |
| `PROMPT_CACHING_SCOPE_BETA_HEADER` | `prompt-caching-scope-2026-01-05` | 缓存范围 |
| `FAST_MODE_BETA_HEADER` | `fast-mode-2026-02-01` | 快速模式 |
| `REDACT_THINKING_BETA_HEADER` | `redact-thinking-2026-02-12` | Thinking 编辑 |
| `TOKEN_EFFICIENT_TOOLS_BETA_HEADER` | `token-efficient-tools-2026-03-28` | Token 高效工具 |
| `ADVISOR_BETA_HEADER` | `advisor-tool-2026-03-01` | Advisor 工具 |

---

## 十九、43 个工具完整清单

### 19.1 工具总览表

| # | Tool 名称 | 作用简述 | Input 关键字段 | ReadOnly | ConcurrencySafe | Destructive |
|---|-----------|----------|----------------|----------|-----------------|-------------|
| 1 | Agent | 启动子Agent执行任务 | description, prompt, subagent_type, model, isolation | ✗ | ✗ | ✗ |
| 2 | AskUserQuestion | 向用户提问获取信息 | questions | ✓ | ✓ | ✗ |
| 3 | Bash | 执行Shell命令 | command, timeout | ✗ | ✗ | ✗ |
| 4 | Brief | 生成对话摘要 | (无) | ✓ | ✓ | ✗ |
| 5 | Config | 读取/更新配置 | key, value | ✗ | ✓ | ✗ |
| 6 | EnterPlanMode | 进入计划模式 | (无) | ✗ | ✓ | ✗ |
| 7 | EnterWorktree | 进入工作树 | directory, branch | ✗ | ✗ | ✗ |
| 8 | ExitPlanMode | 退出计划模式 | plan | ✗ | ✓ | ✗ |
| 9 | ExitWorktree | 退出工作树 | (无) | ✗ | ✗ | ✗ |
| 10 | Edit | 编辑文件内容 | file_path, old_string, new_string | ✗ | ✗ | ✗ |
| 11 | Read | 读取文件内容 | file_path, offset, limit | ✓ | ✓ | ✗ |
| 12 | Write | 写入文件内容 | file_path, content | ✗ | ✗ | ✓ |
| 13 | Glob | 文件模式匹配 | pattern, path | ✓ | ✓ | ✗ |
| 14 | Grep | 代码搜索 | pattern, path, include | ✓ | ✓ | ✗ |
| 15 | ListMcpResources | 列出MCP资源 | server_name | ✓ | ✓ | ✗ |
| 16 | LSP | 语言服务器查询 | query, file_path | ✓ | ✓ | ✗ |
| 17 | McpAuth | MCP认证管理 | server_name, action | ✗ | ✗ | ✗ |
| 18 | mcp | MCP工具占位 | (由MCP服务器定义) | ✗ | ✗ | ✗ |
| 19 | NotebookEdit | 编辑Jupyter单元格 | notebook_path, cell_id, new_source | ✗ | ✗ | ✗ |
| 20 | PowerShell | 执行PowerShell命令 | command, timeout | ✗ | ✗ | ✗ |
| 21 | ReadMcpResource | 读取MCP资源 | server_name, uri | ✓ | ✓ | ✗ |
| 22 | RemoteTrigger | 触发远程操作 | url, payload | ✗ | ✗ | ✗ |
| 23 | REPL | 启动交互式环境 | command, timeout | ✗ | ✗ | ✗ |
| 24 | ScheduleCron | 创建定时任务 | cron, command, description | ✗ | ✗ | ✗ |
| 25 | SendMessage | 发送消息给Agent | to, content | ✗ | ✓ | ✗ |
| 26 | Skill | 加载技能执行 | skill_name, input | ✗ | ✗ | ✗ |
| 27 | Sleep | 暂停执行 | duration_ms | ✓ | ✓ | ✗ |
| 28 | SyntheticOutput | 生成合成输出 | content, format | ✓ | ✓ | ✗ |
| 29 | TaskCreate | 创建结构化任务 | subject, description, activeForm | ✗ | ✓ | ✗ |
| 30 | TaskGet | 获取任务详情 | taskId | ✓ | ✓ | ✗ |
| 31 | TaskList | 列出所有任务 | (无) | ✓ | ✓ | ✗ |
| 32 | TaskOutput | 获取任务输出 | task_id | ✓ | ✓ | ✗ |
| 33 | TaskStop | 停止后台任务 | task_id | ✗ | ✓ | ✗ |
| 34 | TaskUpdate | 更新任务属性 | taskId, status, subject | ✗ | ✓ | ✗ |
| 35 | TeamCreate | 创建团队 | name, description | ✗ | ✗ | ✗ |
| 36 | TeamDelete | 删除团队 | team_name | ✗ | ✗ | ✓ |
| 37 | TodoWrite | 管理待办事项 | todos | ✗ | ✗ | ✗ |
| 38 | ToolSearch | 搜索可用工具 | query | ✓ | ✓ | ✗ |
| 39 | WebFetch | 抓取网页内容 | url, prompt | ✓ | ✓ | ✗ |
| 40 | WebSearch | 网络搜索 | query, allowed_domains | ✓ | ✓ | ✗ |

### 19.2 工具执行框架 (Tool.ts)

```typescript
// Tool.ts - 工具接口定义
export type Tool<Input, Output, P> = {
  name: string;                    // 工具主名称
  aliases?: string[];              // 备用名称
  searchHint?: string;             // 搜索提示词
  
  // 核心方法
  call(args, context, canUseTool, parentMessage, onProgress?):
    Promise<ToolResult<Output>>;
  description(input, options): Promise<string>;
  
  // Schema
  inputSchema: Input;
  outputSchema?: z.ZodType<Output>;
  
  // 安全标志
  isReadOnly(input): boolean;
  isConcurrencySafe(input): boolean;
  isDestructive?(input): boolean;
  
  // 权限
  checkPermissions(input, context): Promise<PermissionResult>;
  validateInput?(input, context): Promise<ValidationResult>;
  
  // UI 渲染
  prompt(options): Promise<string>;
  userFacingName(input): string;
  renderToolUseMessage?(input, options): ReactNode;
  renderToolResultMessage?(result, options): ReactNode;
};
```

---

## 二十、Types 目录完整类型清单

### 20.1 权限类型 (permissions.ts)

```
ExternalPermissionMode, PermissionMode, PermissionBehavior,
PermissionRuleSource, PermissionRuleValue, PermissionRule,
PermissionUpdate, AdditionalWorkingDirectory, PermissionResult,
ToolPermissionRulesBySource, ToolPermissionContext, RiskLevel
```

### 20.2 Hooks 类型 (hooks.ts)

```
HookEvent, PromptRequest, PromptResponse, HookJSONOutput,
HookCallback, HookCallbackMatcher, HookProgress, HookResult,
AggregatedHookResult, PermissionRequestResult
```

### 20.3 Command 类型 (command.ts)

```
LocalCommandResult, PromptCommand, LocalCommand, Command,
CommandAvailability, CommandBase, ResumeEntrypoint,
CommandResultDisplay, LocalJSXCommand
```

### 20.4 IDs 类型 (ids.ts)

```
SessionId, AgentId, asSessionId(), asAgentId(), toAgentId()
```

### 20.5 Logs 类型 (logs.ts)

```
SerializedMessage, LogOption, SummaryMessage, TranscriptMessage,
Entry, sortLogs()
```

### 20.6 TextInput 类型 (textInputTypes.ts)

```
InlineGhostText, VimMode, PromptInputMode, QueuedCommand,
OrphanedPermission, isValidImagePaste()
```

---

## 结语

Claude Code 的强大不仅在于其 AI 模型的能力，更在于其**精巧的工程架构**。通过深入分析源码，我们可以看到：

1. **安全是第一公民**：多层权限系统、沙箱执行、AI 风险分类，确保 AI 在安全边界内运行

2. **工具是核心竞争力**：43 个专业工具、统一抽象层、智能协作机制，覆盖完整开发工作流

3. **扩展性是生命力**：Hooks、Plugins、Skills、MCP 四层扩展机制，构建开放生态

4. **性能是基础保障**：并行预取、智能缓存、延迟加载，提供流畅用户体验

5. **企业级是护城河**：策略管理、审计日志、多租户支持，满足企业安全合规需求

Claude Code 代表了 AI 编程助手的最高水平，其技术架构值得每一位开发者学习和借鉴。

---

> **文档信息**
> - 版本：2.1.88
> - 分析日期：2026-04-01
> - 源码文件：4756 个（含 1884 个 TypeScript/TSX 源文件）
> - 文档字数：约 28000 字
> - 章节数量：21 个主章节 + 核心竞争力总结 + 结语
> - 架构图：15+ 个 ASCII 架构图
> - 代码示例：65+ 个
> - 工具清单：40 个工具完整表格
> - 覆盖模块：核心架构、安全体系、工具系统(40个工具)、多Agent、扩展系统、上下文记忆、性能优化、UI/UX、企业特性、API层、全局状态、任务调度、远程连接、Buddy系统、查询配置、输出样式、Beta Headers、类型定义、Entry Points
