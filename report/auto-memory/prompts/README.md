# Auto Memory Prompts

本目录用于存放 Claude Code `auto-memory` 相关的 prompt 文档，目标是同时满足两种阅读方式：

- **读完整 prompt**：直接查看一份完整的主系统提示词
- **按主题拆解**：查看各个主题段落的职责与来源

## 阅读入口

- **完整主 prompt**：[`00-system-prompt.md`](./00-system-prompt.md)
- **主题拆分文件**：`01` 到 `08`

如果你只是想知道 auto-memory 在主 Agent 里到底收到什么提示词，优先看 `00-system-prompt.md`。

## 文件结构

| 文件 | 类型 | 说明 |
|------|------|------|
| [00-system-prompt.md](./00-system-prompt.md) | 完整 prompt | 主 Agent 的 auto-memory system prompt 汇总版，尽量展开最终文案，仅保留必要变量 |
| [01-base-opener.md](./01-base-opener.md) | 主题 prompt | 基础提示 / subagent opener |
| [02-memory-types.md](./02-memory-types.md) | 主题 prompt | 四种核心记忆类型：`user`、`feedback`、`project`、`reference` |
| [03-what-not-to-save.md](./03-what-not-to-save.md) | 主题 prompt | 哪些内容不应该保存为记忆 |
| [04-how-to-save.md](./04-how-to-save.md) | 主题 prompt | 如何写入主题文件和更新索引 |
| [05-frontmatter-example.md](./05-frontmatter-example.md) | 主题 prompt | 记忆文件的 YAML frontmatter 格式规范 |
| [06-trusting-recall.md](./06-trusting-recall.md) | 主题 prompt | 如何验证和使用召回的记忆 |
| [07-when-to-access.md](./07-when-to-access.md) | 主题 prompt | 何时应该读取和使用记忆 |
| [08-memory-drift-caveat.md](./08-memory-drift-caveat.md) | 主题 prompt | 记忆可能漂移/过时的警告 |

## 代码来源

本目录内容主要整理自以下源码：

- `src/memdir/memdir.ts`
  - `buildMemoryLines(...)`
  - `loadMemoryPrompt()`
- `src/memdir/memoryTypes.ts`
  - `TYPES_SECTION_INDIVIDUAL`
  - `WHAT_NOT_TO_SAVE_SECTION`
  - `WHEN_TO_ACCESS_SECTION`
  - `TRUSTING_RECALL_SECTION`
  - `MEMORY_FRONTMATTER_EXAMPLE`
- `src/services/extractMemories/prompts.ts`
  - extraction subagent 相关 prompt

## 当前组织方式

### 1. 完整主 Prompt

`00-system-prompt.md` 是本目录的主入口，目标是：

- 尽量还原主 Agent 实际使用的 auto-memory prompt
- 尽量展开静态文案
- 只保留少量必要变量，例如路径、阈值等

### 2. 主题拆分 Prompt

`01` 到 `08` 保留为按主题拆分的文档，方便：

- 单独分析某段 prompt 的语义
- 对照源码常量与分段设计
- 做局部迭代或评审

## 变量策略

本目录遵循以下原则：

- **尽量展开**：普通静态文案直接写成最终样子
- **少量保留变量**：仅在这些变量确实有意义时保留
  - 路径类变量，如 `${memoryDir}`
  - 阈值类变量，如 `${MAX_ENTRYPOINT_LINES}`
- **避免过度模板化**：不为了“看起来像源码”而保留太多插值表达式

## 范围说明

需要区分两类 prompt：

- **主 Agent 的 auto-memory system prompt**
  - 主要来自 `src/memdir/memdir.ts`
  - 本目录现在以 `00-system-prompt.md` 作为完整存档入口

- **memory extraction / consolidation 相关 prompt**
  - 例如 `src/services/extractMemories/prompts.ts`
  - 它们和 auto-memory 强相关，但不是主 Agent 的同一份系统提示词

因此，`01-base-opener.md` 这类文件更偏向子流程 prompt；`00-system-prompt.md` 则对应你最关心的“完整主 prompt”。

## 相关文档

- [记忆系统架构总览](../memory-system-architecture.md)
- [记忆主题提取机制](../memory-topic-extraction.md)
