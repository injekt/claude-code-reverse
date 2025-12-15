# Claude Code 逆向分析文档

** [文档站](https://claude-code-reverse.vercel.app/) **

## 项目概述

本项目对 Claude Code (v2.0.62) 进行逆向工程分析，提取其核心架构、Agent 逻辑和精华设计模式。

## 文档目录

| 文档 | 内容 |
|-----|------|
| [01-project-structure.md](./01-project-structure.md) | 工程结构分析 - 目录结构、依赖、模块架构 |
| [02-agent-loop.md](./02-agent-loop.md) | Agent 核心循环 - 主循环、消息处理、子代理系统 |
| [03-tool-system.md](./03-tool-system.md) | 工具系统详解 - 所有工具的接口、实现、权限 |
| [04-highlights.md](./04-highlights.md) | 精华总结 - 最值得学习的设计模式和实现细节 |
| [05-prompts-catalog.md](./05-prompts-catalog.md) | 提示词完整目录 - 所有提示词原文与中文翻译 |
| [06-prompts-analysis.md](./06-prompts-analysis.md) | 提示词精髓解析 - 设计哲学与工程智慧 |
| [07-network-requests.md](./07-network-requests.md) | 网络请求分析 - API 端点、请求响应、认证流程 |
| [08-sdk-tools-analysis.md](./08-sdk-tools-analysis.md) | SDK 工具类型分析 - 工具输入参数的完整定义 |

## 快速导航

### 想了解整体架构？
→ [01-project-structure.md](./01-project-structure.md)

### 想了解 AI 如何与工具交互？
→ [02-agent-loop.md](./02-agent-loop.md)

### 想了解各个工具如何实现？
→ [03-tool-system.md](./03-tool-system.md)

### 想快速获取核心洞察？
→ [04-highlights.md](./04-highlights.md)

### 想了解所有提示词？
→ [05-prompts-catalog.md](./05-prompts-catalog.md)

### 想深入理解提示词设计？
→ [06-prompts-analysis.md](./06-prompts-analysis.md)

### 想了解网络请求和 API 交互？
→ [07-network-requests.md](./07-network-requests.md)

## 关键发现

### 核心架构
```
CLI → Agent Loop → Tool System → Permission → API
```

### 工具列表 (19个)
- **文件操作**: Read, Write, Edit, Glob, Grep
- **终端操作**: Bash, BashOutput, KillShell
- **子代理**: Task, AgentOutput
- **网络**: WebFetch, WebSearch
- **交互规划**: TodoWrite, AskUserQuestion, EnterPlanMode, ExitPlanMode
- **其他**: NotebookEdit, Skill, SlashCommand, MCP

### 子代理类型
- `general-purpose` - 通用研究任务
- `Explore` - 快速代码库探索
- `Plan` - 软件架构规划（只读）
- `claude-code-guide` - 文档查询专家
- `statusline-setup` - 状态行配置

### 提示词体系
- **63 个提示词**，分 15 大类
- 分层架构：身份 → 行为 → 工具 → 领域 → 上下文

### 技术栈
- Node.js >= 18 (ES Module)
- Anthropic SDK
- Tree-sitter (代码解析)
- Ripgrep (文本搜索)
- Sharp (图像处理)

## 分析方法

1. **静态分析**: 分析 cli.js 混淆代码，提取字符串常量、函数结构
2. **类型分析**: 分析 sdk-tools.d.ts 获取完整的工具接口定义
3. **模式识别**: 识别代码中的设计模式和架构风格
4. **行为推断**: 根据代码结构推断运行时行为

## 声明

本分析仅用于学习和研究目的。Claude Code 的版权属于 Anthropic PBC。

## 相关资源

- [Claude Code 官方文档](https://docs.anthropic.com/claude-code)
- [Anthropic API 文档](https://docs.anthropic.com/api)
- [Model Context Protocol](https://modelcontextprotocol.io)
