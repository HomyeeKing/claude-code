# Auto Memory 存储抽象设计：从当前文件接口到 `MemoryManager`

本文基于当前仓库里 `auto memory` 的实现，目标不是只总结“本地文件怎么存”，而是进一步抽出一层统一的 `MemoryManager` 抽象，方便后续演进出不同的记忆后端实现，例如：

- 基于当前 Markdown 文件系统的 `FileMemoryManager`
- 基于数据库 / 表结构 / 全文索引的 `DatabaseMemoryManager`
- 未来可能的远程 memory service 实现

本文**聚焦单人 auto memory**，不展开 `team memory` 的 pull / push / watch 同步逻辑。

## 一、先说结论

当前仓库里，`auto memory` 还没有显式的 `MemoryManager` 类。现有能力分散在几层函数和类型里：

1. **路径层**：决定 memory 存到哪里
2. **协议层**：定义 `MEMORY.md` 和主题文件如何组织
3. **索引层**：扫描已有 memory 文件并构造 manifest
4. **查询层**：从已有记忆中筛选与当前 query 相关的记忆
5. **写入编排层**：在对话结束后自动提取、增量写入并更新索引

从抽象设计视角看，这套实现已经天然具备了一个统一管理器的雏形，只是目前还没有被收敛成一个类。  
因此，更准确的结论是：

- 当前代码已经有了 **memory CRUD + recall + index** 的大部分能力基础
- 缺的是一个统一的**领域抽象层**，把“文件协议细节”从“记忆管理能力”里剥离出来
- 后续最合适的演进方式，是抽出 `abstract class MemoryManager`，再让 `FileMemoryManager` 和 `DatabaseMemoryManager` 分别实现

最值得关注的文件仍然是：

- `src/memdir/paths.ts`
- `src/memdir/memdir.ts`
- `src/memdir/memoryTypes.ts`
- `src/memdir/memoryScan.ts`
- `src/memdir/findRelevantMemories.ts`
- `src/services/extractMemories/extractMemories.ts`
- `src/services/extractMemories/prompts.ts`

## 二、当前实现为什么适合抽象成 `MemoryManager`

### 2.1 当前实现已经覆盖了记忆管理的核心能力

虽然现在没有一个统一类，但站在能力层面，当前代码已经覆盖了典型的记忆管理系统所需能力：

- **Create**：新增主题 memory 文件，并更新 `MEMORY.md`
- **Read**：扫描 memory 文件、读取 frontmatter、按需 recall
- **Update**：优先更新已有主题文件，而不是重复新建
- **Delete**：prompt 已经约束“如果用户要求 forget，需要删除相关内容”
- **List / Query**：通过扫描和 manifest 获得清单
- **Recall / Search**：根据 query 从记忆集合里筛出最相关记忆
- **Index maintenance**：维护 `MEMORY.md` 作为入口索引

也就是说，当前不是“没有 manager”，而是**manager 的职责被拆散到了多个函数模块中**。

### 2.2 当前实现里“通用能力”和“文件协议细节”混在一起

现在的设计里，以下两类概念是耦合在一起的：

- **通用记忆管理能力**
  - 创建记忆
  - 更新记忆
  - 删除记忆
  - 查询记忆
  - 查找相关记忆
- **文件实现特有细节**
  - 路径怎么拼
  - frontmatter 怎么解析
  - `MEMORY.md` 怎么组织
  - 主题文件怎么命名

如果后续接数据库，后面这组细节都会变化，但前面那组能力不会变化。  
所以抽象的核心思路应该是：

- 把“**记忆管理能力**”收敛到 `MemoryManager`
- 把“**文件存储协议**”收敛到 `FileMemoryManager`
- 把“**DB 表结构 / 检索索引**”收敛到 `DatabaseMemoryManager`

## 三、当前 file-based memory 的真实能力拆解

这一节先把当前仓库已经存在的能力拆出来，作为后面定义 `MemoryManager` 的依据。

## 四、路径相关接口：决定存储后端的物理位置

核心文件：`src/memdir/paths.ts`

主要接口：

- `isAutoMemoryEnabled()`
- `isExtractModeActive()`
- `getMemoryBaseDir()`
- `getAutoMemPath()`
- `getAutoMemDailyLogPath(date?)`
- `getAutoMemEntrypoint()`
- `isAutoMemPath(absolutePath)`

