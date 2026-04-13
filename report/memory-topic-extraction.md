# Claude Code 不同主题记忆的提取机制

本文聚焦回答一个问题：**Claude Code 中不同主题的记忆，是如何从对话里被提取出来，并最终汇总到单独 Markdown 文件中的？**

## 一、先说结论

Claude Code 对“不同主题记忆”的处理，不是把整段对话原样保存，而是走一条**提取 -> 归类 -> 落盘 -> 索引**的链路：

1. 在安全时机触发记忆提取
2. 由后台 Agent 从当前上下文中筛选出值得长期保留的信息
3. 将这些信息归入某个记忆主题类型
4. 把每个主题写入单独的 Markdown 文件
5. 再把这些主题文件登记到 `MEMORY.md` 里，形成统一入口

因此，“不同主题的记忆”最终不是混在一个总文件里，而是：

- **每个主题一个独立 Markdown 文件**
- **`MEMORY.md` 负责做索引入口**

## 二、记忆是从什么时候开始被提取的

当前主要有两条提取链路。

### 2.1 Auto Memory：对话结束后自动提取

**代码位置**：`src/services/extractMemories/extractMemories.ts`

Auto Memory 会在一轮查询结束时触发，典型条件是：

- 模型已经产出最终响应
- 当前轮没有继续发起工具调用
- 处于适合做后台提炼的安全点

这时候系统会启动一个 **forked agent**，从当前主 Agent 的上下文里提取可长期复用的信息。

它的特点是：

- 继承当前上下文
- 共享提示缓存
- 不阻塞主对话
- 提取结果可直接回写到记忆系统

也就是说，真正做“总结主题记忆”的，不是主对话线程本身，而是这个后台提取 Agent。

### 2.2 Session Memory：长会话达到阈值后提取

**代码位置**：`src/services/SessionMemory/sessionMemory.ts`

除了每轮结束时的 Auto Memory，系统还会在长会话中按阈值提取 Session Memory。其典型门槛包括：

- 消息 token 数达到初始化阈值
- 与上次提取相比，token 数继续增长到阈值
- 工具调用次数达到阈值
- 当前处于无工具调用的安全提取点

这条链路更偏向“把长会话沉淀成阶段性总结”，本质上也是为后续主题化记忆提供原料。

## 三、从当前会话提取记忆的具体策略流程

如果只看“当前会话里的对话记录是怎么被提取成 memory 的”，主链路其实集中在 `src/services/extractMemories/extractMemories.ts` 和 `src/services/extractMemories/prompts.ts`。

下面按运行顺序拆开说明。

### 3.1 触发时机：只在一轮对话完整结束后触发

记忆提取不是每来一条消息就立刻执行，而是在一轮 query loop 完成后触发。也就是：

- 模型已经给出最终响应
- 当前轮没有继续工具调用
- stop hook 开始执行后台逻辑

这意味着系统是按“**一段已完成的对话结果**”来提取记忆，而不是边说边抽取。

### 3.2 输入范围：只取上次提取之后新增的 user / assistant 消息

系统内部维护了一个游标：`lastMemoryMessageUuid`。

每次提取时，不会重新扫描整段历史，而是调用 `countModelVisibleMessagesSince(messages, lastMemoryMessageUuid)`，只统计这之后新增的 **model-visible messages**。

这里所谓的 model-visible message，只包括：

- `user`
- `assistant`

不包括：

- progress message
- system message
- attachment message

所以，真正作为“提取原料”的，是：

> **上次记忆提取之后，到当前这一轮结束为止，新增的 user / assistant 对话内容。**

### 3.3 互斥判断：如果主 Agent 已经写过 memory，就不再重复提取

在启动后台提取前，系统会先调用 `hasMemoryWritesSince(messages, lastMemoryMessageUuid)` 检查：

- 从上次游标之后
- 是否已经有 assistant message 通过 `Write/Edit` 写入了 auto-memory 路径

如果已经写过，就直接跳过本次 forked extraction，并把游标推进到最后一条消息。

这个策略的目的是：

- 避免主 Agent 和后台 subagent 对同一段会话重复写 memory
- 保证一段对话只被一条记忆链路消费一次

### 3.4 节流与串行：不是每个可提取点都一定执行

即使到了可提取时机，也不一定立刻跑 forked agent。

