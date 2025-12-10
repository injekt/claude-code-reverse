# Claude Code 网络请求分析

## 概述

Claude Code 的网络请求主要分为以下几类：
1. **AI 对话 API** - 与 Claude 模型交互
2. **OAuth 认证** - 用户登录和令牌管理
3. **用户账户 API** - 配置文件、设置、权限
4. **遥测监控** - 使用统计、错误上报、特性开关
5. **版本检查** - 更新检测和下载

---

## 1. AI 对话 API

### 1.1 主对话请求 (messages.stream)

**用途**: 与 Claude 模型进行流式对话，是最核心的 API 调用

**端点**: `https://api.anthropic.com/v1/messages` (流式)

**调用方式**:
```javascript
client.beta.messages.stream(params, { signal })
```

**请求主要内容**:
- `model` - 模型名称 (如 claude-sonnet-4-20250514)
- `max_tokens` - 最大输出 token 数
- `messages` - 对话消息数组 (含 cache_control)
- `system` - 系统提示词
- `tools` - 可用工具列表
- `metadata` - 元数据 (user_id 等)
- `betas` - 启用的 beta 特性
- `thinking` - 思考模式配置 (budget_tokens)
- `temperature` - 温度参数

**响应主要内容** (流式事件):
- `message_start` - 消息开始，包含 usage 信息
- `content_block_start/delta/stop` - 内容块流式传输
- `message_delta` - 消息结束，包含 stop_reason、usage
- `thinking_delta` - 思考过程（如启用）

**请求头**:
```
x-api-key: <API_KEY>
anthropic-beta: oauth-2025-04-20
x-app: cli
User-Agent: claude-code/2.0.62
```

### 1.2 Token 计数请求

**用途**: 计算消息的 token 数量

**端点**: `/v1/messages/count_tokens`

**请求主要内容**: 同上，但不实际生成回复

**响应主要内容**:
- `input_tokens` - 输入 token 数

### 1.3 配额检查请求

**用途**: 检查用户配额状态

**调用**: 发送最小请求 (max_tokens=1, content="quota") 检查响应头

**响应头关键字段**:
- `anthropic-ratelimit-unified-*-utilization` - 速率限制使用率
- `anthropic-ratelimit-unified-*-reset` - 重置时间

---

## 2. OAuth 认证

### 2.1 授权 URL

**Console 授权**: `https://console.anthropic.com/oauth/authorize`
**Claude.ai 授权**: `https://claude.ai/oauth/authorize`

**参数**:
- `client_id`: `9d1c250a-e61b-44d9-88ed-5944d1962f5e`
- `code_challenge` - PKCE 挑战码
- `state` - 状态参数
- `scope` - 权限范围 (user:profile, user:inference, user:sessions:claude_code)

### 2.2 Token 交换

**端点**: `https://console.anthropic.com/v1/oauth/token`

**请求主要内容**:
```json
{
  "grant_type": "authorization_code",
  "code": "<授权码>",
  "redirect_uri": "http://localhost:<port>/callback",
  "client_id": "<CLIENT_ID>",
  "code_verifier": "<PKCE验证器>"
}
```

**响应主要内容**:
- `access_token` - 访问令牌
- `refresh_token` - 刷新令牌
- `expires_in` - 过期时间（秒）
- `scope` - 授予的权限范围

### 2.3 Token 刷新

**端点**: `https://console.anthropic.com/v1/oauth/token`

**请求主要内容**:
```json
{
  "grant_type": "refresh_token",
  "refresh_token": "<刷新令牌>",
  "client_id": "<CLIENT_ID>"
}
```

### 2.4 创建 API Key

**端点**: `https://api.anthropic.com/api/oauth/claude_cli/create_api_key`

**请求**: POST，带 Bearer token 认证

**响应主要内容**:
- `raw_key` - 生成的 API 密钥

---

## 3. 用户账户 API

### 3.1 获取 CLI Profile

**端点**: `GET /api/claude_cli_profile`

**请求头**: `x-api-key`, `anthropic-beta`

**请求参数**: `account_uuid`

**响应**: 用户 CLI 配置信息

### 3.2 获取 OAuth Profile

**端点**: `GET /api/oauth/profile`

**请求头**: `Authorization: Bearer <token>`

**响应**: OAuth 用户信息

### 3.3 获取用户角色

**端点**: `GET /api/oauth/claude_cli/roles`

**响应主要内容**:
- `organization_role` - 组织角色
- `workspace_role` - 工作空间角色
- `organization_name` - 组织名称

### 3.4 账户设置

**获取**: `GET /api/oauth/account/settings`
**更新**: `PATCH /api/oauth/account/settings`

**设置项**: `grove_enabled` 等

### 3.5 首次使用日期

**端点**: `GET /api/organization/claude_code_first_token_date`

**响应**: `first_token_date` - 首次使用日期

### 3.6 Sonnet 1M 访问权限

**端点**: `GET /api/organization/{org_id}/claude_code_sonnet_1m_access`

**响应**:
- `has_access` - 是否有访问权限
- `has_access_not_as_default` - 非默认访问权限

---

## 4. 遥测与监控

### 4.1 事件日志批量上报

