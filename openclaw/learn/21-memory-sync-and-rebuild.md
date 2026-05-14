# 21. Memory 同步阶段与重建过程

这篇文档解释 `memory_search` 背后的索引如何同步，以及什么时候会重建。

先记住一句话：

```text
Markdown / session transcript 是原始数据
SQLite / QMD index 是检索用的索引
sync 是把原始数据更新到索引
rebuild 是把索引整体重新建一遍
```

本文主要讲两个后端：

```text
builtin backend = 默认 SQLite 索引
QMD backend     = 外部 qmd sidecar 索引
```

相关入口：

- `extensions/memory-core/src/memory/manager.ts`：builtin `MemoryIndexManager`
- `extensions/memory-core/src/memory/manager-sync-ops.ts`：builtin 同步和重建主逻辑
- `extensions/memory-core/src/memory/manager-embedding-ops.ts`：单个文件如何切 chunk、embedding、写索引
- `extensions/memory-core/src/memory/qmd-manager.ts`：QMD 后端同步和 collection 修复逻辑
- `docs/concepts/memory-builtin.md:76`：官方文档里的 builtin indexing 概览
- `docs/concepts/memory-qmd.md:53`：官方文档里的 QMD indexing 概览

## 一、什么是同步

同步不是把 Markdown 文件整份塞进 prompt。

同步的目标是维护一个检索索引：

```text
MEMORY.md
memory/*.md
memorySearch.extraPaths
session transcripts（如果启用）
        |
        v
扫描文件、计算 hash、切 chunk、生成 embedding
        |
        v
SQLite tables / QMD collections
        |
        v
memory_search 查询索引
```

builtin 后端的索引表由 `ensureMemoryIndexSchema(...)` 建立：

- `packages/memory-host-sdk/src/host/memory-schema.ts:13`：`meta` 表，保存索引元信息
- `packages/memory-host-sdk/src/host/memory-schema.ts:19`：`files` 表，保存每个文件的 path/source/hash/mtime/size
- `packages/memory-host-sdk/src/host/memory-schema.ts:28`：`chunks` 表，保存 chunk 文本、行号、模型、embedding JSON
- `packages/memory-host-sdk/src/host/memory-schema.ts:43`：`embedding_cache`，可选 embedding 缓存
- `packages/memory-host-sdk/src/host/memory-schema.ts:66`：`chunks_fts`，FTS5 全文索引
- `extensions/memory-core/src/memory/manager-sync-ops.ts:287`：`chunks_vec`，sqlite-vec 向量表

所以同步阶段的本质是：

```text
让索引数据库和当前 workspace 的记忆文件保持一致
```

### SQLite tables 和 QMD collections 到底存了什么

你的理解大方向是对的，但要分 builtin SQLite 和 QMD 两种后端看。

builtin SQLite 后端里，数据库会保存“文件级记录”和“chunk 级记录”：

- `files` 表保存每个源文件的 `path/source/hash/mtime/size`，这里的 `hash` 是文件内容 hash，用来判断文件有没有变化。
- `chunks` 表保存切出来的 chunk：`path/source/start_line/end_line/hash/model/text/embedding/updated_at`。这里的 `text` 就是 chunk 文本，`hash` 是 chunk 内容 hash。
- `chunks_fts` 是全文检索索引，保存可被 FTS 查询的 chunk 文本和定位字段。
- `chunks_vec` 是 sqlite-vec 向量表，保存 chunk id 对应的向量。
- `embedding_cache` 保存 provider/model/provider_key/hash 到 embedding 的缓存，避免同一个 chunk 重复调用 embedding provider。

所以 builtin SQLite 不是只存“文件路径和 hash”，它确实会存 Markdown/session transcript 切出来的 chunk 文本、chunk hash、embedding、行号范围和检索辅助索引。

QMD backend 的表达要更谨慎。OpenClaw 不直接在自己的代码里创建 `chunks` 表，也不自己定义 QMD 内部 schema。它做的是：

```text
配置 managed collections
  -> 调 qmd collection add/remove
  -> 调 qmd update
  -> 需要语义检索时再调 qmd embed
```

也就是说，QMD collections 是由外部 `qmd` sidecar 管理的索引集合。OpenClaw 能通过 QMD index 的 `documents` 表查到 `collection/path/hash/active` 之类的文档定位信息，但 chunk 怎么落库、hash 怎么组织、向量怎么存，是 qmd 自己的内部实现，不是 OpenClaw 的 builtin SQLite schema。

因此更准确的说法是：

```text
builtin SQLite:
  OpenClaw 自己维护 files/chunks/chunks_fts/chunks_vec/embedding_cache。
  里面明确保存文件 hash、chunk 文本、chunk hash、embedding 和行号。

QMD collections:
  OpenClaw 维护 collection 配置和同步触发。
  实际文档索引、hash、embedding/chunk 索引由外部 qmd 管理。
```

同步的目标也可以理解成最终一致性：

```text
源文件是事实来源
SQLite / QMD index 是检索缓存

文件变更后，索引可能短暂落后
watcher / search bootstrap / interval / CLI index 会触发同步
同步完成后，索引追上源文件
```

所以它不是强实时一致。比如你刚改完一个 Markdown 文件时，本次搜索可能还命中旧索引；watch debounce 或下一次搜索触发的 sync 完成后，后续搜索才会看到新内容。代码里也体现了这个设计：搜索发现 dirty 时通常会启动后台同步，而不一定阻塞当前搜索。

