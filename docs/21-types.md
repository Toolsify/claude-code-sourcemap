## 二十一、Types 目录完整类型清单

### 21.1 权限类型 (permissions.ts)

```
ExternalPermissionMode, PermissionMode, PermissionBehavior,
PermissionRuleSource, PermissionRuleValue, PermissionRule,
PermissionUpdate, AdditionalWorkingDirectory, PermissionResult,
ToolPermissionRulesBySource, ToolPermissionContext, RiskLevel
```

### 21.2 Hooks 类型 (hooks.ts)

```
HookEvent, PromptRequest, PromptResponse, HookJSONOutput,
HookCallback, HookCallbackMatcher, HookProgress, HookResult,
AggregatedHookResult, PermissionRequestResult
```

### 21.3 Command 类型 (command.ts)

```
LocalCommandResult, PromptCommand, LocalCommand, Command,
CommandAvailability, CommandBase, ResumeEntrypoint,
CommandResultDisplay, LocalJSXCommand
```

### 21.4 IDs 类型 (ids.ts)

```
SessionId, AgentId, asSessionId(), asAgentId(), toAgentId()
```

### 21.5 Logs 类型 (logs.ts)

```
SerializedMessage, LogOption, SummaryMessage, TranscriptMessage,
Entry, sortLogs()
```

### 21.6 TextInput 类型 (textInputTypes.ts)

```
InlineGhostText, VimMode, PromptInputMode, QueuedCommand,
OrphanedPermission, isValidImagePaste()
```

---
