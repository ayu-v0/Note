# 16. Agent 的工具隔离是怎么做的

这篇解释：

```text
不同 Agent 为什么能看到不同工具？
工具隔离是靠文件夹、内存，还是配置？
```

## 先给结论

Agent 的工具隔离主要靠配置策略过滤，不是靠内存。

OpenClaw 在每次 Agent run 之前，会先根据当前 `agentId`、model provider、session、sandbox、sender 权限等信息，算出“这次模型能看到哪些工具”。只有通过过滤的工具，才会发给模型。

你可以把它理解成：

```text
所有已注册工具
  -> 按全局 tools 配置过滤
  -> 按 provider 配置过滤
  -> 按当前 agentId 的 tools 配置过滤
  -> 按 group/channel/session 配置过滤
  -> 按 sandbox 配置过滤
  -> 按 subagent/owner-only 等特殊规则过滤
  -> 最终发给模型的工具列表
```

所以模型不是“知道所有工具但被提醒不要用”，而是大多数情况下根本看不到被禁用的工具。

## 配置在哪里

工具隔离主要写在 OpenClaw 主配置文件里：

```text
用户主目录/.openclaw/openclaw.json
```

相关字段有两层：

```json5
{
  tools: {
    profile: "coding",
    allow: [],
    alsoAllow: [],
    deny: [],
    byProvider: {},
    sandbox: {
      tools: {
        allow: [],
        alsoAllow: [],
        deny: [],
      },
    },
  },
  agents: {
    list: [
      {
        id: "main",
        tools: {
          profile: "full",
          allow: [],
          alsoAllow: [],
          deny: [],
          byProvider: {},
          sandbox: {
            tools: {
              allow: [],
              alsoAllow: [],
              deny: [],
            },
          },
        },
      },
    ],
  },
}
```

简单说：

- 顶层 `tools` 是全局工具策略。
- `agents.list[].tools` 是某个 Agent 自己的工具策略。
- per-agent 配置会参与当前 Agent 的工具过滤。
- `deny` 优先级很高，被 deny 的工具不能被后面的 allow 重新放回来。

## 一个简单例子

假设你希望：

- `main` Agent 功能完整。
- `family` Agent 只能读文件，不能执行命令、不能写文件、不能开浏览器。

可以这样配：

```json5
{
  agents: {
    list: [
      {
        id: "main",
        default: true,
        tools: {
          profile: "full",
        },
      },
      {
        id: "family",
        tools: {
          allow: ["read"],
          deny: ["exec", "process", "write", "edit", "apply_patch", "browser"],
        },
      },
    ],
  },
}
```

结果：

```text
main   -> 看到 full profile 的工具
family -> 基本只看到 read
```

## profile 是什么

`profile` 是一组预设工具集合。它先决定一个基础工具范围，再叠加 allow/deny。

常见 profile：

| profile | 含义 |
| --- | --- |
| `full` | 尽量完整的工具面 |
| `coding` | 适合写代码：文件、运行、web、session、media 等 |
| `messaging` | 适合频道机器人：消息和会话相关工具 |
| `minimal` | 很窄，主要是 `session_status` |

示例：

```json5
{
  tools: {
    profile: "coding",
  },
  agents: {
    list: [
      {
        id: "support",
        tools: {
          profile: "messaging",
        },
      },
    ],
  },
}
```

含义：

- 默认 Agent 用 `coding` 工具集。
- `support` Agent 改用更窄的 `messaging` 工具集。

## allow、alsoAllow、deny 的区别

### `allow`

显式允许某些工具。

如果配置了 allowlist，工具集合会被收窄到 allow 里匹配到的工具。

```json5
{
  tools: {
    allow: ["read", "web_search"],
  },
}
```

### `alsoAllow`

给 profile 额外加工具，常用于“profile 本来没有这个工具，但我想额外打开”。

例如 `coding` 默认不包含完整浏览器自动化工具，如果要打开：

```json5
{
  tools: {
    profile: "coding",
    alsoAllow: ["browser"],
  },
}
```

### `deny`

显式禁止某些工具。

`deny` 优先级高。即使前面 profile 或 allow 允许了，deny 仍然会把它拿掉。

```json5
{
  tools: {
    profile: "full",
    deny: ["exec", "process"],
  },
}
```

## group:* 工具组

配置里可以不用一个个写工具名，而是写工具组。

常见组：

