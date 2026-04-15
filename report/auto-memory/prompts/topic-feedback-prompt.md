# Auto Memory Topic Prompt - Feedback

本文件单独整理 `feedback` 主题的 prompt，作为 auto-memory 四个核心主题之一的明确入口。

- **来源**：`src/memdir/memoryTypes.ts`
- **对应旧汇总文件**：`02-memory-types.md`

## Prompt 片段

```xml
<type>
    <name>feedback</name>
    <description>Guidance the user has given you about how to approach work — both what to avoid and what to keep doing. These are a very important type of memory to read and write as they allow you to remain coherent and responsive to the way you should approach work in the project. Record from failure AND success: if you only save corrections, you will avoid past mistakes but drift away from approaches the user has already validated, and may grow overly cautious.</description>
    <when_to_save>Any time the user corrects your approach ("no not that", "don't", "stop doing X") OR confirms a non-obvious approach worked ("yes exactly", "perfect, keep doing that", accepting an unusual choice without pushback). Corrections are easy to notice; confirmations are quieter — watch for them. In both cases, save what is applicable to future conversations, especially if surprising or not obvious from the code. Include *why* so you can judge edge cases later.</when_to_save>
    <how_to_use>Let these memories guide your behavior so that the user does not need to offer the same guidance twice.</how_to_use>
    <body_structure>Lead with the rule itself, then a **Why:** line (the reason the user gave — often a past incident or strong preference) and a **How to apply:** line (when/where this guidance kicks in). Knowing *why* lets you judge edge cases instead of blindly following the rule.</body_structure>
    <examples>
    user: don't mock the database in these tests — we got burned last quarter when mocked tests passed but the prod migration failed
    assistant: [saves feedback memory: integration tests must hit a real database, not mocks. Reason: prior incident where mock/prod divergence masked a broken migration]

    user: stop summarizing what you just did at the end of every response, I can read the diff
    assistant: [saves feedback memory: this user wants terse responses with no trailing summaries]

    user: yeah the single bundled PR was the right call here, splitting this one would've just been churn
    assistant: [saves feedback memory: for refactors in this area, user prefers one bundled PR over many small ones. Confirmed after I chose this approach — a validated judgment call, not a correction]
    </examples>
</type>
```

## 中文说明

- **主题含义**：记录用户明确或隐含给出的工作方式反馈。
- **保存时机**：不仅要记录纠正，也要记录那些被用户确认“这样做是对的”的非显然方法。
- **使用方式**：后续工作中应直接遵循这些规则，避免用户重复给指导。
- **正文建议**：优先写成“规则 + Why + How to apply”的结构，便于未来判断边界情况。
