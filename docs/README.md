# Claude Code 技术架构深度解析

> **版本**: 2.1.88 | **分析日期**: 2026-04-01

## 目录

| # | 文件 | 章节 |
|---|------|------|
| 01 | [项目概述与技术栈](01-project-overview.md) | 技术栈、项目规模 |
| 02 | [核心架构设计](02-core-architecture.md) | 系统架构、入口、查询引擎 |
| 03 | [安全执行体系](03-security-system.md) | 权限系统、沙箱、风险评估 |
| 04 | [工具系统架构](04-tools-system.md) | 工具抽象层、43个工具 |
| 05 | [多Agent协作系统](05-multi-agent.md) | Agent架构、任务委派 |
| 06 | [扩展性设计](06-extension-system.md) | Hooks、插件、MCP |
| 07 | [上下文与记忆系统](07-context-memory.md) | 上下文窗口、长期记忆 |
| 08 | [性能优化策略](08-performance.md) | 启动优化、缓存 |
| 09 | [UI/UX架构](09-ui-ux.md) | 终端UI、组件层次 |
| 10 | [企业级特性](10-enterprise.md) | 策略管理、审计 |
| 11 | [API服务层深度分析](11-api-layer.md) | API调用、重试、缓存 |
| 12 | [全局状态机分析](12-global-state.md) | 状态架构、初始化 |
| 13 | [任务调度系统](13-task-scheduling.md) | 任务类型、生命周期 |
| 14 | [远程连接与通信](14-remote-connection.md) | WebSocket、权限桥接 |
| 15 | [AI伴侣系统](15-buddy-system.md) | Buddy数据模型、UI |
| 16 | [查询配置与Token预算](16-query-token.md) | QueryConfig、预算管理 |
| 17 | [输出样式系统](17-output-styles.md) | 样式加载、Markdown解析 |
| 18 | [核心架构COMPLETE补充](18-core-complete.md) | AppState、Store、Selectors |
| 19 | [43个工具完整清单](19-tools-complete.md) | 工具表格、Tool.ts接口 |
| 20 | [Beta Headers列表](20-beta-headers.md) | 所有Beta特性开关 |
| 21 | [Types类型定义](21-types.md) | 权限、Hooks、Command类型 |
| 22 | [核心竞争力总结](22-competitiveness.md) | 技术、架构、产品优势 |
| 23 | [结语](23-conclusion.md) | 总结 |

## 文档信息

- 版本：2.1.88
- 分析日期：2026-04-01
- 源码文件：4756 个（含 1884 个 TypeScript/TSX 源文件）
- 总字数：约 28000 字
- 章节数量：23 个主章节
- 架构图：15+ 个 ASCII 架构图
- 代码示例：65+ 个
- 工具清单：40 个工具完整表格

## 快速导航

### 核心架构
- [核心架构设计](02-core-architecture.md) - 系统整体设计
- [全局状态机](12-global-state.md) - 状态管理
- [核心架构补充](18-core-complete.md) - AppState、Store详情

### 安全与权限
- [安全执行体系](03-security-system.md) - 多层权限系统

### 工具系统
- [工具系统架构](04-tools-system.md) - 工具抽象层
- [43个工具清单](19-tools-complete.md) - 完整工具表格

### 扩展性
- [扩展性设计](06-extension-system.md) - Hooks、插件、MCP
- [Beta Headers](20-beta-headers.md) - 实验性功能

### 远程与协作
- [多Agent协作](05-multi-agent.md) - Agent系统
- [远程连接](14-remote-connection.md) - WebSocket、桥接
- [任务调度](13-task-scheduling.md) - 任务管理

### API与性能
- [API服务层](11-api-layer.md) - API调用细节
- [性能优化](08-performance.md) - 启动、缓存优化

### 用户体验
- [UI/UX架构](09-ui-ux.md) - 终端UI设计
- [Buddy系统](15-buddy-system.md) - AI伴侣
- [输出样式](17-output-styles.md) - 样式系统

### 类型定义
- [Types类型](21-types.md) - 完整类型清单
