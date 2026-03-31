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