系统还会做两层控制：

1. **节流控制**
   - 用 `turnsSinceLastExtraction` 统计自上次提取以来经过了多少个 eligible turn
   - 再配合 feature gate `tengu_bramble_lintel` 决定是否这次真的执行

2. **串行控制**
   - 如果当前已有提取在进行，新的上下文不会并发启动第二个提取
   - 而是先存入 `pendingContext`
   - 等当前提取结束后，再补跑一次 trailing extraction

这保证了：

- 不会多个 extraction subagent 同时抢同一份上下文
- 也不会因为连续多轮响应而漏掉后续新增消息

### 3.5 提取前准备：先给子 Agent 一份现有记忆清单

当系统决定真的执行提取时，会先扫描当前 memory 目录：

1. `scanMemoryFiles(memoryDir, signal)` 读取现有 `.md` 记忆文件头信息
2. `formatMemoryManifest(...)` 把这些文件整理成一份清单字符串

这份 manifest 会被直接注入 extraction prompt，目的是告诉子 Agent：

- 现在已经有哪些 memory 文件
- 优先更新已有文件
- 不要无脑新建重复记忆

因此，提取不是“每次都从零总结”，而是“**基于新增对话，对现有记忆做增量更新**”。

### 3.6 Prompt 构造：明确要求只分析最近新增消息

随后系统会构造 extraction prompt：

- auto-only 模式：`buildExtractAutoOnlyPrompt(newMessageCount, existingMemories, skipIndex)`
- combined 模式：`buildExtractCombinedPrompt(newMessageCount, existingMemories, skipIndex)`

其中最关键的限制语句在 `opener(...)` 里：

1. 你现在是 **memory extraction subagent**
2. 分析最近 `~newMessageCount` 条消息
3. **只能使用这最近一段消息来更新 persistent memories**
4. **不要再去 grep 源码、读代码、跑 git 做验证**

这一步非常关键，因为它定义了提取策略不是：

- “读完当前仓库后判断哪些适合存 memory”

而是：

- “先锁定当前会话中新增的对话片段，再只根据这段对话做记忆提取”

### 3.7 真正执行提取：用 forked agent 在当前会话上下文上做语义筛选

系统接着调用 `runForkedAgent(...)` 启动分叉子 Agent。

这里的关键点是：

- 子 Agent 继承主对话的当前上下文
- 共享父级 prompt cache
- 但额外收到一条“你现在负责提取 memory”的 user prompt

于是它会在当前对话上下文上执行以下动作：

1. 看最近新增的 user / assistant 消息
2. 判断哪些内容值得长期保留
3. 过滤掉不该保存的内容
4. 再把剩余内容分类为：
   - `user`
   - `feedback`
   - `project`
   - `reference`
5. 决定：
   - 更新已有主题文件
   - 或创建新的主题文件

这里真正的“提取”不是代码写死的 if/else，而是：

> **在受限 prompt 下，由大模型对最近新增对话做语义筛选、语义分类和结构化改写。**

### 3.8 工具限制：子 Agent 只能围绕 memory 目录行动

为防止子 Agent 跑偏，`createAutoMemCanUseTool(memoryDir)` 对它做了严格限制。

允许的主要只有：

- `Read`
- `Grep`
- `Glob`
- 只读 `Bash`
- `Edit/Write`，但仅限 memory 目录内

这意味着它的能力边界是：

- 可以读已有 memory 文件
- 可以更新 memory 文件
- 不能随便改业务代码
- 不能跑有副作用的 shell 命令
- 不能离开 memory 目录做大范围调查

所以它对当前会话的处理，本质上是一个**受限的增量整理器**，而不是全仓库研究 agent。

### 3.9 落盘策略：主题文件才是记忆本体，`MEMORY.md` 只是索引

子 Agent 真正写入时，采用两步：

1. 写主题文件
2. 更新 `MEMORY.md`

但在主流程里，系统后续统计“memory saved”时，会把 `MEMORY.md` 排除掉，只把主题文件当成真正保存的记忆。

也就是说：

- `MEMORY.md` 的变更只是索引维护
- 真正沉淀下来的记忆，是那些 `user_xxx.md`、`feedback_xxx.md`、`project_xxx.md`、`reference_xxx.md` 之类的主题文件

