# 从 Claude Code（或 Codex）过来

[English](coming-from-claude-code.md) | 中文

基于 `@deepseek-ai/dsh` **0.1.0-rc.6**（harness 源码树 `47f9438`）在 2026-08-15 实测；标了 *2026-10-18* 的条目在那天对 0.2.0-rc.2（npm `latest`）和 0.2.1-rc.2（`next`）重新验证过，每次都用全新的 `DSH_HOME`；标了 *2026-10-19* 的条目用同样的方式对 0.2.1-rc.2 重新验证过，它在那天成了 npm `latest`。表中每条关于 dsh 的主张都写明了对应源码；Claude Code 一侧属于外部事实，同样以这一天为准。两个工具都变化很快。

写给：每天在用 Claude Code 或 Codex、想花十分钟弄清楚哪些东西原样能用、哪些换了名字、哪些根本不存在的人。这不是模型质量对比；那需要 benchmark，不是一张表。

一句话版本：项目里的 `CLAUDE.md` 和你的 `SKILL.md` skill 原样能用；MCP server 和 `hooks.json` 各需要一行 YAML；**全局**那份 `CLAUDE.md` 得自己复制成另一个名字——这一样恰恰是所有人都以为能用、实际上不能用的。你失去的是终端：dsh 是一个浏览器应用外加一个单次任务 CLI，你可能见过的 "TUI" 都是社区作品。

## 对照表

