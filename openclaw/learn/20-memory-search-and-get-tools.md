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
memory_get    = 根据搜索结果里的 path / 行号，读取具体记忆文件原文
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

搜索执行在：

- `extensions/memory-core/src/memory/manager.ts:304`：`search(...)`
- `extensions/memory-core/src/memory/manager.ts:437`：关键字搜索和向量搜索路径
- `extensions/memory-core/src/memory/manager.ts:462`：混合搜索合并结果
- `extensions/memory-core/src/memory/manager.ts:471`：按分数过滤
- `extensions/memory-core/src/memory/manager.ts:479`：关键字 fallback

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

## 四、memory_get 的设计

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

执行主线在：

- `extensions/memory-core/src/tools.ts:413`：解析 `corpus`
- `extensions/memory-core/src/tools.ts:418`：加载读取函数和后端配置
- `extensions/memory-core/src/tools.ts:438`：builtin 后端读取 agent memory file
- `extensions/memory-core/src/tools.ts:462`：非 builtin 后端通过 manager readFile
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

## 五、为什么要分成 search 和 get

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

## 六、它的安全边界

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

## 七、设计优点

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
- `extensions/memory-core/src/memory/manager.ts:479`：关键字 fallback

### 优点 6：可以形成后续学习信号

搜索命中会记录 recall 信号：

- `extensions/memory-core/src/tools.ts:106`：recall tracking 是 best-effort
- `extensions/memory-core/src/tools.ts:335`：搜索后 queue recall touch

这给后续 dreaming / promote 这类整理机制提供输入。

## 八、设计缺点

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

## 九、新手理解版

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

