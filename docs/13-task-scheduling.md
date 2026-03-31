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