### 4.1 职责

这组接口负责统一解析 auto memory 的文件系统位置：

- 是否启用 auto memory
- memory 根目录在哪
- 当前项目对应的 memory 目录在哪
- `MEMORY.md` 的绝对路径是什么
- 某个路径是否属于 auto memory 目录

### 4.2 默认目录结构

默认情况下，目录形态大致是：

```text
<memoryBase>/projects/<sanitized-git-root>/memory/
```

也就是说，它是**按项目隔离存储**的。

如果处于 Assistant / Kairos 相关模式，还会用到按日期组织的日志路径：

```text
<autoMemPath>/logs/YYYY/MM/YYYY-MM-DD.md
```

### 4.3 对抽象层的启发

这层说明：

- “记忆存储位置”本身就是一类后端实现细节
- `MemoryManager` 不应该把 `path` 当成唯一核心概念
- 更通用的抽象应该使用 `MemoryIdentity` / `id`
- `path` 应该只是 `FileMemoryManager` 里的一个实现字段

## 五、存储协议相关接口：定义 file-based 的持久化规则

核心文件：`src/memdir/memdir.ts`

关键接口 / 常量：

- `ENTRYPOINT_NAME = 'MEMORY.md'`
- `MAX_ENTRYPOINT_LINES`
- `MAX_ENTRYPOINT_BYTES`
- `truncateEntrypointContent(raw)`
- `ensureMemoryDirExists(memoryDir)`
- `buildMemoryLines(...)`
- `buildMemoryPrompt(...)`
- `loadMemoryPrompt()`

### 5.1 核心约定

这一层定义了 auto memory 的**文件组织协议**：

- 入口文件固定为 `MEMORY.md`
- 真正的记忆内容不全堆在 `MEMORY.md` 中
- 每个主题记忆以单独 Markdown 文件存储
- `MEMORY.md` 只承担“索引入口”的职责

可以简单理解为：

- **主题文件**：存正文
- **`MEMORY.md`**：存导航

### 5.2 目录保证

`ensureMemoryDirExists(memoryDir)` 的职责很直接：

- 确保 memory 目录存在
- 后续写主题文件和更新 `MEMORY.md` 时不需要重复创建目录

### 5.3 入口文件截断

`truncateEntrypointContent(raw)` 用于限制 `MEMORY.md` 被加载进上下文时的体积，避免：

- 行数过多
- 字节数过大
- 对主 prompt 造成上下文压力

这说明 `MEMORY.md` 不是“越大越好”，它更像一个**快速索引页**。

### 5.4 对抽象层的启发

这一层非常关键，因为它说明：

- `MEMORY.md` 不是通用 memory 实体
- 它只是 file-based 实现中的**索引投影（index projection）**
- 所以 `MEMORY.md` 不应该成为 `MemoryManager` 的通用接口中心
- 更合理的抽象是：
  - 通用层暴露 `buildManifest()` / `rebuildIndex()` 一类能力
  - `FileMemoryManager` 内部决定 manifest 是否落到 `MEMORY.md`
  - `DatabaseMemoryManager` 可以完全没有 `MEMORY.md`

## 六、memory 类型相关接口：定义通用元数据语义

核心文件：`src/memdir/memoryTypes.ts`

关键接口 / 常量：

- `MEMORY_TYPES`
- `MemoryType`
- `parseMemoryType(raw)`
- `MEMORY_FRONTMATTER_EXAMPLE`
- `TYPES_SECTION_INDIVIDUAL`
- `TYPES_SECTION_COMBINED`

### 6.1 支持的类型

当前主题文件的 `type` 主要有 4 类：

- `user`
- `feedback`
- `project`
- `reference`

### 6.2 这一层的职责

这组定义负责约束 memory 文件 frontmatter 中的类型字段，是主题文件的**元数据协议层**。

### 6.3 对抽象层的启发

这层是最适合直接保留进通用抽象里的，因为：

- `MemoryType` 不依赖文件系统
- 它描述的是记忆的**业务语义**，不是存储介质细节
- 不管底层是 Markdown 还是数据库，`type` 都仍然是记忆记录的核心字段

也就是说，`MemoryType` 应该上移为 `MemoryManager` 通用领域模型的一部分。

