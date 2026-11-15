---
id: "c2665f41-c969-4053-bb1a-974d89149a67"
title: "bili 上游熔断横幅（左上角提示条）判读与 DNS 取证流程"
description: "bili/billion-context 「N 个上游不可用」横幅的端点、日志指纹、只读取证顺序、处置原则（刷新而非重启），以及 DNS 抖动连带打断 dsh-superagent 自检的教训。"
status: "active"
created_at: "2026-11-03T10:58:03.058Z"
updated_at: "2026-11-03T10:58:03.058Z"
content_hash: "e341b5b44a552f64493840f5e7137cc455d65f14a037cdb46237d21589d3c166"
source_paths:
  - ".local/state/billion-context/bili.log"
  - ".config/billion-context/billion-context.json"
  - ".dsh/profiles/desktop/node_modules/billion-context/dist/index.js"
session_ids:
  - "8d0ee3a5-a166-4c1b-a923-34db50818ef2"
memory_body_ids:
  []
---

# bili（billion-context）左上角「N 个上游不可用 · 约 1s 后重试」提示条：判读与取证流程

结论先行：DSH Web GUI 左上角那条提示条**不是 DSH 报错**，是 bili 的**上游健康 / 熔断横幅**，数据源为 `GET /__bili/overview` 的 `alerts[]`。「约 1s 后重试」是熔断状态下的下一次探测倒计时。遮挡标题栏只是 toast 定位问题，界面本身没坏。

## 关键端点（实测版本 v=0.1.210，commit 0e88a073）

| 用途 | 端点 | 实测结果 |
|---|---|---|
| bili 管理面 | `http://127.0.0.1:18787` | 正常 |
| 提示条数据源 | `GET /__bili/overview` → `alerts[]` | 恢复后为 `[]` |
| 上游视图 | `GET /__bili/upstreams` | 恢复后为 `{"upstreams":[]}` |
| 熔断视图 | `GET /__bili/circuits` | **该版本不存在**，返回 `{"error":{"type":"not_found","message":"no such management endpoint",...}}` |
| 生效配置 | `GET /__bili/config` | 见下 |
| 转发口 | `http://127.0.0.1:18787/bili/<完整上游 URL>` | 可用 |

`/__bili/overview` 顶层键（实测）：`overview`, `version`, `origin`, `commit`, `diskVersion`, `stale`, `lastInstall`, `updateReady`, `autoRestartOnUpdate`, `advisory`, `resolverWedge`, `updatePolicy`, `pluginAdvisories`, `hostPluginNotices`, `pckNotice`, `inFlight`, `blindTunnels`, `conflicts`, `passthrough`, `alerts`, `pckSuppressions`, `storage`, `attribution`。判读健康只看 `alerts` / `blindTunnels.total` / `resolverWedge` / `inFlight`。

## 本次实际根因：瞬时 DNS 解析失败

日志 `.local/state/billion-context/bili.log` 中 18:22:55–18:23:04 连续命中：

```text
[sess=session-2f819830-…] [acp-proxy] upstream dns (pre-response network failure): fetch failed; retrying ...
Error: upstream request failed: kind=dns code=ENOTFOUND errno=-3008 syscall=getaddrinfo message=fetch failed <- getaddrinfo ENOTFOUND <host> url=https://<host>/
[tunnel] denied https://<host>/: cannot resolve tunnel destination api.commandcode.ai (ENOTFOUND): local name resolution failed and a fresh resolver thread also failed
```

即：解析 `api.commandcode.ai` 失败 → bili 重试 3 次 → 对该 host 熔断 → 弹提示条；同一会话的模型请求路径也走该 host，所以那一轮对话直接失败。相邻的 `[acp-registry] conflicting prices ...` 只是价格表冲突告警，与本问题无关。

## 复核顺序（只读，先取证再改配置）

