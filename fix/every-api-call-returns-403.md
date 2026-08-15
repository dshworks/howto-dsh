# Every `/api/*` request returns 403

English | [中文](every-api-call-returns-403.zh.md)

Verified on `@deepseek-ai/dsh` 0.1.0-rc.6, 2026-08-15, by probing a running `dsh web` with `curl` and reading the fence implementation.

## Symptom

The page loads, but its controls do nothing. The browser console shows `transport failure for /api/host.describe` (or `host.pickDirectory`, `host.listDirectory`) with HTTP 403 and a body of `forbidden`. Discussions #313, #128, #2009.

Different symptom, same fence: chat works, Settings 403s, some settings never persist (Discussion #2030). That is the privileged-method rule in the Cause section, not a broken install.

## Cause

Every request under `/api` passes a browser-trust fence before it is dispatched — DNS-rebinding and cross-site defence, not authentication. Three rules:

1. **`Host` must be ours.** Either loopback — `localhost`, `[::1]`, or any IPv4 address in `127.0.0.0/8` — or an authority you declared with `--trusted-host` (exact on `host:port`; any port when the entry has no port). **The port dsh listens on is never compared.** `Host: 127.0.0.1:9999` passes against a server on 8099; `Host: mybox.local:8099` does not.
2. **`Origin`, when the browser sends it, must equal the `Host` authority.** A port mismatch is a mismatch. An explicit `Sec-Fetch-Site: cross-site` is rejected on the same rule.
3. **Fifteen methods stay loopback-only even with `--trusted-host`**, because they re-check the fence with an empty trust list: `settings.describe/openDocument/update/replace/mutate`, `credentials.describe/set/unset`, `agentPreset.read/copy/openDocument/remove`, `host.pickDirectory`, `host.openPath`, `llm.discoverModels`. This is #2030 — through a proxy on a real hostname, chat works and Settings does not.

   It is a list, not a wildcard. `llm.providers` and `llm.models` are deliberately left out so a LAN client's model picker still works; `agentPreset.list` and choosing a preset are not pinned either, because `session.create` already accepts an `agentPreset` and any caller who can start a session can already run commands as that process.

Probed against a server on `127.0.0.1:8099`. Send `Content-Type: application/json` or you hit the media-type guard (415) and never reach the fence:

```sh
J='Content-Type: application/json'
curl -so /dev/null -w '%{http_code}\n' -X POST localhost:8099/api/host.describe -d '{}'                              # 415 — wrong guard
curl -so /dev/null -w '%{http_code}\n' -X POST localhost:8099/api/host.describe -H "$J" -d '{}'                      # 200
curl -so /dev/null -w '%{http_code}\n' -X POST localhost:8099/api/host.describe -H "$J" -H 'Host: 127.0.0.1:9999' -d '{}'   # 200 — listening port not compared
curl -so /dev/null -w '%{http_code}\n' -X POST localhost:8099/api/host.describe -H "$J" -H 'Host: mybox.local:8099' -d '{}' # 403
curl -so /dev/null -w '%{http_code}\n' -X POST localhost:8099/api/host.describe -H "$J" -H 'Origin: http://127.0.0.1:3080' -d '{}' # 403 — Origin/Host disagree
curl -so /dev/null -w '%{http_code}\n' -X POST localhost:8099/api/host.describe -H "$J" -H 'Sec-Fetch-Site: cross-site' -d '{}'     # 403
```

The shipped web server has neither TLS nor auth. The fence is a reachability policy standing in for the authentication that does not exist yet.

## Fix

- **Local use:** open the URL dsh printed (`http://127.0.0.1:3080`). Reaching it by hostname or LAN IP is what breaks; a different loopback port is fine as long as `Host` and `Origin` agree. Browser extensions that rewrite `Origin` will 403 you.
- **Behind a proxy or a container mapping:** declare the authority *the browser types*, not the one dsh listens on — `dsh web --trusted-host mybox.local:8080` (repeatable) — and pass `Host` through the proxy unchanged so `Host` and `Origin` still agree.
- **Remote machine, and you want Settings to work:** a trusted host is not enough, because the configuration plane is pinned to loopback. Tunnel instead: `ssh -L 3080:127.0.0.1:3080 user@box`, then open `http://127.0.0.1:3080` locally. The fence then reads a loopback `Host` and `Origin`, so the privileged methods pass too.
- `--host 0.0.0.0` is rejected by the CLI on purpose: no TLS, no auth in the shipped server.

## Source

`packages/client/connection/src/index.ts` (`PRIVILEGED_METHODS` — the authoritative pinned list, with the reasoning for each inclusion and exclusion); `packages/client/connection/src/api-request-trust.ts` and `loopback-hostname.ts` (the Host/Origin/Fetch-Metadata rules and 127/8 classification); `packages/client/connection/README.md`; `apps/cli/reference/README.md` (`--trusted-host`, `--host 0.0.0.0` unsupported); `.agents/notes/implemented/architecture/2026-07-28-api-browser-trust-boundary.md` (the original decision; its `0.0.0.0` deployment wording predates the current CLI behaviour).
