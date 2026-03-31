## 一、项目概述与技术栈

### 1.1 技术栈总览

| 类别 | 技术选型 | 说明 |
|------|----------|------|
| **运行时** | Node.js / Bun | 支持双运行时，Bun 提供更快启动 |
| **UI 框架** | React + Ink | 终端 UI 渲染，声明式组件模型 |
| **语言** | TypeScript/TSX | 完整类型系统，编译时安全 |
| **状态管理** | 自定义 Store | 类 Zustand 的外部状态管理 |
| **包管理** | npm/bun | 199 个依赖包 |
| **构建工具** | Bun Bundle | 支持 Dead Code Elimination |

### 1.2 项目规模

```
总文件数: 4756 个
源码文件: 1906 个 (.ts/.tsx)
工具实现: 43 个
命令实现: 101 个
UI 组件: 144 个
工具函数: 329 个
依赖包: 199 个
```

### 1.3 目录结构概览

```
restored-src/src/
├── main.tsx                 # CLI 入口，初始化协调
├── QueryEngine.ts           # 查询引擎，会话生命周期
├── Tool.ts                  # 工具抽象层
├── tools/                   # 43 个工具实现
│   ├── BashTool/           # Shell 命令执行
│   ├── FileEditTool/       # 文件编辑
│   ├── AgentTool/          # 多 Agent 协作
│   └── MCPTool/            # MCP 协议集成
├── commands/                # 101 个 CLI 命令
├── services/                # 服务层（API、MCP、分析）
├── utils/                   # 工具函数（git、model、auth、env 等）
├── context/                 # React Context
├── coordinator/             # 多 Agent 协调模式
├── assistant/               # 助手模式（KAIROS）
├── buddy/                   # AI 伴侣 UI
├── remote/                  # 远程会话
├── plugins/                 # 插件系统
├── skills/                  # 技能系统
├── voice/                   # 语音交互
└── vim/                     # Vim 模式
```

---
