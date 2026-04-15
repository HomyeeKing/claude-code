# Claude Code 中 Attachment Message 与 Prompt Cache 的关系

本文总结 `src/utils/attachments.ts` 这一套 `attachment message` 机制，重点回答以下问题：

- `attachment` 到底是什么
- 它会不会进入 prompt
- 它是不是天然不参与 prompt cache
- 为什么代码里还要做 attachment 重排

## 一、核心结论

先给结论版：

- **这里的 attachment 不是传统“上传文件附件”概念，而是会话中的动态附加消息。**
- **attachment 会进入 prompt。** 它不会直接以 `attachment` 类型发给模型，而是先被转换成 `UserMessage[]`。
- **调用 `normalizeMessagesForAPI(...)` 不等于加入缓存。** 这一步只表示它进入了最终 API 请求消息序列。
- **真正决定 prompt cache 的，是后续 `cache_control` 标记加在最终消息序列的什么位置。**
- **attachment 不是天然“不参与缓存”。** 如果它最终位于 cache marker 之前，它同样会成为 cached prefix 的一部分。
- **attachment 也不是统一放到整个序列最后。** 它会在发 API 前经历一次局部重排，目的是让 attachment 尽量贴近相关上下文，同时减少对稳定缓存前缀的破坏。

一句话概括就是：

> **attachment 是一种动态上下文注入机制；它会进入 prompt，但是否进入 cached prefix，取决于它在最终消息序列中的位置，而不是取决于它是不是 attachment。**

## 二、attachment 是什么

`Attachment` 的类型定义在 `src/utils/attachments.ts`：

```440:448:/Users/bytedance/Desktop/homyee/claude-code/src/utils/attachments.ts
export type Attachment =
  /**
   * User at-mentioned the file
   */
  | FileAttachment
  | CompactFileReferenceAttachment
  | PDFReferenceAttachment
  | AlreadyReadFileAttachment
```

内部消息构造函数同样在这个文件里：

```3201:3206:/Users/bytedance/Desktop/homyee/claude-code/src/utils/attachments.ts
export function createAttachmentMessage(
  attachment: Attachment,
): AttachmentMessage {
  return {
    type: 'attachment',
```

这说明系统内部确实存在一类专门的消息：

- 消息类型是 `type: 'attachment'`
- 负载是 `attachment: Attachment`
- 它不是普通用户文本，也不是普通 assistant 回复

### 2.1 attachment 主要包含什么

这套 attachment 的内容非常多，但可以按用途分成几大类。

#### 文件 / IDE / 代码上下文类

典型类型：

- `file`
- `compact_file_reference`
- `pdf_reference`
- `already_read_file`
- `edited_text_file`
- `edited_image_file`
- `directory`
- `selected_lines_in_ide`
- `opened_file_in_ide`

这一类的作用是：

- 把用户 `@` 到的文件、目录、PDF、IDE 选区、文件修改信息注入对话上下文

#### memory 类

典型类型：

- `nested_memory`
- `relevant_memories`
- `current_session_memory`

这一类的作用是：

- 把与当前问题相关的 memory 动态补到上下文里

#### tool / hook / 结构化结果类

典型类型：

- `async_hook_response`
- `hook_*`
- `structured_output`

这一类的作用是：

- 将 hook 执行结果、工具附加上下文、结构化输出注入会话

#### 任务 / 队列 / 状态类

典型类型：

- `queued_command`
- `todo_reminder`
- `task_reminder`
- `task_status`
- `command_permissions`

这一类的作用是：

- 将运行中的任务、待办、权限或排队命令状态注入模型上下文

#### 运行模式 / 系统提醒类

典型类型：

- `plan_mode`
- `plan_mode_reentry`
- `plan_mode_exit`
- `auto_mode`
- `auto_mode_exit`
- `critical_system_reminder`
- `verify_plan_reminder`
- `max_turns_reached`
- `compaction_reminder`
- `context_efficiency`
- `date_change`

这一类的作用是：

- 将运行模式、系统提醒、轮次边界、上下文管理信息动态告诉模型

#### skills / agent / 协作类

典型类型：

