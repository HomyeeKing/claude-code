# Auto Memory - 如何保存记忆

记忆保存采用两步流程。

## 标准保存流程

### 步骤 1: 写入主题文件

将记忆写入它自己的文件（例如，`user_role.md`、`feedback_testing.md`），使用 frontmatter 格式。

### 步骤 2: 更新索引 (MEMORY.md)

在该文件的 `MEMORY.md` 中添加一个指向该文件的指针。

`MEMORY.md` 是一个索引，不是记忆本身 —— 每个条目应该是一行，约 150 个字符以内：

```markdown
- [Title](file.md) — one-line hook
```

**注意**: 
- `MEMORY.md` 没有 frontmatter
- 永远不要将记忆内容直接写入 `MEMORY.md`
- `MEMORY.md` 总是加载到系统提示中 —— 200 行之后会被截断，所以保持索引简洁

## 跳过索引模式 (Skip Index)

在某些情况下（如 Assistant 模式），可能跳过索引更新：

- 只写入主题文件
- 不更新 `MEMORY.md`
- 后续由 `/dream` 或其他机制统一整理

## 保存最佳实践

1. **按主题组织记忆，而不是按时间顺序**
   - 错误: `2024-01-conversations.md`
   - 正确: `user_preferences.md`

2. **更新或移除错误或过时的记忆**
   - 定期审查现有记忆
   - 发现错误时及时修正

3. **不要写入重复的记忆**
   - 首先检查是否存在可以更新的现有记忆
   - 优先更新，而不是创建新文件

## COMBINED 模式的目录选择

在 COMBINED 模式下（支持 private + team 目录）：

- 根据类型的 scope 指导选择目录（private 或 team）
- 每个目录有自己的 `MEMORY.md` 索引
- 两个 `MEMORY.md` 索引都会加载到系统提示中

## 文件命名约定

建议的命名格式：`{type}_{topic}.md`

示例:
- `user_role.md`
- `feedback_testing.md`
- `project_deadlines.md`
- `reference_linear.md`
