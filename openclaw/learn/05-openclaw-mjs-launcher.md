# 05. openclaw.mjs 启动器解析

## 结论

`openclaw.mjs` 是 OpenClaw 的 CLI 启动器，不是主要业务逻辑入口。

它的职责是做启动前的校验、缓存处理、帮助文本快速输出、warning 过滤和构建产物入口加载。真正的 CLI 逻辑在 `dist/entry.js` 或 `dist/entry.mjs`，源码对应 `src/entry.ts`。

`package.json` 里把 npm 命令绑定到了这个文件：

```json
"bin": {
  "openclaw": "openclaw.mjs"
}
```

所以用户运行：

```bash
openclaw ...
```

实际会先进入 `openclaw.mjs`。

## 总体启动链路

```text
openclaw 命令
  -> openclaw.mjs
     -> 检查 Node 版本
     -> 处理 Node compile cache
     -> 尝试 help 快速输出
     -> 安装 warning filter
     -> import dist/entry.js 或 dist/entry.mjs
        -> src/entry.ts 的构建产物
           -> cli/run-main.js
              -> 真正 CLI 命令系统
```

换句话说，`openclaw.mjs` 是一层非常薄但很关键的 bootstrap wrapper。

## Node 版本检查

文件开头定义最低 Node 版本：

```js
const MIN_NODE_MAJOR = 22;
const MIN_NODE_MINOR = 12;
```

`ensureSupportedNodeVersion()` 会读取 `process.versions.node`，解析 major/minor，然后判断是否满足 `22.12+`。

如果不满足，会输出：

```text
openclaw: Node.js v22.12+ is required
```

并给 nvm 用户提示：

```bash
nvm install 22
nvm use 22
nvm alias default 22
```

然后 `process.exit(1)`。

注意：`package.json` 的 `engines.node` 要求可能更高。这里是启动器自己的最低硬拦截。

## 判断源码树还是发布包

`isSourceCheckoutLauncher()` 判断当前启动器是不是在源码 checkout 里：

```js
existsSync(new URL("./.git", import.meta.url)) ||
existsSync(new URL("./src/entry.ts", import.meta.url))
```

只要 `openclaw.mjs` 旁边存在 `.git` 或 `src/entry.ts`，就认为这是源码树。

这个判断主要服务于 compile cache 策略：

- 源码树：禁用 Node compile cache。
- 发布包：尝试启用 Node compile cache。

## Compile Cache 策略

### 源码树禁用 cache

如果当前是源码树，并且 Node compile cache 已经启用，或者用户设置了 `NODE_COMPILE_CACHE`，启动器会重新 spawn 自己。

新进程环境里会设置：

```js
NODE_DISABLE_COMPILE_CACHE: "1"
OPENCLAW_SOURCE_COMPILE_CACHE_RESPAWNED: "1"
```

同时删除：

```js
NODE_COMPILE_CACHE
```

这样可以避免源码 checkout 在开发过程中被 Node compile cache 影响。源码文件经常变化，如果缓存行为介入，可能让开发态启动结果不够直观。

`OPENCLAW_SOURCE_COMPILE_CACHE_RESPAWNED` 是防止无限 respawn 的标记。

### 发布包启用 cache

如果不是源码 checkout，启动器会 best-effort 调用：

```js
module.enableCompileCache()
```

这可以优化已安装 npm 包的重复启动速度。

失败会被忽略，因为 compile cache 只是性能优化，不应该阻止 CLI 启动。

## 模块缺失判断

启动器定义了两个辅助函数：

- `isModuleNotFoundError(err)`
- `isDirectModuleNotFoundError(err, specifier)`

它们用来区分两类错误：

1. `./dist/entry.js` 本身不存在。
2. `./dist/entry.js` 存在，但它内部 import 的依赖不存在。

这是很重要的设计。

如果是第一种，启动器可以 fallback 到 `./dist/entry.mjs`，或者输出友好的 “missing dist/entry.(m)js”。

如果是第二种，启动器必须把真实错误抛出来。否则用户看到的会是误导性的“缺少构建产物”，但真正问题可能是某个依赖损坏。

对应逻辑在 `tryImport()`：

```js
try {
  await import(specifier);
  return true;
} catch (err) {
  if (isDirectModuleNotFoundError(err, specifier)) {
    return false;
  }
  throw err;
}
```

## Warning Filter

`installProcessWarningFilter()` 会尝试加载：

```js
./dist/warning-filter.js
./dist/warning-filter.mjs
```

如果模块存在，并且导出了 `installProcessWarningFilter`，就调用它。

源码实现对应 `src/infra/warning-filter.ts`。它主要过滤一些已知噪音 warning，比如：

- `DEP0040` punycode
- `DEP0060` `util._extend`
- SQLite experimental warning

这里启动器只负责尽早安装过滤器，保证 bootstrap 阶段和 TypeScript runtime 阶段的 warning 表现一致。

## 构建产物缺失提示

真正入口是：

```js
./dist/entry.js
./dist/entry.mjs
```

如果两个都不存在，启动器会抛出：

```text
openclaw: missing dist/entry.(m)js (build output).
```

