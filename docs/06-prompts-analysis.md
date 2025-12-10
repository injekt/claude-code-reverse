# Claude Code 提示词精髓深度解析

## 概述

本文档深入分析 Claude Code 提示词设计的精髓，揭示其背后的设计哲学、工程智慧和最佳实践。这些洞察对于构建高质量的 AI Agent 应用具有重要参考价值。

---

## 一、提示词架构设计

### 1.1 分层架构

Claude Code 的提示词采用清晰的分层架构：

```
┌─────────────────────────────────────────────────────────────────┐
│  Layer 1: Identity (身份层)                                      │
│  "You are Claude Code..."                                        │
│  作用：建立基本身份认知                                           │
├─────────────────────────────────────────────────────────────────┤
│  Layer 2: Behavior (行为层)                                      │
│  Tone, Style, Professional objectivity                          │
│  作用：定义交互风格和行为准则                                      │
├─────────────────────────────────────────────────────────────────┤
│  Layer 3: Capabilities (能力层)                                  │
│  Tool descriptions, Usage policies                               │
│  作用：描述可用工具和使用规则                                      │
├─────────────────────────────────────────────────────────────────┤
│  Layer 4: Domain (领域层)                                        │
│  Git operations, Code references, Security                       │
│  作用：领域特定的操作指南                                         │
├─────────────────────────────────────────────────────────────────┤
│  Layer 5: Context (上下文层)                                     │
│  Environment info, Git status, Model info                        │
│  作用：提供运行时动态信息                                         │
└─────────────────────────────────────────────────────────────────┘
```

**设计智慧:**
- 从抽象到具体的层次递进
- 静态指令与动态上下文分离
- 通用规则与领域规则分离

### 1.2 模块化组合

提示词采用模块化设计，根据场景动态组合：

```javascript
// 伪代码示例
function buildSystemPrompt(config) {
  const parts = [];

  // 1. 始终包含身份
  parts.push(IDENTITY_PROMPT);

  // 2. 根据场景选择行为指南
  if (!config.isSDK) {
    parts.push(TONE_AND_STYLE);
    parts.push(PROFESSIONAL_OBJECTIVITY);
  }

  // 3. 根据可用工具添加工具描述
  if (tools.has('TodoWrite')) {
    parts.push(TASK_MANAGEMENT_PROMPT);
  }

  // 4. 添加领域指南
  parts.push(GIT_COMMIT_GUIDE);
  parts.push(PR_CREATION_GUIDE);

  // 5. 添加动态上下文
  parts.push(buildEnvironmentInfo());

  return parts.join('\n\n');
}
```

---

## 二、核心设计原则解析

### 2.1 "先读后改"原则

**提示词体现:**
```
- NEVER propose changes to code you haven't read.
- You must use your `Read` tool at least once in the conversation before editing.
```

**为什么这很重要:**

1. **防止幻觉:** LLM 可能会"想象"文件内容，导致错误的修改建议
2. **理解上下文:** 只有读取文件才能理解代码风格、命名约定、依赖关系
3. **精确匹配:** Edit 工具的 `old_string` 必须精确匹配，不读取就无法获取

**工程实现:**
```javascript
// Edit 工具的验证逻辑
async checkPermissions(input, context) {
  const hasReadFile = context.conversationHistory
    .some(msg => msg.toolUse?.name === 'Read' &&
                 msg.toolUse?.input?.file_path === input.file_path);

  if (!hasReadFile) {
    return { behavior: 'deny', reason: 'Must read file before editing' };
  }
  return { behavior: 'allow' };
}
```

### 2.2 "避免过度工程化"原则

**提示词体现:**
```
- Don't add features, refactor code, or make "improvements" beyond what was asked.
- Don't create helpers, utilities, or abstractions for one-time operations.
- The right amount of complexity is the minimum needed for the current task.
- Three similar lines of code is better than a premature abstraction.
```

**深层原因:**

1. **LLM 的过度热心:** AI 天然倾向于"做更多"，这需要明确约束
2. **用户期望管理:** 用户期望精确执行指令，不是"惊喜"
3. **代码审查负担:** 额外的更改增加了 review 成本

