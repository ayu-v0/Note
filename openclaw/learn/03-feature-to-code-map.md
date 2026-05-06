# 03. 功能到源码映射

这一篇按功能查实现位置。

## 总表

| 功能 | 用户看到什么 | 实现原理 | 主要源码 |
| --- | --- | --- | --- |
| CLI 启动 | `openclaw ...` 命令 | 外层 launcher 检查 Node 和 dist，再进入 TypeScript/构建入口；CLI 根据 argv 延迟注册命令 | `openclaw.mjs`, `src/entry.ts`, `src/cli/run-main.ts`, `src/cli/program.ts` |
| Gateway | `openclaw gateway` 常驻服务 | CLI 读取配置和认证选项，启动 HTTP/WS server，挂载频道、插件、服务 | `src/cli/gateway-cli/run.ts`, `src/gateway/server.ts`, `src/gateway/server.impl.ts` |
| WebSocket 协议 | Control UI、CLI、节点连接 Gateway | TypeBox 定义 frame 和方法 schema，AJV 校验，req/res/event 三类帧通信 | `src/gateway/protocol/schema/frames.ts`, `src/gateway/protocol/index.ts`, `src/gateway/client.ts` |
| Control UI | 浏览器控制台和聊天页 | Vite + Lit 单页应用，静态资源由 Gateway 服务，前端通过 Gateway WS RPC 操作 | `ui/src/main.ts`, `ui/src/ui/app.ts`, `ui/src/ui/app-gateway.ts`, `ui/src/ui/views/*` |
| Chat | 浏览器里发消息、看流式回复、停止 run | `chat.send` 非阻塞返回 runId，后续通过事件流更新 UI；停止走 `chat.abort` | `ui/src/ui/app-chat.ts`, `ui/src/ui/controllers/chat.ts`, `src/gateway/server-methods/*` |
| 频道 | WhatsApp、Telegram、Slack 等消息入口 | 每个频道适配平台协议；Gateway 通过 channel manager 启停频道 account；消息转成内部 session 输入 | `src/gateway/server-channels.ts`, `src/channels/plugins/*`, `extensions/*` |
| Agent 执行 | AI 回复、调用工具、流式输出 | `agentCommand` 解析会话、模型、skills、workspace，调用 Pi runtime 或 CLI backend | `src/agents/agent-command.ts`, `src/agents/command/attempt-execution.ts`, `src/agents/pi-embedded.ts`, `src/agents/cli-runner.ts` |
| Session | 多轮上下文、群聊隔离、重置 | Gateway 维护 session store 和 transcript JSONL，按来源路由 sessionKey | `docs/concepts/session.md`, `src/config/sessions/*`, `src/routing/*` |
| Skills | Agent 可加载的技能说明 | 从 workspace、个人目录、managed、本地 bundled 等位置加载技能，注入 prompt/env | `docs/concepts/agent.md`, `src/agents/skills.ts`, `src/agents/bootstrap-files.ts` |
| 插件 | provider、频道、工具、语音、搜索扩展 | manifest-first 发现与配置验证；runtime 加载后用 `register(api)` 注册 capability | `src/plugins/*`, `src/plugin-sdk/*`, `packages/plugin-sdk/src/*`, `extensions/*` |
| 配置 | `~/.openclaw/openclaw.json` 和 Control UI Config tab | JSON5 解析、env/include 处理、严格 schema 校验、last-known-good 恢复 | `src/config/io.ts`, `src/config/types.openclaw.ts`, `src/config/validation.ts`, `docs/gateway/configuration.md` |
| 安全 | token/password、设备配对、DM pairing | Gateway auth + device identity + challenge 签名；频道侧 allowlist/pairing | `src/gateway/auth.ts`, `src/gateway/device-auth.ts`, `src/gateway/connection-auth.ts`, `docs/web/control-ui.md` |
| Cron | 定时任务 | Gateway cron service 管理 job、运行记录和投递方式 | `src/gateway/server-cron.ts`, `src/cron/*`, `ui/src/ui/views/cron.ts` |
| Exec approvals | 工具执行审批 | Gateway 管理 approval 请求、Control UI 展示/处理，节点也有独立 approval 设置 | `src/gateway/exec-approval-manager.ts`, `ui/src/ui/controllers/exec-approval.ts`, `ui/src/ui/views/exec-approval.ts` |
| Nodes | iOS/Android/macOS/headless 节点 | 节点用 role `node` 接入同一 WS，声明 caps/commands，由 Gateway 调度命令 | `src/gateway/protocol/schema/nodes.ts`, `src/gateway/server-node-session-runtime.ts`, `ui/src/ui/views/nodes.ts` |
| Media | 图片、音频、视频、文件 | Gateway 存储 managed media，协议和 channel reply pipeline 传递引用或平台附件 | `src/media-*`, `src/gateway/chat-attachments.ts`, `src/plugin-sdk/media-*` |
| 模型 provider | OpenAI、Anthropic、Google、本地兼容接口等 | provider 插件注册 text inference、media、speech、image/video generation 等 capability | `extensions/openai`, `extensions/anthropic`, `extensions/google`, `src/plugins/provider-*`, `src/plugin-sdk/provider-*` |

