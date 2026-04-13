# Auto Memory 本地文件存储相关接口总结

本文聚焦当前仓库中 `auto memory` 的**本地文件存储**相关接口与调用链，只讨论：

- 路径解析与落盘目录
- `MEMORY.md` 与主题文件的存储协议
- 本地扫描 / 读取接口
- 自动提取与写入接口

本文**不包含** `team memory` 的 pull / push / watch 同步逻辑。

## 一、先说结论

`auto memory` 并没有单独抽象出一个 `Storage` interface 或 `FileStore` 类。它的“文件存储接口”是由几组导出函数和类型共同组成的：

1. **路径层**：决定 memory 存到哪里
2. **协议层**：定义 `MEMORY.md` 和主题文件如何组织
3. **读取层**：扫描已有 memory 文件并构造清单
4. **写入层**：在对话结束后自动提取并写回文件

从代码结构上看，最值得关注的文件是：

- `src/memdir/paths.ts`
- `src/memdir/memdir.ts`
- `src/memdir/memoryTypes.ts`
- `src/memdir/memoryScan.ts`
- `src/memdir/findRelevantMemories.ts`
- `src/services/extractMemories/extractMemories.ts`
- `src/services/extractMemories/prompts.ts`

## 二、路径相关 interface

核心文件：`src/memdir/paths.ts`

主要接口：

- `isAutoMemoryEnabled()`
- `isExtractModeActive()`
- `getMemoryBaseDir()`
- `getAutoMemPath()`
- `getAutoMemDailyLogPath(date?)`
- `getAutoMemEntrypoint()`
- `isAutoMemPath(absolutePath)`

### 2.1 职责

这组接口负责统一解析 auto memory 的文件系统位置：

- 是否启用 auto memory
- memory 根目录在哪
- 当前项目对应的 memory 目录在哪
- `MEMORY.md` 的绝对路径是什么
- 某个路径是否属于 auto memory 目录

### 2.2 默认目录结构

默认情况下，auto memory 的目录形态大致是：

```text
<memoryBase>/projects/<sanitized-git-root>/memory/
```

也就是说，它是**按项目隔离存储**的。

如果处于 Assistant / Kairos 相关模式，还会用到按日期组织的日志路径：

```text
<autoMemPath>/logs/YYYY/MM/YYYY-MM-DD.md
```

### 2.3 这一层的意义

如果把 auto memory 看成文件系统上的一套协议，那么 `paths.ts` 就是它的**路径入口层**：

- 上层逻辑不需要自己拼目录
- 写入逻辑只要拿到 `getAutoMemPath()` 或 `getAutoMemEntrypoint()` 即可
- 权限控制也可基于 `isAutoMemPath()` 做边界判断

## 三、存储协议相关 interface

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

### 3.1 核心约定

这一层定义了 auto memory 的**文件组织协议**：

- 入口文件固定为 `MEMORY.md`
- 真正的记忆内容不全堆在 `MEMORY.md` 中
- 每个主题记忆以单独 Markdown 文件存储
- `MEMORY.md` 只承担“索引入口”的职责

可以简单理解为：

- **主题文件**：存正文
- **`MEMORY.md`**：存导航

### 3.2 目录保证

`ensureMemoryDirExists(memoryDir)` 的职责很直接：

- 确保 memory 目录存在
- 后续写主题文件和更新 `MEMORY.md` 时不需要重复创建目录

### 3.3 入口文件截断

`truncateEntrypointContent(raw)` 用于限制 `MEMORY.md` 被加载进上下文时的体积，避免：

- 行数过多
- 字节数过大
- 对主 prompt 造成上下文压力

这说明 `MEMORY.md` 不是“越大越好”，它更像一个**给模型快速浏览的目录页**。

### 3.4 Prompt 入口

`loadMemoryPrompt()` 会把 auto memory 的使用方式注入系统 prompt，告诉主 Agent：

- memory 目录在哪里
- 如何使用 `MEMORY.md`
- 主题文件应如何组织

因此，它虽然不是直接的读写接口，但它决定了模型侧对存储协议的理解方式。

## 四、memory 类型相关 interface

核心文件：`src/memdir/memoryTypes.ts`

关键接口 / 常量：

- `MEMORY_TYPES`
- `MemoryType`
- `parseMemoryType(raw)`
- `MEMORY_FRONTMATTER_EXAMPLE`
- `TYPES_SECTION_INDIVIDUAL`
- `TYPES_SECTION_COMBINED`

### 4.1 支持的类型

当前主题文件的 `type` 主要有 4 类：

- `user`
- `feedback`
- `project`
- `reference`

### 4.2 这一层的职责

这组定义负责约束 memory 文件 frontmatter 中的类型字段，是主题文件的**元数据协议层**。

也就是说，主题文件不是随意命名、随意归类的，而是通过 `MemoryType` 进入一套有限的语义分类体系。

## 五、本地扫描 / 读取相关 interface

核心文件：`src/memdir/memoryScan.ts`

关键接口：

- `interface MemoryHeader`
- `scanMemoryFiles(memoryDir, signal)`
- `formatMemoryManifest(memories)`

### 5.1 `MemoryHeader`

`MemoryHeader` 表示系统扫描一个主题文件后提炼出的最小元信息集合，通常包括：

- `filename`
- `filePath`
- `mtimeMs`
- `description`
- `type`

它可以看作“memory 文件索引项”的类型定义。

### 5.2 `scanMemoryFiles(...)`

这个函数是最核心的本地读取接口之一，职责是：