## 二、什么时候会触发同步

### 1. Manager 初始化

builtin manager 创建时会：

- `extensions/memory-core/src/memory/manager.ts:222`：打开 SQLite 数据库
- `extensions/memory-core/src/memory/manager.ts:229`：确保 schema 存在
- `extensions/memory-core/src/memory/manager.ts:235`：读取旧的 meta
- `extensions/memory-core/src/memory/manager.ts:241`：启动文件 watcher
- `extensions/memory-core/src/memory/manager.ts:242`：启动 session transcript listener
- `extensions/memory-core/src/memory/manager.ts:243`：启动定时同步
- `extensions/memory-core/src/memory/manager.ts:245`：根据是否已有 meta 决定初始 `dirty`

这里的 `dirty` 可以理解成：

```text
当前索引可能不是最新，需要同步
```

如果是 `status` 或 `cli` 用的短生命周期 manager，不会启动 watcher / listener / interval：

- `extensions/memory-core/src/memory/manager.ts:239`

### 2. 文件变化触发 watch 同步

watcher 只在启用了 memory source 且 `sync.watch` 打开时创建：

- `extensions/memory-core/src/memory/manager-sync-ops.ts:406`

它监听这些路径：

- `MEMORY.md`
- `memory/`
- `memorySearch.extraPaths`

相关代码：

- `extensions/memory-core/src/memory/manager-sync-ops.ts:410`：添加默认 watch path
- `extensions/memory-core/src/memory/manager-sync-ops.ts:414`：添加 extra paths
- `extensions/memory-core/src/memory/manager-sync-ops.ts:445`：文件变化时设置 `dirty = true`
- `extensions/memory-core/src/memory/manager-sync-ops.ts:447`：调度 debounced watch sync
- `extensions/memory-core/src/memory/manager-sync-ops.ts:690`：`scheduleWatchSync(...)`
- `extensions/memory-core/src/memory/manager-sync-ops.ts:699`：最终触发 `sync({ reason: "watch" })`

这意味着：

```text
你改了 MEMORY.md 或 memory/*.md
  -> watcher 标记 dirty
  -> 等 debounce
  -> 后台 sync
```

### 3. 搜索时触发同步

搜索时有两层同步保护。

第一层：如果当前完全没有索引内容，第一次搜索会强制同步一次：

- `extensions/memory-core/src/memory/manager.ts:317`：检查是否有 indexed content
- `extensions/memory-core/src/memory/manager.ts:323`：`sync({ reason: "search", force: true })`

目的很直接：

```text
进程刚启动时，watcher 可能还没来得及建索引。
如果第一条 memory_search 直接查空索引，用户会得到空结果。
所以第一次搜索前先做一次同步 bootstrap。
```

第二层：如果已有索引，但 `dirty / sessionsDirty` 为 true，搜索会异步触发同步：

- `extensions/memory-core/src/memory/manager.ts:338`：调用 `startAsyncSearchSync(...)`
- `extensions/memory-core/src/memory/manager-async-state.ts:8`：只有 `onSearch` 打开且 dirty 时才触发
- `extensions/memory-core/src/memory/manager-async-state.ts:11`：后台 `sync({ reason: "search" })`

注意这里是异步。
它通常不会阻塞本次搜索结果。
本次搜索可能还是查旧索引，后续搜索才看到新索引。

### 4. session transcript 变化触发同步

如果配置启用了 `sessions` source，builtin manager 会监听 session transcript 更新：

- `extensions/memory-core/src/memory/manager-sync-ops.ts:461`：`ensureSessionListener()`
- `extensions/memory-core/src/memory/manager-sync-ops.ts:465`：注册 `onSessionTranscriptUpdate(...)`
- `extensions/memory-core/src/memory/manager-sync-ops.ts:473`：记录变动 session file

session 不会每写一行就马上重建索引。
它有 debounce 和阈值：

- `extensions/memory-core/src/memory/manager-sync-ops.ts:83`：session debounce 默认常量
- `extensions/memory-core/src/memory/manager-sync-ops.ts:477`：`scheduleSessionDirty(...)`
- `extensions/memory-core/src/memory/manager-sync-ops.ts:520`：计算 byte threshold
- `extensions/memory-core/src/memory/manager-sync-ops.ts:521`：计算 message threshold
- `extensions/memory-core/src/memory/manager-sync-ops.ts:531`：达到阈值后加入 `sessionsDirtyFiles`
- `extensions/memory-core/src/memory/manager-sync-ops.ts:540`：触发 `sync({ reason: "session-delta" })`

有一个特殊情况：session archive 文件不是持续 append 的 live transcript。
这种文件会直接标记 dirty：

- `extensions/memory-core/src/memory/manager-sync-ops.ts:498`
- `extensions/memory-core/src/memory/manager-sync-ops.ts:511`

### 5. CLI 手动触发

`openclaw memory index` 会直接调用 manager 的 `sync(...)`：

- `extensions/memory-core/src/cli.runtime.ts:1119`：`syncFn({ reason: "cli", force: Boolean(opts.force) })`

`openclaw memory status --deep --index` 也会触发 index：

- `extensions/memory-core/src/cli.runtime.ts:703`
- `extensions/memory-core/src/cli.runtime.ts:712`

命令行里的：

```text
openclaw memory index --force
```

会传：

```text
force = true
```

这通常会走全量重建。

