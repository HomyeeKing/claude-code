# forked subagent 实现梳理

本文基于 `src/` 目录中的真实实现，整理当前仓库里 **forked subagent** 的工作方式，重点回答两个问题：

- forked subagent 具体做了什么
- **主 agent 的上下文是如何传入子 agent 的**

同时也会顺带对比：**forked subagent** 和普通带 `subagent_type` 的子 agent，到底有什么本质区别。

## 结论

先给结论版：

1. **forked subagent 不是“新开一个空白 agent”**，而是尽量复用父 agent 当前这一轮的完整上下文前缀。
2. 它继承的不只是消息，还包括：
   - **父 agent 当前的消息历史**（通过 `forkContextMessages`）
   - **父 agent 已渲染完成的 system prompt**（通过 `renderedSystemPrompt` / `override.systemPrompt`）
   - **父 agent 的工具定义与 thinking 配置**（通过 `availableTools` + `useExactTools: true`）
3. fork 的目标不是单纯“并行”，而是：
   - **把中间过程从主上下文中隔离出去**
   - **尽可能复用父请求的 prompt cache**
   - **让子任务在后台独立推进**
4. 相比之下，普通 `subagent_type` 子 agent 默认是 **fresh context**：它不会自动继承父对话，只会收到主 agent 明确写给它的 prompt。
5. 虽然 fork 会继承父上下文，但它的运行态仍然是 **默认隔离、按需共享**，例如 `readFileState` 和 `contentReplacementState` 会 clone，而 `setAppState` 默认会被置成 no-op。

换句话说：

- **普通 subagent** 更像“把任务转交给一个刚进会议室的同事”
- **forked subagent** 更像“复制当前自己的一份上下文快照，拉一个后台分支继续做”

## 相关文件

这条链路主要涉及下面几个文件：

- `src/tools/AgentTool/AgentTool.tsx`
- `src/tools/AgentTool/runAgent.ts`
- `src/tools/AgentTool/forkSubagent.ts`
- `src/utils/forkedAgent.ts`
- `src/utils/queryContext.ts`
- `src/tools/AgentTool/resumeAgent.ts`
- `src/services/AgentSummary/agentSummary.ts`
- `src/utils/agentContext.ts`

## 一、forked subagent 的入口在哪

最直接的入口在 `AgentTool.tsx`。

这里会先判断当前是不是 fork 路径，然后分叉构造：

- **fork 路径**：复用父 system prompt、父消息、父工具池
- **普通路径**：构建所选 agent 自己的 system prompt，并只传一个新的 user prompt

关键代码在：

```622:632:src/tools/AgentTool/AgentTool.tsx
override: isForkPath ? {
  systemPrompt: forkParentSystemPrompt
} : enhancedSystemPrompt && !worktreeInfo && !cwd ? {
  systemPrompt: asSystemPrompt(enhancedSystemPrompt)
} : undefined,
availableTools: isForkPath ? toolUseContext.options.tools : workerTools,
forkContextMessages: isForkPath ? toolUseContext.messages : undefined,
...(isForkPath && {
  useExactTools: true
}),
```

这几行已经把 fork 的核心语义写得很清楚：

- `override.systemPrompt: forkParentSystemPrompt`
  - 子 agent 不用自己的 system prompt，而是复用父 agent 的 system prompt
- `availableTools: toolUseContext.options.tools`
  - 子 agent 直接用父 agent 当前的工具数组
- `forkContextMessages: toolUseContext.messages`
  - 子 agent 拿到父 agent 当前消息历史
- `useExactTools: true`
  - 不再按常规 subagent 流程重新组装工具与 thinking，而是尽量保持与父请求一致

这也是为什么 fork 不只是“省得写 `subagent_type`”，而是一个**专门为了上下文继承和缓存命中设计的分支**。

## 二、主 agent 的上下文是怎么传进去的

### 1. 消息上下文：通过 `forkContextMessages`

真正把父消息拼进子 agent 的地方在 `runAgent.ts`：

