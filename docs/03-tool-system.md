# Claude Code 工具系统深度解析

## 概述

Claude Code 的工具系统是其核心能力的基础，允许 AI 与本地文件系统、终端、网络等进行交互。本文档详细分析工具系统的设计与实现。

## 工具分类

### 1. 文件操作工具

| 工具 | 功能 | 关键参数 |
|-----|------|---------|
| **Read** | 读取文件内容 | `file_path`, `offset`, `limit` |
| **Write** | 写入/创建文件 | `file_path`, `content` |
| **Edit** | 编辑文件 (精确替换) | `file_path`, `old_string`, `new_string`, `replace_all` |
| **Glob** | 文件模式匹配 | `pattern`, `path` |
| **Grep** | 内容搜索 | `pattern`, `path`, `glob`, `output_mode` |

### 2. 终端工具

| 工具 | 功能 | 关键参数 |
|-----|------|---------|
| **Bash** | 执行 Shell 命令 | `command`, `timeout`, `run_in_background` |
| **BashOutput** | 获取后台命令输出 | `bash_id`, `filter` |
| **KillShell** | 终止后台进程 | `shell_id` |

### 3. 代理工具

| 工具 | 功能 | 关键参数 |
|-----|------|---------|
| **Task** | 启动子代理 | `prompt`, `subagent_type`, `model`, `run_in_background` |
| **AgentOutput** | 获取子代理输出 | `agentId`, `block`, `wait_up_to` |

### 4. 网络工具

| 工具 | 功能 | 关键参数 |
|-----|------|---------|
| **WebFetch** | 获取网页内容 | `url`, `prompt` |
| **WebSearch** | 网络搜索 | `query`, `allowed_domains`, `blocked_domains` |

### 5. 交互与规划工具

| 工具 | 功能 | 关键参数 |
|-----|------|---------|
| **TodoWrite** | 任务管理 | `todos[]` |
| **AskUserQuestion** | 询问用户 | `questions[]` |
| **EnterPlanMode** | 进入规划模式 | (无参数) |
| **ExitPlanMode** | 退出计划模式 | `launchSwarm`, `teammateCount` |

### 6. 其他工具

| 工具 | 功能 | 关键参数 |
|-----|------|---------|
| **NotebookEdit** | Jupyter 编辑 | `notebook_path`, `cell_id`, `new_source`, `edit_mode` |
| **Skill** | 执行技能插件 | `skill` |
| **SlashCommand** | 执行斜杠命令 | `command` |
| **StructuredOutput** | 结构化输出 | (动态 schema) |
| **MCP** | MCP 协议工具 | (动态) |

## 工具 Schema 定义

### Read 工具

```typescript
interface FileReadInput {
  file_path: string;    // 绝对路径
  offset?: number;      // 起始行号
  limit?: number;       // 读取行数
}
```

**特性**:
- 支持读取图片 (PNG, JPG 等) - 多模态能力
- 支持 PDF 文件
- 支持 Jupyter Notebook (.ipynb)
- 行号从 1 开始 (cat -n 格式)
- 超过 2000 字符的行会被截断

### Edit 工具

```typescript
interface FileEditInput {
  file_path: string;     // 绝对路径
  old_string: string;    // 要替换的文本
  new_string: string;    // 替换后的文本
  replace_all?: boolean; // 是否替换全部 (默认 false)
}
```

**特性**:
- 精确文本匹配替换
- 要求 `old_string` 在文件中唯一 (除非用 `replace_all`)
- 保持原始缩进格式
- 拒绝创建新文件 (优先编辑现有文件)

### Bash 工具

```typescript
interface BashInput {
  command: string;                    // 命令
  timeout?: number;                   // 超时 (ms, max 600000)
  description?: string;               // 命令描述
  run_in_background?: boolean;        // 后台运行
  dangerouslyDisableSandbox?: boolean; // 禁用沙箱
}
```

