# Claude Auto-Memory 路径算法整理（可移植实现指南）

本文总结当前代码库中 **auto-memory 默认路径** 的计算方式，并把关键的优先级、边界条件、安全约束整理成一份**可在其他项目中复刻**的实现指南。

## 结论

默认 auto-memory 目录的计算规则是：

```text
<memory-base>/projects/<sanitized-auto-mem-base>/memory/
```

其中：

- `<memory-base>`：通常对应 `~/.claude`
- `<sanitized-auto-mem-base>`：对 `autoMemBase` 做 `sanitizePath()` 后得到的安全目录名
- `autoMemBase`：
  - 优先使用 `findCanonicalGitRoot(getProjectRoot())`
  - 如果当前目录不在 git 仓库中，则回退到 `getProjectRoot()`

也就是：

```ts
const autoMemBase = findCanonicalGitRoot(getProjectRoot()) ?? getProjectRoot()
const memoryPath = join(memoryBaseDir, 'projects', sanitizePath(autoMemBase), 'memory')
```

补充：默认逻辑之前还存在 **override 优先级**（见下文），一旦命中 override，将直接返回指定目录，不再走默认拼接。

## 真实调用链

当前仓库中的核心调用链如下：

1. `getAutoMemPath()`
2. `getAutoMemBase()`
3. `findCanonicalGitRoot(getProjectRoot()) ?? getProjectRoot()`
4. `sanitizePath(autoMemBase)`
5. `join(getMemoryBaseDir(), 'projects', sanitized, 'memory')`

对应源码位置：

- `src/memdir/paths.ts`
- `src/utils/git.ts`
- `src/utils/sessionStoragePortable.ts`
- `src/bootstrap/state.ts`

## 规则拆解

### 1. 先判断是否有显式覆盖

在默认算法之前，源码先检查两类 override：

1. 环境变量：`CLAUDE_COWORK_MEMORY_PATH_OVERRIDE`
2. settings 中的 `autoMemoryDirectory`

如果任意一个存在且合法，则**直接使用该路径**，不会再走默认拼接逻辑。

如果你在另一个仓库里只想复现“默认路径算法”，可以先**忽略 override**，只实现默认分支；如果希望行为完全对齐，请实现相同的优先级链路。

#### 1.1 Override 优先级（从高到低）

- **环境变量全路径 override**：`CLAUDE_COWORK_MEMORY_PATH_OVERRIDE`
  - 语义：直接指定 *完整的 auto-memory 目录*（而不是 base dir），命中后 `getAutoMemPath()` 原样返回该目录。
  - 目的：在“远程/沙箱/多会话”场景让 memory 映射到稳定挂载点，避免每次会话 cwd 不同导致 key 变化。
- **设置项全路径 override**：`autoMemoryDirectory`
  - 注意：当前仓库实现中 **不允许** 从 `projectSettings`（通常是 `.claude/settings.json`）读取该值，避免恶意仓库把路径指向敏感目录；只信任 `policy/local/user` 等来源。

#### 1.2 Override 的安全校验（validateMemoryPath）

为了避免把 auto-memory 目录设置成危险的“过宽 allowlist 根”，实现会拒绝以下输入：

- **相对路径**（例如 `../foo`）
- **根路径/近根路径**（例如 `/`、`/a`）
- **Windows 盘符根**（例如 `C:\` 规范化后为 `C:`）
- **UNC 网络路径**（例如 `\\server\\share` 或 `//server/share`）
- **包含空字节**（`\0`）
- 对 settings 中的 `~/` 做用户友好展开，但拒绝 `~` / `~/` / `~/.` / `~/..` 等会展开到 home 或其父目录的形式

### 2. 计算 `autoMemBase`

算法：

```ts
const autoMemBase = findCanonicalGitRoot(getProjectRoot()) ?? getProjectRoot()
```

这里的关键点不是“当前 cwd”，而是：

- 使用 `getProjectRoot()` 作为项目身份基准
- 如果项目位于 git 仓库中，则优先取 **canonical git root**
- 这样多个 worktree 会共享同一个 auto-memory 目录

> 代码位置：`src/memdir/paths.ts` 的 `getAutoMemBase()`。

### 3. 对 `autoMemBase` 做 `sanitizePath()`

`sanitizePath()` 的核心逻辑：

- 把所有 **非字母数字字符** 替换成 `-`
- 如果结果长度不超过上限，则直接返回
- 如果超过上限，则：
  - 截断到固定长度
  - 追加一个 hash 后缀，避免冲突