| 你熟悉的 | 在 dsh 里 | 依据 |
|---|---|---|
| `claude`（终端里的 TUI） | `npx @deepseek-ai/dsh web`，跑在 `127.0.0.1:3080` 的浏览器应用。源码树里没有 TUI；曾经自带的那个 2026-08-04 删掉了（`.agents/notes/archived/simplification/2026-08-04-remove-tui-package.md`）。社区 TUI 有几个（在 `dsh-plugin` topic 里搜）；官方自己指向的是 `github:deepseek-harness/turtle-ui`。 | `apps/cli/reference/README.md` |
| `claude -p "task"`（无头） | `dsh --profile headless "task"`：只打印最终回答，完成时退出码 0。和 web 应用一样会读你的项目指令文件。*2026-10-19：*`--json` 把逐行的运行事件写到 stdout，代替最终文本（0.2.0-rc.2 里也有）；单独的 JSON 演示 CLI 2026-08-08 删了。要驱动会话的程序仍然用 Python SDK 和 JSON-RPC（`--profile sdk`）。 | `dsh --profile headless --help`、`apps/cli/reference/README.md`、`.agents/notes/archived/simplification/2026-08-08-remove-cli-demo.md` |
| `~/.claude/CLAUDE.md`、`./CLAUDE.md`、`CLAUDE.local.md` | **项目里的文件原样能用，全局那个不行。** 用户全局文件永远是 `$DSH_HOME/AGENTS.md`——`~/.claude/CLAUDE.md` 根本不会被读，你得自己复制过去；而且全局这一层不接受 `.local.md` 覆盖层。往下，从项目根（最近的 `.git`）到 cwd 的每一层目录里的 `AGENTS.md` 和 `CLAUDE.md`，外加 `AGENTS.local.md` / `CLAUDE.local.md` 覆盖层。同一目录内，去掉首尾空白后逐字节相同的兄弟文件只加载一次。渲染预算 64 KiB。有两样东西会悄悄失效：`@path` 导入和 `.claude/rules/` 都不会被解析。 | `packages/context/agent-instructions/README.md` |
| `~/.claude/skills/<name>/SKILL.md`、`.claude/skills/` | 文件格式完全相同。根目录按匹配顺序排列，先命中的赢：`<project>/.dsh/skills`、`<project>/.agents/skills`、配置的额外目录、`$DSH_HOME/skills`、`~/.agents/skills`。你用 `npx skills add` 装进 `~/.agents/skills` 的 skill，因此已经在 dsh 里了；项目层同名的 skill 会把它遮蔽掉。名字必须是 kebab-case；frontmatter 必须有 `name` + `description`；嵌套的 `SKILL.md` 不会被发现。根目录是实时监听的。 | `packages/skill/skill-filesystem/README.md` |
| `/skill-name` | 一样：在输入框里敲 `/<name>`，`/` 菜单的 "Skills" 一栏里列着。`user-invocable: false` 让它不进这个菜单；`disable-model-invocation: true` 让它不暴露给模型。 | `packages/skill/skill-filesystem/README.md`、`packages/client/ui-skill/README.md` |
| `.claude/commands/*.md`（自定义斜杠命令） | 没有单独的概念。一个开着 `user-invocable` 的 skill 就是你的斜杠命令。今天内置的命令：`/compact`、`/export`、`/feedback`、`/goal`、`/permission`、`/plan`、`/model`。 | `packages/interaction/commands/README.md`；`/` 菜单 |
| `hooks.json` / settings 里的 `hooks` | 可选装的桥接插件 `@deepseek-ai/dsh-hooks-claude-code`：原样运行你现有的 `hooks.json`，支持 `SessionStart`、`UserPromptSubmit`、`PreToolUse`、`PostToolUse`、`Stop`、`SubagentStart`、`SubagentStop`。只有 `type: "command"` 的 hook 会跑；`http` / `mcp_tool` / `prompt` / `agent` 类型的会被解析后跳过并打警告。一个进程一份配置，不是每个会话一份。`updatedInput` 改写只记日志，不生效。`${CLAUDE_PROJECT_DIR}` 可用。Codex 有它自己的桥。*2026-10-18：*别从 npm `latest` 装，那个 tag 还停在 `0.0.1-rc.5`，会被 dsh 的兼容性检查拒绝；能用的装法见下面第 4 步。 | `packages/hooks/hooks-claude-code/README.md`（0.2.0）、`packages/experimental/hooks-claude-code/README.md`（0.2.1） |
| `.mcp.json`、`claude mcp add` | 在 patch 层里每个 server 一行 `@deepseek-ai/dsh-mcp-client`；`stdio` 或 `streamable-http`；工具名是 `mcp__<server>__<name>`，和 Claude Code 同一种形状。随 CLI 一起装好，默认一个都不启用。还没有 OAuth 流程。 | `packages/mcp/mcp-client/README.md`、`apps/cli/reference/README.md` |
| 权限模式（`default`、`acceptEdits`、`plan`、`bypassPermissions`） | 权限预设：`workspace-write`（沙箱把写操作限制在工作区 + 临时目录；只有请求越界时才审批）是默认；`danger-full-access`（无沙箱，永不询问）。输入区里的权限控件还提供 Read Only。计划模式是 `/plan` + `exit_plan_mode`，它是软性约束，和沙箱、审批是两回事。 | `docs/subsystems/permission-presets.md`、`docs/subsystems/sandbox.md`、`packages/plan/plan-mode/README.md` |
| 沙箱（macOS 上 bash 的 seatbelt，可选） | 沙箱默认开启：Linux 用 bwrap/Landlock，macOS 用 Seatbelt，Windows 用 ACL（那里以及旧版 Landlock 内核上都只是部分强制）。它只管文件系统效果；网络和进程可见性都不在范围内。如果你的机器上没有一个后端能用，命令不会悄悄裸跑，而是以 `SANDBOX_UNAVAILABLE` 失败。 | `docs/subsystems/sandbox.md`、`packages/sandbox/sandbox-local/README.md` |
| `.claude/agents/*.md`（子 agent） | 是两件事。委派：`subagent` / `subagent_fork` 工具，provider 有 `spawn`、`fork`（进程内）；`codex` 和 `claude-code` provider 存在但不在 base bundle 里。人设 + 工具集：一个 **agent 预设**。*2026-10-19：*预设是 patch 层里的一行 `@deepseek-ai/dsh-agent-preset`（`id`、`order`，以及它组合的子 `plugins`），你自己的也这样声明；预设目录和 `$DSH_HOME/.agent-presets/` 在 2026-09-18 取消了。web profile 自带四个：`standard`、`ptc`、`minimal`、`cordis`，界面上分别叫 Standard / PTC / Minimal / **Creator** mode，最后一个 id 和界面名字对不上；`ptc` 在 2026-08-25 之前叫 `code`。"Agent teams" 不是 dsh 的词。 | `docs/subsystems/subagent.md`、`packages/preset/agent-preset/README.md`、`.agents/notes/implemented/architecture/2026-09-18-declarative-agent-presets.md`、`packages/client/ui-agent-preset/src/client/locales.ts`、`dsh --profile web --dump-config` |
| `~/.claude/settings.json` | 用户设置在 `$DSH_HOME/settings.yaml`；组合（装哪些插件、什么配置）在 `$DSH_HOME/profiles/<name>/cordis.patch.yml` 和 `$DSH_HOME/cordis.patch.yml`。两个文件，两份工作。 | `apps/cli/reference/README.md` |
| `ANTHROPIC_API_KEY`、OAuth 登录、订阅 | 用原生 DeepSeek provider 时：`DEEPSEEK_API_KEY` 或 Settings > Models。只有 API key，没有 OAuth 也没有订阅制，按 DeepSeek API 的 token 价格计费。优先级：环境变量 > `$DSH_HOME/.credentials.yaml` > `./.env` > `$DSH_HOME/.env`。 | `packages/credentials/credentials-local/README.md`、`packages/llm/llm-deepseek/README.md` |
| 选模型、`ultrathink` | `/model` 或输入区里的模型选择器，它同时会列出该模型的 adapter 声明的推理强度档位；DeepSeek adapter 默认 High。provider 目录里有 Anthropic、OpenAI、Bedrock、Vertex、Azure、Codex，所以你可以在 dsh 里跑 Claude 模型。 | `docs/user/guide/providers.md`、`packages/client/ui-model-selection/README.md`、`packages/llm/llm-deepseek/README.md` |
| `--resume`、`--continue`、`~/.claude/projects/*.jsonl` | 左边栏按工作区分组，点会话就行。*2026-10-19：*headless 用 `dsh --profile headless --session-id <id> "continue"` 接着一个会话往下跑（0.2.0-rc.2 里也有）；没有 `--resume` 参数，启动器帮助里的 `dsh tui --resume <session>` 例子指向一个 dsh 并不自带的 profile。日志：`$DSH_HOME/sessions/--<规范化 cwd>--/<编码后的 session id>/session.jsonl.zstd`——JSONL 以连续拼接的 zstd frame 存储；`/export` 下载一个 ZIP。 | `dsh --profile web --help`、`dsh --profile headless --help`、`packages/session/session-persistence-jsonl/README.md` |
| `/cost`、ccusage | 输入框下面的状态行：turns、steps、LLM 耗时、TTFT、tok/s、缓存命中率、输入 / 输出 token。adapter 报告用量时，每一步的数字写在它自己的 `assistant/message` 事件上。 | 实测，另见 `packages/session/session-stats/README.md`、`packages/llm/token-meter/README.md` |
| 插件、marketplace | `dsh plugin --profile web add <npm 包 | github:owner/repo>`；包必须在 `package.json` 里声明 `dsh.bundle`，否则只装成普通依赖、什么都不发生。*2026-10-18：*从 0.2.1 起，在 **Plugins** > **Experimental plugins** 里打开 **Custom plugin markets**，会多出一个 **Discover** 标签页，读取任意插件市场索引 URL；dsh 自己不带任何市场，我们的是 `https://dsh.works/awesome-dsh-plugins/market/index.json`。除此之外，发现渠道是 GitHub 的 `dsh-plugin` topic 和社区注册表（我们的：`dshworks/awesome-dsh-plugins`）。git 包如果带 `prepare` 脚本，在你把它的构建加入 allowlist 之后会在安装时执行代码，而且这些代码不受沙箱约束。 | `docs/user/develop/basic/publish.md`、`docs/user/guide/plugin-market.md`（0.2.1） |
| WebSearch / WebFetch | `web_search` 开着（DeepSeek 的搜索 provider，用你的 DeepSeek key）；`web_fetch` 关着，除非有 patch 启用一个 provider。 | `apps/cli/reference/README.md` |
| VS Code / JetBrains 扩展 | 源码树里没有。Discussion #172 一次要了独立客户端、CLI 和 VS Code 插件三样，截至 2026-08-15 有 159 个赞，是第一天呼声最高的需求；它没有提 JetBrains。 | GitHub Discussion #172（外部） |
| `CLAUDE_PROJECT_DIR` | 桥接的 hook 会拿到 `CLAUDE_PROJECT_DIR`。harness 自己的环境变量是 `DSH_HOME`、`DSH_AGENTS_HOME`、`DSH_PERMISSION_MODE`、`DSH_TOOLS_MODE`，另外 `DSH_WEB_URL` 只有 web profile 才提供。 | `packages/experimental/hooks-claude-code/README.md`（0.2.1 之前在 `packages/hooks/`）、`packages/skill/skill-filesystem/README.md`、`packages/bundle/web-app/README.md`、`apps/cli/reference/README.md` |

