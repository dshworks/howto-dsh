# 所有 `/api/*` 请求都返回 403

[English](every-api-call-returns-403.md) | 中文

基于 `@deepseek-ai/dsh` 0.1.0-rc.6，2026-08-15 实测：用 `curl` 打一个正在跑的 `dsh web`，同时对着实现代码核对。

## 症状

页面能打开，但控件都没有反应。浏览器控制台报 `transport failure for /api/host.describe`（或者 `host.pickDirectory`、`host.listDirectory`），HTTP 403，响应体就是 `forbidden`。对应 Discussions #313、#128、#2009。

另一种症状，同一道关卡：聊天能用，Settings 报 403，有些设置怎么改都存不下来（Discussion #2030）。那是下面"特权方法"那条规则，不是装坏了。

## 原因

`/api` 下的每个请求在被分发之前，都要先过一道浏览器信任检查。它防的是 DNS rebinding 和跨站请求，不是身份认证。三条规则：

1. **`Host` 必须是"我们的"。** 要么是 loopback——`localhost`、`[::1]`，或者 `127.0.0.0/8` 里的任意 IPv4 地址——要么命中你用 `--trusted-host` 声明过的 authority（带端口时精确匹配 `host:port`，不带端口时匹配任意端口）。**dsh 实际监听的端口从来不参与比较。** 服务器跑在 8099，`Host: 127.0.0.1:9999` 照样通过；`Host: mybox.local:8099` 不行。
2. **浏览器带了 `Origin` 时，它必须和 `Host` 的 authority 一致。** 端口不同也算不一致。明确标了 `Sec-Fetch-Site: cross-site` 的请求按同一条规则拒绝。
3. **有 15 个方法即使配了 `--trusted-host` 也只认 loopback**，因为它们会用一个空的信任列表再过一次检查：`settings.describe/openDocument/update/replace/mutate`、`credentials.describe/set/unset`、`agentPreset.read/copy/openDocument/remove`、`host.pickDirectory`、`host.openPath`、`llm.discoverModels`。#2030 就是这么来的：通过代理用真实域名访问时，聊天能用，Settings 不能用。

   这是一份明确的清单，不是通配。`llm.providers` 和 `llm.models` 是特意留在外面的，好让局域网客户端的模型选择器还能用；`agentPreset.list` 和"切换预设"也没有被固定——因为 `session.create` 本来就接受 `agentPreset` 参数，而任何能开会话的调用方，本来就已经能以这个进程的身份执行命令了。

对着 `127.0.0.1:8099` 上的服务器实测。记得发 `Content-Type: application/json`，否则会先撞上媒体类型检查（415），根本走不到信任检查这一步：

```sh
J='Content-Type: application/json'
curl -so /dev/null -w '%{http_code}\n' -X POST localhost:8099/api/host.describe -d '{}'                              # 415，撞的是另一道检查
curl -so /dev/null -w '%{http_code}\n' -X POST localhost:8099/api/host.describe -H "$J" -d '{}'                      # 200
curl -so /dev/null -w '%{http_code}\n' -X POST localhost:8099/api/host.describe -H "$J" -H 'Host: 127.0.0.1:9999' -d '{}'   # 200，监听端口不参与比较
curl -so /dev/null -w '%{http_code}\n' -X POST localhost:8099/api/host.describe -H "$J" -H 'Host: mybox.local:8099' -d '{}' # 403
curl -so /dev/null -w '%{http_code}\n' -X POST localhost:8099/api/host.describe -H "$J" -H 'Origin: http://127.0.0.1:3080' -d '{}' # 403，Origin 和 Host 对不上
curl -so /dev/null -w '%{http_code}\n' -X POST localhost:8099/api/host.describe -H "$J" -H 'Sec-Fetch-Site: cross-site' -d '{}'     # 403
```

自带的 web 服务器既没有 TLS 也没有鉴权。这道检查是一条可达性策略，替的是目前还不存在的身份认证。

## 修

- **本机用**：就打开 dsh 打印的那个地址（`http://127.0.0.1:3080`）。坏事的是用主机名或局域网 IP 去访问；换一个 loopback 端口本身没问题，只要 `Host` 和 `Origin` 对得上。会改写 `Origin` 的浏览器扩展一定 403。
- **走代理或容器映射**：要声明的是**浏览器实际输入的那个 authority**，不是 dsh 监听的那个——`dsh web --trusted-host mybox.local:8080`（可重复）——并且让代理原样透传 `Host`，这样 `Host` 和 `Origin` 才继续一致。
- **服务器在远端，而且想让 Settings 也能用**：光配 trusted host 不够，因为整个配置面被钉在 loopback 上。改用隧道：`ssh -L 3080:127.0.0.1:3080 user@box`，然后在本地打开 `http://127.0.0.1:3080`。这样检查看到的 `Host` 和 `Origin` 都是 loopback，特权方法也能通过。
- `--host 0.0.0.0` 是 CLI 故意拒绝的：自带服务器没有 TLS，也没有鉴权。

## 出处

`packages/client/connection/src/index.ts`（`PRIVILEGED_METHODS`——权威清单，每一项为什么在里面、为什么不在里面都写了）；`packages/client/connection/src/api-request-trust.ts` 和 `loopback-hostname.ts`（Host / Origin / Fetch Metadata 规则，以及 127/8 的判定）；`packages/client/connection/README.md`；`apps/cli/reference/README.md`（`--trusted-host`、不支持 `--host 0.0.0.0`）；`.agents/notes/implemented/architecture/2026-07-28-api-browser-trust-boundary.md`（最初的决策记录，其中关于 `0.0.0.0` 部署的说法早于当前 CLI 行为）。