### 3.10 提取完成后：推进游标，确保下次只处理更新增量

一旦 forked extraction 成功完成，系统会：

1. 取当前 `messages` 的最后一条消息 UUID
2. 写回到 `lastMemoryMessageUuid`
3. 收集本次写入了哪些文件

这样下一次提取时，只会考虑这之后新增的对话，不会重复消费已经处理过的历史消息。

如果这次提取报错，则不会推进游标。这样下次还有机会重新处理这段消息，避免漏提。

### 3.11 把整条链路压缩成一句策略描述

如果只保留最核心的策略，可以表述为：

> **系统在每轮主对话结束后，以“上次提取之后新增的 user / assistant 消息”为唯一原料，先检查是否已被主 Agent 直接写入 memory；若没有，则构造一个只允许关注最近新增消息的 extraction prompt，启动 forked subagent 对这段对话做语义筛选、分类和增量更新，并把结果落到主题 memory 文件与 `MEMORY.md` 索引中。**

## 四、不同主题是怎么区分出来的

**代码位置**：`src/memdir/memoryTypes.ts`

系统在语义层面把记忆分成四种核心类型：

| 类型 | 含义 | 常见内容 |
|------|------|----------|
| `user` | 用户信息 | 用户角色、目标、偏好、习惯 |
| `feedback` | 用户反馈 | 用户纠正、确认过的行为规则 |
| `project` | 项目上下文 | 项目目标、决策、工作动态 |
| `reference` | 外部参考 | 外部系统链接、资料入口 |

这意味着“主题提取”本质上分成两步：

1. 先判断这段信息值不值得保留
2. 再判断它属于哪一种主题类型

例如：

- “用户更喜欢简洁回答”更可能归入 `user`
- “用户要求不要使用 emoji”更可能归入 `feedback`
- “这个仓库采用 pnpm workspace”更可能归入 `project`
- “这个项目的监控面板在某个 Grafana 链接”更可能归入 `reference`

## 五、不同主题为什么会被拆成单独 Markdown 文件

Claude Code 采用的是**索引 + 内容分离**的存储模型。

典型结构如下：

```text
memory/
├── MEMORY.md
├── user_role.md
├── feedback_testing.md
├── project_deadlines.md
└── reference_linear.md
```

这里的设计意图很明确：

- `MEMORY.md` 只负责做总入口
- 每个主题文件只保存某一类相对聚焦的内容

这样做的好处是：

1. 不同主题互不干扰，便于维护
2. 后续做相关性筛选时，可以按文件粒度选择
3. 便于对单个主题做追加、整合或重写
4. 比“一个超大总文件”更适合长期演进

所以你可以把“总结到一个单独的 markdown 文件里”理解成：

- 不是把所有主题总结到一个大文件
- 而是把**每一个主题总结到它自己的单独 Markdown 文件**里

## 六、主题记忆是如何写入单独文件的

记忆写入采用两步：

1. 先写主题文件
2. 再更新 `MEMORY.md` 索引

对应描述见主文档中的写入流程说明。

主题文件的格式通常是：

```markdown
---
name: 用户角色与目标
type: user
description: 记录用户的基本信息和主要目标
---

## 角色
- 全栈开发工程师

## 目标
- 提高代码质量
```

其中：

- `name`：这个主题记忆的人类可读名称
- `type`：主题类型，例如 `user` / `feedback` / `project` / `reference`
- `description`：该主题文件记录的摘要
- 正文：这个主题下真正沉淀下来的结构化内容

因此，所谓“提取成单独 Markdown 文件”，其实就是把提炼后的主题内容落成这种带 frontmatter 的主题文件。

## 七、一个主题文件里的内容是如何形成的

从文档当前信息可以推断，形成过程大致是：

1. 后台 Agent 读取当前轮对话或长会话上下文
2. 从中找出值得长期复用的事实、偏好、规则、项目状态
3. 判断这些内容属于哪个主题类型
4. 生成适合长期保存的表述，而不是原始聊天记录
5. 写入对应主题文件，并更新 `MEMORY.md`

这说明主题文件保存的是：

- **提炼后的结论**
- **结构化后的知识**
- **适合复用的长期信息**

而不是：

- 原始逐轮聊天记录
- 临时推理草稿
- 未经整理的即时上下文

## 八、Assistant 模式下有什么不同