## 三、sync 函数如何防止并发冲突

builtin 的同步入口在：

- `extensions/memory-core/src/memory/manager.ts:618`

关键逻辑：

```text
如果 manager 已关闭
  -> 直接返回

确保 embedding provider 初始化

如果已经有 syncing 在跑
  -> 普通 sync 复用同一个 promise
  -> targeted session sync 排队

否则
  -> runSyncWithReadonlyRecovery(...)
```

对应代码：

- `extensions/memory-core/src/memory/manager.ts:624`：closed 直接返回
- `extensions/memory-core/src/memory/manager.ts:627`：初始化 provider
- `extensions/memory-core/src/memory/manager.ts:628`：如果已有 `this.syncing`
- `extensions/memory-core/src/memory/manager.ts:630`：targeted session sync 排队
- `extensions/memory-core/src/memory/manager.ts:634`：保存当前 sync promise

这避免了多个文件变化、搜索、CLI 同时触发时一起写 SQLite。

targeted session sync 的排队逻辑在：

- `extensions/memory-core/src/memory/manager-sync-control.ts:122`
- `extensions/memory-core/src/memory/manager-sync-control.ts:153`
- `extensions/memory-core/src/memory/manager-sync-control.ts:156`

## 四、真正的同步主流程

真正做事的是：

- `extensions/memory-core/src/memory/manager-sync-ops.ts:978`：`runSync(...)`

流程可以画成：

```text
runSync(params)
  -> ensureVectorReady()
  -> readMeta()
  -> 计算 configuredSources
  -> 计算 configuredScopeHash
  -> 如果是 targeted session sync，优先单独处理
  -> 判断 needsFullReindex
  -> 如果需要全量重建：runSafeReindex(...)
  -> 否则增量同步 memory / sessions
```

### 第一步：加载向量扩展

- `extensions/memory-core/src/memory/manager-sync-ops.ts:992`：`ensureVectorReady()`
- `extensions/memory-core/src/memory/manager-sync-ops.ts:223`：向量扩展加载逻辑
- `extensions/memory-core/src/memory/manager-sync-ops.ts:251`：`loadVectorExtension()`

如果 sqlite-vec 可用，就可以把向量写入 `chunks_vec`。
如果不可用，OpenClaw 仍然可以用 `chunks` 里的 embedding JSON 做进程内 cosine similarity，或者退化成 FTS-only。

### 第二步：读取 meta

- `extensions/memory-core/src/memory/manager-sync-ops.ts:993`：读取 meta
- `extensions/memory-core/src/memory/manager-sync-ops.ts:1358`：`readMeta()`

meta 是判断“要不要全量重建”的核心。

它保存的信息包括：

- provider
- model
- providerKey
- sources
- scopeHash
- chunkTokens
- chunkOverlap
- vectorDims
- ftsTokenizer

类型定义在：

- `extensions/memory-core/src/memory/manager-reindex-state.ts:7`

### 第三步：计算当前配置指纹

同步会根据当前配置计算两个东西：

```text
configuredSources = 当前启用哪些 source，例如 memory / sessions
configuredScopeHash = 当前扫描范围和 multimodal 配置的 hash
```

相关代码：

- `extensions/memory-core/src/memory/manager-sync-ops.ts:994`：`configuredSources`
- `extensions/memory-core/src/memory/manager-sync-ops.ts:995`：`configuredScopeHash`
- `extensions/memory-core/src/memory/manager-reindex-state.ts:19`：source 排序和规范化
- `extensions/memory-core/src/memory/manager-reindex-state.ts:52`：scope hash 计算

`scopeHash` 会包含：

- workspaceDir
- extraPaths
- multimodal.enabled
- multimodal.modalities
- multimodal.maxFileBytes

所以你改了 extraPaths 或 multimodal 配置，索引通常需要全量重建。

### 第四步：targeted session sync

如果传入了 `sessionFiles`，会优先走 targeted session sync：

- `extensions/memory-core/src/memory/manager-sync-ops.ts:1004`
- `extensions/memory-core/src/memory/manager-sync-ops.ts:1006`
- `extensions/memory-core/src/memory/manager-targeted-sync.ts:25`

这个路径只刷新指定 session transcript，不会枚举所有 session 文件。

重要限制：

- `extensions/memory-core/src/memory/manager-session-sync-state.ts:16`：targeted sync 时 `activePaths = null`
- `extensions/memory-core/src/memory/manager-sync-ops.ts:924`：targeted sync 不清理 unrelated session rows

也就是说：

```text
targeted session sync = 快速刷新指定 session
full session sync     = 可以顺便清理已经不存在的 stale session rows
```

## 五、什么时候会全量重建

判断入口：

- `extensions/memory-core/src/memory/manager-sync-ops.ts:1031`
- `extensions/memory-core/src/memory/manager-reindex-state.ts:76`

触发全量重建的条件包括：

### 1. 手动 force

```text
openclaw memory index --force
```

会传 `force = true`。

如果不是 targeted session sync，就会触发 full reindex：

- `extensions/memory-core/src/memory/manager-sync-ops.ts:1032`

### 2. 没有旧 meta

`meta` 不存在说明这个 SQLite index 从未完整建立过，或者旧索引损坏/太旧。

- `extensions/memory-core/src/memory/manager-reindex-state.ts:89`

### 3. embedding provider 或 model 改了

如果 provider/model 和 meta 里保存的不一致，就重建：

