## 九、UI/UX 架构

### 9.1 终端 UI 框架

Claude Code 使用 **Ink**（React for CLI）构建终端界面：

```typescript
// components/App.tsx
export function App({
  getFpsMetrics,
  stats,
  initialState,
  children,
}: Props): React.ReactElement {
  return (
    <FpsMetricsProvider getFpsMetrics={getFpsMetrics}>
      <StatsProvider store={stats}>
        <AppStateProvider
          initialState={initialState}
          onChangeAppState={onChangeAppState}
        >
          {children}
        </AppStateProvider>
      </StatsProvider>
    </FpsMetricsProvider>
  );
}
```

### 9.2 组件层次

```
┌─────────────────────────────────────────────────────────────┐
│                        App (根组件)                          │
├─────────────────────────────────────────────────────────────┤
│  Providers                                                  │
│  ├── FpsMetricsProvider (性能指标)                          │
│  ├── StatsProvider (统计数据)                               │
│  ├── AppStateProvider (应用状态)                            │
│  ├── MailboxProvider (消息队列)                             │
│  └── VoiceProvider (语音集成)                               │
├─────────────────────────────────────────────────────────────┤
│  Screens                                                    │
│  ├── REPL (交互式界面)                                      │
│  ├── Doctor (诊断工具)                                      │
│  └── ResumeConversation (恢复会话)                          │
├─────────────────────────────────────────────────────────────┤
│  Components (144 个)                                        │
│  ├── PromptInput (输入组件)                                 │
│  ├── Messages (消息列表)                                    │
│  ├── ToolUseLoader (工具使用加载)                           │
│  ├── PermissionRequest (权限请求)                           │
│  └── ...                                                    │
└─────────────────────────────────────────────────────────────┘
```

### 9.3 虚拟消息列表

````typescript
// components/VirtualMessageList.tsx
export function VirtualMessageList({
  messages,
  jumpHandle,
  onScroll,
}: VirtualMessageListProps) {
  // 虚拟滚动 - 只渲染可见消息
  const visibleRange = useMemo(() => {
    return calculateVisibleRange(
      scrollTop,
      viewportHeight,
      itemHeights,
    );
  }, [scrollTop, viewportHeight, itemHeights]);
  
  return (
    <Box flexDirection="column">
      {messages.slice(visibleRange.start, visibleRange.end).map(msg => (
        <MessageRow key={msg.id} message={msg} />
      ))}
    </Box>
  );
}
````
```

---
