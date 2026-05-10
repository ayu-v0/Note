# 11. 项目结构图

这篇先回答一个新手最容易卡住的问题：这个仓库这么大，哪些目录是主线，哪些目录是插件、前端、App、文档和验证工具。

## 一句话总览

OpenClaw 是一个多入口 AI Gateway。用户可以从聊天平台、浏览器 Control UI、CLI、移动端节点进入；Gateway 负责认证、协议、路由、插件、会话和 Agent 执行调度；插件负责具体平台或 provider 能力；UI 和 App 只是不同的操作入口。

```text
聊天平台 / Control UI / CLI / macOS App / iOS App / Android App
        |
        v
Gateway: HTTP + WebSocket + auth + config + routing + sessions
        |
        +--> Channels: Telegram / WhatsApp / Slack / Discord / ...
        |
        +--> Plugins: providers / tools / media / search / channel adapters
        |
        +--> Agent runtime: model + tools + skills + workspace + transcript
        |
        +--> Nodes: macOS / iOS / Android / headless device commands
```

## 仓库顶层结构

```text
openclaw/
  openclaw.mjs              CLI 启动器，用户运行 openclaw 时先到这里
  package.json              npm 包入口、scripts、exports、workspace 配置入口
  src/                      核心 TypeScript 代码
  ui/                       浏览器 Control UI，Vite + Lit
  extensions/               内置 plugins，对外文案叫 plugins
  packages/                 发布用 SDK 和共享 package
  apps/                     macOS / iOS / Android / 共享 app 代码
  docs/                     官方文档源
  scripts/                  构建、测试、生成、发布、CI 辅助脚本
  test/                     测试 helpers、fixtures、配置
  skills/                   bundled 或项目内 skill 资源
  .github/                  GitHub Actions、labeler、CI 配置
  .workplace/learn/         给新手看的本地学习笔记
```

## 核心代码结构

```text
src/
  entry.ts                  TypeScript CLI 入口
  cli/                      CLI 命令注册、参数解析、命令路由
  commands/                 具体 CLI 命令实现
  gateway/                  Gateway server、HTTP、WS、auth、methods、nodes
    protocol/               WebSocket wire protocol schema、类型、validator
    server-methods/         Gateway RPC 方法实现
  agents/                   Agent run、工具、skills、workspace、transcript
    command/                一次 Agent 执行的具体流程
    tools/                  暴露给 Agent 的工具
  plugins/                  plugin discovery、manifest、registry、activation
  plugin-sdk/               plugin 作者使用的 public facade
  channels/                 channel 抽象、contracts、plugin bridge
  config/                   openclaw.json 类型、schema、读写、校验
  routing/                  channel/session/agent 路由
  cron/                     定时任务
  tasks/                    background task tracking
  hooks/                    hooks 机制
  acp/                      ACP agent bridge
  mcp/                      MCP server/client 能力
  node-host/                headless node host
  secrets/                  SecretRef 和 credentials
  terminal/                 CLI 表格和终端输出
  tui/                      Terminal UI
```

新手读 `src/` 时不要从目录顺序读。先按运行链路读：

```text
openclaw.mjs
  -> src/entry.ts
  -> src/cli/run-main.ts
  -> src/cli/gateway-cli/run.ts
  -> src/gateway/server.ts
  -> src/gateway/server.impl.ts
```

## Gateway 内部结构

Gateway 是长期运行的中心进程。它把 UI、CLI、聊天平台、移动端节点和 Agent runtime 连在一起。

```text
src/gateway/
  server.ts                 轻量入口，延迟加载实际实现
  server.impl.ts            Gateway 启动和生命周期主实现
  client.ts                 Gateway WS client
  auth.ts                   Gateway token/password/trusted-proxy 等认证
  device-auth.ts            设备配对和 device token
  server-channels.ts        启停 channel account，接入聊天平台
  server-cron.ts            Gateway cron service
  exec-approval-manager.ts  工具执行审批
  control-ui.ts             服务 Control UI 静态资源
  protocol/                 WS 帧、方法、事件、schema
  server-methods/           chat、agent、config、health 等 RPC 方法
```

Gateway 的最重要边界：

- 外部客户端走 WebSocket 协议，不直接调用内部函数。
- 协议 schema 在 `src/gateway/protocol/`。
- 具体 RPC 行为在 `src/gateway/server-methods/`。
- 聊天平台生命周期由 `src/gateway/server-channels.ts` 串起来。

## 插件结构

插件对外叫 plugins，仓库内部目录叫 `extensions/`。

