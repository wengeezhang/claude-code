# Claude Code 源码深度分析

> 本文档基于 `@anthropic-ai/claude-code` npm 包通过 sourcemap 还原出的源码进行分析。
> 仓库根目录：`/Users/bytedance/jimmie/git/claude-code`
> 仅用于研究与学习目的，原始代码版权归 Anthropic 所有。

## 目录

1. [核心 Agent Loop 架构](#一核心-agent-loop-架构)
2. [压缩链路详解](#二压缩链路详解)
3. [React/Ink 终端应用](#三reactink-终端应用)
4. [Client SDK vs Agent SDK vs Claude Code CLI](#四client-sdk-vs-agent-sdk-vs-claude-code-cli)
5. [为什么 npm 包会被称为"源码泄露"](#五为什么-npm-包会被称为源码泄露)
6. [Streaming 输入模式](#六streaming-输入模式)
7. [Agent 执行中的用户输入处理](#七agent-执行中的用户输入处理)
8. [LSP 工具](#八lsp-工具)
9. [Prompt Cache 机制](#九prompt-cache-机制)

---

## 一、核心 Agent Loop 架构

### 心智地图

```
                       ┌──────────────────────────────────────┐
                       │         QueryEngine.submitMessage()    │  AsyncGenerator<SDKMessage>
                       │  ── wrap canUseTool                    │
                       │  ── fetchSystemPromptParts             │
                       │  ── processUserInput                   │
                       │  ── persist transcript                 │
                       │  ── yield system_init                  │
                       │  ── for await ... of query({...})      │
                       │  ── track usage / denials / budget     │
                       └──────────────────────────────────────┘
                                       ↓
                       ┌──────────────────────────────────────┐
                       │            queryLoop (while true)      │
                       │     6 stages per turn:                 │
                       │  ① compression pipeline                │
                       │  ② callModel streaming                 │
                       │  ③ error recovery (413/media/OTK)      │
                       │  ④ stopHooks                           │
                       │  ⑤ tool execution (runTools)           │
                       │  ⑥ attachments + next turn             │
                       └──────────────────────────────────────┘
                                       ↓
                       ┌──────────────────────────────────────┐
                       │           runTools / runToolUse        │
                       │  ── partitionToolCalls (并发批 vs 串行批) │
                       │  ── canUseTool gate (7-step)           │
                       │  ── tool.call() → tool_result          │
                       └──────────────────────────────────────┘
```

### QueryEngine（顶层编排者）

`/src/QueryEngine.ts` 中的 `QueryEngine` 类，核心方法 `submitMessage()` 是一个 `AsyncGenerator<SDKMessage>`。每调用一次相当于把"一条用户消息"喂进系统并消费回所有产生的事件流。

关键流程：
1. **包装 canUseTool**：把外部传入的权限回调包成统一接口
2. **fetchSystemPromptParts**：拉取 system prompt 各组成部分（Claude Code 主提示词、CLAUDE.md 内容、用户上下文）
3. **processUserInput**：消化用户输入，处理斜杠命令和 bash 命令前缀
4. **持久化 transcript**：写入会话日志 jsonl
5. **yield system_init**：发出第一条系统初始化事件
6. **for await of query()**：进入主循环消费消息流
7. **追踪 usage、denials、budget**：累积本次 session 的 token 用量与权限拒绝事件

### queryLoop 的 6 阶段

`/src/query.ts` 的 `queryLoop()` 是一个 `while(true)` 循环，每一轮 = 一次 LLM 请求 + 一批工具执行。State 类型包含：

```typescript
type State = {
  messages: Message[]
  toolUseContext: ToolUseContext
  autoCompactTracking: { consecutiveFailures: number, ... }
  maxOutputTokensRecoveryCount: number
  hasAttemptedReactiveCompact: boolean
  transition: 'continue' | 'done'
}
```

每一轮执行的 6 个阶段：

**Stage 1 — 压缩流水线**：依次跑 `applyToolResultBudget` → `snipCompact`（feature gated）→ `microcompact` → `contextCollapse`（feature gated）→ `autoCompactIfNeeded` → 阻塞限制检查。详见第二章。

**Stage 2 — callModel streaming**：通过 `queryModelWithStreaming()`（`/src/services/api/claude.ts`）发起 SSE 流式请求，逐 chunk 拼接 assistant 消息。

**Stage 3 — 错误恢复**：处理 fallback、413（prompt_too_long）、media_size_error、max_output_tokens 这几类可恢复错误。典型模式是"withhold-then-recover"——先把超量内容暂时移除、重发请求，成功后再补回。

**Stage 4 — stopHooks**：执行用户配置的 hook，可能改变控制流。

**Stage 5 — tool execution**：调 `runTools()`（详见下文）。

**Stage 6 — attachments + next turn**：把 tool_result 拼回 messages 数组，准备下一轮输入。

### Tool 抽象

`/src/Tool.ts` 定义了 `Tool<Input, Output, P>` 接口，约 40 个字段，覆盖：
- 元信息：name、description、inputSchema、outputSchema
- 行为标记：isReadOnly、isConcurrencySafe、isDestructive、isEnabled
- 权限：checkPermissions、requiresUserInteraction
- 执行：call、validateInput、isValidForRender
- UI：renderToolUseMessage、renderResultForAssistant

`buildTool()` 工厂结合 `TOOL_DEFAULTS`（默认值：isEnabled=true、isConcurrencySafe=false、isReadOnly=false、isDestructive=false、checkPermissions=allow）让每个工具只声明自己关心的字段。

### runTools 的并发调度

`/src/services/tools/toolOrchestration.ts` 的 `runTools()` 把 `tool_use` 块列表按并发安全性分批：

```typescript
function partitionToolCalls(toolUseMessages, ctx): Batch[]
// 连续的 isConcurrencySafe 工具被打包到一起 → runToolsConcurrently
// 不安全的工具单独成批 → runToolsSerially
```

并发上限默认 10，可通过 `CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY` 环境变量调整。这是为什么并行用 Read/Grep/Glob 速度飞快（属于 isConcurrencySafe），而 Edit/Write/Bash 必须串行（会改变世界状态）。

### 7 步权限 Gate

`/src/utils/permissions/permissions.ts` 的 `hasPermissionsToUseToolInner()` 按顺序检查：

1. **1a — 全局 deny 规则**：用户配置的 `permissions.deny` 直接拒
2. **1b — 全局 ask 规则**：`permissions.ask` 强制询问用户
3. **1c — tool.checkPermissions**：工具自带的权限逻辑
4. **1d — 工具自定义 deny**：返回 deny 直接拒
5. **1e — requiresUserInteraction**：需要用户介入的强制询问
6. **1f — 内容特定 ask**：基于工具输入内容的细粒度询问（如 Bash 命令前缀）
7. **1g — safety check**：分类器判定是否危险

这 7 步组合让"自动允许已知安全操作 + 询问可能危险操作 + 拒绝明显有害操作"成为可能。

### ToolUseContext

横贯整个调用链的状态聚合器，包含：
- options（工具池、模型、配置）
- abortController（用于 ESC 中止）
- agentId、querySource（区分主线 vs subagent）
- readFileState（已读文件追踪）
- appState（应用全局状态）
- setInProgressToolUseIDs（追踪正在执行的工具）

### 测试性与依赖注入

`/src/query/deps.ts` 把可替换依赖收口到 `QueryDeps`：

```typescript
type QueryDeps = {
  callModel: ...,
  microcompact: ...,
  autocompact: ...,
  uuid: () => string,
}
```

测试里可以用 mock 替换这些，让 queryLoop 的逻辑能在不调真实 API、不真做压缩的情况下被驱动。

---

## 二、压缩链路详解

### 总览

```
                    ┌───────────────── queryLoop / 每一轮开头 ─────────────────┐
                    │                                                          │
  上一轮工具输出 → ① applyToolResultBudget    (就地裁剪超大 tool_result)
                    ↓
                   ② snipCompact              (feature('HISTORY_SNIP'))
                    ↓
                   ③ microCompact             (time-based / cached-MC)
                    ↓
                   ④ contextCollapse          (feature('CONTEXT_COLLAPSE'))
                    ↓
                   ⑤ autoCompactIfNeeded ──┬── trySessionMemoryCompaction
                    │                       └── compactConversation
                    ↓
                   ⑥ blocking-limit check     (PROMPT_TOO_LONG_ERROR_MESSAGE)

       旁路：reactive-compact （on 413 / media）
       旁路：apiMicrocompact  （服务端 context_edits）
       收尾：runPostCompactCleanup
```

设计原则：**渐进地放弃信息**。从只改 tool_result 的内容，到按组丢消息，再到彻底总结重建，越靠后的手段代价越高、对 prompt cache 命中越不友好。

### 1. applyToolResultBudget

第 379 行附近，每轮开头。遍历上一轮工具产出的 `tool_result`，按每个工具声明的 `maxResultSizeChars` 做尾部截断，插入 `[truncated …]` 占位。不调 LLM，对 prompt cache 前缀几乎无害。

### 2. snipCompact

被 `feature('HISTORY_SNIP')` 门控。返回 `{ messages, boundaryMessage, snipTokensFreed }`。基于 `groupMessagesByApiRound` 整组丢弃，避免悬空的 tool_use/tool_result。`snipTokensFreed` 沿 queryLoop 生命期传递，用于后续 blocking-limit 判定时的扣减。

### 3. microCompact（双路）

`/src/services/compact/microCompact.ts`。入口：

```typescript
microcompactMessages(messages, toolUseContext?, querySource?): Promise<{messages}>
```

可清白名单：FILE_READ、SHELL_*、GREP、GLOB、WEB_SEARCH、WEB_FETCH、FILE_EDIT、FILE_WRITE。

**time-based 路径**：`evaluateTimeBasedTrigger()` 检测用户最后发言到现在的间隔，超过 `gapThresholdMinutes` 触发。`maybeTimeBasedMicrocompact()` 把旧 tool_result content 改写成占位符，保留最近 `keepRecent` 组（下限 1）。会调 `notifyCacheDeletion()` 失效 cache。

**cached-microcompact 路径**（ant-only，`feature('CACHED_MICROCOMPACT')`）：用 API `cache_edits` 块在服务端原地删除 tool_result，**不破坏前缀缓存**。客户端维护 `mod.registerToolResult/registerToolMessage` 注册表。日志事件 `tengu_cached_microcompact`。

常量：`IMAGE_MAX_TOKEN_SIZE = 2000`，token 估算膨胀系数 `4/3`。

### 4. contextCollapse

`feature('CONTEXT_COLLAPSE')` 门控。基于 projection 的折叠应用，无显式 boundary message——折叠是隐式记忆。

### 5. autoCompactIfNeeded

`/src/services/compact/autoCompact.ts`，常量：

| 常量 | 值 | 含义 |
|---|---|---|
| `MAX_OUTPUT_TOKENS_FOR_SUMMARY` | 20_000 | 给摘要产出留的输出 token |
| `AUTOCOMPACT_BUFFER_TOKENS` | 13_000 | 阈值缓冲 |
| `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES` | 3 | 熔断器 |

`getEffectiveContextWindowSize(model)` = `contextWindow - reservedForSummary`
`getAutoCompactThreshold(model)` = `effectiveWindow - AUTOCOMPACT_BUFFER_TOKENS`
（可通过 `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` 环境变量覆盖）

`calculateTokenWarningState()` 返回五维状态：
```
{percentLeft, isAboveWarningThreshold, isAboveErrorThreshold, 
 isAboveAutoCompactThreshold, isAtBlockingLimit}
```

`shouldAutoCompact()` 三条递归护栏：
- `querySource ∈ {compact, session_memory, marble_origami}` 直接返回 false
- `feature('REACTIVE_COMPACT')` 开启时让位
- `feature('CONTEXT_COLLAPSE')` 开启时让位

调度顺序：先查熔断 → `shouldAutoCompact` 判定 → 先试 `trySessionMemoryCompaction()` 捷径 → 失败回落到 `compactConversation()` → 更新 `consecutiveFailures`。

### 6. compactConversation 完整流程

`/src/services/compact/compact.ts`，约 1700 行，主函数横跨几百行。常量：

| 常量 | 值 |
|---|---|
| `POST_COMPACT_TOKEN_BUDGET` | 50_000 |
| `POST_COMPACT_MAX_FILES_TO_RESTORE` | 5 |
| `POST_COMPACT_MAX_TOKENS_PER_FILE` | 5_000 |
| `POST_COMPACT_MAX_TOKENS_PER_SKILL` | 5_000 |
| `POST_COMPACT_SKILLS_TOKEN_BUDGET` | 25_000 |
| `MAX_COMPACT_STREAMING_RETRIES` | 2 |
| `MAX_PTL_RETRIES` | 3 |

**流程**：

```
executePreCompactHooks
↓
setStreamMode('requesting')
↓
preCompactTokenCount = estimate
↓
PTL-retry loop (最多 3 次)：
  streamCompactSummary() 
    ├─ forkedAgent path（默认，cache 共享）
    └─ streaming fallback（重试 2 次）
  if PTL error → truncateHeadForPTLRetry() → 重试
↓
clear readFileState
↓
parallel: createPostCompactFileAttachments + createAsyncAgentAttachmentsIfNeeded
↓
追加 plan/skill/tools/agents/mcp delta attachments
↓
processSessionStartHooks('compact')
↓
createCompactBoundaryMessage with preCompactDiscoveredTools
↓
truePostCompactTokenCount = roughTokenCountEstimationForMessages
↓
logEvent('tengu_compact', {...})
```

**streamCompactSummary 的 forkedAgent 路径**：调 `runForkedAgent()`，关键参数：
- `canUseTool: createCompactCanUseTool()` — 拒绝一切工具
- `querySource: 'compact'` — 防递归
- `maxTurns: 1` — 一轮出结果
- `skipCacheWrite: true` — 不污染缓存
- **不设 `maxOutputTokens`** — 保持 cache key 与主线对齐
- 命中事件：`tengu_compact_cache_sharing_success`
- 降级事件：`tengu_compact_cache_sharing_fallback`

**truncateHeadForPTLRetry 算法**：
1. 解析 `getPromptTooLongTokenGap(error)` 拿到要砍的 token 量
2. 用 `groupMessagesByApiRound` 分组
3. 累加丢弃直到覆盖 gap
4. fallback：解析失败则砍前 20%
5. 下限：保留 ≥1 组、丢 ≥1 组
6. 若砍完首条变 assistant，前置 `PTL_RETRY_MARKER` 伪 user 消息
7. 记 `tengu_compact_ptl_retry`

**buildPostCompactMessages 顺序**：

```
[boundaryMarker, ...summaryMessages, ...messagesToKeep, ...attachments, ...hookResults]
```

**annotateBoundaryWithPreservedSegment**：打 `{headUuid, anchorUuid, tailUuid}` 三锚点，多次压缩时仍能定位。

**createPostCompactFileAttachments 预算逻辑**：
- 选 `readFileState` 中时间戳最新的 5 个文件
- 每文件按 5K token 截断
- 总预算 50K token
- 调 `collectReadToolFilePaths(preservedMessages)` 去重
- 排除 plan 文件和 CLAUDE.md（独立通道恢复）

### 7. sessionMemoryCompact（实验性）

`/src/services/compact/sessionMemoryCompact.ts` 的 `trySessionMemoryCompaction()` 是 0 次 LLM 调用的捷径。

触发条件：`tengu_session_memory` + `tengu_sm_compact` 双 GrowthBook flag。

GrowthBook 配置：
- `minTokens: 10_000`
- `minTextBlockMessages: 5`
- `maxTokens: 40_000`

`calculateMessagesToKeepIndex()` 从 `lastSummarizedMessageId` 反扩，用 `adjustIndexToPreserveAPIInvariants()` 防止切断 tool_use/tool_result 对和共享 message.id 的 thinking 块。

返回 `null` 表示不适用，autoCompact 回落到完整 `compactConversation`。

### 8. apiMicrocompact（服务端）

`/src/services/compact/apiMicrocompact.ts` 的 `getAPIContextManagement()` 在请求载荷里附 `context_management` 字段，由服务端执行 edits：

**clear_thinking_20251015**：保留全部 thinking 或仅最近一轮（取决于 1 小时 cache miss 状态）。

**clear_tool_uses_20250919**（ant-only）：
- `trigger: { type: 'input_tokens', value: 180_000 }`
- 目标：砍回 `180_000 - 40_000 = 140_000`
- 清理白名单（content）：SHELL、GLOB、GREP、FILE_READ、WEB_FETCH、WEB_SEARCH
- 排除（tool_use 本身）：FILE_EDIT、FILE_WRITE、NOTEBOOK_EDIT（写操作不能删）

### 9. reactive-compact 与 blocking-limit

`tryReactiveCompact`（feature 门控）：API 返回 413 / media_size_error 且开关打开时触发，置 `hasAttemptedReactiveCompact = true` 防重试。

`blocking-limit check`（line 637）：扣除 `snipTokensFreed` 后判定 `isAtBlockingLimit`，yield `PROMPT_TOO_LONG_ERROR_MESSAGE`，return `{ reason: 'blocking_limit' }`。

### 10. postCompactCleanup

`/src/services/compact/postCompactCleanup.ts` 的 `runPostCompactCleanup(querySource?)`：
- `resetMicrocompactState()`
- `resetContextCollapse()`（feature gated，主线专属）
- `getUserContext.cache.clear()` + `resetGetMemoryFilesCache('compact')`
- `clearSystemPromptSections()`、`clearClassifierApprovals()`、`clearSpeculativeChecks()`
- `clearBetaTracingState()`、`sweepFileContentCache`、`clearSessionMessagesCache()`
- **故意不调** `resetSentSkillNames()`（避免重复注入 4K token skill listing）

### 设计观察

**渐进放弃**：每一层放弃得更多但触发成本更高，按"放弃信息量 vs 保留缓存"做帕累托排序。

**Prompt cache 是一等公民**：forkedAgent 不设 maxOutputTokens、cached-microcompact 走 cache_edits、partialCompact 默认 'from' 方向、sessionMemory 反扩——都在维护前缀哈希。

**熔断与递归保护**：`MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3`、querySource 排除、`hasAttemptedReactiveCompact` 一次性。

**API 契约执行者**：user-first、tool_use ↔ tool_result 配对、message.id 整体性——`groupMessagesByApiRound`、`adjustIndexToPreserveAPIInvariants`、PTL 伪 marker 都是为此。

**可观测性**：`tengu_compact`、`tengu_compact_ptl_retry`、`tengu_cached_microcompact`、`tengu_compact_cache_sharing_*` 等独立事件维度。

**Feature flag + DCE**：`HISTORY_SNIP`、`CONTEXT_COLLAPSE`、`REACTIVE_COMPACT`、`CACHED_MICROCOMPACT` 等通过 Bun tree-shaking 在不同 build 中差异化呈现。

**压缩 = 新会话**：成功后调 `processSessionStartHooks('compact')` 重新注入 plan/skill/tools delta，从 LLM 视角等同于"带记忆的新会话启动"。

---

## 三、React/Ink 终端应用

### Ink 是什么

React 三段式架构：虚拟 DOM → reconciler → renderer。React 自己只做前两段，最后一段可换：
- `react-dom` → 浏览器 DOM
- `react-native` → 移动原生视图
- **`ink` → 终端**

Ink 把 JSX（`<Box>`、`<Text>`）喂给自定义 reconciler，用 **Yoga 布局引擎**（Facebook React Native 的 Flexbox C++ 实现）算字符网格坐标，diff 后通过 ANSI 转义序列写到 stdout。终端成了一块字符像素屏，Ink 是图形驱动。

### Claude Code 的深度定制

仓库里 `/src/ink/ink.tsx` 是 **fork 并重写的 Ink**，不是 npm 上的包。相关文件：
- `reconciler.ts`、`renderer.ts`、`screen.ts`
- `selection.ts`、`hit-test.ts`、`searchHighlight.ts`
- `termio/csi.ts`、`termio/dec.ts`、`termio/osc.ts`

实现了：终端 I/O 协议、鼠标点击命中测试、文本选区、超链接池、搜索高亮、滚动捕获——已经是**字符画 GUI**。

### 组件化 TUI

```tsx
<Box flexDirection="column">
  <ThinkingIndicator active={isThinking} />
  <ToolUseList tools={inProgressTools} />
  <TokenBudgetBar used={tokens} max={contextWindow} />
  <InputPrompt onSubmit={handleSubmit} />
</Box>
```

仓库里大量 `.tsx`：`tools/WebSearchTool/UI.tsx`、`tools/SkillTool/UI.tsx`、`components/StructuredDiff.tsx`、`components/StatusNotices.tsx`——每个工具、每个 notice、每个权限 prompt 都是独立组件。

### 终端能力 hook 化

`/src/ink/hooks/use-input.ts` 暴露 `useInput(handler)`：
```tsx
useInput((input, key) => {
  if (key.escape) cancel()
})
```

类似还有 `useFocus`、`useTerminalSize`、`useStdout`——把终端底层能力装进 React 状态机。

### 与 Agent Loop 解耦

UI 循环和 Agent Loop 是两条独立异步流。`QueryEngine.submitMessage()` 是 AsyncGenerator，UI 用 `useEffect` 订阅：

```tsx
useEffect(() => {
  (async () => {
    for await (const msg of queryEngine.submitMessage(input)) {
      setMessages(prev => [...prev, msg])
    }
  })()
}, [input])
```

权限询问是会合点：Agent Loop 在需要许可时塞一个待办 → UI 渲染 yes/no → 用户按键 resolve Promise → 答案回到 Agent Loop。

### 总结

**Claude Code = 在终端里运行的 GUI 程序，UI 用 React 声明式描述，Ink 把 React 树翻译成 ANSI 控制序列**。它不是传统 "print → readline → print" 的 CLI，而是全屏接管终端、有焦点管理、有鼠标事件、有字符级增量渲染的交互式 TUI。同一份业务代码（QueryEngine、queryLoop、压缩链路）能同时支撑交互模式、headless 模式和 SDK 嵌入模式。

---

## 四、Client SDK vs Agent SDK vs Claude Code CLI

### 抽象栈

```
┌──────────────────────────────────────────────────────────────────┐
│ Claude Code CLI   ← 终端 TUI + Agent Loop + 工具 + UI 全套         │  用户
├──────────────────────────────────────────────────────────────────┤
│ Agent SDK         ← Agent Loop + 工具 + 上下文管理（无 UI）         │  开发者（产品代码）
├──────────────────────────────────────────────────────────────────┤
│ Client SDK        ← 只有 API 调用封装(messages.create)             │  开发者（底层代码）
├──────────────────────────────────────────────────────────────────┤
│ Anthropic HTTP API ← 原始 REST                                    │  协议
└──────────────────────────────────────────────────────────────────┘
```

### 边界对比表

| 维度 | Client SDK | Agent SDK | Claude Code CLI |
|---|---|---|---|
| API 调用 | 你来 | SDK 内部 | SDK 内部 |
| Agent Loop（tool_use 循环） | **你写** | 内置 | 内置 |
| 内置工具 | 无 | 40+ 个 | 40+ 个 |
| 权限系统 | 无 | 可编程 canUseTool | 交互式 prompt + 配置 |
| 上下文压缩 | 你来 | 内置六级流水线 | 内置 |
| MCP 集成 | 无 | 内置 | 内置 |
| Subagent / forked agent | 无 | 内置 Task 工具 | 内置 |
| UI 层 | 无 | **无** | React/Ink TUI |
| 会话持久化 | 无 | 可选 | 默认开启 |
| 斜杠命令 / plugin / hooks | 无 | 部分（hooks） | 全部 |
| 入口 | `client.messages.create()` | `query({...})` AsyncGenerator | `claude` 命令 |
| 最适合 | 非 agent 应用 | 嵌进自家产品 | 终端交互 |

### 三段代码形态

**Client SDK**：

```typescript
let resp = await client.messages.create({ model, tools, messages });
while (resp.stop_reason === 'tool_use') {
  const results = await Promise.all(
    resp.content
      .filter(b => b.type === 'tool_use')
      .map(async b => ({ tool_use_id: b.id, content: await myTools[b.name](b.input) }))
  );
  messages.push({ role: 'assistant', content: resp.content });
  messages.push({ role: 'user', content: results.map(r => ({ type: 'tool_result', ...r })) });
  resp = await client.messages.create({ model, tools, messages });
}
```

**Agent SDK**：

```typescript
for await (const msg of query({
  prompt: "Fix the bug in auth.ts",
  options: { cwd, allowedTools: ['Read','Edit','Bash'], canUseTool: myGate }
})) {
  console.log(msg);
}
```

**Claude Code CLI**：

```bash
$ claude
> Fix the bug in auth.ts
```

### 选型决策

| 场景 | 推荐 | 理由 |
|---|---|---|
| Interactive development | CLI | 看中间过程、迭代 prompt、人在回路批准 |
| One-off tasks | CLI | `claude -p` 一行命令最快 |
| CI/CD pipelines | SDK | 编程控制退出码、日志、错误处理 |
| Custom applications | SDK | 自家产品 UI 嵌入 agent |
| Production automation | SDK | 稳定性、可观测性、可测试性 |

**先问"是不是 agent 任务"**：不是 → Client SDK；是 → 再问要不要终端交互：要 → CLI，不要 → Agent SDK。

### 一句话

**Client SDK 给你 API；Agent SDK 给你 Agent；Claude Code CLI 给你一个装着 Agent 的终端应用。** 三者核心差异是"谁拥有那个 `while (stop_reason === 'tool_use')` 循环"——你、SDK、还是 CLI。

---

## 五、为什么 npm 包会被称为"源码泄露"

### npm 发布的不是源码

```
src/*.ts ──编译──> dist/*.js ──bundle──> cli.bundle.js ──minify──> cli.min.js
  ↑                ↑                      ↑                         ↑
 你写的           编译产物               单文件打包                   压缩混淆
 易读             仍可读                 模块边界消失                 几乎不可读
```

Claude Code 用 **Bun 打包器**把几百个 `.ts/.tsx` 合成一个 cli.js：
- 所有 `import` 内联展开
- `feature()` 门控里走不到的分支 tree-shaking 掉
- 变量名 minify 成 `a`、`b`、`_0x123`
- TS / JSX 语法擦除
- 注释删除
- 字符串提到常量池

`npm install` 拿到的就是这坨 minified bundle。理论上是文本，但和源码不是一回事。

### Source map 撕开了黑盒

Bundler 生成 `cli.js.map` 用于调试时反向映射回源文件。**关键是 source map 规范的 `sourcesContent` 字段装的是原始源码字符串**——为了在没有本地工作副本时也能调试。

一旦某个商业 npm 包发布时把 source map 也打进去了，源码就跟着走了：

```
npm pack @anthropic-ai/claude-code   # 拿到 tarball
解压 → 找到 cli.js.map
用 webcrack / sourcemap-unpack 解出 sourcesContent
↓
得到几百个原始 /src/*.ts 文件 —— 这就是你我现在翻的这份代码
```

### 为什么会这样发布

通常不是主动开源，是几种情况之一：

1. **构建配置遗漏**：`sourceMap: true` 没显式关
2. **Sentry 集成失配**：上传顺序错或同时进了发布包
3. **调试需要**：CLI 工具 stack trace 对用户体验很重要——`QueryEngine.ts:456` 比 `cli.min.js:1:384572` 有用得多

业界观察 Claude Code 接近第三种：Anthropic 应清楚源码会被还原，选择了"可读 > 保密"的取舍。真正的 secrets（API key、敏感 prompts）不会进 npm 包。

### "可读"≠"可合法用"

`@anthropic-ai/claude-code` 的 license 是 Anthropic 商业许可，不是 MIT/Apache。即使源码可还原：
- 研究、学习、借鉴设计思想 — 通常没问题
- 直接抄代码、fork 出去发布、嵌进自家闭源产品卖 — 违反许可

### 一句话

**npm 分发 JS ≠ 源码可读，source map 才是那座桥**。Claude Code 被称作"源码泄露"指的是随 npm 一起发布的 source map 让源码从产物形态退回到了源码形态。

---

## 六、Streaming 输入模式

### `prompt` 的两种形态

**字符串**（最简形态）：
```typescript
for await (const msg of query({ prompt: "Fix the bug in auth.ts" })) { ... }
```

**AsyncGenerator**（流式输入）：
```typescript
async function* generateMessages() {
  yield { type: "user", message: { role: "user", content: "Analyze this codebase" }};
  await new Promise(r => setTimeout(r, 2000));
  yield { 
    type: "user", 
    message: { 
      role: "user", 
      content: [
        { type: "text", text: "Review this diagram" },
        { type: "image", source: { type: "base64", media_type: "image/png", data: ... }}
      ]
    }
  };
}

for await (const message of query({
  prompt: generateMessages(),
  options: { maxTurns: 10, allowedTools: ["Read", "Grep"] }
})) {
  console.log(message);
}
```

字符串 prompt 是退化的 AsyncGenerator——只 yield 一次就结束。

### 双向流协议

两条流首尾衔接：
```
你的 generator → SDK 消费 → agent loop → SDK 产生输出流 → 你的 for await 消费
```

互相背压：generator yield 不被 await 就停在 yield 那一行；输出流不被 for await 消费就 yielding 暂停。

### 时序图各节点对应源码

| 图中节点 | 对应源码 |
|---|---|
| Initialize with AsyncGenerator | `query()` 内部 `QueryEngine.submitMessage()` |
| Yield Message 1 | `processUserInput` 注入 messages |
| Execute tools | `runTools()` / `runToolUse()` |
| Read files / Write files | 具体 Tool 的 `call()` 执行 |
| Stream partial response | `queryModelWithStreaming` SSE 切片 |
| Complete Message 1 | 一轮 queryLoop 结束（stop_reason: end_turn） |
| Yield Message 2 + Image | generator 二次 yield，注入同一 session |
| Queue Message 3 | commandQueue 排队（详见第七章） |
| Interrupt/Cancel | AbortController.abort |
| Handle interruption | 工具收到 signal、loop 提前跳出 |
| Session stays alive | session ID、文件状态、history 不重启 |
| Persistent file system state | 工具改真实磁盘文件，跨消息累积 |

### 为什么这样设计

字符串 prompt 只能写"问→答→问→答"。流式 prompt 让：

1. **agent 跑的过程中追加输入**：示例的"先扫安全 → 2 秒后附架构图"无需重构上下文
2. **外部事件作输入源**：generator 里可以是 `await fromKafka()`、`await webhook.next()`、`await waitForUserApproval()`——agent 挂在任何事件总线上响应

### `options` 是流的全局约束

`maxTurns: 10` 限制 queryLoop 迭代次数；`allowedTools: ['Read','Grep']` 通过 `getTools(ctx)` 过滤工具池。这两条与流式 prompt 正交：**一份配置 + 一条输入流 + 一条输出流**。

### 一句话

**字符串 prompt 是 AsyncGenerator prompt 的退化情况**。理解 SDK 的输入是流不是值，"long-running agent"、"event-driven agent"、"multi-modal mid-conversation injection" 都自然落到同一抽象。这是 Agent SDK 区别于 Client SDK 最深刻处——**它不是帮你少写循环，是帮你把 agent 接入任何事件源**。

---

## 七、Agent 执行中的用户输入处理

### 核心数据结构：模块级 commandQueue

`/src/utils/messageQueueManager.ts`：

```typescript
const commandQueue: QueuedCommand[] = []
```

**与 React 完全无关的、模块全局的命令队列**。承载所有"想被 agent 处理的命令"——用户 prompt、任务通知、悬空权限请求都进同一队。

优先级：`'now' > 'next' > 'later'`。
- `'next'`：用户输入默认
- `'later'`：后台任务通知
- `'now'`：紧急注入

相同优先级 FIFO。

### 双消费者模型

队列在模块层，因为既要被 React（`useSyncExternalStore` + `subscribeToCommandQueue`）订阅，又要被非 React 代码（`print.ts` 流式循环）读取。每次 mutation 调 `notifySubscribers()` 重建冻结 snapshot。

### 场景 A：用户敲完一段 + 按 Enter

```
键盘 → useInput → 输入框累积 → Enter → handlePromptSubmit → enqueue
                                                                ↓
                                              notifySubscribers
                                                                ↓
                              REPL useEffect 醒来，但 isQueryActive === true
                                                                ↓
                                       UI 渲染 "queued: ..." 预览
                                                                ↓
                                         (当前回合跑完)
                                                                ↓
                                  popAllEditable(inputValue, 0)   line 2167
                                                                ↓
                              所有可编辑命令拼成新 prompt → onQuery()
```

**多条排队会被合并**而不是分别处理三次——同一上下文的连续追加意图，分别处理会做无用功。

### 场景 B：用户按 ESC

```typescript
abortController?.abort('user-cancel')   // line 2147, 2152
```

每个 `onQuery()` 新建 AbortController，signal 传到所有可能阻塞处：
- `queryModelWithStreaming(..., { signal })` — 立即停 SSE
- 每个工具 `call()` 的 ctx 带 abortController — 长任务自杀
- `runTools` 批次循环每次检查 signal

**ESC 不退出会话**——session ID、文件状态、history 都还在。如果队列还有排队命令，默认还会调 `clearCommandQueue()`。

### 场景 C：打字但没回车

`useInput` 捕获键盘事件，输入框 React state 累积。**完全是 UI 本地行为，agent lifeline 感知不到**。

按 UP 键或空输入框 ESC 触发 `popAllEditable()` 反悔机制：
```typescript
const newInput = [...queuedTexts, currentInput].filter(Boolean).join('\n')
```

把队列文本拼到当前输入框前，光标重新计算，图片附件跟回。

不可编辑的命令（系统注入、channel 消息）不会被 popAll 拉走。

### 与 streaming AsyncGenerator 的关系

时序图里 "Queue Message 3" 对应的就是 commandQueue。SDK 模式 generator yield = enqueue，CLI 模式键盘 Enter = enqueue。**不同输入源，同一队列，同一 dequeue 节奏**。

```
键盘事件 → InputBox → handlePromptSubmit → enqueue
                                            ↓
                                  (between turns)
                                            ↓
                            popAllEditable / dequeue → onQuery → query()
```

### 设计要点

**优先级三档**：用户敲键盘永远不会被后台 cron 提醒挤掉。

**队列在模块层、UI 在 React 层**：避免 agent 跑到一半冲掉 React state。

**dequeue 在回合间隙**：模型每轮看到稳定快照，不是流动目标。

**ESC 分级**：打字时撤回；agent 跑时 abort；abort 后清队列。

### 一句话

**用户在 agent 执行时输入新内容，先进入全局命令队列，agent 不会被立刻打断（除非按 ESC）；当前回合跑完后，所有排队的可编辑命令一次性合并成下一轮输入。** 这条队列把"流式输入"和"回合制 agent 循环"两种节奏耦合起来——这是 Claude Code 在终端里表现得像"实时对话"、底层却是"严格回合制 LLM 调用"的关键。

---

## 八、LSP 工具

### 文本工具的局限

LSP 工具出现前，Claude Code 看代码靠**纯文本手段**——Read、Grep、Glob。这套对"找一段代码"够用，对"理解代码"是瞎子。重命名 `User` 为 `Account` 时，正则会顺带改注释、字符串、变量名 `currentUser` 等等。

### LSP 是什么

**Language Server Protocol**，Microsoft 为 VS Code 设计、现成事实标准。把"语言相关智能能力"剥离成独立进程（language server），编辑器和 server 用 JSON-RPC 通信。

| 语言 | Server |
|---|---|
| TypeScript | tsserver |
| Python | pyright / pylsp |
| Go | gopls |
| Rust | rust-analyzer |
| Java | jdtls |

server 内置真正的语义理解：AST、类型推导、跨文件符号表、引用图。VS Code 的 F12 跳转、悬停看类型、自动重命名都靠 LSP。

LSP 协议标准请求：`textDocument/definition`、`textDocument/references`、`textDocument/hover`、`textDocument/documentSymbol`、`workspace/symbol`、`textDocument/implementation`、`textDocument/diagnostic` 等。

### Claude Code 的 LSP Tool

`/src/tools/LSPTool/schemas.ts` 是单工具承载多 operation 的 discriminated union：

```typescript
operation: 'goToDefinition' | 'findReferences' | 'hover' 
         | 'documentSymbol' | 'workspaceSymbol' | ...
```

模型给 `{operation, filePath, line, character}`，`/src/services/lsp/manager.ts` 翻译成 JSON-RPC 发给 language server，回复包成 tool_result。

模型获得的能力：
- **跳转定义**：编译器认证的真定义，不是搜索可能匹配
- **找所有引用**：真正使用这个类型的所有位置（同名不同作用域不出现）
- **悬停查类型**：完整类型信息含泛型展开和推导
- **列文档符号 / 全局符号搜索**
- **找实现**：interface 在哪些 class 里实现
- **调用层级**：函数 A 被谁调用、调用谁

### 杀手锏：edit 后自动喂 diagnostics

Edit 工具写完文件后通过 LSP manager 推 `textDocument/didChange` 给 server，server 立即重做类型检查并返回 diagnostics。**Claude Code 把这些 diagnostics 拼进 Edit 的 tool_result**。

效果：反馈循环从"改完→运行 build→看输出→再改"变成"改完→tool_result 直接说新引入了 3 个 type error"。**反馈延迟从分钟级降到亚秒级**。

### Plugin 化的工程取舍

不内置 server 二进制是因为：

1. **体积**：每个 server 几十到几百 MB
2. **版本耦合**：tsserver 要跟项目 TS 版本对齐才准确
3. **可拓展性**：硬编码支持哪些语言不现实

使用流程：装 "code intelligence plugin" → plugin 提供 server 启动配置 → 用户自己装 server binary。

### 与 Grep / Read 的互补

| 任务 | 用什么 | 为什么 |
|---|---|---|
| 找含某字符串的代码 | Grep | 快、跨语言 |
| 找某符号的精确定义 | LSP goToDefinition | 走类型不被同名干扰 |
| 改函数名所有调用点 | LSP findReferences | 不误改字符串和注释 |
| 看变量推导出来什么类型 | LSP hover | Grep/Read 做不到 |
| 调研第三方库 | Grep / Read | LSP 在 node_modules 里通常没索引 |
| 改完看有没有破坏 | LSP diagnostics（自动） | 比 build 快几个数量级 |

### 与工具系统的关系

LSP Tool 遵循标准 `Tool<Input, Output>` 接口，特殊之处：

1. **外部依赖进程**：language server 是长生命期子进程
2. **默认禁用、按 plugin 启用**：没装 server 的语言对模型不可见
3. **与 Edit 工具隐式耦合**：Edit 写完文件触发 LSP 通知 + diagnostics 拼回

### 一句话

**LSP 工具让 Claude 在改代码这件事上从"按字符串想"升级到"按符号想"，并把 IDE 的实时类型反馈接入到模型的 tool_result 流里**——把 Claude Code 从"高级 grep + sed"推向"真正能写工业代码的协作伙伴"。它让模型能用编译器的眼睛看代码。

---

## 九、Prompt Cache 机制

### 核心定位

**Prompt cache 是 Anthropic API 服务端的能力，不是 Claude Code 本地的**。客户端只是通过请求标记"使唤"它，缓存数据在 Anthropic 数据中心 GPU 显存/内存里，本地一字节都没存。

### 缓存的是什么

不是"字符串 → 回答"，是 **transformer 处理 prompt 前缀时计算的 KV cache**。

每个 transformer 层在每个 token 位置算出 K（key）和 V（value）张量，是处理后续 token 的 attention 输入。Anthropic 服务端把"算到某位置时的整张 KV cache"存下来，下次同样前缀进来直接复用，省掉那部分 forward pass。

寻址 key 是前缀字节哈希，value 是 KV cache 张量。**只能在服务端**——KV cache 是 GPU 私有张量，传不出来也没用。

### 客户端怎么使唤

API 协议的 `cache_control` 标记：

```json
{
  "type": "text",
  "text": "<long system prompt or tool definition>",
  "cache_control": { "type": "ephemeral" }
}
```

服务端看到标记就把"从 prompt 开头到这个 block 结尾"的前缀缓存。这个标记叫 **cache breakpoint**。

`/src/services/api/promptCacheBreakDetection.ts`、`/src/services/api/claude.ts` 决定每次请求把 breakpoint 打哪。每请求最多 4 个，最佳布局：

```
[system prompt]                    ← breakpoint 1（几千字，session 内不变）
[tool definitions]                 ← breakpoint 2（几万字，几乎不变）
[stable conversation history]      ← breakpoint 3（已发生过的对话）
[recent turns / new user message]  ← 不打 breakpoint，每次新算
```

### 命中条件：字节级前缀完全一致

不是语义匹配，是字节级前缀匹配。**任何一个字符差异都 cache miss**。失效场景：

- system prompt 加一行
- 工具定义改一个描述字段
- 早期消息被 microcompact 改了 content
- prompt 里注入时间戳
- 工具列表顺序变化
- JSON 字段顺序变化

这就是为什么压缩链路所有策略都"避免改动前缀字节"——cached-microcompact 走服务端 cache_edits、forked agent 不设 maxOutputTokens、preservedSegment 锚定边界——都是为了不破坏前缀。

### 经济模型

| 类型 | 相对原价 |
|---|---|
| 普通 input token | 1.0x |
| **cache write** | 1.25x |
| **cache read** | 0.1x |
| output token | 5x（与缓存无关） |

第一次走过某前缀多花 25%，后续命中降到 1/10。长会话总开销可降到不开缓存的 10-15%。

Claude Code 这种长 session、工具池庞大、每轮重发整个 history 的应用必须依赖 prompt cache——否则经济上跑不动。

### 监测

`tengu_compact_cache_sharing_success/fallback` 等事件埋点统计命中率。response 里 `cache_creation_input_tokens` / `cache_read_input_tokens` 字段告诉客户端实际写入和读取了多少缓存 token——客户端无法预知，事后看账单。

### TTL

- **默认 5 分钟**（ephemeral）。命中或写入续期。
- **1 小时 extended**（`cache_control.ttl: "1h"`，beta header）。付更高写入费换更长 TTL。
- 容量满 LRU 淘汰，可能比 TTL 更早。

`/src/services/api/promptCacheBreakDetection.ts` 的 break detection 根据 response 实际 cache_read 量估测前缀是否过期。

### Claude Code 的实际布局

| 段 | breakpoint | 内容 |
|---|---|---|
| System prompt | 1 | Claude Code 主提示词、CLAUDE.md、用户配置 |
| Tool definitions | 2 | 40+ 工具的 JSON Schema + 描述 |
| User context | 3 | git status、目录结构、CLAUDE.md hierarchy |
| Conversation + new input | 不打 | 每次新增 |

**每次新一轮 LLM 调用，前面 95%+ 的 token 走 cache_read（0.1x），只有最新一两条走全价**。所以单轮成本不高。

### 为什么压缩纠结 cache

`autoCompactIfNeeded` 排在最后才触发，因为 compact 是**唯一会大幅破坏前缀缓存**的操作。压缩后整段历史替换成摘要，前缀字节全变，下一轮 cache 全失效，全部 cache_write 重写。这次 API 调用费用可能是前面几十次的总和。

`microCompact` 的 cached-microcompact 路径特别——它通过 API 的 `cache_edits` 让服务端在缓存内部原地删除片段，缓存哈希被特殊处理保持不变。"在不解压缩的情况下编辑压缩文件"。

`forked agent` 跑 compact summary 时复用主线 cache 也是同样动机——主线那段长前缀已在服务端缓存，让 forked agent 用同一个 key（不改 system prompt、不改 tools、不改 thinking config），它的请求蹭主线缓存，整段输入费用 ≈ 0。

### 一句话

**Prompt cache 是 Anthropic API 服务端对 transformer KV cache 做的内容寻址缓存；客户端通过 `cache_control` 标记告诉服务端记住前缀，命中条件是前缀字节完全一致。本地不存任何缓存数据，本地只能控制"打不打标记"和"怎么布局让前缀稳定"——但效果上能把长会话输入成本降到 10%**。这也是为什么 Claude Code 工程里那么多奇怪细节都在为同一件事服务：**让前缀字节尽可能不变**。

---

## 附录：关键文件索引

### 核心循环
- `/src/QueryEngine.ts` — 顶层 AsyncGenerator
- `/src/query.ts` — queryLoop 6 阶段
- `/src/query/deps.ts` — 依赖注入
- `/src/query/config.ts` — 每轮快照

### 工具系统
- `/src/Tool.ts` — Tool<Input, Output, P> 接口
- `/src/tools.ts` — getAllBaseTools / assembleToolPool
- `/src/services/tools/toolOrchestration.ts` — runTools / partitionToolCalls
- `/src/services/tools/StreamingToolExecutor.ts` — 边流式接收边执行
- `/src/services/tools/toolExecution.ts` — runToolUse 执行入口

### 权限
- `/src/hooks/useCanUseTool.tsx` — React 路由
- `/src/utils/permissions/permissions.ts` — 7 步 gate

### 压缩
- `/src/services/compact/autoCompact.ts` — 阈值与熔断
- `/src/services/compact/compact.ts` — compactConversation 主流程
- `/src/services/compact/microCompact.ts` — 双路 microcompact
- `/src/services/compact/sessionMemoryCompact.ts` — 0 LLM 捷径
- `/src/services/compact/apiMicrocompact.ts` — 服务端 context_edits
- `/src/services/compact/postCompactCleanup.ts` — 收尾清理
- `/src/services/compact/prompt.ts` — 摘要 prompt 模板
- `/src/services/compact/grouping.ts` — API round 分组

### UI
- `/src/ink/ink.tsx` — fork 重写的 Ink 主类
- `/src/ink/components/App.tsx` — 应用根组件
- `/src/ink/hooks/use-input.ts` — 键盘输入 hook
- `/src/screens/REPL.tsx` — 主交互界面

### 输入与队列
- `/src/utils/messageQueueManager.ts` — 全局命令队列
- `/src/utils/processUserInput/processUserInput.ts` — 输入分发
- `/src/utils/handlePromptSubmit.ts` — Enter 提交

### LSP
- `/src/tools/LSPTool/LSPTool.ts` — 工具实现
- `/src/tools/LSPTool/schemas.ts` — operation 联合类型
- `/src/services/lsp/manager.ts` — server 进程管理

### API
- `/src/services/api/claude.ts` — queryModelWithStreaming
- `/src/services/api/promptCacheBreakDetection.ts` — 缓存命中检测

---

*本文档总结了 Claude Code 源码分析的对话过程，按主题重新组织。每一节都以源码片段为依据，结合架构层面的设计观察。*
