# 09. 开发和验证流程

这一篇讲你真正开始改代码时该怎么做。

## 环境基础

仓库要求 Node 22+。`openclaw.mjs` 自己最低检查 Node 22.12+，但实际开发应以仓库脚本和 package metadata 为准。

包管理器以 `pnpm` 为主，同时要注意 Node 和 Bun 路径都要保持可用。当前这个 shell 里 `pnpm` 不在 PATH，所以本次学习笔记只运行了 `node scripts/docs-list.js`，没有跑 `pnpm` 包装命令。

## 常用命令

| 目标 | 命令 |
| --- | --- |
| 安装依赖 | `pnpm install` |
| 运行 CLI | `pnpm openclaw ...` |
| dev 入口 | `pnpm dev` |
| 启动 Gateway | `pnpm gateway:dev` 或 `pnpm openclaw gateway` |
| 构建 | `pnpm build` |
| changed gate | `pnpm check:changed` |
| 查看 changed lanes | `pnpm changed:lanes --json` |
| 测试 | `pnpm test` |
| 针对文件测试 | `pnpm test <path-or-filter>` |
| extension 测试 | `pnpm test:extensions` 或 `pnpm test extensions/<id>` |
| 类型检查 | `pnpm tsgo*` |
| 格式检查 | `pnpm format:check` |
| 格式化 | `pnpm format` |
| docs 索引 | `pnpm docs:list` 或 `node scripts/docs-list.js` |

规则：不要直接跑 raw `vitest`，用仓库包装命令。

## 修改前的标准动作

1. 看根 `AGENTS.md`。
2. 如果要进子目录，读对应 scoped `AGENTS.md`。
3. 跑或读 `docs:list`，只读相关 docs。
4. 用 `rg` 找入口，不要盲读大文件。
5. 确认 owner：core、plugin、UI、app、docs、script。
6. 确认这次改动是否触碰 public contract。

public contract 包括：

- Gateway protocol schema
- plugin SDK exports
- config schema
- plugin manifest schema
- docs URL / CLI command behavior
- channel message behavior
- provider tool schema
- app protocol models

## 代码修改策略

小改动优先在 owner 模块修。

例子：

- Telegram 行为问题：先看 `extensions/telegram`，不是先改 core。
- Config UI 显示问题：先看 `ui/src/ui/views/config*` 和 config controller。
- Gateway RPC 问题：看 `src/gateway/server-methods/*` 和 protocol schema。
- Agent tool 问题：看 `src/agents/tools/*` 和 tool policy。
- Provider 模型行为：看对应 provider plugin。

只有多个 owner 共同需要时，才抽通用 seam。

## 测试策略

验证要和风险匹配。

| 改动 | 推荐验证 |
| --- | --- |
| 单个纯函数 | 对应 colocated unit test |
| Gateway method | 对应 `src/gateway/server-methods/*.test.ts` 或 gateway config |
| protocol schema | protocol tests + generated/codegen 检查 |
| UI 状态处理 | UI node/browser test + 手动或截图验证 |
| plugin manifest/config | plugin tests + config validation |
| channel runtime | owner plugin test，必要时 live/e2e |
| SDK export | plugin SDK API/check |
| build/runtime boundary | `pnpm build`，检查动态 import 影响 |

本仓库规则里，广泛的 `pnpm check`、全量 `pnpm test`、Docker/E2E/live/package build 更适合 Testbox。窄范围本地验证用于开发循环。

## changed gate 怎么想

`pnpm check:changed` 会根据 diff 选择 lane。新手只需要先记住：

- core prod 改动：core typecheck + core tests。
- core tests 改动：core test typecheck/tests。
- extension prod 改动：extension prod typecheck + extension tests。
- extension tests 改动：extension test typecheck/tests。
- public SDK/plugin contract 改动：还要覆盖 extension prod/test。
- root/config 不明确改动：可能扩大到 all lanes。

如果 changed gate 扩得很大，不要硬在本地跑完；按仓库规则考虑 Testbox。

## 格式化和 lint

本仓库用 `oxfmt`，不是 Prettier。

常见命令：

```bash
pnpm format:check
pnpm format
pnpm exec oxfmt --check --threads=1 <files...>
pnpm exec oxfmt --write --threads=1 <files...>
```

lint 用 repo wrapper：

```bash
pnpm lint:core
pnpm lint:extensions
pnpm lint:all
```

不要随手引入新的 formatter/linter 调用。

## 文档和 changelog

行为/API 改动通常要同步 docs。

需要特别注意：

- 对外文档和 UI 文案说 plugin/plugins，不说 extensions。
- 新 channel/plugin/app/doc surface 可能要更新 `.github/labeler.yml` 和 GitHub labels。
- 用户可见修复通常需要 changelog。
- changelog bullet 单行，不换行。

学习笔记写在 `.workplace/learn/`，不等于官方 docs；这里是给新手读源码的本地知识库。

## Git 工作流

仓库要求：

- commit 用 `scripts/committer "<msg>" <file...>`。
- 只 stage 本次意图文件。
- 不要 revert 用户已有改动。
- 不要手动 stash，除非用户明确要求。
- `main` 不要 merge commit，push 前通常 rebase。

你现在学习阶段不一定要 commit，但要养成先看 `git status` 的习惯。

## 当前环境提示

这次整理时发现：

- `.workplace/learn/` 已有 01-05 五篇学习笔记和 README。
- `pnpm docs:list` 因 `pnpm` 不在 PATH 失败。
- `node scripts/docs-list.js` 可用，已经用于读取 docs 索引。
- PowerShell 输出中文存在编码显示问题，但文件内容本身是 UTF-8。