**反模式示例:**
```javascript
// 用户请求: "修复 getUserName 函数中的 bug"

// ❌ 过度工程化的响应:
// 1. 修复 bug
// 2. 顺便重构了整个 User 类
// 3. 添加了新的 UserService
// 4. 更新了所有调用点
// 5. 添加了单元测试

// ✅ 正确的响应:
// 只修复 getUserName 函数中的 bug，保持其他代码不变
```

### 2.3 "专业客观性"原则

**提示词体现:**
```
Prioritize technical accuracy and truthfulness over validating the user's beliefs.
Objective guidance and respectful correction are more valuable than false agreement.
Avoid using over-the-top validation like "You're absolutely right".
```

**这解决什么问题:**

1. **RLHF 带来的谄媚倾向:** 模型可能过度倾向于同意用户
2. **技术讨论需要诚实:** 错误的肯定可能导致 bug
3. **建立可信赖的关系:** 用户更信任诚实的助手

**对比示例:**
```
用户: "我觉得用全局变量存储用户会话是最好的方案"

❌ 谄媚响应:
"您完全正确！全局变量确实是一个简单直接的方案..."

✅ 专业客观响应:
"全局变量在单用户场景下可以工作，但在并发环境中会导致会话混淆。
建议考虑使用 session 中间件或 context 对象来管理用户会话状态。"
```

### 2.4 "完整完成任务"原则

**提示词体现:**
```
IMPORTANT: Complete tasks fully. Do not stop mid-task or leave work incomplete.
Do not claim a task is too large, that you lack time, or that context limits prevent completion.
You have unlimited context through summarization.
```

**为什么需要这个:**

1. **防止推诿:** LLM 可能会找各种借口停止工作
2. **用户体验:** 半完成的任务比不开始更糟糕
3. **利用摘要能力:** Claude Code 有上下文压缩机制

**常见的推诿模式 (被禁止):**
```
❌ "这个任务比较复杂，建议分成多个步骤..."
❌ "由于上下文限制，我无法一次完成..."
❌ "这需要更多时间，我们下次继续..."
❌ "这超出了当前会话的范围..."
```

---

## 三、工具提示词设计智慧

### 3.1 工具边界的明确划分

**Bash 工具的限制:**
```
DO NOT use it for file operations (reading, writing, editing, searching, finding files)
- use the specialized tools for this instead.
```

**为什么这样设计:**

| 使用 Bash | 使用专用工具 |
|-----------|-------------|
| `cat file.txt` | Read 工具 |
| `grep pattern` | Grep 工具 |
| `find . -name` | Glob 工具 |
| `sed -i` | Edit 工具 |

**好处:**
1. **权限控制:** 专用工具有更细粒度的权限检查
2. **输出规范:** 专用工具的输出格式一致
3. **错误处理:** 专用工具有更好的错误信息
4. **用户体验:** UI 可以更好地展示专用工具的结果

### 3.2 工具描述的结构化

每个工具描述都遵循固定结构：

```
1. 简短的功能说明（一句话）
2. 重要提示/警告（IMPORTANT/CRITICAL）
3. 使用步骤（如果复杂）
4. Usage notes（详细说明）
5. 示例（可选）
6. 常见错误和解决方案（可选）
```

**示例分析 - Bash 工具描述:**

```
┌─────────────────────────────────────────────────────────────┐
│ 1. 功能说明                                                  │
│ "Executes a given bash command in a persistent shell        │
│  session with optional timeout..."                          │
├─────────────────────────────────────────────────────────────┤
│ 2. 重要警告                                                  │
│ "IMPORTANT: This tool is for terminal operations...         │
│  DO NOT use it for file operations"                         │
├─────────────────────────────────────────────────────────────┤
│ 3. 使用步骤                                                  │
│ "1. Directory Verification: ..."                            │
│ "2. Command Execution: ..."                                 │
├─────────────────────────────────────────────────────────────┤
│ 4. Usage notes                                               │
│ "- The command argument is required..."                     │
│ "- You can specify an optional timeout..."                  │
├─────────────────────────────────────────────────────────────┤
│ 5. 反模式警告                                                │
│ "Avoid using Bash with `find`, `grep`, `cat`..."            │
└─────────────────────────────────────────────────────────────┘
```

