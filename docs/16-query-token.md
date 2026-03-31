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
