# Claude Code 分层压缩策略说明

本文总结 `claude-code` 中围绕上下文窗口管理实现的“分层压缩策略”。这里的“压缩”不是指 gzip 这类字节压缩，而是指：**当会话上下文不断膨胀时，系统如何通过摘要、保留尾部消息、重挂关键附件和多级兜底机制，把上下文重新压回可继续工作的范围内。**

## 一、核心结论

Claude Code 的上下文压缩不是单一动作，而是一套**分层、分阶段、带兜底的上下文治理体系**。

它的目标不是简单“删旧消息”，而是：

- **优先让对话继续进行**
- **尽量保留继续工作必须的状态**
- **在压缩失败时继续降级兜底**
- **避免因为上下文过大而让整条会话卡死**

一句话概括就是：

> **先估算 token，再按阈值触发分层 compact；优先保留近期工作态和关键附件，必要时用 session memory、reactive compact 和 prompt-too-long 重试做兜底。**

## 二、整体分层结构

从主流程视角看，这套压缩体系大致可以分成五层。

| 层级 | 策略 | 主要作用 | 典型入口 |
|------|------|----------|----------|
| **第 0 层** | Token 预算与阈值判断 | 决定何时预警、何时自动压缩、何时阻断 | `src/services/compact/autoCompact.ts` |
| **第 1 层** | Microcompact | 在真正总结前先做轻量减负 | `src/commands/compact/compact.ts`、`src/services/compact/microCompact.ts` |
| **第 2 层** | Session Memory Compact | 优先复用已有会话记忆，减少重复总结 | `src/services/compact/sessionMemoryCompact.ts` |
| **第 3 层** | Traditional Compact | 调模型生成结构化摘要，并重建必要上下文 | `src/services/compact/compact.ts` |
| **第 4 层** | Reactive / PTL Retry | 当压缩请求或主请求仍然过长时继续降级兜底 | `reactiveCompact`、`truncateHeadForPTLRetry()` |

下面分别展开。

## 三、第 0 层：Token 预算与阈值判断

### 3.1 为什么先做预算

Claude Code 并不是等模型真正报错了才考虑压缩，而是先根据上下文窗口和当前消息估算 token，用“预算驱动”的方式提前决策。

核心逻辑在 `src/services/compact/autoCompact.ts`：

- 先拿到模型的上下文窗口大小
- 再预留一部分 token 给“压缩摘要这次调用本身的输出”
- 得到一个 `effectiveContextWindow`
- 再减去 `AUTOCOMPACT_BUFFER_TOKENS` 作为自动压缩阈值

也就是说，系统不是把上下文用到 100% 才开始处理，而是**提前留出压缩操作本身所需的安全空间**。

### 3.2 关键阈值分别干什么

默认值位于 `src/services/compact/autoCompact.ts`：

- **`MAX_OUTPUT_TOKENS_FOR_SUMMARY = 20_000`**：给 compaction 输出摘要预留的最大 token
- **`AUTOCOMPACT_BUFFER_TOKENS = 13_000`**：距离有效窗口还剩多少时触发自动压缩
- **`WARNING_THRESHOLD_BUFFER_TOKENS = 20_000`**：用于 UI 预警
- **`ERROR_THRESHOLD_BUFFER_TOKENS = 20_000`**：用于错误阈值判断
- **`MANUAL_COMPACT_BUFFER_TOKENS = 3_000`**：在 auto-compact 关闭时，仍为手动 `/compact` 留出空间

因此，这一层本质回答的是：

- **什么时候提醒用户上下文快满了**
- **什么时候自动 compact**
- **什么时候即使不自动 compact，也必须阻断继续请求**

### 3.3 自动压缩何时启用

自动压缩还受配置和 feature gate 控制：

- 可被 `DISABLE_COMPACT` 全局关闭
- 可被 `DISABLE_AUTO_COMPACT` 单独关闭
- 也受用户配置项 `autoCompactEnabled` 控制