### 4. 拼出最终目录

默认目录结构：

```text
~/.claude/projects/<sanitized-auto-mem-base>/memory/
```

其中 `memory` 是固定目录名。

## `findCanonicalGitRoot()`：worktree 归一到主仓根目录

`findCanonicalGitRoot()` 的目标是让同一仓库的不同 worktree 共享同一个 memory 目录（同一“项目身份”）。实现要点：

- 普通仓库：`.git` 是目录，直接返回 `gitRoot`（no-op）。
- worktree：`.git` 是文件，内容形如 `gitdir: <path>`。
  - 解析出 `worktreeGitDir`
  - 读取 `commondir` 找到共享的 `.git` 目录
  - 做结构校验（防攻击）：
    - `worktreeGitDir` 必须是 `<commonDir>/worktrees/<name>` 的直接子目录
    - `<worktreeGitDir>/gitdir` 必须反向指回 `<gitRoot>/.git`（并对 `gitRoot` 做 realpath）
  - canonical root 一般是 `dirname(commonDir)`（即主仓 working directory）
  - 对“bare-repo worktree”场景：当 `commonDir` 的 basename 不是 `.git` 时，用 `commonDir` 作为稳定身份

> 代码位置：`src/utils/git.ts` 的 `resolveCanonicalRoot()`。

## `sanitizePath()` 的精确定义

当前仓库里的实现语义如下：

```ts
function sanitizePath(name: string): string {
  const sanitized = name.replace(/[^a-zA-Z0-9]/g, '-')

  if (sanitized.length <= MAX_SANITIZED_LENGTH) {
    return sanitized
  }

  const hash = typeof Bun !== 'undefined'
    ? Bun.hash(name).toString(36)
    : simpleHash(name)

  return `${sanitized.slice(0, MAX_SANITIZED_LENGTH)}-${hash}`
}
```

### 行为说明

- 输入：任意路径字符串，例如：
  - `/Users/bytedance/Desktop/homyee/claude-code`
  - `/tmp/worktrees/feature-x`
- 输出：适合作为目录名的字符串，例如：
  - `-Users-bytedance-Desktop-homyee-claude-code`
  - `-tmp-worktrees-feature-x`

### 规则说明

- `/` 会变成 `-`
- `.` 会变成 `-`
- `:` 会变成 `-`
- 空格会变成 `-`
- 其他所有非 `[a-zA-Z0-9]` 的字符都会变成 `-`

### 长路径处理

如果 sanitize 后的字符串太长：

- 先保留前 `MAX_SANITIZED_LENGTH` 个字符
- 再拼接 `-<hash>`

这样可以同时满足：

- 路径长度可控
- 不同长路径不容易互相冲突

## `canonical git root` 的含义

这一步是为了让 **同一个仓库的不同 worktree** 映射到同一个 memory 目录。

### 普通仓库

如果当前目录是普通 git 仓库：

- `findCanonicalGitRoot()` 返回普通仓库根目录

例如：

```text
/Users/bytedance/Desktop/homyee/claude-code
```

### worktree

如果当前目录是 git worktree：

- `findGitRoot()` 先找到 worktree 目录
- 再通过 `.git` 文件中的 `gitdir:`
- 再读取 `commondir`
- 最终解析回主仓库对应的 canonical root

结果就是：

- **不同 worktree 不会各自生成一份 memory**
- 而是共享主仓库对应的 memory 目录

### 非 git 目录

如果当前目录不在 git 仓库中：

- `findCanonicalGitRoot()` 返回 `null`
- 回退到 `getProjectRoot()`

这样即使没有 git，也仍然能得到稳定目录。

## 可直接移植的实现示例

下面给出一份**不依赖 Bun**、适合在另一个仓库里复用的 TypeScript 示例。

你只需要根据自己的工程替换：

- `findCanonicalGitRoot()`
- `getProjectRoot()`
- `getMemoryBaseDir()`
- `simpleHash()`

如果你想更贴近当前仓库，`MAX_SANITIZED_LENGTH` 的值是 **200**，fallback hash 在 Node 路径下等价于：

```ts
function simpleHash(str: string): string {
  return Math.abs(djb2Hash(str)).toString(36)
}
```

也就是说，当前仓库对超长路径的处理并不是随便拼一个随机串，而是：

- 先 `replace(/[^a-zA-Z0-9]/g, '-')`
- 超长后截断到 **200**
- 再追加 `-` 和一个 **base36 的 djb2 hash**

