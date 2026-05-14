# 18. OpenClaw 的学习机制：记忆、会话和技能

这篇回答一个核心问题：

```text
让 OpenClaw 做一件事情之后，它会不会自动把这件事“学习”为一个技能？
```

结论先说清楚：

```text
普通任务不会默认变成技能。

OpenClaw 的“学习”不是改模型参数，而是把信息写入文件或状态：
1. 会话 transcript：记录这次聊天和工具调用。
2. workspace 启动文件：AGENTS.md、MEMORY.md 等会被注入上下文。
3. memory：把事实、偏好、历史上下文写入 MEMORY.md / memory/*.md，并提供搜索工具。
4. skills：把可复用操作流程写成 SKILL.md。
5. Skill Workshop：一个默认关闭的实验插件，可以把可复用流程提议或写成 workspace skill。
```

所以，“会记住”和“会变成技能”是两件不同的事。

## 一、OpenClaw 里“学习”到底是什么意思

OpenClaw 不会像训练模型一样把新知识写进模型权重。

它的学习更接近下面这个过程：

```text
用户/Agent 产生信息
        |
        v
写入某种持久化位置
        |
        v
下次运行时通过 prompt、工具或索引重新取出来
        |
        v
模型在当前上下文里使用这些信息
```

这意味着：

- 如果只存在聊天上下文里，它只是会话历史。
- 如果写入 `MEMORY.md`，它是长期记忆。
- 如果写入 `memory/YYYY-MM-DD.md`，它是日常短期记录。
- 如果写入 `skills/<name>/SKILL.md`，它才是技能。
- 如果写入 cron、tasks、commitments，它是自动化/跟进状态，不是技能。

## 二、会话 transcript：记录发生过什么，但不是技能

会话负责记录一次对话的历史。

文档说明会话状态位置：

- `docs/concepts/session.md:95`：`sessions.json`
- `docs/concepts/session.md:96`：`<sessionId>.jsonl`

代码里路径由 `src/config/sessions/paths.ts` 负责：

- `src/config/sessions/paths.ts:9`：`resolveAgentSessionsDir(...)`
- `src/config/sessions/paths.ts:17`：默认目录是 `agents/<agentId>/sessions`
- `src/config/sessions/paths.ts:30`：`resolveDefaultSessionStorePath(...)` 返回 `sessions.json`

可以把它理解成：

```text
sessions.json     = 这一堆 session 的索引和元数据
<sessionId>.jsonl = 某个 session 的具体聊天 transcript
```

这层的作用是“延续对话”和“可追溯历史”，不是把经验固化为技能。

## 三、workspace 启动文件：每次启动时注入的长期上下文

Agent 的 workspace 是一个真实文件夹。里面的启动文件会被 OpenClaw 读入 prompt。

相关文档：

- `docs/concepts/agent.md:25` 到 `docs/concepts/agent.md:39`
- `docs/reference/token-use.md:22`

关键文件包括：

```text
AGENTS.md
SOUL.md
TOOLS.md
IDENTITY.md
USER.md
HEARTBEAT.md
BOOTSTRAP.md
MEMORY.md
```

代码里定义这些文件的是 `src/agents/workspace.ts`：

- `src/agents/workspace.ts:18` 到 `src/agents/workspace.ts:28`：定义默认文件名。
- `src/agents/workspace.ts:168` 到 `src/agents/workspace.ts:176`：`VALID_BOOTSTRAP_NAMES` 只允许这些受支持的启动文件名。
- `src/agents/workspace.ts:615`：`loadWorkspaceBootstrapFiles(...)` 读取 workspace 启动文件。
- `src/agents/bootstrap-files.ts:246` 到 `src/agents/bootstrap-files.ts:254`：在一次 run 前解析并过滤要注入的启动文件。

新手可以这样理解：

```text
AGENTS.md / SOUL.md / TOOLS.md / USER.md
    = Agent 的长期规则、身份、用户偏好、工具使用说明

MEMORY.md
    = 长期记忆

BOOTSTRAP.md
    = 第一次初始化时用，完成后移除
```

这些文件会影响 Agent 以后怎么回答，但它们也不是 skills。

## 四、Memory：记住事实、偏好、历史上下文

OpenClaw 的 memory 主要是“文件 + 搜索工具”。

文档 `docs/concepts/memory.md` 说得很明确：

