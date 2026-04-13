# Assistant Mode 下 Auto Memory 行为总结

本文总结 `Assistant Mode`（代码中以 `KAIROS` / `kairosActive` 为核心分支）下，`auto memory` 的行为方式、控制链路与和普通模式的差异。

## 一、核心结论

在 `Assistant Mode` 下，Auto Memory 的核心策略不是“实时维护 `MEMORY.md`”，而是：

- **通过 system prompt 定义记忆写入规则**
- **由主 Agent 在会话过程中自行调用通用文件工具落盘**
- **把新记忆先追加到按天组织的 daily log**
- **后续再由 `/dream` 流程提炼成 topic files 和 `MEMORY.md`**

一句话概括就是：

> **策略靠 system prompt，落盘靠文件工具，日期切换靠上下文，索引汇总靠 `/dream`。**

## 二、和普通 Auto Memory 的主要区别

### 2.1 普通模式

普通 Auto Memory 更偏向“**一轮 query loop 结束后**再提取”。它会在主响应完成、安全点到达后，执行记忆提取，然后直接写主题文件并更新 `MEMORY.md`。

### 2.2 Assistant Mode

Assistant Mode 认为会话可能是长期、持续的，因此不再把 `MEMORY.md` 当作实时可写索引，而是改为：

- 在工作过程中持续记录
- 先写 daily log
- 保持日志为 append-only
- 由后续夜间整理过程做蒸馏与索引维护

也就是说，Assistant Mode 下更像“**先采集原始增量，再异步整理**”。

## 三、谁在控制这条流程

### 3.1 核心控制层：system prompt

这条流程的主控制器是 memory prompt，也就是 system prompt 中注入的 Auto Memory 规则。

在 `src/memdir/memdir.ts` 中，Assistant Mode 会切换到专门的 daily-log prompt：

```ts
if (feature('KAIROS') && autoEnabled && getKairosActive()) {
  return buildAssistantDailyLogPrompt(skipIndex)
}
```

这意味着：

- 开启 `Assistant Mode`
- 命中 `KAIROS + autoEnabled + kairosActive`
- 系统不再给模型普通 memory prompt
- 而是给它一份“如何写 daily log”的专用规则

对应源码位置：`src/memdir/memdir.ts`

### 3.2 执行层：主 Agent + 通用文件工具

代码里没有看到一个专门的 `appendDailyLog()` 或“自动后台记日志器”。

实际执行方式是：

- 主 Agent 读到 system prompt 中的规则
- 自己判断当前是否出现“值得长期记住”的信息
- 然后调用通用文件工具（如 `Write` / `Edit`）写入 daily log

因此，这套设计更像是：

- **规则驱动**，不是单独的后台自动写盘器驱动
- **模型自主执行**，不是框架静默替模型写盘

## 四、模型怎么知道 today’s log 在哪

模型不是自己猜 today’s log 的位置，而是系统把**路径模板**和**当天日期**都提供给它。

### 4.1 路径模板来自 memory prompt

在 `buildAssistantDailyLogPrompt()` 中，系统会构造 daily log 的路径模式：

```ts
const logPathPattern = join(memoryDir, 'logs', 'YYYY', 'MM', 'YYYY-MM-DD.md')
```

然后把这条规则直接写进 prompt，告诉模型：

- today’s log 的路径模式是什么
- 用 `currentDate` 替换 `YYYY-MM-DD`
- 跨天时切到新的一天文件

这说明 today’s log 的位置是**显式告知**模型的，不是让它自由推测。

### 4.2 日期来自上下文

`src/context.ts` 会把当前日期注入上下文：