```text
extensions/
  openai/                   provider plugin 示例
    openclaw.plugin.json    manifest，静态描述插件能力和配置
    index.ts                runtime 注册入口
  telegram/                 channel plugin 示例
    openclaw.plugin.json
    index.ts
  brave/                    search plugin 示例
  fal/                      media plugin 示例
  codex/                    coding harness plugin 示例
```

一个插件通常先看两个文件：

```text
extensions/<id>/openclaw.plugin.json
extensions/<id>/index.ts
```

理解顺序：

```text
manifest: 插件是谁、有什么配置、声明哪些能力
        |
        v
runtime index.ts: 插件加载后调用 register(api)
        |
        v
src/plugins/registry.ts: core 通过通用 capability 使用插件
```

## Control UI 结构

Control UI 是浏览器里的操作台，由 Gateway 服务静态资源，通过 WebSocket 和 Gateway 通信。

```text
ui/
  src/main.ts               前端入口
  src/ui/app.ts             根 Lit component 和全局状态
  src/ui/gateway.ts         GatewayBrowserClient
  src/ui/app-gateway.ts     连接 Gateway、处理 hello/snapshot/events
  src/ui/app-chat.ts        聊天发送、停止、流式事件应用
  src/ui/controllers/       RPC/controller 层
  src/ui/views/             页面视图
  src/i18n/                 国际化
  public/                   静态资源
```

UI 修改一般按这个顺序找：

```text
views/<page>.ts
  -> controllers/<feature>.ts
  -> gateway.ts / app-gateway.ts
  -> src/gateway/server-methods/<feature>.ts
  -> src/gateway/protocol/schema/*
```

## Apps 结构

```text
apps/
  macos/                    macOS app、Gateway 管理、平台集成
  ios/                      iOS node app
  android/                  Android node app
  shared/                   Apple 平台共享模型和协议
  macos-mlx-tts/            macOS 本地 TTS sidecar
  swabble/                  独立 app 或实验面
```

App 不是 core 的替代品。它们通常连接 Gateway，或作为 node 暴露设备能力。

## 文档和验证结构

```text
docs/
  start/                    安装、上手、onboarding
  concepts/                 架构、Agent、session、features
  cli/                      CLI 命令参考
  gateway/                  Gateway 配置和运行
  web/                      Control UI、WebChat、TUI
  channels/                 各聊天平台
  plugins/                  plugin 架构、manifest、SDK
  tools/                    agent tools、skills、browser、media、search
  automation/               cron、tasks、hooks、standing orders
  reference/                测试、release、token、templates

scripts/
  check-changed.mjs         changed gate 入口
  changed-lanes.mjs         判断改动需要哪些 lane
  test-projects.mjs         测试项目包装入口
  run-vitest.mjs            Vitest 包装器
  docs-list.js              docs 索引
```

## 一次消息的结构流

```text
用户发消息
  |
  +-- Control UI: ui/src/ui/app-chat.ts
  |       -> ui/src/ui/controllers/chat.ts
  |
  +-- 聊天平台 plugin: extensions/<channel>/
          -> src/gateway/server-channels.ts
  |
  v
Gateway RPC / channel event
  -> src/gateway/server-methods/*
  -> src/routing/*
  -> src/agents/agent-command.ts
  -> src/agents/command/attempt-execution.ts
  -> provider plugin / embedded runtime / CLI backend
  -> transcript + streaming events
  -> Gateway event
  -> UI 或原聊天平台回复
```

## 新手查代码的入口口诀

- 想看启动：`openclaw.mjs`、`src/entry.ts`、`src/cli/run-main.ts`。
- 想看 Gateway：`src/cli/gateway-cli/run.ts`、`src/gateway/server.impl.ts`。
- 想看协议：`src/gateway/protocol/schema/frames.ts`。
- 想看 UI 发请求：`ui/src/ui/controllers/`。
- 想看聊天界面：`ui/src/ui/app-chat.ts`。
- 想看 Agent 执行：`src/agents/agent-command.ts`。
- 想看插件：`extensions/<id>/openclaw.plugin.json` 和 `extensions/<id>/index.ts`。
- 想看配置：`src/config/types.openclaw.ts` 和 `src/config/io.ts`。

## 验证命令

```bash
node scripts/docs-list.js
rg "startGatewayServer" src/gateway src/cli
rg "GatewayBrowserClient" ui/src
rg "definePluginEntry" extensions src/plugin-sdk
rg "registerProvider|registerChannel" extensions src/plugins
rg "agentCommand" src/agents src/gateway
```

## 资料来源

- `docs/concepts/architecture.md`
- `.workplace/learn/03-feature-to-code-map.md`
- `.workplace/learn/08-subsystem-map.md`
- `package.json`