1. `curl -sS -m 5 http://127.0.0.1:18787/__bili/overview`，看 `alerts`（含 `kind`/`host`/`hint`/`firstSeen`）。
2. `grep -a -nE 'upstream-reset|circuit OPEN|kind=dns|ENOTFOUND|upstream awaiting first byte' ~/.local/state/billion-context/bili.log | tail -30`。
3. 解析是否恢复：`dig +short api.commandcode.ai`（本次为 Clash fake-ip `198.18.0.132`，属正常）；系统 DNS 当时是 `114.114.114.114`。
4. 可达性 A/B：直连与 `-x http://127.0.0.1:7896` 各测一次，本次都是 `200`（1.38s / 1.36s）。**403 也代表可达，不等于故障。**
5. 经 bili 转发口请求同一 host：本次 `200`，0.69s。
6. 看 `forward POST → ...` 行是否恢复连续（恢复后每 20–30s 一条为正常）。

本次恢复后的完整健康证据：`alerts=[]`、`blindTunnels={"total":0,"hosts":{}}`、`resolverWedge=null`、`inFlight=0`、`advisory=null`、`conflicts.sessions=0`。

## 处置原则

- 瞬时 DNS 抖动：**刷新页面（Cmd+R）即可让提示条消失**，不要改 bili 配置。
- **不要重启 bili**（`node .../billion-context/dist/index.js start --host 127.0.0.1 --port 18787`）；重启会切断正在转发的会话，反而中断当前对话。
- 降低复发率只能动系统 DNS（`114.114.114.114` → `223.5.5.5` / `1.1.1.1`），属系统级改动，须先获用户同意。
- 只有持续熔断且确实要用该上游时，才去查 `.config/billion-context/billion-context.json`（`providers` 按 host 配 proxy）与 profile `cordis.patch.yml` 的账号 / `autoSwitch`，改前备份。

当前生效配置（`/__bili/config`）形态：`providers` 里 `https://chatgpt.com` 与 `https://api.openai.com` 各指向 `http://127.0.0.1:7896`，`upstreamProxy: null`、`upstreamProxyMode: "direct"`。配置里**没有** `api.commandcode.ai` 的定向代理项，它依赖系统 DNS 直连。

## 未验证的假设（勿当结论引用）

曾假设根因是「bili 的 `readWindowsSystemProxy()` 在非 win32 直接返回 `{enabled:false}`，只继承 `INHERITED_PROXY_VARS` 与 `BILI_*` 变量；而 dsh-codex-subscription 的 `resolveCodexOAuthProxy` 用 `scutil --proxy` 能读到 macOS 系统代理，两层认知不一致导致 direct」。本次实际故障是 DNS ENOTFOUND，该代理认知差异**未被证实为本次成因**，仅在出现「chatgpt.com 被判 direct 且长时间首字节超时」时作为排查方向复验（bundle 可用 `grep -a -o '/__bili/[a-z0-9/_-]*' .dsh/profiles/desktop/node_modules/billion-context/dist/index.js | sort -u` 枚举端点）。

## 连带影响：dsh-superagent 链路自检被打断

同一波 DNS 抖动打断了首次自检执行：README 的 commit 已在 worktree 生成但未 push、未开 PR。处理方式是重试 ticket（`{"retried":true}`），随后 `load.running=1`。

三个实例状态（`GET <instance>/setup` 与 `/load`）：3082 `kaixin`、3083 `python` setup 全通；3084 `personal-proxy` 一度报 `agentCliSignedOut`（`error connecting to api.github.com`），同属网络抖动，重试后恢复为 `lqq151510/personal-proxy` / agent `lqq151510`。教训：`agentCliSignedOut` + `api.github.com` 连接错误要先怀疑网络/代理，不要直接判为账号失效。

## 相关文件

- 生效配置：`.config/billion-context/billion-context.json`
- 日志：`.local/state/billion-context/bili.log`
- bili 安装：`.dsh/profiles/desktop/node_modules/billion-context/dist/index.js`（5.6MB 单行 bundle）
- profile 上层配置：`.dsh/profiles/desktop/cordis.patch.yml`
