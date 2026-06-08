# Claude Code 源码学习指南

> **目标：** 详细了解 Claude Code 的实现细节，并独立完成复现
>
> **策略：** 由内到外、由核心到外围，每个阶段结束时都有一个可运行的更完整版本

---

## 分层架构总览

在进入任何具体模块之前，先建立全局视角。Claude Code 共 5 层，自上而下：

```
┌─────────────────────────────────────────────┐
│  终端 UI 层                                  │
│  Ink/React 组件树，键盘输入，消息渲染         │
│  src/screens/REPL.tsx, src/components/       │
├─────────────────────────────────────────────┤
│  查询编排层                                  │
│  消息预处理、System Prompt 构建、并发守卫     │
│  handlePromptSubmit, onQuery, onQueryImpl    │
├─────────────────────────────────────────────┤
│  Agentic Loop 层                             │
│  while(true) 决策循环，错误恢复，压缩管道    │
│  src/query.ts, src/QueryEngine.ts            │
├─────────────────────────────────────────────┤
│  工具执行层                                  │
│  权限检查，并发/串行调度，60 个工具实现       │
│  src/services/tools/, packages/builtin-tools │
├─────────────────────────────────────────────┤
│  API 通信层                                  │
│  流式 API，事件状态机，多 Provider 适配      │
│  src/services/api/, Provider adapters        │
└─────────────────────────────────────────────┘
```

**各层的关键设计原则：**
- UI 层通过 `for await (event of query(...))` 消费 Agentic Loop 产出的 StreamEvent，两层**完全解耦**
- Agentic Loop 通过 `deps.callModel()` 调用 API 层，依赖注入方便测试
- 所有 Provider（Anthropic / OpenAI / Gemini / Grok）输出统一为 `BetaRawMessageStreamEvent`，工具执行层**不感知** Provider 的存在
- 权限检查在工具调用前的最后一刻发生，不在 Loop 里，不在 API 里

---

## 总体学习路线

```
阶段 1:  数据契约         → 理解所有消息类型（基础）
阶段 2:  最小 MVP         → 能对话的 CLI（验证理解）
阶段 3:  工具系统         → 工具接口、注册、调度（核心）
阶段 4:  API 通信层       → 流式 API 与 Provider 适配（核心）
阶段 5:  权限系统         → 工具执行前的安全门
阶段 6:  完整 Loop        → 所有错误恢复路径
阶段 7:  终端 UI          → Ink/React 渲染
阶段 8:  记忆与上下文注入 → 持久化记忆 + System Prompt 构建
阶段 9:  Context 压缩管理 → 运行时 context window 管理
阶段 10: MCP 协议         → 外部工具扩展
阶段 11: Hook 系统        → 三种 hook 的拦截与扩展
阶段 12: 多 Agent         → 子 Agent 系统
阶段 13: 多 Provider      → OpenAI/Gemini/Grok 适配
阶段 14: 构建工程化       → Bun build + 代码分割
```

---

## 阶段 1：理解数据契约（3-4 天）

**核心原则：** 复现时最大的坑是不理解数据结构，导致后续各层对不上。

### 必读文件

| 文件 | 重点 |
|------|------|
| `src/types/message.ts` | UI 层消息类型（re-export + 扩展） |
| `packages/@ant/model-provider/` | 基础消息类型定义源头 |
| `src/types/permissions.ts` | PermissionMode, PermissionResult |
| `src/types/tools.ts` | 工具 Progress 类型 |
| `src/Tool.ts` | **最重要的类型文件**：Tool 接口、ToolUseContext |

### 关键消息类型图

```
Message（所有消息的联合类型）
├── UserMessage      { type: 'user', role: 'user', message: { content: ContentItem[] } }
├── AssistantMessage { type: 'assistant', role: 'assistant', message.content, stop_reason }
├── SystemMessage    { type: 'system', level: SystemMessageLevel }
├── AttachmentMessage { type: 'attachment', attachment: {...} }
├── ProgressMessage  { type: 'progress', ... }
└── TombstoneMessage { type: 'tombstone', message: AssistantMessage }

ContentItem（消息内容块）
├── TextBlock        { type: 'text', text: string }
├── ToolUseBlock     { type: 'tool_use', id: string, name: string, input: Record }
├── ToolResultBlock  { type: 'tool_result', tool_use_id: string, content: string|ContentItem[] }
└── ThinkingBlock    { type: 'thinking', thinking: string, signature: string }
```