- `dynamic_skill`
- `skill_listing`
- `skill_discovery`
- `invoked_skills`
- `agent_mention`
- `agent_listing_delta`
- `teammate_mailbox`
- `team_context`

这一类的作用是：

- 为多 agent、skills 和团队协作场景提供额外上下文

所以这里的 attachment，本质上不是“上传的文件附件”，而是：

> **文件上下文、memory、tool/hook 结果、任务状态、运行模式提醒、协作信息等动态内容的统一注入机制。**

## 三、attachment 如何进入 prompt

内部的 `attachment message` 不会原样发给 API，而是先转成普通消息。

对应函数在 `src/utils/messages.ts`：

```3453:3456:/Users/bytedance/Desktop/homyee/claude-code/src/utils/messages.ts
export function normalizeAttachmentForAPI(
  attachment: Attachment,
): UserMessage[] {
```

这表示：

- attachment 会被转换成一个或多个 `UserMessage`
- 然后再参与最终的 API 请求组装

例如一些 attachment 会被包装成：

- meta user message
- 模拟工具调用/工具结果的消息
- 文件读取或系统提醒消息

所以从机制上看：

- **attachment 会进入 prompt**
- 但它不是以 `type: 'attachment'` 的原始形式进入，而是经过 API 归一化之后进入

## 四、`normalizeMessagesForAPI(...)` 到底意味着什么

`normalizeMessagesForAPI(...)` 表示：

- 内部消息会被整理成最终 API 需要的消息序列
- attachment、system、user、assistant 等会在这里统一归一化

对应实现开头在 `src/utils/messages.ts`：

```1989:2000:/Users/bytedance/Desktop/homyee/claude-code/src/utils/messages.ts
export function normalizeMessagesForAPI(
  messages: Message[],
  tools: Tools = [],
): (UserMessage | AssistantMessage)[] {
  // First, reorder attachments to bubble up until they hit a tool result or assistant message
  const reorderedMessages = reorderAttachmentsForAPI(messages).filter(
    m => !((m.type === 'user' || m.type === 'assistant') && m.isVirtual),
  )
```

需要特别强调的是：

- **调用 `normalizeMessagesForAPI(...)` 不等于加入缓存**
- 它只意味着：**这条内容进入了最终 prompt**

缓存与否，不由这一步决定，而由后面的 `cache_control` 标记决定。

## 五、prompt cache 的标记是怎么加的

真正和缓存相关的是 `src/services/api/claude.ts` 里的 `addCacheBreakpoints(...)`。

### 5.1 只选一个消息位置作为 cache marker

```3078:3095:/Users/bytedance/Desktop/homyee/claude-code/src/services/api/claude.ts
// Exactly one message-level cache_control marker per request.
const markerIndex = skipCacheWrite ? messages.length - 2 : messages.length - 1
const result = messages.map((msg, index) => {
  const addCache = index === markerIndex
  if (msg.type === 'user') {
    return userMessageToMessageParam(
      msg,
      addCache,
```

这说明：

- 一次请求只会有一个 message-level cache marker
- 默认是最后一条消息
- `skipCacheWrite = true` 时，会退到倒数第二条消息

### 5.2 `cache_control` 加在 block 上，而不是所有消息上

对 user message：

```594:605:/Users/bytedance/Desktop/homyee/claude-code/src/services/api/claude.ts
if (typeof message.message.content === 'string') {
  return {
    role: 'user',
    content: [
      {
        type: 'text',
        text: message.message.content,
        ...(enablePromptCaching && {
          cache_control: getCacheControl({ querySource }),
        }),
      },
    ],
  }
}
```

对 block 数组形式的 user message：

```611:618:/Users/bytedance/Desktop/homyee/claude-code/src/services/api/claude.ts
content: message.message.content.map((_, i) => ({
  ..._,
  ...(i === message.message.content.length - 1
    ? enablePromptCaching
      ? { cache_control: getCacheControl({ querySource }) }
      : {}
    : {}),
})),
```

这说明：

- 不是每个 block 都加 `cache_control`
- 通常只给被选中的那条消息的**最后一个合适 block**加 `cache_control`

### 5.3 `cache_control` 长什么样

`getCacheControl(...)` 会返回：

