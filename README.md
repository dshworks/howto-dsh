# howto-dsh

Verified field notes for [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness) (`dsh`).

Every claim below was tested against `deepseek-ai/deepseek-harness` at `0.1.0-rc.5` (2026-08-13) — source paths included so you can re-verify. dsh is a developer preview and the team promises compatibility-breaking changes, so treat everything here as dated, not eternal.

English first; 中文版 planned. PRs with verified corrections welcome.

## Quick facts

- Run: `npx @deepseek-ai/dsh web` → Web UI at `http://127.0.0.1:3080`. Set the API key in Settings → Models (hot-applies, no restart).
- Composition model: everything is a Cordis plugin. A **bundle** is what you author and distribute (npm package with `dsh.bundle` in package.json); a **profile** is what a user boots (`dsh --profile <name>`). Nothing is both.
- Patch layers apply in order: bundle patches → profile `cordis.patch.yml` → `$DSH_HOME/cordis.patch.yml` → each `--patch` flag. Later wins per row.
- Sanity-check a composition without booting: `dsh --profile <name> --dump-config`.

## Traps

1. **The `.dsh-plugin` manifest format is dead.** Removed 2026-08-09 with no migration (`.agents/notes/implemented/simplification/2026-08-09-remove-repository-plugin.md`). Any tutorial teaching `.dsh-plugin` or `dsh-plugin-prepare` predates the removal. Current path: `dsh.bundle.patch` in package.json.
2. **`--patch` paths must be absolute.** A relative path fails (`docs/user/develop/basic/index.md`).
3. **Git installs execute code on your machine at install time, outside any sandbox.** The docs say so verbatim. Authors must ship a `prepare` script; users must allowlist the build via `allowBuilds` in the profile's `pnpm-workspace.yaml`, and should pin a commit SHA (`docs/user/develop/basic/publish.md`).
4. **A patch replaces a row's entire `config`** — no deep-merge. Overriding one field means restating the row's whole config.
5. **A package without `dsh.bundle` installs as a plain dependency** and activates no layer; you get a warning, not an error.
6. **cordis.yml YAML tags:** `!!js` is allowed only under a plugin's `config` and an entry's `disabled` — never `!js`, never elsewhere.

## Skills — Claude-compatible

dsh reads Anthropic-format `SKILL.md` skills, so skills written for Claude Code work as-is. Discovery roots, highest rank first (`packages/skill/skill-filesystem/README.md`):

| Rank | Path |
|---|---|
| 100 | `<projectRoot>/.dsh/skills` |
| 200 | `<projectRoot>/.agents/skills` |
| 300 | `Config.customSkillDirs` |
| 400 | `<dshHome>/skills` |
| 500 | `~/.agents/skills` (or `$DSH_AGENTS_HOME`) |

Rules: `<name>/SKILL.md` directory bundles or flat `<name>.md` files; **no nested discovery**. Frontmatter needs `name` + `description` (optional `whenToUse`, `disable-model-invocation`, `user-invocable`). Names must be kebab-case — a camelCase name silently drops the whole skill. Roots are live-watched; a skill authored mid-session appears immediately.

## Bring your Claude Code hooks

dsh ships an official bridge that runs an existing `hooks.json` faithfully:

```yaml
- dsh-hooks-claude-code:
    configPath: ./.claude/hooks.json
    pluginRoot: ./.claude/plugins/my-plugin
    projectDir: .
```

Supported events map to harness points: `SessionStart`, `UserPromptSubmit`, `PreToolUse`, `PostToolUse`, `Stop`, `SubagentStart`, `SubagentStop`. `${CLAUDE_PLUGIN_ROOT}` / `${CLAUDE_PROJECT_DIR}` substitution works, and `CLAUDE_PROJECT_DIR` is exported to hook processes.

Limits (as shipped): only `type: "command"` shell hooks run — `http` / `mcp_tool` / `prompt` / `agent` hooks are parsed and skipped with a warning; `updatedInput` rewrites are logged but not honored; config is process-level (one `hooks.json` per run, not per session); default timeout 10 minutes.

A Codex bridge exists too: regex-only matchers, snake_case payloads, only blocking decisions honored.

Native alternative: the harness's own typed interception points (`tools/pre-execute`, `ctx.tools.guard()`, `agent/turn-stopping`, …) are the canonical surface — the bridges exist for compatibility (`packages/hooks/README.md`).

## Themes: the gap

A theme registration API exists (`packages/client/ui-theme`), but third-party theme ids don't persist to settings — built-in preference is Light/Dark/System only, and there's no validation that an override set is complete. The community workaround is shipping skins as plugins that re-register at load (see `zhu1090093659/dsh-web-ui`). Know this before promising a user their theme survives a restart.

## Sandboxing note

`@deepseek-ai/node-addon-landlock-run` is a standalone ~300-line C11 Landlock launcher (static musl), published separately and usable outside dsh for any harness that runs untrusted commands. Fail-closed by design: no platform package → probe reports `unusable` → consumer falls closed. Exit code 125 means launcher failure — but a child can also legitimately return 125.

## Links

- Official: [repo](https://github.com/deepseek-ai/deepseek-harness) · [docs](https://deepseek-harness.github.io/deepseek-harness/en/guide) · [landing](https://www.deepseek.com/harness/en/) · [Cordis paper](https://github.com/cordiverse/paper) · [Discord](https://discord.gg/Ycq5dCaS4)
- Badge: the official opt-in "powered by dsh" badge lives at `packages/skill/skill-badge` (brand blue `#4D6BFE`; don't restyle it — the asset says so).

## License

CC BY 4.0. Attribution: dshworks/howto-dsh.
