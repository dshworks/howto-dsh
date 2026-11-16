# `dsh plugin add` 装上的是你刚发布的上一个版本

[English](plugin-add-installs-the-previous-release.md) | 中文

基于 `@deepseek-ai/dsh` 0.2.2-rc.1 与 pnpm 12.18.0，2026-11-16 实测：在临时 `DSH_HOME` 里从 npm 安装两个插件，并对照 `dsh-v0.2.2-rc.1` 的插件管理器源码。

## 症状

你发布了一个修复版。一小时后有人运行 `dsh plugin add <你的包>`，装上的却是上一个版本，而 npm 上 `latest` 已经是新版。输出里没有任何提示说还有更新的版本。

2026-11-16 约 05:40 UTC 实测：

| 包 | npm 最新版（发布时间） | `dsh plugin add <包名>` 实际安装 |
|---|---|---|
| `dsh-context` | 0.85.1（11-15 18:48 UTC，约 11 小时前） | 0.84.1 |
| `dsh-media-desk` | 2.7.2（11-15 17:12 UTC，约 12.5 小时前） | 2.7.1 |

## 原因

版本不是 dsh 选的。`dsh plugin add` 和 Web UI 里的安装器都在 profile 目录里运行 `pnpm add`，用的是你 `PATH` 上的 `pnpm`（`packages/boot/plugin-manager/src/operations.ts:34`、`:369`）。从 pnpm 11 起，`minimumReleaseAge` 默认为 24 小时：不指定版本时，会解析到发布满一天的最新版本。

dsh 知道这件事。插件管理器会报告 pnpm 实际装上的版本，类型注释写明 `minimumReleaseAge` 可能让它比最新发布更旧（`plugin-manager/src/types.ts:176`）；桌面端打包 README 也点名了 pnpm 11 默认的 24 小时 `minimumReleaseAge`。

这是供应链防护，不是 dsh 的 bug：一个被投毒的版本有一天时间被发现，然后才会进入别人的 profile。

## 解决

- **要装发布不满一天的版本，就指定版本号。** `dsh plugin add dsh-context@0.85.1` 装上了 0.85.1，pnpm 把豁免记进了 profile：

  ```
  Added 1 entry to minimumReleaseAgeExclude in pnpm-workspace.yaml (set minimumReleaseAgeStrict to true to gate these updates with a prompt):
    dsh-context@0.85.1
  ```

  豁免只对这一个确切版本有效，下次不指定版本的更新仍要等满一天。
- **插件作者：** 发修复版时，在发布说明里写上带版本号的命令（`dsh plugin add <包名>@<版本>`），否则用户要晚一天才拿到修复。自己的安装测试也要指定版本，否则测的是上一个版本。
- **确认装了什么：** 安装结束时 pnpm 会打印 `+ <包名> <版本>`。这一行才是实际安装的版本，不是 npm 上的 `latest`。

未测试：`PATH` 上的 `pnpm` 低于 11 的情况；也没有测试关闭这个时间窗口。关掉它会让该 profile 里所有包都失去这层防护。