- `extensions/memory-core/src/memory/manager-reindex-state.ts:90`
- `extensions/memory-core/src/memory/manager-reindex-state.ts:91`

原因：

```text
不同 embedding model 生成的向量空间不同
旧向量不能和新 query 向量混用
```

### 4. providerKey 改了

- `extensions/memory-core/src/memory/manager-reindex-state.ts:92`

`providerKey` 用来区分同一个 provider/model 下不同 endpoint 或参数身份。
如果它变了，embedding cache 和索引需要重新对齐。

### 5. sources 改了

- `extensions/memory-core/src/memory/manager-reindex-state.ts:93`

比如以前只索引 `memory`，后来启用 `sessions`。
索引里的 `source` 集合变化了，需要重建。

### 6. scopeHash 改了

- `extensions/memory-core/src/memory/manager-reindex-state.ts:97`

比如：

- workspace 变了
- `extraPaths` 变了
- multimodal 开关或类型变了
- multimodal 最大文件大小变了

### 7. chunking 配置改了

- `extensions/memory-core/src/memory/manager-reindex-state.ts:98`
- `extensions/memory-core/src/memory/manager-reindex-state.ts:99`

chunk tokens / overlap 改了，旧 chunk 的行号范围和 embedding 输入都可能不同。

### 8. vector 维度信息缺失

- `extensions/memory-core/src/memory/manager-reindex-state.ts:100`

如果当前 vector store ready，但旧 meta 没有 `vectorDims`，需要重建补齐。

### 9. FTS tokenizer 改了

- `extensions/memory-core/src/memory/manager-reindex-state.ts:101`

FTS tokenizer 影响关键词索引方式。
比如 `unicode61` 和 `trigram` 的索引结构不同。

## 六、增量同步如何工作

如果不需要全量重建，就走增量同步：

- `extensions/memory-core/src/memory/manager-sync-ops.ts:1066`
- `extensions/memory-core/src/memory/manager-sync-ops.ts:1071`
- `extensions/memory-core/src/memory/manager-sync-ops.ts:1076`

### memory 文件增量同步

入口：

- `extensions/memory-core/src/memory/manager-sync-ops.ts:716`

流程：

```text
listMemoryFiles(...)
  -> buildFileEntry(...)
  -> 和 files 表里的 hash 比较
  -> hash 一样：跳过
  -> hash 不一样：indexFile(...)
  -> 原来存在、现在不存在：删除 stale rows
```

相关代码：

- `extensions/memory-core/src/memory/manager-sync-ops.ts:737`：扫描 memory files
- `packages/memory-host-sdk/src/host/internal.ts:139`：`listMemoryFiles(...)`
- `packages/memory-host-sdk/src/host/internal.ts:163`：加入根 `MEMORY.md`
- `packages/memory-host-sdk/src/host/internal.ts:170`：递归扫描 `memory/`
- `packages/memory-host-sdk/src/host/internal.ts:174`：扫描 `extraPaths`
- `packages/memory-host-sdk/src/host/internal.ts:219`：`buildFileEntry(...)`
- `packages/memory-host-sdk/src/host/internal.ts:286`：根据文件内容计算 hash
- `extensions/memory-core/src/memory/manager-sync-ops.ts:757`：加载旧文件状态
- `extensions/memory-core/src/memory/manager-sync-ops.ts:774`：hash 相同就跳过
- `extensions/memory-core/src/memory/manager-sync-ops.ts:784`：调用 `indexFile(...)`
- `extensions/memory-core/src/memory/manager-sync-ops.ts:795`：清理 stale rows

所以增量同步不是每次都重新 embedding 所有文件。
它会先看 hash。

### session 文件增量同步

入口：

- `extensions/memory-core/src/memory/manager-sync-ops.ts:814`

流程：

```text
决定要扫描哪些 session files
  -> resolveMemorySessionSyncPlan(...)
  -> buildSessionEntry(...)
  -> 和旧 hash 比较
  -> 未变：跳过
  -> 变了：indexFile(... source=sessions ...)
  -> 如果是全量枚举：清理 stale session rows
```

相关代码：

- `extensions/memory-core/src/memory/manager-sync-ops.ts:836`：targeted sync 时只看指定 session files
- `extensions/memory-core/src/memory/manager-sync-ops.ts:841`：非 targeted 时枚举 agent session files
- `extensions/memory-core/src/memory/manager-session-sync-state.ts:3`：制定 session sync plan
- `extensions/memory-core/src/memory/manager-sync-ops.ts:884`：`buildSessionEntry(...)`
- `extensions/memory-core/src/memory/manager-sync-ops.ts:895`：查旧 hash
- `extensions/memory-core/src/memory/manager-sync-ops.ts:901`：hash 相同跳过
- `extensions/memory-core/src/memory/manager-sync-ops.ts:912`：变更后重新 index
- `extensions/memory-core/src/memory/manager-sync-ops.ts:924`：targeted sync 不做 stale prune
- `extensions/memory-core/src/memory/manager-sync-ops.ts:930`：全量枚举时清理 stale rows

session line number 有额外处理：

- `packages/memory-host-sdk/src/host/internal.ts:467`：`remapChunkLines(...)`

因为 session JSONL 会先被 flatten 成文本，再切 chunk。
最终索引里存的 line range 要映射回原始 JSONL 行号。

## 七、单个文件如何被写入索引

