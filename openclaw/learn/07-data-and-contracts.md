# 07. 关键数据和合同

OpenClaw 很大，但很多代码都围绕少数“合同”转。理解这些合同，比记住某个文件细节更重要。

## Config: 用户期望的来源

默认配置是 JSON5，通常位于 OpenClaw state dir 下的 `openclaw.json`。源码里不要把这个写成绝对路径；路径解析由 `src/config/paths.ts` 管。

配置合同覆盖：

- agents
- gateway
- models/providers
- channels
- plugins
- tools
- cron
- sessions
- secrets
- auth / security

核心文件：

- `src/config/types.openclaw.ts`
- `src/config/io.ts`
- `src/config/validation.ts`
- `src/config/zod-schema.ts`
- `docs/gateway/configuration.md`

读法：

1. 先看 docs 里的配置说明。
2. 再看 TypeScript 类型。
3. 再看 zod schema。
4. 最后看 `io.ts` 怎么读、写、校验、恢复 last-known-good。

## Plugin manifest: 插件控制面合同

每个 native plugin 的根目录需要 `openclaw.plugin.json`。它是插件在不执行 runtime 代码时暴露给 core 的 metadata。

manifest 常见职责：

- plugin id、name、version、kind
- config schema
- providers / channels / command aliases / CLI backends 的 owner 信息
- setup/onboarding metadata
- activation hints
- contracts.tools
- model catalog 或 provider discovery metadata

关键规则：

- manifest 用于 discovery 和 validation，不等于 runtime behavior。
- runtime 行为仍由 `index.ts` 里的 `definePluginEntry({ register(api) { ... } })` 决定。
- 如果新加插件 config key，要同步更新 manifest 的 `configSchema`。
- `activation` 是加载规划 metadata，不是生命周期 hook。

核心文件：

- `docs/plugins/manifest.md`
- `docs/plugins/architecture.md`
- `src/plugins/manifest-types.ts`
- `src/plugins/discovery.ts`
- `src/plugins/activation-planner.ts`
- `src/plugin-sdk/plugin-entry.ts`

## Gateway protocol: 进程间通信合同

Gateway protocol 是 Control UI、CLI client、nodes、WebChat 和 Gateway 的共同语言。

基础 frame 形态：

```json
{ "type": "req", "id": "...", "method": "...", "params": {} }
{ "type": "res", "id": "...", "ok": true, "payload": {} }
{ "type": "event", "event": "...", "payload": {}, "seq": 1 }
```

连接第一步是 `connect`。服务端先发 `connect.challenge`，客户端带签名或 token 完成认证。

核心文件：

- `src/gateway/protocol/schema/frames.ts`
- `src/gateway/protocol/schema.ts`
- `src/gateway/protocol/index.ts`
- `src/gateway/client.ts`
- `ui/src/ui/gateway.ts`

读法：

1. 先看 `frames.ts`，理解 req/res/event。
2. 再看 schema 目录里按功能拆分的方法参数。
3. 再看 `index.ts` 如何编译 validator。
4. 最后看 UI 的 `GatewayBrowserClient` 如何发 request 和处理 event。

## Session: 多轮上下文合同

Session 决定“这句话属于哪段对话”。它影响上下文、transcript、工具可见性、delivery target。

常见 session 类型：

- main session
- DM session
- group session
- cron session
- hook session
- node session
- subagent / ACP session

核心概念：

- Gateway 根据来源、agentId、channel、peer、thread 等解析 session key。
- transcript 写成本地 JSONL。
- 多 agent 场景下 session key 通常带 agent 归属。
- 群聊和 DM 的隔离策略由配置和 channel routing 决定。

核心文件：

- `docs/concepts/session.md`
- `src/routing/*`
- `src/config/sessions/*`
- `src/gateway/server-session-key.ts`
- `src/agents/command/session.ts`
- `src/sessions/*`

## Agent event: 流式执行合同

Agent run 不是只返回一个字符串。执行过程中会产生多类事件：

- lifecycle：start、finish、error、abort、timeout
- assistant text delta 或最终文本
- tool call / tool result
- approval requested / resolved
- compaction、fallback、usage 等状态

Gateway 负责把这些事件转成前端和 channel 能消费的形式。

核心文件：

- `src/infra/agent-events.ts`
- `src/agents/internal-events.ts`
- `src/gateway/server-methods/agent.ts`
- `src/gateway/server-methods/agent-job.ts`
- `ui/src/ui/app-gateway.ts`
- `ui/src/ui/app-tool-stream.ts`

## Tool schema: 模型可调用能力合同

Agent 工具要给模型暴露参数 schema。这里要非常保守，因为不同 provider 对 JSON Schema 支持不完全一样。

常见工具：

- `message`：跨 channel 发消息、反应、fetch、poll 等。
- `web_search` / `web_fetch`
- `image_generate` / `video_generate` / `music_generate`
- `pdf`
- `nodes`
- `cron`
- `sessions_*`
- `update_plan`

核心文件：

- `docs/tools/index.md`
- `src/agents/tools/common.ts`
- `src/agents/tools/message-tool.ts`
- `src/agents/tools/web-search.ts`
- `src/agents/tools/web-fetch.ts`
- `src/agents/tools/image-generate-tool.ts`
- `src/agents/tool-policy.ts`

规则：

- schema 尽量用简单 object、string enum、array。
- 避免 provider 不支持的复杂 `anyOf`。
- 工具是否可用由 config、agent policy、plugin registration 和运行上下文共同决定。

## Auth 和 pairing: 安全合同

OpenClaw 是能操作本机、聊天账号、浏览器和远程节点的系统，所以认证边界非常重要。

主要机制：

- Gateway token/password
- trusted proxy / Tailscale identity
- device pairing
- DM pairing / allowlist
- exec approval
- plugin approval
- SecretRef / credentials
- sandbox policy

核心文件：

- `docs/channels/pairing.md`
- `docs/web/control-ui.md`
- `docs/tools/exec-approvals.md`
- `src/gateway/auth.ts`
- `src/gateway/device-auth.ts`
- `src/gateway/connection-auth.ts`
- `src/gateway/exec-approval-manager.ts`
- `src/secrets/*`

读安全相关代码时的原则：

- 先确认 threat model 和默认拒绝策略。
- 不要把 UI 便利性放在认证边界前面。
- 不要在日志、docs、tests 里写真实凭据。
- 修 channel DM 行为时先读对应 channel docs 和 pairing docs。

## Media: 引用和存储合同

图片、音频、视频、文件不会简单地作为大字符串在所有层里传来传去。OpenClaw 会使用 managed media、attachment refs、provider-specific payload 等机制。

核心文件：

- `docs/tools/media-overview.md`
- `src/media/*`
- `src/media-generation/*`
- `src/media-understanding/*`
- `src/gateway/chat-attachments.ts`
- `src/plugin-sdk/media-*`

读法：

- 如果是用户上传到聊天：从 Gateway attachment pipeline 看。
- 如果是模型生成媒体：从 media-generation tool 和 provider plugin 看。
- 如果是 channel 发送附件：从具体 channel plugin 看。