- 扫描 memory 目录下的 `.md` 文件
- 排除 `MEMORY.md`
- 读取文件头部并解析 frontmatter
- 提取出可用于索引和筛选的元数据

这说明系统在消费记忆时，并不是每次都全文加载所有主题文件，而是优先读取**轻量级头部信息**。

### 5.3 `formatMemoryManifest(...)`

这个函数会把扫描结果整理成 manifest 文本，供后续 recall 或 extraction 阶段使用。

它的价值在于：

- 把分散在文件系统里的主题文件先压缩成“目录清单”
- 让模型先看摘要、再决定是否进一步使用某个主题文件

## 六、相关记忆选择 interface

核心文件：`src/memdir/findRelevantMemories.ts`

关键接口：

- `RelevantMemory`
- `findRelevantMemories()`

### 6.1 职责

这层不是底层存储接口，但它是**存储消费侧接口**。

大致流程是：

1. `scanMemoryFiles()` 扫描本地主题文件
2. `formatMemoryManifest()` 构造 manifest
3. `findRelevantMemories()` 从 manifest 中挑选当前 query 最相关的记忆

因此，它依赖底层文件存储层提供的“已落盘记忆文件清单”。

## 七、自动提取 / 写入相关 interface

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

### 7.1 触发职责

这一层是 auto memory 的**写入编排核心**。

它负责：

- 在一轮对话结束后触发自动提取
- 只处理上次提取之后新增的消息
- 先扫描现有 memory 文件
- 再让后台 agent 决定要新建或更新哪些主题文件
- 最后写回主题文件与 `MEMORY.md`

### 7.2 写入权限边界

`createAutoMemCanUseTool(memoryDir)` 很关键，它定义了 memory 提取子 Agent 的写权限边界：

- 可用：`Read` / `Grep` / `Glob`
- Bash 基本只允许只读行为
- `Edit` / `Write` 仅允许落在 auto memory 目录内

因此，它实际上是一个**memory 文件写入安全接口**。

### 7.3 避免重复写入

`hasMemoryWritesSince(...)` 的作用是：

- 如果主 Agent 当前轮已经直接写过 memory
- 那就跳过后台 extraction

这样可以避免：

- 主 Agent 和提取 Agent 双写
- 同一轮对话生成重复 memory 文件

### 7.4 增量提取

`countModelVisibleMessagesSince(...)` 体现了它的增量策略：

- 不是每次回看整个历史
- 而是只提取上次成功提取之后新增的可见消息

这使得 auto memory 更像一个**增量持久化系统**，而不是每轮全量重建。

## 八、Prompt 侧的“存储规范接口”

核心文件：`src/services/extractMemories/prompts.ts`

虽然这里不属于 TypeScript interface，但它实际上定义了模型在写 memory 时必须遵守的文件规范。

可以概括成几条：

- 先写主题文件
- 再更新 `MEMORY.md`
- `MEMORY.md` 只做索引，不存主题正文
- 优先更新已有主题，而不是无条件新建文件
- 主题文件按语义组织，而不是简单按时间堆积

因此，这套系统的“文件存储协议”并不是只靠类型系统来约束，而是：

- **代码限制路径与权限**
- **prompt 约束文件格式与更新策略**

## 九、整体调用链

### 9.1 读链路

主链路可以概括为：

1. `loadMemoryPrompt()` 把 memory 使用方式注入系统 prompt
2. 主 Agent 先消费 `MEMORY.md` 作为入口索引
3. 需要时通过 `scanMemoryFiles()` 获取主题文件头部信息
4. 再由 `findRelevantMemories()` 选出最相关的主题文件

也就是说，读取并不是“全量展开所有 memory 文件”，而是：

- 先看入口索引
- 再看清单
- 最后按需聚焦

### 9.2 写链路

写入链路大致是：

1. 一轮对话结束
2. `executeExtractMemories()` 启动后台提取
3. 检查当前轮是否已经写过 memory
4. `scanMemoryFiles()` 扫描已有主题文件
5. `formatMemoryManifest()` 构造 manifest
6. 后台 agent 提取新增记忆并归类
7. 写入主题文件
8. 更新 `MEMORY.md`

因此，它是一个典型的：

**对话结束 -> 增量提取 -> 文件落盘 -> 索引更新**

的流程。

## 十、最值得重点记住的 interface

如果只想抓主干，建议优先记住下面这些：

### 路径层

- `getAutoMemPath()`
- `getAutoMemEntrypoint()`
- `isAutoMemPath()`

### 协议层

- `ENTRYPOINT_NAME`
- `ensureMemoryDirExists()`
- `loadMemoryPrompt()`

### 类型层

- `MemoryType`
- `parseMemoryType()`

### 读取层

- `MemoryHeader`
- `scanMemoryFiles()`
- `formatMemoryManifest()`
- `findRelevantMemories()`

### 写入层

- `createAutoMemCanUseTool()`
- `executeExtractMemories()`

## 十一、一句话结论

`auto memory` 的本地文件存储接口，本质上就是：

- 用 `src/memdir/paths.ts` 决定落盘位置
- 用 `src/memdir/memdir.ts` 定义 `MEMORY.md + 主题文件` 协议
- 用 `src/memdir/memoryTypes.ts` 定义主题分类与 frontmatter 语义
- 用 `src/memdir/memoryScan.ts` 建立本地文件索引
- 用 `src/memdir/findRelevantMemories.ts` 做相关记忆选择
- 用 `src/services/extractMemories/extractMemories.ts` 完成自动提取与受限写入

换句话说，它不是“一个单独的存储类”，而是一整套围绕 Markdown 文件组织起来的本地持久化协议。
