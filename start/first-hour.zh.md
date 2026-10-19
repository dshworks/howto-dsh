# dsh 的第一个小时

[English](first-hour.md) | 中文

基于 `@deepseek-ai/dsh` **0.1.0-rc.6**（harness 源码树 `47f9438`，tag 0.1.0-rc.5）在 2026-08-15 实测，Linux，Node 22.23。截图全部来自这次实测。dsh 还是开发者预览版，官方明说会有破坏兼容的改动；如果这里写的按钮或路径和你屏幕上的不一样，以你的屏幕为准，然后告诉我们。

写给：装好了 Node 和 git、手里有 DeepSeek API key、想用一次坐下来的时间判断这东西适不适合自己的人。不写给：从 Claude Code 迁移过来的人，那篇是 [coming-from-claude-code.zh.md](coming-from-claude-code.zh.md)。

读完这篇你会：启动 dsh，把它指向一个有两个失败测试的小仓库，看着它修好，自己核对 diff，读一遍 trajectory，最后看到账单：大约三分之一美分。

## 0. 哪个才是官方的（30 秒）

- 仓库：`github.com/deepseek-ai/deepseek-harness`。npm：`@deepseek-ai/dsh`。文档：`deepseek-harness.github.io/deepseek-harness`。
- 不是它：PyPI 上的 `dsh`、`dscode`、任何没有 `@deepseek-ai/` 前缀却叫 `deepseek-harness` 的 npm 包，以及任何标题里写着"官方"但域名不是 `deepseek.com` 的站点。
- 没有官方插件市场，没有官方桌面端，没有官方 TUI。凡是提供这些的都是社区作品，包括我们的。

## 1. 启动

harness 的开发目标是 Node `^22.19.0 || >=24.0.0`（写在仓库根 `package.json` 的 `engines` 里）。发布出来的 CLI 包并没有声明 `engines`，所以 Node 版本低了 npx 不会拦你，只会在后面某个不那么显眼的地方挂掉。先 `node --version` 看一眼。

```sh
mkdir -p /tmp/first-hour && cd /tmp/first-hour
npx @deepseek-ai/dsh web
```

第一次运行会先下载包。之后终端只打一行 `dsh web: http://127.0.0.1:3080`，然后进程一直挂在前台。打开这个地址。3080 被占了就 `npx @deepseek-ai/dsh web --port 3081`。`--host 0.0.0.0` 是故意拒绝的：自带的服务器没有 TLS，也没有鉴权。要从别的机器访问，用隧道而不是把它暴露出去（`ssh -L 3080:127.0.0.1:3080 user@box`）——直接架反向代理会让一半界面返回 403，原因见 [fix/every-api-call-returns-403.zh.md](../fix/every-api-call-returns-403.zh.md)。

第一屏是一个叫 "Internal Testing Notice" 的弹窗。读一遍，点 Continue，以后不会再出现。

![Internal Testing Notice](../assets/first-hour/01-internal-testing-notice.jpg)

然后是空的主页：左边栏有 New Session、Workspaces、Settings；中间是 "Into the Unknown"、一个工作区选择器、一个 agent 预设选择器，还有一个在你选好工作区之前一直禁用的输入区。

![空的主页](../assets/first-hour/02-home-into-the-unknown.jpg)

## 2. 给它 key

三种方式，按 dsh 解析的先后顺序（`packages/credentials/credentials-local/README.md`）：

1. 环境变量：`DEEPSEEK_API_KEY=sk-... npx @deepseek-ai/dsh web`。如果这个变量已经在你的 shell 里导出了，第一次打开 Settings > Models 时 DeepSeek 卡片就已经是绿点。界面改不了来自环境变量的 key，要换只能重启。
2. Settings > Models > DeepSeek > Edit。保存到 `$DSH_HOME/.credentials.yaml`，权限 0600，界面上只写不读（它会告诉你 key 已设置，但再也不会显示出来）。立即生效，不用重启。
3. dsh 启动时所在目录下的 `.env`，或者 `$DSH_HOME/.env`。优先级低于上面两个。

`$DSH_HOME` 默认是 `~/.dsh`。设置、凭据、profile 和会话日志都在它下面，第 8 节会把它打开看。

![Settings > Models](../assets/first-hour/04-settings-models.jpg)

既然在 Settings 里，顺便看两样东西，什么都别动：General 显示默认 agent 预设（"Standard mode"）和默认权限（"Workspace Write"）；Agent presets 显示四个自带预设（Standard、Code、Minimal、Creator）的卡片，可以查看、可以复制，不能编辑。今天要碰的旋钮就这么多。*2026-10-19，0.2.1-rc.2，本页未重跑：*第二个预设现在叫 PTC mode（id 是 `ptc`，2026-08-25 从 `code` 改名）。

