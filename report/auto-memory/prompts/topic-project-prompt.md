# Auto Memory Topic Prompt - Project

本文件单独整理 `project` 主题的 prompt，作为 auto-memory 四个核心主题之一的明确入口。

- **来源**：`src/memdir/memoryTypes.ts`
- **对应旧汇总文件**：`02-memory-types.md`

## Prompt 片段

```xml
<type>
    <name>project</name>
    <description>Information that you learn about ongoing work, goals, initiatives, bugs, or incidents within the project that is not otherwise derivable from the code or git history. Project memories help you understand the broader context and motivation behind the work the user is doing within this working directory.</description>
    <when_to_save>When you learn who is doing what, why, or by when. These states change relatively quickly so try to keep your understanding of this up to date. Always convert relative dates in user messages to absolute dates when saving (e.g., "Thursday" → "2026-03-05"), so the memory remains interpretable after time passes.</when_to_save>
    <how_to_use>Use these memories to more fully understand the details and nuance behind the user's request and make better informed suggestions.</how_to_use>
    <body_structure>Lead with the fact or decision, then a **Why:** line (the motivation — often a constraint, deadline, or stakeholder ask) and a **How to apply:** line (how this should shape your suggestions). Project memories decay fast, so the why helps future-you judge whether the memory is still load-bearing.</body_structure>
    <examples>
    user: we're freezing all non-critical merges after Thursday — mobile team is cutting a release branch
    assistant: [saves project memory: merge freeze begins 2026-03-05 for mobile release cut. Flag any non-critical PR work scheduled after that date]

    user: the reason we're ripping out the old auth middleware is that legal flagged it for storing session tokens in a way that doesn't meet the new compliance requirements
    assistant: [saves project memory: auth middleware rewrite is driven by legal/compliance requirements around session token storage, not tech-debt cleanup — scope decisions should favor compliance over ergonomics]
    </examples>
</type>
```

## 中文说明

- **主题含义**：记录项目中的背景、目标、计划、限制、截止日期、事件和决策动机。
- **保存时机**：当你了解到“谁在做什么、为什么做、何时完成”这类无法从代码或 git 直接推出的信息时。
- **使用方式**：帮助你理解用户任务背后的真实上下文，从而给出更合适的建议。
- **特殊注意**：项目记忆容易过时，保存时应尽量把相对日期转成绝对日期，并写清 `Why` 与 `How to apply`。
