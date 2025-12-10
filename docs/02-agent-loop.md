# Claude Code Agent 核心循环分析

## 概述

Claude Code 的核心是一个 Agent 循环，它负责与 Claude API 交互、处理工具调用、管理对话上下文。本文档深入分析这一核心机制。

详细的网络请求分析见 [07-network-requests.md](./07-network-requests.md)。

## Agent 循环架构

```
┌────────────────────────────────────────────────────────────────┐
│                     Agent Main Loop                             │
│                                                                 │
│  ┌─────────────┐    ┌──────────────┐    ┌─────────────────┐   │
│  │ User Input  │───▶│ Build Prompt │───▶│ Call Claude API │   │
│  └─────────────┘    └──────────────┘    └────────┬────────┘   │
│                                                   │             │
│         ┌─────────────────────────────────────────┘             │
│         ▼                                                       │
│  ┌──────────────┐                                               │
│  │ Parse Response│                                              │
│  └──────┬───────┘                                               │
│         │                                                       │
│         ▼                                                       │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                   Response Type?                          │  │
│  │  ┌────────────┐  ┌────────────┐  ┌──────────────────┐   │  │
│  │  │   Text     │  │  Tool Use  │  │  End Turn        │   │  │
│  │  │  Output    │  │  Request   │  │  (stop_reason)   │   │  │
│  │  └─────┬──────┘  └─────┬──────┘  └────────┬─────────┘   │  │
│  └────────┼───────────────┼──────────────────┼─────────────┘  │
│           │               │                  │                 │
│           ▼               ▼                  ▼                 │
│      Display to      Execute Tool       Return to User         │
│        User          & Get Result                              │
│                           │                                    │
│                           ▼                                    │
│                   ┌──────────────┐                             │
│                   │ Add Tool     │                             │
│                   │ Result to    │──────────────┐              │
│                   │ Messages     │              │              │
│                   └──────────────┘              │              │
│                                                 │              │
│  ◀──────────────────────────────────────────────┘              │
│              (Continue Loop)                                    │
└────────────────────────────────────────────────────────────────┘
```

## 核心代码结构 (反混淆)

### 消息处理流程

从混淆代码中提取的核心逻辑：

```javascript
// 伪代码 - 基于逆向分析还原
class AgentLoop {
  constructor(config) {
    this.messages = [];
    this.controller = new AbortController();
    this.client = new AnthropicClient(config);
  }

  async run(userMessage) {
    // 添加用户消息
    this.messages.push({
      role: 'user',
      content: userMessage
    });

    // 主循环
    while (true) {
      // 调用 Claude API
      const response = await this.client.messages.create({
        model: this.model,
        messages: this.messages,
        tools: this.getTools(),
        system: this.getSystemPrompt(),
        stream: true  // 使用流式响应
      });

      // 处理响应
      const result = await this.processResponse(response);

      // 检查停止条件
      if (result.stop_reason === 'end_turn') {
        break;
      }

      // 如果有工具调用，执行并继续
      if (result.stop_reason === 'tool_use') {
        const toolResults = await this.executeTools(result.tool_calls);
        this.messages.push({
          role: 'assistant',
          content: result.content
        });
        this.messages.push({
          role: 'user',
          content: toolResults
        });
      }
    }
  }
}
```

### 工具调用检测

```javascript
// 从源码提取的工具类型检测
function isToolUse(block) {
  return block.type === "tool_use" ||
         block.type === "server_tool_use" ||
         block.type === "mcp_tool_use";
}
```

### 流式响应处理

```javascript
// 流式消息事件处理
switch (event.type) {
  case "message_start":
    // 初始化消息容器，获取初始 usage
    message.usage = event.message.usage;
    break;

  case "content_block_start":
    // 新的内容块开始 (text/tool_use/thinking)
    break;

  case "content_block_delta":
    // 增量内容更新
    break;

  case "content_block_stop":
    // 内容块结束
    break;

  case "thinking_delta":
    // 思考过程增量（如启用 thinking 模式）
    break;

  case "message_delta":
    // 消息级别更新
    message.stop_reason = event.delta.stop_reason;
    message.usage.output_tokens = event.usage.output_tokens;
    message.usage.cache_read_input_tokens = event.usage.cache_read_input_tokens;
    break;

  case "message_stop":
    // 消息完成
    break;
}
```

## stop_reason 类型

| 值 | 含义 | 后续操作 |
|---|---|---|
| `end_turn` | 正常结束 | 结束循环，返回给用户 |
| `tool_use` | 需要执行工具 | 执行工具，将结果加入消息，继续循环 |
| `max_tokens` | 达到 token 上限 | 可能需要截断或继续 |
| `stop_sequence` | 遇到停止序列 | 根据上下文处理 |

## 消息格式

### 用户消息

```json
{
  "role": "user",
  "content": [
    {
      "type": "text",
      "text": "帮我创建一个文件"
    }
  ]
}
```

### 助手消息 (包含工具调用)

```json
{
  "role": "assistant",
  "content": [
    {
      "type": "text",
      "text": "我来帮你创建文件。"
    },
    {
      "type": "tool_use",
      "id": "toolu_xxx",
      "name": "Write",
      "input": {
        "file_path": "/path/to/file.txt",
        "content": "文件内容"
      }
    }
  ]
}
```

### 工具结果消息

