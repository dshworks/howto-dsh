# Boot fails with a wall of `AggregateError`

English | [中文](boot-fails-with-a-wall-of-aggregateerror.zh.md)

Verified on `@deepseek-ai/dsh` 0.1.0-rc.6, 2026-08-15. Both wall-producing failures were reproduced on this machine; the quoted errors are verbatim.

## Symptom

`dsh web` (or `dsh --profile <name> ...`) exits non-zero after roughly 40-60 lines beginning with:

```
Error: dsh: plugin tree failed to load: failed to apply loader entry include (cordis:include): ...
AggregateError: loader entries failed to apply
    at EntryGroup.update (.../cordis-plugin-loader/lib/index.js:91:35)
```

then nested `[cause]:` blocks. It usually worked yesterday; the common triggers are an agent editing the profile, a plugin you installed, or another process taking the port (Discussions #2056, #1197, #2015).

## Cause

dsh composes a profile fail-closed: one plugin row that cannot load fails the include, then the tree, then boot. The outer stack is propagation — the innermost `[cause]` is the only actionable line. Two produce this wall:

| Innermost line | Meaning |
|---|---|
| `failed to import loader entry oops (dsh-does-not-exist): Cannot find package 'dsh-does-not-exist' imported from ~/.dsh/profiles/headless/` | a row names a package that is not installed in *that* profile: a typo, a package that renamed itself, a `link:` to a directory that moved, or a `dsh plugin remove` that left the row behind |
| `failed to apply loader entry webserver (@deepseek-ai/dsh-host-webserver): listen EADDRINUSE: address already in use 127.0.0.1:3080` | another process holds the port, often a previous `dsh web` you thought you stopped |

A bad row can come from any of these layers, in application order: each bundle listed under `dsh.profile.bundles` in `profiles/<name>/package.json`, then `profiles/<name>/cordis.patch.yml`, then `$DSH_HOME/cordis.patch.yml`, then each `--patch` file.

Not this wall: a single short line like `dsh: failed to parse overlay ~/.dsh/profiles/headless/cordis.patch.yml: YAMLException: unexpected end of the stream within a flow collection (4:1)`. That is malformed YAML, and it already names the line. Fix it there — a patch file is a top-level YAML array whose empty state is `[]`.

## Fix

1. Read the innermost `Cannot find package` or `EADDRINUSE` line. Everything above it is the same failure being re-thrown.
2. **Port:** `dsh web --port 3081`, or find and stop the old process (`lsof -i :3080`).
3. **Missing package:** install it into the profile that failed (`dsh plugin --profile <name> add <package>`), delete the hand-written row that names it, or drop the bundle that contributes it. Hand-written rows live in the profile patch, the home patch, or a `--patch` file; bundles are listed in the profile's `package.json`. `dsh plugin --profile <name> remove <package>` reconciles the bundle list but leaves rows you wrote by hand alone.
4. **Prove the installation itself is sound** with a throwaway home on a free port: `DSH_HOME="$(mktemp -d)" dsh web --port 3099`. A fresh home has only the shipped bundles and empty user patches. If that boots, the fault is in your own `$DSH_HOME` or in something it references — not in the dsh you installed.
5. **Read the composed tree without booting it:** `dsh --profile <name> --dump-config`. Each row is printed under a comment naming the layer that contributed it and every overlay that changed it (`# == @deepseek-ai/dsh-base, patched by @deepseek-ai/dsh-headless`), so a foreign row stands out. Patch targets that matched nothing are reported on stderr, which is exactly the row you are hunting.

## Do not

- Do not `rm -rf ~/.dsh` to make it go away. Your sessions, credentials, and presets are in there. Move it aside (`mv ~/.dsh ~/.dsh.bak`) if you must, then copy back `sessions/` and `.credentials.yaml`.
- Do not ask an agent to repair the profile that hosts it. Edit the file yourself.

## Source

`apps/cli/reference/README.md` (layer order, `--dump-config` semantics, unmatched patch targets on stderr); `packages/host/webserver/README.md` ("A listen failure (EADDRINUSE...) throws out of activation and rejects Loader composition"); `docs/user/develop/basic/publish.md` (bundle vs plain dependency).