```358:373:/Users/bytedance/Desktop/homyee/claude-code/src/services/api/claude.ts
export function getCacheControl({
  scope,
  querySource,
}: {
  scope?: CacheScope
  querySource?: QuerySource
} = {}): {
  type: 'ephemeral'
  ttl?: '1h'
  scope?: CacheScope
} {
  return {
    type: 'ephemeral',
```

也就是说，它的本质是：

- `type: 'ephemeral'`
- 某些场景可带 `ttl: '1h'`
- 某些 system prompt block 可带 `scope: 'global'`

## 六、prefix 到底是什么

这里说的 prefix，不是指某一个独立 message，也不是单指 system prompt。

更准确地说，prefix 指的是：

> **最终 API 请求里，从最开始到带 `cache_control` 的那个 block 为止的全部内容。**

也就是按最终请求顺序串起来的：

- system prompt blocks
- user messages
- assistant messages
- 其中包含的 block 内容
- 以及 attachment 归一化后生成的 user message 内容

只要它们位于 `cache_control` 之前或标记所在 block 本身，它们就属于 prefix。

所以：

- `cache_control` 不是“只缓存这个 block”
- 而是在定义：“**缓存前缀到这里为止**”

## 七、attachment 会不会参与缓存

答案是：

- **会进入 prompt**
- **可能进入缓存前缀**
- **但不是天然一定进入，也不是天然一定排除**

真正决定它是否参与 prefix 的，是：

- attachment 最终被 normalize 后落在什么位置
- cache marker 在哪里

因此需要避免两个误解：

### 误解 1：attachment 天然不参与缓存

不对。它如果落在 marker 之前，同样会进入 cached prefix。

### 误解 2：attachment 一定都放在最后，所以天然不缓存

也不对。attachment 不会被简单粗暴地统一放到整个序列最后。

## 八、为什么还需要 attachment 重排

这是最容易混淆的地方。

代码里在 `normalizeMessagesForAPI(...)` 的开头写了：

```1996:2000:/Users/bytedance/Desktop/homyee/claude-code/src/utils/messages.ts
// First, reorder attachments to bubble up until they hit a tool result or assistant message
const reorderedMessages = reorderAttachmentsForAPI(messages).filter(
  m => !((m.type === 'user' || m.type === 'assistant') && m.isVirtual),
)
```

这说明 attachment 在发 API 前会做一次**局部重排**。

### 8.1 这不是“全部扔到最后”

它的目标不是：

- 把所有 attachment 统一丢到请求末尾

而是：

- 让 attachment 尽量贴近它依赖或解释的上下文
- 同时不要跨越某些关键消息边界

这里的边界主要是：

- `assistant message`
- `tool result`

也就是说，attachment 的排布更像：

- **局部靠后**
- **贴近相关上下文**
- **但不是全局队尾堆叠**

### 8.2 为什么这样设计

原因可以从两个角度理解。

#### 从语义角度

很多 attachment 是这一轮动态生成的，例如：

- tool 执行后的附加结果
- hook 补充说明
- memory prefetch
- 运行模式提醒

这些内容在语义上通常属于某个最近上下文，而不是一个独立、遥远的尾部备注。

如果全部机械地放到最后：

- 会削弱 attachment 与相关工具结果、assistant 输出之间的关联
- 会让消息结构更难以保持自然的 turn 语义

#### 从缓存角度

attachment 又常常是本轮动态内容，因此确实应该尽量靠后，避免破坏前面稳定的共享前缀。

所以最终策略不是“全局最后”，而是：

> **attachment 尽量停留在后段，并贴近相关上下文，从而在语义正确性和缓存稳定性之间做平衡。**

## 九、tool message 和 attachment 的关系

前面分析 attachment 时，另一个容易混淆的问题是 tool message。

这里也顺手明确一下：

- tool 相关内容也会进入 prompt
- 如果它位于 marker 之前，也同样可能参与 cached prefix
- 代码里甚至还会给位于 cached prefix 内的 `tool_result` block 加 `cache_reference`

所以同样不能简单说：

- tool message 不参与缓存

更准确的说法是：

