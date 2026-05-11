# 17. 工具注册机制和外部扩展工具

这篇解释：

```text
“工具是否注册”是什么意思？
OpenClaw 的工具注册机制是什么？
如果我要扩展一个外部工具，应该怎么做？
```

## 先给结论

工具不是只要写在配置里就能用。

一个工具要能被 Agent 使用，至少要满足三件事：

```text
1. 有工具实现代码
2. 工具被注册到 OpenClaw 的 plugin registry
3. 当前 Agent 的工具策略允许它出现
```

配置里的 `tools.allow` / `agents.list[].tools.allow` 只是“允许使用某个工具名”。如果没有任何内置工具或插件真正注册这个名字，这个工具仍然不存在。

所以：

```text
配置 = 允许范围
注册 = 让 OpenClaw 知道这个工具真实存在，并知道怎么执行它
```

## 什么叫“工具注册”

“工具注册”就是某段代码在 OpenClaw 启动或插件激活时调用：

```ts
api.registerTool(...)
```

把一个工具交给 OpenClaw。

这个工具通常包含：

- `name`：工具名，例如 `my_tool`
- `description`：给模型看的工具说明
- `parameters`：参数 schema，告诉模型怎么传参
- `execute`：真正执行工具的函数

示意：

```ts
api.registerTool({
  name: "my_tool",
  description: "Do a thing",
  parameters: Type.Object({
    input: Type.String(),
  }),
  async execute(_id, params) {
    return {
      content: [{ type: "text", text: `Got: ${params.input}` }],
    };
  },
});
```

注册后，OpenClaw 才知道：

```text
有一个叫 my_tool 的工具；
它需要 input 参数；
模型可以按 schema 调用它；
调用时要执行 execute 函数。
```

## 内置工具和插件工具

OpenClaw 的工具来源大致有几类。

### 内置工具

内置工具由 core 创建，例如：

- `read`
- `write`
- `edit`
- `apply_patch`
- `exec`
- `web_search`
- `web_fetch`
- `message`
- `sessions_*`
- `cron`
- `gateway`
- `nodes`
- `image_generate`

核心入口之一是：

```text
src/agents/pi-tools.ts
```

里面的 `createOpenClawCodingTools(...)` 会创建一批 OpenClaw 内置工具，再经过工具策略过滤。

### 插件工具

插件工具由 plugin 注册。

插件入口通常长这样：

```ts
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";

export default definePluginEntry({
  id: "my-plugin",
  name: "My Plugin",
  description: "Adds a custom tool",
  register(api) {
    api.registerTool({
      name: "my_tool",
      description: "Do a thing",
      parameters: Type.Object({
        input: Type.String(),
      }),
      async execute(_id, params) {
        return {
          content: [{ type: "text", text: `Got: ${params.input}` }],
        };
      },
    });
  },
});
```

源码入口：

- `src/plugin-sdk/plugin-entry.ts`
- `src/plugins/registry.ts`
- `docs/plugins/building-plugins.md`
- `docs/plugins/sdk-overview.md`

## manifest 也必须声明工具名

OpenClaw 的 native plugin 不只要在 runtime 里 `registerTool`，还要在插件根目录的 `openclaw.plugin.json` 里声明工具名。

示例：

```json
{
  "id": "my-plugin",
  "name": "My Plugin",
  "description": "Adds a custom tool to OpenClaw",
  "contracts": {
    "tools": ["my_tool"]
  },
  "activation": {
    "onStartup": true
  },
  "configSchema": {
    "type": "object",
    "additionalProperties": false
  }
}
```

这里的关键是：

```json
"contracts": {
  "tools": ["my_tool"]
}
```

它的作用是告诉 OpenClaw：

```text
这个插件声明自己拥有 my_tool 这个工具。
```

为什么需要 manifest？

- OpenClaw 可以不执行插件 runtime，就知道插件拥有哪些工具。
- config validation 可以判断 `tools.allow: ["my_tool"]` 该由哪个插件负责。
- activation planner 可以知道什么时候需要加载这个插件。
- 插件工具和配置表单、诊断、发现逻辑能对齐。

## runtime 注册必须和 manifest 对齐

源码里 `src/plugins/registry.ts` 的 `registerTool(...)` 会检查：

```text
runtime 注册的工具名
  是否在 manifest contracts.tools 里声明过
```

如果插件 runtime 里写了：

```ts
api.registerTool({
  name: "my_tool",
  /* ... */
});
```

但 manifest 没写：

```json
"contracts": {
  "tools": ["my_tool"]
}
```

OpenClaw 会报插件诊断：

```text
plugin must declare contracts.tools for: my_tool
```

反过来，如果配置里允许了一个工具，但没有任何已启用插件注册它，OpenClaw 也不会让模型继续假装能用。

这就是前面说的 fail closed。

## 一个真实例子：Firecrawl

`extensions/firecrawl/openclaw.plugin.json` 里声明：

```json
{
  "id": "firecrawl",
  "contracts": {
    "webFetchProviders": ["firecrawl"],
    "webSearchProviders": ["firecrawl"],
    "tools": ["firecrawl_search", "firecrawl_scrape"]
  }
}
```

`extensions/firecrawl/index.ts` 里 runtime 注册：