这意味着 Claude Code 的策略不是“强制自动压缩”，而是：**允许 auto-compact、manual compact、reactive compact 分别受不同开关控制。**

## 四、第 1 层：Microcompact —— 先做轻量减负

### 4.1 它解决什么问题

在手动 `/compact` 时，系统不会立刻把整段历史拿去做总结，而是先运行一次 `microcompactMessages()`。

它的角色更像是“预处理”或“瘦身”：

- 先剔除一部分不必要的上下文膨胀
- 尽可能在不丢关键状态的前提下缩小待总结内容
- 降低后续正式 compaction 的输入成本和失败概率

这一步不是完整摘要，而是为真正的摘要层腾挪空间。

### 4.2 为什么要单独做这一层

如果没有这层，传统 compaction 会直接面对完整且可能非常肥大的历史：

- 压缩 API 调用更容易自己就触发 `prompt_too_long`
- 总结成本更高
- 更容易把后续要保留的 token 预算吃掉

所以 microcompact 的作用可以概括为：

> **在“总结旧历史”之前，先把上下文整理成更适合被总结的形态。**

### 4.3 它当前主要压什么内容

从 `src/services/compact/microCompact.ts` 的实现看，Microcompact 现在主要针对的是**旧的工具结果（tool_result）**，而不是普通的用户文本或 assistant 文本。

它关注的工具类型包括：

- `FileRead`
- shell / bash 类工具
- `Grep`
- `Glob`
- `WebSearch`
- `WebFetch`
- `FileEdit`
- `FileWrite`

这些内容有两个共同特点：

- **通常体积很大**，容易快速拉高上下文 token
- **随着时间推移价值衰减很快**，尤其是长输出、搜索结果和大文件读取内容

因此，Microcompact 的核心思路不是“压缩整段语义历史”，而是：

> **优先清理最贵、最机械、最容易老化的上下文块。**

### 4.4 它现在有两条执行路径

`microcompactMessages()` 现在并不是单一路径，而是按优先级分成两种实现：

1. **time-based microcompact**
2. **cached microcompact**

两者都不满足时，就直接返回原消息，不做处理。

#### 4.4.1 time-based microcompact

这是基于“距离上一条主线程 assistant 消息已经过去多久”来触发的路径。

它的设计前提是：如果间隔已经很长，那么服务端 prompt cache 基本已经过期，下一次请求大概率要把整段前缀重写一次。既然如此，不如在请求发出前，先把旧工具结果清掉，缩小这次实际要重发的 prompt。

默认配置在 `src/services/compact/timeBasedMCConfig.ts` 中：

- `gapThresholdMinutes = 60`
- `keepRecent = 5`

也就是说，超过阈值后，系统会：

- 找到所有可 compact 的工具结果
- **至少保留最近若干个**（默认最近 5 个）
- 把更早的 `tool_result.content` 直接替换为固定占位符：
  - `[Old tool result content cleared]`

这里非常关键的一点是：

- **它不删除整条消息**
- **也不删除 tool_result 这个结构块本身**
- **只是清空 tool_result 的正文内容**

这样做的好处是，模型仍然知道：

- 之前调用过某个工具
- 那个工具确实返回过结果
- 只是旧结果正文已经不再保留

这是一种“**保留调用骨架，移除高成本正文**”的策略。

#### 4.4.2 cached microcompact

另一条路径是 cached microcompact，它的目标不是直接改写本地消息，而是：

- 利用 cache editing 能力
- 在 API 层声明“哪些旧 tool result 可以从缓存前缀中删除”
- 从而**缩小本次请求上下文，同时尽量保住 prompt cache 命中**

这条路径和 time-based 最大的区别在于：

- **time-based**：直接修改本地消息内容
- **cached microcompact**：本地消息不变，只生成 `cache_edits` 给 API 层使用

它的执行思路大致是：

- 遍历消息，登记哪些 `tool_result` 属于可 compact 工具
- 按内部状态和配置判断哪些旧结果该删
- 生成 `pendingCacheEdits`
- 把这些编辑指令附着在返回的 `compactionInfo` 中，交给后续 API 调用层处理

