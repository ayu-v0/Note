# 02. 运行链路

这一篇按“发生一次消息请求”来理解 OpenClaw。

## 1. CLI 启动

入口有两层：

1. `openclaw.mjs`
   npm 包或源码 checkout 的外层启动器。它检查 Node 版本，尝试加载 `dist/entry.js` 或 `dist/entry.mjs`。
2. `src/entry.ts`
   TypeScript 源码入口。它处理 argv、profile、container、compile cache、Windows argv 标准化，然后导入 `src/cli/run-main.ts`。

关键文件：

- `openclaw.mjs`
- `src/entry.ts`
- `src/cli/run-main.ts`

## 2. CLI 路由到 Gateway 命令

`runCli()` 是 CLI 主路由。它会：

- 加载 `.env` 和 runtime env。
- 检查 Node runtime。
- 对 `openclaw gateway` 走快速路径，减少启动开销。
- 构建 Commander program。
- 注册内置命令和插件命令。
- 最后解析 argv。

当用户运行：

```bash
openclaw gateway --port 18789 --verbose
```

会进入：

- `src/cli/run-main.ts`
- `src/cli/gateway-cli/run.ts`

`src/cli/gateway-cli/run.ts` 负责：

- 读取配置。
- 解析 port、bind、auth、tailscale 等选项。
- 检查是否允许未配置启动。
- 解析 Gateway auth。
- 检查非 loopback 绑定是否有认证。
- 导入 `src/gateway/server.js` 并调用 `startGatewayServer()`。

## 3. Gateway server 启动

`src/gateway/server.ts` 是轻量 wrapper，它延迟加载 `server.impl.ts`。真正的大量启动逻辑在 `src/gateway/server.impl.ts`。

Gateway 启动阶段大致会做：

- 读取和物化 runtime config。
- 准备插件 metadata snapshot。
- 启动 HTTP / WebSocket server。
- 配置 Control UI 静态资源。
- 准备认证、设备配对、rate limit。
- 启动频道 manager。
- 启动 cron、runtime services、hooks、健康检查。
- 建立 Gateway event stream。

关键文件：

- `src/gateway/server.ts`
- `src/gateway/server.impl.ts`
- `src/gateway/server-startup.ts`
- `src/gateway/server-runtime-services.ts`
- `src/gateway/server-ws-runtime.ts`
- `src/gateway/server-channels.ts`

## 4. WebSocket 协议握手

所有 Control UI、CLI client、节点都通过 Gateway WebSocket 协议通信。协议基本形态：

```json
{ "type": "req", "id": "...", "method": "...", "params": {} }
{ "type": "res", "id": "...", "ok": true, "payload": {} }
{ "type": "event", "event": "...", "payload": {}, "seq": 1 }
```

连接第一步必须是 `connect`。连接前 Gateway 会发 `connect.challenge` nonce，客户端签名后发送 connect 参数。

实现位置：

- `src/gateway/client.ts`
- `src/gateway/protocol/schema/frames.ts`
- `src/gateway/protocol/index.ts`
- `src/gateway/server-ws-runtime.ts`

协议 schema 使用 TypeBox 描述，再用 AJV 编译 validator。这样 Control UI、CLI、节点和 Gateway 之间不会随便传 ad hoc JSON。

## 5. Control UI 如何通信

Control UI 是一个浏览器应用：

```text
ui/src/main.ts
  -> ui/src/ui/app.ts
  -> ui/src/ui/app-gateway.ts
  -> GatewayBrowserClient
  -> Gateway WebSocket
```

聊天相关逻辑：

- `ui/src/ui/app-chat.ts` 处理发送、停止、排队、slash command。
- `ui/src/ui/controllers/chat.ts` 封装 `chat.send`、`chat.history`、`chat.abort` 等 RPC。
- `ui/src/ui/app-gateway.ts` 接收 Gateway event，更新 UI state。

Control UI 不直接运行 Agent，也不直接读写 session 文件。它通过 Gateway RPC 操作。

