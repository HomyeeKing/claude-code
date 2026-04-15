# Claude Code 记忆系统架构设计

Claude Code 实现了一个**多层次、文件化的记忆系统**，用于在对话之间持久化上下文信息。系统以文件系统为持久化载体，借助 AI 完成记忆提取、相关性选择和整合提炼。若按生命周期理解，这套系统可以拆成三个连续过程：**记忆的生产与存储、记忆的消费、记忆的汰换**。

## 一、记忆的生产与存储

这一部分回答两个问题：**记忆从什么时候开始产生**，以及**它最终存储在哪里**。

### 1.1 记忆什么时候开始产生

Claude Code 的记忆不是在每一轮对话开始时就立即写入，而是在满足特定条件后，以后台任务或安全提取点的方式沉淀下来。当前主要有两条生产链路。

#### 1.1.1 Auto Memory：在对话循环结束后提取

**文件位置**：`src/services/extractMemories/extractMemories.ts`

Auto Memory 会在每次查询循环结束时运行，也就是模型已经产出最终响应、且当前轮没有继续发起工具调用的时候。这样做的目的是在不打断主对话的前提下，利用本轮上下文提炼出可长期复用的信息。

它采用 **forked agent** 机制执行后台提取。这里的 forked agent 可以理解为：从当前主 Agent 的上下文中分叉出一个独立的子 Agent，让它复用已有上下文和提示缓存，去执行记忆提取或整合等后台任务，而不阻塞主对话流程。

它有几个关键特征：

- **继承上下文**：子 Agent 不是从零开始，而是复用父 Agent 已有信息
- **共享提示缓存**：避免重复计算，降低 token 消耗
- **执行隔离**：主 Agent 继续与用户交互，forked agent 在旁路执行提取
- **结果回写**：提取完成后，将结果沉淀为记忆文件、索引更新或整合后的主题内容
- **适合后台任务**：尤其适合记忆提取、相关性筛选、夜间整合这类不需要阻塞回答的工作

工作机制可以概括为：

- 在每次查询循环结束时运行
- 使用 forked agent 模式
- 与主 Agent 互斥：如果主 Agent 已写入记忆，则跳过

对应的工具权限也被严格限制：

```typescript
export function createAutoMemCanUseTool(memoryDir: string): CanUseToolFn {
  // 允许：READ/GREP/GLOB（无限制）
  // 允许：Bash（只读命令）
  // 允许：EDIT/WRITE（仅限 auto-memory 目录内）
  // 拒绝：其他所有操作
}
```

#### 1.1.2 Session Memory：在会话达到阈值后提取

**文件位置**：`src/services/SessionMemory/sessionMemory.ts`

除了 Auto Memory，系统还维护一条更偏“长会话摘要”的链路：Session Memory。它不是每轮都触发，而是要在会话增长到一定规模后才开始提取。

默认阈值如下：

```typescript
export const DEFAULT_SESSION_MEMORY_CONFIG: SessionMemoryConfig = {
  minimumMessageTokensToInit: 10000,
  minimumTokensBetweenUpdate: 5000,
  toolCallsBetweenUpdates: 3,
}
```

触发条件由 `shouldExtractMemory(messages)` 控制，核心判断包括：

1. 满足初始化 token 阈值
2. 满足 token 增长阈值
3. 满足工具调用阈值
4. 最后助手轮次无工具调用，处于安全提取点

```typescript
export function shouldExtractMemory(messages: Message[]): boolean {
  // 1. 满足初始化阈值
  // 2. 满足 token 增长阈值
  // 3. 满足工具调用阈值
  // 4. 最后助手轮次无工具调用（安全提取点）
}
```

### 1.2 会产生哪些类型的记忆

**文件位置**：`src/memdir/memoryTypes.ts`

在语义层面，系统把记忆分成四种核心类型：

| 类型 | 说明 | 示例 |
|------|------|------|
| **user** | 用户信息 | 角色、目标、偏好 |
| **feedback** | 反馈 | 用户纠正和确认的行为指导 |
| **project** | 项目 | 工作动态、目标、决策 |
| **reference** | 参考 | 外部系统链接（Linear、Grafana 等） |

这里的“类型”回答的是“这条记忆在说什么”，而不是“它存在哪一层”。

### 1.3 这些记忆有哪些作用域

**文件位置**：`src/utils/memory/types.ts`

如果说“类型”定义了记忆的内容语义，那么“作用域”定义的就是这条记忆可以在多大范围内被共享和复用。