## 七、本地扫描 / 读取接口：当前的 list + header query 能力

核心文件：`src/memdir/memoryScan.ts`

关键接口：

- `interface MemoryHeader`
- `scanMemoryFiles(memoryDir, signal)`
- `formatMemoryManifest(memories)`

### 7.1 `MemoryHeader`

`MemoryHeader` 表示系统扫描一个主题文件后提炼出的最小元信息集合，通常包括：

- `filename`
- `filePath`
- `mtimeMs`
- `description`
- `type`

它其实就是 file-based 实现里的**轻量索引记录**。

### 7.2 `scanMemoryFiles(...)`

这个函数是最核心的本地读取接口之一，职责是：

- 扫描 memory 目录下的 `.md` 文件
- 排除 `MEMORY.md`
- 读取文件头部并解析 frontmatter
- 提取出可用于索引和筛选的元数据

这说明系统在消费记忆时，并不是每次都全文加载所有主题文件，而是优先读取**轻量级头部信息**。

### 7.3 `formatMemoryManifest(...)`

这个函数会把扫描结果整理成 manifest 文本，供 recall 或 extraction 阶段使用。

它的价值在于：

- 把分散在文件系统里的主题文件先压缩成“目录清单”
- 让模型先看摘要、再决定是否进一步使用某个主题文件

### 7.4 对抽象层的启发

这一层对应到 `MemoryManager` 后，至少可以抽出两类能力：

- `listMemories(query)`：列出满足条件的记忆
- `buildManifest(memories?)`：把记忆集合投影成供 recall / extraction 使用的目录文本

其中：

- `MemoryHeader` 更像 file-based 的内部 DTO
- 通用层更适合使用 `MemoryRecord` 或 `MemorySearchResult`

## 八、相关记忆选择接口：当前的 recall 能力

核心文件：`src/memdir/findRelevantMemories.ts`

关键接口：

- `RelevantMemory`
- `findRelevantMemories()`

### 8.1 职责

这层不是底层存储接口，但它是**存储消费侧的高级查询接口**。

当前流程大致是：

1. `scanMemoryFiles()` 扫描本地主题文件
2. `formatMemoryManifest()` 构造 manifest
3. `findRelevantMemories()` 从 manifest 中挑选当前 query 最相关的记忆

### 8.2 对抽象层的启发

这说明 `MemoryManager` 不能只停留在 CRUD。  
因为对 memory 系统来说，真正高频的上层诉求不是“按主键查一条”，而是：

- 给定当前 query，找出最有价值的记忆
- 支持排除已经 surfaced 的记忆
- 支持结合 recent tools 做 recall 降噪

所以 `findRelevantMemories()` 应该保留在 `MemoryManager` 中，作为**高级查询接口**，而不是只留在 file-based 工具函数里。

## 九、自动提取 / 写入接口：当前的写入编排层

核心文件：`src/services/extractMemories/extractMemories.ts`

关键接口：

- `createAutoMemCanUseTool(memoryDir)`
- `initExtractMemories()`
- `executeExtractMemories()`
- `drainPendingExtraction()`

内部关键辅助逻辑：

- `hasMemoryWritesSince(...)`
- `extractWrittenPaths(...)`
- `countModelVisibleMessagesSince(...)`

### 9.1 触发职责

这一层是 auto memory 的**写入编排核心**。

它负责：

- 在一轮对话结束后触发自动提取
- 只处理上次提取之后新增的消息
- 先扫描现有 memory 文件
- 再让后台 agent 决定要新建或更新哪些主题文件
- 最后写回主题文件与 `MEMORY.md`

### 9.2 写入权限边界

`createAutoMemCanUseTool(memoryDir)` 定义了提取子 Agent 的写权限边界：

- 可用：`Read` / `Grep` / `Glob`
- Bash 基本只允许只读行为
- `Edit` / `Write` 仅允许落在 auto memory 目录内

因此，它实际上是一个**memory 文件写入安全接口**。

### 9.3 避免重复写入

`hasMemoryWritesSince(...)` 的作用是：

- 如果主 Agent 当前轮已经直接写过 memory
- 那就跳过后台 extraction

这样可以避免：

- 主 Agent 和提取 Agent 双写
- 同一轮对话生成重复 memory 文件

