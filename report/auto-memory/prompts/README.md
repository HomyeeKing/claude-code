# Auto Memory Prompts

本目录包含 Claude Code Auto Memory 系统的各个主题的 prompt 模板。

这些 prompt 原本分散在以下代码文件中：
- `src/services/extractMemories/prompts.ts`
- `src/memdir/memoryTypes.ts`

## 文件结构

| 文件 | 主题 | 说明 |
|------|------|------|
| [01-base-opener.md](./01-base-opener.md) | 基础提示 | Memory extraction subagent 的角色定义和核心约束 |
| [02-memory-types.md](./02-memory-types.md) | 记忆类型 | 四种核心记忆类型：user、feedback、project、reference |
| [03-what-not-to-save.md](./03-what-not-to-save.md) | 排除规则 | 哪些内容不应该保存为记忆 |
| [04-how-to-save.md](./04-how-to-save.md) | 保存指南 | 如何写入主题文件和更新索引 |
| [05-frontmatter-example.md](./05-frontmatter-example.md) | Frontmatter 格式 | 记忆文件的 YAML frontmatter 格式规范 |
| [06-trusting-recall.md](./06-trusting-recall.md) | 信任回忆 | 如何正确使用和验证召回的记忆 |
| [07-when-to-access.md](./07-when-to-access.md) | 访问时机 | 何时应该读取和使用记忆 |
| [08-memory-drift-caveat.md](./08-memory-drift-caveat.md) | 记忆漂移 | 记忆可能过时的警告和验证策略 |

## 使用方式

这些 prompt 文件用于构建：

1. **主 Agent 的 memory prompt** (`src/memdir/memdir.ts`)
   - 基础说明
   - 显式 remember/forget 指令
   - 记忆类型分类法
   - 不该存什么
   - 如何保存
   - 何时访问 memory
   - 如何验证 recalled memory
   - memory 与 plan/tasks 的边界

2. **Extraction subagent 的 prompt** (`src/services/extractMemories/prompts.ts`)
   - opener (基础提示)
   - 工具约束
   - 现有记忆清单
   - 记忆类型分类法
   - 不该存什么
   - 如何保存

## Prompt 拼装逻辑

最终的 memory prompt 不是由单个文件构成，而是由多个片段共同拼装：

```
基础说明
+ 显式 remember/forget 规则
+ 类型分类法 (TYPES_SECTION)
+ 不该存什么 (WHAT_NOT_TO_SAVE)
+ 如何保存 (HOW_TO_SAVE)
+ 何时访问 (WHEN_TO_ACCESS)
+ 如何验证 (TRUSTING_RECALL)
+ 记忆漂移警告 (MEMORY_DRIFT_CAVEAT)
+ 目录与索引约束
+ 子 Agent 工具限制
```

## 记忆类型快速参考

| 类型 | 内容 | Scope |
|------|------|-------|
| user | 用户角色、目标、偏好、知识 | private |
| feedback | 用户纠正和确认的行为指导 | private/team |
| project | 工作动态、目标、决策 | private/team |
| reference | 外部系统链接 | team |

## 相关文档

- [记忆系统架构总览](../memory-system-architecture.md)
- [记忆主题提取机制](../memory-topic-extraction.md)