- **旧的、已经沉淀到前面的 tool 内容可能在 prefix 里**
- **最新这一轮新产生的 tool 内容通常更像尾部动态内容**

## 十、最终结论

如果只记住一句话，我建议记这句：

> **attachment 是一种动态上下文注入机制；它会被转换成 `UserMessage` 进入 prompt，但是否进入 prompt cache 前缀，不由 attachment 类型本身决定，而由它在最终消息序列中的位置和 `cache_control` marker 的边界共同决定。**

如果再展开一层，可以记成下面四点：

- **attachment 会进 prompt**，不是纯 UI 装饰
- **`normalizeMessagesForAPI(...)` 只代表进入最终 prompt，不代表加入缓存**
- **缓存边界由 `addCacheBreakpoints(...)` 通过 `cache_control` 标记决定**
- **attachment 会经过局部重排，目标是兼顾语义邻近性与缓存前缀稳定性，而不是一律丢到全局最后**

## 十一、一个不带源码的直观示例

如果只想直观理解 `normalizeMessagesForAPI(...)` 做了什么，可以把它想成“把内部消息整理成最终发给模型的 user / assistant 消息序列”。

### 11.1 假设内部消息原本长这样

先假设系统内部当前有这么几条消息：

- 一条用户消息：`帮我看看 src/query.ts`
- 一条 attachment：`file`
  - 含义是：用户通过 `@` 或别的方式把 `src/query.ts` 这个文件附加进来了
- 一条 assistant 消息：`我先阅读这个文件`
- 一条 attachment：`edited_text_file`
  - 含义是：系统想提醒模型，这个文件刚刚被修改过

注意，此时 attachment 还是内部的 `type: 'attachment'` 消息，不是最终 API 里的标准 user / assistant message。

### 11.2 `normalizeMessagesForAPI(...)` 之后可以近似理解成这样

经过归一化后，最终发给模型的消息结构可以近似理解成：

- `user`
  - 内容：`帮我看看 src/query.ts`
- `user`
  - 内容：
    - 一个“你刚刚读到了 `src/query.ts` 文件”的附加说明
    - 以及文件内容本身，或者等价的工具读取结果表示
- `assistant`
  - 内容：`我先阅读这个文件`
- `user`
  - 内容：
    - 一个“注意，这个文件刚刚被修改过”的附加说明
    - 以及相关 diff/snippet

也就是说，attachment 在归一化后，不再表现为“attachment 类型”，而是被展开成普通的 `user` 消息内容。

### 11.3 再举一个更贴近运行时的例子

假设某一轮里，内部消息顺序大致是：

- 用户：`帮我分析最近改过的文件`
- assistant：发起工具调用，读取文件
- user：工具结果返回，包含读取到的文件内容
- attachment：`relevant_memories`
- attachment：`diagnostics`
- attachment：`plan_mode`

那么在 `normalizeMessagesForAPI(...)` 之后，可以近似把它理解成：

- `user`
  - `帮我分析最近改过的文件`
- `assistant`
  - 一个包含工具调用信息的 assistant 消息
- `user`
  - 一个包含工具结果的 user 消息
- `user`
  - 一条 memory 补充消息
  - 内容大意是：`下面这些 memory 与当前问题相关……`
- `user`
  - 一条 diagnostics 补充消息
  - 内容大意是：`当前这些文件有诊断信息……`
- `user`
  - 一条 plan mode 提醒消息
  - 内容大意是：`当前处于 plan mode，应先分析和规划……`

你可以看到，attachment 在这里的效果就是：

- **把系统内部的附加状态**
- **转成最终模型能读懂的普通 user message**

### 11.4 为什么这一步很重要

因为模型 API 并不直接理解内部的 `attachment` 类型。

所以系统必须先把 attachment 解释成模型能消费的内容，比如：

- 一段文本提醒
- 一段 system-reminder 风格的文本
- 一个模拟的工具调用/工具结果上下文
- 一段 memory 摘要
- 一段文件 diff 或诊断内容

因此可以把 `normalizeMessagesForAPI(...)` 理解成：

> **把内部的“附加上下文对象”翻译成最终 prompt 里的普通对话内容。**

### 11.5 这一步和缓存的关系

