# Claude Code 工程结构分析

## 概述

Claude Code 是 Anthropic 官方的 CLI 工具，版本 2.0.62。整个项目被打包成单个混淆的 JavaScript 文件，但通过逆向分析可以还原其核心架构。

## 目录结构

```
claude-code/
├── cli.js              # 主入口文件 (~11MB, 4836行压缩代码)
├── package.json        # NPM 包配置
├── LICENSE.md          # 许可证
├── README.md           # 说明文档
├── sdk-tools.d.ts      # TypeScript 类型定义 (65KB)
├── tree-sitter.wasm    # 通用语法解析器 (~200KB)
├── tree-sitter-bash.wasm # Bash 语法解析器 (~1.4MB)
├── node_modules/       # 运行时依赖
│   └── @img/sharp-*/   # 图像处理库 (用于截图分析)
└── vendor/
    └── ripgrep/        # 内置的 ripgrep 搜索工具
```

## 核心技术栈

### 运行时依赖
- **Node.js >= 18.0.0**: 运行环境要求
- **ES Module**: 使用现代模块系统
- **Sharp**: 图像处理 (截图、图片分析)
- **Tree-sitter**: 代码语法解析
- **Ripgrep**: 高性能文本搜索

### 内嵌库 (打包在 cli.js 中)
- **Lodash**: 工具函数库 (memoize, isEqual, get 等)
- **Zod**: Schema 验证
- **Anthropic SDK**: Claude API 客户端 (对话请求)
- **Axios**: HTTP 客户端 (OAuth、遥测等非对话请求)
- **RxJS**: 响应式编程 (处理流)
- **Statsig SDK**: 特性开关
- **Sentry SDK**: 错误监控
- **GrowthBook**: A/B 测试

## 模块架构 (推断)

```
┌─────────────────────────────────────────────────────────────┐
│                        CLI Layer                             │
│  (参数解析、会话管理、用户交互)                               │
├─────────────────────────────────────────────────────────────┤
│                      Agent Core                              │
│  (主循环、消息处理、工具调度)                                 │
├─────────────────────────────────────────────────────────────┤
│                     Tool System                              │
│  ┌─────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌─────────┐            │
│  │Bash │ │ Read │ │ Edit │ │ Glob │ │  Grep   │            │
│  └─────┘ └──────┘ └──────┘ └──────┘ └─────────┘            │
│  ┌─────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌─────────┐            │
│  │Write│ │ Task │ │ MCP  │ │WebFetch│ │TodoWrite│           │
│  └─────┘ └──────┘ └──────┘ └──────┘ └─────────┘            │
├─────────────────────────────────────────────────────────────┤
│                   Permission System                          │
│  (权限控制、沙箱、用户授权)                                   │
├─────────────────────────────────────────────────────────────┤
│                    Hooks System                              │
│  (PreToolUse, PostToolUse, Notification 等)                  │
├─────────────────────────────────────────────────────────────┤
│                   API Communication                          │
│  (Anthropic API, 流式响应, 重试机制)                         │
└─────────────────────────────────────────────────────────────┘
```

## 关键文件分析

### cli.js

主入口文件，包含所有业务逻辑。代码特点：
- 高度压缩和混淆
- 变量名被替换为短名称 (如 `XO9`, `GO9`, `QK1`)
- 函数使用工厂模式懒加载
- 字符串常量保留可读性

### sdk-tools.d.ts

TypeScript 类型定义文件，定义了所有工具的输入参数类型：

```typescript
export type ToolInputSchemas =
  | AgentInput        // 子代理任务
  | BashInput         // Shell 命令执行
  | BashOutputInput   // 后台命令输出
  | ExitPlanModeInput // 退出计划模式
  | FileEditInput     // 文件编辑
  | FileReadInput     // 文件读取
  | FileWriteInput    // 文件写入
  | GlobInput         // 文件匹配
  | GrepInput         // 内容搜索
  | KillShellInput    // 终止后台命令
  | McpInput          // MCP 协议
  | NotebookEditInput // Jupyter 编辑
  | TodoWriteInput    // 任务列表
  | WebFetchInput     // 网页获取
  | WebSearchInput    // 网页搜索
  | AskUserQuestionInput // 用户提问
  | AgentOutputInput  // 获取子代理输出
```