```ts
import path from 'path'

const AUTO_MEM_DIRNAME = 'memory'
const MAX_SANITIZED_LENGTH = 200

function simpleHash(input: string): string {
  let hash = 0
  for (let i = 0; i < input.length; i++) {
    hash = (hash * 31 + input.charCodeAt(i)) | 0
  }
  return Math.abs(hash).toString(36)
}

export function sanitizePath(name: string): string {
  const sanitized = name.replace(/[^a-zA-Z0-9]/g, '-')
  if (sanitized.length <= MAX_SANITIZED_LENGTH) {
    return sanitized
  }
  return `${sanitized.slice(0, MAX_SANITIZED_LENGTH)}-${simpleHash(name)}`
}

export function getDefaultAutoMemoryPath(options: {
  projectRoot: string
  canonicalGitRoot: string | null
  memoryBaseDir: string
}): string {
  const autoMemBase = options.canonicalGitRoot ?? options.projectRoot
  return path.join(
    options.memoryBaseDir,
    'projects',
    sanitizePath(autoMemBase),
    AUTO_MEM_DIRNAME,
  )
}
```

## 推荐你在其他项目中保持一致的行为清单

- **身份基准**：优先 `canonical git root`，否则 `projectRoot`（不要用 cwd）。
- **目录结构**：固定 `projects/<key>/memory/`。
- **key 生成**：`sanitizePath()` 规则一致（`[^a-zA-Z0-9]` → `-`），长度上限 200，超长加 base36 hash 后缀。
- **override 行为**：如果你要对齐当前仓库，建议实现同样的 override + 安全校验（尤其是“排除 projectSettings 来源”）。

## 最小复刻版本

## 更贴近当前仓库语义的伪代码

```ts
function getAutoMemPath(): string {
  const override = getAutoMemPathOverride() ?? getAutoMemPathSetting()
  if (override) return override

  const autoMemBase = findCanonicalGitRoot(getProjectRoot()) ?? getProjectRoot()
  const key = sanitizePath(autoMemBase)

  return join(getMemoryBaseDir(), 'projects', key, 'memory')
}
```

## 例子

### 例 1：普通 git 仓库

输入：

```text
projectRoot = /Users/bytedance/Desktop/homyee/claude-code
canonicalGitRoot = /Users/bytedance/Desktop/homyee/claude-code
```

输出：

```text
~/.claude/projects/-Users-bytedance-Desktop-homyee-claude-code/memory/
```

### 例 2：worktree

输入：

```text
projectRoot = /Users/bytedance/Desktop/tmp/claude-code-feature-a
canonicalGitRoot = /Users/bytedance/Desktop/homyee/claude-code
```

输出仍然是：

```text
~/.claude/projects/-Users-bytedance-Desktop-homyee-claude-code/memory/
```

也就是说，worktree 会和主仓共享 memory。

### 例 3：非 git 目录

输入：

```text
projectRoot = /Users/bytedance/Desktop/sandbox/no-git-project
canonicalGitRoot = null
```

输出：

```text
~/.claude/projects/-Users-bytedance-Desktop-sandbox-no-git-project/memory/
```

## 迁移实现时的建议

如果你想在另一个仓库中尽可能对齐当前行为，建议至少保留以下几点：

- **优先用 canonical git root 而不是当前 cwd**
- **worktree 要映射到主仓 identity**
- **sanitize 规则统一使用 `[^a-zA-Z0-9] -> '-'`**
- **长路径要截断并追加 hash**
- **最终目录结构保持 `projects/<sanitized>/memory/`**

## 最小复刻版本

如果你只需要“足够像”的实现，而不必 100% 对齐当前仓库，可用下面这个最小版本：

```ts
const key = sanitizePath(findCanonicalGitRoot(projectRoot) ?? projectRoot)
const memoryPath = path.join(claudeHome, 'projects', key, 'memory')
```

## 对照源码位置

- `src/memdir/paths.ts`
  - `getAutoMemBase()`
  - `getAutoMemPath()`
- `src/utils/git.ts`
  - `findGitRoot()`
  - `findCanonicalGitRoot()`
  - `resolveCanonicalRoot()`
- `src/utils/sessionStoragePortable.ts`
  - `sanitizePath()`
- `src/bootstrap/state.ts`
  - `getProjectRoot()`

## 一句话总结

在当前代码库里，**默认 memory 路径算法**可以概括为：

```ts
join(memoryBaseDir, 'projects', sanitizePath(findCanonicalGitRoot(projectRoot) ?? projectRoot), 'memory')
```