```json
{
  "role": "user",
  "content": [
    {
      "type": "tool_result",
      "tool_use_id": "toolu_xxx",
      "content": "File created successfully"
    }
  ]
}
```

## 子代理 (Subagent) 系统

Claude Code 支持启动子代理来处理复杂任务：

```javascript
// 子代理类型
const subagentTypes = [
  'general-purpose',    // 通用研究任务，搜索代码，执行多步骤任务
  'Explore',            // 快速代码库探索，支持 quick/medium/very thorough 三档
  'Plan',               // 软件架构规划（只读模式，不能写文件）
  'claude-code-guide',  // 文档查询专家（Claude Code/Agent SDK/API文档）
  'statusline-setup'    // 状态行配置代理
];

// 子代理 @ 提及检测
const agentMention = prompt.match(/^@agent-(\w+)/);
```

### 子代理执行流程

```
┌─────────────────┐
│   Main Agent    │
└────────┬────────┘
         │
         │ Task tool call
         ▼
┌─────────────────┐
│  Spawn Subagent │
│  (new context)  │
└────────┬────────┘
         │
         │ Independent loop
         ▼
┌─────────────────┐
│ Subagent works  │
│   autonomously  │
└────────┬────────┘
         │
         │ Return result
         ▼
┌─────────────────┐
│   Main Agent    │
│ continues with  │
│    result       │
└─────────────────┘
```

## Hooks 系统

支持的 Hook 事件：

```javascript
const hookEvents = [
  'PreToolUse',         // 工具执行前
  'PostToolUse',        // 工具执行后
  'PostToolUseFailure', // 工具执行失败后
  'Notification',       // 通知
  'UserPromptSubmit',   // 用户提交提示
  'SessionStart',       // 会话开始
  'SessionEnd',         // 会话结束
  'Stop',               // 停止
  'SubagentStart',      // 子代理启动
  'SubagentStop',       // 子代理停止
  'PreCompact',         // 上下文压缩前
  'PermissionRequest'   // 权限请求
];
```

## Context 管理

### Token 计数与使用统计

```javascript
// 使用量追踪 (从 API 响应获取)
{
  input_tokens: number,              // 输入 token
  output_tokens: number,             // 输出 token
  cache_read_input_tokens: number,   // 缓存读取 token
  cache_creation_input_tokens: number, // 缓存创建 token
  // 服务端工具使用
  server_tool_use: {
    web_search_requests: number      // 网页搜索次数
  }
}
```

### Context Window 大小

```javascript
function getContextWindow(model) {
  // 特定模型有 1M context
  if (model.includes("[1m]")) {
    return 1000000;
  }
  // 默认 200K
  return 200000;
}
```

## 错误处理与重试

```javascript
// API 调用包含重试逻辑
const response = await this.client.messages.create({
  // ... params
}, {
  signal: this.controller.signal,
  // 自动重试配置由 SDK 处理
});
```

### 错误类型处理

| 错误类型 | HTTP 状态 | 处理方式 |
|---------|----------|---------|
| `authentication_error` | 401 | API Key 无效，提示重新认证 |
| `permission_error` | 403 | 无权限，检查组织策略 |
| `rate_limit_error` | 429 | 显示警告，等待重置 |
| `overloaded_error` | 529 | 服务繁忙，自动重试 |
| 网络错误 | - | 根据 maxRetries 重试 |

## 性能优化

1. **流式响应**: 实时显示输出，提升用户体验
2. **缓存机制**: 利用 Claude API 的 prompt caching (`cache_control: {type: "ephemeral"}`)
3. **并行工具执行**: 多个独立工具可并行执行
4. **增量更新**: 只传输变化的内容
5. **速率限制监控**: 通过响应头 `anthropic-ratelimit-*` 提前预警

## 关键常量

```javascript
const COMPLETE = "C";  // 完成状态
const ERROR = "E";     // 错误状态
const NEXT = "N";      // 继续状态

const SDK_VERSION = "3.12.1";  // Anthropic SDK 版本
```

## Swarm 多代理协作模式

Claude Code 支持 Swarm 模式，可以启动多个子代理协作完成复杂任务：

```javascript
// Swarm 团队操作
const swarmOperations = {
  spawnTeam: "创建新团队",
  spawn: "生成队友（worker 类型）",
  assignTask: "分配任务给队友",
  // 领导者/协调者负责汇总结果
};

// 在 ExitPlanMode 中可启动 Swarm
interface ExitPlanModeInput {
  launchSwarm?: boolean;      // 是否启动 Swarm
  teammateCount?: number;     // 队友数量
}
```

## 会话质量监控

系统会自动检测用户挫败感和特定请求：

```javascript
// 用户状态分析
{
  isFrustrated: boolean,  // 是否感到沮丧（重复纠正、负面语言）
  prRequest: boolean      // 是否请求创建 PR
}
```

## 辅助子系统

### 会话标题生成
- 自动生成简洁标题（50字符内）
- 生成对应的 git 分支名

### 提示建议生成
- 任务完成后自动建议下一步
- 3-8 词的简短建议（如 "run the tests"）

### 话题检测
- 用于更新终端标题
- 提取当前工作主题

## 下一步

- [03-tool-system.md](./03-tool-system.md) - 工具系统详解
- [04-highlights.md](./04-highlights.md) - 精华总结