也就是说，cached microcompact 更像是：

> **不改变本地会话视图，但在请求发送层面“逻辑删除”旧工具结果。**

### 4.5 它为什么优先做这两种事情

因为 Microcompact 的目标不是重构语义，而是**尽量低成本地回收上下文空间**。

相比于直接做一次完整 summary：

- 它更便宜
- 更快
- 更不容易失败
- 对当前会话语义的扰动更小

尤其是 cached microcompact，还额外考虑了 prompt cache：

- 不是简单粗暴地改 prompt
- 而是尽量用 cache editing 保住前缀缓存收益

所以这层更像“请求前垃圾回收”，而不是“上下文重写器”。

### 4.6 它不做什么

为了避免把 Microcompact 和完整 compact 混淆，有必要明确它**不负责**什么：

- **不生成会话摘要**
- **不重写用户意图**
- **不总结文件修改和工作进度**
- **不恢复 plan、skills、附件和 hook 上下文**

这些能力都属于后面的 `compactConversation()` 或 `session memory compact` 层。

所以从职责边界看：

- **Microcompact**：轻量缩减上下文体积
- **Traditional Compact**：重建可继续工作的上下文快照

### 4.7 用一句话概括这一层

如果把这一层再压缩成一句话，可以说：

> **Microcompact 是正式 compaction 之前的轻量瘦身层，主要回收旧 tool_result 的 token 占用；在缓存仍有价值时优先做 cache-level 删除，在缓存已冷时直接清空旧结果正文。**

## 五、第 2 层：Session Memory Compact —— 用已有会话记忆替代重复总结

### 5.1 它为什么是更高优先级

在自动压缩和手动压缩中，Claude Code 都会优先尝试 `trySessionMemoryCompaction()`。

也就是说，如果系统已经有一份可用的 session memory，它会优先把它当成“旧对话的现成摘要”，而不是每次重新把整段历史送去让模型再总结一遍。

这层的核心价值是：

- **减少重复调用模型做总结**
- **降低 token 消耗**
- **让压缩更稳定**
- **把“历史沉淀”与“当前尾部工作态”分开管理**

### 5.2 它保留什么

Session Memory Compact 并不是只留下一个 memory 文件内容，而是采取“**摘要 + 保留尾部消息**”的策略。

其默认保留配置在 `src/services/compact/sessionMemoryCompact.ts`：

- **`minTokens = 10_000`**：压缩后至少保留这么多 token 的近期内容
- **`minTextBlockMessages = 5`**：至少保留一定数量的文本消息
- **`maxTokens = 40_000`**：保留尾部的硬上限

也就是说，这层策略不是把整个对话粗暴替换成一条 summary，而是：

- 用 session memory 代表更早的历史
- 同时保留一段最近的原始消息尾部
- 避免模型只看到抽象总结而丢失最近几轮的执行上下文

### 5.3 它适合解决什么问题

如果一个会话已经运行很久，并且系统持续提取过 session memory，那么这层通常是最划算的选择：

- 历史部分已经被外部化到 memory 中
- 当前只需要保住最近的工作尾部
- 不必每次都重新“总结历史”

可以把这一层理解为：

> **把早期会话历史沉淀为长期摘要，把最近的执行态继续留在当前上下文里。**

## 六、第 3 层：Traditional Compact —— 生成结构化摘要并重建工作上下文

### 6.1 这是最核心的一层

当 Session Memory Compact 不可用时，系统会进入传统的 `compactConversation()` 路径。这一层是最完整、最通用的压缩实现。

它做的不是“删消息”，而是一套完整的上下文重建流程：

1. 组织要送去总结的消息
2. 剥离对摘要无帮助但很贵的内容
3. 调模型生成结构化摘要
4. 在消息流中插入 compact boundary
5. 重新附加文件、plan、skills、工具状态、hook 恢复上下文

### 6.2 它如何处理原消息

这层 compaction 生成的结果结构是：