```368:373:src/tools/AgentTool/runAgent.ts
const contextMessages: Message[] = forkContextMessages
  ? filterIncompleteToolCalls(forkContextMessages)
  : []
const initialMessages: Message[] = [...contextMessages, ...promptMessages]
```

这段逻辑的含义是：

- 如果传了 `forkContextMessages`
  - 先用 `filterIncompleteToolCalls()` 清理掉不完整的 tool call，避免 API 报错
  - 然后把这些父消息作为子 agent 的上下文前缀
- 如果没传
  - `contextMessages` 就是空数组
  - 说明这个子 agent 默认不会自动看到父会话历史

所以：

- **forked subagent**：`initialMessages = 父消息 + fork 指令消息`
- **普通 subagent**：`initialMessages = [] + 新 prompt`

这就是“主 agent 上下文如何传入”的第一层答案：**通过 `forkContextMessages` 传消息历史**。

### 2. system prompt：通过 `renderedSystemPrompt` / `override.systemPrompt`

在 fork 路径里，`AgentTool.tsx` 不会重新为 FORK agent 生成新的 system prompt，而是优先使用父线程已经渲染好的 prompt：

```495:512:src/tools/AgentTool/AgentTool.tsx
if (isForkPath) {
  if (toolUseContext.renderedSystemPrompt) {
    forkParentSystemPrompt = toolUseContext.renderedSystemPrompt;
  } else {
    // Fallback: recompute.
    // ...
    forkParentSystemPrompt = buildEffectiveSystemPrompt({
      mainThreadAgentDefinition,
      toolUseContext,
      customSystemPrompt: toolUseContext.options.customSystemPrompt,
      defaultSystemPrompt,
      appendSystemPrompt: toolUseContext.options.appendSystemPrompt
    });
  }
  promptMessages = buildForkedMessages(prompt, assistantMessage);
}
```

这里有两个关键点：

1. **优先使用 `toolUseContext.renderedSystemPrompt`**
   - 这是父 agent 当前轮次已经渲染完成的 prompt bytes
   - 这么做是为了让 fork 与父请求拥有尽可能一致的 cache key 前缀
2. **只有拿不到时才 fallback 重建**
   - 代码注释明确说了：重新生成 system prompt 可能因为运行时状态变化而导致 cache miss

也就是说，主 agent 的“身份说明、系统规则、附加系统提示”等内容，并不是让 fork 再算一遍，而是**直接沿用父 agent 已经算好的版本**。

### 3. 工具集与 thinking：通过 `availableTools` + `useExactTools`

fork 路径还会显式传入：

- `availableTools: toolUseContext.options.tools`
- `useExactTools: true`

对应 `runAgent.ts` 中的逻辑：

```500:503:src/tools/AgentTool/runAgent.ts
const resolvedTools = useExactTools
  ? availableTools
  : resolveAgentTools(agentDefinition, availableTools, isAsync).resolvedTools
```

以及：

```666:695:src/tools/AgentTool/runAgent.ts
const agentOptions: ToolUseContext['options'] = {
  isNonInteractiveSession: useExactTools
    ? toolUseContext.options.isNonInteractiveSession
    : isAsync
      ? true
      : (toolUseContext.options.isNonInteractiveSession ?? false),
  appendSystemPrompt: toolUseContext.options.appendSystemPrompt,
  tools: allTools,
  commands: [],
  debug: toolUseContext.options.debug,
  verbose: toolUseContext.options.verbose,
  mainLoopModel: resolvedAgentModel,
  thinkingConfig: useExactTools
    ? toolUseContext.options.thinkingConfig
    : { type: 'disabled' as const },
  // ...
  ...(useExactTools && { querySource }),
}
```

这里的设计非常关键：

- 对普通 subagent，通常会重新解析 agent 允许的工具，并把 thinking 关掉来省 token
- 对 forked subagent，则尽可能让：
  - 工具定义一致
  - `thinkingConfig` 一致
  - `isNonInteractiveSession` 一致

