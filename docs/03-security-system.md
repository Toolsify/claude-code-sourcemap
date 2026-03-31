## 三、安全执行体系

### 3.1 多层权限系统

Claude Code 实现了业界领先的 **24 个权限模块**，形成纵深防御：

```
┌─────────────────────────────────────────────────────────────┐
│                    Permission Architecture                   │
├─────────────────────────────────────────────────────────────┤
│  Layer 1: Permission Modes                                  │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐          │
│  │ default │ │  plan   │ │ accept  │ │ bypass  │          │
│  │ (询问)  │ │ (计划)  │ │ (自动)  │ │ (绕过)  │          │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘          │
├─────────────────────────────────────────────────────────────┤
│  Layer 2: Permission Rules                                  │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐          │
│  │ AlwaysAllow │ │ AlwaysDeny  │ │  AlwaysAsk  │          │
│  │ (白名单)    │ │ (黑名单)    │ │ (强制询问)  │          │
│  └─────────────┘ └─────────────┘ └─────────────┘          │
├─────────────────────────────────────────────────────────────┤
│  Layer 3: Intelligent Classifier                            │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐          │
│  │   Bash      │ │  Filesystem │ │  Network    │          │
│  │ Classifier  │ │  Validator  │ │  Filter     │          │
│  └─────────────┘ └─────────────┘ └─────────────┘          │
├─────────────────────────────────────────────────────────────┤
│  Layer 4: Sandbox Isolation                                 │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐          │
│  │ Filesystem  │ │   Network   │ │  Process    │          │
│  │ Restrictions│ │  Restrictions│ │  Isolation  │          │
│  └─────────────┘ └─────────────┘ └─────────────┘          │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 权限模式详解

```typescript
// utils/permissions/PermissionMode.ts
export const PERMISSION_MODE_CONFIG = {
  default: {
    title: 'Default',
    shortTitle: 'Default',
    symbol: '',
    color: 'text',
    external: 'default',
  },
  plan: {
    title: 'Plan Mode',
    shortTitle: 'Plan',
    symbol: PAUSE_ICON,  // ⏸
    color: 'planMode',
    external: 'plan',
  },
  acceptEdits: {
    title: 'Accept edits',
    shortTitle: 'Accept',
    symbol: '⏵⏵',
    color: 'autoAccept',
    external: 'acceptEdits',
  },
  bypassPermissions: {
    title: 'Bypass Permissions',
    shortTitle: 'Bypass',
    symbol: '⏵⏵',
    color: 'error',
    external: 'bypassPermissions',
  },
  auto: {
    title: 'Auto mode',
    shortTitle: 'Auto',
    symbol: '⏵⏵',
    color: 'warning',
    external: 'default',
  },
};
```

### 3.3 权限上下文

```typescript
// Tool.ts - 工具权限上下文
export type ToolPermissionContext = {
  // 当前权限模式
  mode: PermissionMode;
  
  // 额外工作目录（多目录支持）
  additionalWorkingDirectories: Map<string, AdditionalWorkingDirectory>;
  
  // 权限规则
  alwaysAllowRules: ToolPermissionRulesBySource;
  alwaysDenyRules: ToolPermissionRulesBySource;
  alwaysAskRules: ToolPermissionRulesBySource;
  
  // 绕过权限模式
  isBypassPermissionsModeAvailable: boolean;
  
  // 自动模式（AI 分类器）
  isAutoModeAvailable?: boolean;
  
  // 危险规则过滤
  strippedDangerousRules?: ToolPermissionRulesBySource;
  
  // 避免权限提示（后台 Agent）
  shouldAvoidPermissionPrompts?: boolean;
};
```

### 3.4 沙箱执行机制

```typescript
// utils/sandbox/sandbox-adapter.ts
export class SandboxManager {
  // 检查沙箱是否启用
  static isSandboxingEnabled(): boolean {
    return getSettingsForSource('policySettings')
      ?.sandbox?.enabled ?? false;
  }
  
  // 文件系统限制
  static getFsRestrictions(): {
    allowRead: string[];
    allowWrite: string[];
    denyRead: string[];
    denyWrite: string[];
  } {
    const settings = getSettingsForSource('policySettings');
    return {
      allowRead: settings?.sandbox?.filesystem?.allowRead ?? [],
      allowWrite: settings?.sandbox?.filesystem?.allowWrite ?? [],
      denyRead: settings?.sandbox?.filesystem?.denyRead ?? [],
      denyWrite: settings?.sandbox?.filesystem?.denyWrite ?? [],
    };
  }
  
  // 网络限制
  static getNetworkRestrictions(): {
    allowHosts: NetworkHostPattern[];
    denyHosts: NetworkHostPattern[];
  } {
    const settings = getSettingsForSource('policySettings');
    return {
      allowHosts: settings?.sandbox?.network?.allowHosts ?? [],
      denyHosts: settings?.sandbox?.network?.denyHosts ?? [],
    };
  }
}
```

### 3.5 智能风险评估

```typescript
// utils/permissions/bashClassifier.ts
export type ClassifierResult = {
  matches: boolean;
  matchedDescription?: string;
  confidence: 'high' | 'medium' | 'low';
  reason: string;
};

export type ClassifierBehavior = 'deny' | 'ask' | 'allow';
 
// AI 驱动的命令分类
export async function classifyBashCommand(
  command: string,
  cwd: string,
  descriptions: string[],
  behavior: ClassifierBehavior,
  signal: AbortSignal,
  isNonInteractiveSession: boolean,
): Promise<ClassifierResult> {
  // 1. 检查危险模式
  if (matchesDangerousPattern(command)) {
    return {
      matches: true,
      confidence: 'high',
      reason: 'Command matches dangerous pattern',
    };
  }
  
  // 2. AI 分类（ANT-ONLY）
  if (isClassifierPermissionsEnabled()) {
    return await aiClassify(command, cwd, descriptions, signal);
  }
  
  return { matches: false, confidence: 'high', reason: 'Safe' };
}
```

---