| 组 | 大致包含 |
| --- | --- |
| `group:runtime` | `exec`、`process`、`code_execution` |
| `group:fs` | `read`、`write`、`edit`、`apply_patch` |
| `group:web` | `web_search`、`x_search`、`web_fetch` |
| `group:ui` | `browser`、`canvas` |
| `group:messaging` | `message` |
| `group:sessions` | sessions 相关工具 |
| `group:media` | image/music/video/tts 工具 |
| `group:openclaw` | OpenClaw 内置工具 |

例如：

```json5
{
  agents: {
    list: [
      {
        id: "reader",
        tools: {
          allow: ["group:fs", "web_fetch"],
          deny: ["write", "edit", "apply_patch"],
        },
      },
    ],
  },
}
```

这个 Agent 可以用文件组，但写入类工具又被 deny 掉，所以更接近只读。

## provider 级别隔离

工具还可以按模型 provider 限制。

例如某个 provider 不适合调用工具，或者你只希望某个 provider 使用最小工具集：

```json5
{
  tools: {
    profile: "coding",
    byProvider: {
      "google-antigravity": {
        profile: "minimal",
      },
    },
  },
}
```

也可以在某个 Agent 里写：

```json5
{
  agents: {
    list: [
      {
        id: "reviewer",
        tools: {
          byProvider: {
            "openai/gpt-5.4": {
              deny: ["exec", "process"],
            },
          },
        },
      },
    ],
  },
}
```

含义：

```text
reviewer Agent 使用 openai/gpt-5.4 时，不能用 exec/process。
```

## sandbox 也会限制工具

工具策略和 sandbox 是两层东西：

- 工具策略决定模型能不能看到/调用某个工具。
- sandbox 决定工具真正执行时能接触哪些系统资源。

例如：

```json5
{
  agents: {
    list: [
      {
        id: "public",
        sandbox: {
          mode: "all",
          scope: "agent",
        },
        tools: {
          allow: ["read", "web_fetch"],
          deny: ["exec", "write", "edit", "apply_patch"],
          sandbox: {
            tools: {
              allow: ["read", "web_fetch"],
            },
          },
        },
      },
    ],
  },
}
```

这里有两件事同时发生：

1. `public` Agent 的工具列表被收窄。
2. 它的运行环境也进入 sandbox。

所以即使某个工具被允许，执行时仍然可能被 sandbox 或工具自己的路径策略拦住。

## owner-only 工具

有些工具属于高权限工具，例如：

- `gateway`
- `cron`
- `nodes`

这些工具有 owner-only 保护。非 owner sender 默认看不到，或者即使包装后执行也会报错。

相关源码里叫：

```text
applyOwnerOnlyToolPolicy(...)
```

这表示工具隔离不只看 Agent 配置，也看消息发送者是不是 owner。

## subagent 工具有额外限制

Subagent 不是普通主 Agent。它会被额外限制一些系统级和协调类工具。

默认会限制的方向包括：

- 不让 subagent 直接操作 `gateway`
- 不让 subagent 随便 `sessions_send`
- 叶子 subagent 不能继续无限制 spawn 子任务

相关逻辑在：

```text
src/agents/pi-tools.policy.ts
```

里面有 `SUBAGENT_TOOL_DENY_ALWAYS` 和 `SUBAGENT_TOOL_DENY_LEAF` 这类规则。

## fail closed：配置错了会直接停

OpenClaw 对工具 allowlist 是偏保守的。

如果你写了：

```json5
{
  agents: {
    list: [
      {
        id: "dbbot",
        tools: {
          allow: ["query_db"],
        },
      },
    ],
  },
}
```

但实际没有任何内置工具或已启用插件注册 `query_db`，OpenClaw 不会假装它能用，也不会让模型继续“脑补查询结果”。

它会在模型调用前失败。

这叫 fail closed：宁愿停止，也不让配置错误变成不安全或不真实的行为。

## 源码入口

理解工具隔离主要看这些文件：

- `docs/tools/index.md`
- `docs/tools/multi-agent-sandbox-tools.md`
- `src/config/types.tools.ts`
- `src/agents/pi-tools.policy.ts`
- `src/agents/tool-policy.ts`
- `src/agents/tool-policy-shared.ts`
- `src/agents/tool-policy-pipeline.ts`
- `src/agents/pi-embedded-runner/effective-tool-policy.ts`
- `src/agents/sandbox/tool-policy.ts`

## 新手记忆版

```text
agentId 决定当前是哪一个 Agent。

当前 Agent 的 tools 配置决定它能看到哪些工具。

deny 比 allow 更强。

profile 是基础工具包。

alsoAllow 是在 profile 上额外加工具。

sandbox 是执行环境隔离，不等于工具列表隔离。

最终发给模型的是过滤后的工具列表。
```