### 9.4 增量提取

`countModelVisibleMessagesSince(...)` 体现了它的增量策略：

- 不是每次回看整个历史
- 而是只提取上次成功提取之后新增的可见消息

### 9.5 对抽象层的启发

这里最重要的结论是：

- `extractMemories` 不是 `MemoryManager` 本身
- 它更像是**MemoryManager 的上游编排器**
- 它应该依赖 `MemoryManager` 提供的 CRUD / query / manifest 能力
- 而不是直接依赖 `scanMemoryFiles()`、`formatMemoryManifest()` 这类文件函数

换句话说，后续更理想的依赖方向应是：

```text
extractMemories orchestration -> MemoryManager abstraction -> concrete backend
```

而不是：

```text
extractMemories orchestration -> file helpers directly
```

## 十、Prompt 侧的“存储规范接口”

核心文件：`src/services/extractMemories/prompts.ts`

虽然这里不属于 TypeScript interface，但它实际上定义了模型在写 memory 时必须遵守的文件规范：

- 先写主题文件
- 再更新 `MEMORY.md`
- `MEMORY.md` 只做索引，不存主题正文
- 优先更新已有主题，而不是无条件新建文件
- 主题文件按语义组织，而不是简单按时间堆积

因此，这套系统的“文件存储协议”并不是只靠类型系统来约束，而是：

- **代码限制路径与权限**
- **prompt 约束文件格式与更新策略**

这也说明，在 `FileMemoryManager` 中，部分约束会是：

- 存储层规则
- 业务层约束
- agent prompt 约束

三者共同组成的结果。

## 十一、建议抽出的统一领域模型

如果要把当前实现抽成 `MemoryManager`，建议先统一几类领域对象，而不是直接暴露文件路径和 frontmatter。

### 11.1 `MemoryIdentity`

用于标识一条 memory 的身份。

建议：

- 通用层只要求它能唯一定位一条 memory
- file-based 实现可用 `path` 或派生 slug 表达
- DB 实现可用 `id` / `uuid`

示意：

```ts
type MemoryIdentity = {
  id: string
}
```

### 11.2 `MemoryMetadata`

用于承载通用元信息。

```ts
type MemoryMetadata = {
  type: MemoryType
  name: string
  description: string | null
  createdAt?: string
  updatedAt?: string
  source?: 'auto_extract' | 'manual' | 'migration'
  tags?: string[]
}
```

### 11.3 `MemoryRecord`

表示一条完整 memory 记录。

```ts
type MemoryRecord = {
  identity: MemoryIdentity
  metadata: MemoryMetadata
  content: string
}
```

### 11.4 `MemoryQuery`

用于列表和过滤。

```ts
type MemoryQuery = {
  type?: MemoryType
  keyword?: string
  updatedAfterMs?: number
  updatedBeforeMs?: number
  limit?: number
}
```

### 11.5 `CreateMemoryInput` / `UpdateMemoryInput`

用于增删改场景。

```ts
type CreateMemoryInput = {
  metadata: MemoryMetadata
  content: string
  preferredSlug?: string
}

type UpdateMemoryInput = {
  metadata?: Partial<MemoryMetadata>
  content?: string
}
```

### 11.6 `RecallQuery` / `MemorySearchResult`

用于 recall 查询。

```ts
type RecallQuery = {
  query: string
  recentTools?: readonly string[]
  alreadySurfacedIds?: ReadonlySet<string>
  limit?: number
}

type MemorySearchResult = {
  record: MemoryRecord
  score?: number
  reason?: string
}
```

## 十二、建议的 `MemoryManager` 抽象类

### 12.1 设计目标

这个抽象类的目的，不是把所有后端强行做成一样，而是统一**上层真正关心的能力**：

- 记忆的增删改查
- 列表与筛选
- recall 查询
- manifest / index 维护

### 12.2 推荐接口

```ts
abstract class MemoryManager {
  abstract createMemory(input: CreateMemoryInput): Promise<MemoryRecord>

  abstract getMemory(identity: MemoryIdentity): Promise<MemoryRecord | null>

  abstract updateMemory(
    identity: MemoryIdentity,
    patch: UpdateMemoryInput,
  ): Promise<MemoryRecord>

  abstract deleteMemory(identity: MemoryIdentity): Promise<void>

  abstract listMemories(query?: MemoryQuery): Promise<MemoryRecord[]>

  abstract findRelevantMemories(
    query: RecallQuery,
  ): Promise<MemorySearchResult[]>

  abstract buildManifest(memories?: MemoryRecord[]): Promise<string>

  abstract rebuildIndex(): Promise<void>
}
```

