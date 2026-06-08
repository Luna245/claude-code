# Claude Code 完整消息链路深度解析

> **范围：** 从用户在终端输入一条消息，到 Agentic Loop 调用工具，再到任务最终完成的完整流程。
>
> **深度：** 源码级，包含关键代码引用、数据结构变换、设计决策原因。

---

## 目录

1. [全景架构图](#1-全景架构图)
2. [第一层：用户输入捕获](#2-第一层用户输入捕获)
3. [第二层：输入预处理](#3-第二层输入预处理)
4. [第三层：Query 触发与上下文构建](#4-第三层query触发与上下文构建)
5. [第四层：query() 入口与 queryLoop 初始化](#5-第四层query入口与queryloop初始化)
6. [第五层：预处理管道（5 步压缩链）](#6-第五层预处理管道5步压缩链)
7. [第六层：流式 API 调用](#7-第六层流式api调用)
8. [第七层：流式事件处理状态机](#8-第七层流式事件处理状态机)
9. [第八层：工具执行系统](#9-第八层工具执行系统)
10. [第九层：权限检查](#10-第九层权限检查)
11. [第十层：循环决策（继续 vs 终止）](#11-第十层循环决策继续-vs-终止)
12. [第十一层：结果回流到 UI](#12-第十一层结果回流到ui)
13. [错误恢复全图](#13-错误恢复全图)
14. [关键数据结构变换图](#14-关键数据结构变换图)

---

## 1. 全景架构图

```
用户按下 Enter
     │
     ▼
┌─────────────────────────────────────────────────────────┐
│  终端 UI 层 (Ink/React)                                  │
│                                                         │
│  PromptInput              ─── 键盘输入捕获               │
│      │                                                  │
│      ▼                                                  │
│  onSubmit (REPL.tsx:3858)  ─── 分流：命令 vs 普通文本    │
│      │                                                  │
│      ▼                                                  │
│  handlePromptSubmit()      ─── 斜杠命令处理 / 消息构建   │
│      │                                                  │
│      ▼                                                  │
│  processUserInput()        ─── 内容块组装 / Hooks         │
│      │                                                  │
│      ▼                                                  │
│  onQuery (REPL.tsx:3507)   ─── 并发守卫 / 消息入队        │
│      │                                                  │
│      ▼                                                  │
│  onQueryImpl (REPL.tsx:3224) ─── System Prompt 构建       │
└─────────────────────────┬───────────────────────────────┘
                          │
                          ▼  for await (event of query(...))
┌─────────────────────────────────────────────────────────┐
│  Agentic Loop 层 (query.ts)                              │
│                                                         │
│  query()                   ─── Langfuse trace / 最终清理  │
│      │                                                  │
│      ▼                                                  │
│  queryLoop()               ─── while(true) 核心循环      │
│      │                                                  │
│      ├── [预处理管道]                                    │
│      │   applyToolResultBudget                          │
│      │   snipCompact                                    │
│      │   microcompact                                   │
│      │   contextCollapse                                │
│      │   autocompact                                    │
│      │                                                  │
│      ├── [API 调用]                                      │
│      │   callModel() → 流式事件                          │
│      │                                                  │
│      ├── [工具执行]                                      │
│      │   StreamingToolExecutor / runTools()             │
│      │                                                  │
│      └── [循环决策]                                      │
│          needsFollowUp? → continue / return              │
└─────────────────────────┬───────────────────────────────┘
                          │  yield StreamEvent
                          ▼
┌─────────────────────────────────────────────────────────┐
│  UI 更新层 (REPL.tsx)                                    │
│                                                         │
│  onQueryEvent()            ─── 处理每个 StreamEvent      │
│      │                                                  │
│      ▼                                                  │
│  setMessages / setStreamingText ─── React 状态更新       │
│      │                                                  │
│      ▼                                                  │
│  Messages.tsx 重新渲染      ─── 终端显示更新              │
└─────────────────────────────────────────────────────────┘
```

---

## 2. 第一层：用户输入捕获

### 组件树

```
REPL.tsx
└── PromptInput/PromptInput.tsx
    └── packages/@ant/ink/hooks/useInput.ts  ← 底层键盘事件
```

### PromptInput 的核心职责

`PromptInput` 组件本质上是一个受控的文本输入区域，它：
1. 通过 Ink 的 `useInput` hook 捕获每个按键
2. 维护 `inputBuffer`（多行文本缓冲区）
3. 检测 `Enter`（提交）vs `Shift+Enter`（换行）
4. 支持粘贴内容（图片 / 文本）的预处理

当用户按下 `Enter`，PromptInput 调用父组件传入的 `onSubmit` prop。

### onSubmit 的路由逻辑（REPL.tsx:3858）

```typescript
const onSubmit = useCallback(async (input: string, helpers, ...) => {
  // 1. 如果有 pipe 目标，路由到 pipe（不走 AI）
  if (routeToSelectedPipes(input)) { ... return }

  // 2. 斜杠命令（immediate 类型）直接执行，不走队列
  if (input.trim().startsWith('/') && matchingCommand?.immediate) {
    void executeImmediateCommand()
    return
  }

  // 3. 远程模式处理
  if (activeRemote.isRemoteMode) { ... }

  // 4. Idle 检测（长时间未操作提示刷新）
  // ...

  // 5. ✅ 正常路径：进入 handlePromptSubmit
  await handlePromptSubmit({ input, helpers, onQuery, ... })
}, [...deps])
```

**关键设计：** `onSubmit` 对 `messages` 的读取通过 `messagesRef.current`（而非 closure 中的 `messages`），原因是避免每次消息更新（约每轮 30 次 setMessages）都重建 `onSubmit`，导致内存泄漏（每次重建会 pin 住整个 REPL 渲染作用域 + messages 数组）。

---

## 3. 第二层：输入预处理

### handlePromptSubmit 的职责

**文件：** `src/utils/handlePromptSubmit.ts`

```typescript
export async function handlePromptSubmit(params) {
  // 1. 判断是否正在加载（如果在加载，将 input 入队等待）
  if (params.queryGuard.isActive || params.isExternalLoading) {
    enqueue({ value: input, mode })
    return
  }

  // 2. 调用 processUserInput 处理输入
  const result = await processUserInput({
    input,
    mode,
    context,
    pastedContents,
    ideSelection,
    messages,
    ...
  })

  // 3. 根据 result 决定是否触发 onQuery
  if (result.shouldQuery) {
    await params.onQuery(
      result.messages,
      abortController,
      true,
      result.allowedTools ?? [],
      result.model ?? mainLoopModel,
      ...
    )
  }
}
```

### processUserInput 的消息构建（processUserInput.ts）

这是消息从"用户输入字符串"变成"API 消息对象"的地方。

```typescript
export async function processUserInput(params) {
  const { input, mode, pastedContents, ideSelection } = params

  // 1. 处理斜杠命令
  if (input.startsWith('/')) {
    const command = parseSlashCommand(input)
    return processSlashCommand(command, ...)
  }

  // 2. 处理普通文本
  return processTextPrompt({
    input,
    pastedContents,  // 粘贴的图片/文本内容
    ideSelection,    // IDE 选中的代码片段
    ...
  })
}
```

**processTextPrompt 的输出（消息构建核心）：**

```typescript
// 用户输入："帮我修复 bug"
// + 粘贴了一张截图
// + IDE 选中了一段代码

// 输出的 UserMessage：
{
  type: 'user',
  role: 'user',
  message: {
    content: [
      { type: 'text', text: '帮我修复 bug' },
      { type: 'image', source: { type: 'base64', data: '...', media_type: 'image/png' } },
      { type: 'text', text: '<ide_selection>..代码片段..</ide_selection>' }
    ]
  },
  uuid: 'xxx-yyy-zzz',
  timestamp: 1234567890
}
```

### UserPromptSubmit Hooks

在 `processUserInput` 中，有一个关键的 hook 执行点：

```typescript
// processTextPrompt.ts
const hookResult = await executeUserPromptSubmitHooks(input, context)
if (hookResult.blocked) {
  // hook 阻止了提交，显示错误信息
  return { messages: [createSystemMessage(hookResult.reason)], shouldQuery: false }
}
// hook 可以修改消息内容
const finalInput = hookResult.modifiedInput ?? input
```

这是 Claude Code 的 hook 系统入口之一，允许外部扩展（如 IDE 插件）在消息发送前拦截/修改内容。

---

## 4. 第三层：Query 触发与上下文构建

### onQuery（REPL.tsx:3507）

```typescript
const onQuery = useCallback(async (
  newMessages: MessageType[],
  abortController: AbortController,
  shouldQuery: boolean,
  additionalAllowedTools: string[],
  mainLoopModelParam: string,
  ...
) => {
  // 1. 并发守卫：如果已有 query 在执行，将新消息入队
  const thisGeneration = queryGuard.tryStart()
  if (thisGeneration === null) {
    // 将用户消息文本入队，等当前 query 完成后处理
    newMessages.filter(m => m.type === 'user').forEach(msg => enqueue(msg))
    return false
  }

  try {
    // 2. 追加新消息到全局消息列表（立即更新 UI）
    setMessages(oldMessages => [...oldMessages, ...newMessages])

    // 3. 进入 onQueryImpl 执行 API 调用
    await onQueryImpl(latestMessages, newMessages, abortController, ...)
  } finally {
    // 4. 清理：结束 guard，重置加载状态，通知 bridge 客户端
    if (queryGuard.end(thisGeneration)) {
      resetLoadingState()
      await mrOnTurnComplete(messagesRef.current, aborted)
    }
  }
}, [...deps])
```

**QueryGuard 是什么：** 一个简单的状态机，防止并发调用 `onQuery`。`tryStart()` 原子性地从 idle→running，返回 generation 编号；`end(generation)` 确保只有最新的 generation 能重置状态，解决取消+重提交的竞争条件。

### onQueryImpl（REPL.tsx:3224）— 构建执行上下文

```typescript
const onQueryImpl = useCallback(async (
  messagesIncludingNewMessages,
  newMessages,
  abortController,
  shouldQuery,
  additionalAllowedTools,
  mainLoopModelParam,
  effort,
) => {
  // 1. 设置允许的工具（skill 的 frontmatter 可以限制工具集）
  store.setState(prev => ({
    ...prev,
    toolPermissionContext: {
      ...prev.toolPermissionContext,
      alwaysAllowRules: { command: additionalAllowedTools }
    }
  }))

  // 2. 构建 ToolUseContext（工具执行的完整上下文）
  const toolUseContext = getToolUseContext(
    messagesIncludingNewMessages,
    newMessages,
    abortController,
    mainLoopModelParam,
  )

  // 3. 并行构建 System Prompt（异步）
  const [defaultSystemPrompt, baseUserContext, systemContext] = await Promise.all([
    getSystemPrompt(freshTools, mainLoopModelParam, additionalWorkingDirs, freshMcpClients),
    getUserContext(),
    getSystemContext(),
  ])

  // 4. 组合最终 System Prompt
  const systemPrompt = buildEffectiveSystemPrompt({
    mainThreadAgentDefinition,
    toolUseContext,
    customSystemPrompt,  // 用户通过 --system-prompt 传入的自定义提示
    defaultSystemPrompt,
    appendSystemPrompt,  // 用户通过 --append-system-prompt 追加的内容
  })
  toolUseContext.renderedSystemPrompt = systemPrompt

  // 5. ✅ 调用 query()，消费所有 StreamEvent
  for await (const event of query({
    messages: messagesIncludingNewMessages,
    systemPrompt,
    userContext,
    systemContext,
    canUseTool,
    toolUseContext,
    querySource: 'repl_main_thread',
  })) {
    onQueryEvent(event)  // 每个事件立即更新 UI
  }
}, [...deps])
```

### System Prompt 的分层构建

```
buildEffectiveSystemPrompt()
    │
    ├── getSystemPrompt(tools, model, workingDirs, mcpClients)
    │       │
    │       ├── getCLISyspromptPrefix()       ← 固定工具使用规则（~2000 字）
    │       ├── getToolDescriptions(tools)    ← 工具列表（格式化 XML）
    │       └── getMcpToolDescriptions(...)   ← MCP 工具（动态）
    │
    ├── getUserContext()
    │       ├── 读取 CLAUDE.md 文件层级         ← 项目指令
    │       ├── 读取 memory files               ← ~/.claude/projects/.../memory/
    │       └── 读取 ~/.claude/CLAUDE.md        ← 全局用户指令
    │
    └── getSystemContext()
            ├── 当前日期/时间
            ├── git status / branch
            └── 当前工作目录
```

**注意：** System Prompt 里的工具描述用 XML 格式包裹，每个工具的 `description` 字段**直接影响 Claude 选择该工具的概率**。这是工具 Prompt Engineering 的关键所在。

---

## 5. 第四层：query() 入口与 queryLoop 初始化

**文件：** `src/query.ts:275`

### query() 的职责

```typescript
export async function* query(params: QueryParams): AsyncGenerator<...> {
  // 1. 创建或复用 Langfuse trace（子 Agent 复用父 Agent 的 trace）
  const langfuseTrace = params.toolUseContext.langfuseTrace ??
    (isLangfuseEnabled() ? createTrace(...) : null)

  let terminal: Terminal | undefined
  try {
    // 2. ✅ 进入核心循环
    terminal = yield* queryLoop(paramsWithTrace, consumedCommandUuids, consumedAutonomyCommands)
  } catch (error) {
    throw error
  } finally {
    // 3. 清理（无论正常结束、异常、还是被 .return() 中止）
    await finalizeAutonomyCommandsForTurn(...)  // 自主模式命令队列清理
    endTrace(langfuseTrace, ...)                // Langfuse trace 结束
    await flushLangfuse()                       // 释放 span 内存（防止 571MB 内存泄漏）
    gPerf.clearMarks()                          // 清理 JSC 原生 Performance 缓冲区
  }

  // 4. 通知命令生命周期完成（只有正常返回时到达这里，throw/abort 不会）
  for (const uuid of consumedCommandUuids) {
    notifyCommandLifecycle(uuid, 'completed')
  }
  return terminal!
}
```

### queryLoop() 的初始状态（query.ts:392）

```typescript
async function* queryLoop(params, consumedCommandUuids, consumedAutonomyCommands) {
  // 不可变参数（整个循环期间不变）
  const { systemPrompt, userContext, systemContext, canUseTool, ... } = params

  // 可变的跨迭代状态
  let state: State = {
    messages: params.messages,        // 消息历史（每轮可能更新）
    toolUseContext: params.toolUseContext,
    maxOutputTokensOverride: undefined,
    autoCompactTracking: undefined,
    maxOutputTokensRecoveryCount: 0,
    hasAttemptedReactiveCompact: false,
    turnCount: 1,
    pendingToolUseSummary: undefined,  // 上一轮的工具摘要（异步生成）
    stopHookActive: undefined,
    transition: undefined,             // 上一轮为什么继续（防死循环的关键）
  }

  // 快照不可变的环境状态（statsig/feature flags 状态在 entry 时固定）
  const config = buildQueryConfig()

  // 每个 query() 调用只 prefetch 一次记忆（不随循环迭代重复）
  using pendingMemoryPrefetch = startRelevantMemoryPrefetch(state.messages, state.toolUseContext)

  // ── 开始主循环 ──
  while (true) { ... }
}
```

**State 设计的妙处：** `state` 是一个不可变值对象，每次 `continue` 前必须 `state = { ...state, changed_fields }` 创建新对象。这样 `transition` 字段就能精确记录"上一轮为什么继续"，让当前迭代能检测特定的恢复状态，防止同一路径死循环。

---

## 6. 第五层：预处理管道（5 步压缩链）

每次循环迭代的第一步：把消息历史通过 5 个过滤器处理，生成适合 API 调用的 `messagesForQuery`。

```
messagesForQuery（原始消息历史）
     │
     ▼ 1. applyToolResultBudget()
     │    按 maxResultSizeChars 截断超长的工具结果
     │    超出部分存到 contentReplacementState，可按需恢复
     │
     ▼ 2. snipCompactIfNeeded()    [feature: HISTORY_SNIP]
     │    移除最旧的 N 轮消息，释放 context 空间
     │    记录 snipTokensFreed（传给 autocompact 调整阈值）
     │
     ▼ 3. microcompact()
     │    对超长的单个工具结果生成内联摘要（不压缩整段历史）
     │    使用 Haiku 模型，成本低，速度快
     │
     ▼ 4. applyCollapsesIfNeeded() [feature: CONTEXT_COLLAPSE]
     │    折叠历史中可逆的"信息块"（如重复的读文件结果）
     │    折叠是"暂存"的，出现 413 时可以"展开 drain"来恢复
     │
     ▼ 5. autocompact()
     │    当 token 数超过阈值（通常 context window 的 85%）时：
     │    调用 Claude 生成整段历史的摘要，替换原始历史
     │    这是"核弹级"压缩，不可逆
     │
     ▼
messagesForQuery（处理后，准备发给 API）
```

### 关键设计：snipTokensFreed 的传递

```typescript
// snip 释放了 2000 token
snipTokensFreed = snipResult.tokensFreed  // = 2000

// autocompact 的阈值检查必须扣除这个值
const currentTokens = tokenCountWithEstimation(messagesForQuery) - snipTokensFreed
// 否则会出现：snip 后 token 数已在阈值下，但 autocompact 用的是 snip 前的计数，
// 导致触发不必要的压缩（"双重压缩"）
```

### autocompact 的触发条件

```typescript
// autoCompact.ts
export function calculateTokenWarningState(tokenCount, model) {
  const contextWindow = getEffectiveContextWindowSize(model)
  const threshold = contextWindow * 0.85  // 85% 时触发 autocompact

  return {
    isAtWarningLevel: tokenCount > threshold * 0.75,   // 75%: 黄色警告
    isAtBlockingLimit: tokenCount > threshold,          // 85%: 触发压缩
  }
}
```

---

## 7. 第六层：流式 API 调用

### callModel 的调用参数

```typescript
for await (const message of deps.callModel({
  messages: prependUserContext(messagesForQuery, userContext), // ← 把 userContext 合并到第一条消息
  systemPrompt: fullSystemPrompt,
  thinkingConfig: toolUseContext.options.thinkingConfig,
  tools: toolUseContext.options.tools,
  signal: toolUseContext.abortController.signal,  // ← 中断控制
  options: {
    model: currentModel,
    fastMode: appState.fastMode,
    querySource,
    agents: toolUseContext.options.agentDefinitions.activeAgents,
    maxOutputTokensOverride,
    mcpTools: appState.mcp.tools,
    ...
  }
})) { ... }
```

### claude.ts：构建 API 请求

**文件：** `src/services/api/claude.ts`

```typescript
// 关键：API 请求参数的构建
const requestParams: BetaMessageStreamParams = {
  model: resolvedModel,
  max_tokens: maxOutputTokens,  // 默认 8192，可被 maxOutputTokensOverride 覆盖
  messages: normalizeMessagesForAPI(messages, tools),  // 清理 UI 专用字段
  system: buildSystemPromptWithCacheControl(systemPrompt),  // 注入 cache_control
  tools: buildToolSchemas(tools),  // 工具 JSON Schema
  betas: getMergedBetas(model, tools, options),  // beta 功能标志
  thinking: thinkingConfig,   // 扩展思考模式配置
  stream: true,
}

// 发起流式请求
const stream: Stream<BetaRawMessageStreamEvent> = await client.beta.messages.stream(requestParams)
```

### normalizeMessagesForAPI 的作用

这个函数把 UI 内部格式的消息转换为 Anthropic API 格式：

```typescript
// UI 格式（包含大量 UI 专用字段）：
{
  type: 'user',
  role: 'user',
  uuid: 'xxx',
  timestamp: 1234567890,
  isMeta: false,
  toolUseResult: { ... },  // UI 渲染用的原始工具输出
  message: {
    content: [{ type: 'text', text: '...' }]
  }
}

// API 格式（只保留 Anthropic SDK 需要的字段）：
{
  role: 'user',
  content: [{ type: 'text', text: '...' }]
}
```

### Prompt Cache 的 cache_control 注入

```typescript
// api.ts: splitSysPromptPrefix
// System Prompt 被分成两段：
// 1. 前缀（工具描述等固定内容）← 添加 cache_control: { type: 'ephemeral' }
// 2. 动态部分（git status、CLAUDE.md 等每次变化的内容）← 不加 cache_control

// 这样前缀部分可以被缓存，节省 60-70% 的 token 费用
```

---

## 8. 第七层：流式事件处理状态机

**文件：** `src/services/api/claude.ts` 的 `queryModelWithStreaming()` 函数

### 6 种事件类型处理

```
API 返回的 BetaRawMessageStreamEvent 流：
─────────────────────────────────────────────────────────

message_start
  └── 初始化 partialMessage，记录 TTFT（首字节延迟）

content_block_start (type=text, index=0)
  └── 创建 TextBlock，初始化 text = ''

content_block_delta (text_delta)
  └── text += delta.text  ← 逐 token 累加

content_block_stop (index=0)
  └── 构建完整 AssistantMessage，yield！
      → REPL 收到，立即渲染文字（打字机效果）

content_block_start (type=tool_use, index=1)
  └── 创建 ToolUseBlock，name = 'Bash', input = ''

content_block_delta (input_json_delta)
  └── input += delta.partial_json  ← JSON 字符串增量拼接
      '{"command":"' → 'ls -la"}' → 完整 JSON

content_block_stop (index=1)
  └── JSON.parse(input) → 构建完整 ToolUseBlock，yield AssistantMessage
      → REPL 收到，渲染工具调用卡片
      → StreamingToolExecutor 接收 tool_use block，立即开始执行工具！

message_delta (stop_reason='tool_use')
  └── 回写 stop_reason 到最后一条 AssistantMessage
      （回写：直接修改 message.stop_reason，不创建新对象）

message_stop
  └── 流结束标记（无操作）
```

### 关键设计：stop_reason 的"回写"

```typescript
// 为什么 stop_reason 要"回写"而不是在 content_block_stop 时就有？
// 因为 stop_reason 在 message_delta 事件才确定（在所有 content_block 处理完之后）
// 但 AssistantMessage 在 content_block_stop 就已经 yield 出去了

// 解决方案：直接修改对象属性（不用 Object.assign 或新对象）
// 这样可以保持 prompt cache 的字节一致性（同一个对象引用）
if (part.type === 'message_delta') {
  lastAssistantMessage.message.stop_reason = part.delta.stop_reason  // 直接改属性
  lastAssistantMessage.message.usage = { ...lastAssistantMessage.message.usage, ...part.usage }
}
```

### backfillObservableInput 的 clone 守卫

```typescript
// query.ts:968
// 问题：tool.backfillObservableInput 可能扩展 input（添加新字段，如 expanded file_path）
// 如果直接修改 assistantMessage 对象，会破坏 prompt cache（字节不一致）
// 解决方案：只在"真的添加了新字段"时才克隆，否则直接 yield 原对象

let yieldMessage = message  // 默认不克隆
for (const block of contentArr) {
  if (block.type === 'tool_use' && tool.backfillObservableInput) {
    const inputCopy = { ...block.input }
    tool.backfillObservableInput(inputCopy)
    // 关键守卫：只有新增了字段（不是覆盖），才需要克隆
    const addedFields = Object.keys(inputCopy).some(k => !(k in block.input))
    if (addedFields) {
      yieldMessage = { ...message, message: { ..., content: clonedContent } }  // 克隆
    }
  }
}
yield yieldMessage  // 原对象留在 assistantMessages 中（用于 API 回传）
```

---

## 9. 第八层：工具执行系统

### 两种工具执行模式

```typescript
// query.ts:1654
const toolUpdates = streamingToolExecutor
  ? streamingToolExecutor.getRemainingResults()   // 模式 1：流式并行
  : runTools(toolUseBlocks, assistantMessages, canUseTool, toolUseContext)  // 模式 2：标准批量
```

### 模式 1：StreamingToolExecutor（流式并行，更快）

```
时间轴：
─────────────────────────────────────────────────────
           流式阶段                    工具执行阶段
─────────────────────────────────────────────────────
t=0:  content_block_stop (tool_use: Bash)
       └── streamingToolExecutor.addTool(toolBlock)
           └── 立即开始执行 Bash 工具！（不等流结束）

t=1:  content_block_stop (tool_use: FileRead)
       └── streamingToolExecutor.addTool(toolBlock)
           └── 立即开始执行 FileRead 工具！

t=2:  message_stop（流结束）

t=3:  streamingToolExecutor.getRemainingResults()
       └── Bash 工具已完成 → yield result
       └── FileRead 工具已完成 → yield result

─────────────────────────────────────────────────────
结果：工具执行时间与流式时间重叠，总时间更短
```

### 模式 2：runTools（标准批量，更稳）

**文件：** `src/services/tools/toolOrchestration.ts`

```typescript
export async function* runTools(toolUseMessages, assistantMessages, canUseTool, context) {
  // 1. 将工具调用分组（并发安全 vs 非并发安全）
  for (const { isConcurrencySafe, blocks } of partitionToolCalls(toolUseMessages, context)) {
    if (isConcurrencySafe) {
      // 只读工具（Glob, Grep, FileRead）→ 并发执行，最高 10 个同时运行
      for await (const update of runToolsConcurrently(blocks, ...)) {
        yield { message: update.message, newContext: currentContext }
      }
    } else {
      // 写工具（FileEdit, Bash, FileWrite）→ 串行执行
      for await (const update of runToolsSerially(blocks, ...)) {
        yield { message: update.message, newContext: currentContext }
      }
    }
  }
}
```

**并发安全的判断依据（partitionToolCalls）：**
- `FileReadTool`, `GlobTool`, `GrepTool` → 只读，可并发
- `BashTool`, `FileEditTool`, `FileWriteTool` → 有副作用，串行
- 两种类型混合时：先并发执行只读部分，再串行执行写入部分

### 单个工具的执行流程（toolExecution.ts）

```typescript
export async function* runToolUse(
  toolUseBlock: ToolUseBlock,
  canUseTool: CanUseToolFn,
  toolUseContext: ToolUseContext,
) {
  // 1. 查找工具（内置工具 or MCP 工具）
  const tool = findToolByName(toolUseContext.options.tools, toolUseBlock.name)
  if (!tool) {
    yield createToolResult(toolUseBlock.id, `Tool not found: ${toolUseBlock.name}`)
    return
  }

  // 2. ✅ 权限检查（见第 9 层）
  const permissionResult = await canUseTool(tool, toolUseBlock.input, toolUseContext)

  if (permissionResult.result === 'deny') {
    yield createToolResult(toolUseBlock.id, `Permission denied: ${permissionResult.reason}`)
    return
  }

  if (permissionResult.result === 'ask') {
    // 暂停执行，等待用户在 UI 确认
    const userDecision = await waitForUserConfirmation(toolUseBlock, permissionResult)
    if (!userDecision.allowed) {
      yield createToolResult(toolUseBlock.id, CANCEL_MESSAGE)
      return
    }
  }

  // 3. 执行工具
  const startTime = Date.now()
  try {
    for await (const progress of tool.call(toolUseBlock.input, toolUseContext)) {
      if (isToolProgressData(progress)) {
        // yield 进度更新（显示工具执行中的状态）
        yield createProgressMessage(toolUseBlock, progress)
      } else {
        // 最终结果
        const toolResult = createToolResultMessage(toolUseBlock, progress)
        yield toolResult
      }
    }
  } catch (error) {
    yield createToolResult(toolUseBlock.id, `Error: ${errorMessage(error)}`)
  } finally {
    addToToolDuration(Date.now() - startTime)
  }
}
```

---

## 10. 第九层：权限检查

### canUseTool 的完整判断链

**文件：** `src/hooks/useCanUseTool.tsx`

```
canUseTool(tool, input, context)
    │
    ├── 1. 检查 PermissionMode
    │       bypassPermissions → allow（CI 模式，无需确认）
    │       plan mode → 只读工具 allow，其他 deny
    │
    ├── 2. 检查 alwaysAllowRules
    │       命令在白名单中 → allow（用户之前选择了"永远允许"）
    │
    ├── 3. 调用 tool.checkPermissions(input, context)
    │       每个工具可以有自己的权限逻辑
    │       │
    │       └── BashTool 的权限检查（bashPermissions.ts）：
    │               ├── 解析命令的语义（是否只读？有无副作用？）
    │               ├── 检查路径是否在允许的工作目录内
    │               ├── 检测危险命令（rm -rf, format, etc.）
    │               └── PermissionResult: allow / deny / ask
    │
    └── 4. 最终结果
            allow → 继续执行
            deny  → 返回错误信息给 Claude
            ask   → UI 显示权限请求对话框，等待用户确认
```

### Bash 工具的命令语义分析

**文件：** `packages/builtin-tools/src/tools/BashTool/bashSecurity.ts`（2634 行）

这是整个权限系统最复杂的部分。它必须在不实际执行命令的情况下，判断一条 bash 命令是否有副作用。

```typescript
// 例子：分析 "ls -la | grep .ts" 的语义
// 1. 解析管道：["ls -la", "grep .ts"]
// 2. ls -la → 只读（列目录）
// 3. grep .ts → 只读（过滤）
// 4. 整体 → 只读，allow

// 例子：分析 "cat file.txt | tee output.txt" 的语义
// 1. 解析管道：["cat file.txt", "tee output.txt"]
// 2. cat file.txt → 只读
// 3. tee output.txt → 写文件！
// 4. 整体 → 写操作，ask（需要用户确认）

// 例子：分析 "find . -name '*.log' -exec rm {} \\;" 的语义
// 1. 检测到 -exec 参数中有 rm
// 2. rm → 删除文件
// 3. 整体 → 危险删除操作，ask + 警告
```

### 权限确认 UI

当 `permissionResult.result === 'ask'` 时，工具执行器**暂停**，UI 显示 `PermissionRequest` 对话框。

用户的选择：
- **Allow** → 本次允许，继续执行
- **Allow and Remember** → 加入 `alwaysAllowRules`，本次 + 未来同类命令都允许
- **Deny** → 拒绝，工具返回 `CANCEL_MESSAGE`

---

## 11. 第十层：循环决策（继续 vs 终止）

这是 query.ts 的"大脑"，决定每轮迭代后是继续还是退出。

### 终止条件列表

| 终止原因 | 触发位置 | 条件 |
|---------|---------|------|
| `completed` | 行 1632 | AI 未发出 tool_use，stop hooks 通过 |
| `blocking_limit` | 行 829 | Token 数超过硬限制（auto-compact 关闭时）|
| `aborted_streaming` | 行 1323 | 用户 Ctrl+C（流式阶段）|
| `aborted_tools` | 行 1764 | 用户 Ctrl+C（工具执行阶段）|
| `stop_hook_prevented` | 行 1554 | Stop hook 返回 preventContinuation |
| `model_error` | 行 1537 | API 返回错误（限流、认证等）|
| `prompt_too_long` | 行 1447 | 413 错误且所有恢复路径耗尽 |
| `image_error` | 行 1447 | 图片/PDF 超大且 reactive compact 无效 |
| `max_turns` | 行 1755 | 轮次超过 maxTurns |
| `hook_stopped` | 行 1789 | Post-sampling hook 停止循环 |

### 继续条件（continue 路径）

```
needsFollowUp = true（有 tool_use block）
    ↓
执行工具 → 收集 toolResults
    ↓
state = {
  messages: [...messagesForQuery, ...assistantMessages, ...toolResults],
  turnCount: turnCount + 1,
  transition: { reason: 'next_turn' },
  ...
}
continue  ← 下一轮循环
```

### 恢复路径（特殊的 continue）

```
isWithheld413 = true（收到了 413 但被暂扣）
    ↓
优先尝试 collapse_drain_retry（便宜，commit 暂存的折叠）
    ↓（失败）
尝试 reactive_compact_retry（生成摘要，替换历史）
    ↓（失败）
yield lastMessage（显示错误）
return { reason: 'prompt_too_long' }
```

```
isWithheldMaxOutputTokens = true（输出被截断）
    ↓
首次：提升 maxOutputTokensOverride = 64K
state = { maxOutputTokensOverride: ESCALATED_MAX_TOKENS, transition: { reason: 'max_output_tokens_escalate' } }
continue  ← 用更大的 output limit 重试
    ↓（还是截断）
注入恢复消息 "Output token limit hit. Resume directly..."
state = { maxOutputTokensRecoveryCount: count+1, transition: { reason: 'max_output_tokens_recovery' } }
continue  ← 最多重试 3 次
    ↓（第 4 次还是截断）
yield 错误消息
return { reason: 'model_error' }
```

### Stop Hooks（任务完成前的最后检查点）

```typescript
// query/stopHooks.ts
// 当 needsFollowUp = false（AI 表示任务完成）时调用

const stopHookResult = yield* handleStopHooks(
  messagesForQuery,
  assistantMessages,
  systemPrompt,
  userContext,
  systemContext,
  toolUseContext,
  querySource,
  stopHookActive,
)

if (stopHookResult.preventContinuation) {
  return { reason: 'stop_hook_prevented' }  // 钩子强制终止
}

if (stopHookResult.blockingErrors.length > 0) {
  // 钩子注入阻塞错误消息，强制 AI 重新审视结果
  state = {
    messages: [...messagesForQuery, ...assistantMessages, ...stopHookResult.blockingErrors],
    stopHookActive: true,
    transition: { reason: 'stop_hook_blocking' },
    // ⚠️ 保留 hasAttemptedReactiveCompact，防止死循环
    hasAttemptedReactiveCompact,
    ...
  }
  continue  ← AI 需要重新思考
}

// 全部通过 → 真正结束
return { reason: 'completed' }
```

**Stop Hook 的使用场景：**
- 验证 Claude 是否真的完成了任务（如测试是否通过）
- 检查输出格式是否符合要求
- 在任务完成时触发通知

---

## 12. 第十一层：结果回流到 UI

### StreamEvent 的类型和处理

`query()` 是一个 AsyncGenerator，每次 `yield` 产出一个 StreamEvent，REPL 的 `onQueryEvent` 处理器实时接收。

**文件：** `src/screens/REPL.tsx:3079`

```typescript
const onQueryEvent = useCallback((event: StreamEvent | Message | ...) => {
  switch (event.type) {
    case 'stream_request_start':
      // API 请求刚开始，显示 spinner
      break

    case 'assistant':
      // 收到 AI 文本或工具调用
      if (hasTextContent(event)) {
        setStreamingText(extractText(event))  // 实时更新打字机效果
      }
      setMessages(prev => [...prev, event])  // 添加到消息历史

    case 'user':
      // 工具结果（tool_result）
      setMessages(prev => [...prev, event])  // 显示工具执行结果

    case 'progress':
      // 工具执行进度（如 Bash 命令的实时输出）
      setStreamingToolUses(prev => updateProgress(prev, event))

    case 'system':
      // 系统消息（compact 边界、错误提示等）
      setMessages(prev => [...prev, event])

    case 'tombstone':
      // 删除孤儿消息（模型 fallback 时）
      setMessages(prev => prev.filter(m => m.uuid !== event.message.uuid))
  }
}, [...deps])
```

### setMessages 的实现细节

`setMessages` 不是简单的 React `useState` setter。它：
1. 更新 React 状态（触发重渲染）
2. **同步更新** `messagesRef.current`（供 `onSubmit` 等 callback 即时读取，避免 stale closure）
3. 在会话转录功能开启时，同步写入磁盘

```typescript
// REPL.tsx 中 setMessages 的自定义实现
const setMessages = useCallback((updater) => {
  setMessagesState(prev => {
    const next = typeof updater === 'function' ? updater(prev) : updater
    messagesRef.current = next  // 同步更新 ref（不等 React 调度）
    return next
  })
}, [])
```

### React 渲染流程

```
setMessages(...)
     │
     ▼ React 调度（batched）
REPL 重新渲染
     │
     ▼
<Messages messages={messagesRef.current} />
     │
     ▼
<VirtualMessageList>  ← 虚拟滚动（只渲染可见区域）
     │
     ├── AssistantMessage → 渲染 AI 文字（Markdown → ANSI）
     ├── ToolUseBlock → 渲染工具调用卡片（展开/折叠）
     └── ToolResultBlock → 渲染工具输出（截断显示）
```

### 流式文字的"打字机效果"

```typescript
// 流式阶段：实时更新
setStreamingText(extractedText)  // 每个 content_block_delta 事件触发

// 流结束：把 streaming text 转为正式消息
// （content_block_stop 时已 yield AssistantMessage，
//  UI 转而渲染 message，streamingText 清空）
setStreamingText(null)
```

---

## 13. 错误恢复全图

```
                   流式 API 调用
                        │
              ┌─────────┴──────────┐
              │                    │
         成功完成              发生错误
              │                    │
         处理 tool_use         ┌───┴────────────────────────────┐
              │                │                                │
         检测 stop_reason     413 (prompt too long)         max_output_tokens
                           (error withheld)               (error withheld)
                                │                                │
                    ┌───────────┴──────────┐            ┌───────┴──────────┐
                    │                      │            │                  │
             context_collapse       reactive_compact  escalate to 64K  recovery msg
             drain (commit          (generate summary, (retry same req) (inject hint,
             staged collapses)      replace history)   max 1 time)     max 3 times)
                    │                      │            │                  │
             ┌──────┴───────┐      ┌───────┴──────┐    │            ┌─────┴──────┐
             │              │      │              │    │            │            │
          Success         Fail   Success         Fail │         Success       Fail
          continue        │      continue        │    │         continue  surface error
             │            └──────►reactive_compact    │
             │                       │                │
             │                   ┌───┴───┐            │
             │                   │       │            │
             │               Success   Fail           │
             │               continue  │              │
             │                         ▼              │
             │                   surface error ◄──────┘
             │                   return 'prompt_too_long' / 'model_error'
             │
          ┌──┴──────────────────────────────┐
          │         正常完成路径             │
          │                                 │
          │   needsFollowUp = false         │
          │         ↓                       │
          │   handleStopHooks()             │
          │         ↓                       │
          │   preventContinuation? → return │
          │   blockingErrors?    → continue │
          │   pass               → return   │
          │         ↓                       │
          │   TOKEN_BUDGET check            │
          │         ↓                       │
          │   return 'completed' ✅          │
          └─────────────────────────────────┘
```

---

## 14. 关键数据结构变换图

### 从用户输入到 API 请求

```
用户键入: "帮我找出 src/ 目录下所有未使用的 import"
    │
    ▼ processTextPrompt()
UserMessage {
  type: 'user',
  uuid: 'a1b2c3',
  message: { content: [{ type: 'text', text: '帮我找出...' }] },
  isMeta: false,
}
    │
    ▼ normalizeMessagesForAPI()
API 格式: { role: 'user', content: [{ type: 'text', text: '...' }] }
    │
    ▼ + system prompt + tools schema
BetaMessageStreamParams {
  model: 'claude-sonnet-4-6',
  max_tokens: 8192,
  messages: [{ role: 'user', content: [...] }],
  system: [{ type: 'text', text: '...', cache_control: { type: 'ephemeral' } }],
  tools: [{ name: 'Bash', description: '...', input_schema: {...} }, ...],
  stream: true,
}
```

### API 响应到工具执行

```
API 流式响应 (BetaRawMessageStreamEvent):
  content_block_start  (tool_use, name='Glob', index=0)
  content_block_delta  {"pattern": "**
  content_block_delta  /*.ts", "path": "src/"}
  content_block_stop   (index=0)

    │
    ▼ 流式处理器组装
AssistantMessage {
  type: 'assistant',
  message: {
    content: [{
      type: 'tool_use',
      id: 'toolu_01XXX',
      name: 'Glob',
      input: { pattern: '**/*.ts', path: 'src/' }
    }],
    stop_reason: 'tool_use',  // ← 在 message_delta 事件回写
  }
}

    │
    ▼ 工具执行
ToolUseBlock: { id: 'toolu_01XXX', name: 'Glob', input: {...} }
    │
    ▼ GlobTool.call(input, context)
    ├── yield ProgressData { type: 'running', ... }
    └── return ToolResult { output: ['src/utils/foo.ts', 'src/main.ts', ...] }

    │
    ▼ createToolResultMessage()
UserMessage {
  type: 'user',
  message: {
    content: [{
      type: 'tool_result',
      tool_use_id: 'toolu_01XXX',
      content: 'src/utils/foo.ts\nsrc/main.ts\n...'
    }]
  }
}

    │
    ▼ 追加到 messages，开始下一轮循环
messages = [
  UserMessage (原始用户问题),
  AssistantMessage (tool_use: Glob),
  UserMessage (tool_result: 42 个文件),
  ... 后续轮次 ...
]
```

### 最终完成时的消息历史

```
messages (完整会话记录):
  [0] UserMessage    "帮我找出 src/ 目录下所有未使用的 import"
  [1] AssistantMessage  "我来帮你检查。" + tool_use: Glob("**/*.ts")
  [2] UserMessage    tool_result: ["src/utils/foo.ts", "src/main.ts", ...]
  [3] AssistantMessage  tool_use: Grep("^import", "src/**/*.ts")
  [4] UserMessage    tool_result: [120 条 import 语句]
  [5] AssistantMessage  tool_use: FileEdit("src/utils/foo.ts", ...)
  [6] UserMessage    tool_result: "File updated successfully"
  [7] AssistantMessage  "已清理 3 个文件中的 5 条未使用 import：..."
       stop_reason: 'end_turn'  ← needsFollowUp = false → 循环结束
```

---

## 附录：关键函数调用链速查

```
用户 Enter
  └── REPL.tsx:onSubmit
      └── handlePromptSubmit.ts:handlePromptSubmit
          └── processUserInput/processUserInput.ts:processUserInput
              └── processUserInput/processTextPrompt.ts:processTextPrompt
                  └── hooks.ts:executeUserPromptSubmitHooks  [hook 系统]
          └── REPL.tsx:onQuery
              └── QueryGuard.tryStart()
              └── REPL.tsx:onQueryImpl
                  ├── context.ts:getSystemPrompt / getUserContext / getSystemContext
                  ├── utils/systemPrompt.ts:buildEffectiveSystemPrompt
                  └── query.ts:query()    [for await]
                      └── query.ts:queryLoop()    [while true]
                          ├── utils/toolResultStorage.ts:applyToolResultBudget
                          ├── services/compact/snipCompact.ts:snipCompactIfNeeded
                          ├── services/compact/microCompact.ts:microcompact
                          ├── services/contextCollapse/index.ts:applyCollapsesIfNeeded
                          ├── services/compact/autoCompact.ts:autocompact
                          └── services/api/claude.ts:queryModelWithStreaming()    [for await]
                              └── @anthropic-ai/sdk:client.beta.messages.stream()
                          └── services/tools/toolOrchestration.ts:runTools()
                              └── services/tools/toolExecution.ts:runToolUse()
                                  ├── hooks/useCanUseTool.ts:canUseTool()
                                  ├── components/permissions/PermissionRequest.tsx  [UI 弹窗]
                                  └── tools/BashTool/BashTool.ts:call()  [工具执行]
                          └── query/stopHooks.ts:handleStopHooks()
              └── REPL.tsx:onQueryEvent()    [每个 yield 触发]
                  └── setMessages() → React 重渲染 → 终端显示更新
```
