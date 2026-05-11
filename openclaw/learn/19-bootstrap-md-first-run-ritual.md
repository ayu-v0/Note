# 19. BOOTSTRAP.md 是什么：第一次初始化脚本

你提到的 `BOOSTRAST.md` 应该是 `BOOTSTRAP.md`。

它不是项目源码里的业务文件，也不是普通长期记忆文件。它是 OpenClaw 给一个全新 Agent workspace 准备的“一次性初始化脚本”。

## 先给结论

```text
BOOTSTRAP.md = 新 workspace 第一次运行时的初始化仪式说明。

它来自 OpenClaw 内置模板：
docs/reference/templates/BOOTSTRAP.md

它会被复制到 Agent workspace：
<workspace>/BOOTSTRAP.md

它的任务是引导 Agent 和用户完成身份、称呼、偏好、人格等基础设定。

完成后要删除，因为它只适合第一次启动。
```

## 它从哪里来的

模板文件在仓库里：

- `docs/reference/templates/BOOTSTRAP.md`

模板内容的结尾明确写着：

- `docs/reference/templates/BOOTSTRAP.md:58`：完成后删除这个文件。

OpenClaw 不是凭空生成 `BOOTSTRAP.md`，而是读取这个模板，再写入 workspace。

读取模板的代码在 `src/agents/workspace.ts`：

- `src/agents/workspace.ts:27`：`DEFAULT_BOOTSTRAP_FILENAME = "BOOTSTRAP.md"`
- `src/agents/workspace.ts:584`：`loadTemplate(DEFAULT_BOOTSTRAP_FILENAME)`
- `src/agents/workspace.ts:585`：`writeFileIfMissing(bootstrapPath, bootstrapTemplate)`

所以链路是：

```text
docs/reference/templates/BOOTSTRAP.md
        |
        v
loadTemplate("BOOTSTRAP.md")
        |
        v
writeFileIfMissing(<workspace>/BOOTSTRAP.md)
```

## 什么时候生成

Agent 每次运行前会先确保 workspace 存在。

入口在 `src/agents/agent-command.ts`：

- `src/agents/agent-command.ts:390`：调用 `ensureAgentWorkspace(...)`
- `src/agents/agent-command.ts:392`：如果没有 `skipBootstrap`，就启用 bootstrap 文件创建

核心逻辑在 `src/agents/workspace.ts` 的 `ensureAgentWorkspace(...)`：

- `src/agents/workspace.ts:462`：函数入口
- `src/agents/workspace.ts:486`：如果没有要求生成 bootstrap files，就只返回 workspace
- `src/agents/workspace.ts:496`：计算 `<workspace>/BOOTSTRAP.md` 路径
- `src/agents/workspace.ts:572`：只有在没有 `bootstrapSeededAt`、没有 `setupCompletedAt`、且当前没有 `BOOTSTRAP.md` 时，才考虑创建
- `src/agents/workspace.ts:577` 到 `src/agents/workspace.ts:582`：如果 workspace 看起来已经配置过，就标记完成，不再生成
- `src/agents/workspace.ts:584` 到 `src/agents/workspace.ts:592`：否则读取模板并写入 `BOOTSTRAP.md`

这说明它主要出现在：

```text
全新的 workspace
或者部分初始化失败、状态缺失、但系统判断还需要 bootstrap 的 workspace
```

它不会在每次启动时都重新创建。

## 怎么判断是不是“全新 workspace”

代码会检查 workspace 里是否已经有典型文件或用户内容。

在 `src/agents/workspace.ts:499` 到 `src/agents/workspace.ts:514` 附近，`isBrandNewWorkspace` 会检查这些东西：

```text
AGENTS.md
SOUL.md
TOOLS.md
IDENTITY.md
USER.md
HEARTBEAT.md
memory/
.git
MEMORY.md
```

如果这些都没有，才更像一个全新 workspace。

还有一个“看起来已经配置过”的判断：

- `src/agents/workspace.ts:236`：`workspaceProfileLooksConfigured(...)`
- `src/agents/workspace.ts:241` 到 `src/agents/workspace.ts:249`：如果 `SOUL.md`、`IDENTITY.md`、`USER.md` 跟模板不同，或者已有 `memory/` / `MEMORY.md` 等用户内容，就认为 workspace 已经有配置痕迹。

所以如果你已经有自己的 workspace 文件，OpenClaw 会尽量避免又塞一个首次启动脚本进去。

## 它生成后怎么被 Agent 使用

`BOOTSTRAP.md` 会被当成 workspace bootstrap file 加载。

读取 workspace 启动文件的代码：