- `docs/concepts/memory.md:10`：记忆是 workspace 里的普通 Markdown 文件。
- `docs/concepts/memory.md:15`：`MEMORY.md` 是长期记忆。
- `docs/concepts/memory.md:19`：`memory/YYYY-MM-DD.md` 是日常记录。
- `docs/concepts/memory.md:35` 到 `docs/concepts/memory.md:42`：`memory_search` 和 `memory_get` 是记忆工具。

memory-core 插件注册这两个工具：

- `extensions/memory-core/index.ts:190`：注册 `memory_search`
- `extensions/memory-core/index.ts:194`：注册 `memory_get`

工具说明也写在代码里：

- `extensions/memory-core/src/tools.ts:238`：`memory_search` 搜索 `MEMORY.md + memory/*.md + indexed session transcripts`
- `extensions/memory-core/src/tools.ts:402`：`memory_get` 精确读取记忆文件片段

系统提示里也会提醒模型该如何使用 memory：

- `extensions/memory-core/src/prompt-section.ts:7` 到 `extensions/memory-core/src/prompt-section.ts:24`

简单说：

```text
memory_search = 先找相关记忆
memory_get    = 再读取具体文件和行
```

### Dreaming 和 promote

Memory 还有一个“整理/晋升”机制。它不是把任务变技能，而是把短期记忆候选晋升到 `MEMORY.md`。

相关文档：

- `docs/concepts/memory.md:146` 到 `docs/concepts/memory.md:160`

相关代码：

- `extensions/memory-core/src/short-term-promotion.ts:1570`：目标文件是 `MEMORY.md`
- `extensions/memory-core/src/short-term-promotion.ts:1580` 到 `extensions/memory-core/src/short-term-promotion.ts:1611`：按分数、召回次数、查询多样性等条件筛选候选
- `extensions/memory-core/src/short-term-promotion.ts:1632` 到 `extensions/memory-core/src/short-term-promotion.ts:1637`：把符合条件的内容追加到 `MEMORY.md`

所以 Dreaming / promote 是：

```text
短期记忆 -> 长期记忆 MEMORY.md
```

不是：

```text
短期任务 -> 技能 SKILL.md
```

## 五、Skills：真正的“技能”是什么

Skill 是一个目录，里面必须有 `SKILL.md`。

文档：

- `docs/tools/skills.md:13`：每个 skill 是包含 `SKILL.md` 的目录。
- `docs/tools/creating-skills.md:10`：skill 用来教 Agent 何时、如何使用工具。

最小结构：

```text
skills/
  my-workflow/
    SKILL.md
```

`SKILL.md` 至少需要：

```markdown
---
name: my-workflow
description: Explain when this workflow should be used.
---

具体操作步骤写在这里。
```

### 技能从哪里加载

文档 `docs/tools/skills.md:18` 到 `docs/tools/skills.md:29` 列出了加载位置。

代码实现集中在 `src/agents/skills/workspace.ts`：

- `src/agents/skills/workspace.ts:712`：managed skills 目录默认是 OpenClaw 配置目录下的 `skills`
- `src/agents/skills/workspace.ts:713`：workspace skills 是 `<workspace>/skills`
- `src/agents/skills/workspace.ts:714`：bundled skills 是安装包内置技能
- `src/agents/skills/workspace.ts:718` 到 `src/agents/skills/workspace.ts:722`：还会合并 `skills.load.extraDirs` 和 plugin skills
- `src/agents/skills/workspace.ts:769`：注释写明优先级：`extra < bundled < managed < personal < project < workspace`

也就是说，同名 skill 冲突时，workspace 里的版本优先级最高。

### 怎么识别一个目录是 skill

`src/agents/skills/local-loader.ts` 负责从本地目录读取 skill：

- `src/agents/skills/local-loader.ts:41`：固定读取 `SKILL.md`
- `src/agents/skills/local-loader.ts:53`：解析 frontmatter
- `src/agents/skills/local-loader.ts:60` 到 `src/agents/skills/local-loader.ts:61`：没有 `name` 或 `description` 就不加载
- `src/agents/skills/local-loader.ts:102`：`loadSkillsFromDirSafe(...)` 读取一个 skill root

所以，随便写一个 Markdown 文件不会变技能；必须符合 `SKILL.md` 的目录结构和 frontmatter。

### 技能怎样进入 Agent prompt

技能不会把完整 `SKILL.md` 全部塞进 prompt。

OpenClaw 先构建一个技能快照：

