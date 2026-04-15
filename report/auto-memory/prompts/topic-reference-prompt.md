# Auto Memory Topic Prompt - Reference

本文件单独整理 `reference` 主题的 prompt，作为 auto-memory 四个核心主题之一的明确入口。

- **来源**：`src/memdir/memoryTypes.ts`
- **对应旧汇总文件**：`02-memory-types.md`

## Prompt 片段

```xml
<type>
    <name>reference</name>
    <description>Stores pointers to where information can be found in external systems. These memories allow you to remember where to look to find up-to-date information outside of the project directory.</description>
    <when_to_save>When you learn about resources in external systems and their purpose. For example, that bugs are tracked in a specific project in Linear or that feedback can be found in a specific Slack channel.</when_to_save>
    <how_to_use>When the user references an external system or information that may be in an external system.</how_to_use>
    <examples>
    user: check the Linear project "INGEST" if you want context on these tickets, that's where we track all pipeline bugs
    assistant: [saves reference memory: pipeline bugs are tracked in Linear project "INGEST"]

    user: the Grafana board at grafana.internal/d/api-latency is what oncall watches — if you're touching request handling, that's the thing that'll page someone
    assistant: [saves reference memory: grafana.internal/d/api-latency is the oncall latency dashboard — check it when editing request-path code]
    </examples>
</type>
```

## 中文说明

- **主题含义**：记录外部系统里的信息入口，以及这些入口分别是干什么的。
- **保存时机**：当你得知某类 bug、反馈、监控、文档或任务追踪是在项目外部哪个系统中查看时。
- **使用方式**：后续一旦用户提到相关外部系统，可以快速知道该去哪里查最新信息。
- **边界**：这里保存的是“去哪里看”和“它的用途”，不是把外部系统里的实时内容整体抄进记忆。
