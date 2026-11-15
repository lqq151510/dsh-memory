---
id: "9e973980-e800-4287-a74a-a9cd8d5112d3"
title: "dsh-superagent-lite 二期：接真实 DSH 事件（live 总线）与热重载接入流程"
description: "dsh-superagent-lite 第二阶段：从 seed 模拟器改为订阅真实宿主事件（inbox/subagent/error/tools/result）的映射规则、live 计数器语义、触发器 id 约定、插件 enable 的正确 entryId 与客户端刷新验证流程，以及仍未实现的部分。"
status: "active"
created_at: "2026-11-02T17:04:58.220Z"
updated_at: "2026-11-02T17:04:58.220Z"
content_hash: "e9802e7322a96377fa3c020f36efc61db99e8ea3a9bd3fa845dac7a0f33b7924"
source_paths:
  - "plugins/dsh-superagent-lite/index.js"
  - "plugins/dsh-superagent-lite/client.js"
  - "plugins/dsh-superagent-lite/test/smoke.mjs"
  - "plugins/dsh-superagent-lite/cordis.patch.yml"
  - "plugins/dsh-superagent-lite/README.md"
  - ".dsh/profiles/desktop/package.json"
  - ".dsh/profiles/desktop/cordis.patch.yml"
session_ids:
  - "9847b022-e6fd-4197-8dae-50b47bcab65b"
memory_body_ids:
  []
---

# dsh-superagent-lite 二期：接真实 DSH 事件（live 总线）

一期（安装、总线/Router/面板骨架、HTTP 数据通道、逻辑 Worker）见文档 `e19c02d2-2a56-4f20-9ce8-b38172ef8952`。本文只记二期：把「seed 造数据的面板模拟器」改成订阅真实宿主事件的接入层，以及本地热重载接入的踩坑流程。

## 触发这次改造的判断

一期面板里所有 30 条触发器都是 `seed` 造出来的，面板状态与 DSH 实际使用完全无关——用户当场指出「光做出来没融入项目」。结论：面板必须由真实事件驱动，否则仿真数据会让人误判功能已可用。

## 真实宿主事件 → 总线对象映射

宿主半边在 `apply()` 里订阅（用 `ctx.on`），订阅名单实测为：
`api-session/added`、`api-session/removed`、`api-session/status`、`agent/status`、`agent/inbox/inserted`、`agent/error`、`subagent/start`、`subagent/end`、`tools/result`。

| 真实事件 | 映射为 | 面板可见效果 |
|---|---|---|
| `api-session/added`、`agent/status` | Worker（真实会话） | 会话数 / running 数随开关会话变化 |
| `agent/inbox/inserted` | Trigger | 当前 App 里发出的每条消息进总线，先过 §3.1 程序层 Fast Path，再按「已进入该会话 inbox」记 direct 直通 |
| `subagent/start`、`subagent/end` | Worker（`·sub`） | 模型开子代理时面板真的多一个 worker |
| `agent/error` | Trigger（`source=error`） | 进 Router 队列待认领；30s 内重复报错只累加次数，不刷屏 |
| `tools/result` | 仅计数 | 每次工具调用都入队会淹没信号，因此只累计到 live.toolCalls |

约定：触发器 id `l*` = 真实事件，`t*` = 模拟注入，一眼区分真假数据。面板加了「暂停接入」（paused）开关。

## live 段与观测口径

`summary()` 增加 `live: { paused, events, errors, toolCalls, sessions, running, listenerErrors }`；`/superagent/api/state` 与 `superagent` 工具的 `status` 同源。`listenerErrors` 是监听器自身异常计数，用来区分「宿主没事件」和「订阅写错」。worker 投影形如 `54c197ad:subagent:idle`（短 id:类型:状态）。

## 接入与热重载流程（踩坑记录）

- 注册行在 `.dsh/profiles/desktop/cordis.patch.yml` 的 `insert:` 下：`id: superagent-lite`、`name: dsh-superagent-lite`，配置项 `sources`、`mutePatterns`、`defaultSource`、`maxTriggers`；profile `package.json` 依赖为 `link:/Users/liuyongze/plugins/dsh-superagent-lite`。
- plugin-manager 的 enable 目标必须写**运行时 entryId**：用模块名 `dsh-superagent-lite` 或 `superagent-lite` 都会返回 `unknown-plugin`，正确值是 `include:superagent-lite`（先 disable 再 enable 即热重载宿主半边，返回 `application: applied`）。
- 反复热重载会带出无关告警：`experimental-inspector: globalThis.fetch is an accessor and cannot be observed safely`，不影响本插件。
- 客户端半边是 `client.js`（约 22.6KB，`index.js` 约 25.9KB），宿主的热重载不会重新加载它，**必须刷新页面**才生效。这是「后端已 applied 但页面没变」的常见误判点。

## 验证结果（分层，勿混报）

- 静态：`node --check` 通过；版本 0.2.0，README 同步。
- 单测：`test/smoke.mjs` 8/8 通过。桩 `ctx` 需要提供 `tools.register`、`webServer.register`、`effect`、**`ctx.on`**（缺 `ctx.on` 时首轮报 `TypeError: ctx.on is not a function`）；inject 为 `['tools','webServer']`。一期断言里 worker 列表从 `['worker-1']` 改带状态后缀，属实现变更导致的用例更新，不是新功能。
- 运行时：真实子代理拉起的实测快照 `live: events=2 toolCalls=6 sessions=2 listenerErrors=0`，workers = 子代理 + 当前会话，triggers = `l1 im direct`（自己发的消息）+ `l2 agent direct`（子代理结束插回的 Background subagent 通知）——两条均为真实事件。
- 未验证：页面视觉效果（无可驱动浏览器）。

## 仍未实现（明确边界）

- Router 的「分发队首」目前只做记账认领（`router:take`），**不会创建真实 Worker 会话**。
- 未接真实 IM / GitHub 事件源；Watchdog 未做。
- 下一步三选一：① 让 Router 真正开子会话（接 subagent runtime，会自动烧 token，需先划定授权范围）；② 接 Watchdog 定时自查；③ 先用几天真实数据再决定。
