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