**端点**: `POST /api/event_logging/batch`

**用途**: 批量上报使用事件

**请求主要内容**: 事件数组，包含类型、时间戳、属性

### 4.2 Metrics 上报

**端点**: `POST /api/claude_code/metrics`

**用途**: 上报使用指标

### 4.3 用户反馈

**端点**: `POST /api/claude_cli_feedback`

**请求主要内容**:
```json
{
  "content": "<反馈内容JSON>"
}
```

**响应**: `feedback_id`

### 4.4 VCS 账户关联

**端点**: `POST /api/claude_code/link_vcs_account`

**请求主要内容**:
```json
{
  "vcs_type": "github",
  "vcs_host": "<主机>",
  "vcs_username": "<用户名>",
  "git_user_email": "<邮箱>"
}
```

---

## 5. 特性开关 (Statsig)

### 5.1 初始化

**端点**: `https://statsig.anthropic.com/v1/`

**用途**: 获取特性开关配置

**属性**:
- 用户 ID（哈希）
- 环境 (production/development)

### 5.2 远程评估

**端点**: `POST /api/eval/{clientKey}`

**请求主要内容**: 用户属性、强制变体

### 5.3 异常上报

**端点**: `https://statsigapi.net/v1/sdk_exception`

**用途**: 上报 Statsig SDK 异常

---

## 6. 错误监控 (Sentry)

**DSN**: `https://e531a1d9ec1de9064fae9d4affb0b0f4@o1158394.ingest.us.sentry.io/4508259541909504`

**用途**: 错误追踪和性能监控

---

## 7. 版本检查与更新

### 7.1 版本列表

**主端点**: `https://downloads.claude.ai/claude-code-releases`
**备用**: `https://storage.googleapis.com/claude-code-dist-.../claude-code-releases`

**用途**: 获取可用版本列表

### 7.2 更新日志

**端点**: `https://raw.githubusercontent.com/anthropics/claude-code/refs/heads/main/CHANGELOG.md`

---

## 8. A/B 测试 (GrowthBook)

**端点**: `https://cdn.growthbook.io`

**用途**: 获取实验配置

---

## 9. 云服务认证 (AWS/Azure/GCP)

### AWS Bedrock
- `bedrock-runtime.{region}.amazonaws.com` - Bedrock Runtime
- `cognito-identity.{region}.amazonaws.com` - Cognito 认证
- `sts.{region}.amazonaws.com` - STS 令牌服务
- `oidc.{region}.amazonaws.com` - OIDC 服务

### Azure
- `login.microsoftonline.com` - Azure AD 认证
- `cognitiveservices.azure.com` - 认知服务

### GCP
- `oauth2.googleapis.com` - OAuth
- `aiplatform.googleapis.com` - AI 平台

---

## 10. HTTP 客户端配置

### Anthropic SDK 客户端

```javascript
{
  defaultHeaders: {
    "x-app": "cli",
    "User-Agent": "claude-code/2.0.62",
    // 可选
    "x-claude-remote-container-id": "<容器ID>",
    "x-claude-remote-session-id": "<会话ID>",
    "x-anthropic-additional-protection": "true"
  },
  maxRetries: <重试次数>,
  timeout: 600000,  // 10分钟
}
```

### Axios 实例 (DQ)

用于非 Anthropic API 的 HTTP 请求（OAuth、遥测等）

---

## 请求流程图

```
┌─────────────────────────────────────────────────────────────┐
│                      用户输入                                │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    认证检查                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │ API Key 模式 │  │  OAuth 模式  │  │ 云服务模式   │      │
│  │ x-api-key    │  │ Bearer token │  │ AWS/Azure/GCP│      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    特性开关检查 (Statsig)                    │
│              获取用户可用的功能和实验分组                     │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    构建请求                                  │
│  - 组装 messages 数组                                        │
│  - 添加 system prompt                                        │
│  - 配置 tools                                                │
│  - 设置 cache_control                                        │
│  - 添加 metadata                                             │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│              发送流式请求 (messages.stream)                   │
│                api.anthropic.com/v1/messages                 │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    处理流式响应                              │
│  - message_start: 获取初始 usage                            │
│  - content_block_delta: 流式输出文本/工具调用                │
│  - message_delta: 获取 stop_reason, 最终 usage              │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    事件上报                                  │
│  - 使用统计 → /api/event_logging/batch                      │
│  - 指标 → /api/claude_code/metrics                          │
│  - 错误 → Sentry                                            │
└─────────────────────────────────────────────────────────────┘
```

---

## 错误处理

### 认证错误 (401)
- `authentication_error` - API Key 无效
- Token 过期 → 自动刷新

### 权限错误 (403)
- `permission_error` - 无权访问
- 组织策略限制

### 速率限制 (429)
- 检查 `anthropic-ratelimit-*` 响应头
- 显示警告并等待重置

### 网络错误
- 自动重试（配置 maxRetries）
- 回退到备用端点

---

## 安全考虑

1. **PKCE 流程** - OAuth 使用 code_verifier/code_challenge
2. **令牌存储** - 本地安全存储，定期刷新
3. **请求签名** - AWS 请求使用 SigV4
4. **敏感数据** - API Key 不记录到日志
