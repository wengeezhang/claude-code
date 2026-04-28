# Claude Code 源码分析

> 基于泄露的 Claude Code CLI 源码镜像（2026 年 3 月通过 npm sourcemap 泄露）。核心代码约 6900 行，整体 50+ 子系统。

---

## 一、整体架构

### 1. 入口与启动（[src/main.tsx:1-150](src/main.tsx#L1-L150)）

[main.tsx](src/main.tsx)（4683 行）是 CLI 单文件入口，用 Commander.js 解析命令，React/Ink 渲染。启动顺序被刻意优化：

- 第 9–20 行：`profileCheckpoint` → `startMdmRawRead` → `startKeychainPrefetch` 必须在其他 import 之前执行，让 macOS `plutil` / keychain 子进程与后续 ~135ms 的模块求值并行。
- `feature('FLAG')`（Bun 的构建期常量）配合 `require()` 实现**死代码消除**：KAIROS、COORDINATOR_MODE、HISTORY_SNIP、PROACTIVE 等模式在构建时被裁掉。
- 真正的交互式入口在 [replLauncher.tsx](src/replLauncher.tsx)；headless/SDK 路径在 [entrypoints/sdk](src/entrypoints/sdk)。

### 2. 核心对话循环：QueryEngine + query.ts

这是整个 agent 的心脏。

**[QueryEngine.ts](src/QueryEngine.ts)**（1295 行）—— 面向 SDK 的有状态会话类。一个会话一个 `QueryEngine`，每次 `submitMessage()` 是一轮。它负责：

