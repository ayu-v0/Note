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
5. `05-openclaw-mjs-launcher.md`
   单独理解 `openclaw.mjs` 启动器：它只做 bootstrap，不承载主要业务逻辑。
6. `06-architecture-invariants.md`
   先记住项目的架构边界：core、plugins、Gateway、Agent、UI、apps 各自负责什么。
7. `07-data-and-contracts.md`
   认识贯穿全项目的数据合同：config、manifest、session、Gateway frame、tool schema、auth。
8. `08-subsystem-map.md`
   从目录和子系统角度建立全局地图，知道一个需求大概会落在哪些文件里。
9. `09-development-workflow.md`
   了解本仓库的开发、测试、格式化、文档和提交流程。
10. `10-learning-roadmap.md`
    给新手的分阶段学习路线和练习任务。

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

按仓库规则我先尝试运行了 `pnpm docs:list`，但当前 shell 找不到 `pnpm`。随后直接运行了等价脚本 `node scripts/docs-list.js`，并结合仓库内相关 Markdown 和源码入口整理这些笔记。