### 自测问题

- `tool_use_id` 是如何把 tool_use 和 tool_result 配对的？
- `TombstoneMessage` 什么时候出现？（答：模型 fallback 时清除孤儿消息）
- `ToolUseContext` 里有哪些字段，为什么每次工具调用都要传它？

---

## 阶段 2：最小 MVP（3-4 天）

**目标：** 写出一个能与 Claude 对话、能执行一次工具调用的最小程序。

先只关注 `src/query.ts` 的**正常路径**（跳过所有 `feature()` 条件分支和错误恢复）：

```
while(true) {
  1. 调用 API → 流式收集 AssistantMessages
  2. 提取所有 tool_use blocks
  3. 如果没有 tool_use → return（结束）
  4. 并行执行所有工具 → 收集 ToolResults
  5. 把结果追加到消息列表 → continue（下一轮）
}
```

此阶段的 API 调用和工具接口**只需粗读**，目的是跑通流程。**阶段 3、4 会深入**。

**MVP 验收：**
```bash
echo "创建 hello.ts 文件，写入 console.log('hello')" | node your-cli.js -p
# 应该能看到 Claude 调用 FileWriteTool 并创建文件
```

---

## 阶段 3：工具系统（4-5 天）

> 这是最容易被忽视的核心模块。Agentic Loop 是"决策框架"，工具才是"真正干活的部分"。

### 3.1 Tool 接口契约

**文件：** `src/Tool.ts`

```typescript
interface Tool {
  name: string
  description: string          // ⚠️ 直接影响 AI 选择此工具的概率，是 prompt engineering
  inputSchema: ToolInputJSONSchema   // Zod schema → JSON Schema，发给 API
  call(input, context): AsyncGenerator<ToolProgressData, ToolResult>
  // AsyncGenerator：执行过程中可以 yield 进度，最后 return 结果
  checkPermissions?(input, context): Promise<PermissionResult>
  backfillObservableInput?(input): void  // 填充可观察字段（不破坏 prompt cache）
}
```

### 3.2 工具注册表（条件装载）

**文件：** `src/tools.ts`

```typescript
// 工具按 feature flag / USER_TYPE / 运行时条件 装载
export function getAllBaseTools(): Tool[] {
  return [
    ...coreTools,                                    // 始终装载
    ...(feature('DAEMON') ? daemonTools : []),       // feature 门控
    ...(process.env.USER_TYPE === 'ant' ? antTools : []),  // 用户类型门控
    ...mcpTools,                                     // MCP 动态注册
  ]
}
```

60 个工具按功能分类：
| 分类 | 工具 | 要点 |
|------|------|------|
| 文件操作 | FileRead, FileWrite, FileEdit, Glob, Grep | 最高频，理解权限与结果格式 |
| Shell 执行 | BashTool, PowerShellTool | 最复杂的安全分析 |
| Agent 系统 | AgentTool, TaskCreate/Update/List | 多 Agent 入口 |
| Web/MCP | WebFetch, WebSearch, MCPTool | 外部数据源 |
| 调度 | CronCreate, CronDelete | 定时任务 |
| 工具发现 | SearchExtraTools, ExecuteExtraTool | TF-IDF 延迟发现 |

### 3.3 工具调度：并发 vs 串行

**文件：** `src/services/tools/toolOrchestration.ts`

```
partitionToolCalls(toolUseMessages, context)
    │
    ├── 只读工具（Glob, Grep, FileRead）→ isConcurrencySafe: true
    │   runToolsConcurrently()  最多 10 个同时执行
    │
    └── 写入工具（Bash, FileEdit, FileWrite）→ isConcurrencySafe: false
        runToolsSerially()  严格串行
```

**关键设计：** 为什么写入工具必须串行？
因为 `FileEdit` 依赖文件的当前状态，两个 FileEdit 并发会产生竞态条件，patch 会互相覆盖。

### 3.4 单工具执行流程

**文件：** `src/services/tools/toolExecution.ts`

```
runToolUse(toolUseBlock, canUseTool, context)
    │
    ├── findToolByName()  → 查找工具（内置 or MCP）
    ├── canUseTool()      → 权限检查（见阶段 5）
    │
    ├── 如果 result === 'ask' → 暂停，等用户在 UI 确认
    │
    └── tool.call(input, context)
            for await (progress of generator):
                yield ProgressMessage  → 实时显示执行状态
            return ToolResult
```

