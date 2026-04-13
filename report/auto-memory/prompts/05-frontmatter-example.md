# Auto Memory - Frontmatter 格式示例

记忆文件使用 YAML frontmatter + Markdown 正文的格式。

## Frontmatter 示例

```yaml
---
name: 用户角色与目标
type: user
description: 记录用户的基本信息和主要目标
---
```

## 字段说明

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `name` | string | 是 | 这个主题记忆的人类可读名称 |
| `type` | string | 是 | 主题类型: `user` / `feedback` / `project` / `reference` |
| `description` | string | 是 | 该主题文件记录的摘要 |

## 完整文件示例

### User 类型示例

```markdown
---
name: 用户角色与目标
type: user
description: 记录用户的基本信息和主要目标
---

## 角色
- 全栈开发工程师
- 专注于 React 和 Node.js

## 目标
- 提高代码质量
- 优化性能
```

### Feedback 类型示例

```markdown
---
name: 测试偏好
type: feedback
description: 用户关于测试方法的反馈
---

集成测试必须命中真实数据库，而不是 mock。

**Why:** 上季度 mock 和生产环境差异掩盖了一个损坏的迁移。

**How to apply:** 所有数据库相关的集成测试都应该使用真实数据库实例。
```

### Project 类型示例

```markdown
---
name: 发布冻结
type: project
description: 移动团队发布分支切割期间的合并冻结
---

合并冻结从 2026-03-05 开始，因为移动团队要切割发布分支。

**Why:** 移动发布周期要求代码冻结以确保稳定性。

**How to apply:** 标记任何安排在该日期之后的非关键 PR 工作。
```

### Reference 类型示例

```markdown
---
name: Bug 追踪系统
type: reference
description: 外部 bug 追踪系统的位置
---

Pipeline bugs 在 Linear 项目 "INGEST" 中跟踪。
```

## 设计意图

- **结构化元数据**: YAML frontmatter 提供机器可解析的结构化数据
- **人工可读**: Markdown 正文便于人类阅读和维护
- **版本控制友好**: 文本格式便于 git  diff 和版本控制
