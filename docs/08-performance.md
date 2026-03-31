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