| 作用域 | 存储路径 | 说明 | 使用场景 |
|--------|----------|------|----------|
| **User** | `~/.claude/agent-memory/` | 用户级记忆，跨项目共享 | 适合保存用户长期稳定的角色信息、工作偏好、常用约束等跨项目可复用知识 |
| **Project** | `.claude/agent-memory/` | 项目级记忆，项目内共享 | 适合保存当前项目的约定、架构决策、领域背景、协作规范等项目上下文 |
| **Local** | `.claude/agent-memory-local/` | 本地级记忆，不上传 | 适合保存本地调试习惯、设备相关配置、个人私有上下文等仅当前机器有效的信息 |
| **Managed** | - | 托管记忆 | 适合由平台统一管理、无需用户手动维护的记忆内容 |
| **AutoMem** | - | 自动记忆 | 适合沉淀对话过程中自动提取出的用户偏好、项目事实和阶段性结论 |
| **TeamMem** | - | 团队记忆（功能门控） | 适合沉淀团队共享的规范、流程、集体经验和可跨成员复用的知识 |

此外，Agent 专用记忆还提供更明确的三种范围：

**文件位置**：`src/tools/AgentTool/agentMemory.ts`

```typescript
export type AgentMemoryScope = 'user' | 'project' | 'local'

// user:   <memoryBase>/agent-memory/<agentType>/
// project: <cwd>/.claude/agent-memory/<agentType>/
// local:   <cwd>/.claude/agent-memory-local/<agentType>/
```

### 1.4 记忆最终存储在哪里

Claude Code 的记忆系统是文件化的，核心优势是可持久化、可人工编辑、也便于版本控制。典型目录结构如下：

```text
~/.claude/
├── projects/
│   └── <sanitized-git-root>/
│       └── memory/
│           ├── MEMORY.md              # 索引文件（入口点）
│           ├── user_role.md           # 用户记忆文件
│           ├── feedback_testing.md    # 反馈记忆
│           ├── project_deadlines.md   # 项目记忆
│           └── reference_linear.md    # 参考记忆
│
├── agent-memory/
│   └── <agentType>/
│       └── MEMORY.md                  # Agent 专用记忆
│
└── session-memory/
    └── session-memory.md              # 会话记忆
```

这里采用的是**索引与内容分离**的组织方式：

- `MEMORY.md` 是入口索引
- 各个主题记忆存放在独立 Markdown 文件中

### 1.5 存储路径是如何解析的

**文件位置**：`src/memdir/paths.ts`

Auto Memory 的路径解析优先级如下：

1. `CLAUDE_COWORK_MEMORY_PATH_OVERRIDE` 环境变量
2. `autoMemoryDirectory` 设置项
3. 默认路径：`<memoryBase>/projects/<sanitized-git-root>/memory/`

底层先通过 `getMemoryBaseDir()` 确定基础目录，再由 `getAutoMemPath()` 计算当前项目的 Auto Memory 目录。

```typescript
export function getMemoryBaseDir(): string {
  if (process.env.CLAUDE_CODE_REMOTE_MEMORY_DIR) {
    return process.env.CLAUDE_CODE_REMOTE_MEMORY_DIR
  }
  return getClaudeConfigHomeDir()
}

export const getAutoMemPath = memoize((): string => {
  const override = getAutoMemPathOverride() ?? getAutoMemPathSetting()
  if (override) return override
  const projectsDir = join(getMemoryBaseDir(), 'projects')
  return join(projectsDir, sanitizePath(getAutoMemBase()), AUTO_MEM_DIRNAME) + sep
}, () => getProjectRoot())
```

系统是否启用 Auto Memory 也有一条独立的优先级链：

1. `CLAUDE_CODE_DISABLE_AUTO_MEMORY` 环境变量
2. `CLAUDE_CODE_SIMPLE`（`--bare`）
3. CCR 无持久存储
4. `settings.json` 中的 `autoMemoryEnabled`
5. 默认启用

```typescript
export function isAutoMemoryEnabled(): boolean {
  // 优先级链（首个定义的值生效）：
  // 1. CLAUDE_CODE_DISABLE_AUTO_MEMORY 环境变量
  // 2. CLAUDE_CODE_SIMPLE (--bare) → 禁用
  // 3. CCR 无持久存储 → 禁用
  // 4. settings.json 中的 autoMemoryEnabled
  // 5. 默认：启用
}
```

### 1.6 记忆是如何写入的

记忆落盘采用两步保存流程：

1. 写入独立主题文件，例如 `user_role.md`
2. 在 `MEMORY.md` 中加入对应的索引条目