这个示例最容易帮助理解的一点是：

- attachment 一旦完成归一化
- 它在最终结构里看起来就只是普通的 user message

所以后续从缓存视角看，系统并不会说：

- “这是 attachment，所以不算 prompt”

而是会说：

- “这是最终消息序列中的一条 user message，它位于什么位置？”

也就是说：

- **归一化之后，attachment 在缓存判断里看的是位置，不再看它原来是不是 attachment。**

## 十二、attachment 归一化后，`cache_control` 是怎么加的

前面讲的是：

- attachment 会先变成普通的 `user message`
- 然后进入最终消息序列

接下来真正和缓存有关的问题是：

> **这些已经归一化完成的消息里，哪一条会被加上 `cache_control`？加上以后会发生什么？**

### 12.1 一个最简单的例子

假设经过 `normalizeMessagesForAPI(...)` 之后，最终消息序列变成这样：

- `user`
  - `请帮我分析 src/query.ts`
- `user`
  - `这是 src/query.ts 的文件内容……`
- `assistant`
  - `我先阅读并分析这个文件`
- `user`
  - `注意：这个文件刚刚被修改过，这是相关 diff……`

如果这是一次普通请求，并且系统决定把最后一条消息作为 cache marker，那么效果可以近似理解成：

- 前三条消息保持不变
- **最后这一条 `user` 消息的最后一个 block 上，会被加一个 `cache_control`**

此时最终效果不是：

- 只有最后那个 block 被缓存

而是：

- **从请求最开始到这个带 `cache_control` 的 block 为止的所有内容，形成一个 prefix**

也就是这个例子里，等价于：

- 第一条 `user`
- 第二条 `user`
- 那条 `assistant`
- 最后一条 `user` 及其末尾 block

这一整段共同构成缓存前缀。

### 12.2 如果最后一条消息正好就是 attachment 变成的 user message

这是最容易理解的情况。

比如某一轮最后新增的是一个 attachment，原本内部含义是：

- `relevant_memories`

归一化以后，它变成一条普通 `user` 消息，例如：

- `以下 memory 与当前问题相关：...`

如果这条消息刚好是最终序列里的最后一条，那么系统很可能会：

- 把这条消息选为 cache marker 所在消息
- 再把 `cache_control` 加到这条消息的最后一个 block 上

最终效果可以理解成：

- **这条 attachment 展开的 user message，也会一起进入 prefix**

所以这里最重要的结论是：

- **attachment 一旦被归一化成 user message，就不会因为“原来是 attachment”而被特殊排除**
- **它是否进入 prefix，只看它有没有落在 marker 之前或 marker 所在消息上**

### 12.3 如果 attachment 在后面，但不是 marker 那条消息

再看另一种情况。

假设最终序列是：

- `user`：用户提问
- `assistant`：工具调用
- `user`：工具结果
- `user`：一个 diagnostics attachment 展开的消息
- `user`：一个 plan mode attachment 展开的消息

如果系统把最后一条 `user` 作为 marker，那么结果可以近似理解成：

- 前面的用户提问在 prefix 里
- assistant 工具调用在 prefix 里
- tool result 在 prefix 里
- diagnostics attachment 也在 prefix 里
- 最后一条 plan mode attachment 因为自己就是 marker 所在消息，也在 prefix 里

也就是说，这时并不是只有最前面的历史消息被缓存，
而是**到 marker 为止的整段都被算进 prefix**。

### 12.4 如果想让 attachment 更像“动态尾部”

如果某个 attachment 希望尽量不影响稳定前缀，那么更理想的位置通常是：

- 它处于消息序列靠后的地方
- 并且最好落在 marker 之后，或者发生在不会被写入共享 prefix 的那部分尾部

这也是为什么前面一直强调：

- attachment 是否“参与缓存”，主要看位置
- 而不是看它是不是 attachment

### 12.5 `cache_control` 加完以后，消息看起来会怎样

如果继续用直观方式去理解，最终结构可以把它想成这样：

- `user`
  - `请帮我分析 src/query.ts`
- `user`
  - `这是 src/query.ts 的内容……`
- `assistant`
  - `我先分析这个文件`
