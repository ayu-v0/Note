# 12. 具体架构图

这篇比 `11-project-structure-diagram.md` 更细。重点不是列目录，而是把“请求从哪里来、经过哪些模块、最后到哪里去”画清楚。

## 图 1：整体系统架构

```mermaid
flowchart TB
  subgraph Entrypoints["用户入口"]
    ChatPlatforms["聊天平台\nTelegram / WhatsApp / Slack / Discord / ..."]
    WebUI["Control UI\nui/src/ui/*"]
    CLI["CLI\nopenclaw.mjs\nsrc/entry.ts\nsrc/cli/*"]
    Apps["macOS / iOS / Android Apps\napps/*"]
    Automations["Cron / Hooks / Tasks\nsrc/cron src/hooks src/tasks"]
  end

  subgraph Gateway["Gateway 常驻进程\nsrc/gateway/server.impl.ts"]
    HttpServer["HTTP server\nControl UI / Canvas / routes"]
    WsServer["WebSocket server\nsrc/gateway/protocol/*"]
    Auth["认证 + 设备配对\nsrc/gateway/auth.ts\nsrc/gateway/device-auth.ts"]
    Methods["RPC methods\nsrc/gateway/server-methods/*"]
    ChannelManager["Channel manager\nsrc/gateway/server-channels.ts"]
    RuntimeServices["运行时服务\ncron / approvals / media / nodes"]
  end

  subgraph Core["核心域模型"]
    Config["配置\nsrc/config/*"]
    Routing["路由\nsrc/routing/*"]
    Sessions["Session + transcript\nsrc/config/sessions/*"]
    Agents["Agent runtime\nsrc/agents/*"]
    Tools["Agent tools\nsrc/agents/tools/*"]
  end

  subgraph Plugins["Plugins\nextensions/*"]
    ChannelPlugins["Channel plugins\ntelegram / discord / slack / ..."]
    ProviderPlugins["Provider plugins\nopenai / anthropic / google / ..."]
    ToolPlugins["Tool/search/media plugins\nbrave / tavily / fal / ..."]
  end

  subgraph External["外部系统"]
    LLMs["模型/provider API"]
    MessagingApis["聊天平台 API"]
    DeviceApis["设备能力\ncamera / screen / location"]
  end

  ChatPlatforms --> ChannelPlugins
  ChannelPlugins --> ChannelManager
  WebUI --> WsServer
  CLI --> WsServer
  Apps --> WsServer
  Automations --> Methods

  WsServer --> Auth
  Auth --> Methods
  Methods --> Routing
  Methods --> RuntimeServices
  ChannelManager --> Routing
  Routing --> Sessions
  Routing --> Agents
  Agents --> Tools
  Agents --> ProviderPlugins
  Tools --> ToolPlugins

  ChannelPlugins <--> MessagingApis
  ProviderPlugins <--> LLMs
  RuntimeServices <--> DeviceApis

  Config --> Gateway
  Config --> Core
  Plugins --> Gateway
```

怎么读这张图：

- 左边是入口：用户可能从聊天平台、浏览器、CLI、App 或自动化任务进来。
- 中间是 Gateway：它负责认证、协议、RPC、频道生命周期和运行时服务。
- 下方是核心域模型：配置、路由、session、Agent、tools。
- 右边是 plugins 和外部系统：具体 provider、聊天平台、搜索、媒体都在插件侧。

## 图 2：Gateway 内部更细结构

```mermaid
flowchart LR
  subgraph Startup["启动链路"]
    Launcher["openclaw.mjs"]
    Entry["src/entry.ts"]
    RunMain["src/cli/run-main.ts"]
    GatewayCli["src/cli/gateway-cli/run.ts"]
    Server["src/gateway/server.ts"]
    ServerImpl["src/gateway/server.impl.ts"]
  end

  subgraph GatewayImpl["src/gateway/server.impl.ts 组装的服务"]
    ConfigLoad["读取配置\nsrc/config/io.ts"]
    PluginBootstrap["插件 metadata + runtime bootstrap\nsrc/plugins/*"]
    Http["HTTP host\nControl UI / Canvas"]
    Ws["WS handlers\nconnect / req / res / event"]
    Auth["auth + pairing"]
    RpcDispatch["RPC dispatch\nserver-methods/*"]
    ChannelSvc["Channel lifecycle\nserver-channels.ts"]
    CronSvc["Cron service\nserver-cron.ts"]
    NodeSvc["Node sessions\nserver-node-session-runtime.ts"]
    ApprovalSvc["Exec approvals\nexec-approval-manager.ts"]
  end

  Launcher --> Entry --> RunMain --> GatewayCli --> Server --> ServerImpl
  ServerImpl --> ConfigLoad
  ServerImpl --> PluginBootstrap
  ServerImpl --> Http
  ServerImpl --> Ws
  Ws --> Auth
  Auth --> RpcDispatch
  ServerImpl --> ChannelSvc
  ServerImpl --> CronSvc
  ServerImpl --> NodeSvc
  ServerImpl --> ApprovalSvc
```

