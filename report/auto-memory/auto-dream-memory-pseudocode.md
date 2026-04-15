# Auto Dream 记忆整理并写入 `MEMORY.md` 伪代码

本文把 `autoDream` 对记忆的**整理、提炼、去重、回写 `MEMORY.md`** 的过程，抽象成一份便于理解的伪代码。

目标不是逐行复刻源码，而是提炼出真实实现中的**关键调度逻辑**与**整理逻辑**。

## 一、流程总览

在当前仓库里，这条链路可以拆成两层：

- **调度层**：决定要不要触发一次 `autoDream`
- **整理层**：真正读取 logs / 既有 memory，整理后写回主题文件和 `MEMORY.md`

可以概括为：

1. 判断功能是否开启
2. 判断距离上次 consolidation 是否足够久
3. 判断是否累积了足够多的新 session
4. 尝试获取 consolidation 锁，避免并发整理
5. 启动 forked agent 执行 `/dream` 式整理
6. 读取 daily logs、现有 memory files、必要时读取 transcripts
7. 把新信息归并到主题 memory files
8. 更新 `MEMORY.md` 作为索引入口

## 二、调度层伪代码

这一层对应 `src/services/autoDream/autoDream.ts` 的职责。

```text
function runAutoDream(context):
    cfg = getConfig()  // 默认 minHours=24, minSessions=5
                       // 含义：至少隔一段时间、且积累足够多会话后才做一次重整理

    if not isAutoDreamEnabled():
        return         // 总开关未开，直接跳过

    if kairosActive():
        return         // Assistant/KAIROS 模式不走这里，改由单独的 disk-skill dream 处理

    if isRemoteMode():
        return         // 远端模式下不在这里做本地 consolidation

    if not isAutoMemoryEnabled():
        return         // 连 auto memory 都没开，就没有后续整理基础

    lastConsolidatedAt = readLastConsolidatedAt()
    hoursSince = now() - lastConsolidatedAt
                // 这里读的是上次 consolidation 的时间戳，不是上次对话时间

    if hoursSince < cfg.minHours:
        return         // 时间间隔太短，避免每轮都做重型整理

    if sessionScanTooFrequent():
        return         // 即便时间 gate 已过，也要限制 session 扫描频率，避免频繁 IO

    sessionIds = listSessionsTouchedSince(lastConsolidatedAt)
    sessionIds = excludeCurrentSession(sessionIds)
                // 当前会话总是“新”的，排除它可以避免误触发

    if count(sessionIds) < cfg.minSessions:
        return         // 新增会话样本还不够，暂时不值得做一次 consolidation

    priorMtime = tryAcquireConsolidationLock()
    if priorMtime == null:
        return         // 说明已有别的进程在做 consolidation，避免并发覆盖

    taskId = registerDreamTask(sessionCount = count(sessionIds))
            // 注册后台任务，便于 UI/任务面板展示进度和结果

    try:
        memoryRoot = getAutoMemPath()
        transcriptDir = getProjectTranscriptDir()
        // memoryRoot: 记忆目录根路径
        // transcriptDir: 会话 transcript 存放目录，仅在必要时做窄搜索

        prompt = buildConsolidationPrompt(
            memoryRoot,
            transcriptDir,
            extraContext = sessionIds
        )
        // prompt 不直接实现整理，而是把整理规则、目标和边界交给 forked agent

        result = runForkedAgent(
            prompt = prompt,
            canUseTool = createAutoMemCanUseTool(memoryRoot),
            querySource = "auto_dream"
        )
        // forked agent 负责实际读取 / 合并 / 写回
        // canUseTool 限制了它只能在 memoryRoot 内安全写入，并只做只读探索命令

        completeDreamTask(taskId)

        if result.touchedFiles is not empty:
            appendSystemMessage("Improved memory files")
            // 这里只回报“记忆有被整理过”，不把全部细节塞回主对话

    catch error:
        failDreamTask(taskId)
        rollbackConsolidationLock(priorMtime)
        // 失败时把锁时间戳回滚，避免一次异常导致后续长时间都无法重新触发
```

## 三、整理层伪代码

这一层对应 `buildConsolidationPrompt()` 驱动的 dream 行为，也就是 forked agent 真正在做的事。