真正把一个文件变成 chunks 的入口：

- `extensions/memory-core/src/memory/manager-embedding-ops.ts:645`：`indexFile(...)`

### 1. 没有 embedding provider：FTS-only

如果 provider 不存在：

- `extensions/memory-core/src/memory/manager-embedding-ops.ts:649`
- `extensions/memory-core/src/memory/manager-embedding-ops.ts:655`
- `extensions/memory-core/src/memory/manager-embedding-ops.ts:656`
- `extensions/memory-core/src/memory/manager-embedding-ops.ts:660`

流程：

```text
读取 Markdown 文本
  -> chunkMarkdown(...)
  -> writeChunks(... model="fts-only", embeddings=[])
```

这种情况下没有向量检索，但 FTS5 关键词搜索仍然可用。

multimodal 文件在 FTS-only 下会被跳过：

- `extensions/memory-core/src/memory/manager-embedding-ops.ts:651`

### 2. 有 embedding provider：切 chunk + embedding

普通 Markdown：

- `extensions/memory-core/src/memory/manager-embedding-ops.ts:685`：读取文本
- `packages/memory-host-sdk/src/host/internal.ts:362`：`chunkMarkdown(...)`
- `extensions/memory-core/src/memory/manager-embedding-ops.ts:688`：限制 embedding 输入 token

chunk 信息包括：

- startLine
- endLine
- text
- hash
- embeddingInput

见：

- `packages/memory-host-sdk/src/host/internal.ts:386`
- `packages/memory-host-sdk/src/host/internal.ts:389`
- `packages/memory-host-sdk/src/host/internal.ts:393`
- `packages/memory-host-sdk/src/host/internal.ts:394`

然后生成 embedding：

- `extensions/memory-core/src/memory/manager-embedding-ops.ts:701`：batch embedding
- `extensions/memory-core/src/memory/manager-embedding-ops.ts:703`：inline embedding

再写入索引：

- `extensions/memory-core/src/memory/manager-embedding-ops.ts:727`：确保 vector table 维度
- `extensions/memory-core/src/memory/manager-embedding-ops.ts:728`：`writeChunks(...)`

### 3. writeChunks 写哪些表

`writeChunks(...)` 在：

- `extensions/memory-core/src/memory/manager-embedding-ops.ts:578`

它会：

```text
clearIndexedFileData(...)
  -> 删除这个 path/source 旧 chunks、旧 vector rows、旧 FTS rows

for each chunk:
  -> INSERT chunks
  -> 如果 vectorReady，写 chunks_vec
  -> 如果 FTS 可用，写 chunks_fts

最后 upsert files 表
```

关键代码：

- `extensions/memory-core/src/memory/manager-embedding-ops.ts:587`：先清理旧数据
- `extensions/memory-core/src/memory/manager-embedding-ops.ts:596`：写 `chunks`
- `extensions/memory-core/src/memory/manager-embedding-ops.ts:618`：写 `chunks_vec`
- `extensions/memory-core/src/memory/manager-embedding-ops.ts:628`：写 `chunks_fts`
- `extensions/memory-core/src/memory/manager-embedding-ops.ts:642`：写 `files`

## 八、全量重建过程

如果判断 `needsFullReindex = true`，builtin 后端会走 safe reindex：

- `extensions/memory-core/src/memory/manager-sync-ops.ts:1046`
- `extensions/memory-core/src/memory/manager-sync-ops.ts:1057`
- `extensions/memory-core/src/memory/manager-sync-ops.ts:1171`

### safe reindex 的核心思路

不是直接清空当前 SQLite。
而是：

```text
当前 index.sqlite 继续可用
        |
        v
创建 index.sqlite.tmp-<uuid>
        |
        v
在临时 DB 里完整建新索引
        |
        v
构建成功后，把 temp DB 原子替换成正式 DB
        |
        v
失败则删除 temp，保留旧 DB
```

这就是为什么叫 safe reindex。

关键代码：

- `extensions/memory-core/src/memory/manager-sync-ops.ts:1176`：正式 db path
- `extensions/memory-core/src/memory/manager-sync-ops.ts:1177`：临时 db path
- `extensions/memory-core/src/memory/manager-sync-ops.ts:1178`：打开临时 DB
- `extensions/memory-core/src/memory/manager-sync-ops.ts:1207`：manager 暂时切到临时 DB
- `extensions/memory-core/src/memory/manager-sync-ops.ts:1211`：临时 DB 建 schema
- `extensions/memory-core/src/memory/manager-sync-ops.ts:1216`：`runMemoryAtomicReindex(...)`

### safe reindex 会复用 embedding cache

重建前会把旧 DB 的 embedding cache seed 到新 DB：

- `extensions/memory-core/src/memory/manager-sync-ops.ts:322`：`seedEmbeddingCache(...)`
- `extensions/memory-core/src/memory/manager-sync-ops.ts:1220`：reindex build 前 seed cache

这样 forced reindex 不一定要重新调用 provider embedding 所有 chunk。
如果 chunk hash、provider、model、providerKey 一样，cache 可以复用。

### safe reindex 会全量同步 enabled sources

在临时 DB 内：

- `extensions/memory-core/src/memory/manager-sync-ops.ts:1221`：判断是否同步 memory
- `extensions/memory-core/src/memory/manager-sync-ops.ts:1228`：全量 sync memory files
- `extensions/memory-core/src/memory/manager-sync-ops.ts:1232`：判断是否同步 sessions
- `extensions/memory-core/src/memory/manager-sync-ops.ts:1233`：全量 sync session files

