# 08. 子系统地图

这一篇按目录和子系统查找入口。你不需要一次读完，遇到问题时回来查。

## 顶层目录

| 路径 | 作用 |
| --- | --- |
| `src/` | 核心 TypeScript 源码 |
| `ui/` | Control UI，Vite + Lit 前端 |
| `extensions/` | bundled plugins，内部目录名是 extensions，对外称 plugins |
| `packages/` | 发布用 SDK 和 plugin package contract |
| `apps/` | macOS、iOS、Android 和共享 app 代码 |
| `docs/` | 官方文档源 |
| `scripts/` | 构建、测试、生成、发布、检查脚本 |
| `test/` | 测试配置、helpers、fixtures |
| `config/` | lint、format、tsconfig、工具配置 |
| `security/` | 安全扫描、模型和规则 |
| `skills/` | bundled 或项目内 skill 资源 |
| `.github/` | GitHub Actions、labeler、CI 配置 |

## `src/` 里的主干

| 路径 | 关注点 |
| --- | --- |
| `src/entry.ts` | TypeScript CLI 入口 |
| `src/cli/` | Commander 命令注册和 CLI 路由 |
| `src/commands/` | 具体 CLI 命令实现 |
| `src/gateway/` | Gateway server、WS、HTTP、methods、auth、channels、nodes |
| `src/gateway/protocol/` | Gateway wire protocol schema 和 validator |
| `src/agents/` | agent run、tools、skills、workspace、runtime、transcript |
| `src/plugins/` | plugin discovery、manifest、registry、activation |
| `src/plugin-sdk/` | 给 plugin 作者使用的 public facade |
| `src/channels/` | channel 抽象、contracts、plugin bridge |
| `src/config/` | config 类型、schema、读写、热更新、兼容 |
| `src/routing/` | channel/session/agent 路由 |
| `src/cron/` | scheduled jobs |
| `src/tasks/` | background task tracking |
| `src/hooks/` | hooks 机制 |
| `src/acp/` | ACP agent bridge |
| `src/mcp/` | MCP server/client 相关能力 |
| `src/node-host/` | headless node host |
| `src/secrets/` | SecretRef 和 credentials |
| `src/security/` | 安全检查和策略 |
| `src/terminal/` | CLI 表格和终端输出 |
| `src/tui/` | Terminal UI |

## `extensions/` 的常见类型

| 类型 | 例子 | 主要职责 |
| --- | --- | --- |
| Model provider | `extensions/openai`, `extensions/anthropic`, `extensions/google`, `extensions/xai` | 注册 LLM、media、embedding、search 等 provider capability |
| Channel | `extensions/telegram`, `extensions/discord`, `extensions/slack`, `extensions/whatsapp` | 接聊天平台，收消息、发回复、处理平台事件 |
| Search / fetch | `extensions/brave`, `extensions/tavily`, `extensions/firecrawl`, `extensions/searxng` | 给 web_search / web_fetch 提供后端 |
| Media | `extensions/fal`, `extensions/minimax`, `extensions/comfy`, `extensions/runway` | 图片、视频、音乐、语音生成 |
| Coding harness | `extensions/codex`, `extensions/acpx`, `extensions/opencode` | 外部编码 agent / ACP backend |
| Diagnostics | `extensions/diagnostics-otel`, `extensions/diagnostics-prometheus` | 观测和指标 |
| QA | `extensions/qa-channel`, `extensions/qa-lab`, `extensions/qa-matrix` | 自动化测试和场景验证 |

一个插件通常至少看两个文件：

- `extensions/<id>/openclaw.plugin.json`
- `extensions/<id>/index.ts`

复杂插件还会有：

- `src/`
- `api.ts`
- `runtime-api.ts`
- setup / docs / tests

## `ui/` 的主干

| 路径 | 关注点 |
| --- | --- |
| `ui/src/main.ts` | 前端入口 |
| `ui/src/ui/app.ts` | 根 Lit component 和全局状态 |
| `ui/src/ui/gateway.ts` | GatewayBrowserClient |
| `ui/src/ui/app-gateway.ts` | 连接、hello、snapshot、event 处理 |
| `ui/src/ui/app-chat.ts` | 聊天交互 |
| `ui/src/ui/controllers/*` | RPC/controller 层 |
| `ui/src/ui/views/*` | 页面视图 |
| `ui/src/i18n/*` | 国际化 |
| `ui/public/` | 静态资源 |