### 3.5 工具发现（延迟加载）

**文件：** `src/services/searchExtraTools/toolIndex.ts`

60 个工具不会全部出现在每次 API 请求的 `tools` 参数里（会超出 token 限制）。延迟工具通过 TF-IDF 语义搜索按需发现：

```
用户发消息 → searchExtraToolsPrefetch 分析消息关键词
    → TF-IDF 匹配相关工具
    → 将匹配的工具动态注入本轮 tools 列表
```

**精读工具（按复杂度排序）：**

| 工具 | 文件 | 学习重点 |
|------|------|---------|
| `FileReadTool` | `FileReadTool/` | 最简单，理解 input → result 全流程 |
| `GlobTool` | `GlobTool/` | description 如何影响 AI 的工具选择 |
| `FileEditTool` | `FileEditTool/FileEditTool.ts` | 精确字符串匹配替换，unique match 保证 |
| `BashTool` | `BashTool/` | 安全分析 + 权限 + 沙盒，最复杂 |
| `AgentTool` | `AgentTool/runAgent.ts` | 递归调用 queryLoop，子 Agent 入口 |

---

## 阶段 4：API 通信层（3-4 天）

> 工具系统负责"干什么"，API 通信层负责"怎么和 Claude 说话"。

### 4.1 流式事件状态机

**文件：** `src/services/api/claude.ts` 的 `queryModelWithStreaming()`

```
API 返回 BetaRawMessageStreamEvent 流：

message_start            → 初始化，记录 TTFT（首 token 延迟）
  │
  content_block_start    → 创建内容块（text / tool_use / thinking）
  content_block_delta    → 增量追加（text += delta，input += partial_json）
  content_block_stop     → 构建完整块，yield AssistantMessage ← UI 立即渲染
  │
  content_block_start    → 下一个内容块（AI 可以交替输出文字和工具调用）
  ...
  │
message_delta            → 回写 stop_reason 到最后一条消息
message_stop             → 流结束
```

**3 个关键细节：**

1. **stop_reason 回写**：`stop_reason` 在 `message_delta` 才确定，但 AssistantMessage 在 `content_block_stop` 就 yield 了，所以必须直接修改对象属性（不能用新对象替换，否则破坏 prompt cache）

2. **tool_use input 增量拼接**：
   ```typescript
   // content_block_delta 阶段：字符串拼接
   input += delta.partial_json  // '{"command":"' → 'ls -la"}'
   // content_block_stop 阶段：一次性 parse
   block.input = JSON.parse(input)
   ```

3. **StreamingToolExecutor 并行**：tool_use block 在流式阶段一出现就立即开始执行工具，不等流结束，总时间更短

### 4.2 normalizeMessagesForAPI

**文件：** `src/utils/messages.ts`

UI 内部消息格式包含大量 UI 专用字段（`uuid`, `timestamp`, `isMeta`, `toolUseResult` 等），发给 API 前必须清理：

```typescript
// UI 格式 → API 格式
{ type:'user', uuid:'xxx', timestamp:123, message:{ content:[...] } }
    ↓ normalizeMessagesForAPI()
{ role:'user', content:[...] }
```

### 4.3 Prompt Cache 注入

**文件：** `src/utils/api.ts` 的 `splitSysPromptPrefix()`

System Prompt 分两段，只有固定前缀部分打 `cache_control`，节省 60-70% token：

```
[固定前缀：工具规则、工具描述] ← cache_control: { type: 'ephemeral' }（可缓存）
[动态部分：git status、CLAUDE.md、日期] ← 无 cache_control（每次都变）
```

### 4.4 Provider 选择逻辑

**文件：** `src/utils/model/providers.ts`

```
优先级：modelType 参数 > 环境变量 > 默认 firstParty
    │
    ├── CLAUDE_CODE_USE_OPENAI=1  → OpenAI 兼容层
    ├── CLAUDE_CODE_USE_GEMINI=1  → Gemini 兼容层
    ├── CLAUDE_CODE_USE_GROK=1    → Grok 兼容层
    ├── AWS_PROFILE 存在           → Bedrock
    └── 默认                      → Anthropic firstParty
```

所有 Provider 的输出都是 `BetaRawMessageStreamEvent`，下游完全不感知 Provider 的差异。