完成后写 meta：

- `extensions/memory-core/src/memory/manager-sync-ops.ts:1242`：构造 meta
- `extensions/memory-core/src/memory/manager-sync-ops.ts:1265`：`writeMeta(meta)`
- `extensions/memory-core/src/memory/manager-sync-ops.ts:1266`：prune embedding cache

### 原子替换怎么做

原子替换逻辑在：

- `extensions/memory-core/src/memory/manager-atomic-reindex.ts:91`

它处理的不只是主 DB 文件，还包括 WAL sidecar：

- `extensions/memory-core/src/memory/manager-atomic-reindex.ts:63`：suffixes 是 `""`, `"-wal"`, `"-shm"`

替换流程：

- `extensions/memory-core/src/memory/manager-atomic-reindex.ts:80`：准备 backup path
- `extensions/memory-core/src/memory/manager-atomic-reindex.ts:81`：正式 DB 先移动到 backup
- `extensions/memory-core/src/memory/manager-atomic-reindex.ts:83`：temp DB 移到正式位置
- `extensions/memory-core/src/memory/manager-atomic-reindex.ts:85`：如果 temp 移动失败，恢复 backup
- `extensions/memory-core/src/memory/manager-atomic-reindex.ts:88`：成功后删除 backup
- `extensions/memory-core/src/memory/manager-atomic-reindex.ts:101`：失败时删除 temp

Windows 或其他平台可能短时间锁文件，所以 rename 有重试：

- `extensions/memory-core/src/memory/manager-atomic-reindex.ts:23`
- `extensions/memory-core/src/memory/manager-atomic-reindex.ts:31`

### 重建完成后重新打开正式 DB

- `extensions/memory-core/src/memory/manager-sync-ops.ts:1275`：重新打开正式 DB
- `extensions/memory-core/src/memory/manager-sync-ops.ts:1276`：重置 vector state
- `extensions/memory-core/src/memory/manager-sync-ops.ts:1277`：确保 schema
- `extensions/memory-core/src/memory/manager-sync-ops.ts:1278`：恢复 vector dims

如果中间失败：

- `extensions/memory-core/src/memory/manager-sync-ops.ts:1279`
- `extensions/memory-core/src/memory/manager-sync-ops.ts:1283`

会恢复原来的 DB state。

## 九、unsafe reindex 是什么

还有一个 `runUnsafeReindex(...)`：

- `extensions/memory-core/src/memory/manager-sync-ops.ts:1288`

它会直接：

- `extensions/memory-core/src/memory/manager-sync-ops.ts:1295`：`resetIndex()`
- `extensions/memory-core/src/memory/manager-sync-ops.ts:1304`：重建 memory
- `extensions/memory-core/src/memory/manager-sync-ops.ts:1309`：重建 sessions

`resetIndex()` 会：

- `extensions/memory-core/src/memory/manager-sync-ops.ts:1345`：删除 `files`
- `extensions/memory-core/src/memory/manager-sync-ops.ts:1346`：删除 `chunks`
- `extensions/memory-core/src/memory/manager-sync-ops.ts:1349`：删除 FTS table
- `extensions/memory-core/src/memory/manager-sync-ops.ts:1353`：删除 vector table
- `extensions/memory-core/src/memory/manager-sync-ops.ts:1355`：清理 session dirty files

但代码注释说明它是 test perf 路径：

- `extensions/memory-core/src/memory/manager-sync-ops.ts:1293`

正常运行不应该走这个。

## 十、异常恢复：readonly 和 provider fallback

### readonly SQLite 恢复

同步外面包了一层 readonly recovery：

- `extensions/memory-core/src/memory/manager.ts:664`
- `extensions/memory-core/src/memory/manager.ts:733`
- `extensions/memory-core/src/memory/manager-sync-control.ts:83`

如果遇到 SQLite readonly 错误：

- `extensions/memory-core/src/memory/manager-sync-control.ts:34`
- `extensions/memory-core/src/memory/manager-sync-control.ts:96`

它会：

```text
关闭旧 DB handle
重新 open DB
reset vector state
ensure schema
读取 meta 恢复 vectorDims
重试 runSync
```

对应代码：

- `extensions/memory-core/src/memory/manager-sync-control.ts:104`
- `extensions/memory-core/src/memory/manager-sync-control.ts:107`
- `extensions/memory-core/src/memory/manager-sync-control.ts:108`
- `extensions/memory-core/src/memory/manager-sync-control.ts:109`
- `extensions/memory-core/src/memory/manager-sync-control.ts:110`
- `extensions/memory-core/src/memory/manager-sync-control.ts:113`

### embedding provider fallback

如果同步时 embedding / batch 出错，builtin 会尝试 fallback provider：

- `extensions/memory-core/src/memory/manager-sync-ops.ts:1089`
- `extensions/memory-core/src/memory/manager-sync-ops.ts:1092`
- `extensions/memory-core/src/memory/manager-sync-ops.ts:1094`

错误是否适合 fallback 的判断很宽：

- `extensions/memory-core/src/memory/manager-sync-ops.ts:1105`

fallback 成功后会：

```text
切换 provider
重新计算 providerKey
重新 resolve batch config
force safe reindex
```

相关代码：