![Settings > Agent presets](../assets/first-hour/06-settings-agent-presets.jpg)

## 3. 选工作区：这是边界，不只是一个文件夹

先别把 dsh 指向你真正的项目。另开一个终端——第一个还在跑服务器——做一个一次性的小项目，带真的、会失败的测试，这样你能凭结果判断，而不是凭感觉。下面这个大约 30 行：

```sh
mkdir -p /tmp/first-hour/tally && cd /tmp/first-hour/tally
cat > package.json <<'EOF'
{ "name": "tally", "version": "0.1.0", "type": "module",
  "scripts": { "test": "node --test" } }
EOF
cat > tally.js <<'EOF'
import { readFileSync } from "node:fs";
export function tally(text) {
  const counts = new Map();
  for (const word of text.toLowerCase().split(/\s+/)) {
    if (!word) continue;
    counts.set(word, (counts.get(word) ?? 0) + 1);
  }
  return counts;
}
export function top(counts, n = 3) {
  return [...counts.entries()].sort((a, b) => a[1] - b[1]).slice(0, n);
}
if (process.argv[2]) {
  for (const [w, n] of top(tally(readFileSync(process.argv[2], "utf8")))) console.log(`${n}\t${w}`);
}
EOF
cat > tally.test.js <<'EOF'
import test from "node:test";
import assert from "node:assert/strict";
import { tally, top } from "./tally.js";
test("punctuation does not split words into garbage", () => {
  const c = tally("Hello, hello! HELLO world.");
  assert.equal(c.get("hello"), 3);
  assert.equal(c.get("world"), 1);
});
test("top returns the most frequent first", () => {
  assert.deepEqual(top(tally("a a a b b c"), 2), [["a", 3], ["b", 2]]);
});
EOF
printf 'the cat sat on the mat. The mat was flat, the cat was fat.\n' > sample.txt
git init -q && git add . && git commit -qm "tally: initial" && npm test
```

`npm test` 最后打出 `# pass 0` / `# fail 2`。两个埋好的 bug：标点粘在单词上，`top` 按升序排。交给 dsh 之前先提交这份初始代码——diff 是你评判 agent 的依据。

回到浏览器：点 "Choose workspace"。如果 dsh 能访问图形显示、而且你不是通过 SSH 启动的，弹出来的是操作系统的文件夹对话框；否则是一个浏览器内的选择器，我们的截图就是后者（一台没有图形显示的 Linux 机器）。不管哪种，选 `/tmp/first-hour/tally`；浏览器内的那个有一个铅笔图标可以直接输路径。（精确规则在 `packages/host/directory-picker-auto/README.md`。）

工作区就是权限边界。`workspace-write` 是权限预设，不是旁边那个 agent 预设，它的意思是：agent 的 shell 和文件工具可以在这个目录和一个临时目录里写，别处不行，而且在这个边界之内做什么都不需要问你。只有想跨出边界时才会问。这就是为什么下面整个运行过程里一次审批弹窗都没有。

![选好工作区](../assets/first-hour/07-workspace-chosen-composer.jpg)

输入区现在显示着工作区、"Standard mode"、"Workspace Write" 和模型，默认是 `DeepSeek-V4-Flash · High`。全都别动。

## 4. 跑

原样粘贴，回车：

```
Run `npm test`. Two tests fail. Fix tally.js so both pass without touching the tests, and do not add any dependency. Then tell me, one line each, what the two bugs were.
```

我们这次从上到下是这样（下面的 `09-` 和 `10-`）：会话有了标题（"Fix failing tests in tally.js."）；两行 "Context injection"，分别是系统提示词和 skill 目录；一行 "Think"；然后是工具卡片：`Bash · Show working directory and list files`、`Glob`、四个 `Read`、`Bash · Run the test suite`、两个对 `tally.js` 的 `Edit`、`Bash · Re-run the test suite`、`Bash · Run tally CLI on sample file`，最后是一段点出两个 bug 的回答。8 步，大约 30 秒。你那次中间会不一样，但形状是一样的。

![运行中](../assets/first-hour/09-session-running-tool-cards.jpg)
![完成](../assets/first-hour/10-session-done-answer.jpg)

没有任何审批。如果这让你意外，回去重读第 3 节；然后决定你喜不喜欢这样。不喜欢的话，输入区里那个 "Workspace Write" 控件可以把当前会话切成 Read Only、Workspace Write 或 Full access，键盘上敲 `/permission` 也是同一件事。

## 5. 自己核对

agent 说两个测试都过了。这句话不是证据。在刚才那个第二终端里：

