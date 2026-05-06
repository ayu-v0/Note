# 04. 新手源码阅读路线

## 不要从最大文件开始

这个项目是大型 TypeScript monorepo。直接打开 `src/gateway/server.impl.ts` 或 `src/agents/agent-command.ts` 从头读，很容易迷路。建议按“用户行为”读。

## 路线 A：理解启动

目标：知道 `openclaw gateway` 怎么跑起来。

按顺序读：

1. `openclaw.mjs`
2. `src/entry.ts`
3. `src/cli/run-main.ts`
4. `src/cli/gateway-cli/run.ts`
5. `src/gateway/server.ts`
6. `src/gateway/server.impl.ts`

重点问题：

- 哪些逻辑是为了快速显示 help？
- 哪些模块是延迟 import 的？
- Gateway 启动前为什么要先解析 config 和 auth？
- 非 loopback bind 为什么要求认证？

## 路线 B：理解一条聊天消息

目标：知道用户发一句话后，代码怎么变成 AI 回复。

Control UI 入口：

1. `ui/src/ui/app-chat.ts`
2. `ui/src/ui/controllers/chat.ts`
3. `ui/src/ui/app-gateway.ts`
4. `src/gateway/protocol/schema/frames.ts`
5. `src/gateway/server-methods/*`
6. `src/agents/agent-command.ts`
7. `src/agents/command/attempt-execution.ts`

频道入口：

1. `docs/channels/index.md`
2. `src/gateway/server-channels.ts`
3. `src/channels/plugins/index.ts`
4. 选一个具体插件，比如 `extensions/line/` 或 `extensions/telegram` 相关文件
5. `src/agents/agent-command.ts`

重点问题：

- 消息如何绑定到 session？
- `chat.send` 为什么先返回 runId？
- assistant/tool/lifecycle 事件在哪里发出？
- transcript 在哪里写入？
- 最终回复由谁投递到频道？

## 路线 C：理解插件

目标：知道 OpenClaw 如何扩展 provider、频道、工具。

按顺序读：

1. `docs/plugins/architecture.md`
2. `docs/plugins/manifest.md`
3. `docs/plugins/sdk-overview.md`
4. `src/plugin-sdk/plugin-entry.ts`
5. `src/plugins/registry.ts`
6. `src/plugins/discovery.ts`
7. `src/plugins/activation-planner.ts`
8. 任意一个小插件的 `extensions/<id>/openclaw.plugin.json`
9. 同一个插件的 `extensions/<id>/index.ts`

重点问题：

- 哪些信息来自 manifest？
- 哪些行为必须等 runtime 加载后才知道？
- `register(api)` 注册了什么？
- 插件为什么不能直接 import core internals？
- capability 和 plugin 的区别是什么？

## 路线 D：理解配置

目标：知道 `openclaw.json` 如何被读取、校验和应用。

按顺序读：

1. `docs/gateway/configuration.md`
2. `src/config/types.openclaw.ts`
3. `src/config/paths.ts`
4. `src/config/io.ts`
5. `src/config/validation.ts`
6. `src/gateway/config-reload.ts`
7. `ui/src/ui/views/config.ts`

重点问题：

- 默认配置路径怎么确定？
- 为什么配置是 JSON5？
- 未知 key 为什么会阻止 Gateway 启动？
- Control UI 的表单 schema 从哪里来？
- last-known-good 如何避免坏配置卡死服务？

## 路线 E：理解 Control UI

目标：知道浏览器端如何和 Gateway 同步状态。

按顺序读：

1. `docs/web/control-ui.md`
2. `ui/src/main.ts`
3. `ui/src/ui/app.ts`
4. `ui/src/ui/app-gateway.ts`
5. `ui/src/ui/gateway.ts`
6. `ui/src/ui/controllers/*`
7. `ui/src/ui/views/*`

重点问题：

- 前端如何保存 Gateway URL 和 token？
- `hello-ok` 带了哪些初始 snapshot？
- UI 是如何处理 event stream 的？
- 哪些状态来自 `chat.history`，哪些来自实时事件？

## 常用搜索方式

这个仓库很大，建议用 `rg`：

```bash
rg "chat.send" src ui
rg "registerProvider" src extensions
rg "registerChannel" src extensions
rg "startGatewayServer" src
rg "ConnectParamsSchema" src
rg "sessionKey" src/gateway src/agents
```

## 读代码时的几个原则

### 先找边界，再看细节

例如你要看 Telegram 消息如何处理，不要先读所有频道代码。先确认边界：

- Gateway 频道管理：`src/gateway/server-channels.ts`
- 频道插件注册：`src/channels/plugins/*`
- Telegram 自己的插件或实现目录
- Agent 执行入口：`src/agents/agent-command.ts`

### 看到动态 import，要想到启动性能

项目里大量使用动态 import。很多不是随意写法，而是为了：

- help 命令不要加载整个 Gateway。
- Gateway 启动不要过早加载所有插件。
- 控制平面不要执行 runtime code。
- 热路径只读轻量 metadata。

### 区分控制平面和运行平面

控制平面：

- manifest
- schema
- config validation
- setup hints
- activation planning
- UI form metadata

运行平面：

- provider HTTP 请求
- channel listener
- tool execution
- Agent loop
- plugin runtime service

这个区别是理解 OpenClaw 架构的关键。

### 区分 core 和 plugin owner

Core 负责通用合同和编排，插件负责具体 provider、频道或功能行为。

例如：

- Core 不应该硬编码某个 provider 的私有策略。
- Channel plugin 应该拥有平台特有的消息发送、编辑、reaction 逻辑。
- Provider plugin 应该拥有厂商 auth、catalog、runtime hook。
- 多个插件共同需要的能力，应该抽象成 SDK capability。

## 推荐第一个实战任务

如果你想通过改代码学习，建议从低风险入口开始：

1. 给某个 Control UI 小视图加一个只读显示字段。
2. 给某个已有纯函数加测试。
3. 阅读并注释一个小插件的 manifest 和 `index.ts`。
4. 跟踪一个 `chat.history` 请求从 UI 到 Gateway method 的路径。

暂时不要一开始就改：

- Gateway 协议 schema。
- 插件 loader。
- session transcript 写入。
- provider stream wrapper。
- 启动和动态 import 边界。

这些区域影响面很大，适合建立基础后再动。

