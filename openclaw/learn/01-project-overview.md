# 01. 项目总览

## 一句话理解

OpenClaw 是一个自托管的个人 AI 助手网关。你在自己的机器或服务器上运行一个 Gateway，它连接 WhatsApp、Telegram、Slack、Discord、Signal、WebChat、移动节点等入口，把收到的消息路由给 AI Agent，再把回复发回原来的聊天界面或浏览器 Control UI。

换成程序视角：

```text
聊天软件 / Web UI / CLI / 移动节点
        |
        v
Gateway: 认证、协议、路由、频道连接、会话、配置、插件
        |
        v
Agent runtime: 模型、工具、技能、上下文、会话记录
        |
        v
频道回复 / UI 流式事件 / CLI 输出
```

## 主要功能

### 多频道聊天入口

OpenClaw 支持很多聊天软件。内置或插件频道包括 WhatsApp、Telegram、Slack、Discord、Google Chat、Signal、iMessage、BlueBubbles、Matrix、LINE、Mattermost、Microsoft Teams、Nostr、Twitch、Zalo、WebChat 等。

实现思路：

- 每个频道负责和对应平台 API 或协议通信。
- Gateway 统一管理频道生命周期。
- 收到消息后转成 OpenClaw 内部的会话和 Agent 输入。
- 发回复时再由频道适配器转回平台格式。

相关位置：

- `docs/channels/index.md`
- `src/channels/plugins/index.ts`
- `src/gateway/server-channels.ts`
- `extensions/*/`

### Gateway 控制平面

Gateway 是长时间运行的核心进程。它负责：

- 启动 HTTP 和 WebSocket 服务。
- 提供 Control UI 静态资源。
- 维护 WebSocket 协议。
- 管理认证、设备配对、会话、频道状态、定时任务、日志、插件。
- 把用户请求交给 Agent runtime。

相关位置：

- `src/cli/gateway-cli/run.ts`
- `src/gateway/server.ts`
- `src/gateway/server.impl.ts`
- `src/gateway/protocol/index.ts`
- `src/gateway/protocol/schema/frames.ts`

### Agent runtime

Agent runtime 是实际“思考和执行”的部分。它会：

- 解析模型和 provider。
- 准备工作区和 bootstrap 文件。
- 加载 skills。
- 组装系统提示词和上下文。
- 调用内嵌 Pi runtime 或外部 CLI backend。
- 执行工具。
- 写入 session transcript。
- 流式发出 assistant/tool/lifecycle 事件。

相关位置：

- `docs/concepts/agent.md`
- `docs/concepts/agent-loop.md`
- `src/agents/agent-command.ts`
- `src/agents/command/attempt-execution.ts`
- `src/agents/pi-embedded.ts`
- `src/agents/cli-runner.ts`

### Control UI

Control UI 是浏览器里的控制台。它是 Vite + Lit 单页应用，由 Gateway 提供静态文件，直接用 WebSocket 和 Gateway 通信。

它可以做：

- 聊天和流式查看工具输出。
- 查看频道、节点、会话、健康状态。
- 编辑配置。
- 管理 cron、skills、exec approvals。
- 查看日志、更新状态。

相关位置：

- `docs/web/control-ui.md`
- `ui/src/main.ts`
- `ui/src/ui/app.ts`
- `ui/src/ui/app-gateway.ts`
- `ui/src/ui/app-chat.ts`
- `ui/src/ui/views/*`
- `src/gateway/control-ui.ts`

### 插件系统

插件是 OpenClaw 的扩展边界。模型 provider、频道、语音、图片生成、搜索、工具、HTTP route、CLI 命令等都可以由插件注册。

核心原则：

- 插件身份和配置先通过 `openclaw.plugin.json` 描述。
- Gateway 可以在不执行插件 runtime 的情况下做发现和验证。
- 真正运行时，插件通过 `register(api)` 注册能力。
- Core 通过通用 capability 合同调用插件，而不是硬编码某个厂商或频道。

相关位置：

- `docs/plugins/architecture.md`
- `docs/plugins/sdk-overview.md`
- `docs/plugins/manifest.md`
- `src/plugins/registry.ts`
- `src/plugin-sdk/plugin-entry.ts`
- `packages/plugin-sdk/src/*`
- `extensions/*/openclaw.plugin.json`
- `extensions/*/index.ts`

### 配置与安全

配置默认在 `~/.openclaw/openclaw.json`，格式是 JSON5。Gateway 会严格校验配置，未知 key 或类型错误可能阻止启动。

重要安全机制：

- Gateway WebSocket 认证：token、password、trusted-proxy、Tailscale identity 等。
- 设备配对：新浏览器、移动节点、远程客户端需要授权。
- DM pairing 和 allowlist：陌生私聊默认不会直接进入 Agent。
- 非 main session 可使用 sandbox。
- SecretRef 和 credentials 文件避免明文泄漏。

相关位置：

- `docs/gateway/configuration.md`
- `docs/concepts/session.md`
- `src/config/types.openclaw.ts`
- `src/config/io.ts`
- `src/config/paths.ts`
- `src/gateway/auth.ts`
- `src/gateway/device-auth.ts`

## 代码组织速览

| 路径 | 作用 |
| --- | --- |
| `src/` | 核心 TypeScript 代码 |
| `src/cli/` | CLI 命令、启动路由、命令注册 |
| `src/gateway/` | Gateway server、WebSocket、HTTP、Control UI、认证、节点、事件 |
| `src/gateway/protocol/` | Gateway wire protocol 的 TypeBox schema、类型和 AJV validator |
| `src/agents/` | Agent 命令、模型选择、工具、session、workspace、Pi/CLI runtime |
| `src/channels/` | Core 频道抽象和频道插件注册接口 |
| `src/plugins/` | 插件发现、manifest、registry、加载、能力注册 |
| `src/plugin-sdk/` | 插件作者使用的 SDK facade |
| `packages/plugin-sdk/` | 发布包形式的 SDK re-export |
| `extensions/` | Bundled plugins，包括 provider、频道、工具、语音等 |
| `ui/` | Control UI 前端 |
| `docs/` | 官方文档源 |
| `scripts/` | 构建、测试、发布、检查脚本 |

