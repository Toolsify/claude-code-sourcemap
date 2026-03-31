## 五、多 Agent 协作系统

### 5.1 Agent 架构设计

Claude Code 实现了先进的多 Agent 系统，支持任务委派和并行执行：

```
┌─────────────────────────────────────────────────────────────┐
│                    Multi-Agent Architecture                  │
├─────────────────────────────────────────────────────────────┤
│                     Main Thread (主 Agent)                   │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  User Query → QueryEngine → Tool Selection → Execute │  │
│  └───────────────────────────────────────────────────────┘  │
├─────────────────────────────────────────────────────────────┤
│                      Agent Tool (Agent 工具)                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │  Foreground │  │  Background │  │   Remote    │        │
│  │  (前台执行)  │  │  (后台执行)  │  │  (远程执行)  │        │
│  └─────────────┘  └─────────────┘  └─────────────┘        │
├─────────────────────────────────────────────────────────────┤
│                    Agent Isolation (隔离机制)                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │  Worktree   │  │   Remote    │  │    Team     │        │
│  │  (工作树)   │  │  (远程环境)  │  │  (团队协作)  │        │
│  └─────────────┘  └─────────────┘  └─────────────┘        │
└─────────────────────────────────────────────────────────────┘
```

### 5.2 AgentTool 实现

```typescript
// tools/AgentTool/AgentTool.tsx
const baseInputSchema = lazySchema(() => z.object({
  description: z.string().describe('A short (3-5 word) description of the task'),
  prompt: z.string().describe('The task for the agent to perform'),
  subagent_type: z.string().optional().describe('The type of specialized agent'),
  model: z.enum(['sonnet', 'opus', 'haiku']).optional().describe('Model override'),
  run_in_background: z.boolean().optional().describe('Run in background'),
}));

const fullInputSchema = lazySchema(() => {
  const multiAgentInputSchema = z.object({
    name: z.string().optional().describe('Name for the spawned agent'),
    team_name: z.string().optional().describe('Team name for spawning'),
    mode: permissionModeSchema().optional().describe('Permission mode'),
  });
  
  return baseInputSchema().merge(multiAgentInputSchema).extend({
    isolation: z.enum(['worktree', 'remote']).optional().describe('Isolation mode'),
    cwd: z.string().optional().describe('Working directory override'),
  });
});
```

### 5.3 任务委派机制

```typescript
// tools/AgentTool/runAgent.ts
export async function runAgent(
  input: AgentInput,
  context: ToolUseContext,
  canUseTool: CanUseToolFn,
): Promise<ToolResult<AgentOutput>> {
  // 1. 创建 Agent ID
  const agentId = createAgentId();
  
  // 2. 选择执行模式
  if (input.isolation === 'worktree') {
    // 工作树隔离模式
    const worktree = await createAgentWorktree(agentId, input.cwd);
    return runInWorktree(agentId, input, worktree, context);
  }
  
  if (input.run_in_background) {
    // 后台执行模式
    return runInBackground(agentId, input, context);
  }
  
  // 3. 前台执行
  return runInForeground(agentId, input, context);
}
```

### 5.4 工作树隔离

```typescript
// utils/worktree.ts
export async function createAgentWorktree(
  agentId: AgentId,
  baseBranch?: string,
): Promise<WorktreeInfo> {
  const gitRoot = await findGitRoot();
  const worktreePath = join(gitRoot, '.claude', 'worktrees', agentId);
  
  // 创建 Git 工作树
  await exec(`git worktree add ${worktreePath} ${baseBranch || 'HEAD'}`);
  
  return {
    path: worktreePath,
    agentId,
    createdAt: Date.now(),
  };
}

export async function removeAgentWorktree(
  worktree: WorktreeInfo,
): Promise<void> {
  // 清理工作树
  await exec(`git worktree remove ${worktree.path} --force`);
}
```

---