因为这些字段都会影响请求前缀与缓存命中。

所以主 agent 上下文传入的第三层答案是：**不仅传消息和 system prompt，还传 cache-sensitive 的工具和 thinking 配置**。

## 三、fork 的 prompt 是怎么构造的

fork 不只是“把父历史塞进去”，它还会额外构造一段针对 fork worker 的专用指令。

这部分在 `forkSubagent.ts`。

### 1. `FORK_AGENT` 是一个特殊 agent 定义

```60:71:src/tools/AgentTool/forkSubagent.ts
export const FORK_AGENT = {
  agentType: FORK_SUBAGENT_TYPE,
  whenToUse:
    'Implicit fork — inherits full conversation context. Not selectable via subagent_type; triggered by omitting subagent_type when the fork experiment is active.',
  tools: ['*'],
  maxTurns: 200,
  model: 'inherit',
  permissionMode: 'bubble',
  source: 'built-in',
  baseDir: 'built-in',
  getSystemPrompt: () => '',
}
```

这里最重要的不是它的字段值，而是注释语义：

- `model: 'inherit'`
- `permissionMode: 'bubble'`
- `getSystemPrompt: () => ''` 实际上不会使用

因为 fork 真正要用的是 **父 prompt**，而不是这个 synthetic agent 自己生成的 prompt。

### 2. `buildForkedMessages()` 会复制父 assistant 消息并补齐 tool_result

fork 的难点在于：父 agent 上一条 assistant 消息里，可能已经包含一堆 `tool_use` block。如果直接截一半上下文给子 agent，很容易造成 API 侧的 tool_use / tool_result 不配对。

所以 `buildForkedMessages()` 会做一个特殊处理：

```107:168:src/tools/AgentTool/forkSubagent.ts
export function buildForkedMessages(
  directive: string,
  assistantMessage: AssistantMessage,
): MessageType[] {
  const fullAssistantMessage: AssistantMessage = {
    ...assistantMessage,
    uuid: randomUUID(),
    message: {
      ...assistantMessage.message,
      content: [...assistantMessage.message.content],
    },
  }

  const toolUseBlocks = assistantMessage.message.content.filter(
    (block): block is BetaToolUseBlock => block.type === 'tool_use',
  )

  const toolResultBlocks = toolUseBlocks.map(block => ({
    type: 'tool_result' as const,
    tool_use_id: block.id,
    content: [
      {
        type: 'text' as const,
        text: FORK_PLACEHOLDER_RESULT,
      },
    ],
  }))

  const toolResultMessage = createUserMessage({
    content: [
      ...toolResultBlocks,
      {
        type: 'text' as const,
        text: buildChildMessage(directive),
      },
    ],
  })

  return [fullAssistantMessage, toolResultMessage]
}
```

它做了三件事：

1. **复制父 assistant message**，保留其中的 thinking / text / tool_use
2. 为其中每个 `tool_use` 生成一个统一占位的 `tool_result`
3. 把真正要做的 fork 指令追加到同一个 user message 里

这背后的目的仍然是两个：

- 保证消息结构对 API 合法
- 让多个 fork child 共享尽量一致的请求前缀，从而命中 prompt cache

### 3. child message 会强行改写行为约束

`buildChildMessage()` 里还会注入一段很强的规则，例如：

- 你是 forked worker，不是主 agent
- 不要继续再 fork
- 不要对话，不要插入多余 commentary
- 直接使用工具执行，然后最后一次性汇报

这说明 fork 的定位非常明确：**它是后台工作分身，不是一个继续和用户多轮交互的新会话**。

## 四、为什么普通 subagent 默认拿不到父上下文

这一点其实正好从 `runAgent.ts` 反推出来：

```368:373:src/tools/AgentTool/runAgent.ts
const contextMessages: Message[] = forkContextMessages
  ? filterIncompleteToolCalls(forkContextMessages)
  : []
const initialMessages: Message[] = [...contextMessages, ...promptMessages]
```