### 12.3 为什么用 `abstract class` 而不是纯 `interface`

采用 `abstract class MemoryManager` 的理由是：

- 后续不同实现很可能共享一部分默认流程
- 比如“先 list -> 再 buildManifest -> 再 recall”的流程可以沉淀成基类方法
- 也可能共享一些输入校验、类型约束、默认 limit、日志打点

因此，抽象类比纯接口更适合当前场景。

### 12.4 哪些职责属于抽象类，哪些属于具体实现

建议划分如下：

- **抽象类负责**
  - 统一 API 形态
  - 统一领域模型
  - 统一默认流程
  - 统一错误语义
- **具体实现负责**
  - 文件路径解析 / 表结构映射
  - frontmatter / SQL row 序列化
  - `MEMORY.md` 或 DB index 的维护
  - 具体 recall 候选集的构造方式

## 十三、`FileMemoryManager` 应该如何映射当前仓库

如果基于当前仓库落地一个 `FileMemoryManager`，它应当不是全新发明，而是把现有函数能力收拢到一个类里。

### 13.1 能力映射表

建议映射如下：

- `getAutoMemPath()` / `getAutoMemEntrypoint()`
  - 映射到 `FileMemoryManager` 的目录定位能力
- `ensureMemoryDirExists()`
  - 映射到初始化 / 写入前准备
- `scanMemoryFiles()`
  - 映射到 `listMemories()` 的底层候选获取
- `formatMemoryManifest()`
  - 映射到 `buildManifest()`
- `findRelevantMemories()`
  - 映射到 `findRelevantMemories()` 的 file-based 实现
- prompt 里的 frontmatter 约束
  - 映射到 `createMemory()` / `updateMemory()` 的序列化规则
- `MEMORY.md` 更新逻辑
  - 映射到 `rebuildIndex()`

### 13.2 `FileMemoryManager` 中哪些概念是实现细节

这些都应当只存在于 `FileMemoryManager` 内部，不应污染通用抽象：

- 主题文件路径
- `.md` 扩展名
- frontmatter 解析与写回
- `MEMORY.md`
- 目录递归扫描
- `mtimeMs`

### 13.3 `FileMemoryManager` 的职责边界

建议它负责：

- 路径解析
- memory 文件读写
- frontmatter 与正文互转
- list / query / manifest
- 入口索引文件 `MEMORY.md` 的维护

但不建议它负责：

- 对话结束时何时触发提取
- 是否跳过当前轮提取
- 叉出 agent 做总结

这些更适合留在 `extractMemories` 一类编排层里。

## 十四、`FileMemoryManager` 伪代码

下面的伪代码不是最终实现，而是一个**贴近当前仓库结构**的设计草图。

### 14.1 类结构

```ts
abstract class MemoryManager {
  abstract createMemory(input: CreateMemoryInput): Promise<MemoryRecord>
  abstract getMemory(identity: MemoryIdentity): Promise<MemoryRecord | null>
  abstract updateMemory(
    identity: MemoryIdentity,
    patch: UpdateMemoryInput,
  ): Promise<MemoryRecord>
  abstract deleteMemory(identity: MemoryIdentity): Promise<void>
  abstract listMemories(query?: MemoryQuery): Promise<MemoryRecord[]>
  abstract findRelevantMemories(
    query: RecallQuery,
  ): Promise<MemorySearchResult[]>
  abstract buildManifest(memories?: MemoryRecord[]): Promise<string>
  abstract rebuildIndex(): Promise<void>
}

class FileMemoryManager extends MemoryManager {
  constructor(
    private readonly memoryDir: string,
    private readonly entrypointPath: string,
  ) {
    super()
  }
}
```

### 14.2 初始化与路径解析