UI 读法：

1. 先看 `app.ts` 的 state。
2. 再看 `app-gateway.ts` 如何应用 Gateway snapshot 和 events。
3. 具体页面再进 `views/*`。
4. 发送请求时找 `controllers/*`。

## `apps/` 的主干

| 路径 | 关注点 |
| --- | --- |
| `apps/macos/` | macOS app、Gateway 管理、平台集成 |
| `apps/ios/` | iOS node app |
| `apps/android/` | Android node app |
| `apps/shared/` | Apple 平台共享模型和协议 |
| `apps/macos-mlx-tts/` | macOS 本地 TTS sidecar |
| `apps/swabble/` | 独立 app/实验面 |

注意：移动端和 macOS 相关修改通常要考虑真实设备、签名、协议 codegen、版本同步。

## `docs/` 的主干

| 路径 | 何时读 |
| --- | --- |
| `docs/start/` | 安装、上手、onboarding |
| `docs/concepts/` | 架构、agent、session、features |
| `docs/cli/` | CLI 命令参考 |
| `docs/gateway/` | Gateway 配置和运行 |
| `docs/web/` | Control UI、WebChat、TUI |
| `docs/channels/` | 各聊天平台 |
| `docs/plugins/` | plugin 架构、manifest、SDK |
| `docs/tools/` | agent tools、skills、browser、media、search |
| `docs/automation/` | cron、tasks、hooks、standing orders |
| `docs/security/` | threat model、安全响应 |
| `docs/reference/` | 测试、release、token、templates |

本仓库有 `docs:list` 索引脚本。当前环境找不到 `pnpm` 时，可以直接运行：

```bash
node scripts/docs-list.js
```

## `scripts/` 的常见类型

| 类型 | 例子 |
| --- | --- |
| 构建 | `scripts/build-all.mjs`, `scripts/tsdown-build.mjs`, `scripts/ui.js` |
| 测试包装 | `scripts/test-projects.mjs`, `scripts/run-vitest.mjs` |
| changed gate | `scripts/check-changed.mjs`, `scripts/changed-lanes.mjs` |
| 生成校验 | `scripts/generate-*.ts`, `scripts/*-check*.mjs` |
| plugin inventory | `scripts/generate-plugin-inventory-doc.mjs`, `scripts/sync-plugin-versions.ts` |
| release | `scripts/release-check.ts`, `scripts/full-release-validation-at-sha.mjs` |
| docker/e2e | `scripts/e2e/*`, `scripts/test-docker-all.mjs` |
| GitHub/Testbox | `scripts/blacksmith-testbox-runner.mjs`, `scripts/github/*` |

不要把 `scripts/` 当成杂物箱。很多脚本是 CI 和 release 合同的一部分。

## 查入口的 rg 模板

```bash
rg "chat.send" src ui
rg "agent.wait" src/gateway src/agents ui
rg "registerProvider" src extensions
rg "registerChannel" src extensions
rg "openclaw.plugin.json" extensions
rg "startGatewayServer" src/gateway
rg "ConnectParamsSchema" src/gateway/protocol
rg "resolve.*Session" src/gateway src/routing src/agents
rg "GatewayBrowserClient" ui/src
```

## scoped `AGENTS.md`

仓库有多个 scoped `AGENTS.md`。进入这些子树工作前要读对应文件：

- `docs/AGENTS.md`
- `extensions/AGENTS.md`
- `src/gateway/AGENTS.md`
- `src/gateway/protocol/AGENTS.md`
- `src/plugin-sdk/AGENTS.md`
- `src/plugins/AGENTS.md`
- `src/channels/AGENTS.md`
- `src/agents/AGENTS.md`
- `ui/AGENTS.md`
- `scripts/AGENTS.md`
- `test/helpers/AGENTS.md`

这不是形式主义。很多边界规则只在 scoped guide 里写清楚。