```text
function consolidateMemories(memoryRoot, transcriptDir):
    ensure memoryRoot exists
    // 如果记忆目录还不存在，先创建；否则后续扫描和写入都会失败

    // Phase 1: Orient
    existingFiles = listFiles(memoryRoot)
    entrypoint = read(memoryRoot + "/MEMORY.md")
    topicFiles = listTopLevelMemoryFiles(excluding = ["MEMORY.md", "logs/", "sessions/"])
    existingTopics = skim(topicFiles)
    // 先看已有主题，目标是“增量改进”而不是“从零重写”
    // 这样能减少重复文件，也能延续已存在的主题组织方式

    if logs directory exists:
        recentLogs = readRecentDailyLogs(memoryRoot + "/logs/")
    else:
        recentLogs = []
    // daily logs 是 Assistant Mode 下的原始观察流，优先级高于 transcript 全量翻阅

    if sessions directory exists:
        recentSessionNotes = readRecentSessionArtifacts()
    else:
        recentSessionNotes = []
    // 这里的 session artifacts 可以理解为补充材料，不一定总会使用

    // Phase 2: Gather signal
    candidateFacts = []
    candidateFacts += extractImportantSignals(recentLogs)
    candidateFacts += detectMemoryDrift(existingTopics, codebaseOrKnownFacts)
    // 一部分候选信息来自“新日志”，另一部分来自“旧记忆已经漂移”
    // autoDream 不只是新增记忆，也承担修正旧记忆的职责

    if some fact still lacks context:
        candidateFacts += grepNarrowTranscriptHints(transcriptDir)
        // transcript 只做窄范围补证，不做全文重读
        // 否则成本太高，也容易把噪音重新引入 memory

    candidateFacts = filterWorthRemembering(candidateFacts)
    candidateFacts = normalizeDates(candidateFacts)         // 相对时间 -> 绝对时间，避免过几天后失去可读性
    candidateFacts = removeNoiseAndDuplicates(candidateFacts)
    // 这一步是在把“原始观察”变成“长期可复用事实”

    // Phase 3: Consolidate into topic files
    for each fact in candidateFacts:
        topic = classifyMemoryType(fact)                    // user / feedback / project / reference
        targetFile = findBestExistingTopicFile(fact, existingTopics)
        // 优先找最合适的已有主题文件，而不是轻易创建新文件

        if targetFile exists:
            mergedContent = mergeFactIntoTopicFile(targetFile, fact)
            mergedContent = deleteContradictedStatements(mergedContent)
            write(targetFile, mergedContent)
            // 核心思想：更新旧主题 + 去掉被证伪内容，而不是简单 append
        else:
            newFile = createTopicFilename(topic, fact)
            newContent = buildMemoryFileWithFrontmatter(
                type = topic,
                name = summarizeTitle(fact),
                description = summarizeDescription(fact),
                body = organizeIntoDurableSections(fact)
            )
            write(memoryRoot + "/" + newFile, newContent)
            // 新建文件时要直接落成“主题化结构”，而不是把日志原文搬过去

    // Phase 4: Rebuild index
    latestTopicFiles = scanTopLevelMemoryFiles(memoryRoot)
    validEntries = []
    // 到这里，真正的知识已经在 topic files 中；接下来只是重建入口索引

    for each file in latestTopicFiles:
        if file is stale or contradicted and should be removed:
            skip
        else:
            validEntries.append(
                buildIndexLine(file)   // - [Title](file.md) — one-line hook
            )
            // 每一行只保留“指针 + 一句话钩子”，不要把正文塞进 MEMORY.md

    validEntries = deduplicate(validEntries)
    validEntries = shortenVerboseEntries(validEntries)
    validEntries = keepEntrypointSmall(validEntries)       // 行数、字节数受限
    // MEMORY.md 是给模型快速定向的目录页，必须轻量，否则反而损害 recall 效率

    write(memoryRoot + "/MEMORY.md", join(validEntries, "\n"))

    return summaryOfChanges()
    // 最终返回简短摘要，告诉主流程本次整理改了什么
```

## 四、面向 Assistant Mode 的完整伪代码

如果把 `Assistant Mode` 里的 daily log 也算进来，那么完整链路更接近下面这样：

```text
while conversation is running:
    if assistant notices long-term-useful information:
        append bullet with timestamp into logs/YYYY/MM/YYYY-MM-DD.md
        // 前台会话阶段只做“采集”，不急着实时维护 topic files 或 MEMORY.md

periodically:
    runAutoDream()
    // 后台定期做“整理”，把采集和提炼分离开

function runAutoDream():
    if trigger gates not satisfied:
        return

    acquire consolidation lock
    // 防止多个 dream 同时改同一批 memory 文件

    recentLogs = read daily logs since last consolidation
    currentIndex = read MEMORY.md
    existingTopicFiles = scan existing memory topics
    // 一边看新日志，一边看当前索引和历史主题，确保是“增量整合”而不是盲写

    distilledFacts = distill(recentLogs, existingTopicFiles)
    distilledFacts = dedupe(distilledFacts)
    distilledFacts = fixDriftAndContradictions(distilledFacts)
    // distilledFacts 才是适合长期保留的稳定事实，不等于原始日志内容

    for each distilledFact:
        update existing topic file or create a new one
        // 输出层永远优先落到 topic files，而不是直接展开写进 MEMORY.md

    MEMORY.md = rebuild concise index from all valid topic files
    // MEMORY.md 始终只承担索引职责

    mark consolidation completed
    // 记录本次整理时间，为下一轮时间 gate 提供依据
```

## 五、可以把它理解成什么

`autoDream` 不是“直接把日志拼接进 `MEMORY.md`”，而是一个**分层整理器**：

- **daily log**：原始观察流，append-only
- **topic memory files**：整理后的长期主题知识
- **`MEMORY.md`**：只保留导航索引

所以它真正做的是：

- 从原始日志里抽取长期有效的信息
- 合并进已有主题，而不是制造重复文件
- 删除或修正已经漂移的旧记忆
- 最后重建一个简洁的 `MEMORY.md`

## 六、最精简版本

如果只保留一句最核心的伪代码，可以写成：

```text
collect recent logs -> distill durable facts -> merge into topic memory files -> prune contradictions -> rebuild concise MEMORY.md index
```

## 七、对应源码锚点

- 调度入口：`src/services/autoDream/autoDream.ts`
- 锁与时间戳：`src/services/autoDream/consolidationLock.ts`
- 整理提示词：`src/services/autoDream/consolidationPrompt.ts`
- memory 组织协议：`src/memdir/memdir.ts`
- memory 类型：`src/memdir/memoryTypes.ts`
