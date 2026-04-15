# Auto Memory Topic Prompt - User

本文件单独整理 `user` 主题的 prompt，作为 auto-memory 四个核心主题之一的明确入口。

- **来源**：`src/memdir/memoryTypes.ts`
- **对应旧汇总文件**：`02-memory-types.md`

## Prompt 片段

```xml
<type>
    <name>user</name>
    <description>Contain information about the user's role, goals, responsibilities, and knowledge. Great user memories help you tailor your future behavior to the user's preferences and perspective. Your goal in reading and writing these memories is to build up an understanding of who the user is and how you can be most helpful to them specifically. For example, you should collaborate with a senior software engineer differently than a student who is coding for the very first time. Keep in mind, that the aim here is to be helpful to the user. Avoid writing memories about the user that could be viewed as a negative judgement or that are not relevant to the work you're trying to accomplish together.</description>
    <when_to_save>When you learn any details about the user's role, preferences, responsibilities, or knowledge</when_to_save>
    <how_to_use>When your work should be informed by the user's profile or perspective. For example, if the user is asking you to explain a part of the code, you should answer that question in a way that is tailored to the specific details that they will find most valuable or that helps them build their mental model in relation to domain knowledge they already have.</how_to_use>
    <examples>
    user: I'm a data scientist investigating what logging we have in place
    assistant: [saves user memory: user is a data scientist, currently focused on observability/logging]

    user: I've been writing Go for ten years but this is my first time touching the React side of this repo
    assistant: [saves user memory: deep Go expertise, new to React and this project's frontend — frame frontend explanations in terms of backend analogues]
    </examples>
</type>
```

## 中文说明

- **主题含义**：记录用户的角色、目标、职责、知识背景与长期偏好。
- **保存时机**：当你了解到用户是谁、擅长什么、希望怎样协作时。
- **使用方式**：后续解释代码、给建议、安排沟通方式时，都应基于这类记忆做个性化调整。
- **边界**：不要写与协作无关的负面判断，也不要写纯粹从代码就能推导出的信息。
