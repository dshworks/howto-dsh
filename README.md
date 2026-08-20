# howto-dsh

Verified field notes for [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness) (`dsh`), DeepSeek's agent harness. Traps, skills, hooks, profiles.

**Every claim here was tested against a real run, with source paths included so you can re-verify.** Each page states the version and date it was verified against; this README's notes are from `0.1.0-rc.5` (2026-08-13), the pages below from `0.1.0-rc.6` (2026-08-15). dsh is a developer preview and the team promises compatibility-breaking changes, so treat everything here as dated, not eternal.

PRs with verified corrections welcome. Not affiliated with DeepSeek.

<sub>AI agents / LLMs: the org index lives at <a href="https://dshworks.github.io/llms.txt">dshworks.github.io/llms.txt</a>.</sub>

## Pages

**start/** — one sitting each, run every command.

| | English | 中文 |
|---|---|---|
| Your first hour: boot, a real repo with two failing tests, the diff, the trajectory, the bill | [first-hour.md](start/first-hour.md) | [first-hour.zh.md](start/first-hour.zh.md) |
| Coming from Claude Code or Codex: what carries over, what is renamed, what does not exist | [coming-from-claude-code.md](start/coming-from-claude-code.md) | [coming-from-claude-code.zh.md](start/coming-from-claude-code.zh.md) |

**fix/** — one symptom per page, reproduced before it was written.

| | English | 中文 |
|---|---|---|
| Boot fails with a wall of `AggregateError` | [boot-fails-with-a-wall-of-aggregateerror.md](fix/boot-fails-with-a-wall-of-aggregateerror.md) | [.zh.md](fix/boot-fails-with-a-wall-of-aggregateerror.zh.md) |
| Every `/api/*` request returns 403 | [every-api-call-returns-403.md](fix/every-api-call-returns-403.md) | [.zh.md](fix/every-api-call-returns-403.zh.md) |

`build/` (writing a plugin) and `run/` (running dsh for other people) are the next two tracks. They are empty today; this table is the honest state of the repo.

## What changed in 0.1.0-rc.8

`0.1.0-rc.8` shipped 2026-08-19, two releases past the pages above. These four
notes are **read from the rc.8 source tree, not re-run on a machine** — that
distinction is the whole point of this repo, so it is stated rather than
blurred. The pages themselves still say rc.6 until somebody re-runs them.

- **`dsh web` now opens your browser.** It waits for the whole Loader tree to
  settle, then hands the canonical host URL to the OS. `--no-open` turns it
  off, and a non-empty `SSH_CONNECTION` or `SSH_TTY` suppresses it
  automatically, because over SSH the forwarded address belongs to your client.
  A failed handoff prints why and leaves the server running.
  (`apps/cli/reference/README.md`)
- **The Claude Code and Codex subagents are separate optional Bundles now.**
  `dsh plugin --profile <name> add @deepseek-ai/dsh-subagent-codex` — and the
  trap: **bundle membership is fixed at Profile start.** Adding or removing one
  changes the manifest on disk while the running Profile keeps the set it
  booted with, so you must restart that Profile. Ordinary edits to the Profile
  or to the home `cordis.patch.yml` still hot-reload; this one does not. On the
  next start each Bundle registers a *dormant* provider, and a copied Preset
  still has to enable the matching tool row itself.
  (`apps/cli/reference/README.md`, `packages/subagent/subagent-claude-code/README.md`)
- **`SubagentReportDelivery` renamed `'wakeup'` to `'next-step'`**, and the
  semantics moved with it: it now uses `Agent.steer()`, waking an idle parent
  or joining a running parent's nearest step boundary. Anything that passed the
  string `'wakeup'` breaks. (`docs/subsystems/subagent.md`)
- **The SQLite session store changed format incompatibly** for read, write, and
  fork performance. Budget for it before upgrading a machine that has sessions
  you care about. (release notes; `docs/subsystems/persistence.md`)

Two things the pages below depend on that did **not** move between rc.6 and
rc.8, checked file by file: the `/api` browser-trust fence
(`packages/client/connection/src/{index,api-request-trust,loopback-hostname}.ts`
are byte-identical) and the bundle-vs-plain-dependency rules in
`docs/user/develop/basic/publish.md`. So the 403 page and the composition model
below still hold.

## Start in 60 seconds

```sh
npx @deepseek-ai/dsh web
```

The Web UI comes up at `http://127.0.0.1:3080` — and from rc.8 on, in your
browser, unless you pass `--no-open`. Set the API key in Settings > Models
(hot-applies, no restart). To sanity-check a composition without booting:
`dsh --profile <name> --dump-config`.

## When to use, when to skip

Use these notes when a tutorial fails and you suspect the harness changed underneath it, or before you ship your first bundle. Skip them if you want stable reference documentation: that is the [official docs'](https://deepseek-harness.github.io/deepseek-harness/en/guide) job, and nothing here is guaranteed past the version each claim was verified against.

## The composition model in 30 seconds

- Everything is a Cordis plugin. A **bundle** is what you author and distribute (npm package with `dsh.bundle` in package.json); a **profile** is what a user boots (`dsh --profile <name>`). Nothing is both.
- Patch layers apply in order: bundle patches, then profile `cordis.patch.yml`, then `$DSH_HOME/cordis.patch.yml`, then each `--patch` flag. Later wins per row.

## Traps

1. **The `.dsh-plugin` manifest format is dead.** Removed 2026-08-09 with no migration (`.agents/notes/implemented/simplification/2026-08-09-remove-repository-plugin.md`). Any tutorial teaching `.dsh-plugin` or `dsh-plugin-prepare` predates the removal. Current path: `dsh.bundle.patch` in package.json.
2. **`--patch` paths must be absolute.** A relative path fails (`docs/user/develop/basic/index.md`).
3. **Git installs execute code on your machine at install time, outside any sandbox.** The docs say so verbatim. Authors must ship a `prepare` script; users must allowlist the build via `allowBuilds` in the profile's `pnpm-workspace.yaml`, and should pin a commit SHA (`docs/user/develop/basic/publish.md`). This is not a corner case: of the 18 bundle-format plugins verified by [awesome-dsh-plugins](https://github.com/dshworks/awesome-dsh-plugins) on 2026-08-13, only 4 are on npm; the rest install via `dsh plugin --profile <p> add github:owner/repo` (a `#path:/subdir` suffix reaches monorepo subpackages).
4. **A patch replaces a row's entire `config`, with no deep-merge.** Overriding one field means restating the row's whole config.
5. **A package without `dsh.bundle` installs as a plain dependency** and activates no layer; you get a warning, not an error.
6. **cordis.yml YAML tags:** `!!js` is allowed only under a plugin's `config` and an entry's `disabled`. Never `!js`, never elsewhere.
7. **Repo descriptions lag migrations; trust the manifest.** Seen in the wild 2026-08-13: `AshesofPlato/whale-girl`'s description still teaches the removed `.dsh-plugin` + config.yaml install while the repo itself already migrated to `dsh.bundle` (its README installs via `dsh plugin add github:...`). Verify against package.json, not the repo card.
8. **`dsh.plugin.json` is not a harness manifest.** The `omdsh-dev` plugin family ships one in every repo root; nothing in the harness source reads that file. It is third-party metadata for better-sidebar's own tab registry. The real install path is still `dsh.bundle` in package.json.
9. **npm versions are fragmented across the monorepo.** The harness repo tags `0.1.0-rc.5` while published subpackages disagree (`dsh-agent@0.1.0-rc.6`, `dsh@0.0.1-rc.5`, Cordis at 4.x). Anchor compat claims to the harness release tag, not to whichever dependency version npm shows you.

## Skills: Claude-compatible

dsh reads Anthropic-format `SKILL.md` skills, so skills written for Claude Code work as-is. Discovery roots, highest rank first (`packages/skill/skill-filesystem/README.md`):

| Rank | Path |
|---|---|
| 100 | `<projectRoot>/.dsh/skills` |
| 200 | `<projectRoot>/.agents/skills` |
| 300 | `Config.customSkillDirs` |
| 400 | `<dshHome>/skills` |
| 500 | `~/.agents/skills` (or `$DSH_AGENTS_HOME`) |

Rules: `<name>/SKILL.md` directory bundles or flat `<name>.md` files; **no nested discovery**. Frontmatter needs `name` + `description` (optional `whenToUse`, `disable-model-invocation`, `user-invocable`). Names must be kebab-case; a camelCase name silently drops the whole skill. Roots are live-watched; a skill authored mid-session appears immediately.

## Bring your Claude Code hooks

dsh ships an official bridge that runs an existing `hooks.json` faithfully:

```yaml
- dsh-hooks-claude-code:
    configPath: ./.claude/hooks.json
    pluginRoot: ./.claude/plugins/my-plugin
    projectDir: .
```

Supported events map to harness points: `SessionStart`, `UserPromptSubmit`, `PreToolUse`, `PostToolUse`, `Stop`, `SubagentStart`, `SubagentStop`. `${CLAUDE_PLUGIN_ROOT}` / `${CLAUDE_PROJECT_DIR}` substitution works, and `CLAUDE_PROJECT_DIR` is exported to hook processes.

Limits (as shipped): only `type: "command"` shell hooks run; `http` / `mcp_tool` / `prompt` / `agent` hooks are parsed and skipped with a warning. `updatedInput` rewrites are logged but not honored. Config is process-level (one `hooks.json` per run, not per session). Default timeout 10 minutes.

A Codex bridge exists too: regex-only matchers, snake_case payloads, only blocking decisions honored.

Native alternative: the harness's own typed interception points (`tools/pre-execute`, `ctx.tools.guard()`, `agent/turn-stopping`, and friends) are the canonical surface; the bridges exist for compatibility (`packages/hooks/README.md`).

## Themes: the gap

A theme registration API exists (`packages/client/ui-theme`), but third-party theme ids don't persist to settings: built-in preference is Light/Dark/System only, and there's no validation that an override set is complete. The community workaround is shipping skins as plugins that re-register at load (see `zhu1090093659/dsh-web-ui`). Know this before promising a user their theme survives a restart. For what exists today, see [awesome-dsh-themes](https://github.com/dshworks/awesome-dsh-themes).

## Sandboxing note

`@deepseek-ai/node-addon-landlock-run` is a standalone ~300-line C11 Landlock launcher (static musl), published separately and usable outside dsh for any harness that runs untrusted commands. Fail-closed by design: no platform package means the probe reports `unusable` and the consumer falls closed. Exit code 125 means launcher failure, though a child can also legitimately return 125.

## Links

- Official: [repo](https://github.com/deepseek-ai/deepseek-harness) · [docs](https://deepseek-harness.github.io/deepseek-harness/en/guide) · [landing](https://www.deepseek.com/harness/en/) · [Cordis paper](https://github.com/cordiverse/paper) · [Discord](https://discord.gg/Ycq5dCaS4)
- Community: [dshworks.github.io](https://dshworks.github.io) · [awesome-dsh-plugins](https://github.com/dshworks/awesome-dsh-plugins) · [awesome-dsh-themes](https://github.com/dshworks/awesome-dsh-themes)
- Badge: the official opt-in "powered by dsh" badge lives at `packages/skill/skill-badge` (brand blue `#4D6BFE`; don't restyle it, the asset says so).

## License

[CC BY 4.0](LICENSE). Attribution: dshworks/howto-dsh.
