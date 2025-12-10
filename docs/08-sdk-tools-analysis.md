# sdk-tools.d.ts 深度分析

## 概述

`sdk-tools.d.ts` 是 Claude Code 的 TypeScript 类型定义文件，定义了所有工具的输入参数结构。

### 文件元信息

```typescript
// 自动生成，源自 JSON Schema
// 使用 json-schema-to-typescript 工具生成
```

**文件大小**: ~65KB
**定义的接口**: 20 个
**总行数**: 1506 行

---

## 作用

### 1. API 契约定义

这个文件定义了 Claude 模型调用工具时的**输入参数结构**。当 Claude 返回 `tool_use` 类型的响应时，`input` 字段必须符合这里定义的类型。

```
Claude API Response
       │
       ▼
┌─────────────────────────────┐
│  {                          │
│    "type": "tool_use",      │
│    "name": "Bash",          │
│    "input": BashInput  ◄────┼── 必须符合类型定义
│  }                          │
└─────────────────────────────┘
```

### 2. 类型安全

- 确保工具调用参数的类型正确性
- IDE 可以提供自动补全和类型检查
- 防止运行时因参数类型错误导致的崩溃

### 3. 文档作用

每个字段都有详细的 JSDoc 注释，说明：
- 参数用途
- 约束条件
- 使用示例

---

## 工具输入类型总览

```typescript
export type ToolInputSchemas =
  | AgentInput          // Task 工具 - 启动子代理
  | BashInput           // Bash 工具 - 执行命令
  | BashOutputInput     // BashOutput - 读取后台命令输出
  | ExitPlanModeInput   // 退出计划模式
  | FileEditInput       // Edit 工具 - 编辑文件
  | FileReadInput       // Read 工具 - 读取文件
  | FileWriteInput      // Write 工具 - 写入文件
  | GlobInput           // Glob 工具 - 文件匹配
  | GrepInput           // Grep 工具 - 内容搜索
  | KillShellInput      // 终止后台命令
  | ListMcpResourcesInput   // 列出 MCP 资源
  | McpInput            // MCP 通用输入
  | NotebookEditInput   // Jupyter 编辑
  | ReadMcpResourceInput    // 读取 MCP 资源
  | TodoWriteInput      // 任务列表管理
  | WebFetchInput       // 网页获取
  | WebSearchInput      // 网页搜索
  | AskUserQuestionInput    // 向用户提问
  | AgentOutputInput;   // 获取子代理输出
```

---

## 各工具详细分析

### 1. AgentInput (Task 工具)

启动子代理执行复杂任务。

```typescript
interface AgentInput {
  description: string;      // 必填: 3-5 词的任务描述
  prompt: string;           // 必填: 详细的任务说明
  subagent_type: string;    // 必填: 子代理类型
  model?: "sonnet" | "opus" | "haiku";  // 可选: 指定模型
  resume?: string;          // 可选: 恢复之前的代理 (agent ID)
  run_in_background?: boolean;  // 可选: 后台运行
}
```

**关键点**:
- `subagent_type` 决定子代理的能力（如 Explore、Plan、general-purpose）
- `resume` 允许继续之前中断的任务
- 后台运行配合 `AgentOutputInput` 使用

---

### 2. BashInput

执行 Shell 命令。

```typescript
interface BashInput {
  command: string;          // 必填: 要执行的命令
  timeout?: number;         // 可选: 超时毫秒数 (最大 600000)
  description?: string;     // 可选: 命令描述 (5-10 词)
  run_in_background?: boolean;  // 可选: 后台运行
  dangerouslyDisableSandbox?: boolean;  // 可选: 禁用沙箱 (危险)
}
```

**关键点**:
- `timeout` 最大 10 分钟
- `description` 要求用主动语态，如 "List files in current directory"
- `dangerouslyDisableSandbox` 用于需要绕过沙箱的特殊情况

---

### 3. BashOutputInput

读取后台命令的输出。

```typescript
interface BashOutputInput {
  bash_id: string;      // 必填: 后台 shell 的 ID
  filter?: string;      // 可选: 正则表达式过滤输出行
}
```

**关键点**:
- 配合 `BashInput.run_in_background` 使用
- `filter` 过滤后，不匹配的行将永久丢失

---

### 4. FileEditInput (Edit 工具)

精确字符串替换编辑文件。