- `src/agents/skills/workspace.ts:908`：`buildWorkspaceSkillSnapshot(...)`
- `src/agents/skills/workspace.ts:957`：`resolveWorkspaceSkillPromptState(...)`
- `src/agents/skills/workspace.ts:1003`：`resolveSkillsPromptForRun(...)`

Agent 命令执行前会决定是否刷新技能快照：

- `src/agents/agent-command.ts:632`：读取当前 skills snapshot version
- `src/agents/agent-command.ts:637` 到 `src/agents/agent-command.ts:640`：判断当前 session 是否需要刷新 snapshot
- `src/agents/agent-command.ts:650`：调用 `buildWorkspaceSkillSnapshot(...)`
- `src/agents/agent-command.ts:673` 到 `src/agents/agent-command.ts:685`：把新的 `skillsSnapshot` 写回 session entry

真正运行 Agent 时再解析这个 prompt：

- `src/agents/pi-embedded-runner/run/attempt.ts:748`：决定本次 run 是否使用 `skillsSnapshot`
- `src/agents/pi-embedded-runner/run/attempt.ts:756` 到 `src/agents/pi-embedded-runner/run/attempt.ts:763`：应用 skill 的环境变量覆盖
- `src/agents/pi-embedded-runner/run/attempt.ts:766` 到 `src/agents/pi-embedded-runner/run/attempt.ts:773`：生成本次 run 的 `skillsPrompt`

系统提示里明确要求模型：

- `src/agents/system-prompt.ts:197`：构造 Skills section
- `src/agents/system-prompt.ts:205`：如果一个 skill 明确适用，先用 read 工具读取 `<location>` 指向的 `SKILL.md`
- `src/agents/system-prompt.ts:207`：如果没有明确适用的 skill，就不要读任何 `SKILL.md`

这点很重要：

```text
prompt 里通常只放“有哪些技能”和“SKILL.md 在哪里”。
模型需要时再 read 对应 SKILL.md。
```

## 六、Skill Workshop：把可复用流程变成技能的桥

如果你问的是“OpenClaw 会不会把自己做过的事总结成技能”，对应机制是 Skill Workshop。

但它有两个关键限制：

```text
1. 它是实验插件。
2. 默认关闭，必须显式启用。
```

文档说明：

- `docs/plugins/skill-workshop.md:11`：实验功能，默认关闭。
- `docs/plugins/skill-workshop.md:16` 到 `docs/plugins/skill-workshop.md:24`：它把可复用流程写成 workspace skills。
- `docs/plugins/skill-workshop.md:49`：必须在 `plugins.entries.skill-workshop` 中显式启用。

启用示例：

```json5
{
  plugins: {
    entries: {
      "skill-workshop": {
        enabled: true,
        config: {
          autoCapture: true,
          approvalPolicy: "pending",
          reviewMode: "hybrid"
        }
      }
    }
  }
}
```

推荐新手先用：

```text
approvalPolicy: "pending"
```

意思是：先提出技能更新建议，不直接写文件。

### Skill Workshop 注册了什么

插件入口是 `extensions/skill-workshop/index.ts`。

关键代码：

- `extensions/skill-workshop/index.ts:28`：注册 `skill_workshop` 工具
- `extensions/skill-workshop/index.ts:41`：在 `before_prompt_build` 注入提示，让 Agent 知道可以使用 Skill Workshop
- `extensions/skill-workshop/index.ts:51`：监听 `agent_end`
- `extensions/skill-workshop/index.ts:53`：只有启用、允许 autoCapture、且 reviewMode 不是 `off` 时才会自动捕获
- `extensions/skill-workshop/index.ts:66`：通过启发式从消息中生成 proposal
- `extensions/skill-workshop/index.ts:108`：也可以用 LLM reviewer 复盘 transcript 生成 proposal
- `extensions/skill-workshop/index.ts:75` 和 `extensions/skill-workshop/index.ts:127`：把 proposal 应用或存储

配置默认值在 `extensions/skill-workshop/src/config.ts`：

- `extensions/skill-workshop/src/config.ts:40`：插件 entry 加载后，内部 `enabled` 默认 true
- `extensions/skill-workshop/src/config.ts:41`：`autoCapture` 默认 true
- `extensions/skill-workshop/src/config.ts:42`：`approvalPolicy` 默认 pending
- `extensions/skill-workshop/src/config.ts:43`：`reviewMode` 默认 hybrid