- `boundaryMarker`
- `summaryMessages`
- `messagesToKeep`（如果有）
- `attachments`
- `hookResults`

这说明 compact 后的新上下文并不是只有 summary，而是一个**可继续工作的最小工作集**。

### 6.3 它会先清理哪些“高成本但低价值”的内容

在真正总结前，传统 compaction 会先做两类清理。

#### 6.3.1 剥离图片与文档

图片和文档对“生成会话摘要”帮助有限，但非常耗 token，甚至会导致 compaction 这次调用本身超长。因此系统会把它们替换成 `[image]`、`[document]` 这类标记，仅保留“这里曾经有媒体”的事实。

#### 6.3.2 剥离之后会被重新注入的 attachment

例如某些技能发现、技能列表类 attachment，本来就会在 post-compact 阶段根据当前状态重新公告，因此没必要再送进 summarizer 增加负担。

这说明 Claude Code 的思路不是“原样压缩”，而是：

> **先把上下文改写成适合总结的版本，再做总结。**

### 6.4 摘要本身长什么样

Compact 并不是自由发挥式总结，而是通过专门 prompt 让模型输出结构化结果。

`src/services/compact/prompt.ts` 的设计体现了两个特征：

- 先生成一个 `<analysis>` 草稿区
- 再生成一个 `<summary>` 结果区

最终系统会把 `<analysis>` 去掉，只把整理后的 summary 作为后续上下文的一部分。这么做的目的是：

- 允许模型在摘要时有更好的中间思考质量
- 但不把草稿垃圾继续带进后续上下文

换句话说，这里压缩的不是“原消息文本”，而是**将对话重写为面向继续工作的结构化知识块**。

### 6.5 压缩后会恢复哪些关键状态

这是 Claude Code 压缩体系最重要的设计之一：**压缩会吃掉消息历史，因此必须主动补回工作继续所需的状态。**

主要包括以下几类。

#### 6.5.1 最近读过的文件

系统会根据 `readFileState` 重新附加最近读过的文件内容，用于避免模型压缩后立刻又不得不重新读文件。

这里有明确预算：

- 最多恢复 **5 个文件**
- 总预算 **50,000 tokens**
- 每个文件最多 **5,000 tokens**

它体现的是一种非常典型的“工作态恢复”思路：

- 不是恢复所有历史读过的文件
- 而是按最近访问优先
- 且受预算约束

#### 6.5.2 Plan 文件

如果当前 session 有 plan 文件，系统会把它作为 `plan_file_reference` attachment 重新挂回去。这样 compact 后模型仍然知道当前计划是什么，而不必从 summary 里重新提炼计划结构。

#### 6.5.3 已调用过的 skills

如果本 session 中启用了某些 skill，系统会把 skill 内容按“最近使用优先”的方式截断后重新注入：

- 每个 skill 最多 **5,000 tokens**
- 总 skill 预算 **25,000 tokens**

它的设计重点不是完整恢复 skill 文档，而是优先保留 skill 文件开头那部分最关键的使用说明。

#### 6.5.4 Deferred tools、Agent 列表、MCP 指令

这些状态在压缩前可能以 delta attachment 形式存在。compaction 后，系统会根据当前真实状态重新生成并附加，确保模型在下一轮还能知道：

- 当前有哪些 deferred tools 可用
- 当前有哪些 agent 类型可用
- 当前 MCP server 给出了哪些 instruction

#### 6.5.5 SessionStart / PostCompact Hook 恢复上下文

压缩成功后，还会重新运行相关 hook，把某些会话启动时需要的上下文再补回来，例如 CLAUDE.md 一类通过 hook 注入的上下文。

### 6.6 这一层的本质是什么

Traditional Compact 可以概括为：

> **把“对话历史”压缩成“可继续工作的上下文快照”。**

它并不追求完整保真，而是追求“恢复后还能接着做事”。

## 七、第 4 层：Reactive Compact 与 PTL Retry —— 当主动压缩不够时的兜底

### 7.1 Reactive Compact 解决什么问题