重点：

- `src/gateway/server.ts` 是轻入口，实际重活在 `src/gateway/server.impl.ts`。
- Gateway 启动时会同时准备 config、plugin、HTTP、WebSocket、channel、cron、node、approval 等服务。
- 外部调用不是随便进内部函数，而是走 WebSocket frame，再分发到 `src/gateway/server-methods/*`。

## 图 3：WebSocket 协议结构

```mermaid
sequenceDiagram
  participant Client as UI/CLI/App/Node
  participant WS as Gateway WS
  participant Auth as Auth + Device Pairing
  participant Methods as server-methods/*
  participant Events as Gateway Events

  Client->>WS: connect params\nminProtocol/maxProtocol/client/device/auth
  WS->>Auth: validate token/password/device signature
  Auth-->>WS: accepted or rejected
  WS-->>Client: hello-ok\nfeatures + snapshot + policy + auth

  Client->>WS: req {id, method, params}
  WS->>Methods: dispatch by method
  Methods-->>WS: result or error
  WS-->>Client: res {id, ok, payload/error}

  Events-->>WS: presence/chat/agent/health/tick
  WS-->>Client: event {event, payload, seq, stateVersion}
```

对应源码：

- frame schema：`src/gateway/protocol/schema/frames.ts`
- 协议汇总和 validator：`src/gateway/protocol/index.ts`
- 浏览器客户端：`ui/src/ui/gateway.ts`
- Gateway client：`src/gateway/client.ts`

## 图 4：Control UI 发一条消息

```mermaid
sequenceDiagram
  participant User as 用户
  participant ChatUI as ui/src/ui/app-chat.ts
  participant ChatCtl as ui/src/ui/controllers/chat.ts
  participant WsClient as ui/src/ui/gateway.ts
  participant Gateway as Gateway WS
  participant Method as src/gateway/server-methods/*
  participant Agent as src/agents/agent-command.ts
  participant Runtime as src/agents/command/attempt-execution.ts
  participant Provider as Provider plugin

  User->>ChatUI: 输入消息并发送
  ChatUI->>ChatCtl: chat.send(...)
  ChatCtl->>WsClient: req method=chat.send
  WsClient->>Gateway: WebSocket req
  Gateway->>Method: dispatch chat.send
  Method-->>WsClient: ack runId / accepted
  Method->>Agent: start agent run
  Agent->>Runtime: attempt execution
  Runtime->>Provider: model inference / stream / tools
  Provider-->>Runtime: assistant/tool/lifecycle events
  Runtime-->>Gateway: agent stream events
  Gateway-->>WsClient: event agent/chat delta/final
  WsClient-->>ChatUI: apply events to UI state
```

为什么 `chat.send` 不是直接返回最终文本：

- Agent run 可能很久。
- 回复可能是流式的。
- 工具调用、生命周期、错误、最终消息都需要通过事件持续推给 UI。
- 所以先返回 `runId`，后续通过 `event` 更新界面。

## 图 5：聊天平台消息进入 Agent

```mermaid
sequenceDiagram
  participant Platform as Telegram/Slack/Discord/...
  participant Plugin as extensions/<channel>/
  participant ChannelMgr as src/gateway/server-channels.ts
  participant Routing as src/routing/*
  participant Session as Session store / transcript
  participant Agent as src/agents/agent-command.ts
  participant Provider as Provider plugin
  participant Reply as Channel reply pipeline

  Platform->>Plugin: inbound platform event
  Plugin->>ChannelMgr: normalized channel message
  ChannelMgr->>Routing: resolve channel/account/session/agent
  Routing->>Session: load or create session
  Routing->>Agent: enqueue agent run
  Agent->>Provider: model/tool execution
  Provider-->>Agent: response payloads
  Agent->>Session: write transcript
  Agent-->>Reply: final reply payload
  Reply->>Plugin: channel-specific send/edit/react
  Plugin->>Platform: outbound message
```

这个图的边界很重要：

- 平台细节在 `extensions/<channel>/`。
- Gateway 只处理通用 channel/session/agent 合同。
- 发送回平台时，也要回到 channel plugin 做平台格式转换。

## 图 6：Agent runtime 内部

```mermaid
flowchart TB
  AgentRpc["Gateway RPC: agent / agent.wait"]
  AgentCommand["agentCommand\nsrc/agents/agent-command.ts"]
  Resolve["解析运行上下文\nsession / workspace / model / thinking / skills / auth profile"]
  Queue["session lane + global queue\n串行化同一 session"]
  SessionLock["session write lock\n保护 transcript"]
  Attempt["attemptExecution\nsrc/agents/command/attempt-execution.ts"]
  Embedded["Embedded Pi runtime\nsrc/agents/pi-embedded.ts"]
  CliBackend["CLI backend\nsrc/agents/cli-runner.ts"]
  EventBridge["event bridge\nassistant / tool / lifecycle"]
  Transcript["transcript + usage + result"]
  Wait["agent.wait\nwaitForAgentRun"]

  AgentRpc --> AgentCommand
  AgentCommand --> Resolve
  Resolve --> Queue
  Queue --> SessionLock
  SessionLock --> Attempt
  Attempt --> Embedded
  Attempt --> CliBackend
  Embedded --> EventBridge
  CliBackend --> EventBridge
  EventBridge --> Transcript
  AgentRpc --> Wait
  EventBridge --> Wait
```

