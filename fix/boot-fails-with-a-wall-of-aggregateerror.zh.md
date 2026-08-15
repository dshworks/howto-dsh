# 启动刷出一整屏 `AggregateError`

[English](boot-fails-with-a-wall-of-aggregateerror.md) | 中文

基于 `@deepseek-ai/dsh` 0.1.0-rc.6，2026-08-15 实测。两种会刷屏的故障都在本机复现过，下面的报错均为原文。

## 症状

`dsh web`（或者 `dsh --profile <name> ...`）非零退出，退出前刷出约 40 到 60 行，开头是：

```
Error: dsh: plugin tree failed to load: failed to apply loader entry include (cordis:include): ...
AggregateError: loader entries failed to apply
    at EntryGroup.update (.../cordis-plugin-loader/lib/index.js:91:35)
```

后面跟着一层套一层的 `[cause]:`。它往往"昨天还好好的"；常见触发点是 agent 改了 profile、你装了个插件，或者别的进程占了端口（Discussions #2056、#1197、#2015）。

## 原因

dsh 按 fail-closed 的方式组合 profile：一行插件装不起来，include、插件树、启动会依次失败。外层那一堆只是逐层转抛，真正能动手的是最里层的 `[cause]`。会刷出这一屏的有两种：

| 最里层那行 | 含义 |
|---|---|
| `failed to import loader entry oops (dsh-does-not-exist): Cannot find package 'dsh-does-not-exist' imported from ~/.dsh/profiles/headless/` | 某一行写了**那个 profile 里**并没有装的包：拼错、包改了名、`link:` 指向的目录挪走了，或者 `dsh plugin remove` 之后那一行还留着 |
| `failed to apply loader entry webserver (@deepseek-ai/dsh-host-webserver): listen EADDRINUSE: address already in use 127.0.0.1:3080` | 端口被别的进程占着，通常是你以为已经停掉的上一个 `dsh web` |

坏行可能来自这些层，按生效顺序：`profiles/<name>/package.json` 里 `dsh.profile.bundles` 列出的每个 bundle、`profiles/<name>/cordis.patch.yml`、`$DSH_HOME/cordis.patch.yml`，最后是各个 `--patch` 文件。

不是这一屏的情况：如果只有短短一行，比如 `dsh: failed to parse overlay ~/.dsh/profiles/headless/cordis.patch.yml: YAMLException: unexpected end of the stream within a flow collection (4:1)`，那是 YAML 写坏了，而且它已经告诉你第几行。按那个行号去改就行——patch 文件是一个顶层 YAML 数组，空的状态是 `[]`。

## 修

1. 只读最里层那行 `Cannot find package` 或 `EADDRINUSE`。它上面的所有内容都是同一个错误被反复往上抛。
2. **端口**：`dsh web --port 3081`，或者找出旧进程停掉（`lsof -i :3080`）。
3. **缺包**：往出错的那个 profile 里装上（`dsh plugin --profile <name> add <package>`），或者删掉写着它的那一行手写配置，或者把带进这一行的 bundle 移除。手写的行在 profile patch、home patch 或某个 `--patch` 文件里；bundle 列在 profile 的 `package.json` 里。注意 `dsh plugin --profile <name> remove <package>` 只同步 bundle 列表，不会动你手写的行。
4. **先证明这份安装本身没问题**：用一个一次性的 home 和一个空闲端口启动 `DSH_HOME="$(mktemp -d)" dsh web --port 3099`。全新的 home 只有自带的 bundle 和空的用户 patch。如果这样能起来，问题就在你自己的 `$DSH_HOME` 或者它引用的东西上，而不在你装的这份 dsh 上。
5. **不启动也能看组合出来的树**：`dsh --profile <name> --dump-config`。每一行上面都有注释，写明它来自哪一层、又被哪些 overlay 改过（`# == @deepseek-ai/dsh-base, patched by @deepseek-ai/dsh-headless`），外来的行一眼就能看出来。没有匹配到任何目标的 patch 会打到 stderr 上——那通常正是你在找的那一行。

## 不要

- 不要为了让它别报错就 `rm -rf ~/.dsh`。你的会话、凭据和预设都在里面。真要动就先挪开（`mv ~/.dsh ~/.dsh.bak`），再把 `sessions/` 和 `.credentials.yaml` 拷回来。
- 不要让 agent 去修承载它自己的那个 profile。这个文件你自己改。

## 出处

`apps/cli/reference/README.md`（层的顺序、`--dump-config` 的语义、未匹配的 patch 会报到 stderr）；`packages/host/webserver/README.md`（"A listen failure (EADDRINUSE...) throws out of activation and rejects Loader composition"）；`docs/user/develop/basic/publish.md`（bundle 和普通依赖的区别）。