### 3.3 TodoWrite 的心理学设计

**提示词中的关键技巧:**

```
These tools are also EXTREMELY helpful for planning tasks...
If you do not use this tool when planning, you may forget to do important tasks
 - and that is unacceptable.
```

**设计意图:**
1. **制造紧迫感:** "unacceptable" 强调重要性
2. **预防遗忘:** 明确指出不使用的后果
3. **频繁使用:** "VERY frequently" 引导高频使用

**任务状态的强制约束:**
```
- Exactly ONE task must be in_progress at any time (not less, not more)
- Mark tasks complete IMMEDIATELY after finishing (don't batch completions)
```

**为什么是 ONE:**
- 防止 AI 声称在"并行"处理多个任务
- 确保专注和可追踪
- 用户清楚知道当前在做什么

---

## 四、安全设计模式

### 4.1 Git 操作的安全边界

**多层防护:**

```
Layer 1: 默认禁止
  - NEVER update the git config
  - NEVER run destructive git commands

Layer 2: 显式要求
  - unless the user explicitly requests

Layer 3: 警告确认
  - warn the user if they request it

Layer 4: 验证检查
  - ALWAYS check authorship before amending
```

**安全命令列表:**
```
✅ 安全 (可以直接执行):
- git status
- git diff
- git log
- git branch (查看)

⚠️ 需要确认:
- git commit
- git push
- git checkout

❌ 禁止 (除非明确要求):
- git push --force
- git reset --hard
- git config --global
```

### 4.2 双用途工具的授权上下文

**提示词:**
```
Dual-use security tools (C2 frameworks, credential testing, exploit development)
require clear authorization context: pentesting engagements, CTF competitions,
security research, or defensive use cases.
```

**授权判断流程:**

```
用户请求 → 是安全相关工具？
              │
              ├─ 否 → 正常执行
              │
              └─ 是 → 检查授权上下文
                        │
                        ├─ CTF/比赛 → 允许
                        ├─ 渗透测试合约 → 允许
                        ├─ 安全研究 → 允许
                        ├─ 防御目的 → 允许
                        └─ 其他/不明确 → 拒绝并解释
```

---

## 五、上下文管理策略

### 5.1 动态信息注入

**环境信息模板:**
```
<env>
Working directory: ${cwd}
Is directory a git repo: ${isGit}
Platform: ${platform}
OS Version: ${osVersion}
Today's date: ${date}
</env>
```

**为什么用 XML 标签:**
1. **明确边界:** LLM 清楚知道这是结构化数据
2. **易于解析:** 如果需要，可以程序化处理
3. **与内容区分:** 不会与用户输入混淆

### 5.2 模型自我认知

```
You are powered by the model named Claude Opus 4.5.
The exact model ID is claude-opus-4-5-20251101.

Assistant knowledge cutoff is January 2025.
```

**作用:**
1. **准确的能力认知:** 知道自己是什么模型
2. **时间意识:** 知道知识截止日期
3. **版本追踪:** 用于调试和分析

### 5.3 背景知识注入

```xml
<claude_background_info>
The most recent frontier Claude model is Claude Opus 4.5
(model ID: 'claude-opus-4-5-20251101').
</claude_background_info>
```

**使用场景:**
- 用户询问"最新的 Claude 模型是什么"
- 需要比较不同模型能力时
- 讨论 API 版本选择时

---

## 六、子代理提示词设计

### 6.1 能力限制声明

**Plan Agent:**
```
CRITICAL: This is a READ-ONLY task. You CANNOT edit, write, or create files.
```

**为什么显式声明:**
1. **防止越权:** 即使工具可用，也明确禁止使用
2. **角色清晰:** 规划者不是执行者
3. **错误预防:** 避免规划阶段产生副作用

### 6.2 专业化分工