## 头十分钟做五件事

1. 在一个已经有 `CLAUDE.md` 的仓库里启动。问它："原样引用你拿到的所有项目指令文件。"我们在一个只有两行 `CLAUDE.md` 的仓库里用 headless 试了；它原样引了出来并说出了路径。项目里的文件不用改名——但别忘了先把 `~/.claude/CLAUDE.md` 复制成 `$DSH_HOME/AGENTS.md`，全局那份不会自己过来。
2. 在输入框里敲 `/`。`~/.agents/skills` 里合法、且允许用户调用、又没有被同名的高优先级 skill 遮蔽的，都会出现在 "Skills" 一栏。新建一个 `<project>/.agents/skills/<name>/SKILL.md`，再敲一次 `/`：不用重启就出现了。

   ![斜杠菜单里的项目 skill](../assets/coming-from-claude-code/02-project-skill-in-slash-menu.jpg)

3. 接一个 MCP server。编辑 `$DSH_HOME/profiles/web/cordis.patch.yml`（初始内容是 `[]`）：

   ```yaml
   - insert:
       - id: mcp-github
         name: '@deepseek-ai/dsh-mcp-client'
         config:
           serverName: github
           transport: stdio
           command: npx
           args: ['-y', '@modelcontextprotocol/server-github']
           env:
             GITHUB_TOKEN: !!js process.env.GITHUB_TOKEN
   ```

   这个文件是实时重载的；工具会以 `mcp__github__*` 出现。我们用 `@modelcontextprotocol/server-everything` 在 headless profile 里验证了这个写法：下一次运行就多了 13 个 `mcp__everything__*` 工具，不需要 `dsh plugin add`，因为客户端随 CLI 一起装好了。记住 server 命令是在沙箱之外运行的受信代码，这就是默认一个都不开的原因。

