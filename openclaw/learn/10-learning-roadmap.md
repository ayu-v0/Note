# 10. 新手学习路线

这一篇给你一条可执行路线。目标不是“背完整仓库”，而是逐步建立能独立定位问题的能力。

## 阶段 1：先建立心智模型

阅读顺序：

1. `.workplace/learn/01-project-overview.md`
2. `.workplace/learn/02-runtime-flow.md`
3. `.workplace/learn/06-architecture-invariants.md`
4. `docs/concepts/architecture.md`
5. `docs/concepts/agent.md`
6. `docs/concepts/session.md`

要能回答：

- OpenClaw 为什么需要 Gateway？
- Control UI 为什么不直接运行 agent？
- plugin manifest 和 plugin runtime 有什么区别？
- session 解决什么问题？
- provider、channel、tool 分别是什么？

## 阶段 2：跑通入口链路

目标：理解 `openclaw gateway` 怎么启动。

阅读：

1. `.workplace/learn/05-openclaw-mjs-launcher.md`
2. `openclaw.mjs`
3. `src/entry.ts`
4. `src/cli/run-main.ts`
5. `src/cli/gateway-cli/run.ts`
6. `src/gateway/server.ts`
7. `src/gateway/server.impl.ts`

练习：

```bash
rg "startGatewayServer" src
rg "runGatewayCommand" src/cli src/gateway
rg "OPENCLAW_DISABLE_CLI_STARTUP_HELP_FAST_PATH" .
```

要能回答：

- launcher 和 CLI 业务逻辑的分界在哪里？
- 为什么 help 有 fast path？
- Gateway 启动前为什么要先解析 auth/config？
- `server.ts` 为什么延迟加载 `server.impl.ts`？

## 阶段 3：跟一条聊天消息

目标：从 UI 发消息一路追到 agent run。

阅读：

1. `docs/web/control-ui.md`
2. `ui/src/ui/app-chat.ts`
3. `ui/src/ui/controllers/chat.ts`
4. `ui/src/ui/app-gateway.ts`
5. `ui/src/ui/gateway.ts`
6. `src/gateway/protocol/schema/frames.ts`
7. `src/gateway/server-methods/agent.ts`
8. `src/agents/agent-command.ts`
9. `src/agents/command/attempt-execution.ts`

练习：

```bash
rg "chat.send" ui/src src/gateway
rg "chat.history" ui/src src/gateway
rg "chat.abort" ui/src src/gateway
rg "agent.wait" ui/src src/gateway
```

要能回答：

- `chat.send` 为什么不是直接等最终文本？
- runId 在 UI 和 Gateway 中怎么使用？
- 哪些事件会更新聊天界面？
- transcript 什么时候写？

## 阶段 4：理解一个插件

目标：看懂一个插件从 manifest 到 runtime 注册。

建议先选简单插件，再看复杂插件。

阅读：

1. `docs/plugins/architecture.md`
2. `docs/plugins/manifest.md`
3. `docs/plugins/sdk-overview.md`
4. `src/plugin-sdk/plugin-entry.ts`
5. `src/plugins/registry.ts`
6. `extensions/telegram/openclaw.plugin.json` 或一个你感兴趣的 plugin manifest
7. 同一插件的 `index.ts`

练习：

```bash
rg "definePluginEntry" extensions/telegram extensions/openai extensions/brave
rg "registerProvider" extensions/openai src/plugins
rg "registerChannel" extensions/telegram src/plugins
rg "contracts" extensions/*/openclaw.plugin.json
```

要能回答：

- 哪些信息在 manifest 里？
- 哪些信息只能 runtime 注册后知道？
- `api.registerProvider` 和 `api.registerChannel` 写入哪里？
- activation hint 是什么，不是什么？

## 阶段 5：理解配置

目标：知道 `openclaw.json` 怎么从文本变成运行时行为。

阅读：

1. `docs/gateway/configuration.md`
2. `src/config/types.openclaw.ts`
3. `src/config/zod-schema.ts`
4. `src/config/io.ts`
5. `src/config/paths.ts`
6. `ui/src/ui/views/config.ts`
7. `ui/src/ui/controllers/config.ts`

练习：

```bash
rg "OpenClawSchema" src/config
rg "loadConfig" src/config src/gateway ui/src
rg "last-known-good" src docs
rg "configSchema" ui/src src/config src/plugins
```

要能回答：

- 为什么 config 是 strict validation？
- UI 的 config form schema 从哪来？
- plugin config key 为什么必须同步 manifest schema？
- 哪些 config reload 可以 hot apply，哪些需要 restart？

## 阶段 6：认识工具和安全边界

阅读：

1. `docs/tools/index.md`
2. `docs/tools/exec-approvals.md`
3. `docs/channels/pairing.md`
4. `src/agents/tools/common.ts`
5. `src/agents/tools/message-tool.ts`
6. `src/agents/tool-policy.ts`
7. `src/gateway/exec-approval-manager.ts`

练习：

```bash
rg "create.*Tool" src/agents/tools
rg "toolPolicy" src/agents src/config
rg "exec.approval" src ui
rg "dmPolicy" src docs extensions
```

要能回答：

- 工具是怎么暴露给模型的？
- 为什么 tool schema 要保守？
- exec approval 保护什么？
- DM pairing 和 Gateway device pairing 有什么区别？

## 阶段 7：了解验证体系

阅读：

1. `docs/reference/test.md`
2. `docs/ci.md`
3. `AGENTS.md` 的 Commands / Gates / Tests 部分
4. `scripts/test-projects.mjs`
5. `scripts/check-changed.mjs`

练习：

```bash
node scripts/docs-list.js
rg "changed:lanes" package.json scripts
rg "test-projects" package.json scripts
rg "run-vitest" package.json scripts
```

要能回答：

- 为什么不用 raw `vitest`？
- changed gate 怎么决定跑哪些 lane？
- 哪些验证适合本地，哪些适合 Testbox？
- docs-only 改动一般怎么验证？

## 推荐的第一个实战任务

从低风险任务开始：

1. 找一个 `src/agents/tools/*` 里的纯 helper，补一个小测试。
2. 给 Control UI 某个只读状态增加一个显示字段。
3. 读一个小插件的 manifest 和 `index.ts`，写一页本地笔记。
4. 跟踪一次 `chat.history` 请求，从 UI controller 到 Gateway method。

暂时不要一开始就改：

- Gateway protocol schema。
- plugin loader / registry。
- session transcript 写入。
- provider streaming wrapper。
- config migration。
- 动态 import 边界。

这些区域影响面大，适合你读完前面几阶段后再动。