- `extensions/memory-core/src/memory/manager-sync-ops.ts:1127`
- `extensions/memory-core/src/memory/manager-sync-ops.ts:1161`
- `extensions/memory-core/src/memory/manager-sync-ops.ts:1163`
- `extensions/memory-core/src/memory/manager-sync-ops.ts:1164`

## 十一、QMD 后端的同步和重建

QMD 后端不是自己维护 `chunks` 表。
它通过外部 `qmd` 命令管理 collections 和 index。

官方文档也说明：

- `docs/concepts/memory-qmd.md:53`：OpenClaw 从 workspace memory files 和 qmd paths 创建 collections
- `docs/concepts/memory-qmd.md:54`：然后运行 `qmd update`
- `docs/concepts/memory-qmd.md:57`：语义模式还会运行 `qmd embed`

### QMD 初始化

QMD manager 会为每个 agent 创建独立的 QMD home：

- `extensions/memory-core/src/memory/qmd-manager.ts:361`：OpenClaw state dir
- `extensions/memory-core/src/memory/qmd-manager.ts:363`：agent qmd dir
- `extensions/memory-core/src/memory/qmd-manager.ts:369`：`XDG_CONFIG_HOME`
- `extensions/memory-core/src/memory/qmd-manager.ts:370`：`XDG_CACHE_HOME`
- `extensions/memory-core/src/memory/qmd-manager.ts:371`：QMD index path

QMD collections 在：

- `extensions/memory-core/src/memory/qmd-manager.ts:504`：`ensureCollections()`

它会：

```text
列出现有 collections
迁移旧 collection
检查每个 managed collection
必要时 remove + add
```

关键代码：

- `extensions/memory-core/src/memory/qmd-manager.ts:508`：列出现有 collections
- `extensions/memory-core/src/memory/qmd-manager.ts:510`：迁移旧 unscoped collections
- `extensions/memory-core/src/memory/qmd-manager.ts:512`：遍历配置 collection
- `extensions/memory-core/src/memory/qmd-manager.ts:517`：需要 rebind 时先 remove
- `extensions/memory-core/src/memory/qmd-manager.ts:529`：`addCollection(...)`

如果启用了 QMD sessions，会额外加一个 sessions collection：

- `extensions/memory-core/src/memory/qmd-manager.ts:386`
- `extensions/memory-core/src/memory/qmd-manager.ts:396`
- `extensions/memory-core/src/memory/qmd-manager.ts:399`
- `extensions/memory-core/src/memory/qmd-manager.ts:402`

### QMD 触发同步

QMD 的 `sync(...)` 很薄：

- `extensions/memory-core/src/memory/qmd-manager.ts:1275`
- `extensions/memory-core/src/memory/qmd-manager.ts:1287`

它本质上调用：

```text
runUpdate(reason, force)
```

QMD update 的触发点包括：

- `extensions/memory-core/src/memory/qmd-manager.ts:444`：onBoot update
- `extensions/memory-core/src/memory/qmd-manager.ts:456`：interval update
- `extensions/memory-core/src/memory/qmd-manager.ts:463`：embed interval
- `extensions/memory-core/src/memory/qmd-manager.ts:1530`：watcher
- `extensions/memory-core/src/memory/qmd-manager.ts:1582`：watch 触发 sync
- `extensions/memory-core/src/memory/qmd-manager.ts:1597`：session-start 触发 sync
- `extensions/memory-core/src/memory/qmd-manager.ts:1606`：搜索时 dirty 则 sync

QMD watcher 和 builtin watcher 类似，也是文件变更后标记 dirty：

- `extensions/memory-core/src/memory/qmd-manager.ts:1555`
- `extensions/memory-core/src/memory/qmd-manager.ts:1557`

### QMD runUpdate 做什么

入口：

- `extensions/memory-core/src/memory/qmd-manager.ts:1459`

关键流程：

```text
如果已有 pending update
  -> 非 force 复用 pending
  -> force 排队

如果 debounce 未过且非 force
  -> skip

withQmdUpdateQueue(...)
  -> exportSessions()
  -> qmd update
  -> dirty = false

如果语义模式需要 embedding
  -> qmd embed
```

相关代码：

- `extensions/memory-core/src/memory/qmd-manager.ts:1467`：已有 pending update
- `extensions/memory-core/src/memory/qmd-manager.ts:1469`：force update 排队
- `extensions/memory-core/src/memory/qmd-manager.ts:1479`：debounce skip
- `extensions/memory-core/src/memory/qmd-manager.ts:1491`：先 export sessions
- `extensions/memory-core/src/memory/qmd-manager.ts:1494`：运行 qmd update
- `extensions/memory-core/src/memory/qmd-manager.ts:1495`：清 dirty
- `extensions/memory-core/src/memory/qmd-manager.ts:1500`：决定是否 qmd embed
- `extensions/memory-core/src/memory/qmd-manager.ts:1503`：运行 `qmd embed`

`shouldRunEmbed(...)` 在：

- `extensions/memory-core/src/memory/qmd-manager.ts:1658`

它会根据：

- searchMode 是否用 vectors
- force 是否为 true
- lastEmbedAt
- embedInterval
- embedBackoffUntil

决定是否运行 `qmd embed`。

### QMD 的“重建”更多是 collection 修复

QMD 不走 builtin 那种 temp SQLite 原子替换。
它的重建主要是：

```text
managed collections 出错
  -> remove collection
  -> add collection
  -> retry qmd update
```

入口：

