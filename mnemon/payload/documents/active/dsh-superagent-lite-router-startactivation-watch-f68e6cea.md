---
id: "f68e6cea-71a8-4231-a63b-dba6577e70dd"
title: "dsh-superagent-lite 三期：Router 派真子会话（startActivation 契约）与 Watchdog 落地"
description: "三期改造记录：本机 DSH 的 ctx.subagents 只暴露 startActivation(spec)（无 start），据此让 Router 用 spawn provider 真开子会话并按 delivery:'parent' 回灌；同时落地定时 Watchdog，以及护栏参数、验证证据与仍缺的自动决策循环。"
status: "active"
created_at: "2026-11-02T23:56:46.982Z"
updated_at: "2026-11-02T23:56:46.982Z"
content_hash: "cb4848c837d43ccc87c039f29b24eaa32f5811253e2503c532328d7b40d5814d"
source_paths:
  - "plugins/dsh-superagent-lite/index.js"
  - "plugins/dsh-superagent-lite/client.js"
  - "plugins/dsh-superagent-lite/test/smoke.mjs"
  - "plugins/dsh-superagent-lite/README.md"
session_ids:
  - "cff96e3b-aa3b-46da-8ae6-de988592b055"
memory_body_ids:
  []
---

# dsh-superagent-lite 三期：Router 派真子会话与 Watchdog

一期的插件骨架见 `e19c02d2-2a56-4f20-9ce8-b38172ef8952`，二期真实事件接入见 `9e973980-e800-4287-a74a-a9cd8d5112d3`。本文只记三期：把「Worker 是逻辑实体」换成真开 DSH 子会话，并补上 Watchdog。

## 卡住几小时的根因：子代理 API 版本对不上

本机安装版（`/Applications/DeepSeek Harness.app/Contents/Resources/app.asar`）里，`ctx.subagents` **没有 `start()`**，只有 `startActivation(spec)`。从 asar 解出的 `dsh-subagent` 契约（`typert.host.js` 类型清单 + `lib/index.js` 实现）确认的调用形态：

```ts
interface SubagentActivationSpec {
  provider: string          // 本机可用：'spawn' | 'fork'
  label: string
  childId?: SessionId
  request: Omit<SubagentStartRequest, 'label' | 'signal'>  // { prompt, parent, cwd?, agentPreset?, outputSchema?, maxDepth? }
  signal: AbortSignal
  delivery: 'parent' | 'caller'
}
```

可行调用（已在插件里生效）：

```js
ctx.subagents.startActivation({
  provider: 'spawn',
  label: 'router-worker',
  request: { prompt, parent },          // parent 取自 ctx.agents.get(sessionId)
  signal: controller.signal,
  delivery: 'parent',                   // 子会话结果按 parent 投递回灌，形成 inbox 事件
})
```

要点：
- `parent` 必须是活的 Agent 句柄，走 `ctx.agents.get(sessionId)` 获取，不要依赖事件缓存里的快照。
- `delivery: 'parent'` 是闭环关键：子会话回传的结果会成为 Router 的 inbox 事件（live id `l*`，`source=agent`，`route=direct`），不需要另建结果通道。
- 契约还给出 `SubagentCapabilities { agentOptions, agentPreset, outputSchema, depthLimit }` 与 catalog 事件（`one-shot` / `external` / `continuable`），后续若要接 fork 或续聊模式按此扩展。

## 护栏（默认不自动烧 token）

- `dispatch.auto = false`：只有手动点面板按钮或显式调工具才派活。
- `maxPerHour: 3` 限频；`timeoutMs: 10min`，到点用 `AbortController` abort。
- 派活成功才计入 `counters.dispatched`，失败回滚不记账。

## Watchdog

主系统之外的定时器（默认 5 分钟，`unref` 不阻止进程退出），查三类异常：监听器报错、队列积压 > 20、标记 running 但 15 分钟无事件。面板有独立卡片，也支持手动触发。首轮实测 `{runs: 1, findings: []}`。

## 验证证据（真跑通，非记账）

`superagent spawn`（trigger id `t8`）返回 `ok:true, provider:'spawn', worker:1280228f...`；子会话实际启动并把结果回灌成 `l12`（source=agent，direct）；`counters.dispatched=1`；Worker 列表出现 `1280228f·sub`。冒烟测试 11/11 通过，临时诊断字段已删。

## 仍然缺的（不要误当完整 Super-agent）

| 能力 | 状态 |
|---|---|
| 看真实事件 | ✅ |
| 派真活（手动） | ✅ |
| 自动决策循环 | ❌ `decide`/`spawn` 仍需人点或工具调用，`auto` 默认关 |
| 外部事件源（IM/GitHub/CI） | ❌ 只吃进程内事件 |
| 面板视觉 | ⚠️ 本机无可用浏览器，未验证 |

开 `dispatch.auto=true` 即让它对 queued 事件自主派活，会真实消耗 token，须用户明确同意后再开。

## 接入注意

宿主半边改动需重新 apply 插件（`plugin_manager` 把 `include:superagent-lite` 关一次再开）；客户端 `client.js` 改动需刷新页面。apply 时会带一条与本次无关的警告：`experimental-inspector: globalThis.fetch is an accessor and cannot be observed safely`。