- 装配系统提示：[QueryEngine.ts:288-308](src/QueryEngine.ts#L288-L308) 调用 `fetchSystemPromptParts()` 拿到 `defaultSystemPrompt / userContext / systemContext`，再按 coordinator 模式 merge。
- 包装 `canUseTool` 追踪 permission denials（[QueryEngine.ts:243-271](src/QueryEngine.ts#L243-L271)）。
- 跨轮维护 `mutableMessages / readFileState / totalUsage / discoveredSkillNames`。

**[query.ts](src/query.ts)**（1729 行）—— 真正的 agent loop 生成器。处理自动 compact（[services/compact/autoCompact.ts](src/services/compact/autoCompact.ts)）、reactive compact、context collapse、snip compaction、工具结果预算、fallback 模型重试。消息流是 `AsyncGenerator<SDKMessage>`，REPL 和 SDK 都消费这同一个流。

**[Tool.ts](src/Tool.ts)**（792 行）—— 工具协议。核心是 `ToolUseContext`（[Tool.ts:158-300](src/Tool.ts#L158-L300)），几乎每个子系统都挂在上面：permission context、MCP clients、agent IDs、file history、通知、进度回调、abort signal、content replacement budget……`Tool<Input, Output, Progress>` 是所有工具的通用签名（`call / description / inputSchema / isConcurrencySafe / isReadOnly / interruptBehavior`）。

### 3. 工具层（40+ 工具，全部在 [src/tools/](src/tools/)）

[tools.ts](src/tools.ts) 是注册中枢。典型特征：

- **基础**：Bash、FileRead/Write/Edit、Glob、Grep、WebFetch、WebSearch、NotebookEdit、TodoWrite。
- **Agent 编排**：[AgentTool](src/tools/AgentTool/AgentTool.tsx)（1397 行）—— 启动 subagent，支持本地/远程、worktree 隔离、auto-background、fork 继承 parent 的 prompt cache（[forkSubagent.ts](src/tools/AgentTool/forkSubagent.ts)）。
- **Skill**：[SkillTool](src/tools/SkillTool)，动态发现 + 按需加载。
- **ToolSearch**：把罕用工具标为 "deferred"，只在需要时通过关键字搜索加载 schema——控制 token 预算。
- **条件工具**（只在特定 feature 下注册）：SleepTool（PROACTIVE/KAIROS）、Cron*（AGENT_TRIGGERS）、RemoteTriggerTool、MonitorTool、PushNotificationTool。
- **BashTool** ([BashTool.tsx:1-60](src/tools/BashTool/BashTool.tsx#L1-L60)) 特别复杂：AST 解析命令、wildcard 权限匹配、sandbox 判定、后台任务、sed 编辑解析、image 输出、git 操作追踪。

### 4. 周边系统

- **[services/mcp/](src/services/mcp/)** —— MCP 客户端、官方 registry、Claude.ai MCP 配置同步、企业策略过滤、VSCode SDK MCP。
- **[hooks/](src/hooks/)** —— 70+ React 钩子，驱动 Ink UI（补全、历史、通知、IDE 集成、swarm 协调、远程会话）。
- **[ink/](src/ink/)** —— 自定义 React 终端 renderer（reconciler、布局、bidi、ANSI 处理、终端查询），替代 stock ink 以支持 FPS tracker、selection、hit-test。
- **[services/autoDream/](src/services/autoDream/)** —— 后台整理 `MEMORY.md` 的子 agent（config + consolidationLock + consolidationPrompt），对应 "Dream" 系统。
- **[coordinator/coordinatorMode.ts](src/coordinator/coordinatorMode.ts)** —— Swarm 多 agent 编排（构建时可裁掉）。
- **[buddy/](src/buddy/)** —— Tamagotchi 彩蛋（Mulberry32 PRNG，18 种物种）。
- **[assistant/](src/assistant)** —— KAIROS "always-on" 模式；[gate.ts](src/assistant/gate.ts) 控制准入。
- **[commands/](src/commands/)** —— slash 命令定义（/init、/review、/compact、/fast 等），[commands.ts](src/commands.ts) 做聚合和远程模式过滤。
- **[entrypoints/sdk](src/entrypoints/sdk)** —— 把 QueryEngine 暴露成 Claude Agent SDK。

### 5. 架构亮点

1. **构建期 feature flag + 条件 require** = 同一份源码，不同形态产物（内部 ant 版、公共版、KAIROS、coordinator），没有运行时开销。
2. **启动期并行 I/O** — keychain、MDM、GrowthBook、AWS/GCP 凭证、MCP 注册表全部 prefetch。
3. **Prompt cache 一等公民** — `renderedSystemPrompt` 被冻结并在 fork agent 间共享，避免 GrowthBook 冷热切换导致 cache miss（[Tool.ts:293-299](src/Tool.ts#L293-L299)）。
4. **Tool result budget** — `contentReplacementState` 跨会话追踪工具输出大小，超标自动替换为 tombstone。
5. **Message pipeline 分层** — user input → processUserInput → QueryEngine → query loop → Tool.call → UI progress → transcript 持久化，全程 generator。

---

## 二、从 `claude` 命令行到输出结果的完整流程

下面按**时间线**串起来讲，每个阶段给出关键文件与行号。

### 阶段 0：Bun/Node 进入 [main.tsx](src/main.tsx)（副作用优先）

CLI 打包进单个 JS 文件，Node/Bun 开始执行 [main.tsx:1-20](src/main.tsx#L1-L20) 的三条 **top-level side effect**（顺序极重要，都在 import 完成之前触发）：

1. `profileCheckpoint('main_tsx_entry')` — 启动打点。
2. `startMdmRawRead()` — fork `plutil` / `reg query` 子进程读 MDM 配置。
3. `startKeychainPrefetch()` — **并行**读 macOS Keychain 的 OAuth token 和 legacy API key。

用意：让这些 I/O 和后续 ~135 ms 的 import 求值重叠。剩下的 import 进来时，子进程结果已就绪。

### 阶段 1：Commander 解析命令行

[main.tsx:902](src/main.tsx#L902) 创建 `new CommanderCommand()`，注册几十个子命令（`mcp`、`auth`、`plugin`、`doctor`、`update`、`install`、`assistant`、`agents`、`task` …）。不带子命令时走**默认 action**——也就是 REPL。

默认 action 里依次：
- 跑 `runMigrations()` ([main.tsx:326](src/main.tsx#L326)) — 版本迁移，例如把 `sonnet-4.5` 迁到 `sonnet-4.6`。
- `init()` ([entrypoints/init.ts](src/entrypoints/init.ts)) — 读取分层 settings、MDM、plugin、skill、apply env、初始化 analytics。
- 并行 prefetch：`fetchBootstrapData`、`prefetchOfficialMcpUrls`、`prefetchPassesEligibility`、`prefetchAwsCredentialsAndBedRockInfoIfSafe`、`loadPolicyLimits`、`initializeGrowthBook`。
- `showSetupScreens()` — 首次登录、trust dialog、onboarding（[interactiveHelpers.tsx](src/interactiveHelpers.tsx)）。
- 解析 `--continue` / `--resume` / `--print` / `connect` / `ssh` / `assistant` 分支。普通情况走 `launchRepl(...)`（[main.tsx:3134](src/main.tsx#L3134)）。

### 阶段 2：拉起 React/Ink UI —— `launchRepl`

[replLauncher.tsx:12-22](src/replLauncher.tsx#L12-L22) 是个极小的薄包装，**懒加载** `App` 和 `REPL`（避免首屏引入 ink 模块）：

```ts
const { App }  = await import('./components/App.js')
const { REPL } = await import('./screens/REPL.js')
await renderAndRun(root, <App {...appProps}><REPL {...replProps}/></App>)
```

- `App` ([components/App.tsx](src/components/App.tsx)) —— 55 行，是 store provider + FPS tracker + 全局 context wrapper。
- `REPL` ([screens/REPL.tsx](src/screens/REPL.tsx)) —— **5005 行**，整个终端 UI 与对话生命周期的指挥中心。
- `renderAndRun` 使用 [src/ink/](src/ink/) 里自研的 React → 终端 reconciler（不是 stock ink）。

REPL 挂载完成后显示 `>` 提示符，光标等待输入；同时后台还在：
- 连 MCP server（`useManageMCPConnections`）。
- 订阅 FS watchers 监听 settings / skills 变更。
- 初始化 LSP manager（[services/lsp/manager.ts](src/services/lsp/manager.ts)）。
- Kairos/proactive 后台 agent（若启用）。

### 阶段 3：用户按下回车 —— `onSubmit`

用户输入 "帮我改个 bug"，回车后触发 [REPL.tsx:3142](src/screens/REPL.tsx#L3142) 的 `onSubmit`：

1. **滚动重置** `repinScroll()`，恢复 proactive loop。
2. **斜杠命令快路径** ([REPL.tsx:3161-3282](src/screens/REPL.tsx#L3161-L3282))：若 `input` 以 `/` 开头、命中 `immediate: true` 或来自 keybinding（如 `/compact`、`/clear`、`/help`），直接 `matchingCommand.load()` + 执行 local-jsx，**绕过 agent**，return。
3. **idle-return 检查**：空闲 > 75 min 且 token > 100k，弹 "要不要新开对话？" 对话框。
4. **加入 history**（上下方向键回放）。
5. **入队 or 立即执行**：若 Claude 正忙 → 插入 `messageQueueManager`（[utils/messageQueueManager.ts](src/utils/messageQueueManager.ts)），待当前 turn 完成再 drain；否则走 `handlePromptSubmit` → `executeUserInput`。

### 阶段 4：消息规范化 —— `processUserInput`

[utils/processUserInput/processUserInput.ts:85](src/utils/processUserInput/processUserInput.ts#L85) 负责把原始字符串变成 agent 能吃的 `Message[]`：

- 展开 `[Pasted text #N]` 占位符成真实文本。
- 解析 `@file` mention、IDE selection、粘贴的图片。
- 若是 `!cmd`（bash mode）→ 就地 `exec` 并把 stdout 包进 `<local-command-stdout>`。
- 若是 `/slashcmd`（非 immediate）→ 查命令 registry（[commands.ts](src/commands.ts)），按类型走：
  - `local` —— JS 函数直接跑。
  - `prompt` —— 展开成 prompt 附到 user message 前。
  - `local-jsx` —— 挂 React 组件。
- 组装 attachments（CLAUDE.md、nested memory、queued notifications）。
- 触发 `UserPromptSubmit` hooks（[utils/sessionStart.ts](src/utils/sessionStart.ts) 家族）——hook 可以 block 提交、改写 prompt、注入 context。
- 返回 `{ messages, shouldQuery, additionalAllowedTools, ... }`。

`executeUserInput` 然后拿这个结果调用 `onQuery`。

### 阶段 5：组装上下文 —— `onQueryImpl`

[REPL.tsx:2661-2803](src/screens/REPL.tsx#L2661-L2803) 在真正发请求前做六件事：

1. `diagnosticTracker.handleQueryStart` + 关闭 IDE 里之前打开的 diff。
2. `generateSessionTitle` — 首轮用 Haiku 起个会话标题（[REPL.tsx:2684-2699](src/screens/REPL.tsx#L2684-L2699)）。
3. 应用 skill 作用域的 `allowedTools`。
4. `getToolUseContext(...)` — 构建整个 `ToolUseContext`（[Tool.ts:158-300](src/Tool.ts#L158-L300)）：tools、commands、MCP clients、agent definitions、permission context、file state cache、abort controller……
5. **并行**拉取 ([REPL.tsx:2768-2772](src/screens/REPL.tsx#L2768-L2772))：
   - `checkAndDisableBypassPermissionsIfNeeded`
   - `getSystemPrompt(tools, model, cwd, mcpClients)` — 组装包含工具清单、CLAUDE.md、平台信息、当前日期的系统提示。
   - `getUserContext()` / `getSystemContext()` — 用户元数据和环境快照。
6. `buildEffectiveSystemPrompt()` 叠加 `customSystemPrompt` / agent definition，**冻结** `toolUseContext.renderedSystemPrompt`（[Tool.ts:293-299](src/Tool.ts#L293-L299)）—— 用于子 agent 共享同一 prompt cache。

然后：

```ts
for await (const event of query({
  messages, systemPrompt, userContext, systemContext,
  canUseTool, toolUseContext, querySource: 'repl_main_thread'
})) {
  onQueryEvent(event)
}
```

### 阶段 6：Agent 核心循环 —— [query.ts:219](src/query.ts#L219)

`query()` 是 `AsyncGenerator`，里面套 `queryLoop`（[query.ts:241](src/query.ts#L241)），循环体每一"轮"的步骤：

#### 6.1 每轮预处理（[query.ts:307-549](src/query.ts#L307-L549)）

- `startRelevantMemoryPrefetch` / `startSkillDiscoveryPrefetch` —— 用 `using` 语法与模型流并发启动。
- `yield { type: 'stream_request_start' }` —— UI 把 spinner 拉起来。
- `applyToolResultBudget` —— 超额的旧 tool result 换成 tombstone（token budget）。
- HISTORY_SNIP 裁旧消息、microcompact 做分块压缩、CONTEXT_COLLAPSE 折叠历史、autocompact 若超阈值整段压缩成摘要（[query.ts:400-543](src/query.ts#L400-L543)）。
- 选择模型：`getRuntimeMainLoopModel({ permissionMode, mainLoopModel, exceeds200kTokens })`。

#### 6.2 调用 Claude API（[query.ts:659-863](src/query.ts#L659-L863)）

```ts
for await (const message of deps.callModel({
  messages: prependUserContext(messagesForQuery, userContext),
  systemPrompt: fullSystemPrompt,
  thinkingConfig,
  tools,
  signal,
  options: { model, toolChoice, fallbackModel, mcpTools, ... }
}))
```

`deps.callModel` = [queryModelWithStreaming](src/services/api/claude.ts#L752) → `queryModel` → `@anthropic-ai/sdk` 的 `/v1/messages?stream=true`。返回的是 `AssistantMessage` / `StreamEvent` / `SystemAPIErrorMessage` 混合流。收到的每条 `assistant` 消息：

- `yield` 给上层 UI 即时显示（流式渲染）。
- 若包含 `tool_use` block → 存进 `toolUseBlocks`，`needsFollowUp = true`。
- 若 `StreamingToolExecutor` 开启，tool_use 一出现就开始**推测执行**（与模型继续输出并行）——[query.ts:841-844](src/query.ts#L841-L844)。
- `FallbackTriggeredError` → 切到 fallback 模型重试整轮（[query.ts:894-951](src/query.ts#L894-L951)）。

#### 6.3 执行工具 —— [services/tools/toolOrchestration.ts:19](src/services/tools/toolOrchestration.ts#L19) 的 `runTools`

```ts
const toolUpdates = streamingToolExecutor
  ? streamingToolExecutor.getRemainingResults()
  : runTools(toolUseBlocks, assistantMessages, canUseTool, toolUseContext)
```

`runTools` 做的事：
- `partitionToolCalls` —— 连续的 **concurrency-safe**（= read-only）工具合并成一 batch 并发跑；写操作单独串行（[toolOrchestration.ts:91-115](src/services/tools/toolOrchestration.ts#L91-L115)）。
- 每个 tool 进入 [toolExecution.ts:337 `runToolUse`](src/services/tools/toolExecution.ts#L337)，它：
  1. `findToolByName` —— 在 available + legacy alias 里找。
  2. `tool.inputSchema.safeParse(input)` —— **Zod 校验**（模型经常传错字段），失败返回 `InputValidationError` 附 schema hint。
  3. `tool.validateInput?()` —— 工具自定义校验。
  4. `checkPermissionsAndCallTool` ([toolExecution.ts:599](src/services/tools/toolExecution.ts#L599))：
     - BashTool 提前启动 classifier speculation。
     - 跑 **PreToolUse hooks**。
     - `canUseTool()` ([hooks/useCanUseTool.tsx](src/hooks/useCanUseTool.tsx))：按 permission mode (`default` / `plan` / `acceptEdits` / `bypassPermissions`) 检查 `alwaysAllow/Deny/Ask` 规则，必要时弹确认对话框。
     - 通过后 `await tool.call(callInput, ctx, canUseTool, parentMessage, onProgress)` ([toolExecution.ts:1207](src/services/tools/toolExecution.ts#L1207))。
     - 跑 **PostToolUse hooks**。
  5. 包装成 `tool_result` block 的 `UserMessage`，`yield` 出去。

#### 6.4 把工具结果拼回去，进入下一轮

`toolResults` 被 `normalizeMessagesForAPI` 过滤后 push 到 `messages`；循环回 6.1，带着新的历史再发一次 API 请求——直到模型返回**没有 tool_use 的 assistant 消息**（`needsFollowUp = false`）。

#### 6.5 终止检查（[query.ts:1062-1376](src/query.ts#L1062-L1376)）

- `executePostSamplingHooks` —— 采样后 hook。
- **prompt-too-long 恢复**：collapse-drain → reactive compact → 真失败才 surface（[query.ts:1100-1183](src/query.ts#L1100-L1183)）。
- **max_output_tokens 恢复**：先升到 64k 重试，再发 "继续" 用户消息多轮续写（[query.ts:1188-1256](src/query.ts#L1188-L1256)）。
- `handleStopHooks` —— Stop hook 可以 block 终止、注入错误让模型再说一轮（[query.ts:1267-1306](src/query.ts#L1267-L1306)）。
- TOKEN_BUDGET 超限时决定 `continue` / 结束（[query.ts:1308-1376](src/query.ts#L1308-L1376)）。

模型产出最终纯文本响应且所有 hook 放行 → generator 返回 `{ reason: 'completed' }`。

### 阶段 7：UI 消费流 —— `onQueryEvent`

[REPL.tsx:2584](src/screens/REPL.tsx#L2584) 的 `onQueryEvent` 把 generator 吐出来的 event 转成 React state：

- `stream_request_start` → spinner on。
- `assistant` message 增量 → 追加到 `messages[]`，[components/Messages.tsx](src/components/Messages.tsx) 渲染。
- `tool_use` → [components/ToolUse.tsx](src/components/ToolUse.tsx) 显示工具名/参数预览。
- `tool_result` → 可能用专属 renderer（`BashToolResultMessage`、`FileEditToolDiff` 等）。
- `tombstone` → 移除指定 message（fallback / budget 失效）。

每条都写入 transcript（`recordTranscript`，[utils/sessionStorage.ts](src/utils/sessionStorage.ts)）——用于 `--continue`、`/resume`、teleport。

### 阶段 8：回合结束

[REPL.tsx:2810-2853](src/screens/REPL.tsx#L2810-L2853)：

- `fireCompanionObserver` —— BUDDY 彩蛋更新宠物反应。
- Ant-only：统计 TTFT / OTPS / hook / tool 耗时并写入 `ApiMetricsMessage`。
- `resetLoadingState()` —— spinner off，abort controller 清空。
- `onTurnComplete(messages)` —— 触发 `Stop` hook、`flushSessionStorage`、通知 mobile 端。
- 从 queue 里 drain 下一条用户消息（若上一轮被打断又排队了新输入）。
- 光标回到 prompt —— **一个 turn 结束**。

---

## 三、关键对象的生命周期

| 对象 | 创建点 | 作用域 | 关键用途 |
|---|---|---|---|
| `QueryEngine` | SDK/headless：一个会话一个 | 会话级 | 跨 turn 存消息、usage、denials |
| `ToolUseContext` | `getToolUseContext` 每轮重建 | 一轮 | 工具运行所需一切——权限、MCP、abort、file cache |
| `AbortController` | 每次提交一个 | 一轮 | Ctrl+C / 新消息打断 |
| `FileStateCache` | REPL 挂载时一个 | 会话级 | 校验 Edit/Write 前后文件未被外部改 |
| `renderedSystemPrompt` | `onQueryImpl` 冻结 | 一轮 + fork 子 agent | 保 prompt cache |
| `mutableMessages` | `QueryEngine` / REPL `messages` state | 会话级 | 构造 API 请求 + 渲染 |

---

## 四、时序图

```
 user types "fix bug" ↵
        │
        ▼
 REPL.onSubmit ───────────────────────── immediate slash? ──► run local cmd, done
        │
        ▼  (enqueue if busy)
 executeUserInput
        │
        ▼
 processUserInput ── hooks.UserPromptSubmit ─► Message[]
        │
        ▼
 onQueryImpl: assemble systemPrompt/userContext/ToolUseContext
        │
        ▼
 query() async generator ──────────────────┐
   └─ queryLoop (while true):              │ yields events
        │                                  ▼
        ├─ autocompact / snip / collapse   UI streams messages
        ├─ deps.callModel (Claude /v1/messages?stream=true)
        │   └─ for each streamed block ── yield
        ├─ runTools
        │   └─ runToolUse
        │       ├─ Zod validate
        │       ├─ PreToolUse hooks
        │       ├─ canUseTool (permission)
        │       ├─ tool.call(...)   ◄── actual side effect
        │       └─ PostToolUse hooks
        ├─ append tool_results to history
        └─ no tool_use in last assistant msg?  ─► break
        │
        ▼
 handleStopHooks → token budget check → return
        │
        ▼
 onQueryImpl tail: metrics, resetLoadingState, onTurnComplete
        │
        ▼
 prompt idle, transcript flushed, ready for next turn
```

---

## 五、上下文压缩机制详解

Claude Code 并不是"写满了就截断"那么简单——它把压缩拆成了**六个独立的层**，从轻到重分级处理 context 膨胀。每一层都是 `query()` 循环在每轮开头依次跑的（见 [query.ts:369-543](src/query.ts#L369-L543)）。

### 1. 阈值与预算（[autoCompact.ts:28-144](src/services/compact/autoCompact.ts#L28-L144)）

所有判断的锚点：

| 常量 | 值 | 含义 |
|---|---|---|
| `MAX_OUTPUT_TOKENS_FOR_SUMMARY` | 20,000 | 为压缩的输出保留的 token（p99.99 ≈ 17,387） |
| `AUTOCOMPACT_BUFFER_TOKENS` | 13,000 | effective window 再留 13k 给模型回答 |
| `WARNING_THRESHOLD_BUFFER_TOKENS` | 20,000 | 离 autocompact 还有 20k 时 UI 变黄 |
| `MANUAL_COMPACT_BUFFER_TOKENS` | 3,000 | 硬阻塞线（手动 `/compact` 的底线） |
| `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES` | 3 | 熔断器：连续失败 3 次就放弃 |

`effectiveWindow = modelContextWindow - 20000`（给 summary 输出预留），`autocompactThreshold = effectiveWindow - 13000`。200k 的 Sonnet → ≈167k 触发 autocompact。

### 2. 六层压缩栈（按 query.ts 调用顺序）

```
每轮循环开头:
  ① applyToolResultBudget   ── 工具结果预算（字节级替换）
  ② HISTORY_SNIP            ── 按字节裁旧消息
  ③ microcompact            ── 清旧 tool_result 内容
  ④ CONTEXT_COLLAPSE        ── 折叠 N 轮对话为摘要
  ⑤ autocompact             ── 整段历史 → 一段 summary
       └ trySessionMemoryCompaction (实验性优先)
  ⑥ reactiveCompact         ── 兜底：API 返回 413 才触发
```

#### ① Tool result budget —— [toolResultStorage.ts:924](src/utils/toolResultStorage.ts#L924) `applyToolResultBudget`

**最轻**，每轮都跑。作用：**单条 user message 的聚合 tool_result 字节数超限**时，把部分 tool_result 的 `content` 替换成 `[大结果已持久化到 $path，用 Read 读取]` 指针。

- 决策被 `ContentReplacementState` **冻结**：某条 tool_use_id 一旦被替换，以后每次重放都用同一字符串（[toolResultStorage.ts:402](src/utils/toolResultStorage.ts#L402)）——保 prompt cache 一致。
- `Read` 工具的结果标记 `maxResultSizeChars: Infinity`，永不被替换（模型自己有 `maxTokens` 限制）。
- 被替换的原文落盘到 `.claude/tool-results/`，模型可以用 `Read` 随时拿回。

#### ② HISTORY_SNIP —— 按字节裁（feature-gated，ant-only）

在 [query.ts:400-410](src/query.ts#L400-L410) 调用。纯机械裁最老的若干条消息，返回 `snipTokensFreed`——这个数字会被 autocompact 当作已释放的 token 纳入判断（否则 autocompact 还按旧 usage 判，会在 snip 刚好够用时多做一次无谓 compact）。

#### ③ microcompact —— [microCompact.ts:253](src/services/compact/microCompact.ts#L253) `microcompactMessages`

**增量清旧 tool_result 的内容，不动消息结构。** 两个触发条件二选一：

**a. Time-based**（[microCompact.ts:446-530](src/services/compact/microCompact.ts#L446-L530)）

```ts
gapMinutes = (now - lastAssistantMessageTimestamp) / 60_000
if (gapMinutes >= config.gapThresholdMinutes) {
  // 只保留最近 N 条 tool_result，其余的 content 替换为
  // '[Old tool result content cleared]'
}
```

用意：**空闲太久 → 服务器 prompt cache 必然已过期**，反正要整体重算，那就干脆先把老的 tool 结果清掉、减小重算体量。只清白名单里的 7 种工具结果：`FileRead / Bash / Grep / Glob / WebSearch / WebFetch / FileEdit / FileWrite`（[microCompact.ts:41-50](src/services/compact/microCompact.ts#L41-L50)）。

**b. Cached microcompact**（CACHED_MICROCOMPACT gate）

更高级：**不改本地消息**，而是往 API 请求里加一个 `cache_edits` block，告诉服务器"这几个 tool_use_id 从你的缓存前缀里删掉"——本地仍有完整历史，但服务器缓存和实际发送的内容已瘦身。

关键逻辑在 [microCompact.ts:305-399](src/services/compact/microCompact.ts#L305-L399)：
- `registerToolResult` 按 user message 把 tool_use_id 入账。
- 超阈值时 `createCacheEditsBlock`，缓存到 `pendingCacheEdits`。
- API 层插入 `cache_edits`；响应里带回 `cache_deleted_input_tokens`，算出真实省下的 token 后 **延迟 yield 一条 boundary 系统消息**（[query.ts:870-891](src/query.ts#L870-L891)）。

#### ④ CONTEXT_COLLAPSE —— 分段折叠（feature-gated）

在 [query.ts:440-447](src/query.ts#L440-L447) 调用，比 autocompact 更细粒度：把 **中间的若干轮对话** 折叠成一段摘要，保留头尾，而不是像 autocompact 那样整段压成一个 summary。

重点：
- 结果不写入 REPL 消息数组，而是存在 collapse store 里，每次 `projectView()` **读时投影**。
- 这样跨 turn 持久化就等于重放 commit log，UI 仍能看到完整滚屏，但 API 只看到折叠视图。
- 跑在 autocompact 之前——**如果 collapse 就能把上下文压到阈值下，autocompact 自然 no-op**，保留更多原始细节。

#### ⑤ autocompact —— 核心的"大压缩"

入口 [autoCompact.ts:241 `autoCompactIfNeeded`](src/services/compact/autoCompact.ts#L241)。流程：

1. `shouldAutoCompact()` 判断 `tokenCount - snipTokensFreed >= autocompactThreshold`。
2. **先试 session memory 压缩** ([autoCompact.ts:288-310](src/services/compact/autoCompact.ts#L288-L310))——把早期消息拿去跑一个 ctx-agent 提炼成结构化的 session memory；实验性路径，成功就避开 full compact。
3. 否则走 [compact.ts:387 `compactConversation`](src/services/compact/compact.ts#L387)：
   a. 触发 `PreCompact` hooks（可注入 custom instructions）。
   b. 用 [compact/prompt.ts `BASE_COMPACT_PROMPT`](src/services/compact/prompt.ts#L61) 作为 user message，fork 一个 subagent 让**模型自己给对话写摘要**，要求 9 段结构化输出（Primary Request / Technical Concepts / Files / Errors / Problem Solving / All User Messages / Pending Tasks / Current Work / Next Step）。
   c. 摘要 request 本身若 413，按 API-round 分组裁掉最老的组重试（最多 `MAX_PTL_RETRIES`，[compact.ts:462-491](src/services/compact/compact.ts#L462-L491)）。
   d. 成功后**清空 `readFileState` 缓存**，并用 `createPostCompactFileAttachments` 重新 attach 最多 N 个最近访问的文件——保证压缩后模型还能 "看到" 当前的关键文件状态。
   e. 组装 `CompactionResult`（[compact.ts:330](src/services/compact/compact.ts#L330) `buildPostCompactMessages`）：
      ```
      [boundaryMarker, ...summaryMessages, ...messagesToKeep, ...attachments, ...hookResults]
      ```
   f. 触发 `SessionStart` hook（type=`'compact'`）。
4. 熔断器：连续 3 次失败后这个会话彻底不再尝试 autocompact（[autoCompact.ts:67-70](src/services/compact/autoCompact.ts#L67-L70) + [:260-265](src/services/compact/autoCompact.ts#L260-L265)）——2026-03 的一次事故里有 1,279 个会话连续失败 50+ 次，每天浪费 ~25 万次 API 调用。

Prompt cache 优化：`tengu_compact_cache_prefix` gate 让压缩 fork 共享主会话的 prompt prefix，从 98% cache miss 降到大幅命中（[compact.ts:431-438](src/services/compact/compact.ts#L431-L438)）。

#### ⑥ reactiveCompact —— 413 兜底（ant-only）

不在预处理里跑，而是**等 API 真的返回 `prompt_too_long` 再触发**，位置在 [query.ts:1119-1175](src/query.ts#L1119-L1175)。

- 流程里先 `withhold` 住这条错误消息不 yield 给 UI；
- 调 `tryReactiveCompact()`，成功就用 post-compact messages 替换历史、`continue` 重试当前 turn；
- 失败才 yield 出原错误。

还有 `max_output_tokens` 恢复路径（[query.ts:1188-1256](src/query.ts#L1188-L1256)）：
- 第一次遇到 → 把 `maxOutputTokensOverride` 从 8k 升到 64k 重试**同一请求**。
- 再失败 → 注入 "Output token limit hit. Resume directly" 的 user meta message，让模型多轮接龙续写（最多 `MAX_OUTPUT_TOKENS_RECOVERY_LIMIT` 次）。

### 3. 压缩结果的"边界消息"

每次压缩（auto / manual / snip / microcompact）都生成一条 `SystemCompactBoundaryMessage`，[compact.ts:598](src/services/compact/compact.ts#L598) `createCompactBoundaryMessage`。它：

- 作为消息数组里**真实存在**的一条记录，界定"这之前已压缩 / 之后是新对话"。
- `compactMetadata` 带 `trigger: 'auto' | 'manual'`、`preCompactTokenCount`、`preservedSegment` (保留原消息 UUID 链接用于 resume)。
- REPL 用 `getMessagesAfterCompactBoundary()`（[query.ts:365](src/query.ts#L365)）每次 API 请求只发边界之后的消息。
- `/resume` 时根据 `preservedSegment` 的 `headUuid/anchorUuid/tailUuid` 重建消息拓扑（[compact.ts:349](src/services/compact/compact.ts#L349) `annotateBoundaryWithPreservedSegment`）。

### 4. 一张决策图

```
每轮 query loop 开头:
  messages[]
     │
     ▼
  ① applyToolResultBudget      ←─ 单条 user message 字节超限？
     │                              ↓ yes: content → 持久化指针
     ▼
  ② HISTORY_SNIP (feature)     ←─ 裁最老几条
     │
     ▼
  ③ microcompact
     ├ time-based: 空闲 N min? ── 清旧 tool_result content
     └ cached MC:  数量超阈值?  ── 发 cache_edits，服务器删
     │
     ▼
  ④ CONTEXT_COLLAPSE (feature) ←─ 中间段折叠成摘要
     │
     ▼
  ⑤ autocompact
     │   tokenCount >= threshold?
     │     ↓ yes
     │   trySessionMemoryCompaction → 成功 return
     │     ↓ fail
     │   compactConversation
     │     - PreCompact hooks
     │     - fork agent 生成 summary (9 段结构)
     │     - 清 readFileState，重新 attach 文件
     │     - 生成 boundaryMarker
     │     - SessionStart hook
     │   → 失败 3 次熔断
     │
     ▼
  → API request
     ↓ 返回 413?
  ⑥ reactiveCompact (withhold 住 error，重试 compact 再 continue)
     ↓ 返回 max_output_tokens?
     escalate 8k → 64k → "resume" user meta → 多轮续写
```

### 5. 关键设计点

1. **分层 vs 一刀切**：从"替换一个 tool result"到"整段 summary"，同一会话里多层机制并存，按 token 压力**逐级升级**。
2. **Prompt cache 友好**：`ContentReplacementState` 用 `seenIds` 冻结决策，`cached microcompact` 直接用服务器侧 cache_edits，`tengu_compact_cache_prefix` 让 compact fork 共享主会话前缀——尽一切可能避免 cache 失效。
3. **前置 vs 反应式**：多数层是 pre-request 预防；reactive compact 只在 API 真失败才触发，作为 413 的最后防线。
4. **熔断 + 追踪**：`AutoCompactTrackingState.consecutiveFailures` + `turnId/turnCounter` 既防死循环，又能让 `tengu_compact` 事件区分 same-chain loop vs cross-agent vs 手动 compact。
5. **可恢复边界**：所有压缩都产生 boundary message 并保留原消息 UUID，`/resume`、`--continue`、teleport 能精确重建拓扑。

---

## 六、Context Collapse 详解（feature-gated）

### 1. 是什么

**Context Collapse 是一种"细粒度、读时投影、段级、不破坏滚屏"的压缩机制**，比 autocompact 更轻：autocompact 把整段历史压成一个 summary，collapse 则**挑历史里的若干"段"(span)替换成 `<collapsed id="...">summary</collapsed>` 占位符**。

⚠️ `services/contextCollapse/` 目录在外部 build 里被 `feature('CONTEXT_COLLAPSE')` DCE 掉了，本镜像没有实际实现文件。下面分析基于：
- [query.ts:18-19](src/query.ts#L18-L19) 的 `require()`
- [query.ts:440-447](src/query.ts#L440-L447) 调用点
- [types/logs.ts:240-295](src/types/logs.ts#L240-L295) 持久化条目类型
- [autoCompact.ts:174-182, 215-223](src/services/compact/autoCompact.ts#L174-L182) 让位逻辑
- [analyzeContext.ts:1119-1126](src/utils/analyzeContext.ts#L1119-L1126) UI 集成
- [REPL.tsx:3677-3688](src/screens/REPL.tsx#L3677-L3688) rewind 处理
- [sessionStorage.ts:1208-1215, 3694-3697](src/utils/sessionStorage.ts#L1208-L1215) resume 重放

### 2. 核心数据结构

每个 commit = "把 [firstArchivedUuid, lastArchivedUuid] 这段消息**视为**一个折叠占位符"：

```ts
type ContextCollapseCommitEntry = {
  type: 'marble-origami-commit',          // 'marble_origami' 是内部代号
  collapseId: string,                      // 16 位折叠 ID
  summaryUuid: string,                     // 摘要占位符的 uuid
  summaryContent: string,                  // <collapsed id="...">text</collapsed> 完整串
  summary: string,                         // 纯文本摘要（给 ctx_inspect 用）
  firstArchivedUuid: string,               // ★ span 起点
  lastArchivedUuid: string,                // ★ span 终点
}

type ContextCollapseSnapshotEntry = {
  type: 'marble-origami-snapshot',
  staged: Array<{
    startUuid: string,
    endUuid: string,
    summary: string,
    risk: number,                          // ★ 折叠风险评分
    stagedAt: number,                      // ★ 入队时间
  }>,
  armed: boolean,                          // 触发器是否已 armed
  lastSpawnTokens: number,                 // 上次 spawn 时的 token 数
}
```

**关键事实**：
- **archived 消息不持久化** —— 它们已经在 transcript 里以原始形式存在
- **collapse store** 是独立的 module-level 状态，存 commit list + staged queue + spawn state
- 两份数据**永不合并到磁盘**

### 3. 读时投影 —— `projectView`

```
原始消息数组（REPL 持有）
   m1, m2, m3, m4, m5, m6, m7, m8, m9, m10
                  ▲              ▲
              first(uuid)   last(uuid)
                  └──── commit A ────┘ summary="..."

projectView(messages, collapseStore) =
   m1, m2, [<collapsed id="A">summary</collapsed>], m8, m9, m10
                    ↑ 不存在于 messages 数组,
                       是投影时从 collapse store 拼进来的
```

**REPL UI 看完整原始版本**，**API 看投影后的折叠版本**——透明压缩。

### 4. ctx-agent 工作链路

代号 **`marble_origami`**（[autoCompact.ts:174-182](src/services/compact/autoCompact.ts#L174-L182)），是个**后台 fork agent**：

```
1. 主会话每涨 token / 每过几轮
   ↓
2. ctx-agent armed (snapshot.armed = true)
   ↓
3. 后台 fork: 让 ctx-agent 扫一段历史
   ↓
4. 它产出若干 candidate span,每个有:
   - [startUuid, endUuid] 边界
   - summary（一句话总结这段做了什么）
   - risk（折叠这段会丢多少信息）
   ↓
5. 进 staged 队列（还没生效）
   ↓
6. applyCollapsesIfNeeded 在每轮 query 开头:
   - 看 token 压力
   - 从 staged 里挑 risk 最低的若干 span 提交（commit）
   - commit 的 span 立刻进入 projectView
   ↓
7. 持久化到 transcript
   （marble-origami-commit / -snapshot 两类条目）
```

### 5. `applyCollapsesIfNeeded` 的内部步骤

调用签名 ([query.ts:440-447](src/query.ts#L440-L447))：

```ts
applyCollapsesIfNeeded(
  messagesForQuery: Message[],
  toolUseContext: ToolUseContext,
  querySource: QuerySource,
): { messages: Message[] }
```

按周边代码反推：

1. **`projectView(messagesForQuery)`** —— 读 collapse store 的 commit 列表，把 `[firstArchivedUuid, lastArchivedUuid]` 之间的消息**替换**成 `<collapsed id="...">summary</collapsed>` user message。
2. **算当前 token** —— `tokenCountWithEstimation(projectedMessages)`。
3. **决定要不要 commit 更多 staged span**：90% 开始 commit 低风险的，95% 阻塞性 commit。
4. **新 commit 的 span 通过 `appendEntry` 持久化**到 transcript。
5. **返回**包含投影后消息的 `{ messages }`。

注意函数**不 yield 任何东西**——整个 collapse 对 UI 透明。UI 只在 `/context` 命令里显式调 `projectView` 才能看到投影视图（[context.tsx:20-26](src/commands/context/context.tsx#L20-L26)）。

### 6. 与 autocompact 的让位关系

[query.ts:428-431](src/query.ts#L428-L431) 的注释点明了 collapse 存在的全部价值：

> Runs BEFORE autocompact so that if collapse gets us under the autocompact threshold, autocompact is a no-op and we keep granular context instead of a single summary.

```
传统:
  history (180k) ──autocompact──► [ summary (17k) + recent (50k) ]
                                         ↑
                                  整个历史变成一个粗摘要

Collapse:
  history (180k) ──collapse──► [ early-msgs (20k) + <collapsed A> + 
                                  middle-msgs (15k) + <collapsed B> + 
                                  recent (50k) ]
                                         ↑
                                  保留全部细节脉络,
                                  只折叠"已不重要"的中段
```

互斥关系：
- `shouldAutoCompact` ([autoCompact.ts:215-223](src/services/compact/autoCompact.ts#L215-L223))：collapse 启用时 autocompact **完全让位**。
- `analyzeContext.ts:1119-1126`：UI 不显示 autocompact 的预留 buffer。
- ContextVisualization.tsx 改用 collapse 自己的阈值梯度。

### 7. 413 兜底路径

[query.ts:1085-1117](src/query.ts#L1085-L1117) —— API 真返回 413 时：

```ts
if (isWithheld413) {
  if (feature('CONTEXT_COLLAPSE') && contextCollapse &&
      state.transition?.reason !== 'collapse_drain_retry') {
    const drained = contextCollapse.recoverFromOverflow(messagesForQuery, querySource)
    if (drained.committed > 0) {
      state = {
        ...,
        messages: drained.messages,
        transition: { reason: 'collapse_drain_retry', committed: drained.committed },
      }
      continue                           // 重试本轮
    }
  }
}
```

**`recoverFromOverflow`** —— 把 staged 队列里**所有**没 commit 的 span 强制全部 commit，重新投影，重试请求。`transition.reason === 'collapse_drain_retry'` 标记防止"drain → 仍 413 → 再 drain"死循环。

### 8. Rewind / Resume 处理

**Rewind**（[REPL.tsx:3677-3688](src/screens/REPL.tsx#L3677-L3688)）必须 `resetContextCollapse()`：

> Commits whose archived span was past the rewind point can't be projected anymore (projectView silently skips them) but the staged queue and ID maps reference stale uuids. Simplest safe reset: drop everything.

**Resume** ([sessionStorage.ts:3694-3697](src/utils/sessionStorage.ts#L3694-L3697)) 反过来加载：

```ts
} else if (entry.type === 'marble-origami-commit') {
  // 加载到 commit log（按顺序）
} else if (entry.type === 'marble-origami-snapshot') {
  // 加载 staged + armed + lastSpawnTokens（last-wins）
}
```

恢复后：commit list 重放、staged 队列重新填充、ctx-agent 从 `lastSpawnTokens` 接着累计。

### 9. 与 SessionMemory 的对比

| | SessionMemory | Context Collapse |
|---|---|---|
| 维护对象 | 一个固定结构的 `summary.md` markdown 文件 | 多个独立的 commit（每个对应一段 span） |
| 触发时机 | 每涨 N token + N tool call 的 hook | ctx-agent 后台 spawn + applyCollapsesIfNeeded 提交 |
| 折叠粒度 | 整段历史 → 一份笔记 | **段级**——可以折叠多个独立区间 |
| 保留细节 | 笔记是高度概括的 9 段总结 | **保留段间的具体消息**，仅折叠选中段 |
| 替换内容 | 替换整段历史 | 替换 [start, end] 之间的具体消息为 `<collapsed>` |
| 持久化 | 单个 `summary.md` 文件 | `marble-origami-commit` + `marble-origami-snapshot` 两类 transcript entries |
| 用法 | `trySessionMemoryCompaction` 走整段替换 | `applyCollapsesIfNeeded` 做读时投影 |
| 透明度 | 被压缩消息**消失**（用 messagesToKeep 保最近的） | 原消息留在 REPL 滚屏，仅投影时替换 |

### 10. 代号 `marble_origami` 的来历

- **marble**（大理石）：每个 collapse commit 是一块独立的、不可变的"石头"
- **origami**（折纸）：把一段消息"折"成一个紧凑表示

[types/logs.ts:249-253](src/types/logs.ts#L249-L253) 的注释解释为什么必须用混淆字符串：

> Discriminator is obfuscated to match the gate name. sessionStorage.ts isn't feature-gated, so a descriptive string here would leak into external builds via the appendEntry dispatch / loadTranscriptFile parser even though nothing in an external build ever writes or reads this entry.

字面量字符串会被外部构建引用到，所以用代号防泄密——典型的 ant-only feature 处理。

### 11. 一图概括

```
                  ┌──────────────────────────────────┐
                  │     主会话 messages 数组         │
                  │     m1, m2, m3, m4, m5, m6 ...   │  ◄── REPL UI 显示这个
                  └──────────────────────────────────┘
                                    │
                ctx-agent (后台)    │
                    │               │
                    ▼               │
                ┌─────────┐         │
                │ staged  │ <span, │
                │ queue   │  risk> │
                └─────────┘         │
                    │               │
                    ▼               │
              applyCollapsesIfNeeded
                    │               │
                    ▼               │
                ┌──────────────────────────┐
                │ collapse store (commit log)│
                │ [{startUuid, endUuid,    │
                │   summaryContent}, ...]  │
                └──────────────────────────┘
                    │               │
                    ▼               ▼
              projectView(messages, store)
                    │
                    ▼
            ┌─────────────────────────────┐
            │ messagesForQuery（投影版本） │
            │ m1, m2, <collapsed A>,      │
            │ m5, <collapsed B>           │  ◄── 发给 API 的是这个
            └─────────────────────────────┘
```

---

## 七、一句话总结

Claude Code 是一个 **React/Ink 终端应用 + 自研 agent loop + 40+ 工具生态 + 企业级配置/权限/遥测栈**，所有模式（REPL、headless SDK、KAIROS、coordinator）共享同一个 `QueryEngine` → `query()` 核心生成器。代码量真正集中在 [main.tsx](src/main.tsx)、[query.ts](src/query.ts)、[REPL.tsx](src/screens/REPL.tsx)、[AgentTool.tsx](src/tools/AgentTool/AgentTool.tsx)、[BashTool.tsx](src/tools/BashTool/BashTool.tsx) 这五个文件（合计 ~14000 行），其余是辅助基础设施。