```ts
currentDate: `Today's date is ${getLocalISODate()}.`
```

此外，源码注释还说明跨午夜时会通过 `date_change` attachment 告诉模型日期已变化，因此模型知道什么时候要从旧文件切到新文件。

### 4.3 最终解析逻辑

因此，模型知道 today’s log 的过程是：

- 系统先确定 `memoryDir`
- prompt 提供 `logs/YYYY/MM/YYYY-MM-DD.md` 的路径模式
- 上下文提供 `currentDate`
- 模型将日期代入模板，得到当天具体文件路径

## 五、具体写入策略是什么

Assistant Mode 下，daily log 的策略可以拆成六条：

### 5.1 记什么

根据 `src/memdir/memdir.ts` 中的 `What to log` 段落，重点包括：

- 用户纠正和偏好
- 关于用户角色、目标的事实
- 代码中无法直接推导出的项目上下文
- 外部系统指针（如 dashboard、项目链接、沟通渠道）
- 用户显式要求记住的事项

### 5.2 记到哪

写入 daily log，而不是直接改 `MEMORY.md`：

- 路径形态：`<autoMemPath>/logs/YYYY/MM/YYYY-MM-DD.md`

对应源码位置：`src/memdir/paths.ts`

### 5.3 什么时候记

关键规则是：

> **As you work**

也就是：

- 在会话进行过程中记录
- 遇到值得长期保存的信息就记录
- 不是必须等到整轮结束
- 更不是等整场会话结束后再统一回写

### 5.4 怎么记

写法也被 prompt 规定了：

- 每条写成**简短、带时间戳的 bullet**
- 文件不存在时首次创建
- 父目录不存在时一并创建

### 5.5 不要怎么记

Prompt 同时给出了明确限制：

- **不要重写旧日志**
- **不要重新组织日志内容**
- **日志必须保持 append-only**
- **不要直接编辑 `MEMORY.md`**

这体现了它对 daily log 的定位：它是原始增量记录流，而不是整理后的知识库入口。

### 5.6 后续谁来整理

后续由单独的 `/dream` 流程消费这些 append-only logs，并把它们蒸馏成：

- topic memory files
- `MEMORY.md`

所以 `MEMORY.md` 在 Assistant Mode 下是“提炼后的索引”，不是“实时写入入口”。

## 六、`MEMORY.md` 在 Assistant Mode 里的角色

在 Assistant Mode 下，`MEMORY.md` 并没有消失，但角色发生了变化：

- **它仍会被加载进上下文**，供模型快速定向
- **它是 distilled index**，也就是“提炼后的目录索引”
- **它不是新记忆的第一落点**

这正是 daily log 设计成立的关键原因：

- 长会话里新信息不断出现
- 如果每次都实时维护 `MEMORY.md`，索引会迅速膨胀并失去“轻量入口”作用
- 因此先写原始日志，再夜间统一提炼，会更稳定也更可维护

## 七、辅助机制分别负责什么

这套行为不是只有一个开关，而是几层能力配合：

### 7.1 system prompt

负责定义 policy：

- 该记什么
- 写到哪里
- 如何组织
- 不该怎么写

### 7.2 文件工具

负责真正的落盘执行：

- `Write` 用于创建新文件或整文件写入
- `Edit` 更适合修改已有日志

### 7.3 日期上下文

负责决定“今天到底是哪一天”：

- `currentDate`
- `date_change`

### 7.4 `/dream`

负责后续 consolidation：

- 读取 daily logs
- 提炼长期记忆
- 更新主题文件与 `MEMORY.md`

## 八、行为时序图（概念）

可以把 Assistant Mode 下的 Auto Memory 理解成以下时序：

1. 启用 `Assistant Mode`
2. 系统注入 assistant daily-log memory prompt
3. 主 Agent 在工作过程中发现值得记住的信息
4. 根据 prompt 规则计算 today’s log 路径
5. 调用通用文件工具，把新条目 append 到 daily log
6. 会话持续进行，日志持续增长
7. 后续 `/dream` 读取 daily logs，提炼主题文件和 `MEMORY.md`

## 九、最关键的源码依据

### 9.1 `src/memdir/memdir.ts`

这份文件是 Assistant Mode daily-log 策略最核心的依据，明确说明：

- 会话是 long-lived
- 新记忆应写入按天命名的日志
- 记录方式是 append-only
- `MEMORY.md` 是 distilled index

### 9.2 `src/memdir/paths.ts`

这份文件给出 daily log 的标准路径形态：

- `<autoMemPath>/logs/YYYY/MM/YYYY-MM-DD.md`

### 9.3 `src/context.ts`

这份文件说明日期信息来自上下文：

- `currentDate`

## 十、最终总结

Assistant Mode 下的 Auto Memory 可以概括为：

- **不直接把新信息写进 `MEMORY.md`**
- **通过 system prompt 把“追加到 daily log”的策略交给主 Agent 执行**
- **通过路径模板 + 当前日期，让模型知道 today’s log 在哪**
- **通过通用文件工具完成真正写盘**
- **通过 `/dream` 在后续把原始日志提炼成结构化长期记忆**

从设计思想上看，这是一种把“**记忆采集**”与“**记忆整理/索引**”分离的架构：

- 前者强调低摩擦、不中断、持续追加
- 后者强调结构化、可读性与长期可维护性