- `user`
  - `注意：这个文件刚被修改过……`
  - `cache_control: 这一条的最后一个 block 带上缓存边界`

这时可以把它理解成：

- 前三条是前缀的一部分
- 最后一条也仍然是前缀的一部分
- 只是**前缀的边界在它这里结束**

所以 `cache_control` 的真实效果不是：

- “只缓存这一条消息”

而是：

- “**缓存从开头到我这里为止**”

### 12.6 一个更贴近 attachment 的完整例子

假设某次请求最终整理后的消息顺序是：

- `user`
  - `帮我分析最近改过的文件`
- `assistant`
  - `我会先读取相关文件`
- `user`
  - 读取文件后的工具结果
- `user`
  - `以下 memory 与当前问题相关：...`
- `user`
  - `以下 diagnostics 与这些文件相关：...`

如果最后一条 diagnostics 消息被选中作为 marker 所在消息，那么最终效果可以理解成：

- 这五条都会进入 prefix
- diagnostics 这条消息不是“被排除的动态尾巴”
- 它反而成了“prefix 截止点所在的最后一条消息”

反过来说，如果 diagnostics attachment 出现在 marker 之后，那么它就更像：

- 当前轮额外新增的动态尾部
- 进入 prompt，但不属于已经写下来的缓存前缀

### 12.7 最终可以怎么记

关于 attachment 和 `cache_control` 的关系，最容易记住的版本是：

- **第一步**：attachment 先被翻译成普通 `user message`
- **第二步**：系统在最终消息序列里选一个边界位置
- **第三步**：在那条消息的最后一个合适 block 上加 `cache_control`
- **第四步**：从请求开头到这个 block 为止的全部内容，成为 prefix

所以 attachment 的命运并不是：

- “天然不缓存”

而是：

- “**先变成普通消息，再按最终位置决定它是前缀的一部分，还是尾部动态内容。**”

## 十三、紧贴 assistant / tool-result 后插入 attachment 时，缓存怎么用

这一节只看你真正关心的情况：

- attachment 不是丢到全局最后
- 而是贴着最近的 `assistant` 或 `tool_result` 后面插入
- 这时缓存是怎么工作的

### 13.1 一个最小示例

假设某轮整理后的最终消息顺序是：

- `user`：`帮我分析 auth.ts`
- `assistant`：`我先读取这个文件`
- `user`：工具结果，包含 `auth.ts` 内容
- `user`：attachment 展开的补充消息，内容是 `这个文件刚被修改过，diff 如下……`

这里最后一条 attachment 没有被丢到整个请求最末尾之外的远处，而是**紧贴刚才那段 assistant / tool-result 上下文**。

### 13.2 这时缓存怎么理解

如果系统把最后一条消息作为 cache marker，那么效果就是：

- 前三条在 prefix 里
- 最后一条 attachment 展开的 `user` 消息也在 prefix 里
- prefix 到这条 attachment 的最后一个 block 为止结束

也就是说，此时 attachment 不是“缓存外的尾巴”，而是：

- **贴着相关上下文插入**
- **同时作为这一轮 prefix 的末端边界**

### 13.3 为什么这种摆法有意义

它同时满足两件事：

- **语义贴近**：模型会把这条 attachment 看成“刚才那次读取/分析的补充信息”
- **缓存局部变化**：新增变化仍然集中在这一轮后段，不会去改动更早的稳定历史

所以它不是“为了缓存，把 attachment 一律扔到最后”，而是：

- **把 attachment 放在离相关上下文最近的位置**
- **然后让 cache 边界正好落在这里结束**

### 13.4 这就是当前实现想要的效果

最实用的理解方式是：

- 更早的历史消息继续作为稳定前缀的一部分复用缓存
- 当前轮新产生的 attachment 贴着本轮 assistant / tool-result 出现
- 它既保持语义连续，也只把缓存变化限制在本轮末端附近

一句话总结：

> **当前实现不是把 attachment 全塞到最后，而是让它贴着最近相关的 assistant / tool-result 停住；如果 cache marker 也落在这里，那么 attachment 就会作为“前缀末端”一起进入缓存，而不是破坏更早的稳定前缀。**