在 Assistant 模式下，系统不会优先把内容直接写成主题文件，而是先写入按日期组织的日志：

```text
<autoMemPath>/logs/YYYY/MM/YYYY-MM-DD.md
```

这意味着它多了一层中间态：

1. 先把运行中的观察追加到日志
2. 再由 `/dream` 做后续提炼
3. 最终生成主题文件和 `MEMORY.md`

也就是说，在 Assistant 模式里：

- **日志是原始材料**
- **主题 Markdown 文件是提炼后的长期记忆结果**

## 九、最终 prompt 是如何拼装出来的

如果要理解“模型为什么能区分 `user / feedback / project / reference`”，只看 `memoryTypes.ts` 里的某一个变量还不够，更准确的做法是看**最终 prompt 是怎样由多个片段拼接出来的**。

### 9.1 主 Agent 的 memory prompt 是怎么拼的

主 Agent 使用的 memory prompt 入口在：`src/memdir/memdir.ts` 的 `buildMemoryLines(...)`。

它会把几类内容依次拼成一个完整 prompt：

1. **基础说明**
   - 告诉模型 memory 目录在哪里
   - 告诉模型目录已存在，直接写即可
   - 告诉模型要长期建设这套记忆系统

2. **显式记忆/遗忘指令**
   - 如果用户明确要求 remember，就立即保存
   - 如果用户要求 forget，就找到并删除对应记忆

3. **记忆类型分类法**
   - 来自 `src/memdir/memoryTypes.ts`
   - individual 模式使用 `TYPES_SECTION_INDIVIDUAL`
   - combined 模式使用 `TYPES_SECTION_COMBINED`

4. **哪些内容不要保存**
   - 来自 `WHAT_NOT_TO_SAVE_SECTION`

5. **如何写入记忆文件**
   - 说明是两步写入：主题文件 + `MEMORY.md` 索引
   - frontmatter 格式来自 `MEMORY_FRONTMATTER_EXAMPLE`

6. **什么时候该访问 memory**
   - individual 模式通过 `WHEN_TO_ACCESS_SECTION`
   - combined 模式直接内联拼接 `MEMORY_DRIFT_CAVEAT`

7. **如何信任和验证回忆**
   - 来自 `TRUSTING_RECALL_SECTION`

8. **memory 与 plan / tasks 的边界**
   - 告诉模型什么应该写进 memory
   - 什么应该留在 plan 或任务系统里

9. **搜索历史上下文的补充说明**
   - 由 `buildSearchingPastContextSection(...)` 追加

也就是说，主 Agent 最终看到的不是单一 prompt，而是：

```text
基础说明
+ 显式 remember/forget 规则
+ 类型分类法
+ 不该存什么
+ 如何保存
+ 何时访问 memory
+ 如何验证 recalled memory
+ memory 与其他持久化手段的边界
+ 搜索过去上下文的补充说明
```

### 9.2 `memoryTypes.ts` 里哪些变量真正参与了拼装

在主 Agent 的 memory prompt 中，`memoryTypes.ts` 里最关键的变量有：

| 变量名 | 作用 |
|--------|------|
| `TYPES_SECTION_INDIVIDUAL` | 单目录模式下的记忆分类法 |
| `TYPES_SECTION_COMBINED` | private/team 双目录模式下的记忆分类法 |
| `WHAT_NOT_TO_SAVE_SECTION` | 定义不应保存为 memory 的内容 |
| `WHEN_TO_ACCESS_SECTION` | 规定什么时候应该读取 memory |
| `MEMORY_DRIFT_CAVEAT` | 强调 memory 可能过时，需要验证 |
| `TRUSTING_RECALL_SECTION` | 规定如何使用和校验 recalled memory |
| `MEMORY_FRONTMATTER_EXAMPLE` | 规定记忆文件 frontmatter 格式 |

其中最核心的“分类”部分，确实主要由：

- `TYPES_SECTION_INDIVIDUAL`
- `TYPES_SECTION_COMBINED`

来定义。

### 9.3 是否还会拼其他文件的变量

会。完整 prompt 不只来自 `memoryTypes.ts`。

#### 来自 `src/memdir/memdir.ts`

- `ENTRYPOINT_NAME`
- `MAX_ENTRYPOINT_LINES`
- `DIR_EXISTS_GUIDANCE`
- `buildMemoryLines(...)` 内部写死的说明文字