---

## 阶段 5：权限系统（3-4 天）

### 权限判断三层结构

```
PermissionMode（会话级）
  ├── 'default'             → 每次工具调用都提示用户
  ├── 'acceptEdits'         → 自动允许文件编辑，Bash 还是提示
  ├── 'bypassPermissions'   → 全自动（CI/pipe 模式）
  └── 'plan'               → 只读，不允许任何修改

ToolPermissionContext（工具级）
  ├── alwaysAllowRules      → 已授权的命令列表
  ├── additionalWorkingDirectories → 工作目录白名单
  └── mcp 授权列表

PermissionResult（判断结果）
  ├── { result: 'allow', ... }
  ├── { result: 'deny', reason }
  └── { result: 'ask', prompt }  → 触发 UI 弹出权限提示
```

### 核心文件

| 文件 | 重点 |
|------|------|
| `src/types/permissions.ts` | PermissionMode 枚举定义 |
| `src/hooks/useCanUseTool.tsx` | `canUseTool` 核心判断逻辑 |
| `packages/builtin-tools/src/tools/BashTool/bashSecurity.ts` | 2634 行！Bash 命令安全分析 |
| `packages/builtin-tools/src/tools/BashTool/readOnlyValidation.ts` | 1990 行！只读命令检测 |
| `src/components/permissions/` | UI 层权限提示组件 |

**自测：** 为什么 `readOnlyValidation.ts` 有 1990 行？
→ Bash 命令只读检测需要解析 shell 语法，识别重定向/管道/命令替换等所有可能的写操作，不能简单地字符串匹配。

---

## 阶段 6：完整 Agentic Loop + 错误恢复（4-5 天）

回到 `src/query.ts`，现在读**全部**内容。

### 6 个恢复机制

| 机制 | 触发条件 | 防止死循环的守卫 |
|------|---------|-----------------|
| `max_output_tokens` 提升 | 输出被截断，首次触发 | `maxOutputTokensOverride !== undefined` |
| `max_output_tokens` 恢复 | 提升后仍截断 | `MAX_OUTPUT_TOKENS_RECOVERY_LIMIT = 3` |
| `reactive_compact` | 触发 413 prompt-too-long | `hasAttemptedReactiveCompact` 标志 |
| `autocompact` | 每轮开始 token 超阈值 | 内置频率限制 |
| `microcompact` | 工具结果太长 | 按 token 预算截断 |
| `stop_hook_blocking` | Stop hook 注入阻塞消息 | `stopHookActive` + 不重置 reactive compact 守卫 |

### State 不可变更新模式

```typescript
// ✅ 正确：创建新对象
state = {
  ...state,
  turnCount: state.turnCount + 1,
  transition: { reason: 'next_turn' }
}
continue

// ❌ 错误：就地修改
state.turnCount++
```

**`transition` 字段的作用：** 让下一轮迭代能检测上一轮为什么继续，防止同一恢复路径死循环。

### QueryEngine 与 query.ts 的分工

```
query.ts (queryLoop)              QueryEngine.ts
─────────────────────             ──────────────────────────
处理 API 通信                     管理会话生命周期
处理工具执行                      维护 messages[] 历史
处理错误恢复                      处理文件快照（file history）
产出 StreamEvent                  触发 attribution（提交归因）
                                  触发 compaction 的上层决策
```

---

## 阶段 7：终端 UI（Ink）（4-5 天）

### Ink 框架基础

```
packages/@ant/ink/src/
├── core/          → Ink 渲染引擎（React 树 → 终端 ANSI escape codes）
├── components/    → Box, Text, Newline 等基础组件
├── hooks/         → useInput, useStdin, useFocus
└── keybindings/   → 快捷键绑定系统
```

**关键概念：** Ink 用 yoga-wasm 做 flexbox 布局，把 React 虚拟 DOM diff 转换为终端 ANSI escape codes。

### REPL 主屏幕（6541 行）

精读 `src/screens/REPL.tsx` 的关键区域：
1. `useEffect` 初始化（MCP 连接、配置加载）
2. `onSubmit` callback — 用户提交时的入口，分流命令/普通文本/远程
3. `onQuery` callback — 并发守卫（QueryGuard），触发 API 调用
4. `onQueryImpl` — 构建 system prompt + 调用 `query()`
5. `onQueryEvent` — 处理 query() 产出的 StreamEvent，更新 UI

