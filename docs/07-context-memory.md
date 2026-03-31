## 七、上下文与记忆系统

### 7.1 上下文窗口管理

Claude Code 支持从 200K 到 1M token 的动态上下文窗口：

```
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