**特性**:
- 持久化 Shell 会话
- 最大输出限制 (默认 30000 字符)
- 支持后台运行长时间命令
- 沙箱保护机制

### Grep 工具

```typescript
interface GrepInput {
  pattern: string;       // 正则表达式
  path?: string;         // 搜索路径
  glob?: string;         // 文件过滤 (*.js)
  output_mode?: 'content' | 'files_with_matches' | 'count';
  '-B'?: number;         // 前置上下文行
  '-A'?: number;         // 后置上下文行
  '-C'?: number;         // 前后上下文行
  '-n'?: boolean;        // 显示行号
  '-i'?: boolean;        // 忽略大小写
  type?: string;         // 文件类型 (js, py, etc)
  head_limit?: number;   // 限制结果数
  multiline?: boolean;   // 多行模式
}
```

**特性**:
- 基于 ripgrep (rg) 实现
- 支持多种输出模式
- 支持文件类型过滤
- 支持多行匹配

### Task (Agent) 工具

```typescript
interface AgentInput {
  description: string;      // 任务简述 (3-5词)
  prompt: string;           // 完整任务描述
  subagent_type: string;    // 子代理类型
  model?: 'sonnet' | 'opus' | 'haiku';  // 模型选择
  resume?: string;          // 恢复之前的代理
  run_in_background?: boolean;  // 后台运行
}
```

**子代理类型**:
- `general-purpose`: 通用研究任务，搜索代码，执行多步骤任务
- `Explore`: 快速代码库探索（支持 quick/medium/very thorough 三档深度）
- `Plan`: 软件架构规划（只读模式，不能写文件）
- `claude-code-guide`: 文档查询专家（Claude Code/Agent SDK/API 文档）
- `statusline-setup`: 状态行配置代理

**子代理特别注意事项**:
- 子代理必须使用绝对路径（无法访问父级工作目录）
- 子代理不应使用 emoji（除非用户明确要求）
- 子代理不应主动使用 TodoWrite（避免与父级冲突）

## 工具执行流程