```typescript
interface FileEditInput {
  file_path: string;    // 必填: 绝对路径
  old_string: string;   // 必填: 要替换的文本
  new_string: string;   // 必填: 替换后的文本 (必须不同于 old_string)
  replace_all?: boolean;    // 可选: 替换所有出现 (默认 false)
}
```

**关键点**:
- 使用精确字符串匹配，不是正则
- `old_string` 必须在文件中唯一（除非用 `replace_all`）
- 编辑前必须先用 Read 工具读取文件

---

### 5. FileReadInput (Read 工具)

读取文件内容。

```typescript
interface FileReadInput {
  file_path: string;    // 必填: 绝对路径
  offset?: number;      // 可选: 起始行号
  limit?: number;       // 可选: 读取行数
}
```

**关键点**:
- 支持分页读取大文件
- 可以读取图片、PDF、Jupyter notebook

---

### 6. FileWriteInput (Write 工具)

创建或覆盖文件。

```typescript
interface FileWriteInput {
  file_path: string;    // 必填: 绝对路径
  content: string;      // 必填: 文件内容
}
```

**关键点**:
- 会覆盖已存在的文件
- 写入前必须先读取已存在的文件

---

### 7. GlobInput

文件模式匹配。

```typescript
interface GlobInput {
  pattern: string;      // 必填: glob 模式 (如 "**/*.js")
  path?: string;        // 可选: 搜索目录 (默认当前目录)
}
```

**关键点**:
- 支持标准 glob 语法
- 结果按修改时间排序

---

### 8. GrepInput

内容搜索，基于 ripgrep。

```typescript
interface GrepInput {
  pattern: string;      // 必填: 正则表达式模式
  path?: string;        // 可选: 搜索路径
  glob?: string;        // 可选: 文件过滤 (如 "*.js")
  output_mode?: "content" | "files_with_matches" | "count";
  "-B"?: number;        // 可选: 匹配前显示的行数
  "-A"?: number;        // 可选: 匹配后显示的行数
  "-C"?: number;        // 可选: 匹配前后显示的行数
  "-n"?: boolean;       // 可选: 显示行号 (默认 true)
  "-i"?: boolean;       // 可选: 忽略大小写
  type?: string;        // 可选: 文件类型 (js, py, rust 等)
  head_limit?: number;  // 可选: 限制输出条目数
  offset?: number;      // 可选: 跳过前 N 条
  multiline?: boolean;  // 可选: 多行匹配模式
}
```

**关键点**:
- 直接映射到 ripgrep 参数
- `output_mode` 默认为 `files_with_matches`（只返回文件名）
- 支持分页 (`offset` + `head_limit`)

---

### 9. KillShellInput

终止后台 Shell。

```typescript
interface KillShellInput {
  shell_id: string;     // 必填: 要终止的 shell ID
}
```

---

### 10. NotebookEditInput

编辑 Jupyter Notebook。

```typescript
interface NotebookEditInput {
  notebook_path: string;    // 必填: .ipynb 文件的绝对路径
  cell_id?: string;         // 可选: 要编辑的 cell ID
  new_source: string;       // 必填: 新的 cell 内容
  cell_type?: "code" | "markdown";  // 可选: cell 类型
  edit_mode?: "replace" | "insert" | "delete";  // 可选: 编辑模式
}
```

**关键点**:
- `insert` 模式时 `cell_type` 是必需的
- `cell_id` 用于定位 cell（insert 时表示在其后插入）

---

### 11. MCP 相关

```typescript
// 列出 MCP 资源
interface ListMcpResourcesInput {
  server?: string;      // 可选: 过滤特定服务器
}

// 读取 MCP 资源
interface ReadMcpResourceInput {
  server: string;       // 必填: MCP 服务器名
  uri: string;          // 必填: 资源 URI
}

// 通用 MCP 输入 (动态)
interface McpInput {
  [k: string]: unknown;
}
```

---

### 12. TodoWriteInput

管理任务列表。

```typescript
interface TodoWriteInput {
  todos: {
    content: string;    // 任务内容 (祈使句，如 "Fix bug")
    status: "pending" | "in_progress" | "completed";
    activeForm: string; // 进行时形式 (如 "Fixing bug")
  }[];
}
```

**关键点**:
- 每个任务需要两种形式：祈使句 (content) 和进行时 (activeForm)
- 同时只能有一个任务处于 `in_progress` 状态

---

### 13. WebFetchInput

获取网页内容。

```typescript
interface WebFetchInput {
  url: string;          // 必填: 要获取的 URL
  prompt: string;       // 必填: 对内容的处理提示
}
```