```ts
class FileMemoryManager extends MemoryManager {
  static createFromCurrentProject() {
    return new FileMemoryManager(getAutoMemPath(), getAutoMemEntrypoint())
  }

  private async ensureReady(): Promise<void> {
    await ensureMemoryDirExists(this.memoryDir)
  }

  private resolveTopicPath(input: CreateMemoryInput): string {
    const slug = sanitizeMemorySlug(input.preferredSlug ?? input.metadata.name)
    return join(this.memoryDir, `${slug}.md`)
  }
}
```

这一层直接复用当前 `paths.ts` 和 `memdir.ts` 的职责。

### 14.3 `createMemory()` 伪代码

```ts
async createMemory(input: CreateMemoryInput): Promise<MemoryRecord> {
  await this.ensureReady()

  const filePath = this.resolveTopicPath(input)
  const content = this.serializeMemoryFile(input)

  await writeFile(filePath, content)
  await this.rebuildIndex()

  return {
    identity: { id: filePath },
    metadata: input.metadata,
    content: input.content,
  }
}
```

这对应当前系统中的：

- 先确定目录
- 再写主题文件
- 最后更新 `MEMORY.md`

### 14.4 `getMemory()` 伪代码

```ts
async getMemory(identity: MemoryIdentity): Promise<MemoryRecord | null> {
  const filePath = identity.id
  if (!(await exists(filePath))) {
    return null
  }

  const raw = await readFile(filePath)
  return this.parseMemoryFile(filePath, raw)
}
```

在 file-based 实现里，`identity.id` 可以先直接落成 path；  
未来若要收敛，也可以改成内部 id 到 path 的映射。

### 14.5 `updateMemory()` 伪代码

```ts
async updateMemory(
  identity: MemoryIdentity,
  patch: UpdateMemoryInput,
): Promise<MemoryRecord> {
  const current = await this.getMemory(identity)
  if (!current) {
    throw new Error('Memory not found')
  }

  const next: MemoryRecord = {
    ...current,
    metadata: {
      ...current.metadata,
      ...patch.metadata,
    },
    content: patch.content ?? current.content,
  }

  await writeFile(identity.id, this.serializeRecord(next))
  await this.rebuildIndex()
  return next
}
```

这和当前 prompt 里的要求一致：

- 优先更新已有主题
- 不要无条件新建重复 memory

### 14.6 `deleteMemory()` 伪代码

```ts
async deleteMemory(identity: MemoryIdentity): Promise<void> {
  await removeFile(identity.id)
  await this.rebuildIndex()
}
```

对于 file-based 实现，删除的副作用除了删主题文件，还包括同步刷新 `MEMORY.md`。

### 14.7 `listMemories()` 伪代码

```ts
async listMemories(query?: MemoryQuery): Promise<MemoryRecord[]> {
  await this.ensureReady()

  const headers = await scanMemoryFiles(this.memoryDir, AbortSignal.timeout(5000))
  const filtered = this.filterHeaders(headers, query)

  return Promise.all(
    filtered.map(header =>
      this.getMemory({ id: header.filePath }),
    ),
  ).then(records => records.filter(Boolean) as MemoryRecord[])
}
```

这个版本保留了当前实现的核心策略：

- 先扫轻量 header
- 再按需要 hydrate 成完整记录

### 14.8 `buildManifest()` 伪代码

```ts
async buildManifest(memories?: MemoryRecord[]): Promise<string> {
  const records = memories ?? (await this.listMemories())
  const headers = records.map(record => ({
    filename: this.identityToFilename(record.identity),
    filePath: record.identity.id,
    mtimeMs: this.getUpdatedAtMs(record),
    description: record.metadata.description,
    type: record.metadata.type,
  }))

  return formatMemoryManifest(headers)
}
```

这里可以直接复用当前 `formatMemoryManifest()` 的文本协议。

### 14.9 `findRelevantMemories()` 伪代码

```ts
async findRelevantMemories(
  query: RecallQuery,
): Promise<MemorySearchResult[]> {
  const relevant = await findRelevantMemories(
    query.query,
    this.memoryDir,
    AbortSignal.timeout(5000),
    query.recentTools ?? [],
    query.alreadySurfacedIds ?? new Set(),
  )

  const records = await Promise.all(
    relevant.map(item => this.getMemory({ id: item.path })),
  )

  return records
    .filter(Boolean)
    .map(record => ({
      record: record as MemoryRecord,
    }))
}
```

