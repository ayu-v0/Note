# 20. memory_search 和 memory_get：OpenClaw 如何取回记忆

这篇回答这个问题：

```text
memory_search 和 memory_get 是怎么设计的？
为什么要分成两个工具？
这种设计有什么优点和缺点？
```

## 先给结论

OpenClaw 没有把所有记忆一次性塞进模型上下文。
它把“找记忆”和“读原文”拆成两个工具：

```text
memory_search = 搜索索引，找到可能相关的记忆片段
memory_get    = 根据搜索结果里的 path / 行号，读取工具允许访问的 memory/wiki 原文片段
```

你可以把它理解成：

```text
memory_search 像搜索引擎
memory_get    像打开搜索结果里的那篇文章
```

这样设计的核心思想是：

```text
先用搜索减少范围，再用读取确认原文。
```

这比“每次都把 MEMORY.md 和 memory/*.md 全部塞进 prompt”更省 token，也更容易追溯来源。

## 一、这两个工具从哪里注册

记忆工具是 `memory-core` 插件注册出来的。

入口在：

- `extensions/memory-core/index.ts:178`：注册 memory capability
- `extensions/memory-core/index.ts:190`：注册 `memory_search`
- `extensions/memory-core/index.ts:194`：注册 `memory_get`

代码不是无条件暴露工具，而是先判断当前 agent 是否真的有 memory 上下文：

- `extensions/memory-core/index.ts:48`：`hasMemoryToolContext(...)`

如果没有可用的 memory 配置，工具不会作为可用工具暴露给模型。

工具本身是 lazy tool：

- `extensions/memory-core/index.ts:85`：`createLazyMemoryTool(...)`
- `extensions/memory-core/index.ts:122`：创建懒加载版 `memory_search`
- `extensions/memory-core/index.ts:134`：创建懒加载版 `memory_get`

新手可以这样理解：

```text
OpenClaw 启动时：
1. memory-core 插件检查当前 agent 是否启用了记忆能力
2. 如果启用了，就向工具系统注册 memory_search / memory_get
3. 工具真正执行时，才加载后端搜索和读取逻辑
```

## 二、prompt 怎么引导模型使用它们

模型不是凭空知道什么时候用记忆工具。
`memory-core` 会给 prompt 加一段记忆使用说明。

相关代码：

- `extensions/memory-core/src/prompt-section.ts:3`：构造 memory prompt section
- `extensions/memory-core/src/prompt-section.ts:7`：检查是否有 `memory_search`
- `extensions/memory-core/src/prompt-section.ts:8`：检查是否有 `memory_get`
- `extensions/memory-core/src/prompt-section.ts:17`：告诉模型遇到历史工作、决策、日期、人物、偏好、待办等问题时，先 `memory_search`，再 `memory_get`

所以它的理想工作流是：

```text
用户问到历史/偏好/之前做过的事
        |
        v
模型调用 memory_search
        |
        v
拿到相关片段、path、行号、citation
        |
        v
模型调用 memory_get 读取原文
        |
        v
基于原文回答用户
```

## 三、memory_search 的设计

`memory_search` 负责“找”。

工具创建位置：

- `extensions/memory-core/src/tools.ts:228`：`createMemorySearchTool(...)`

参数定义在：

- `extensions/memory-core/index.ts:61`：`MemorySearchSchema`

它主要接收这些参数：

```text
query      = 搜索问题
maxResults = 最多返回多少条
minScore   = 最低相关度
corpus     = 搜索范围：memory / wiki / all / sessions
```

执行主线在：

- `extensions/memory-core/src/tools.ts:246`：检查 `query`
- `extensions/memory-core/src/tools.ts:249`：解析 `corpus`
- `extensions/memory-core/src/tools.ts:256`：判断是否搜索 memory corpus
- `extensions/memory-core/src/tools.ts:299`：调用 `memory.manager.search(...)`
- `extensions/memory-core/src/tools.ts:321`：给结果加 citation
- `extensions/memory-core/src/tools.ts:335`：记录 recall 信号，供后续 dreaming / promote 使用
- `extensions/memory-core/src/tools.ts:360`：搜索 supplement corpus，比如 wiki
- `extensions/memory-core/src/tools.ts:370`：合并 memory 和 supplement 结果
- `extensions/memory-core/src/tools.ts:376`：返回 JSON 结果

它返回的不是完整文件，而是一组搜索结果。
结果里通常会有：

```text
path       = 记忆文件路径
line range = 行号范围
snippet    = 命中的片段
score      = 相关度
citation   = 可追溯引用
```

### 搜索结果返回后怎么处理

结论：

```text
memory_search 会返回一组搜索结果，不是只返回一条。
系统不会自动拿最高分那条去调用 memory_get。
是否调用 memory_get、读哪一条或哪几条，是后续模型根据搜索结果自己决定的。
```

也就是说，流程是：

```text
memory_search
  -> 返回 top N 个候选片段
  -> 每条包含 path / startLine / endLine / snippet / score / citation

模型阅读这些候选结果
  -> 判断哪条最相关
  -> 必要时调用 memory_get
  -> memory_get 根据 path + 行号读取可访问的原文片段
```

`memory_search` 工具实现里，结果处理链路是：

```text
extensions/memory-core/src/tools.ts:299
  -> rawResults = await memory.manager.search(...)

extensions/memory-core/src/tools.ts:309
  -> filterMemorySearchHitsBySessionVisibility(...)
  -> sessions 来源的命中还要做可见性过滤

extensions/memory-core/src/tools.ts:321
  -> decorateCitations(...)
  -> 给结果加 citation，并把 Source 写入 snippet

extensions/memory-core/src/tools.ts:325
  -> QMD 后端会 clampResultsByInjectedChars(...)
  -> 防止注入给模型的 snippet 总字符数过大

extensions/memory-core/src/tools.ts:327
  -> surfacedMemoryResults = memoryResults.map(...)
  -> 给结果加 corpus 标记

extensions/memory-core/src/tools.ts:370
  -> mergeMemorySearchCorpusResults(...)
  -> 如果 corpus=all，会合并 builtin manager 结果和 wiki supplement 结果
```

最后返回给模型的是：

```text
{
  results: [...],
  provider,
  model,
  fallback,
  citations,
  mode,
  debug
}
```

这里有一个容易误解的边界：`memory_search` 的 `corpus` 支持 `sessions`，但 `memory_get` 的 schema 不支持 `sessions`。
代码里 `MemoryGetSchema` 只允许 `memory / wiki / all`，见 `extensions/memory-core/src/tools.shared.ts:44`。
并且 builtin 读取路径会走 `readMemoryFile(...)`，它在 `packages/memory-host-sdk/src/host/read-file.ts:41` 用 `isMemoryPath(...)` 做路径限制，而 `isMemoryPath(...)` 在 `packages/memory-host-sdk/src/host/internal.ts:92` 只放行 `MEMORY.md`、`dreams.md` 和 `memory/` 下的路径。

所以不能把“搜索结果”理解成“每一条都一定能继续 memory_get 打开”。
对 `memory` / `wiki` 命中，通常可以按 `path + from/lines` 继续读取。
对 `sessions` 命中，工具返回的 `snippet` 和 `citation` 本身就是主要可用内容；不能假设 `memory_get` 可以读取完整 session transcript。

最多返回多少条？

builtin SQLite 后端默认：

```text
src/agents/memory-search.ts:108
  -> DEFAULT_MAX_RESULTS = 6

src/agents/memory-search.ts:271
  -> query.maxResults 默认使用 DEFAULT_MAX_RESULTS
```

也就是说，如果工具调用没有传 `maxResults`，builtin memory manager 默认最多返回 6 条。

但检索时不是只取 6 条候选来算。它会先扩大候选集：

```text
src/agents/memory-search.ts:113
  -> DEFAULT_HYBRID_CANDIDATE_MULTIPLIER = 4

extensions/memory-core/src/memory/manager.ts:365
  -> candidates = min(200, maxResults * candidateMultiplier)
```

默认情况下：

```text
maxResults = 6
candidateMultiplier = 4

候选集 candidates = 24
最终返回 top 6
```

这样做的原因是：先多找一些候选，再经过关键词/向量混合排序、去重、过滤，最后只把最相关的几条交给模型。

builtin 后端最终截断结果的位置包括：

```text
extensions/memory-core/src/memory/manager.ts:433
  -> FTS-only 分支排序后 selectScoredResults(...)

extensions/memory-core/src/memory/manager.ts:472
  -> hybrid 合并后 strict.slice(0, maxResults)

extensions/memory-core/src/memory/manager.ts:495
  -> selectScoredResults(...)
```

`selectScoredResults(...)` 的规则是：

```text
extensions/memory-core/src/memory/manager.ts:500
  -> 先保留 score >= minScore 的 strict 结果

extensions/memory-core/src/memory/manager.ts:503
  -> 如果 strict 有结果，返回 strict.slice(0, maxResults)

extensions/memory-core/src/memory/manager.ts:505
  -> 如果没有 strict，允许 relaxedMinScore 兜底，再 slice(0, maxResults)
```

默认相关度阈值是：

```text
src/agents/memory-search.ts:109
  -> DEFAULT_MIN_SCORE = 0.35
```

QMD 后端也有自己的上限：

```text
extensions/memory-core/src/memory/qmd-manager.ts:1100
  -> resultLimit = min(qmd.limits.maxResults, opts.maxResults ?? qmd.limits.maxResults)

extensions/memory-core/src/memory/qmd-manager.ts:1272
  -> clampResultsByInjectedChars(diversifyResultsBySource(...))
```

QMD 还会在 memory 和 sessions 同时存在时做来源多样化：

```text
extensions/memory-core/src/memory/qmd-manager.ts:2830
  -> diversifyResultsBySource(...)
```

如果 `corpus=all`，工具层还有一个合并上限：

```text
extensions/memory-core/src/tools.ts:369
  -> effectiveMax = max(1, maxResults ?? 10)

extensions/memory-core/src/tools.ts:370
  -> mergeMemorySearchCorpusResults(...)
```

注意这里的 `10` 是 tool 层合并 builtin manager + supplement 的默认返回上限；builtin manager 自身不传 `maxResults` 时仍然默认返回 6 条结果。

是否只会获取相关度最高的去 memory_get？

```text
不会自动获取。
```

`memory_search` 的代码只负责搜索和返回候选结果。真正读取原文的入口是另一个工具：

```text
extensions/memory-core/src/tools.ts:402
  -> name: "memory_get"

extensions/memory-core/src/tools.ts:464
  -> memory.manager.readFile(...)
```

所以系统不会在 `memory_search` 内部自动做：

```text
取 score 最高的一条
自动 memory_get
自动把原文加入上下文
```

实际是：

```text
模型看到搜索结果列表
        |
        v
模型根据 snippet / citation / score 判断哪条值得读
        |
        v
模型主动调用 memory_get
```

模型可能只读最高分那条，也可能读多条：

```text
1. 如果最高分结果已经足够明确，可能只读第一条。
2. 如果多个结果互相补充，可能读两三条。
3. 如果 snippet 已经足够回答，模型有时可能不再调用 memory_get。
   但 prompt 对同时有 memory_search + memory_get 的情况要求：
   search 后再用 memory_get 拉取需要的行。
```

### 搜索后端

OpenClaw 的 memory 搜索不是只有一种实现。

后端选择在：

- `extensions/memory-core/src/memory/search-manager.ts:148`：`getMemorySearchManager(...)`
- `extensions/memory-core/src/memory/search-manager.ts:319`：默认使用 builtin `MemoryIndexManager`

默认 builtin 后端是 SQLite 索引。
文档里说：

- `docs/concepts/memory-builtin.md:9`：builtin engine 是默认记忆引擎
- `docs/concepts/memory-builtin.md:14`：支持 FTS5 / BM25 关键字搜索
- `docs/concepts/memory-builtin.md:15`：支持 vector embeddings
- `docs/concepts/memory-builtin.md:16`：支持 hybrid search
- `docs/concepts/memory-builtin.md:79`：索引 `MEMORY.md` 和 `memory/*.md`，切成约 400 token 的 chunk，重叠约 80 token

这里要特别注意：

```text
SQLite 本体主要负责存储 chunks、FTS5 文本索引、embedding 数据。
OpenClaw 的向量检索不是“SQLite 裸库天然支持”的能力。
```

OpenClaw 有两条向量检索路径：

```text
1. sqlite-vec 可用：
   在 SQLite 里加载 sqlite-vec 扩展，创建 chunks_vec 虚拟表，用 KNN 查询做向量检索。

2. sqlite-vec 不可用：
   仍然把 embedding 存在 SQLite 的 chunks 表里，然后在 OpenClaw 进程内读出 embedding，
   用 cosine similarity 做 fallback 排序。
```

相关代码：

- `packages/memory-host-sdk/src/host/sqlite-vec.ts:29`：加载 `sqlite-vec` 扩展
- `packages/memory-host-sdk/src/host/sqlite-vec.ts:41` 到 `packages/memory-host-sdk/src/host/sqlite-vec.ts:43`：动态导入 `sqlite-vec` 并 load 到 SQLite
- `extensions/memory-core/src/memory/manager-sync-ops.ts:287`：创建 `chunks_vec` 向量虚拟表
- `extensions/memory-core/src/memory/manager-search.ts:136`：优先用 sqlite-vec 的 KNN 查询
- `extensions/memory-core/src/memory/manager-search.ts:247`：fallback 时用 `cosineSimilarity(...)`
- `docs/concepts/memory-builtin.md:129`：sqlite-vec 不可用时自动退回进程内 cosine similarity

所以更精确的说法是：

```text
OpenClaw 的 builtin memory 使用 SQLite 作为索引数据库；
向量检索由 embedding + sqlite-vec 扩展实现；
如果 sqlite-vec 不可用，则退回到进程内 cosine similarity。
```

搜索执行在：

- `extensions/memory-core/src/memory/manager.ts:304`：`search(...)`
- `extensions/memory-core/src/memory/manager.ts:437`：关键字搜索和向量搜索路径
- `extensions/memory-core/src/memory/manager.ts:462`：混合搜索合并结果
- `extensions/memory-core/src/memory/manager.ts:471`：按分数过滤
- `extensions/memory-core/src/memory/manager.ts:485`：关键字 fallback

也就是说，`memory_search` 的设计不是简单 grep。
它更接近：

```text
BM25 关键字搜索
    + 向量语义搜索
    + 混合排序
    + 分数过滤
    + citation 标注
    + 可选 corpus 合并
```

## 四、当前 memory_search 的真实检索逻辑：会不会读完整 Markdown

结论：

```text
正常情况下，不会。
一条消息需要查记忆时，memory_search 主要查 SQLite memory index。
它不会每次把 MEMORY.md / memory/*.md 全部打开，从头到尾扫一遍。
```

但是有两个重要例外：

```text
1. 如果索引还没有内容，第一次 search 会先同步建索引。
   这一步会扫描记忆 Markdown，因为系统必须先把 Markdown 切块并写入索引。

2. 如果索引被标记为 dirty，并且 sync.onSearch 开启，search 会触发后台同步。
   后台同步可能扫描记忆 Markdown，但当前这次搜索仍然主要使用已有索引返回结果。
```

所以要分清三件事：

```text
建索引 / 同步阶段：会扫描记忆 Markdown。
检索阶段：主要查已经建好的 SQLite index。
取原文阶段：memory_get 只读命中的那个文件片段。
```

一条消息需要查记忆时，实际链路是：

```text
用户消息

模型根据 prompt 判断需要查历史记忆

调用 memory_search

extensions/memory-core/src/tools.ts:299
  -> memory.manager.search(query, ...)

extensions/memory-core/src/memory/manager.ts:317
  -> 检查当前是否已经有 indexed content

如果没有 indexed content：
extensions/memory-core/src/memory/manager.ts:323
  -> await this.sync({ reason: "search", force: true })
  -> 强制同步一次，避免刚启动时索引为空导致查不到

如果已有 indexed content：
  -> 不会现场全文读取所有 Markdown
  -> 继续走索引检索
```

同步阶段会扫描 Markdown 文件，入口在：

```text
extensions/memory-core/src/memory/manager-sync-ops.ts:737
  -> listMemoryFiles(...)

packages/memory-host-sdk/src/host/internal.ts:139
  -> 真正列出 MEMORY.md / memory/*.md / extraPaths 等记忆文件
```

扫描出来以后，系统会构造 `fileEntries`，计算文件 hash，并和数据库里已有 hash 对比：

```text
extensions/memory-core/src/memory/manager-sync-ops.ts:762
  -> 读取 existingHashes

extensions/memory-core/src/memory/manager-sync-ops.ts:774
  -> 如果不是 full reindex，且 hash 没变，就跳过这个文件
```

也就是说，同步阶段即使会枚举记忆文件，也不是每次都无脑重建所有内容；未变化的文件会被跳过。

真正检索时，`MemoryIndexManager.search(...)` 走的是索引查询：

```text
extensions/memory-core/src/memory/manager.ts:371
  -> 如果没有 embedding provider，走 keyword / FTS 检索

extensions/memory-core/src/memory/manager.ts:439
  -> 有条件地跑 searchKeyword(...)

extensions/memory-core/src/memory/manager.ts:452
  -> 有条件地跑 searchVector(...)

extensions/memory-core/src/memory/manager.ts:462
  -> mergeHybridResults(...) 合并关键词结果和向量结果
```

其中向量检索也不是读 Markdown 原文再比对，而是查 SQLite 里的 chunk / embedding 数据：

```text
extensions/memory-core/src/memory/manager-search.ts:136
  -> 优先使用 sqlite-vec 的 KNN 查询

extensions/memory-core/src/memory/manager-search.ts:247
  -> sqlite-vec 不可用时，从 chunks 表读 embedding，在进程内计算 cosineSimilarity
```

搜索期间还有一个后台同步逻辑：

```text
extensions/memory-core/src/memory/manager.ts:338
  -> startAsyncSearchSync(...)

extensions/memory-core/src/memory/manager.ts:339
  -> enabled: this.settings.sync.onSearch

extensions/memory-core/src/memory/manager-async-state.ts:8
  -> 如果没有 dirty / sessionsDirty，就不做同步

extensions/memory-core/src/memory/manager-async-state.ts:11
  -> 如果需要同步，后台调用 sync({ reason: "search" })
```

这说明：

```text
search 可以顺手触发同步，但同步是为了刷新索引。
search 本身不是把所有 Markdown 全文读出来再让模型筛。
```

最后，如果搜索结果里有匹配项，模型可以再调用 `memory_get` 读取原文：

```text
extensions/memory-core/src/memory/manager.ts:741
  -> readMemoryFile(...)

packages/memory-host-sdk/src/host/read-file.ts:23
  -> 读取指定 path / from / lines 对应的记忆文件片段
```

所以可以用图书馆类比：

```text
memory_search = 查目录索引，找到可能相关的书和页码
memory_get    = 按目录结果打开其中一本书的指定页
```

不是：

```text
每次用户提问，都把所有记忆 Markdown 从头读一遍。
```

### SQLite memory index 里具体存了什么

结论：

```text
SQLite memory index 不是只存一整篇 Markdown。
它把记忆文件拆成 chunk，然后围绕 chunk 存路径、行号、文本、hash、embedding 和搜索索引。
```

表结构在：

```text
packages/memory-host-sdk/src/host/memory-schema.ts:4
  -> ensureMemoryIndexSchema(...)
```

主要表有这些：

```text
meta
  key
  value
```

`meta` 是元信息表，用来存储索引自身的元数据。

```text
files
  path
  source
  hash
  mtime
  size
```

`files` 存“文件级状态”：

```text
path   = 记忆文件路径，例如 MEMORY.md / memory/2026-05-14.md
source = 来源，常见是 memory，也可以是 sessions
hash   = 文件内容 hash
mtime  = 文件修改时间
size   = 文件大小
```

它的作用是判断文件有没有变。同步时会拿当前文件 hash 和 `files.hash` 对比，没变就跳过：

```text
extensions/memory-core/src/memory/manager-sync-ops.ts:762
extensions/memory-core/src/memory/manager-sync-ops.ts:774
```

最核心的是 `chunks` 表：

```text
chunks
  id
  path
  source
  start_line
  end_line
  hash
  model
  text
  embedding
  updated_at
```

你可以把一行 `chunks` 理解成：

```text
某个记忆文件里的一个片段
```

字段含义是：

```text
id         = chunk 的唯一 id
path       = 这个 chunk 来自哪个文件
source     = 来自 memory 文件，还是 session transcript
start_line = chunk 在原文件中的开始行
end_line   = chunk 在原文件中的结束行
hash       = chunk 文本 hash
model      = 用哪个 embedding model 建的索引；没有 embedding provider 时是 fts-only
text       = chunk 的文本内容
embedding  = chunk 的向量，JSON 字符串形式；没有 embedding 时是 []
updated_at = 写入/更新时间
```

切 chunk 的逻辑在：

```text
packages/memory-host-sdk/src/host/internal.ts:362
  -> chunkMarkdown(content, chunking)
```

这个函数会按配置估算 token/字符长度，把 Markdown 分成多个 chunk，并记录：

```text
startLine
endLine
text
hash
embeddingInput
```

写入 `chunks` 的逻辑在：

```text
extensions/memory-core/src/memory/manager-embedding-ops.ts:596
  -> INSERT INTO chunks (...)
```

写入前的入口是：

```text
extensions/memory-core/src/memory/manager-embedding-ops.ts:645
  -> indexFile(...)
```

如果没有 embedding provider：

```text
extensions/memory-core/src/memory/manager-embedding-ops.ts:656
  -> chunkMarkdown(...)

extensions/memory-core/src/memory/manager-embedding-ops.ts:660
  -> writeChunks(..., "fts-only", chunks, [], false)
```

这时 SQLite 仍然有 chunk 文本和 FTS 索引，但没有真实 embedding。

如果有 embedding provider：

```text
extensions/memory-core/src/memory/manager-embedding-ops.ts:686
  -> chunkMarkdown(...)

extensions/memory-core/src/memory/manager-embedding-ops.ts:701
  -> embed chunks

extensions/memory-core/src/memory/manager-embedding-ops.ts:728
  -> writeChunks(..., provider.model, chunks, embeddings, vectorReady)
```

这时每个 chunk 会多出 embedding，用于语义向量检索。

还有一个 FTS5 虚拟表：

```text
chunks_fts
  text
  id
  path
  source
  model
  start_line
  end_line
```

表结构在：

```text
packages/memory-host-sdk/src/host/memory-schema.ts:66
  -> CREATE VIRTUAL TABLE ... USING fts5(...)
```

写入位置在：

```text
extensions/memory-core/src/memory/manager-embedding-ops.ts:628
  -> INSERT INTO chunks_fts (...)
```

关键词搜索主要查 `chunks_fts`：

```text
extensions/memory-core/src/memory/manager-search.ts:326
  -> FROM chunks_fts WHERE chunks_fts MATCH ?
```

所以关键词搜索不是读 Markdown 文件，而是查 FTS 表里的 chunk 文本。

如果 sqlite-vec 可用，还会有一个向量虚拟表：

```text
chunks_vec
  id
  embedding
```

表结构在：

```text
extensions/memory-core/src/memory/manager-sync-ops.ts:287
  -> CREATE VIRTUAL TABLE IF NOT EXISTS chunks_vec USING vec0(...)
```

写入位置在：

```text
extensions/memory-core/src/memory/manager-embedding-ops.ts:618
  -> replaceMemoryVectorRow(...)

extensions/memory-core/src/memory/manager-vector-write.ts:20
  -> INSERT INTO chunks_vec (id, embedding)
```

向量搜索时，`chunks_vec` 只负责按 embedding 找近似相似的 id，真正返回结果还要 join 回 `chunks` 表拿路径、行号和文本：

```text
extensions/memory-core/src/memory/manager-search.ts:150
  -> FROM chunks_vec v JOIN chunks c ON c.id = v.id
```

如果 sqlite-vec 不可用，也不会丢掉 embedding。系统会从 `chunks.embedding` 读出 JSON embedding，在进程内算相似度：

```text
extensions/memory-core/src/memory/manager-search.ts:231
  -> SELECT id, path, start_line, end_line, text, embedding, source FROM chunks

extensions/memory-core/src/memory/manager-search.ts:247
  -> cosineSimilarity(queryVec, parseEmbedding(row.embedding))
```

还有一个 embedding 缓存表：

```text
embedding_cache
  provider
  model
  provider_key
  hash
  embedding
  dims
  updated_at
```

表结构在：

```text
packages/memory-host-sdk/src/host/memory-schema.ts:43
  -> CREATE TABLE IF NOT EXISTS embedding_cache (...)
```

写入位置在：

```text
extensions/memory-core/src/memory/manager-embedding-cache.ts:72
  -> INSERT INTO embedding_cache (...)
```

它的作用是避免同一个 chunk 内容反复请求 embedding provider。只要 provider、model、provider_key、chunk hash 一样，就可以复用之前算过的 embedding。

所以，SQLite memory index 可以理解成四层：

```text
files
  文件级索引：这个文件是谁，是否变化

chunks
  内容级索引：文件被拆成哪些片段，每个片段的文本、行号、hash、embedding

chunks_fts
  关键词索引：给 FTS5 / BM25 用

chunks_vec
  向量索引：给 sqlite-vec KNN 用

embedding_cache
  embedding 缓存：避免重复计算向量
```

最重要的是：

```text
memory_search 查 SQLite 时，主要查的是 chunk 级数据。
返回结果里的 path / startLine / endLine / snippet，都是从这些 chunk 索引数据来的。
```

### corpus 搜索范围是怎么来的

结论：

```text
corpus 是 memory_search 工具调用时的可选参数。
它不是 SQLite 自动判断出来的，也不是 manager 从用户消息里直接解析出来的。
```

工具 schema 定义在：

```text
extensions/memory-core/src/tools.shared.ts:30
  -> MemorySearchSchema
```

允许的值是：

```text
corpus = memory | wiki | all | sessions
```

在插件入口里也有同一份懒加载 schema：

```text
extensions/memory-core/index.ts:48
  -> MemorySearchSchema
```

上游怎么知道要查哪个 corpus？

第一层来自 prompt 和 tool description。

prompt 会告诉模型：

```text
extensions/memory-core/src/prompt-section.ts:17
  -> 遇到 prior work / decisions / dates / people / preferences / todos 时，
     先 run memory_search on MEMORY.md + memory/*.md + indexed session transcripts
```

tool description 又进一步告诉模型：

```text
extensions/memory-core/src/tools.ts:240
  -> corpus=memory 限制到 indexed memory files
  -> corpus=sessions 限制到 indexed session transcripts
  -> corpus=wiki / corpus=all 查 registered compiled-wiki supplements
```

所以“要不要传 corpus、传 memory 还是 sessions”，通常是模型根据工具描述和用户问题自己决定的。

例如：

```json
{
  "query": "上次我们怎么处理 Telegram webhook？"
}
```

如果不传 `corpus`，代码会把它当作默认 memory search：

```text
extensions/memory-core/src/tools.ts:249
  -> requestedCorpus = readStringParam(rawParams, "corpus")

extensions/memory-core/src/tools.ts:255
  -> shouldQueryMemory = requestedCorpus !== "wiki"
```

也就是说：

```text
corpus 不传：
  查 builtin memory manager
  不查 wiki supplement
```

但这里要注意：builtin memory manager 里面可能包含两个 source：

```text
source = memory
source = sessions
```

`corpus` 和 `source` 不是同一个概念：

```text
corpus
  是 memory_search 工具入参层面的搜索范围。

source
  是 builtin memory index 内部每条 chunk 的来源。
```

`corpus` 到 `source` 的映射在：

```text
extensions/memory-core/src/tools.ts:299
```

代码逻辑是：

```text
requestedCorpus === "sessions"
  -> searchSources = ["sessions"]

requestedCorpus === "memory"
  -> searchSources = ["memory"]

requestedCorpus undefined 或 "all"
  -> 不传 sources 给 manager
  -> manager 使用自己配置里启用的所有 sources
```

对应代码：

```text
extensions/memory-core/src/tools.ts:293
  -> requestedCorpus === "sessions" ? ["sessions"]
     : requestedCorpus === "memory" ? ["memory"]
     : undefined

extensions/memory-core/src/tools.ts:307
  -> ...(searchSources ? { sources: searchSources } : {})
```

manager 收到 `sources` 后，会再和自身已启用的 sources 取交集：

```text
extensions/memory-core/src/memory/manager.ts:354
  -> opts.sources 去重后，只保留 this.sources 里启用的 source

extensions/memory-core/src/memory/manager.ts:363
  -> 没有显式 sources 时，使用 [...this.sources]
```

所以，最终是否能查到 sessions，不只看模型有没有传 `corpus: "sessions"`，还要看配置里有没有启用 sessions source。

builtin memory 的 source 配置来自：

```text
src/agents/memory-search.ts:119
  -> DEFAULT_SOURCES = ["memory"]

src/agents/memory-search.ts:121
  -> normalizeSources(...)

src/agents/memory-search.ts:179
  -> experimental.sessionMemory 默认 false

src/agents/memory-search.ts:241
  -> sources = normalizeSources(overrides?.sources ?? defaults?.sources, sessionMemory)
```

配置类型在：

```text
src/config/types.tools.ts:351
  -> memorySearch.sources?: Array<"memory" | "sessions">

src/config/types.tools.ts:371
  -> memorySearch.experimental.sessionMemory?: boolean
```

这说明：

```text
默认只启用 source=memory。
即使配置 sources 里写了 sessions，如果 experimental.sessionMemory 没开，normalizeSources 也不会加入 sessions。
```

换句话说，builtin 后端要搜索 session transcript，通常需要同时满足：

```text
agents.defaults.memorySearch.sources 或 agent.memorySearch.sources 包含 "sessions"

并且：

agents.defaults.memorySearch.experimental.sessionMemory
或 agent.memorySearch.experimental.sessionMemory 为 true
```

否则 `corpus: "sessions"` 最终会因为 manager 没有启用 sessions source 而返回空结果。

QMD 后端的 session source 逻辑略有不同。

QMD 的普通 memory collections 来自：

```text
packages/memory-host-sdk/src/host/backend-config.ts:342
  -> resolveDefaultCollections(...)
```

这些默认 collection 的 kind 是：

```text
kind = "memory"
```

如果启用了 QMD sessions：

```text
packages/memory-host-sdk/src/host/backend-config.ts:262
  -> resolveSessionConfig(...)

extensions/memory-core/src/memory/qmd-manager.ts:386
  -> this.sessionExporter = this.qmd.sessions.enabled ? ...

extensions/memory-core/src/memory/qmd-manager.ts:399
  -> 追加 kind: "sessions" 的 session collection
```

QMD 搜索时也会把工具传入的 `sources` 变成 collection 过滤：

```text
extensions/memory-core/src/memory/qmd-manager.ts:1104
  -> requestedSources = opts.sources

extensions/memory-core/src/memory/qmd-manager.ts:1105
  -> listManagedCollectionNames(requestedSources)

extensions/memory-core/src/memory/qmd-manager.ts:3075
  -> 只保留 kind 在 sources 里的 collection
```

所以 QMD 后端里：

```text
corpus=sessions
  -> sources=["sessions"]
  -> 只搜 kind=sessions 的 QMD collection

corpus=memory
  -> sources=["memory"]
  -> 只搜 kind=memory 的 QMD collection
```

还有 wiki / all：

```text
extensions/memory-core/src/tools.ts:256
  -> shouldQuerySupplements = requestedCorpus === "wiki" || requestedCorpus === "all"

extensions/memory-core/src/tools.shared.ts:152
  -> searchMemoryCorpusSupplements(...)
```

规则是：

```text
corpus=wiki
  -> 不查 builtin memory manager
  -> 只查 supplement corpus，比如 wiki

corpus=all
  -> 查 builtin memory manager
  -> 再查 supplement corpus
  -> 最后 mergeMemorySearchCorpusResults(...) 合并

corpus=memory
  -> 只查 builtin source=memory
  -> 不查 wiki supplement

corpus=sessions
  -> 只查 builtin source=sessions
  -> 不查 wiki supplement

corpus 不传
  -> 查 builtin memory manager
  -> manager 使用已启用的所有 source
  -> 不查 wiki supplement
```

注意：这里的 builtin memory manager 不是固定等于“只查 memory 文件”。
如果配置启用了 session memory，并且内部 `sources` 包含 `sessions`，它也可能返回 session transcript 命中。
所以 `corpus=all` 更准确地说是“builtin manager 的结果 + supplement/wiki 的结果”。

最后还有一道 session 可见性过滤：

```text
extensions/memory-core/src/tools.ts:309
  -> filterMemorySearchHitsBySessionVisibility(...)

extensions/memory-core/src/session-search-visibility.ts:38
  -> 非 sessions 命中直接保留
  -> sessions 命中必须通过 session visibility guard
```

这意味着：

```text
就算索引里搜到了 sessions 结果，也不一定会返回给当前请求者。
它还要满足当前 requesterSessionKey 的 session history 可见性规则。
```

所以完整判断链路是：

```text
模型/tool call 是否传 corpus
        |
        v
tools.ts 把 corpus 转成 shouldQueryMemory / shouldQuerySupplements / searchSources
        |
        v
manager 根据 searchSources 和自身 this.sources 做过滤
        |
        v
SQLite/QMD 只查对应 source/collection
        |
        v
tools.ts 对 sessions 命中再做 session visibility 过滤
```

### corpus 工具设计逻辑：不是代码猜问题，而是模型选择 + 代码路由

这个设计最容易误解的一点是：

```text
OpenClaw 代码本身并没有写一个规则：
如果用户说“上次会话”就一定查 sessions；
如果用户说“文档”就一定查 wiki；
如果用户说“记忆”就一定查 memory。
```

真正的设计是：

```text
模型负责理解用户问题，决定 memory_search 的 corpus 参数。
代码负责验证 corpus 参数，并把它路由到对应的数据源。
```

第一步：工具 schema 把可选范围限制死。

```text
extensions/memory-core/src/tools.shared.ts:30
  -> MemorySearchSchema
```

schema 只允许：

```text
memory
wiki
all
sessions
```

这意味着模型不能随便传：

```text
corpus = "chat"
corpus = "database"
corpus = "history"
```

第二步：工具描述告诉模型每个 corpus 的语义。

```text
extensions/memory-core/src/tools.ts:240
```

这段 description 明确写了：

```text
corpus=memory
  -> indexed memory files

corpus=sessions
  -> indexed session transcripts

corpus=wiki
  -> registered compiled-wiki supplements

corpus=all
  -> builtin manager + supplement
```

所以模型为什么“能够判断”？

因为模型在生成工具调用时能看到：

```text
1. 用户问题
2. memory_search 的工具名
3. memory_search 的 description
4. MemorySearchSchema 允许的 corpus 枚举
```

它会基于自然语言语义选择参数。例如：

```text
用户问：我们在 MEMORY.md 里记了什么偏好？
模型可能调用：
memory_search({ query: "...", corpus: "memory" })

用户问：上个会话里我让你修了什么？
模型可能调用：
memory_search({ query: "...", corpus: "sessions" })

用户问：查一下项目 wiki 里关于 agent routing 的说明
模型可能调用：
memory_search({ query: "...", corpus: "wiki" })

用户问：把记忆和 wiki 都查一下
模型可能调用：
memory_search({ query: "...", corpus: "all" })
```

第三步：代码不再“理解问题”，只做确定性路由。

```text
extensions/memory-core/src/tools.ts:249
  -> 读取 requestedCorpus

extensions/memory-core/src/tools.ts:256
  -> requestedCorpus !== "wiki" 时查 builtin memory manager

extensions/memory-core/src/tools.ts:257
  -> requestedCorpus === "wiki" 或 "all" 时查 supplement corpus
```

然后再把 `corpus` 映射成 builtin memory 内部的 `sources`：

```text
extensions/memory-core/src/tools.ts:293
  -> corpus=sessions 映射成 sources=["sessions"]
  -> corpus=memory 映射成 sources=["memory"]
  -> corpus 不传或 corpus=all 时，不显式传 sources
```

也就是说，代码里的判断是枚举分支，不是语义推理：

```text
if corpus === "sessions":
  searchSources = ["sessions"]
else if corpus === "memory":
  searchSources = ["memory"]
else:
  searchSources = undefined
```

第四步：manager 只接受已经启用的 source。

```text
extensions/memory-core/src/memory/manager.ts:354
  -> opts.sources 和 this.sources 取交集

extensions/memory-core/src/memory/manager.ts:363
  -> 没传 sources 时，使用 this.sources 的全部已启用 source
```

所以即使模型传了：

```json
{ "corpus": "sessions" }
```

如果配置没有启用 sessions source，manager 也不会搜索 sessions。

第五步：wiki/all 走 supplement 机制。

```text
extensions/memory-core/src/tools.ts:359
  -> shouldQuerySupplements 时调用 searchMemoryCorpusSupplements(...)

extensions/memory-core/src/tools.shared.ts:152
  -> searchMemoryCorpusSupplements(...)
```

supplement 的设计逻辑是：memory-core 不把 wiki 硬编码成 SQLite 的 source，而是通过注册的 supplement 扩展提供额外语料。这样 `wiki` 可以独立于 builtin memory index。

第六步：all 是组合查询。

```text
extensions/memory-core/src/tools.ts:370
  -> mergeMemorySearchCorpusResults(...)

extensions/memory-core/src/tools.ts:374
  -> requestedCorpus === "all" 时 balanceCorpora
```

也就是说：

```text
corpus=all
  不是一个新的底层索引
  而是同时查 builtin manager 和 supplement corpus
  再把两个结果集合并、排序、平衡
```

所以这个工具的整体设计可以概括成：

```text
schema 约束可选值
        |
        v
description 告诉模型每个值是什么意思
        |
        v
模型根据用户问题选择 corpus
        |
        v
tools.ts 把 corpus 映射成 shouldQueryMemory / shouldQuerySupplements / sources
        |
        v
manager/supplement 执行真正搜索
        |
        v
session 结果再经过可见性过滤
```

这种设计的优点是：

```text
1. 工具接口简单：只暴露一个 corpus 参数。
2. 模型有语义判断能力，可以根据用户问题选择范围。
3. 代码路径确定：拿到 corpus 后就是明确的枚举路由。
4. 安全边界清楚：sessions 结果还要过 session visibility guard。
5. 可扩展：wiki 不需要塞进 SQLite source，可以通过 supplement 注册。
```

缺点是：

```text
模型可能选错 corpus。
比如应该查 sessions 时没传 corpus，或者应该查 memory 时传了 sessions。
所以工具描述和 prompt 写得越清楚，模型选择就越稳定。
```

## 五、memory_get 的设计

`memory_get` 负责“读”。

工具创建位置：

- `extensions/memory-core/src/tools.ts:393`：`createMemoryGetTool(...)`

参数定义在：

- `extensions/memory-core/index.ts:73`：`MemoryGetSchema`

它主要接收：

```text
path   = 要读取的记忆文件路径
from   = 从第几行开始
lines  = 读取多少行
corpus = memory / wiki / all
```

注意这里没有 `sessions`。
这不是漏写，而是代码里的真实工具边界：`MemorySearchSchema` 支持 `sessions`，但 `MemoryGetSchema` 不支持。

执行主线在：

- `extensions/memory-core/src/tools.ts:413`：解析 `corpus`
- `extensions/memory-core/src/tools.ts:418`：加载读取函数和后端配置
- `extensions/memory-core/src/tools.ts:438`：builtin 后端读取 agent memory file
- `extensions/memory-core/src/tools.ts:464`：非 builtin 后端通过 manager readFile
- `extensions/memory-core/src/tools.ts:206`：把读取错误转换成 JSON 结果

实际文件读取在 memory host SDK：

- `packages/memory-host-sdk/src/host/read-file.ts:23`：`readMemoryFile(...)`
- `packages/memory-host-sdk/src/host/read-file.ts:41`：只允许读取 memory path
- `packages/memory-host-sdk/src/host/read-file.ts:69`：拒绝不在允许范围内的 path
- `packages/memory-host-sdk/src/host/read-file.ts:99`：构造有边界的读取结果
- `packages/memory-host-sdk/src/host/read-file.ts:110`：`readAgentMemoryFile(...)`
- `packages/memory-host-sdk/src/host/read-file.ts:128`：使用默认行数和最大字符限制

默认读取限制在：

- `packages/memory-host-sdk/src/host/read-file-shared.ts:3`：默认读取 120 行
- `packages/memory-host-sdk/src/host/read-file-shared.ts:4`：默认最大 12000 字符
- `packages/memory-host-sdk/src/host/read-file-shared.ts:49`：如果内容被截断，返回 continuation 和 nextFrom

所以 `memory_get` 不是任意文件读取工具。
它是一个有边界的“记忆文件读取工具”。

## 六、为什么要分成 search 和 get

如果只有 `memory_get`，模型必须已经知道文件路径。
但模型通常不知道应该读哪个文件。

如果只有 `memory_search`，模型只能看到片段，不能确认上下文是否完整。

所以两个工具组合起来：

```text
memory_search 解决“我该看哪里”
memory_get    解决“原文到底怎么说”
```

这是一种常见的 RAG 设计：

```text
retrieval first, source read second
```

中文可以理解为：

```text
先检索，再取证。
```

## 七、它的安全边界

### 1. 工具不是总是存在

工具注册前会检查 memory context：

- `extensions/memory-core/index.ts:48`：`hasMemoryToolContext(...)`

没有配置时，不暴露工具。

### 2. 读取路径有限制

memory 文件范围在：

- `packages/memory-host-sdk/src/host/internal.ts:92`：`isMemoryPath(...)`
- `packages/memory-host-sdk/src/host/internal.ts:139`：`listMemoryFiles(...)`
- `packages/memory-host-sdk/src/host/internal.ts:163`：列出 root `MEMORY.md`
- `packages/memory-host-sdk/src/host/internal.ts:170`：列出 `memory/`

默认 memory path 主要包括：

```text
MEMORY.md
memory/**/*.md
```

不是任意项目文件。

### 3. session 搜索有可见性过滤

会话 transcript 可以被索引，但不是无条件暴露。

相关代码：

- `extensions/memory-core/src/session-search-visibility.ts:14`：session visibility 过滤
- `extensions/memory-core/src/session-search-visibility.ts:26`：创建 visibility guard
- `extensions/memory-core/src/session-search-visibility.ts:59`：只有 guard 允许的 session hit 才返回

### 4. 返回内容有长度边界

读取结果会按行数和字符预算截断：

- `packages/memory-host-sdk/src/host/read-file-shared.ts:39`：字符预算适配
- `packages/memory-host-sdk/src/host/read-file-shared.ts:49`：返回 continuation 信息

## 八、设计优点

### 优点 1：节省上下文

记忆可能很多。
如果每次都把所有 `MEMORY.md` 和 `memory/*.md` 注入 prompt，会浪费大量 token。

现在的方式是：

```text
只在需要时搜索
只读取命中的部分
```

### 优点 2：比纯关键字更容易找回相关内容

文档说明：

- `docs/concepts/memory-search.md:73`：vector search 用于语义相近的内容
- `docs/concepts/memory-search.md:75`：BM25 keyword search 用于精确词匹配

所以即使用户措辞和记忆文件里的措辞不同，也可能搜到。

### 优点 3：可追溯

`memory_search` 会给结果加 citation：

- `extensions/memory-core/src/tools.citations.ts:17`：`decorateCitations(...)`
- `extensions/memory-core/src/tools.citations.ts:31`：citation 形如 `<path>#Lx-Ly`

这让回答可以回到文件和行号，而不是“模型好像记得”。

### 优点 4：可扩展

搜索后端不是写死的。
可以走 builtin，也可以走 QMD 等后端。

相关代码：

- `extensions/memory-core/src/memory/search-manager.ts:223`：QMD manager 和 fallback manager
- `extensions/memory-core/src/memory/search-manager.ts:228`：fallback 使用 builtin `MemoryIndexManager`

这说明 OpenClaw 把“工具接口”和“搜索实现”分开了。

### 优点 5：失败时能降级

如果 embedding provider 不可用，搜索可以返回 warning / fallback 信息，而不是直接让整个 agent 崩溃。

相关代码：

- `extensions/memory-core/src/tools.shared.ts:118`：构造 unavailable search result
- `extensions/memory-core/src/memory/manager.ts:485`：关键字 fallback

### 优点 6：可以形成后续学习信号

搜索命中会记录 recall 信号：

- `extensions/memory-core/src/tools.ts:106`：recall tracking 是 best-effort
- `extensions/memory-core/src/tools.ts:335`：搜索后 queue recall touch

这给后续 dreaming / promote 这类整理机制提供输入。

更具体地说，这个信号不是为了改变本次 `memory_search` 的返回结果。
它的作用是把“这段短期记忆被真实问题搜到过”记录下来，供后续长期化判断使用。

代码链路是：

```text
memory_search 命中 memory 结果
  -> extensions/memory-core/src/tools.ts:335
  -> queueShortTermRecallTracking(...)
  -> extensions/memory-core/src/short-term-promotion.ts:910
  -> recordShortTermRecalls(...)
  -> 写入 memory/.dreams/short-term-recall.json
```

`queueShortTermRecallTracking(...)` 里面用 `void recordShortTermRecalls(...).catch(...)`：

- `extensions/memory-core/src/tools.ts:114`：异步触发记录
- `extensions/memory-core/src/tools.ts:120`：注释明确说 recall tracking 是 best-effort，不能阻塞正常 memory recall

所以它是“后台统计”，不是“搜索必要步骤”。
即使写入失败，本次搜索仍然要正常返回。

写进去的核心字段在 `ShortTermRecallEntry`：

- `extensions/memory-core/src/short-term-promotion.ts:65`：entry 结构定义
- `path / startLine / endLine`：命中的短期记忆位置
- `snippet`：命中的内容片段
- `recallCount`：被用户查询召回的次数
- `dailyCount`：dreaming 日常扫描带来的信号次数
- `totalScore / maxScore`：搜索相关度累计和最高值
- `queryHashes`：不同查询的哈希，用来判断是不是不同问题都找到了它
- `recallDays`：在哪些日期被召回，用来判断是否跨天重复出现
- `conceptTags`：概念标签，供 promotion 评分使用

它不是记录所有搜索结果。
`recordShortTermRecalls(...)` 会过滤：

- `extensions/memory-core/src/short-term-promotion.ts:928`：只保留 `source === "memory"` 且 `isShortTermMemoryPath(...)` 的结果
- `extensions/memory-core/src/short-term-promotion.ts:859`：`isShortTermMemoryPath(...)` 排除 `memory/dreaming/`，只接受短期日记、短期 session corpus 等路径
- `extensions/memory-core/src/short-term-promotion.ts:947`：跳过空 snippet 或被 dreaming 内容污染的 snippet

为什么要这样设计？

因为 OpenClaw 需要区分：

```text
短期记忆里偶然出现过一次
长期记忆里应该稳定保存
```

一次搜索命中还不够说明它值得长期保存。
但如果同一段内容多次被用户问题召回、跨日期出现、相关度高、被不同查询命中，就说明它可能是稳定偏好、重要决策或反复有用的事实。

后续 promotion 评分会读取这份 store：

- `extensions/memory-core/src/short-term-promotion.ts:1235`：遍历 short-term recall entries
- `extensions/memory-core/src/short-term-promotion.ts:1255`：计算平均相关度 `avgScore`
- `extensions/memory-core/src/short-term-promotion.ts:1256`：计算频率分
- `extensions/memory-core/src/short-term-promotion.ts:1257`：统计不同 query 数量
- `extensions/memory-core/src/short-term-promotion.ts:1273`：计算跨天 consolidation 分
- `extensions/memory-core/src/short-term-promotion.ts:1280`：按 frequency / relevance / diversity / recency / consolidation / conceptual 合成最终 promotion score

最后如果分数和门槛达标，`applyShortTermPromotions(...)` 会把候选追加到 `MEMORY.md`：

- `extensions/memory-core/src/short-term-promotion.ts:1551`：`applyShortTermPromotions(...)`
- `extensions/memory-core/src/short-term-promotion.ts:1641`：有待追加内容时写入 `MEMORY.md`
- `extensions/memory-core/src/short-term-promotion.ts:1659`：把 entry 标记为 `promotedAt`

所以 `tools.ts:335` 的真正目的可以总结为：

```text
本次 search 找到的短期记忆
  -> 记录成 recall signal
  -> 积累“被反复需要”的证据
  -> dreaming / memory promote 根据这些证据做长期化
  -> 合格内容进入 MEMORY.md
```

测试也验证了这个设计：

- `extensions/memory-core/src/tools.recall-tracking.test.ts:83`：搜索后会调用 `recordShortTermRecalls(...)`
- `extensions/memory-core/src/tools.recall-tracking.test.ts:92`：慢速 recall 写入不会阻塞工具返回
- `extensions/memory-core/src/tools.recall-tracking.test.ts:140`：会把 dreaming timezone 传入 recall tracking
- `extensions/memory-core/src/tools.citations.test.ts:217`：搜索命中会持久化到 `memory/.dreams/short-term-recall.json`

## 九、设计缺点

### 缺点 1：搜索可能漏掉

`memory_search` 依赖：

```text
索引是否新鲜
chunk 切分是否合适
embedding 是否可用
query 写得是否清楚
分数阈值是否合理
```

所以它可能漏掉真正相关的记忆。

### 缺点 2：搜索可能误命中

语义搜索和混合排序会返回“看起来相关”的内容。
这不等于它一定正确。

所以才需要 `memory_get` 再读原文确认。

### 缺点 3：多一次工具调用，延迟更高

理想流程是：

```text
memory_search -> memory_get -> 回答
```

比直接回答多了至少一到两次工具调用。
这会增加延迟。

### 缺点 4：依赖模型主动使用工具

prompt 会提醒模型用工具，但模型仍然可能忘记搜。

也就是说，设计上给了能力，但不是强制所有历史相关问题都必须调用。

### 缺点 5：文件型记忆可能变乱

默认 memory 是 Markdown 文件：

```text
MEMORY.md
memory/*.md
```

优点是透明、容易人工编辑。
缺点是时间久了可能出现：

```text
重复记录
过期记录
互相矛盾
格式不统一
同一事实散落多处
```

搜索工具只负责找回，不负责自动判断哪条才是最终事实。

### 缺点 6：session 记忆有隐私和可见性复杂度

文档说明 session transcript 可以被索引：

- `docs/concepts/memory-search.md:135`
- `docs/reference/memory-config.md:439`

这有助于找回历史对话，但也带来隐私、权限、可见性和过期管理问题。

所以代码里必须有 session visibility guard。

### 缺点 7：结果会被截断

为了控制上下文，`memory_get` 有默认行数和字符限制。
这意味着模型可能需要继续用 `nextFrom` 读下一段。

如果模型没有继续读，就可能只看到部分上下文。

## 十、一个具体例子：用户问“上次我们怎么处理 Telegram webhook 的？”

假设 Agent workspace 里有这些记忆文件：

```text
MEMORY.md
memory/2026-05-02.md
memory/2026-05-10.md
```

其中 `memory/2026-05-10.md` 里有一段：

```markdown
## Telegram webhook 调试

用户的 Telegram bot 收不到消息。最后确认原因是 webhook URL 配置到了旧域名。
处理步骤：
1. 用 openclaw channels status telegram 检查账号状态。
2. 用 Telegram getWebhookInfo 确认当前 webhook URL。
3. 重新设置 webhook 到新的 Gateway public URL。
4. 发送测试消息确认 Gateway 收到 update。
```

现在用户问：

```text
上次 Telegram 收不到消息，我们最后是怎么修的？
```

### 第一步：模型判断要查记忆

这个问题里有“上次”“最后怎么修的”，明显是在问历史上下文。
`memory-core` 的 prompt 会提醒模型：

- `extensions/memory-core/src/prompt-section.ts:17`：遇到历史工作、决策、日期、人物、偏好、待办等问题，先 `memory_search`，再 `memory_get`

所以模型会先调用：

```json
{
  "query": "Telegram 收不到消息 最后怎么修 webhook",
  "maxResults": 5,
  "corpus": "memory"
}
```

### 第二步：memory_search 进入工具实现

工具入口在：

- `extensions/memory-core/src/tools.ts:228`：`createMemorySearchTool(...)`
- `extensions/memory-core/src/tools.ts:299`：调用 `memory.manager.search(...)`

这一步不是直接读所有 Markdown 文件，而是搜索已经建立好的 memory index。

可以理解成：

```text
memory_search(query)
        |
        v
memory.manager.search(query)
        |
        v
SQLite memory index
```

### 第三步：查询文本被送入两条搜索路径

在 builtin memory backend 里，搜索主要有两条路径。

第一条是关键字搜索：

```text
Telegram / webhook / 收不到 / 修
        |
        v
FTS5 / BM25
        |
        v
找到包含这些词的 chunk
```

相关代码：

- `extensions/memory-core/src/memory/manager.ts:438`：hybrid 可用时执行 keyword search
- `extensions/memory-core/src/memory/manager-search.ts:340`：FTS5 MATCH 失败时可以 fallback

第二条是向量语义搜索：

```text
"Telegram 收不到消息 最后怎么修 webhook"
        |
        v
embedding provider 把查询变成向量
        |
        v
和 memory chunks 的 embedding 比相似度
```

相关代码：

- `extensions/memory-core/src/memory/manager.ts:451`：执行 vector search
- `extensions/memory-core/src/memory/manager-search.ts:136`：优先用 sqlite-vec 的 KNN 查询
- `extensions/memory-core/src/memory/manager-search.ts:247`：sqlite-vec 不可用时，用进程内 `cosineSimilarity(...)`

所以，即使记忆里写的是“bot 收不到消息”，用户问的是“Telegram 没反应”，向量检索也可能找出来，因为语义接近。

### 第四步：两路结果合并成最终排序

如果 hybrid search 开启，OpenClaw 会把关键字结果和向量结果合并。

相关代码：

- `extensions/memory-core/src/memory/manager.ts:462`：调用 `mergeHybridResults(...)`
- `extensions/memory-core/src/memory/manager.ts:465`：使用 `vectorWeight`
- `extensions/memory-core/src/memory/manager.ts:466`：使用 `textWeight`
- `extensions/memory-core/src/memory/manager.ts:467`：可选 MMR 去重/多样性
- `extensions/memory-core/src/memory/manager.ts:468`：可选 temporal decay

可以想象候选结果像这样：

```text
候选 A：
  path: memory/2026-05-10.md
  lines: 1-9
  keywordScore: 0.82
  vectorScore: 0.91
  finalScore: 0.88

候选 B：
  path: MEMORY.md
  lines: 20-28
  keywordScore: 0.55
  vectorScore: 0.60
  finalScore: 0.58

候选 C：
  path: memory/2026-05-02.md
  lines: 40-48
  keywordScore: 0.40
  vectorScore: 0.30
  finalScore: 0.35
```

最终 `memory_search` 返回的结果会类似：

```json
{
  "results": [
    {
      "path": "memory/2026-05-10.md",
      "from": 1,
      "to": 9,
      "score": 0.88,
      "snippet": "用户的 Telegram bot 收不到消息。最后确认原因是 webhook URL 配置到了旧域名...",
      "citation": "memory/2026-05-10.md#L1-L9"
    }
  ],
  "mode": "hybrid"
}
```

注意：这一步返回的是“命中的片段”，不是完整文件。

### 第五步：模型用 memory_get 读取原文

模型拿到 `path` 和行号后，再调用：

```json
{
  "path": "memory/2026-05-10.md",
  "from": 1,
  "lines": 12,
  "corpus": "memory"
}
```

`memory_get` 的入口在：

- `extensions/memory-core/src/tools.ts:393`：`createMemoryGetTool(...)`
- `extensions/memory-core/src/tools.ts:438`：builtin 后端读取 agent memory file
- `packages/memory-host-sdk/src/host/read-file.ts:23`：`readMemoryFile(...)`

返回结果会是原文片段，比如：

```markdown
## Telegram webhook 调试

用户的 Telegram bot 收不到消息。最后确认原因是 webhook URL 配置到了旧域名。
处理步骤：
1. 用 openclaw channels status telegram 检查账号状态。
2. 用 Telegram getWebhookInfo 确认当前 webhook URL。
3. 重新设置 webhook 到新的 Gateway public URL。
4. 发送测试消息确认 Gateway 收到 update。
```

### 第六步：模型基于原文回答

最后模型回答用户：

```text
上次的原因是 Telegram webhook 还指向旧域名。
我们当时先检查 Telegram 账号状态，再查 getWebhookInfo，确认 webhook URL 不对；
之后把 webhook 重新设置到新的 Gateway public URL，并发测试消息确认 Gateway 收到了 update。
来源：memory/2026-05-10.md#L1-L9
```

这就是完整链路：

```text
用户问题
  |
  v
memory_search
  |
  v
FTS5/BM25 关键字搜索
  +
embedding 向量搜索
  |
  v
hybrid 合并排序
  |
  v
返回 path / line / snippet / citation
  |
  v
memory_get
  |
  v
读取原文
  |
  v
基于原文回答
```

### 如果没有 embedding provider 会怎样

如果没有可用的 embedding provider，OpenClaw 仍然能用 FTS5/BM25 做关键词检索。

也就是：

```text
没有 embedding：
    还能搜 Telegram / webhook 这种精确词。

有 embedding：
    除了精确词，还能搜到“没反应”“收不到消息”“回调地址错了”这类语义相近内容。
```

这也是为什么第 20 篇前面说：

```text
memory_search 不是简单 grep；
它可以是 BM25 + vector + hybrid。
```

## 十一、新手理解版

你可以把整个机制想成下面这张图：

```text
用户问题
  |
  v
模型判断：这是不是和历史、偏好、之前做过的事有关？
  |
  v
调用 memory_search
  |
  v
搜索 SQLite / QMD / wiki / sessions 索引
  |
  v
返回候选片段 + path + 行号 + citation
  |
  v
调用 memory_get
  |
  v
读取 MEMORY.md 或 memory/*.md 中的原文片段
  |
  v
模型基于原文回答
```

一句话总结：

```text
memory_search 让模型找到“可能相关的记忆”；
memory_get 让模型读取“可验证的记忆原文”。
```

这个设计的最大价值是：

```text
省上下文、可追溯、可扩展。
```

最大代价是：

```text
搜索不保证完美，需要索引、工具调用和模型配合。
```