## 6. 频道消息如何进入

频道运行由 `createChannelManager()` 管理。它负责：

- 根据配置找启用的频道和 account。
- 启动频道 runtime。
- 跟踪频道 account snapshot。
- 处理手动停止、重启、健康监控。
- 通过插件或内置频道把平台事件接入 Gateway。

实现位置：

- `src/gateway/server-channels.ts`
- `src/channels/plugins/index.ts`
- `src/channels/plugins/registry.ts`
- `extensions/*/`

当 Telegram、WhatsApp、Slack 等收到消息后，频道代码会把平台消息转成 OpenClaw 内部消息，再交给 Gateway 路由到某个 session。

## 7. 消息路由到 session

OpenClaw 用 session 管理上下文。默认行为：

- DM 默认进入共享 `main` session。
- 群聊、房间、频道通常隔离到不同 session。
- cron 每次可创建独立 session。
- 多用户场景推荐 `session.dmScope: "per-channel-peer"`。

实现位置：

- `docs/concepts/session.md`
- `src/config/sessions/*`
- `src/routing/*`
- `src/gateway/server-session-key.ts`
- `src/agents/command/session.ts`

Session 状态由 Gateway 拥有。transcript 写到本地 JSONL。

## 8. Agent run 执行

Agent 执行入口是 `agentCommand()`。

主流程：

1. 解析 session、agentId、workspace。
2. 解析模型 provider/model、thinking、verbose、trace。
3. 加载 skills snapshot。
4. 准备 auth profile。
5. 构造 run context。
6. 调用 `runAgentAttempt()`。
7. 根据 provider 决定走内嵌 Pi runtime 或 CLI backend。
8. 流式发出 assistant/tool/lifecycle 事件。
9. 写 session transcript。
10. 组装最终回复 payload。

关键文件：

- `src/agents/agent-command.ts`
- `src/agents/command/attempt-execution.ts`
- `src/agents/pi-embedded.ts`
- `src/agents/cli-runner.ts`
- `src/agents/skills.ts`
- `src/agents/workspace.ts`

`runAgentAttempt()` 里有一个重要分支：

- 如果 provider 是 CLI runtime，例如某些 `*-cli` backend，则调用 `runCliAgent()`。
- 否则调用 `runEmbeddedPiAgent()`。

## 9. 回复如何返回

回复路径取决于请求来源：

- Control UI 发起的 chat：Gateway 发 `chat` / `agent` events，前端流式渲染。
- 频道发起的消息：Gateway 通过频道 reply pipeline 把回复发回聊天平台。
- CLI 发起的 agent 命令：CLI 输出结果或等待 run 完成。

相关位置：

- `src/gateway/server-methods/*`
- `src/auto-reply/reply/*`
- `src/agents/command/delivery.runtime.ts`
- `ui/src/ui/app-gateway.ts`
- `ui/src/ui/controllers/chat.ts`

## 10. 插件如何参与

插件参与分两阶段：

### 控制平面

先读 metadata，不执行 runtime：

- `openclaw.plugin.json`
- package `openclaw` block
- manifest registry
- plugin metadata snapshot
- activation planner

用于：

- 判断插件是否存在、启用、禁用。
- 校验 `plugins.entries.<id>.config`。
- 判断频道、provider、CLI backend、setup choice 的归属。
- 给 Control UI 和 onboarding 提供 schema / hints。

### 运行平面

真正需要执行时加载插件 module，调用：

```ts
definePluginEntry({
  id: "...",
  register(api) {
    api.registerProvider(...)
    api.registerChannel(...)
    api.registerTool(...)
  }
})
```

相关位置：

- `docs/plugins/architecture.md`
- `docs/plugins/manifest.md`
- `docs/plugins/sdk-overview.md`
- `src/plugins/registry.ts`
- `src/plugins/discovery.ts`
- `src/plugins/activation-planner.ts`
- `src/plugin-sdk/plugin-entry.ts`
- `extensions/*/index.ts`