| Agent | 专长 | 可用工具 | 禁用工具 |
|-------|------|---------|---------|
| Explore | 代码探索 | Glob, Grep, Read | Edit, Write |
| Plan | 架构设计 | Glob, Grep, Read | Edit, Write, Task |
| Claude Guide | 文档查询 | Read, WebFetch, WebSearch | Edit, Write, Bash |

**分工的好处:**
1. **提示词更聚焦:** 每个 Agent 的指令更简洁
2. **能力更专业:** 专注于特定任务
3. **成本更优化:** 可以使用更小的模型

### 6.3 深度级别设计

**Explore Agent 的三级深度:**
```
"quick" - 基本搜索
"medium" - 中等探索
"very thorough" - 全面分析
```

**用户可以这样调用:**
```
使用 Task 工具，subagent_type=Explore，
提示词包含 "very thorough" 来触发深度搜索
```

---

## 七、提示词工程最佳实践总结

### 7.1 明确性原则

```
✅ "NEVER propose changes to code you haven't read"
❌ "Try to read code before modifying"

✅ "Exactly ONE task must be in_progress at any time"
❌ "Usually have one task in progress"
```

**关键词使用:**
- `NEVER` / `ALWAYS` - 绝对规则
- `MUST` / `REQUIRED` - 强制要求
- `IMPORTANT` / `CRITICAL` - 强调重点
- `AVOID` / `PREFER` - 倾向建议

### 7.2 示例驱动

```
Examples of proper quoting:
  - cd "/Users/name/My Documents" (correct)
  - cd /Users/name/My Documents (incorrect - will fail)
```

**好的示例特征:**
1. 正反对比
2. 简单明了
3. 贴近实际场景
4. 展示边界情况

### 7.3 预防性设计

**预见性地禁止错误行为:**
```
- NEVER use bash echo to communicate with user
- NEVER include line number prefix in old_string
- NEVER commit unless explicitly asked
```

**这基于:**
1. 真实用户反馈
2. 常见错误模式分析
3. LLM 行为特性理解

### 7.4 渐进式复杂度

**简单任务:** 直接执行
**中等任务:** 使用 TodoWrite 跟踪
**复杂任务:** 使用 EnterPlanMode 规划

```
if (task.complexity < 3 steps) {
  直接执行
} else if (task.complexity < architectural decision) {
  使用 TodoWrite
} else {
  使用 EnterPlanMode
}
```

---

## 八、可复用的提示词模式

### 8.1 工具使用引导模式

```
When NOT to use [TOOL]:
- Scenario 1
- Scenario 2

When to use [TOOL]:
- Scenario A
- Scenario B

Usage notes:
- Detail 1
- Detail 2
```

### 8.2 安全边界模式

```
[ACTION] is allowed for:
- Authorized context 1
- Authorized context 2

[ACTION] is prohibited for:
- Forbidden context 1
- Forbidden context 2

When unclear, [DEFAULT_BEHAVIOR]
```

### 8.3 任务执行模式

```
Before [ACTION]:
- Check 1
- Check 2

During [ACTION]:
- Step 1
- Step 2

After [ACTION]:
- Verification 1
- Cleanup
```

---

## 九、结论

Claude Code 的提示词设计体现了以下核心智慧：

1. **分层架构** - 从身份到上下文的清晰层次
2. **明确约束** - 用强硬的语言设定边界
3. **预防错误** - 预见性地禁止常见错误
4. **专业分工** - 通过子代理实现能力分化
5. **用户至上** - 所有设计都服务于用户体验

这些设计模式不仅适用于 Claude Code，也为任何 AI Agent 应用的提示词工程提供了宝贵参考。

---

## 附录：提示词字数统计

| 类别 | 大约字数 | 占比 |
|------|---------|-----|
| 系统身份与行为 | ~800 | 15% |
| 任务执行指南 | ~1200 | 23% |
| 工具描述 | ~2000 | 38% |
| Git 操作指南 | ~600 | 11% |
| 子代理提示词 | ~400 | 8% |
| 上下文信息 | ~300 | 5% |
| **总计** | **~5300** | **100%** |