注意：这不等于插件默认启用。文档已经说明，整个插件 entry 必须先被加载。

### `skill_workshop` 工具怎么写技能

工具实现是 `extensions/skill-workshop/src/tool.ts`。

关键行为：

- `extensions/skill-workshop/src/tool.ts:156`：先准备 proposal 写入内容
- `extensions/skill-workshop/src/tool.ts:174`：auto/apply 时调用 `applyProposalToWorkspace(...)`
- `extensions/skill-workshop/src/tool.ts:189`：pending 时也会先扫描准备内容
- `extensions/skill-workshop/src/tool.ts:224`：手动 apply pending proposal 时写入 workspace

实际写 `SKILL.md` 的代码在 `extensions/skill-workshop/src/skills.ts`：

- `extensions/skill-workshop/src/skills.ts:42`：目标目录限制在 `<workspace>/skills`
- `extensions/skill-workshop/src/skills.ts:56`：目标文件固定是 `SKILL.md`
- `extensions/skill-workshop/src/skills.ts:99`：`prepareProposalWrite(...)` 生成要写入的内容并扫描
- `extensions/skill-workshop/src/skills.ts:141`：`applyProposalToWorkspace(...)`
- `extensions/skill-workshop/src/skills.ts:148`：写入后 bump skills snapshot version，让后续 turn 能刷新技能列表

因此完整链路是：

```text
Agent 做完一次任务
        |
        v
Skill Workshop 在 agent_end 复盘
        |
        v
发现可复用流程
        |
        v
生成 SkillProposal
        |
        v
approvalPolicy = pending  -> 存入 pending，等待用户/工具 apply
approvalPolicy = auto     -> 安全扫描通过后写入
        |
        v
写到 <workspace>/skills/<skill-name>/SKILL.md
        |
        v
bump skills snapshot version
        |
        v
下一次 Agent turn 看到新 skill
```

## 七、哪些东西不是技能

下面这些都可能让 Agent “记得更多”，但它们不是技能：

| 机制 | 存什么 | 是不是技能 |
| --- | --- | --- |
| session transcript | 聊天和工具调用历史 | 不是 |
| `MEMORY.md` | 长期事实、偏好、决策 | 不是 |
| `memory/YYYY-MM-DD.md` | 每日短期记录 | 不是 |
| Dreaming / promote | 把短期记忆晋升到 `MEMORY.md` | 不是 |
| commitments | 推断出的短期跟进事项 | 不是 |
| cron jobs | 定时任务 | 不是 |
| tasks | 后台运行记录 | 不是 |
| `skills/<name>/SKILL.md` | 可复用操作流程 | 是 |
| Skill Workshop | 生成/更新 `SKILL.md` 的机制 | 它本身不是技能，但能创建技能 |

## 八、新手判断：这次任务会被学成技能吗

可以按这个顺序判断：

```text
1. 有没有启用 plugins.entries.skill-workshop？
   没有 -> 不会自动变技能。

2. Skill Workshop 的 approvalPolicy 是 pending 还是 auto？
   pending -> 只生成待审批 proposal，不直接写 SKILL.md。
   auto    -> 安全扫描通过后可以直接写 SKILL.md。

3. 这次任务是不是“可复用流程”？
   是事实/偏好 -> 应该进 MEMORY.md。
   是一次性聊天 -> 只在 transcript。
   是未来提醒 -> 应该进 cron 或 commitments。
   是稳定操作步骤 -> 才适合变 skill。

4. workspace 下有没有出现 skills/<name>/SKILL.md？
   有 -> 才说明技能真正落盘。
```

## 补充：skills snapshot 什么时候刷新，里面到底存了什么

你问的“判断是否需要刷新快照”，核心不在 `SKILL.md` 的 mtime 对比，而在一个内存里的版本号和 session 里缓存的快照版本对比。

### 1. 快照刷新判断的主链路

普通 Agent run 里，判断逻辑在 `src/agents/agent-command.ts`：

- `src/agents/agent-command.ts:630`：加载 `getSkillsSnapshotVersion(...)` 和 `shouldRefreshSnapshotForVersion(...)`
- `src/agents/agent-command.ts:632`：读取当前 workspace 的 `skillsSnapshotVersion`
- `src/agents/agent-command.ts:634`：读取当前 session 里已有的 `currentSkillsSnapshot`
- `src/agents/agent-command.ts:635` 到 `src/agents/agent-command.ts:639`：计算是否要刷新
- `src/agents/agent-command.ts:650`：需要刷新时调用 `buildWorkspaceSkillSnapshot(...)`
- `src/agents/agent-command.ts:673` 到 `src/agents/agent-command.ts:687`：把新的 `skillsSnapshot` 写回 session entry

