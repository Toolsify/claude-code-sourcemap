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
// bootstrap/state.ts - 状态变更
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