记忆文件使用 **YAML frontmatter + Markdown 正文** 格式，既保留结构化元数据，也便于人工阅读和维护。

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

对应目录创建逻辑如下：

```typescript
export async function ensureMemoryDirExists(memoryDir: string): Promise<void> {
  const fs = getFsImplementation()
  await fs.mkdir(memoryDir)
}
```

### 1.7 Assistant 模式下的特殊存储方式

Assistant 模式不是单纯的界面选项，而是一种 Agent 运行时模式。它的目标是让主 Agent 保持前台响应，把耗时任务、子 Agent 执行和记忆沉淀尽量放到后台异步完成。

在这种模式下，系统不会优先直接写主题文件和 `MEMORY.md`，而是先写**追加式日志**：

```text
<autoMemPath>/logs/YYYY/MM/YYYY-MM-DD.md
```

其行为特点包括：

- 自定义系统提示会追加 assistant prompt addendum
- 强依赖 Brief / `SendUserMessage` 路径
- 子 Agent 和 forked command 更倾向于异步后台执行
- 阻塞型 bash 在预算超时后会自动后台化
- 记忆先按日期写入日志，后续再由 `/dream` 提炼为主题文件和 `MEMORY.md`

---

## 二、记忆的消费

生产出的记忆并不会被无条件全部加载。系统会先定位记忆入口，再扫描候选文件，最后只把和当前问题最相关的部分送入上下文。

### 2.1 系统如何开始消费记忆

**文件位置**：`src/memdir/memdir.ts`

记忆消费的入口点是 `MEMORY.md`。它作为整个记忆目录的导航索引，告诉系统有哪些主题文件可供后续参考。

为了避免入口文件无限膨胀，系统会对它施加双重截断限制：

- 最多 200 行
- 最多 25KB

```typescript
export const ENTRYPOINT_NAME = 'MEMORY.md'
export const MAX_ENTRYPOINT_LINES = 200
export const MAX_ENTRYPOINT_BYTES = 25_000

export function truncateEntrypointContent(raw: string): EntrypointTruncation {
  // 按行数和字节数双重限制截断
}
```

### 2.2 系统如何发现可用记忆文件

**文件位置**：`src/memdir/memoryScan.ts`

在读完入口点之后，系统会扫描记忆目录中的主题文件，并读取 frontmatter，得到轻量级的头部信息。每个候选文件会被整理成一个 `MemoryHeader`：

```typescript
export type MemoryHeader = {
  filename: string
  filePath: string
  mtimeMs: number
  description: string | null
  type: MemoryType | undefined
}

export async function scanMemoryFiles(
  memoryDir: string,
  signal: AbortSignal,
): Promise<MemoryHeader[]> {
  // 最多扫描 200 个 .md 文件
  // 按修改时间倒序排列
}
```

这一步的目标不是加载全文，而是先建立一份可筛选、可排序的记忆候选集合。

### 2.3 系统如何决定“读哪些记忆”

**文件位置**：`src/memdir/findRelevantMemories.ts`

系统不会把所有候选记忆都塞入上下文，而是使用 AI 模型做相关性选择，只挑出与当前查询最相关的少数内容。默认最多返回 5 个相关记忆：

```typescript
export async function findRelevantMemories(
  query: string,
  memoryDir: string,
  signal: AbortSignal,
  recentTools: readonly string[] = [],
  alreadySurfaced: ReadonlySet<string> = new Set(),
): Promise<RelevantMemory[]>
```

筛选时会综合考虑：

- 当前 query
- 目标 memoryDir
- 最近使用过的工具
- 已经展示过的记忆集合

这一步是记忆消费链路中的关键控制点，它决定了系统最终“用哪些记忆”，而不是“系统一共存了多少记忆”。

### 2.4 记忆在运行时如何参与上下文

整个消费流程可以理解为三层协作：

1. `MEMORY.md` 提供目录和入口
2. 主题文件提供细节事实
3. 相关性选择控制上下文体积和命中率

这意味着记忆系统并不是单纯的文件读取器，而是一个“先导航、再筛选、后注入”的上下文增强系统。

### 2.5 用户如何主动消费和管理记忆

除了系统自动消费，用户也可以通过命令主动查看、管理和审查记忆。

#### 2.5.1 `/memory` 命令

**文件位置**：`src/commands/memory/memory.tsx`

`/memory` 提供交互式记忆管理界面，用于：

- 选择记忆文件
- 在编辑器中打开
- 创建新记忆文件