注意这里的名字冲突问题：  
真实落代码时，建议把现有函数重命名为更底层的 `findRelevantFileMemories()`，避免和类方法同名。

### 14.10 `rebuildIndex()` 伪代码

```ts
async rebuildIndex(): Promise<void> {
  const memories = await this.listMemories()
  const lines = memories.map(memory => {
    const filename = this.identityToFilename(memory.identity)
    const title = memory.metadata.name
    const hook = memory.metadata.description ?? memory.metadata.type
    return `- [${title}](${filename}) — ${hook}`
  })

  await writeFile(this.entrypointPath, lines.join('\n') + '\n')
}
```

这一步体现的是 file-based 的特有职责：

- 通用层只关心“索引重建”
- file 实现负责把索引投影成 `MEMORY.md`

### 14.11 典型 upsert 流程伪代码

这是最贴近当前 `extractMemories` 编排层的一个流程：

```ts
async upsertMemoryBySemanticKey(input: CreateMemoryInput): Promise<MemoryRecord> {
  const existing = await this.listMemories({
    type: input.metadata.type,
    keyword: input.metadata.name,
    limit: 5,
  })

  const matched = this.pickBestSemanticMatch(existing, input)
  if (matched) {
    return this.updateMemory(matched.identity, {
      metadata: input.metadata,
      content: this.mergeMemoryContent(matched.content, input.content),
    })
  }

  return this.createMemory(input)
}
```

这个 `upsert` 思路与当前 prompt 约束完全一致：

- 优先更新已有主题
- 减少重复 memory
- 保持语义聚合，而不是按时间无限追加新文件

## 十五、`DatabaseMemoryManager` 会如何不同

如果未来接数据库，抽象层大体不变，但底层实现会明显不同：

- `identity.id` 不再是文件路径，而是主键 / UUID
- `metadata` 不再写 frontmatter，而是映射到表字段
- `content` 进入 text/blob 列
- `listMemories()` 不再依赖目录扫描，而是 SQL / 查询引擎
- `findRelevantMemories()` 可以走全文索引、向量检索或混合召回
- `rebuildIndex()` 甚至可以是空操作，或改成更新一张 materialized view

所以，抽象层里真正稳定的是：

- CRUD 语义
- recall 语义
- manifest / index 的高阶能力定义

而不是文件格式本身。

## 十六、推荐的迁移方式

如果后续要在当前仓库里真正落地这套抽象，建议分阶段进行。

### 16.1 第一阶段：只抽象，不改行为

- 新增 `MemoryManager` 抽象类和领域类型
- 新增 `FileMemoryManager`
- 在类内部复用当前 `paths.ts`、`memoryScan.ts`、`findRelevantMemories.ts` 的函数
- 保持外部行为不变

### 16.2 第二阶段：让编排层依赖抽象

- 让 `extractMemories.ts` 不再直接依赖 `scanMemoryFiles()` / `formatMemoryManifest()`
- 改为依赖 `MemoryManager`
- 这样后续切换后端时，编排层不需要重写

### 16.3 第三阶段：替换底层实现

- 补 `DatabaseMemoryManager`
- 在配置层决定注入哪种实现
- 保持上层 `extractMemories`、prompt、recall 入口稳定

## 十七、最终建议

如果只想记住最核心的一句话，可以概括为：

**当前仓库并不缺“记忆能力”，缺的是把这些能力从 file protocol 中抽离出来的 `MemoryManager` 抽象。**

更具体地说：

- `src/memdir/paths.ts` 负责**文件实现的路径定位**
- `src/memdir/memdir.ts` 负责**文件实现的存储协议**
- `src/memdir/memoryTypes.ts` 负责**通用的记忆语义分类**
- `src/memdir/memoryScan.ts` 提供**列表与索引候选能力**
- `src/memdir/findRelevantMemories.ts` 提供**recall 查询能力**
- `src/services/extractMemories/extractMemories.ts` 提供**自动提取编排能力**

后续最合理的演进方式就是：

- 抽象出 `abstract class MemoryManager`
- 用 `FileMemoryManager` 吸收当前 Markdown 存储实现
- 让未来的 `DatabaseMemoryManager` 复用同一组上层接口

这样才能真正做到：

**记忆系统的上层逻辑稳定，而底层存储介质可替换。**