**关键点**:
- 自动将 HTML 转换为 Markdown
- 使用小模型处理内容，根据 `prompt` 提取信息

---

### 14. WebSearchInput

网页搜索。

```typescript
interface WebSearchInput {
  query: string;            // 必填: 搜索查询
  allowed_domains?: string[];   // 可选: 只搜索这些域名
  blocked_domains?: string[];   // 可选: 排除这些域名
}
```

---

### 15. AskUserQuestionInput

向用户提问（最复杂的类型）。

```typescript
interface AskUserQuestionInput {
  questions: Question[];    // 1-4 个问题
  answers?: Record<string, string>;  // 用户的回答
}

// 每个问题的结构
interface Question {
  question: string;     // 完整问题，以 ? 结尾
  header: string;       // 短标签 (最多 12 字符)
  options: Option[];    // 2-4 个选项
  multiSelect: boolean; // 是否允许多选
}

// 每个选项
interface Option {
  label: string;        // 显示文本 (1-5 词)
  description: string;  // 选项说明
}
```

**关键点**:
- 最多 4 个问题，每个问题 2-4 个选项
- 系统会自动添加 "Other" 选项
- `header` 显示为标签/芯片，要极简

**类型定义的冗余**:
文件中 `AskUserQuestionInput` 占了 ~1200 行，因为 TypeScript 的元组类型要求显式列出所有可能的长度组合 (1-4 问题 × 2-4 选项)。

---

### 16. AgentOutputInput

获取后台子代理的输出。

```typescript
interface AgentOutputInput {
  agentId: string;      // 必填: 代理 ID
  block?: boolean;      // 可选: 是否阻塞等待 (默认 true)
  wait_up_to?: number;  // 可选: 最大等待秒数 (最大 300)
}
```

---

### 17. ExitPlanModeInput

退出计划模式。

```typescript
interface ExitPlanModeInput {
  launchSwarm?: boolean;    // 可选: 是否启动 Swarm
  teammateCount?: number;   // 可选: Swarm 队友数量
  [k: string]: unknown;     // 允许扩展属性
}
```

**关键点**:
- 计划模式下 Claude 只能读不能写
- `launchSwarm` 可以启动多代理协作

---

## 缺失的类型

sdk-tools.d.ts 中**未定义**的工具（在 cli.js 中存在）:

| 工具 | 说明 |
|-----|------|
| `Skill` | 执行预定义技能 |
| `SlashCommand` | 执行斜杠命令 |
| `EnterPlanMode` | 进入计划模式 |
| `StructuredOutput` | 结构化输出 |

这些可能是内部工具，或通过其他机制定义。

---

## 类型设计模式

### 1. 必填 vs 可选

- 核心参数为必填 (无 `?`)
- 行为修饰参数为可选 (有 `?`)

### 2. 枚举约束

```typescript
model?: "sonnet" | "opus" | "haiku";
status: "pending" | "in_progress" | "completed";
output_mode?: "content" | "files_with_matches" | "count";
```

### 3. 开放扩展

```typescript
interface McpInput {
  [k: string]: unknown;  // 允许任意属性
}

interface ExitPlanModeInput {
  // ...
  [k: string]: unknown;  // 允许扩展
}
```

---

## 与 cli.js 的关系

```
┌─────────────────────────────────────────────────────────────┐
│                      源码构建时                              │
│                                                             │
│  tool-schemas.json ──┬──► cli.js (工具验证逻辑)            │
│                      │                                      │
│                      └──► sdk-tools.d.ts (类型定义)         │
│                           (json-schema-to-typescript)       │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                        运行时                                │
│                                                             │
│  Claude API Response                                        │
│       │                                                     │
│       ▼                                                     │
│  tool_use.input ──► 验证 (cli.js 内部逻辑)                 │
│       │                                                     │
│       ▼                                                     │
│  执行工具                                                   │
└─────────────────────────────────────────────────────────────┘
```

sdk-tools.d.ts 主要用于：
1. **开发时** - TypeScript 类型检查
2. **文档** - 说明工具参数结构
3. **外部集成** - 第三方工具可以导入这些类型

---

## 总结

| 特性 | 说明 |
|-----|------|
| 生成方式 | 从 JSON Schema 自动生成 |
| 工具数量 | 19 个工具输入类型 |
| 最复杂类型 | AskUserQuestionInput (~1200 行) |
| 最简单类型 | KillShellInput (1 个字段) |
| 设计风格 | 严格类型 + JSDoc 注释 |