有些时候，主动阈值判断并不能完全兜住所有情况。比如：

- 某些 token 估算偏差
- 系统 prompt / tools / userContext 的真实占用比预估大
- 某轮请求临时膨胀

这时主请求可能会直接触发 `prompt_too_long`。为了解决这类情况，系统提供 reactive compact：**等 API 真报上下文过长后，再进入补救压缩路径。**

它的定位是“反应式兜底”，不是主路径，但在异常场景里非常重要。

### 7.2 如果连 compact 请求自己都太长怎么办

更极端的情况是：不是主请求过长，而是“拿去做 compact 的那批消息”本身就太长。此时 `compactConversation()` 内会走 `truncateHeadForPTLRetry()`。

这一步会：

- 先按 API round 给消息分组
- 估算离可接受长度还差多少 token
- 从最老的分组开始丢弃
- 再重试 compact 请求

如果拿不到精确 token gap，就退化成“丢掉最老的约 20% 分组”。

这一步是明显带损的，但它的目标非常明确：

> **宁可损失更老的历史，也不要让用户彻底卡死在无法继续的会话里。**

## 八、Partial Compact：不是只支持整段压缩

除了整段 compact，系统还支持 `partialCompactConversation()`，也就是围绕一个 pivot 对会话的一部分做压缩。

它支持两个方向：

- **`from`**：保留较早部分，压缩较晚部分
- **`up_to`**：压缩较早部分，保留较晚部分

这里的重点在于：不同方向对 prompt cache 和保留段的影响不同。

- `from` 更容易保住较早部分的 prompt cache
- `up_to` 因为摘要会插到保留段前面，更容易让 cache 失效

也就是说，Claude Code 的压缩体系不仅区分“是否压缩”，还区分：

- **压哪一段**
- **压完后 cache 是否还能复用**
- **压完后保留链路如何 relink**

这说明它已经不是一个简单的“摘要开关”，而是一个**面向上下文生命周期管理的精细化系统**。

## 九、失败保护：连续失败的熔断

自动压缩失败时，系统不会无限次在每轮都重试。

`src/services/compact/autoCompact.ts` 中设定了：

- **`MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3`**

连续失败达到上限后，auto-compact 会在该 session 中停止继续尝试，避免：

- 不断发送注定失败的 compact 请求
- 浪费大量 API 调用
- 让系统进入无意义的失败循环

这层机制的作用不是“让压缩更成功”，而是：

> **在已知无法成功时，及时止损。**

## 十、这套分层压缩策略到底在解决什么问题

如果抽象一点看，Claude Code 的分层压缩策略主要在解决四个问题。

### 10.1 避免上下文突然爆掉

通过 token 阈值和 auto-compact，系统尽量在“即将爆掉”之前处理，而不是等 API 硬报错。

### 10.2 避免压缩后丢失工作态

通过保留尾部消息、恢复最近文件、plan、skills 和工具状态，系统保证 compact 后仍能继续工作，而不是只剩一段抽象总结。

### 10.3 避免把每次压缩都做成昂贵的全量重总结

通过 session memory compact，把旧历史提前外部化为 memory，减少重复总结成本。

### 10.4 避免异常情况让会话彻底卡死

通过 reactive compact、PTL retry 和连续失败熔断，系统在极端场景下仍尽量保留“可继续性”。

## 十一、最终总结

Claude Code 的压缩策略不是“压缩旧消息”这么简单，而是一套**分层的上下文续航机制**。

从设计意图看，它的优先级大致是：

1. **先用 token 预算提前判断是否该介入**
2. **能轻量减负就先 microcompact**
3. **有 session memory 就优先复用**
4. **否则走完整的 traditional compact**
5. **如果仍然过长，就 reactive compact / PTL retry**
6. **如果连续失败，就熔断止损**

所以，最准确的理解不是“Claude Code 有一个压缩功能”，而是：

> **Claude Code 实现了一套围绕上下文窗口、工作态恢复、记忆外部化和异常兜底展开的分层压缩体系。**