普通 `subagent_type` 路径不会传 `forkContextMessages`，所以：

- `contextMessages = []`
- `initialMessages = promptMessages`

这和 `AgentTool` 自己的提示文案是吻合的：普通 fresh agent 需要主 agent **在 prompt 里重新讲清楚背景**。

对应提示词里就写得很直白：

```101:113:src/tools/AgentTool/prompt.ts
## Writing the prompt

When spawning a fresh agent (with a `subagent_type`), it starts with zero context.
Brief the agent like a smart colleague who just walked into the room — it hasn't seen this conversation, doesn't know what you've tried, doesn't understand why this task matters.
```

所以不要把两者混为一谈：

- **fork**：继承上下文
- **subagent_type**：默认零上下文，靠 prompt 补背景

## 五、子 agent 的运行态是如何隔离/共享的

消息和 prompt 只是“输入上下文”，真正执行时还有一层 `ToolUseContext`。这部分由 `createSubagentContext()` 负责构造。

关键代码在 `src/utils/forkedAgent.ts`：

```376:417:src/utils/forkedAgent.ts
return {
  readFileState: cloneFileStateCache(
    overrides?.readFileState ?? parentContext.readFileState,
  ),
  nestedMemoryAttachmentTriggers: new Set<string>(),
  loadedNestedMemoryPaths: new Set<string>(),
  dynamicSkillDirTriggers: new Set<string>(),
  discoveredSkillNames: new Set<string>(),
  toolDecisions: undefined,
  contentReplacementState:
    overrides?.contentReplacementState ??
    (parentContext.contentReplacementState
      ? cloneContentReplacementState(parentContext.contentReplacementState)
      : undefined),
  abortController,
  getAppState,
  setAppState: overrides?.shareSetAppState
    ? parentContext.setAppState
    : () => {},
  setAppStateForTasks:
    parentContext.setAppStateForTasks ?? parentContext.setAppState,
```

这里的核心设计是：**默认隔离，按需共享**。

### 1. 默认 clone / fresh 的部分

- `readFileState`：clone
- `contentReplacementState`：clone
- `nestedMemoryAttachmentTriggers`：新建 Set
- `loadedNestedMemoryPaths`：新建 Set
- `dynamicSkillDirTriggers`：新建 Set
- `discoveredSkillNames`：新建 Set
- `toolDecisions`：重置

这意味着子 agent 不会直接污染父 agent 的这些运行态。

### 2. 默认 no-op / 禁写的部分

- `setAppState`：默认变成空函数
- `setInProgressToolUseIDs`：空函数
- `updateFileHistoryState`：空函数

也就是说，大部分“会反向改写主线程状态”的能力，默认都被切断了。

### 3. 仍然保留共享通道的部分

- `setAppStateForTasks`
  - 始终指向 root store，用于后台 bash task / hooks 这类 session 级基础设施
- `updateAttributionState`
  - 直接共享
- `setResponseLength`
  - 只有显式要求时共享
- `abortController`
  - 可配置共享，也可创建 child controller

所以 forked subagent 并不是“复制整个主 agent 进程”，而是：

- **输入上下文尽量复用父前缀**
- **执行期状态默认隔离**
- **少数必要通道保留共享**

## 六、为什么还要 clone `contentReplacementState`

`createSubagentContext()` 里有一段注释非常关键，大意是：

- fork child 会处理父消息里的 `tool_use_id`
- 如果给它一个全新的 replacement state
- 它会对这些 tool results 做出与父线程不同的替换决策
- 最终导致请求前缀不同，cache miss

对应代码：

```388:403:src/utils/forkedAgent.ts
contentReplacementState:
  overrides?.contentReplacementState ??
  (parentContext.contentReplacementState
    ? cloneContentReplacementState(parentContext.contentReplacementState)
    : undefined),
```

这说明 fork 的“继承父上下文”并不只是把消息数组塞过去，而是还要保证**与消息解释相关的局部运行态也能对齐**。