## 入口层

### `openclaw.mjs`

职责：

- 检查 Node 版本。
- 判断是否是源码 checkout。
- 对 help 做快速路径。
- 尝试导入构建产物 `dist/entry.js` / `dist/entry.mjs`。
- 如果缺少 dist，提示需要 build 或安装发布包。

### `src/entry.ts`

职责：

- 标准化 Windows argv。
- 处理 profile/container。
- 处理 compile cache。
- 调用 `runCli()`。

### `src/cli/run-main.ts`

职责：

- CLI 总调度。
- 处理 dotenv、proxy、runtime guard。
- 对 `openclaw gateway` 做 fast path。
- 注册核心命令和插件命令。

## Gateway 层

### `src/cli/gateway-cli/run.ts`

职责：

- 解析 `--port`、`--bind`、`--auth`、`--token`、`--tailscale` 等。
- 读取配置。
- 校验启动条件。
- 解析 Gateway 认证。
- 调用 `startGatewayServer()`。

新手重点看：

- `addGatewayRunCommand()`
- `runGatewayCommand()`
- `resolveGatewayAuth(...)`
- `runGatewayLoop(...)`

### `src/gateway/server.ts`

职责：

- 轻量导出。
- 延迟导入 `server.impl.ts`，减少启动或 help 场景的加载成本。

### `src/gateway/server.impl.ts`

职责：

- Gateway 的实际启动和生命周期。
- 整合配置、插件、频道、HTTP、WS、cron、runtime services、health、Control UI。

这是大文件，新手不建议从头读。先按关键词定位：

- `startGatewayServer`
- `attachGatewayWsHandlers`
- `createChannelManager`
- `prepareGatewayPluginBootstrap`
- `buildGatewayCronService`

## 协议层

### `src/gateway/protocol/schema/frames.ts`

定义最底层 wire frame：

- `ConnectParamsSchema`
- `HelloOkSchema`
- `RequestFrameSchema`
- `ResponseFrameSchema`
- `EventFrameSchema`
- `GatewayFrameSchema`

### `src/gateway/protocol/index.ts`

职责：

- 汇总所有 schema。
- 用 AJV 编译 validator。
- 导出协议类型和验证函数。

## Agent 层

### `src/agents/agent-command.ts`

职责：

- 处理一次 Agent 命令。
- 解析 session、workspace、模型、thinking、skills、auth profile。
- 做 fallback、override、事件注册。
- 最后调用 attempt execution。

### `src/agents/command/attempt-execution.ts`

职责：

- 决定用 CLI backend 还是 embedded Pi runtime。
- 调用 `runCliAgent()` 或 `runEmbeddedPiAgent()`。
- 写 transcript。
- 整理结果和 usage。

新手可以重点理解这个分支：

```text
provider 是 CLI runtime -> runCliAgent()
否则 -> runEmbeddedPiAgent()
```

## 插件层

### `openclaw.plugin.json`

插件 metadata。用于不执行插件代码时完成：

- id/name/version。
- config schema。
- provider/channel/cliBackend ownership。
- setup/onboarding hints。
- activation hints。
- capability contracts。

### `src/plugin-sdk/plugin-entry.ts`

插件作者使用的入口类型和 `definePluginEntry`。

### `src/plugins/registry.ts`

核心 registry。插件 runtime 加载后通过 `api.registerProvider(...)`、`api.registerChannel(...)`、`api.registerTool(...)` 等方法把能力注册到这里。

## UI 层

### `ui/src/main.ts`

前端入口，导入 CSS 和 `ui/app.ts`，生产环境注册 service worker。

### `ui/src/ui/app-gateway.ts`

Control UI 的 Gateway 连接和事件协调层。它维护连接状态、hello 信息、session、health、agents、exec approvals、chat events 等。

### `ui/src/ui/app-chat.ts`

聊天交互逻辑：

- 判断是否忙。
- 发送消息。
- 停止 run。
- 本地排队。
- reset/new 命令确认。
- steer pending run。

## 配置层

### `src/config/types.openclaw.ts`

总配置类型 `OpenClawConfig`。它把 agents、channels、plugins、gateway、models、tools、cron、session、secrets 等配置拼成一个整体。

### `src/config/io.ts`

配置读写核心：

- JSON5 解析。
- include 和 env substitution。
- schema validation。
- runtime snapshot。
- last-known-good 恢复。
- 写入前安全检查。

### `src/config/paths.ts`

状态目录和配置路径解析：

- 默认 state dir：`~/.openclaw`
- 默认 config：`~/.openclaw/openclaw.json`
- 支持 `OPENCLAW_STATE_DIR`
- 支持 `OPENCLAW_CONFIG_PATH`