- `extensions/memory-core/src/memory/qmd-manager.ts:1023`：`rebuildManagedCollectionsForRepair(...)`

它会遍历 managed collections：

- `extensions/memory-core/src/memory/qmd-manager.ts:1024`
- `extensions/memory-core/src/memory/qmd-manager.ts:1026`：remove collection
- `extensions/memory-core/src/memory/qmd-manager.ts:1034`：add collection

触发场景主要有两个：

1. null-byte collection metadata
   - `extensions/memory-core/src/memory/qmd-manager.ts:1045`
   - `extensions/memory-core/src/memory/qmd-manager.ts:1056`

2. duplicate document constraint
   - `extensions/memory-core/src/memory/qmd-manager.ts:1060`
   - `extensions/memory-core/src/memory/qmd-manager.ts:1074`

`qmd update` 失败后会尝试修复并重试一次：

- `extensions/memory-core/src/memory/qmd-manager.ts:1629`
- `extensions/memory-core/src/memory/qmd-manager.ts:1637`
- `extensions/memory-core/src/memory/qmd-manager.ts:1642`

boot update 还有重试机制：

- `extensions/memory-core/src/memory/qmd-manager.ts:1609`
- `extensions/memory-core/src/memory/qmd-manager.ts:1611`
- `extensions/memory-core/src/memory/qmd-manager.ts:1620`

## 十二、sync 和 rebuild 的区别

新手可以这样区分：

```text
sync:
  当前索引结构仍然可信
  只需要把变更文件更新进去
  hash 没变的文件跳过
  删除已经不存在的 stale rows

rebuild:
  当前索引结构或向量语义可能不再可信
  需要重新建完整索引
  builtin 用临时 DB 构建成功后原子替换
  QMD 主要通过重建 managed collections + qmd update 修复
```

更直观一点：

```text
你改了 memory/2026-05-14.md
  -> sync 增量更新这个文件

你改了 embedding model
  -> rebuild，因为旧 embedding 和新 embedding 不能混用

你改了 chunkTokens
  -> rebuild，因为 chunk 边界变了

你改了 extraPaths
  -> rebuild，因为扫描范围变了

你执行 openclaw memory index --force
  -> rebuild
```

## 十三、为什么要这样设计

### 1. 搜索快

搜索时不需要遍历所有 Markdown。
它查的是预先构建好的 SQLite / QMD index。

### 2. 增量成本低

普通文件修改只重新索引 hash 变化的文件。

### 3. 配置变化安全

一旦 provider、model、chunking、source、scope、FTS tokenizer 这些会影响索引语义的配置变了，就全量重建。

### 4. 重建尽量不中断

builtin safe reindex 用临时 DB 构建，成功后再替换。
构建失败时旧 index 还在。

### 5. session 不过度同步

session transcript 是不断增长的。
代码用 debounce、byte threshold、message threshold 控制同步频率，避免每条消息都重建索引。

## 十四、排查问题时看哪里

### 搜索结果旧

先看：

- `memory status` 里的 `dirty`
- 是否 watcher 开启
- 是否 `sync.onSearch` 开启
- 是否刚改完文件但 debounce 还没跑

对应源码：

- `extensions/memory-core/src/memory/manager.ts:779`：status 里 dirty 来自 `dirty || sessionsDirty`
- `extensions/memory-core/src/memory/manager-sync-ops.ts:406`：watcher 创建条件
- `extensions/memory-core/src/memory/manager-async-state.ts:8`：onSearch dirty sync 条件

### 强制重建

命令：

```text
openclaw memory index --force
```

对应源码：

- `extensions/memory-core/src/cli.runtime.ts:1119`
- `extensions/memory-core/src/memory/manager-sync-ops.ts:1032`

### session 搜不到

检查：

- 是否启用了 `sessions` source
- 是否达到 session delta threshold
- 是否是 targeted sync 只刷新了部分 session
- 是否 session visibility 后续过滤掉了结果

相关源码：

- `extensions/memory-core/src/memory/manager-sync-ops.ts:461`
- `extensions/memory-core/src/memory/manager-sync-ops.ts:520`
- `extensions/memory-core/src/memory/manager-session-reindex.ts:1`
- `extensions/memory-core/src/session-search-visibility.ts:14`

### QMD 搜不到

检查：

- QMD binary 是否可用
- collections 是否创建
- 是否需要 `qmd update`
- searchMode 是否需要 embed
- QMD scope 是否允许当前 session 类型

相关源码：

- `extensions/memory-core/src/memory/qmd-manager.ts:431`
- `extensions/memory-core/src/memory/qmd-manager.ts:1275`
- `extensions/memory-core/src/memory/qmd-manager.ts:1459`
- `extensions/memory-core/src/memory/qmd-manager.ts:1658`

## 十五、最终心智模型

把 memory index 想成一本目录：

```text
原始 Markdown / session transcript = 书本正文
SQLite / QMD index                 = 目录和索引卡片
memory_search                      = 查目录
memory_get                         = 按目录指向读正文
sync                               = 更新目录
rebuild                            = 重新做整本目录
```

所以当你问“为什么搜索没找到刚写进去的内容”时，第一反应应该是：

```text
原文是否存在？
索引是否同步？
是否只是 dirty 还没 flush？
是否配置变化需要 rebuild？
是否当前 backend 是 builtin 还是 QMD？
```

这就是同步阶段和重建过程要解决的问题。