#### 2.5.2 `/remember` 技能

**文件位置**：`src/skills/bundled/remember.ts`

`/remember` 更偏向记忆审查，用于：

- 审查当前记忆景观
- 提议将记忆提升到 `CLAUDE.md`
- 检测重复、过时、冲突条目

### 2.6 系统如何识别不同类型的记忆文件

**文件位置**：`src/utils/memoryFileDetection.ts`

在消费阶段，系统还需要判断一个文件属于哪种记忆结构，例如是否是 session memory、是否属于某个作用域、是否属于自动管理的记忆文件。

```typescript
export function detectSessionFileType(filePath: string):
  'session_memory' | 'session_transcript' | null

export function memoryScopeForPath(filePath: string): MemoryScope | null

export function isAutoManagedMemoryFile(filePath: string): boolean
```

这些检测函数是记忆消费链路的底层基础设施，保证系统能在不同文件间采用正确的解释方式。

---

## 三、记忆的汰换

记忆系统并不是把信息无限堆积起来，而是要持续判断哪些记忆已经变旧、哪些需要降权、哪些应该被重新整合为更稳定的长期知识。

### 3.1 为什么需要汰换

如果记忆只生产不整理，系统会遇到几个问题：

- 旧信息持续累积，增加上下文噪音
- 碎片化记忆越来越多，难以复用
- 历史观察容易被误当成当前实时状态

因此，汰换并不一定意味着“删除”，更常见的是**提醒、降权、整合和压缩**。

### 3.2 系统如何判断一条记忆是否变旧

**文件位置**：`src/memdir/memoryAge.ts`

系统通过文件的 `mtimeMs` 计算记忆年龄，并生成人类可读的时间描述：

```typescript
export function memoryAgeDays(mtimeMs: number): number {
  return Math.max(0, Math.floor((Date.now() - mtimeMs) / 86_400_000))
}

export function memoryAge(mtimeMs: number): string {
  const d = memoryAgeDays(mtimeMs)
  if (d === 0) return 'today'
  if (d === 1) return 'yesterday'
  return `${d} days ago`
}
```

这类年龄信息不会直接删除记忆，但会影响系统对它的解释方式。

### 3.3 新鲜度提示如何参与“软性汰换”

系统会为较旧的记忆生成 freshness warning，提醒它只是某个时间点上的观察，而不是实时状态：

```typescript
export function memoryFreshnessText(mtimeMs: number): string {
  const d = memoryAgeDays(mtimeMs)
  if (d <= 1) return ''
  return `This memory is ${d} days old. Memories are point-in-time observations, not live state...`
}
```

这体现的是一种“软性汰换”策略：

- 保留旧记忆，避免知识直接丢失
- 通过提示降低系统对旧信息的盲目信任
- 把“是否仍然可信”交给后续消费链路综合判断

### 3.4 Auto Dream 如何做整合与压缩

**文件位置**：`src/services/autoDream/autoDream.ts`

当系统检测到记忆累积到一定程度后，会触发 Auto Dream 做自动整合。这里同样采用 forked agent，因为整合任务本身较重、运行时机独立，也不需要直接参与当前轮次回答。

夜间整合的门控机制包括：

- **时间门控**：距离上次整合至少经过 `minHours`
- **会话门控**：自上次整合后累计了足够多的新会话
- **锁机制**：防止并发执行整合
- **执行方式**：使用 forked agent 后台完成

默认配置如下：

```typescript
const DEFAULTS: AutoDreamConfig = {
  minHours: 24,
  minSessions: 5,
}
```

Auto Dream 的作用不是简单合并文件，而是把零散日志和碎片化观察提炼为更稳定、更可复用的主题知识。

### 3.5 Assistant 模式下的汰换路径

在 Assistant 模式中，汰换逻辑更加明显，因为系统先写按日期分组的日志，而不是直接维护主题记忆。其完整路径可以概括为：

1. 会话运行期间先追加到当日日志
2. 日志作为原始观察保留下来
3. 夜间由 `/dream` 技能提炼为主题文件和 `MEMORY.md`

这实际上是一种“先追加、后归档、再提炼”的汰换模型。和直接覆盖旧记忆相比，这种方式更适合保留上下文轨迹，同时逐步降低碎片噪音。

### 3.6 运行模式对记忆汰换的影响

当前系统存在几种容易混淆的运行模式，它们对记忆链路的影响并不相同：