### package.json

```json
{
  "name": "@anthropic-ai/claude-code",
  "version": "2.0.62",
  "bin": { "claude": "cli.js" },
  "engines": { "node": ">=18.0.0" },
  "type": "module",
  "optionalDependencies": {
    "@img/sharp-*": "^0.33.5"  // 多平台图像处理
  }
}
```

## API 端点

详细的网络请求分析见 [07-network-requests.md](./07-network-requests.md)。

### 核心 API (api.anthropic.com)
- `/v1/messages` - AI 对话 (流式)
- `/v1/messages/count_tokens` - Token 计数
- `/api/claude_cli_profile` - CLI 配置
- `/api/oauth/profile` - OAuth 用户信息
- `/api/oauth/claude_cli/create_api_key` - 创建 API Key
- `/api/oauth/claude_cli/roles` - 用户角色
- `/api/claude_code/metrics` - 使用指标
- `/api/claude_cli_feedback` - 反馈提交
- `/api/event_logging/batch` - 事件日志
- `/api/organization/{id}/claude_code_sonnet_1m_access` - 1M 权限检查

### OAuth (console.anthropic.com)
- `/oauth/authorize` - 授权入口
- `/v1/oauth/token` - Token 交换/刷新

### 遥测与监控
- `statsig.anthropic.com/v1/` - 特性开关
- `statsigapi.net/v1/sdk_exception` - 异常上报
- `sentry.io` - 错误监控

### 版本检查
- `downloads.claude.ai/claude-code-releases` - 版本列表

## 支持的模型

从代码中提取的模型标识：
- `claude-haiku-4-5` - Haiku 4.5
- `claude-3-5-haiku` - Claude 3.5 Haiku
- `claude-3-5-sonnet` - Claude 3.5 Sonnet
- `claude-3-7-sonnet` - Claude 3.7 Sonnet
- `claude-opus-4` - Claude Opus 4
- `claude-opus-4-1` - Claude Opus 4.1
- `claude-sonnet-4` - Claude Sonnet 4
- `claude-sonnet-4-5` - Claude Sonnet 4.5

## 环境变量

关键环境变量：
- `CLAUDE_CONFIG_DIR` - 配置目录
- `ANTHROPIC_API_KEY` - API 密钥
- `AWS_REGION` / `AWS_DEFAULT_REGION` - AWS 区域
- `CLOUD_ML_REGION` - Vertex AI 区域
- `CLAUDE_BASH_MAINTAIN_PROJECT_WORKING_DIR` - Bash 工作目录
- `CLAUDE_CODE_MAX_OUTPUT_TOKENS` - 最大输出 token
- `VERTEX_REGION_CLAUDE_*` - 各模型的 Vertex 区域

## 配置文件

配置文件路径：
- `~/.claude/settings.json` - 全局设置
- `.claude/settings.json` - 项目设置
- `.claude/settings.local.json` - 本地设置
- `CLAUDE.md` - 项目级 AI 指令文件（由 /init 命令生成）

## 斜杠命令

内置的斜杠命令：
- `/help` - 获取帮助信息
- `/clear` - 清除对话历史
- `/compact` - 压缩上下文（可选参数指定保留的指令）
- `/init` - 分析代码库并创建 CLAUDE.md 文件
- `/review-pr [PR号]` - 审查 Pull Request
- `/pr-comments` - 获取 PR 评论
- `/cost` - 显示当前会话费用
- `/status` - 显示会话状态
- `/config` - 配置设置
- `/output-style [风格]` - 切换输出风格（Default/Explanatory/Learning）

## 输出风格 (Persona)

支持三种输出风格：
1. **Default** - 默认风格，简洁高效
2. **Explanatory** - 教育性模式，在代码前后提供 `★ Insight` 说明
3. **Learning** - 学习模式，邀请用户参与代码编写决策

## 下一步

- [02-agent-loop.md](./02-agent-loop.md) - Agent 核心循环分析
- [03-tool-system.md](./03-tool-system.md) - 工具系统深度解析
- [04-highlights.md](./04-highlights.md) - 精华部分总结