代码逻辑可以翻译成：

```ts
const skillsSnapshotVersion = getSkillsSnapshotVersion(workspaceDir);
const skillFilter = resolveAgentSkillsFilter(cfg, sessionAgentId);
const currentSkillsSnapshot = sessionEntry?.skillsSnapshot;

const shouldRefreshSkillsSnapshot =
  !currentSkillsSnapshot ||
  shouldRefreshSnapshotForVersion(currentSkillsSnapshot.version, skillsSnapshotVersion) ||
  !matchesSkillFilter(currentSkillsSnapshot.skillFilter, skillFilter);

const needsSkillsSnapshot = isNewSession || shouldRefreshSkillsSnapshot;
```

也就是说，只要满足下面任意条件，就需要新的技能快照：

```text
1. 当前 session 没有旧 skillsSnapshot
2. 当前 workspace 的 skills snapshot version 比 session 里缓存的 version 更新
3. 当前 agent 的 skills allowlist / skillFilter 跟旧快照不一致
4. 当前是新 session
```

### 2. 版本号是怎么来的

版本状态在 `src/agents/skills/refresh-state.ts`：

- `src/agents/skills/refresh-state.ts:12`：`bumpVersion(...)`
- `src/agents/skills/refresh-state.ts:38`：`bumpSkillsSnapshotVersion(...)`
- `src/agents/skills/refresh-state.ts:57`：`getSkillsSnapshotVersion(...)`
- `src/agents/skills/refresh-state.ts:65`：`shouldRefreshSnapshotForVersion(...)`

它维护两种版本：

```text
workspaceVersions = 每个 workspace 自己的版本
globalVersion     = 全局版本
```

读取时：

```ts
return Math.max(globalVersion, local);
```

所以只要全局技能环境变了，或这个 workspace 的技能变了，都会让当前 workspace 的 effective version 变大。

真正判断是否刷新的是：

```ts
const cached = typeof cachedVersion === "number" ? cachedVersion : 0;
const next = typeof nextVersion === "number" ? nextVersion : 0;
return next === 0 ? cached > 0 : cached < next;
```

新手可以这样理解：

```text
如果当前版本 next > 0：
    旧快照版本 cached < 当前版本 next，就刷新。

如果当前版本 next == 0：
    说明进程里还没有任何技能变更信号。
    只有旧快照 cached > 0 时才刷新，避免用旧进程留下的高版本缓存误判为仍然有效。
```

### 3. 什么事件会 bump 版本号

`bumpSkillsSnapshotVersion(...)` 会在这些地方触发：

1. 技能文件变化
   - `src/agents/skills/refresh.ts:105`：`ensureSkillsWatcher(...)`
   - `src/agents/skills/refresh.ts:110`：`skills.load.watch !== false` 时启用 watcher
   - `src/agents/skills/refresh.ts:148`：watcher 忽略非 `SKILL.md` 的普通文件
   - `src/agents/skills/refresh.ts:162`：文件事件 debounce 后 bump workspace version
   - `src/agents/skills/refresh.ts:170` 到 `src/agents/skills/refresh.ts:173`：监听 add/change/unlink/unlinkDir

2. Skill Workshop 写入技能
   - `extensions/skill-workshop/src/skills.ts:141`：`applyProposalToWorkspace(...)`
   - `extensions/skill-workshop/src/skills.ts:147`：写入 `SKILL.md`
   - `extensions/skill-workshop/src/skills.ts:148`：写完后 bump snapshot version

3. 配置变化
   - `src/gateway/config-reload.ts:237`：`skills.*` 相关配置变化时 bump，原因是 allowlist 或加载配置变了，旧 session 继续用旧技能列表会出错

4. 远程节点能力变化
   - `src/infra/skills-remote.ts:184`：发现可用远程节点能力时 bump
   - `src/infra/skills-remote.ts:383`：远程节点 bin 探测结果变化时 bump

文档也说明了默认行为：

- `docs/tools/skills.md:403`：mid-session 刷新主要来自 watcher 和新的 eligible remote node
- `docs/tools/skills.md:415`：默认 watch skill folders
- `docs/tools/skills-config.md:90`：`skills.load.watch` 默认 true
- `docs/tools/skills-config.md:91`：`skills.load.watchDebounceMs` 默认 250ms