```sh
npm test            # 应该是：# pass 2 / # fail 0
git diff --stat     # 应该是：tally.js | 5 ++---   （我们这次：2 insertions, 3 deletions）
git diff            # 读那两个 hunk；应该恰好就是那两个 bug
```

如果 diff 动了 `tally.test.js`，或者加了依赖，说明 agent 违反了你的指令；这件事在 30 行的项目上发现，好过在你真正的项目上发现。

## 6. 读 trajectory

会话顶部点 "Trajectory"。这一页按顺序列出每一次模型请求和每一次工具调用，带参数、带结果，配一条输入 / 模型 / 工具耗时的时间线。它和 Chat 页渲染的是同一份数据，只是去掉了叙述。运行出了问题，先看这一页，再决定要不要写更长的提示词。

![Trajectory](../assets/first-hour/11-trajectory-tab.jpg)

## 7. 账单

我们这次运行结束后，输入区下面的状态行是：`1 turns · 8 steps | LLM 15.9s · Tool call 1.7s | TTFT avg 0.8s · 172 tok/s | Cache hit 87% | Input 153K tok`。按步拆开（数据来自会话日志）：

| 步 | 新输入 | 缓存读取 | 输出（含推理） |
|---|---|---|---|
| 1 | 17,139 | 0 | 113 |
| 2 | 310 | 17,152 | 171 |
| 3 | 761 | 17,536 | 603 |
| 4-8 | 每步 33-770 | 每步 18.8K-20.4K | 每步 64-260 |
| 合计 | 19,354 | 133,760 | 1,615 |

这样读：第一次请求带的是系统提示词和工具 schema，大约 17K token；之后每一步都会把整段对话重发一遍，但几乎全部命中缓存，而缓存读取按新输入价格的 2% 计费。所以 "153K input" 这个数字里 87% 是缓存读取。

按 DeepSeek 2026-08-15 的挂牌价（`api-docs.deepseek.com/quick_start/pricing`：V4-Flash 每 1M token 新输入 $0.14 / 缓存 $0.0028 / 输出 $0.28），这次运行花了 **$0.0035**。从 2026-08-16 16:00 UTC 起同一页切换到峰谷计费；同样这次运行谷时 $0.0063，峰时 $0.0125（峰时是 01:00-04:00 和 06:00-10:00 UTC），换 V4-Pro 大约再乘 3。仍然是几美分的量级。什么时候不再是几美分：在大仓库上跑长会话，每一步都重读一大段上下文而且缓存没命中。

## 8. 东西都去哪了

关掉浏览器，留着终端。所有东西都在磁盘上：

```
~/.dsh/
  settings.yaml                 # 界面设置、默认预设、provider
  .credentials.yaml             # 你在界面里设的 key（如果设了）
  profiles/web/                 # 你刚启动的 profile（package.json + cordis.patch.yml）
  sessions/--tmp-first-hour-tally--/session-<id>/session.jsonl.zstd
  storages/                     # 工作区注册表、投影缓存
```

会话日志是只追加的 JSONL，以若干段连续的 Zstandard 帧存储。左边栏的会话列表是从这些文件推导出来的，所以重启之后会话还在，同一个工作区里点 "New Session" 会在旁边再开一个新的。右上角的 `Session log ↓`（或者在输入区里敲 `/export`）会把会话打成一个 ZIP 下载，原始日志在里面。

在跑 dsh 的那个终端里按 Ctrl-C，服务器就停了。下次还是同一条命令；只要 npx 缓存还在，就不会再下载一遍。

## 9. 坏了怎么办

上游 Discussions 第一天刷屏的是两类失败，这里各有一页：

- 启动时刷出一整屏 `AggregateError`（Discussions #2056、#1197、#2015）——[fix/boot-fails-with-a-wall-of-aggregateerror.zh.md](../fix/boot-fails-with-a-wall-of-aggregateerror.zh.md)。一句话版本：只读最里层那个 `[cause]`，其余整屏都可以不看。
- 所有 `/api/*` 请求都返回 403（Discussions #313、#128、#2009、#2030）——[fix/every-api-call-returns-403.zh.md](../fix/every-api-call-returns-403.zh.md)。一句话版本：你不是从 loopback 打开的。

## 接下来

- 你从 Claude Code 或 Codex 过来：[coming-from-claude-code.zh.md](coming-from-claude-code.zh.md) 把你已经会的每个概念都对应了一遍。
- 你想要广度：官方指南（`docs/user/guide/`），以及中文的 Electricitysheep/dsh-handbook。策展不是零和游戏。
- 写插件，以及把 dsh 跑给别人用，是这个仓库接下来的两条线。现在还没写；README 上写着今天已经有什么。
