## 十四、远程连接与通信

### 14.1 远程连接架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                    Claude Code Frontend (CLI/UI)                 │
├─────────────────────────────────────────────────────────────────┤
│                          │ cc:// 协议                           │
│                          v                                      │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              Direct Connect Server                       │   │
│  │  POST /sessions → { sessionId, wsUrl, authToken }       │   │
+  └─────────────────────────────────────────────────────────┘   │
│                          │                                      │
│                          v                                      │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              WebSocket Connection                         │   │
│  │  wss://api.anthropic.com/v1/sessions/ws/{id}/subscribe  │   │
│  └─────────────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────────────┤
├─────────────────────────────────────────────────────────────────┤
│                         Bridge Layer                            │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  - 环境注册与会话管理                                     │   │
│  │  - 令牌轮换与心跳                                        │   │
│  │  - 多会话调度                                            │   │
│  └─────────────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────────────┤
│                      Remote CCR Layer                           │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  RemoteSessionManager                                   │   │
│  │  ├── SessionsWebSocket (订阅与认证)                      │   │
│  │  ├── handleMessage (消息分发)                            │   │
│  │  ├── handleControlRequest (权限请求)                     │   │
│  │  └── sdkMessageAdapter (消息转译)                        │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### 14.2 直接连接管理 (cc://)

```typescript
// server/directConnectManager.ts
export type DirectConnectConfig = {
  serverUrl: string;   // REST API 根 URL
  sessionId: string;   // 会话 ID
  wsUrl: string;       // WebSocket 端点
  authToken: string;   // 认证令牌
};

export class DirectConnectSessionManager {
  private ws: WebSocket | null = null;
  
  async connect(): Promise<void> {
    // 建立 WebSocket 连接
    this.ws = new WebSocket(this.config.wsUrl, {
      headers: {
        Authorization: `Bearer ${this.config.authToken}`,
      },
    });
    
    this.ws.on('open', () => this.callbacks.onConnected());
    this.ws.on('message', (data) => this.handleMessage(data));
    this.ws.on('close', () => this.callbacks.onDisconnected());
    this.ws.on('error', (err) => this.callbacks.onError(err));
  }
  
  private handleMessage(data: string): void {
    const message = JSON.parse(data);
    
    // 控制请求 - 权限
    if (message.type === 'control_request') {
      if (message.request?.subtype === 'can_use_tool') {
        this.callbacks.onPermissionRequest(
          message.request,
          message.request_id,
        );
      }
      return;
    }
    
    // SDK 消息 - 转发给上层
    this.callbacks.onMessage(message);
  }
  
  // 发送用户消息
  sendMessage(content: string): void {
    this.ws?.send(JSON.stringify({
      type: 'user',
      message: { role: 'user', content },
    }));
  }
  
  // 响应权限请求
  respondToPermissionRequest(
    requestId: string,
    result: PermissionResult,
  ): void {
    this.ws?.send(JSON.stringify({
      type: 'control_response',
      subtype: 'success',
      request_id: requestId,
      response: result,
    }));
  }
}
```

### 14.3 WebSocket 通信协议

```typescript
// remote/SessionsWebSocket.ts
export class SessionsWebSocket {
  // 连接 URL
  // wss://api.anthropic.com/v1/sessions/ws/{sessionId}/subscribe
  //   ?organization_uuid={orgUuid}
  
  // 认证消息
  // { type: 'auth', credential: { type: 'oauth', token: '...' } }
  
  // 消息类型
  // - SDKMessage: 模型输出
  // - control_request: 权限/中断请求
  // - control_response: 权限响应
  // - control_cancel_request: 取消请求
  // - keep_alive: 心跳
  // - auth_status: 认证状态
  
  // 重连策略
  private reconnect(): void {
    if (this.permanentCloseCodes.has(this.lastCloseCode)) {
      return;  // 永久关闭，不重连
    }
    
    const delay = Math.min(
      this.baseDelay * Math.pow(2, this.reconnectAttempts),
      this.maxDelay,
    );
    
    setTimeout(() => this.connect(), delay);
    this.reconnectAttempts++;
  }
}
```

### 14.4 权限桥接机制

```typescript
// remote/remotePermissionBridge.ts
export function createSyntheticAssistantMessage(
  request: SDKControlPermissionRequest,
  requestId: string,
): AssistantMessage {
  return {
    type: 'assistant',
    message: {
      role: 'assistant',
      content: [{
        type: 'tool_use',
        id: request.tool_use_id,
        name: request.tool_name,
        input: request.input,
      }],
    },
    // 用于权限对话展示
  };
}

// 工具占位符（本地无实现时）
export function createToolStub(toolName: string): Tool {
  return {
    name: toolName,
    renderToolUseMessage: (input) => ({
      jsx: <Text>Tool: {toolName}</Text>,
    }),
    // 最小实现，仅用于权限对话
  };
}
```

---

## 十五、AI 伴侣系统 (Buddy)