### 状态管理

```
AppState.tsx    → 状态类型定义（React Context）
store.ts        → createStore（类 Zustand 实现，subscribe 机制）
selectors.ts    → 派生状态（减少不必要的重渲染）
```

**QueryGuard** (`src/utils/QueryGuard.ts`)：防止并发 onQuery 的状态机。`tryStart()` 原子性地从 idle→running，返回 generation 编号；`end(generation)` 确保只有最新 generation 能重置状态，解决取消+重提交的竞态。

---

## 阶段 8：记忆与上下文注入（3 天）

> ⚠️ **注意区分：** 本阶段讲的是**持久化记忆**（告诉 Claude "我是谁、这个项目是什么"），
> 与阶段 9 的**运行时 context 压缩**（context window 快满了怎么处理）是完全不同的两件事。

### 持久化记忆的来源

```
System Prompt 分层结构（从上到下优先级递减）：
──────────────────────────────────────────────────
[CLISyspromptPrefix]     固定前缀（工具使用规则，约 2000 字）
[全局 CLAUDE.md]         ~/.claude/CLAUDE.md（用户级指令）
[项目 CLAUDE.md 层级]    当前目录 → 父目录 → ... → $HOME（项目指令）
[memory files]           ~/.claude/projects/<proj>/memory/（自动保存的记忆）
[git status / date]      每次动态生成
[用户自定义系统提示]     --system-prompt / --append-system-prompt
──────────────────────────────────────────────────
```

### 核心文件

| 文件 | 重点 |
|------|------|
| `src/context.ts` | `getSystemContext()` / `getUserContext()` 拼接所有上下文 |
| `src/utils/claudemd.ts` | 发现 CLAUDE.md 文件层级（向上遍历目录树） |
| `src/utils/systemPrompt.ts` | `buildEffectiveSystemPrompt()` 最终组合 |
| `src/constants/system.ts` | 系统提示固定前缀常量 |
| `src/memdir/` | memory files 读写（持久化记忆） |

### Prompt Cache 关键设计

`backfillObservableInput()` 为什么检查"是否真的新增了字段"才克隆？
→ 因为对象**字节一致性**直接影响 prompt cache 命中率。修改对象会破坏 cache。
→ 只追加新字段才克隆，覆盖已有字段不克隆（已有字段覆盖不影响 API 行为，但会破坏 cache）。

---

## 阶段 9：Context 压缩管理（3-4 天）

> ⚠️ **注意区分：** 本阶段讲的是**运行时** context window 管理，
> 解决"对话太长超出 Claude 的 context 上限"的问题，与阶段 8 的持久化记忆无关。

### 5 种压缩策略

| 策略 | 触发时机 | 代价 | 可逆性 |
|------|---------|------|--------|
| `autocompact` | 每轮开始，token > 85% 阈值 | 调用 Claude 生成摘要 | **不可逆** |
| `microcompact` | 单个工具结果过长 | 调用 Haiku 生成摘要 | **不可逆** |
| `snipCompact` | HISTORY_SNIP feature 启用时 | 直接删除旧轮次 | **不可逆** |
| `reactiveCompact` | 收到 API 413 错误 | 调用 Claude 生成摘要 | **不可逆** |
| `contextCollapse` | CONTEXT_COLLAPSE feature 启用时 | 暂存可恢复块 | **可逆（drain 时释放）** |

### 压缩管道（每轮循环开始时执行）

```
messagesForQuery（原始消息历史）
     ↓ applyToolResultBudget()   截断超长工具结果（按 maxResultSizeChars）
     ↓ snipCompactIfNeeded()     移除最旧的 N 轮（记录 snipTokensFreed）
     ↓ microcompact()            对超长的单个工具结果生成内联摘要
     ↓ applyCollapsesIfNeeded()  折叠可恢复的信息块
     ↓ autocompact()             token > 阈值时压缩整段历史
messagesForQuery（处理后，准备发给 API）
```

### snipTokensFreed 传递的必要性

```typescript
snipTokensFreed = snipResult.tokensFreed
// autocompact 的阈值检查必须扣除这个值
const currentTokens = tokenCountWithEstimation(messagesForQuery) - snipTokensFreed
// 否则：snip 已经把 token 降到阈值下，但 autocompact 用的是 snip 前的计数，
// 触发不必要的压缩（"双重压缩"，浪费 API 调用）
```