## 七、fork 在 query 层面如何继续流转

在 `runAgent()` 最终调用 `query()` 时，真正喂给模型的是：

- `messages: initialMessages`
- `systemPrompt: agentSystemPrompt`
- `userContext: resolvedUserContext`
- `systemContext: resolvedSystemContext`
- `toolUseContext: agentToolUseContext`

对应代码：

```747:757:src/tools/AgentTool/runAgent.ts
for await (const message of query({
  messages: initialMessages,
  systemPrompt: agentSystemPrompt,
  userContext: resolvedUserContext,
  systemContext: resolvedSystemContext,
  canUseTool,
  toolUseContext: agentToolUseContext,
  querySource,
  maxTurns: maxTurns ?? agentDefinition.maxTurns,
})) {
```

所以从 query 角度看，fork 和普通 subagent 的差异主要已经在前面准备阶段决定了：

- `initialMessages` 是否包含父上下文
- `agentSystemPrompt` 是否直接复用父 prompt
- `toolUseContext.options` 是否保持 exact match

### `query.ts` 中也会把当前消息窗口保存为后续 fork 的 cache-safe params

例如在自动 compact 前，会构造：

```457:463:src/query.ts
{
  systemPrompt,
  userContext,
  systemContext,
  toolUseContext,
  forkContextMessages: messagesForQuery,
}
```

这类 `CacheSafeParams` 会被后续各种 fork 型流程复用。

## 八、`runForkedAgent()`：仓库里另一条“fork query”通用路径

除了 `AgentTool` 里直接创建 fork child，这个仓库还有一条更底层的通用 fork 路径：`runForkedAgent()`。

它被 `session memory`、`prompt suggestion`、`compact`、`agent summary` 等功能复用。

关键实现：

```506:524:src/utils/forkedAgent.ts
const {
  systemPrompt,
  userContext,
  systemContext,
  toolUseContext,
  forkContextMessages,
} = cacheSafeParams

const isolatedToolUseContext = createSubagentContext(
  toolUseContext,
  overrides,
)

const initialMessages: Message[] = [...forkContextMessages, ...promptMessages]
```

这个函数进一步证明：当前仓库里“fork”的抽象，本质上就是：

1. 拿一组 **cache-safe params**
2. clone 一个隔离的 `ToolUseContext`
3. 把 `forkContextMessages` 当作上下文前缀
4. 追加新的 promptMessages
5. 再跑一轮 query

这和 `AgentTool` 的 fork 路径是同一思想，只是封装层级不同。

## 九、fork child 的生命周期补充

### 1. 会记录 sidechain transcript

`runAgent()` 在开始前和消息流转过程中都会记录 transcript：

```732:742:src/tools/AgentTool/runAgent.ts
void recordSidechainTranscript(initialMessages, agentId).catch(...)
void writeAgentMetadata(agentId, {
  agentType: agentDefinition.agentType,
  ...(worktreePath && { worktreePath }),
  ...(description && { description }),
}).catch(...)
```

这意味着 fork child 虽然脱离主上下文运行，但它自己的历史是可持久化、可恢复的。

### 2. resume 时会继续按 fork 语义恢复

`resumeAgent.ts` 会识别 metadata 里的 `agentType === fork`，然后重新使用父 system prompt：

```102:105:src/tools/AgentTool/resumeAgent.ts
if (meta?.agentType === FORK_AGENT.agentType) {
  selectedAgent = FORK_AGENT
  isResumedFork = true
}
```

以及：

```180:190:src/tools/AgentTool/resumeAgent.ts
override: isResumedFork
  ? { systemPrompt: forkParentSystemPrompt }
  : undefined,
availableTools: workerTools,
forkContextMessages: undefined,
...(isResumedFork && { useExactTools: true }),
```

这里的关键点是：

- resume fork 时，不再重新传 `forkContextMessages`
- 因为 transcript 里已经有最初 fork 时的上下文片段
- 但仍然要保持 fork 的 system prompt / exact tools 语义