```
┌─────────────────────────────────────────────────────────────┐
│                    Tool Execution Flow                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Parse Tool Call                                          │
│     ┌─────────────────────────────────────────┐             │
│     │ {                                        │             │
│     │   type: "tool_use",                      │             │
│     │   id: "toolu_xxx",                       │             │
│     │   name: "Read",                          │             │
│     │   input: { file_path: "/path/file" }     │             │
│     │ }                                        │             │
│     └─────────────────────────────────────────┘             │
│                         │                                    │
│                         ▼                                    │
│  2. Permission Check                                         │
│     ┌─────────────────────────────────────────┐             │
│     │  - Check allowed/denied rules            │             │
│     │  - Apply sandbox restrictions            │             │
│     │  - Request user approval if needed       │             │
│     └─────────────────────────────────────────┘             │
│                         │                                    │
│                         ▼                                    │
│  3. Pre-Tool Hook                                            │
│     ┌─────────────────────────────────────────┐             │
│     │  Execute PreToolUse hooks                │             │
│     │  (can modify or block execution)         │             │
│     └─────────────────────────────────────────┘             │
│                         │                                    │
│                         ▼                                    │
│  4. Execute Tool                                             │
│     ┌─────────────────────────────────────────┐             │
│     │  - Validate input schema                 │             │
│     │  - Run tool-specific logic               │             │
│     │  - Capture output/errors                 │             │
│     └─────────────────────────────────────────┘             │
│                         │                                    │
│                         ▼                                    │
│  5. Post-Tool Hook                                           │
│     ┌─────────────────────────────────────────┐             │
│     │  Execute PostToolUse hooks               │             │
│     │  (or PostToolUseFailure if failed)       │             │
│     └─────────────────────────────────────────┘             │
│                         │                                    │
│                         ▼                                    │
│  6. Return Result                                            │
│     ┌─────────────────────────────────────────┐             │
│     │ {                                        │             │
│     │   type: "tool_result",                   │             │
│     │   tool_use_id: "toolu_xxx",              │             │
│     │   content: "File content here..."        │             │
│     │ }                                        │             │
│     └─────────────────────────────────────────┘             │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## 权限系统

### 权限配置

```json
// .claude/settings.json
{
  "allow": [
    "Read",
    "Glob",
    "Grep",
    "Bash(npm run *)",
    "Edit(src/**)"
  ],
  "deny": [
    "Bash(rm -rf *)",
    "Write(.env)"
  ]
}
```

### 权限规则语法

- `ToolName` - 允许/拒绝整个工具
- `ToolName(pattern)` - 基于参数模式匹配
- 支持 glob 模式

### 沙箱机制

Bash 工具的沙箱保护：
- 限制危险命令
- 文件访问限制
- 网络访问控制
- 可通过 `dangerouslyDisableSandbox` 绕过

### 命令注入检测

系统会分析 Bash 命令前缀，检测潜在的命令注入攻击：
- 检测链式命令（`&&`, `||`, `;`）
- 如果检测到注入，返回 `command_injection_detected`
- 保护用户免受恶意命令伪装

## 内置依赖

### Tree-sitter

用于代码语法解析：
- `tree-sitter.wasm` - 核心解析器
- `tree-sitter-bash.wasm` - Bash 语法

**用途**:
- 提取 Bash 命令中的文件路径
- 代码结构分析
- 安全检查

### Ripgrep

内置的高性能搜索工具：
- 位于 `vendor/ripgrep/`
- Grep 工具的底层实现
- 支持多种文件类型和正则表达式

## 工具输出格式

### 成功响应

```json
{
  "type": "tool_result",
  "tool_use_id": "toolu_xxx",
  "content": "操作成功完成的消息或数据"
}
```

### 错误响应

```json
{
  "type": "tool_result",
  "tool_use_id": "toolu_xxx",
  "content": "Error: 错误描述",
  "is_error": true
}
```

### TodoWrite 特殊响应

```json
{
  "type": "tool_result",
  "tool_use_id": "toolu_xxx",
  "content": "Todos have been modified successfully. Ensure that you continue to use the todo list to track your progress. Please proceed with the current tasks if applicable"
}
```

## MCP (Model Context Protocol)

Claude Code 支持 MCP 协议，允许连接外部工具服务器：

```typescript
// MCP 工具类型
type McpToolUse = {
  type: "mcp_tool_use",
  // ... MCP 特定字段
}

// 检测 MCP 工具调用
function isToolUse(block) {
  return block.type === "tool_use" ||
         block.type === "server_tool_use" ||
         block.type === "mcp_tool_use";
}
```

### MCP 资源操作

```typescript
interface ListMcpResourcesInput {
  server?: string;  // 服务器名称过滤
}

interface ReadMcpResourceInput {
  server: string;   // 服务器名称
  uri: string;      // 资源 URI
}
```

## 最佳实践

### 工具选择原则

1. **文件读取**: 使用 `Read` 而非 `Bash(cat)`
2. **文件搜索**: 使用 `Glob` 而非 `Bash(find)`
3. **内容搜索**: 使用 `Grep` 而非 `Bash(grep)`
4. **文件编辑**: 使用 `Edit` 而非 `Bash(sed)`
5. **文件创建**: 使用 `Write` 而非 `Bash(echo >)`

### 性能优化

1. **并行调用**: 独立工具调用可以并行执行
2. **批量操作**: 减少工具调用次数
3. **精确匹配**: 使用精确的 glob/grep 模式

### 安全考虑

1. 永远不要在命令中包含敏感信息
2. 谨慎使用 `dangerouslyDisableSandbox`
3. 审查 `allow/deny` 规则配置

## 下一步

- [04-highlights.md](./04-highlights.md) - 精华总结
