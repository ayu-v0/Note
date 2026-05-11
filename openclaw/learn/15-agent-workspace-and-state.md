# 15. Agent 的 workspace 和状态目录

这篇专门解释一个新手容易混淆的问题：

```text
Agent 的 workspace 是什么？
它是文件夹数据，还是内存数据？
```

## 先给结论

Agent 的 `workspace` 是一个真实文件夹，不是单纯的内存数据。

它表示这个 Agent 工作时所在的目录。Agent 会在这个目录里理解项目、读取文件、写入文件、运行命令、加载本地说明文件。

你可以把它理解成：

```text
workspace = Agent 的工作现场
```

例如当前这个学习任务里，workspace 就是这个项目目录：

```text
L:\Project\openclaw-main
```

Agent 在这里读源码、读 docs、读 `.workplace/learn`，也把学习笔记写回 `.workplace/learn`。

## workspace 里有什么

workspace 通常包含项目本身的数据，例如：

- 源码
- README
- docs
- AGENTS.md
- 配置模板
- 测试文件
- 本地学习笔记
- 你希望 Agent 操作的项目文件

对 OpenClaw 来说，workspace 是 Agent 运行时最重要的上下文来源之一。

如果一个 Agent 的 workspace 指向某个项目目录，那么它主要就在那个项目范围内工作。

## workspace 不是内存

workspace 不是临时内存对象。

它是磁盘上的目录，所以：

- 文件会长期存在。
- Agent 改过的文件会留在目录里。
- 下次再打开同一个 workspace，文件还在。
- 你可以用编辑器、终端、文件管理器直接查看。

内存里当然也会有一次运行的临时上下文，比如当前 prompt、已读文件片段、工具结果、运行状态。但那些不是 `workspace` 本身。

## 配置里怎么写

Agent 的 workspace 来自主配置文件里的 `agents` 配置。

示意：

```json5
{
  agents: {
    defaults: {
      workspace: "L:/Project/openclaw-main",
    },
    list: [
      {
        id: "main",
        default: true,
        workspace: "L:/Project/openclaw-main",
      },
      {
        id: "work",
        workspace: "L:/Project/work-project",
      },
    ],
  },
}
```

含义：

- `agents.defaults.workspace` 是默认工作目录。
- `agents.list[].workspace` 是某个 Agent 自己的工作目录。
- 如果某个 Agent 没有单独写 `workspace`，通常会继承 defaults。
- 不同 Agent 可以指向不同 workspace。

## workspace 和 agentDir 的区别

文档里还会看到 `agentDir` 或 agent state dir。它和 workspace 不是一回事。

简单记：

```text
workspace = Agent 工作的项目文件夹
agentDir  = OpenClaw 给这个 Agent 存运行状态的目录
```

举例：

```text
workspace:
L:\Project\openclaw-main

agentDir:
用户主目录\.openclaw\agents\main
```

workspace 里放项目和用户资料。

agentDir 里放 OpenClaw 管理的 Agent 状态，例如：

- session transcript
- auth profiles
- agent runtime 状态
- Agent 相关缓存或元数据
- 这个 Agent 的内部状态文件

## 为什么要分开

因为这两个目录解决的问题不同。

workspace 解决：

- Agent 在哪里工作？
- Agent 可以读写哪个项目？
- Agent 应该把文件改在哪里？
- Agent 应该从哪些本地说明文件获得项目规则？

agentDir 解决：

- 这个 Agent 的会话历史存在哪里？
- 这个 Agent 的认证状态存在哪里？
- 这个 Agent 的运行状态和内部数据存在哪里？
- 多个 Agent 的状态如何隔离？

所以多 Agent 场景下，可能出现：

```text
main Agent:
  workspace = L:\Project\openclaw-main
  agentDir  = 用户主目录\.openclaw\agents\main

work Agent:
  workspace = L:\Project\company-project
  agentDir  = 用户主目录\.openclaw\agents\work
```

这表示两个 Agent 可以在不同项目里工作，也有各自独立的状态目录。

## 和 agentId 的关系

`agentId` 决定当前消息属于哪个 Agent。

拿到 `agentId` 后，OpenClaw 会继续解析：

- 这个 Agent 的 workspace
- 这个 Agent 的 agentDir
- 这个 Agent 的 model
- 这个 Agent 的 skills
- 这个 Agent 的 tools policy
- 这个 Agent 的 auth profile
- 这个 Agent 的 session store

所以你可以把关系记成：

```text
用户消息
  -> 解析 agentId
  -> 找到这个 Agent 的 workspace 和 agentDir
  -> 在 workspace 里工作
  -> 把状态和会话写到 agentDir
```

## 对新手最重要的一句话

```text
workspace 是 Agent 做事的真实文件夹；
agentDir 是 OpenClaw 保存这个 Agent 运行状态的真实文件夹；
两者都不是单纯的内存数据。
```

## 相关源码

- `src/agents/agent-scope.ts`
- `src/agents/agent-scope-config.ts`
- `src/agents/workspace.ts`
- `src/config/sessions/paths.ts`
- `src/config/paths.ts`