如果同时检测到 `src/entry.ts` 存在，说明这更像未构建源码树，于是会追加恢复建议：

```text
This install looks like an unbuilt source tree or GitHub source archive.
Build locally with `pnpm install && pnpm build`, or install a built package instead.
For pinned GitHub installs, use `npm install -g github:openclaw/openclaw#<ref>` instead of a raw `/archive/<ref>.tar.gz` URL.
For releases, use `npm install -g openclaw@latest`.
```

这个逻辑对用户很有价值：它把“构建产物不存在”翻译成可执行的修复步骤。

## Help 快速路径

启动器对两个命令做了特殊优化：

```bash
openclaw --help
openclaw -h
openclaw browser --help
openclaw browser -h
```

这两个判断分别是：

```js
isBareRootHelpInvocation(process.argv)
isBrowserHelpInvocation(process.argv)
```

它们要求 `argv.length` 精确匹配，所以这是非常窄的 fast path，不是通用命令解析器。

### 预计算帮助文本

启动器会读取：

```text
dist/cli-startup-metadata.json
```

里面的字段：

```js
rootHelpText
browserHelpText
```

如果存在，就直接 `process.stdout.write()` 输出，不再加载完整 CLI。

这样可以让 `openclaw --help` 很快，因为它不用启动 Commander、Gateway、插件系统等重模块。

### root help 的 fallback

如果 `rootHelpText` 不存在，root help 还有一个 fallback：

```js
./dist/cli/program/root-help.js
./dist/cli/program/root-help.mjs
```

加载后调用：

```js
mod.outputRootHelp()
```

`browser --help` 没有动态 fallback；如果没有预计算文本，就进入完整 CLI 路径。

### 禁用 fast path

可以通过环境变量禁用：

```bash
OPENCLAW_DISABLE_CLI_STARTUP_HELP_FAST_PATH=1
```

禁用后会走完整入口加载流程。

## 最终主流程

文件最后的逻辑可以简化成：

```js
if (helpFastPathOK) {
  // done
} else if (browserHelpFastPathOK) {
  // done
} else {
  await installProcessWarningFilter();
  if (await tryImport("./dist/entry.js")) {
    // done
  } else if (await tryImport("./dist/entry.mjs")) {
    // done
  } else {
    throw new Error(await buildMissingEntryErrorMessage());
  }
}
```

这里的 `import("./dist/entry.js")` 不是为了拿某个函数返回值，而是为了执行入口模块的顶层代码。

## 和 src/entry.ts 的分工

`openclaw.mjs` 只负责“找到并安全加载入口”。

进入 `dist/entry.js` 后，也就是 `src/entry.ts` 的构建产物，才开始处理真正 CLI 启动：

- 判断当前文件是否主模块。
- 设置 `process.title = "openclaw"`。
- 安装 warning filter。
- normalize 环境变量。
- 处理 `--no-color`。
- 标准化 Windows argv。
- 解析 `--profile` / `--dev` / `--container`。
- 最终加载 `./cli/run-main.js` 并调用 `runCli(argv)`。

所以不要在 `openclaw.mjs` 里找 Gateway、插件、Agent loop、配置解析等业务逻辑。那些都在后续入口和各子模块中。

## 为什么这个文件重要

虽然它不包含业务功能，但它决定了 CLI 是否能可靠启动。

它覆盖了几个安装和运行边界：

- Node 版本太旧。
- 源码树未构建。
- GitHub archive 安装方式不对。
- 构建产物是 `.js` 还是 `.mjs`。
- 入口文件缺失和入口内部依赖缺失的区别。
- help 命令的冷启动性能。
- 源码树和发布包的 compile cache 差异。

这类文件的特点是：代码不长，但每个分支通常都对应一个真实的安装、发布或开发环境问题。

## 阅读建议

如果继续追启动链路，建议按这个顺序读：

1. `openclaw.mjs`
2. `src/entry.ts`
3. `src/entry.compile-cache.ts`
4. `src/cli/run-main.ts`
5. `src/cli/program/root-help.ts`
6. `scripts/write-cli-startup-metadata.ts`
7. `test/openclaw-launcher.e2e.test.ts`

其中 `test/openclaw-launcher.e2e.test.ts` 很适合用来理解设计意图，因为它覆盖了：

- 入口内部依赖缺失不能被误报成 missing dist。
- 真正缺少构建产物时要输出友好错误。
- 未构建源码树要给恢复步骤。
- 源码树禁用 compile cache。
- 发布包启用 compile cache。

## 本次解析限制

按仓库规则，我先尝试运行了：

```bash
pnpm docs:list
```

但当前 shell 报 `pnpm` 命令不存在。因此本篇解析基于源码和相关测试文件整理，主要参考：

- `openclaw.mjs`
- `package.json`
- `src/entry.ts`
- `src/entry.compile-cache.ts`
- `src/infra/warning-filter.ts`
- `src/cli/startup-metadata.ts`
- `src/cli/root-help-metadata.ts`
- `scripts/write-cli-startup-metadata.ts`
- `test/openclaw-launcher.e2e.test.ts`
