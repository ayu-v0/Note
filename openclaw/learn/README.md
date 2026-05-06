# OpenClaw 新手学习目录

这组笔记面向刚接触项目的人，目标不是覆盖每一行源码，而是先建立可用的心智模型：OpenClaw 做什么、一次消息如何流动、每个功能大致由哪些模块实现。

## 建议阅读顺序

1. `01-project-overview.md`
   先看项目是什么、能做什么、运行时由哪些大块组成。
2. `02-runtime-flow.md`
   再看一次用户消息从聊天软件或 Control UI 进入 Gateway，到 Agent 执行，再返回回复的完整链路。
3. `03-feature-to-code-map.md`
   按功能查源码位置。适合你想知道“这个按钮/命令/频道/插件在哪里实现”时使用。
4. `04-source-reading-guide.md`
   给新手的源码阅读路线和调试切入点。

## 资料来源

本目录内容基于仓库内文档与源码整理，主要参考：

- `README.md`
- `docs/index.md`
- `docs/concepts/architecture.md`
- `docs/concepts/agent.md`
- `docs/concepts/agent-loop.md`
- `docs/concepts/session.md`
- `docs/concepts/features.md`
- `docs/channels/index.md`
- `docs/web/control-ui.md`
- `docs/plugins/architecture.md`
- `docs/plugins/sdk-overview.md`
- `docs/plugins/manifest.md`
- `docs/gateway/configuration.md`
- `src/entry.ts`
- `src/cli/run-main.ts`
- `src/cli/gateway-cli/run.ts`
- `src/gateway/server.ts`
- `src/gateway/server.impl.ts`
- `src/gateway/client.ts`
- `src/gateway/protocol/index.ts`
- `src/gateway/protocol/schema/frames.ts`
- `src/gateway/server-channels.ts`
- `src/agents/agent-command.ts`
- `src/agents/command/attempt-execution.ts`
- `src/plugins/registry.ts`
- `src/plugin-sdk/plugin-entry.ts`
- `ui/src/main.ts`
- `ui/src/ui/app-gateway.ts`
- `ui/src/ui/app-chat.ts`

## 当前限制

按仓库规则我先尝试运行了 `pnpm docs:list`，但当前 shell 找不到 `pnpm`，所以没有拿到自动文档索引。这里改为直接读取仓库内相关 Markdown 和源码入口。