### 核心文件

```
src/services/compact/
├── autoCompact.ts      → 触发时机（token 阈值计算，85% rule）
├── compact.ts          → 压缩执行（调用 Claude Haiku 生成摘要）
├── microCompact.ts     → 工具结果内联摘要
├── snipCompact.ts      → 历史 snip（保留最近 N 轮）
└── reactiveCompact.ts  → 被动压缩（触发 413 后）
```

---

## 阶段 10：MCP 协议支持（3-4 天）

MCP（Model Context Protocol）= 标准协议，让 Claude 能调用外部 server 提供的工具。

```
src/services/mcp/client.ts               → MCP 客户端，连接到 MCP server
src/services/mcp/MCPConnectionManager.tsx → 管理多个 MCP 连接的生命周期
src/services/mcp/types.ts                → MCPServerConnection, ServerResource
packages/mcp-client/                      → 底层 MCP 协议实现
```

**关键设计：** `src/tools.ts` 中有一段逻辑将 MCP server 的工具**动态注册**到工具列表。这是 Claude Code 可扩展性的核心机制——外部 server 的工具和内置工具对 Agentic Loop 完全透明。

---

## 阶段 11：Hook 系统（2-3 天）

Hook 系统是 Claude Code 的**扩展点机制**，允许在关键节点拦截和修改行为，IDE 插件、CI 验证等都通过 Hook 实现。

### 三种 Hook 及触发时机

```
消息发送前          API 响应后              任务声明完成时
     │                  │                        │
     ▼                  ▼                        ▼
UserPromptSubmit    PostSampling             Stop Hook
     │                  │                        │
拦截/修改用户消息   检查/修改 AI 回复        验证任务是否真的完成
阻止发送（blocked） 添加额外上下文          阻止结束（blockingErrors）
                                            强制重试（preventContinuation）
```

### 核心文件

| 文件 | 重点 |
|------|------|
| `src/utils/hooks.ts` | Hook 注册与执行的核心逻辑 |
| `src/utils/processUserInput/processTextPrompt.ts` | `executeUserPromptSubmitHooks` 调用点 |
| `src/utils/hooks/postSamplingHooks.ts` | PostSampling hook 执行 |
| `src/query/stopHooks.ts` | Stop hook 执行，`handleStopHooks()` |

### Stop Hook 的防死循环设计

```typescript
// query.ts：Stop Hook 的阻塞重试
if (stopHookResult.blockingErrors.length > 0) {
  state = {
    messages: [...messages, ...blockingErrors],
    stopHookActive: true,
    // ⚠️ 关键：保留 hasAttemptedReactiveCompact
    // 如果 compact 已经跑过，stop hook 重试不应该再触发 compact
    // 否则：stop hook 阻塞 → compact → 仍 413 → stop hook 阻塞 → 死循环
    hasAttemptedReactiveCompact,
    transition: { reason: 'stop_hook_blocking' },
  }
  continue
}
```

---

## 阶段 12：多 Agent 系统（4-5 天）

### AgentTool 的工作原理

```
AgentTool.call(input) {
  1. 创建子 agent 的独立消息历史（隔离的 context）
  2. 运行完整的 queryLoop（递归调用！）
  3. 将子 agent 的最终输出作为 ToolResult 返回给父 agent
  4. 子 agent 的工具权限由父 agent 的权限继承
}
```

**关键难点：** 子 agent 有自己独立的 `AbortController`，但父 agent abort 时需要**级联中断**所有子 agent。

精读 `packages/builtin-tools/src/tools/AgentTool/runAgent.ts`（959 行）。

### 子 Agent 与父 Agent 的隔离边界

```
父 Agent (queryLoop)
    │
    └── AgentTool.call()
            │
            └── 子 queryLoop（独立实例）
                    ├── 独立的 messages[]（不共享父 Agent 的历史）
                    ├── 独立的 AbortController（可单独中断）
                    ├── 继承父 Agent 的 PermissionMode
                    └── 继承父 Agent 的 alwaysAllowRules
```

---

## 阶段 13：多 Provider 兼容层（2-3 天）

**设计精髓：** 所有兼容层的输出都是 `BetaRawMessageStreamEvent`，下游（query.ts）**完全不感知** Provider 的存在。这是策略模式的教科书级应用。