Agent run 主要做这些事：

- 解析 session、workspace、模型、skills、auth profile。
- 用队列保证同一个 session 不并发乱写。
- 加 session write lock 保护 transcript。
- 根据 provider/runtime 走 embedded Pi runtime 或 CLI backend。
- 把 assistant/tool/lifecycle 事件桥接回 Gateway。

## 图 7：Plugin 加载和能力注册

```mermaid
flowchart TB
  Config["openclaw.json\nplugins/channels/models config"]
  Discovery["Plugin discovery\nread openclaw.plugin.json"]
  Manifest["Manifest registry\nmetadata only"]
  Validation["enablement + config validation"]
  Plan["activation plan\n按 provider/channel/command/capability 缩小加载范围"]
  RuntimeLoad["runtime loading\nextensions/<id>/index.ts"]
  RegisterApi["register(api)"]
  Registry["central registry\nsrc/plugins/registry.ts"]
  Consumers["consumers\nGateway / Agent / Tools / Channels / UI schema"]

  Config --> Discovery
  Discovery --> Manifest
  Manifest --> Validation
  Validation --> Plan
  Plan --> RuntimeLoad
  RuntimeLoad --> RegisterApi
  RegisterApi --> Registry
  Registry --> Consumers
```

插件分两层看：

- control plane：只读 manifest，不执行插件代码，用来发现、校验、规划。
- runtime plane：加载 `index.ts`，执行 `register(api)`，把 provider/channel/tool/media/search 等能力注册进 registry。

常见注册点：

```text
api.registerProvider(...)
api.registerCliBackend(...)
api.registerChannel(...)
api.registerWebSearchProvider(...)
api.registerWebFetchProvider(...)
api.registerImageGenerationProvider(...)
api.registerSpeechProvider(...)
```

## 图 8：配置如何影响运行

```mermaid
flowchart LR
  Json["~/.openclaw/openclaw.json\nJSON5"]
  Env["env substitution\nOPENCLAW_*"]
  Includes["includes"]
  Schema["strict schema validation\nsrc/config/types.openclaw.ts"]
  Snapshot["runtime config snapshot"]
  GatewayStart["Gateway startup"]
  PluginConfig["plugin config schema"]
  UiConfig["Control UI config forms"]
  Runtime["runtime behavior\nchannels/models/tools/gateway"]

  Json --> Env --> Includes --> Schema --> Snapshot
  Snapshot --> GatewayStart
  Snapshot --> PluginConfig
  Snapshot --> UiConfig
  Snapshot --> Runtime
```

对应源码：

- config 类型：`src/config/types.openclaw.ts`
- config 读写：`src/config/io.ts`
- config 路径：`src/config/paths.ts`
- plugin config schema：`extensions/*/openclaw.plugin.json`

## 图 9：新手最该先掌握的 5 条链路

```text
1. 启动链路
   openclaw.mjs
   -> src/entry.ts
   -> src/cli/run-main.ts
   -> src/cli/gateway-cli/run.ts
   -> src/gateway/server.ts
   -> src/gateway/server.impl.ts

2. UI 请求链路
   ui/src/ui/views/*
   -> ui/src/ui/controllers/*
   -> ui/src/ui/gateway.ts
   -> src/gateway/server-methods/*

3. Agent 执行链路
   src/gateway/server-methods/*
   -> src/agents/agent-command.ts
   -> src/agents/command/attempt-execution.ts
   -> provider plugin / embedded runtime / CLI backend

4. 插件链路
   extensions/<id>/openclaw.plugin.json
   -> extensions/<id>/index.ts
   -> src/plugin-sdk/plugin-entry.ts
   -> src/plugins/registry.ts

5. 协议链路
   src/gateway/protocol/schema/frames.ts
   -> src/gateway/protocol/index.ts
   -> ui/src/ui/gateway.ts
   -> src/gateway/client.ts
```

## 验证命令

```bash
node scripts/docs-list.js
rg "ConnectParamsSchema|RequestFrameSchema|EventFrameSchema" src/gateway/protocol
rg "agentCommand" src/agents src/gateway
rg "attemptExecution|runEmbeddedPiAgent|runCliAgent" src/agents
rg "definePluginEntry|registerProvider|registerChannel" src/plugin-sdk src/plugins extensions
rg "chat.send|agent.wait" ui/src src/gateway
```

## 资料来源

- `docs/concepts/architecture.md`
- `docs/concepts/agent-loop.md`
- `docs/plugins/architecture.md`
- `src/gateway/protocol/schema/frames.ts`
- `.workplace/learn/11-project-structure-diagram.md`