它们决定了：

- `MEMORY.md` 是索引，不是内容本体
- 索引有长度限制
- 目录已经存在，不要先 `mkdir`
- memory 与其他持久化机制的职责边界

#### 来自 `src/memdir/teamMemPrompts.ts`

当启用 team memory 时，系统走 `buildCombinedMemoryPrompt(...)`，这时会额外加入：

- private / team 双目录说明
- `## Memory scope` 段落
- 团队记忆不要保存敏感信息的约束

也就是说，combined 模式下不只是“多了一个 scope 字段”，而是整个 prompt 会换成另一套拼装逻辑。

### 9.4 后台 extraction subagent 的 prompt 又是另一套拼装

后台提取 Agent 使用的不是主 Agent 那套完整 memory prompt，而是 `src/services/extractMemories/prompts.ts` 里的提取专用 prompt。

它也会复用 `memoryTypes.ts` 里的核心变量，但拼装目标不同。

提取 prompt 一般会包含：

1. **opener**
   - 告诉模型：你现在是 memory extraction subagent
   - 只能分析最近 N 条消息
   - 不要额外去读代码、跑 git、做验证

2. **工具约束**
   - 可用哪些读工具
   - 只能在 memory 目录内写入
   - 写能力有限制

3. **existing memories manifest**
   - 如果已有记忆文件，会把列表带进去
   - 目的是避免重复写新文件

4. **显式 remember/forget 指令**

5. **记忆类型分类法**
   - auto-only 用 `TYPES_SECTION_INDIVIDUAL`
   - combined 用 `TYPES_SECTION_COMBINED`

6. **哪些内容不要保存**
   - `WHAT_NOT_TO_SAVE_SECTION`

7. **如何保存**
   - `MEMORY_FRONTMATTER_EXAMPLE`
   - 以及主题文件 + `MEMORY.md` 的写法说明

和主 Agent 不同的是，extraction subagent 的 prompt 更强调：

- 只基于最近消息提取
- 不要跑出 memory 目录做额外调查
- 先检查已有记忆，再决定更新还是新建

### 9.5 可以把 prompt 拼装理解成两层

如果把整个机制抽象一下，可以理解为两层：

#### 第一层：memory taxonomy

这层主要来自 `memoryTypes.ts`，回答的是：

- 有哪些类型
- 每种类型代表什么
- 什么该存、什么不该存
- 应该怎么使用、怎么写

#### 第二层：runtime prompt assembly

这层来自 `memdir.ts`、`teamMemPrompts.ts`、`extractMemories/prompts.ts`，回答的是：

- 当前是主 Agent 还是 extraction subagent
- 当前是单目录还是 private/team 双目录
- 应该如何写入文件
- 是否需要索引
- 工具权限是什么
- 是否允许额外调查

也就是说：

- `memoryTypes.ts` 决定**分类语义**
- 其他文件决定**运行时装配方式**

### 9.6 一句话总结

最终的 memory prompt 并不是单独由 `TYPES_SECTION_INDIVIDUAL` 或 `TYPES_SECTION_COMBINED` 构成，而是由 **记忆类型定义 + 不可保存边界 + 保存格式 + 访问/校验规则 + 目录与索引约束 + 子 Agent 工具限制** 共同拼装出来的；其中 `memoryTypes.ts` 主要负责“分类法”，`memdir.ts` / `teamMemPrompts.ts` / `extractMemories/prompts.ts` 负责“最终装配”。

## 十、可以如何理解“不同主题的提取”

如果把整个过程说得更直白一点，可以理解成：

- 系统不是记住“你说过的每一句话”
- 而是在合适时机判断“哪些信息值得长期记住”
- 然后再判断“这些信息分别属于用户、反馈、项目还是参考”
- 最后把它们分别写进不同的 Markdown 主题文件里

## 十一、一句话总结

Claude Code 对不同主题记忆的提取，本质上是：**在安全时机由后台 Agent 从对话中提炼长期有价值的信息，按 `user / feedback / project / reference` 等主题分类，并分别写入独立 Markdown 文件，再通过 `MEMORY.md` 建立统一索引。**

## 相关文档

- [记忆系统架构总览](./memory-system-architecture.md)