### 4. skills snapshot 是不是把 Markdown 全部加载到内存

结论要分两层说：

```text
不是简单地把所有 SKILL.md Markdown 全文塞进 prompt。
但构建快照时会读取 SKILL.md，解析 frontmatter，并在本轮运行里保留可解析到具体 SKILL.md 文件的 Skill 对象。
```

本地加载技能的代码在 `src/agents/skills/local-loader.ts`：

- `src/agents/skills/local-loader.ts:42`：读取 `SKILL.md` 原始文本
- `src/agents/skills/local-loader.ts:53`：解析 frontmatter
- `src/agents/skills/local-loader.ts:60` 到 `src/agents/skills/local-loader.ts:61`：没有 `name` 或 `description` 就不加载
- `src/agents/skills/local-loader.ts:69`：构造 `skill` 对象

但这个 `skill` 对象主要包含：

```text
name
description
filePath
baseDir
source
sourceInfo
disableModelInvocation
```

也就是说，prompt 里默认不是完整 `SKILL.md` 正文，而是技能目录清单。

prompt 格式在：

- `src/agents/skills/skill-contract.ts:44`：`formatSkillsForPrompt(...)`

它输出的是：

```xml
<available_skills>
  <skill>
    <name>...</name>
    <description>...</description>
    <location>.../SKILL.md</location>
  </skill>
</available_skills>
```

并且明确告诉模型：

```text
Use the read tool to load a skill's file when the task matches its description.
```

所以正确理解是：

```text
skills snapshot = 当前可用技能的“目录快照” + 运行期可用的 resolvedSkills。

它让模型知道：
1. 有哪些技能
2. 每个技能大概适合什么任务
3. 具体 SKILL.md 在哪里

真正的 SKILL.md 正文，需要模型判断适用后再用 read 工具读取。
```

### 5. 快照对象里有哪些字段

类型定义在 `src/agents/skills/types.ts`：

```ts
export type SkillSnapshot = {
  prompt: string;
  skills: Array<{ name: string; primaryEnv?: string; requiredEnv?: string[] }>;
  skillFilter?: string[];
  resolvedSkills?: Skill[];
  version?: number;
};
```

构建位置在 `src/agents/skills/workspace.ts:908`：

```ts
return {
  prompt,
  skills: eligible.map(...),
  skillFilter,
  resolvedSkills,
  version: opts?.snapshotVersion,
};
```

字段可以这样理解：

```text
prompt         = 注入系统提示的技能清单文本
skills         = 轻量技能元信息，用于 env 等运行期逻辑
skillFilter    = 当前 agent 的技能 allowlist 快照
resolvedSkills = 本轮运行可用的 Skill 对象，包含 SKILL.md 路径等
version        = 这份快照对应的刷新版本
```

### 6. 为什么不是把完整正文长期存在 session 里

`resolvedSkills` 是运行期缓存，不应该长期写进 `sessions.json`。

相关代码：

- `src/agents/skills/snapshot-hydration.ts:9`：注释说明 `resolvedSkills` 是 runtime-only
- `src/agents/skills/snapshot-hydration.ts:26`：如果 session 里的 snapshot 没有 `resolvedSkills`，就从 workspace 重新 hydrate
- `src/config/sessions/store-load.ts:68`：注释说明 `resolvedSkills` 可能很大，持久化会让 `sessions.json` 膨胀
- `src/config/sessions/store-load.ts:74`：`stripPersistedSkillsCache(...)`
- `src/config/sessions/store-load.ts:79`：保存 session 时去掉 `resolvedSkills`

这说明 OpenClaw 刻意区分：

```text
持久化到 session 的：
    prompt
    skills
    skillFilter
    version

只在本轮内存里使用的：
    resolvedSkills
```

### 7. “session 版本号”到底是什么

这里容易误解：代码里并没有一个叫“session version”的顶层字段。

`SessionEntry` 顶层主要是：

- `src/config/sessions/types.ts:150`：`SessionEntry`
- `src/config/sessions/types.ts:173`：`updatedAt`
- `src/config/sessions/types.ts:196`：`sessionStartedAt`
- `src/config/sessions/types.ts:328`：`skillsSnapshot?: SessionSkillSnapshot`

真正跟技能快照刷新有关的版本号，是：