- `src/agents/workspace.ts:615`：`loadWorkspaceBootstrapFiles(...)`
- `src/agents/workspace.ts:647` 到 `src/agents/workspace.ts:648`：把 `BOOTSTRAP.md` 加入要读取的文件列表

系统提示会告诉 Agent：当前 bootstrap 还没完成，必须先遵循 `BOOTSTRAP.md`。

相关代码：

- `src/agents/system-prompt.ts:231`：`buildAgentBootstrapSystemContext(...)`
- `src/agents/system-prompt.ts:253`：如果 `BOOTSTRAP.md` 已经注入 Project Context，就让 Agent 直接 follow it
- `src/agents/system-prompt.ts:256`：第一次用户可见回复必须遵循 `BOOTSTRAP.md`，不能只是普通问候

你可以把它理解成：

```text
普通启动文件：提供长期上下文
BOOTSTRAP.md：第一次启动时强制 Agent 先完成初始化流程
```

## 它为什么第一次初始化后要移除

因为它的内容只适合“第一次认识用户和确定身份”。

模板里让 Agent 做这些事：

- 问自己叫什么。
- 问用户是谁、怎么称呼用户。
- 确认 Agent 的风格、边界、偏好。
- 写入 `IDENTITY.md`、`USER.md`、`SOUL.md`。
- 完成后删除 `BOOTSTRAP.md`。

如果不删除，每次新会话都可能重新触发“我刚醒来，我是谁？”这种初始化流程。这会干扰正常使用。

所以设计上是：

```text
BOOTSTRAP.md
    只负责把空白 workspace 变成有身份、有用户资料、有基本行为规则的 workspace。

IDENTITY.md / USER.md / SOUL.md
    负责长期保存初始化结果。

初始化完成后，BOOTSTRAP.md 的使命结束。
```

## 删除后 OpenClaw 怎么知道已经完成

它不是只靠“文件不存在”判断。

OpenClaw 还会写一个 workspace 内部状态文件：

```text
<workspace>/.openclaw/workspace-state.json
```

状态字段在 `src/agents/workspace.ts`：

- `src/agents/workspace.ts:163`：`bootstrapSeededAt`
- `src/agents/workspace.ts:164`：`setupCompletedAt`

含义：

```text
bootstrapSeededAt = 曾经生成过 BOOTSTRAP.md
setupCompletedAt = 初始化已经完成
```

完成状态的修复/判断逻辑：

- `src/agents/workspace.ts:278` 到 `src/agents/workspace.ts:284`：如果曾经 seed 过 bootstrap，但现在 `BOOTSTRAP.md` 不存在，就写入 `setupCompletedAt`
- `src/agents/workspace.ts:288` 到 `src/agents/workspace.ts:304`：如果 `BOOTSTRAP.md` 还在，但 workspace 已有身份/用户/记忆等配置证据，就删除它并标记完成
- `src/agents/workspace.ts:376` 到 `src/agents/workspace.ts:384`：`resolveWorkspaceBootstrapStatus(...)` 用 `setupCompletedAt` 和 `BOOTSTRAP.md` 是否存在判断 pending/complete

测试也覆盖了这个行为：

- `src/agents/workspace.test.ts:79`：全新 workspace 会创建 `BOOTSTRAP.md`
- `src/agents/workspace.test.ts:97`：完成后不会重新创建 `BOOTSTRAP.md`
- `src/agents/workspace.test.ts:254`：如果 `IDENTITY.md` 显示已经配置过，会清理过期的 `BOOTSTRAP.md`
- `src/agents/workspace.test.ts:312`：删除 `BOOTSTRAP.md` 并再次 ensure 后，状态变成 complete

## 能不能跳过它

可以。

配置里可以设置：

```json5
{
  agents: {
    defaults: {
      skipBootstrap: true
    }
  }
}
```

文档位置：

- `docs/concepts/agent.md:44` 到 `docs/concepts/agent.md:47`
- `docs/concepts/agent-workspace.md:42` 到 `docs/concepts/agent-workspace.md:46`

这适合你已经手动准备好了 `AGENTS.md`、`SOUL.md`、`USER.md`、`IDENTITY.md` 等文件，不希望 OpenClaw 插入默认初始化流程的情况。

## 新手心智模型

把 `BOOTSTRAP.md` 想成：

```text
新 Agent 的“开场问卷 + 自我设定脚本”
```

它不是长期记忆，也不是技能。

它只是帮你把这些长期文件初始化出来：

```text
IDENTITY.md  -> Agent 是谁
USER.md      -> 用户是谁
SOUL.md      -> Agent 应该怎么表现
```

一旦这些长期文件有内容，`BOOTSTRAP.md` 就应该消失，避免每次都重复第一次见面的流程。

