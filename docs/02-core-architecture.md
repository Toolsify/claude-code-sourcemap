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
