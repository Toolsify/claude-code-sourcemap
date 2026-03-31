## 十九、43 个工具完整清单

### 19.1 工具总览表
| # | Tool 名称 | 作用简述 | Input 关键字段 | ReadOnly | ConcurrencySafe | Destructive |
|---|-----------|----------|----------------|----------|-----------------|-------------|
| 1 | Agent | 启动子Agent执行任务 | description, prompt, subagent_type, model, isolation | ✗ | ✗ | ✗ |
| 2 | AskUserQuestion | 向用户提问获取信息 | questions | ✓ | ✓ | ✗ |
| 3 | Bash | 执行Shell命令 | command, timeout | ✗ | ✗ | ✗ |
| 4 | Brief | 生成对话摘要 | (无) | ✓ | ✓ | ✗ |
| 5 | Config | 读取/更新配置 | key, value | ✗ | ✓ | ✗ |
| 6 | EnterPlanMode | 进入计划模式 | (无) | ✗ | ✓ | ✗ |
| 7 | EnterWorktree | 进入工作树 | directory, branch | ✗ | ✗ | ✗ |
| 8 | ExitPlanMode | 退出计划模式 | plan | ✗ | ✓ | ✗ |
| 9 | ExitWorktree | 退出工作树 | (无) | ✗ | ✗ | ✗ |
| 10 | Edit | 编辑文件内容 | file_path, old_string, new_string | ✗ | ✗ | ✗ |
| 11 | Read | 读取文件内容 | file_path, offset, limit | ✓ | ✓ | ✗ |
| 12 | Write | 写入文件内容 | file_path, content | ✗ | ✗ | ✓ |
| 13 | Glob | 文件模式匹配 | pattern, path | ✓ | ✓ | ✗ |
| 14 | Grep | 代码搜索 | pattern, path, include | ✓ | ✓ | ✗ |
| 15 | ListMcpResources | 列出MCP资源 | server_name | ✓ | ✓ | ✗ |
| 16 | LSP | 语言服务器查询 | query, file_path | ✓ | ✓ | ✗ |
| 17 | McpAuth | MCP认证管理 | server_name, action | ✗ | ✗ | ✗ |
| 18 | mcp | MCP工具占位 | (由MCP服务器定义) | ✗ | ✗ | ✗ |
| 19 | NotebookEdit | 编辑Jupyter单元格 | notebook_path, cell_id, new_source | ✗ | ✗ | ✗ |
| 20 | PowerShell | 执行PowerShell命令 | command, timeout | ✗ | ✗ | ✗ |
| 21 | ReadMcpResource | 读取MCP资源 | server_name, uri | ✓ | ✓ | ✗ |
| 22 | RemoteTrigger | 触发远程操作 | url, payload | ✗ | ✗ | ✗ |
| 23 | REPL | 启动交互式环境 | command, timeout | ✗ | ✗ | ✗ |
| 24 | ScheduleCron | 创建定时任务 | cron, command, description | ✗ | ✗ | ✗ |
| 25 | SendMessage | 发送消息给Agent | to, content | ✗ | ✓ | ✗ |
| 26 | Skill | 加载技能执行 | skill_name, input | ✗ | ✗ | ✗ |
| 27 | Sleep | 暂停执行 | duration_ms | ✓ | ✓ | ✗ |
| 28 | SyntheticOutput | 生成合成输出 | content, format | ✓ | ✓ | ✗ |
| 29 | TaskCreate | 创建结构化任务 | subject, description, activeForm | ✗ | ✓ | ✗ |
| 30 | TaskGet | 获取任务详情 | taskId | ✓ | ✓ | ✗ |
| 31 | TaskList | 列出所有任务 | (无) | ✓ | ✓ | ✗ |
| 32 | TaskOutput | 获取任务输出 | task_id | ✓ | ✓ | ✗ |
| 33 | TaskStop | 停止后台任务 | task_id | ✗ | ✓ | ✗ |
| 34 | TaskUpdate | 更新任务属性 | taskId, status, subject | ✗ | ✓ | ✗ |
| 35 | TeamCreate | 创建团队 | name, description | ✗ | ✗ | ✗ |
| 36 | TeamDelete | 删除团队 | team_name | ✗ | ✗ | ✓ |
| 37 | TodoWrite | 管理待办事项 | todos | ✗ | ✗ | ✗ |
| 38 | ToolSearch | 搜索可用工具 | query | ✓ | ✓ | ✗ |
| 39 | WebFetch | 抓取网页内容 | url, prompt | ✓ | ✓ | ✗ |
| 40 | WebSearch | 网络搜索 | query, allowed_domains | ✓ | ✓ | ✗ |

---

```typescript
// Tool.ts - 工具接口定义
export type Tool<Input, Output, P> = {
  name: string;                    // 工具主名称
  aliases?: string[];              // 备用名称
  searchHint?: string;             // 搜索提示词
  
  // 核心方法
  call(args, context, canUseTool, parentMessage, onProgress?):
    Promise<ToolResult<Output>>;
  description(input, options): Promise<string>;
  
  // Schema
  inputSchema: Input;
  outputSchema?: z.ZodType<Output>;
  
  // 安全标志
  isReadOnly(input): boolean;
  isConcurrencySafe(input): boolean;
  isDestructive?(input): boolean;
  
  // 权限
  checkPermissions(input, context): Promise<PermissionResult>;
  validateInput?(input, context): Promise<ValidationResult>;
  
  // UI 渲染
  prompt(options): Promise<string>;
  userFacingName(input): string;
  renderToolUseMessage?(input, options): ReactNode;
  renderToolResultMessage?(result, options): ReactNode;
};
```