```text
sessionEntry.skillsSnapshot.version
```

它的类型在：

- `src/config/sessions/types.ts:544`：`SessionSkillSnapshot`
- `src/config/sessions/types.ts:556`：`version?: number`

所以更准确的说法是：

```text
不是 session 自己有版本号；
是 session 里缓存的 skillsSnapshot 有一个 version。
```

### 8. 新开一个会话会不会更新这个 version

要分清两个动作：

```text
新开会话会重新构建/写入这个 session 的 skillsSnapshot。
但新开会话本身不会 bump 全局 skills snapshot version。
```

普通 Agent run 的逻辑在 `src/agents/agent-command.ts`：

- `src/agents/agent-command.ts:632`：先读取当前全局/工作区的 `skillsSnapshotVersion`
- `src/agents/agent-command.ts:639`：`isNewSession || shouldRefreshSkillsSnapshot`
- `src/agents/agent-command.ts:650`：需要时构建快照
- `src/agents/agent-command.ts:662`：把当前 `skillsSnapshotVersion` 写入快照的 `version`
- `src/agents/agent-command.ts:673` 到 `src/agents/agent-command.ts:687`：把快照保存进当前 session

所以新会话的情况是：

```text
isNewSession = true
        |
        v
构建新的 skillsSnapshot
        |
        v
skillsSnapshot.version = 当前 getSkillsSnapshotVersion(workspaceDir) 的结果
        |
        v
保存到这个新 session
```

如果此时没有任何技能变更信号，`getSkillsSnapshotVersion(workspaceDir)` 通常还是 `0`。
那么新 session 里的 `skillsSnapshot.version` 也可能是 `0`。

也就是说：

```text
新开会话会更新“这个 session 缓存的快照内容”；
但不一定会让 version 数字变大。
```

version 数字变大只发生在 `bumpSkillsSnapshotVersion(...)` 被调用时，例如：

```text
SKILL.md 变化
Skill Workshop 写入技能
skills.* 配置变化
远程节点能力变化
```

### 9. 用例子理解

假设当前进程里还没有任何技能变更：

```text
globalVersion = 0
workspaceVersion = 0
getSkillsSnapshotVersion(workspaceDir) = 0
```

你新开 session A：

```text
session A 创建 skillsSnapshot
session A.skillsSnapshot.version = 0
```

你再新开 session B：

```text
session B 也创建 skillsSnapshot
session B.skillsSnapshot.version = 0
```

这两个 session 都有自己的快照，但 version 数字不一定变化。

后来你修改了某个 `SKILL.md`：

```text
watcher -> bumpSkillsSnapshotVersion(workspaceDir)
workspaceVersion = 1770000000000  // 类似 Date.now()
```

下一次 session A 或 B 再运行时：

```text
旧快照 version = 0
当前 version = 1770000000000
0 < 1770000000000
        |
        v
刷新该 session 的 skillsSnapshot
```

所以它的作用不是记录“第几个会话”，而是记录：

```text
这个 session 缓存的技能快照，是基于哪一版技能环境生成的。
```

### 10. 最准确的新手结论

```text
判断是否刷新技能快照：
    看 session 有没有快照；
    看内存版本号是否比 session 版本新；
    看当前 agent 的技能 allowlist 是否变了；
    新 session 也会构建快照。

技能快照不是“把全部 Markdown 正文塞进 prompt”。
它主要是把可用技能的 name / description / SKILL.md path 做成目录清单。
模型看到目录后，只有当任务匹配某个技能时，才用 read 工具读取对应 SKILL.md 正文。
```

## 九、最重要的心智模型

把 OpenClaw 的学习机制理解成四个盒子：

```text
会话盒子：sessions/*.jsonl
    记录刚刚发生了什么。

记忆盒子：MEMORY.md + memory/*.md
    记录事实、偏好、历史上下文。

规则盒子：AGENTS.md / SOUL.md / TOOLS.md / USER.md
    规定 Agent 长期怎么工作。

技能盒子：skills/*/SKILL.md
    记录可复用的操作流程。
```

普通任务默认只会进入“会话盒子”。

如果 Agent 被要求记住事实，它应该写入“记忆盒子”。

如果你希望它以后按某套流程做事，最稳定的方式是写入“技能盒子”。

如果你希望系统自动从工作中提炼技能，就需要启用 Skill Workshop，并从 `approvalPolicy: "pending"` 开始。
