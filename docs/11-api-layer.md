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