```
src/services/api/openai/
├── client.ts          → provider 选择入口
├── transform/         → Anthropic 格式 ↔ OpenAI 格式互转
└── adapter.ts         → 流适配器（把 OpenAI stream 转为 Anthropic stream 格式）
```

**关键：** 适配器的核心职责是**格式转换**，不改变业务逻辑。转换分两个方向：
- 请求方向：Anthropic tool schema → OpenAI function calling schema
- 响应方向：OpenAI SSE events → `BetaRawMessageStreamEvent`

---

## 阶段 14：构建系统（2 天）

**为什么必须代码分割：** Bun/JSC 全量解析 17MB 单文件时 RSS 暴涨到 1GB。分割为 600+ chunk 后 `--version` 只需 35MB RSS。

```
build.ts              → Bun.build() 配置，代码分割
scripts/defines.ts    → MACRO 常量注入（版本号等）
scripts/dev.ts        → 开发模式的 -d flag 注入
vite.config.ts        → 备用构建（Vite + 代码分割）
src/utils/distRoot.ts → 运行时路径定位（import.meta.url 解析）
```

**Feature Flag 系统：**
```typescript
// ✅ 正确（Bun 构建器能静态分析）
if (feature('DAEMON')) { ... }
feature('X') ? a : b

// ❌ 错误（Bun 构建器无法静态消除）
const enabled = feature('DAEMON')  // 不能赋值给变量
if (enabled) { ... }
```

---

## 复现时最容易踩的坑

| 坑 | 说明 | 解法 |
|----|------|------|
| `tool_use_id` 配对 | API 要求每个 tool_use 都有对应的 tool_result，顺序和 ID 必须完全对上 | `ensureToolResultPairing()` in `utils/messages.ts` |
| prompt cache 字节一致性 | 修改消息对象的任何字段都会破坏 cache | 只追加，不修改；`backfillObservableInput` 的克隆守卫 |
| 流式工具并行 | 工具调用在流式传输过程中就开始执行，不是等 stream 结束 | `StreamingToolExecutor` 的设计 |
| thinking block 跨模型 | thinking 的 signature 与模型绑定，fallback 时必须 strip | `stripSignatureBlocks()` |
| Bun `feature()` 限制 | 只能在 `if` 条件或三元表达式中用，不能赋值给变量 | 严格遵守，否则 Bun 构建器无法静态分析 |
| MCP 工具命名冲突 | 内置工具和 MCP 工具可能同名 | `toolMatchesName()` 的前缀匹配逻辑 |
| Ink 重渲染性能 | 消息列表很长时直接渲染会卡 | `useVirtualScroll` + React Compiler memoization |
| 止损的 `transition` 守卫 | 恢复路径死循环（如 reactive_compact → 仍 413 → reactive_compact） | `state.transition?.reason` 检查，防止同一路径重复触发 |
| 写工具并发竞态 | 两个 FileEdit 并发会互相覆盖 patch | `partitionToolCalls` 确保写工具串行 |
| Stop Hook 死循环 | hook 重试时重置了 reactive compact 守卫 → compact → 413 → hook → 死循环 | `hasAttemptedReactiveCompact` 在 stop_hook_blocking 时必须保留 |

---

## 时间规划

```
Week 1:   阶段 1-2    → 数据契约 + 能对话的最小 CLI
Week 2-3: 阶段 3-4    → 工具系统 + API 通信层（两个核心）
Week 4:   阶段 5-6    → 权限系统 + 完整 Loop 错误恢复
Week 5:   阶段 7      → 终端 UI（Ink）
Week 6:   阶段 8-9    → 记忆注入 + Context 压缩
Week 7:   阶段 10-11  → MCP + Hook 系统
Week 8:   阶段 12-13  → 多 Agent + 多 Provider
Week 9:   阶段 14     → 构建工程化
```

---

## 深入参考文档

| 文档 | 内容 |
|------|------|
| `docs/message-flow-deep-dive.md` | **完整消息链路深度解析**（源码级，1200+ 行）|
| `docs/conversation/the-loop.mdx` | Agentic Loop 源码级解析 |
| `docs/conversation/streaming.mdx` | 流式响应机制 |
| `docs/conversation/multi-turn.mdx` | 多轮对话机制 |
| `docs/tools/what-are-tools.mdx` | 工具系统设计 |
| `docs/agent/sub-agents.mdx` | 子 Agent 系统 |