### 3. 可以被周期性再 fork 做 summary

`AgentSummary/agentSummary.ts` 会周期性读取 agent transcript，再 fork 一次来生成简短进度摘要：

```81:118:src/services/AgentSummary/agentSummary.ts
const forkParams: CacheSafeParams = {
  ...baseParams,
  forkContextMessages: cleanMessages,
}

const result = await runForkedAgent({
  promptMessages: [
    createUserMessage({ content: buildSummaryPrompt(previousSummary) }),
  ],
  cacheSafeParams: forkParams,
  canUseTool,
  querySource: 'agent_summary',
  forkLabel: 'agent_summary',
  overrides: { abortController: summaryAbortController },
  skipTranscript: true,
})
```

也就是说，fork 不只是“用户显式调用 Agent tool 时的行为”，而是整个系统内部很多派生能力的基础设施。

## 十、AsyncLocalStorage 里的 agent 身份标记

`src/utils/agentContext.ts` 说明了 subagent 的身份如何在线程级异步上下文中传播：

```32:54:src/utils/agentContext.ts
export type SubagentContext = {
  agentId: string
  parentSessionId?: string
  agentType: 'subagent'
  subagentName?: string
  isBuiltIn?: boolean
  invokingRequestId?: string
  invocationKind?: 'spawn' | 'resume'
  invocationEmitted?: boolean
}
```

这部分不直接决定消息上下文如何传入，但它负责：

- analytics attribution
- parent/child session 关联
- spawn / resume 边界标记

所以如果要完整理解“forked subagent 做了什么”，也要把它看作：

- 一条独立的 query 执行链
- 一份隔离的 `ToolUseContext`
- 一个带 subagent 身份标识的异步执行上下文

## 十一、forked subagent 与普通 subagent 的差异表

| 维度 | forked subagent | 普通 `subagent_type` 子 agent |
| --- | --- | --- |
| 初始消息 | 继承父 `messages`，再追加 fork directive | 默认只有新的 `promptMessages` |
| system prompt | 复用父 `renderedSystemPrompt` | 使用所选 agent 自己的 system prompt |
| 工具定义 | 尽量复用父工具数组 | 按 agent 权限重新解析工具 |
| thinkingConfig | 尽量继承父配置 | 通常禁用 thinking 控制成本 |
| prompt cache 目标 | 强依赖，尽量 cache-identical | 不强调与父请求完全一致 |
| 背景说明方式 | 靠继承父上下文 | 靠主 agent 在 prompt 中重新说明 |
| 典型用途 | 把复杂工作分叉出去、减少主上下文污染 | 独立视角审查、专门角色执行 |

## 十二、最终总结

如果把 forked subagent 的实现压缩成一句话，可以概括为：

> **它会把主 agent 当前轮次的上下文前缀、system prompt 和工具/思考配置尽量原样复用，再在一个隔离的子 `ToolUseContext` 中继续跑一条新的 query 链。**

更具体一点：

1. **入口在 `AgentTool.tsx`**
   - 判断是不是 fork path
   - 决定是否复用父 prompt / tools / messages
2. **消息通过 `forkContextMessages` 传入**
   - 在 `runAgent.ts` 中与 `promptMessages` 拼成 `initialMessages`
3. **system prompt 通过 `renderedSystemPrompt` 传入**
   - 不是重新算一份，而是尽量继承父线程已经渲染好的版本
4. **运行态通过 `createSubagentContext()` 隔离**
   - clone 必要缓存与 replacement state
   - 默认禁止直接改写主线程状态
5. **整个设计围绕 prompt cache 和上下文隔离展开**
   - 这也是它和普通 fresh subagent 最大的差别

所以，当你问“forked subagent 具体做了什么”，最准确的回答不是“它开了一个子 agent”，而是：

> **它复制了主 agent 当前可复用的请求前缀，拉起一条新的子执行链，并把大部分执行噪音留在子链路里，不再继续污染主 agent 的上下文窗口。**