| 模式 | 主要目标 | 触发方式 | 用户通信方式 | 子 Agent / 子任务行为 | 长任务处理 | 记忆行为 |
|------|----------|----------|--------------|------------------------|------------|----------|
| **普通模式（默认 REPL）** | 正常对话与工具调用 | 默认进入 | 直接输出普通 assistant 文本 | 可调用 subagent，但通常按同步流程执行 | 主流程等待结果 | 按常规 Auto Memory / `MEMORY.md` 机制工作 |
| **Brief 模式** | 将用户可见输出收敛到检查点式消息 | `--brief`、`/brief`、`defaultView: 'chat'` 等 | 优先使用 `SendUserMessage` | 不改变调度模型，本身更偏通信层约束 | 不特别改变 | 不改变底层记忆机制 |
| **Proactive 模式** | 让 Agent 主动推进任务 | `--proactive` 或 `CLAUDE_CODE_PROACTIVE` | 结合普通输出或 Brief 输出 | 可以持续自主行动，并接收周期性 `<tick>` 提示 | 支持 `SleepTool` 在空闲时挂起等待 | 不改变记忆结构，重点在自主推进 |
| **Assistant 模式** | 让主 Agent 作为协调者保持前台响应，并将后台任务异步化 | `.claude/settings.json` 中 `assistant: true`，并满足 KAIROS/gate/trust 条件；内部也有 `--assistant` 隐藏入口 | 强依赖 Brief / `SendUserMessage` 路径 | 强制更多子 Agent 与 forked command 走异步、后台回流模式 | 主线程阻塞 bash 会自动后台化，保持主会话响应 | 优先写每日追加日志，后续由 `/dream` 提炼为主题文件和 `MEMORY.md` |

可以粗略理解为：

- **普通模式**：标准同步式对话执行
- **Brief 模式**：主要改变“怎么向用户汇报”
- **Proactive 模式**：主要改变“是否主动推进工作”
- **Assistant 模式**：主要改变“系统如何编排任务与回流结果”，因此也最直接影响记忆的沉淀与汰换路径

---

## 附录一：核心文件索引

| 文件路径 | 职责 | 关键功能 |
|---------|------|----------|
| `src/memdir/memoryTypes.ts` | 记忆类型定义与提示文本 | 定义 4 种记忆类型 |
| `src/memdir/memdir.ts` | 记忆目录核心逻辑、提示构建 | 入口点加载、截断逻辑 |
| `src/memdir/paths.ts` | 路径解析、配置检查 | 路径解析、启用控制 |
| `src/memdir/memoryScan.ts` | 记忆文件扫描、frontmatter 解析 | 扫描记忆文件 |
| `src/memdir/memoryAge.ts` | 记忆年龄计算、新鲜度警告 | 年龄计算、新鲜度文本 |
| `src/memdir/findRelevantMemories.ts` | AI 驱动的相关记忆选择 | 相关性查找 |
| `src/services/extractMemories/extractMemories.ts` | 后台自动记忆提取 | forked agent 提取 |
| `src/services/SessionMemory/sessionMemory.ts` | 会话记忆管理 | 会话记忆触发 |
| `src/services/SessionMemory/sessionMemoryUtils.ts` | 会话记忆工具函数 | 工具函数 |
| `src/services/autoDream/autoDream.ts` | 记忆整合与提炼 | 夜间整合 |
| `src/tools/AgentTool/agentMemory.ts` | Agent 专用记忆 | Agent 记忆作用域 |
| `src/utils/memoryFileDetection.ts` | 记忆文件类型检测 | 文件类型识别 |
| `src/commands/memory/memory.tsx` | `/memory` 命令实现 | 交互式管理 |
| `src/skills/bundled/remember.ts` | `/remember` 技能 | 记忆审查 |
| `src/utils/memory/types.ts` | 记忆作用域类型 | 作用域定义 |

## 附录二：架构总结

### 设计亮点

1. **文件系统持久化**
   - 使用 Markdown 文件存储，便于版本控制和人工编辑
   - YAML frontmatter 提供结构化元数据

2. **AI 驱动自动化**
   - forked agent 模式实现无干扰的后台提取
   - 相关性选择确保只加载有用记忆
   - 自动整合避免记忆碎片化

3. **安全与权限**
   - 严格的工具权限限制
   - 路径验证和防遍历保护
   - 多层配置控制

4. **性能优化**
   - 缓存机制减少重复计算
   - 扫描节流避免频繁 IO
   - 提示缓存共享降低 token 消耗

5. **可扩展性**
   - 模块化设计便于扩展新记忆类型
   - 配置系统支持多种部署场景
   - 功能门控支持渐进式发布