```ts
export default definePluginEntry({
  id: "firecrawl",
  name: "Firecrawl Plugin",
  description: "Bundled Firecrawl search and scrape plugin",
  register(api) {
    api.registerWebFetchProvider(createFirecrawlWebFetchProvider());
    api.registerWebSearchProvider(createFirecrawlWebSearchProvider());
    api.registerTool(createFirecrawlSearchTool(api) as AnyAgentTool);
    api.registerTool(createFirecrawlScrapeTool(api) as AnyAgentTool);
  },
});
```

也就是说：

```text
manifest 声明工具归属
runtime 注册工具实现
config 决定当前 Agent 能不能用
```

## 外部扩展工具怎么做

如果你要给 OpenClaw 增加一个外部工具，推荐做成插件，而不是直接改 core。

最小流程：

1. 创建一个 plugin package。
2. 写 `openclaw.plugin.json`，声明 `contracts.tools`。
3. 写 `index.ts`，用 `definePluginEntry` 导出插件入口。
4. 在 `register(api)` 里调用 `api.registerTool(...)`。
5. 安装插件。
6. 在 `openclaw.json` 里启用插件，并允许工具。

## 外部插件最小结构

```text
my-openclaw-plugin/
  package.json
  openclaw.plugin.json
  index.ts
```

`package.json` 示例：

```json
{
  "name": "@myorg/openclaw-my-plugin",
  "version": "1.0.0",
  "type": "module",
  "openclaw": {
    "extensions": ["./index.ts"]
  }
}
```

`openclaw.plugin.json` 示例：

```json
{
  "id": "my-plugin",
  "name": "My Plugin",
  "description": "Adds one custom tool",
  "contracts": {
    "tools": ["my_tool"]
  },
  "activation": {
    "onStartup": true
  },
  "configSchema": {
    "type": "object",
    "additionalProperties": false
  }
}
```

`index.ts` 示例：

```ts
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
import { Type } from "@sinclair/typebox";

export default definePluginEntry({
  id: "my-plugin",
  name: "My Plugin",
  description: "Adds one custom tool",
  register(api) {
    api.registerTool({
      name: "my_tool",
      description: "Echo input text",
      parameters: Type.Object({
        input: Type.String(),
      }),
      async execute(_id, params) {
        return {
          content: [{ type: "text", text: String(params.input) }],
        };
      },
    });
  },
});
```

## 安装外部插件

官方文档给的安装形式包括：

```bash
openclaw plugins install clawhub:@myorg/openclaw-my-plugin
openclaw plugins install npm:@myorg/openclaw-my-plugin
openclaw plugins install git:github.com/acme/openclaw-plugin@v1.0.0
openclaw plugins install ./my-plugin
openclaw plugins install -l ./my-plugin
```

本地开发时常用：

```bash
openclaw plugins install -l ./my-plugin
```

`-l` 是 link，适合开发调试。

## 配置启用和允许工具

安装插件后，还要确保配置允许它加载，并允许对应工具出现在 Agent 工具列表里。

示例：

```json5
{
  plugins: {
    entries: {
      "my-plugin": {
        enabled: true,
        config: {},
      },
    },
  },
  tools: {
    alsoAllow: ["my_tool"],
  },
}
```

如果是某个 Agent 才能用：

```json5
{
  agents: {
    list: [
      {
        id: "main",
        tools: {
          alsoAllow: ["my_tool"],
        },
      },
      {
        id: "safe",
        tools: {
          deny: ["my_tool"],
        },
      },
    ],
  },
}
```

如果你的配置使用了 `plugins.allow`，还要把插件 id 加进去：

```json5
{
  plugins: {
    allow: ["my-plugin"],
  },
}
```

否则插件可能根本不会加载。

## optional 工具

插件还可以注册 optional tool：

```ts
api.registerTool(
  {
    name: "workflow_tool",
    description: "Run a workflow",
    parameters: Type.Object({
      pipeline: Type.String(),
    }),
    async execute(_id, params) {
      return {
        content: [{ type: "text", text: String(params.pipeline) }],
      };
    },
  },
  { optional: true },
);
```

manifest 里也可以写 metadata：

```json
{
  "contracts": {
    "tools": ["workflow_tool"]
  },
  "toolMetadata": {
    "workflow_tool": {
      "optional": true
    }
  }
}
```

optional 工具通常用于：

- 有副作用的工具
- 需要额外二进制依赖的工具
- 不希望默认加载的工具
- 只有用户显式 allow 后才应该启用的工具

用户启用：

```json5
{
  tools: {
    allow: ["workflow_tool"],
  },
}
```

## 新手记忆版

```text
工具注册 = 把工具实现交给 OpenClaw registry。

manifest contracts.tools = 声明这个插件拥有哪些工具名。

api.registerTool = 注册工具运行时实现。

tools.allow / agents.list[].tools.allow = 允许某个 Agent 使用这个工具。

安装插件 != 当前 Agent 一定能用工具。

配置允许工具 != 工具已经真实存在。
```

## 相关源码和文档

- `docs/plugins/building-plugins.md`
- `docs/plugins/sdk-overview.md`
- `docs/plugins/sdk-entrypoints.md`
- `docs/tools/plugin.md`
- `docs/tools/index.md`
- `src/plugin-sdk/plugin-entry.ts`
- `src/plugins/registry.ts`
- `src/agents/pi-tools.ts`
- `extensions/firecrawl/openclaw.plugin.json`
- `extensions/firecrawl/index.ts`