4. 非要带 hook 的话：*2026-10-19：*`npx` 现在装的是 0.2.1，桥接插件不再随 CLI 一起装好，而且 profile 不会替你装 peer（在 0.2.0 上只要下面这一行就够了）：按你的 dsh 版本把桥接插件和它的协议 peer 一起装上，`dsh plugin --profile web add @deepseek-ai/dsh-hooks-claude-code@0.2.1-rc.2 @deepseek-ai/dsh-hook-protocol@0.2.1-rc.2`（它会警告桥接插件没有声明 `dsh.bundle`；这是预期的，下面这行才是激活它的东西）。两个都要写版本号：不写的话 npm 给你的是 `0.0.1-rc.5`，0.2.0 和 0.2.1 都会拒绝。漏了协议 peer，0.2.1 只打一行 `failed to import ... Cannot find package '@deepseek-ai/dsh-hook-protocol'` 就照常启动，你的 hook 永远不会跑。然后：

   ```yaml
   - insert:
       - id: hooks-cc
         name: '@deepseek-ai/dsh-hooks-claude-code'
         config:
           configPath: ./.claude/hooks.json
           projectDir: .
   ```

   先读这个桥自己的 README，它对自己的定位说得很直白："a native plugin has the full harness API, while this bridge runs only the reference tool's command-hook subset"——同样的事情，原生插件能做得更好：有类型化的返回值，也没有序列化边界。已有的 hooks 搬过来用，新写的东西挂到 harness 自己的扩展点上。

5. 在你原来用 `-p` 的地方试 headless：`dsh --profile headless "summarize this workspace"`。成功时 stdout 上只有最终文本，别的什么都没有。如果你之前在解析 Claude Code 的 stream-json，`--json` 给你的是逐行的运行事件；那是 dsh 自己的事件，解析器得重写。`--session-id <id>` 把后续任务发进一个已有的会话。

## dsh 有而 Claude Code 没有的

- 组合是你的。随发行版提供的能力都是一行行插件，你可以用 `dsh --profile web --dump-config` 看到它、禁用它、替换它，或者在一个热重载的 YAML 文件里改它的配置。改动不用等上游给你开开关。
- 三个操作系统上都默认开着的文件系统沙箱（Windows 和旧 Landlock 内核上只是部分强制），以及一个只在配置好的边界上才问你的权限模型。
- Agent 预设：每个会话一整套工具 + 提示词的组合，按会话切换，你自己的就是多一行 YAML。比 Claude Code 暴露出来的任何东西都更接近"每种活一份 Claude Code 配置"。
- 一份作为真相来源的会话日志：模型的上下文和 UI 都是从它推导出来的。resume 和 export 只是读它；fork 用它开一个子会话。
- Provider 目录。同一个界面跑 Claude、GPT 和开源模型；DeepSeek 适配器只是原生的那个。

## 你会想念的

- 终端。没人应该假装浏览器标签页里的 `dsh web` 和 tmux 里的 `claude` 是同一种工作流。截至 2026-08-15，社区 TUI 也才发布了几天。
- IDE 集成。没有。
- shell 命令之外的 hook；按会话的 hook 配置；`updatedInput`。
- MCP OAuth。
- 稳定性。README 用大写字母写着 THERE WILL BE COMPATIBILITY-BREAKING CHANGES，仅 2026-08-11 一天的改名台账就记录了几十个包名和公开标识符的改名。钉住版本（`npx @deepseek-ai/dsh@0.1.0-rc.6 web`），上周写的东西加载不了时去读 `fix/`。

## 接下来

- 启动就坏：[fix/boot-fails-with-a-wall-of-aggregateerror.zh.md](../fix/boot-fails-with-a-wall-of-aggregateerror.zh.md)。所有 `/api/*` 调用都 403：[fix/every-api-call-returns-403.zh.md](../fix/every-api-call-returns-403.zh.md)。
- 花钱。同样的 token，不同的价目表，外加从 2026-08-16 16:00 UTC 起的峰谷时段；[first-hour.zh.md](first-hour.zh.md) 第 7 节有一张真实账单。
- 写一个插件补上面某个缺口，是这个仓库接下来的一条线。现在还没写。
