## 核心竞争力 COMPLETE 补充

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
