# 06. 架构边界和不变量

这一篇不是讲某个函数，而是讲 OpenClaw 这个仓库最重要的边界。新手先把这些边界记住，后面读代码会少很多迷路。

## 一句话模型

OpenClaw 可以拆成六层：

```text
用户入口: CLI / Control UI / 移动端 / 聊天平台
  -> Gateway: HTTP、WebSocket、认证、配置、会话、调度、插件控制面
  -> Agent runtime: 模型、工具、skills、workspace、transcript
  -> Plugins: provider、channel、tool、CLI backend、setup/onboarding
  -> Apps / nodes: macOS、iOS、Android、headless node
  -> Docs / scripts / tests: 维护和验证体系
```

## Core 不知道具体插件

核心原则：`src/` 里的 core 代码应尽量只认识通用合同，不认识某个具体 provider 或聊天平台的私有行为。

正确方向：

- core 认识 `provider`、`channel`、`tool`、`manifest`、`capability` 这些抽象。
- OpenAI、Telegram、Discord、Slack、WhatsApp 等具体差异放在 `extensions/<id>/`。
- 如果多个插件都需要同一类能力，先考虑抽成 SDK seam 或通用 contract。

危险信号：

- 在 core 里硬编码某个 plugin id。
- core test 直接 import `extensions/*/src/**`。
- 一个插件直接 import 另一个插件的 `src/**`。
- 插件生产代码绕过 `openclaw/plugin-sdk/*` 去 import core 内部路径。

相关入口：

- `AGENTS.md`
- `docs/plugins/architecture.md`
- `src/plugins/registry.ts`
- `src/plugin-sdk/plugin-entry.ts`

## 控制面和运行面分开

OpenClaw 插件系统有一个非常关键的设计：很多信息必须在“不执行插件代码”的情况下可用。

控制面负责：

- 读 `openclaw.plugin.json`
- 验证 config schema
- 生成 UI 表单 hints
- 判断 provider/channel/command/tool 的 owner
- 做 activation planning
- 给 Gateway 启动前的检查提供 metadata

运行面负责：

- 真正 import 插件 module
- 调用 `register(api)`
- 注册 provider、channel、tool、hook、runtime service
- 执行平台 API 请求或模型请求

为什么要这么分：

- 配置错误应该在插件 runtime 加载前就能报出来。
- Gateway 启动不应该为了列出插件就执行全部插件代码。
- UI 和 onboarding 需要 schema，但不应该触发 provider SDK 初始化。
- 第三方插件存在，控制面 metadata 更容易做安全和兼容性检查。

相关入口：

- `docs/plugins/manifest.md`
- `docs/plugins/architecture.md`
- `src/plugins/discovery.ts`
- `src/plugins/activation-planner.ts`
- `src/plugins/metadata-snapshot.ts`

## Gateway 是长驻协调器

Gateway 不是“模型运行器”本身，而是长驻协调器。它负责把很多系统接起来：

- HTTP server
- WebSocket server
- Control UI 静态资源
- Gateway protocol req/res/event
- auth、device pairing、rate limit
- config load / reload
- sessions 和 transcript
- channels manager
- cron / tasks / hooks
- plugin metadata 和 runtime activation
- agent run 调度和事件转发

读 Gateway 时不要从 `src/gateway/server.impl.ts` 第一行读到最后一行。它是汇总层，要按问题定位：

- 启动：`startGatewayServer`
- WebSocket：`server-ws-runtime`
- RPC method：`src/gateway/server-methods/*`
- 频道：`server-channels`
- protocol：`src/gateway/protocol/*`
- Control UI 静态服务：`control-ui`

## Agent runtime 拥有一次“思考和执行”

一次 agent run 大致包括：

1. 解析 agent、session、workspace、model。
2. 加载 skills 和 bootstrap files。
3. 准备 auth profile。
4. 组装 prompt 和上下文。
5. 选择 embedded Pi runtime 或 CLI backend。
6. 流式产出 assistant/tool/lifecycle event。
7. 写 transcript。
8. 返回 final result 或错误。

相关入口：

- `docs/concepts/agent.md`
- `docs/concepts/agent-loop.md`
- `src/agents/agent-command.ts`
- `src/agents/command/attempt-execution.ts`
- `src/agents/pi-embedded.ts`
- `src/agents/cli-runner.ts`

## UI 只通过 Gateway 合同通信

Control UI 是前端应用，不直接读写本地 state 文件，不直接执行 Agent。它通过 Gateway WebSocket 做 RPC 和事件订阅。

关键关系：

```text
ui/src/main.ts
  -> ui/src/ui/app.ts
  -> ui/src/ui/gateway.ts
  -> GatewayBrowserClient
  -> Gateway WebSocket protocol
```

聊天页面不是“直接调用模型”，而是：

```text
app-chat.ts
  -> controllers/chat.ts
  -> client.request("chat.send")
  -> Gateway server-method
  -> agentCommand
  -> Gateway event stream
  -> app-gateway.ts 更新 UI state
```

相关入口：

- `docs/web/control-ui.md`
- `ui/src/ui/gateway.ts`
- `ui/src/ui/app-gateway.ts`
- `ui/src/ui/controllers/chat.ts`
- `ui/src/ui/app-chat.ts`

## Apps 和 nodes 是接入面

`apps/` 不是核心 TypeScript 逻辑的简单附属，它包含平台客户端：

- `apps/macos/`：macOS app / Gateway 管理体验。
- `apps/ios/`：iOS node。
- `apps/android/`：Android node。
- `apps/shared/`：跨 Apple 平台共享代码。

nodes 通过 Gateway protocol 以 `node` 角色接入，声明 capabilities，然后执行 Gateway 下发的命令，例如 camera、screen、system.run 等。

相关入口：

- `docs/nodes/*`
- `src/gateway/protocol/schema/nodes.ts`
- `src/gateway/server-node-session-runtime.ts`
- `src/node-host/*`

## 新手最容易踩的边界

不要把“目录名 extensions”写到用户文档或产品文案里。对外叫 plugin/plugins。

不要为了修某个插件问题去 core 加私有特殊逻辑。先在 owner plugin 里修。

不要绕过 protocol schema 直接传 ad hoc JSON。Gateway、UI、node 都依赖 schema 稳定。

不要随便改 config key。config 是公开合同，涉及 schema、docs、UI、baseline、doctor/migration。

不要把旧配置兼容写进启动热路径。旧配置修复更适合 doctor/fix 或明确的 raw migration。

不要把动态 import 改成静态 import。这个仓库很多动态 import 是为了启动性能和加载边界。

