---
id: "e19c02d2-2a56-4f20-9ce8-b38172ef8952"
title: "dsh-superagent-lite：DSH 插件「事件驱动 super-agent 切片」实现与验证"
description: "本地 DSH 插件 dsh-superagent-lite 的实现结构、关键设计取舍（HTTP 数据通道、单一闭包复用、逻辑 Worker）、安装方式、验证结果与已知边界，供后续扩展 Watchdog / 真实 Worker Session 时参考。"
status: "active"
created_at: "2026-11-02T16:01:54.845Z"
updated_at: "2026-11-02T16:01:54.845Z"
content_hash: "3a85bb4d077128e0fefe7c47651add72026a01a626760114accff24bb0fc3d3e"
source_paths:
  - "plugins/dsh-superagent-lite/index.js"
  - "plugins/dsh-superagent-lite/client.js"
  - "plugins/dsh-superagent-lite/cordis.patch.yml"
  - "plugins/dsh-superagent-lite/test/smoke.mjs"
  - "plugins/dsh-superagent-lite/README.md"
  - ".dsh/profiles/desktop/package.json"
session_ids:
  - "88dc87df-fd96-4051-9362-d5fc472d5dec"
memory_body_ids:
  []
---

# dsh-superagent-lite：事件驱动 super-agent 切片的实现记录

2026-11-02 晚在 `/Users/liuyongze/plugins/dsh-superagent-lite` 从零实现并装进 desktop profile 的本地 DSH 插件。目标不是完整 Super-agent，而是把「两级过滤 + 订阅直通 + Router 批处理 + 会话内面板」做成可运行切片。

## 文件结构与职责

| 文件 | 职责 |
|---|---|
| `index.js` | 宿主侧：两级过滤（程序层 mute/ignore + 语义候选）→ 订阅直通的路由状态机、`superagent` 工具、`/superagent/api/*` HTTP 路由 |
| `client.js` | 浏览器侧：注册 `conversation.view` 槽位的「Super-agent」面板（计数卡、Router 批处理、Thread 订阅、最近 Trigger） |
| `cordis.patch.yml` | 以 bundle patch 插入插件行，所有可调项集中在 `config`（数据源分类、mutePatterns 等） |
| `test/smoke.mjs` | 5 个纯逻辑测试，不依赖 DSH 环境 |
| `locale/en.json`、`locale/zh.json`、`icon.svg` | 面板文案与图标 |
| `README.md` | 对应文章章节、用法、明确未实现项 |

状态持久化在插件自己的运行态目录（示例为 `.dsh/.../bus.json`，含 triggers 与 counters），HTTP 读的是同一份状态。

## 关键设计取舍（重要，后续扩展请沿用）

- **数据通道用插件自注册 HTTP 路由，不用 command / remote RPC**。面板需约 1.2s 轮询；若走 `/superagent status` 命令，每次轮询都会往会话日志写一条 `command/run`，数小时可产生数万条日志。改为宿主注册 `GET /superagent/api/state`（另有 seed 等 POST），页面同源 fetch，零会话日志写入。此套路来自已安装的 save-token 插件（`inject = ["tools","webServer"]` 注册路由）的实际做法，不是猜测。
- **一个操作、两个调用方**：面板按钮与 `superagent` 工具调用同一批闭包函数，避免两套逻辑漂移。
- **Worker 是逻辑实体**：`worker-1` 只是订阅关系中的 id，不创建真实 Session；文章里的真实多会话调度不在本切片内。
- 两级过滤语义：程序层直接挡掉 CI 与静音消息（如「收到」「好的」「晚点聊」「中午吃啥」），通过的进入语义候选队列；Router 以 batch 决定 dispatch，命中已订阅 Thread 的事件直接 `route=direct` 给 worker，不进队列。

## 安装方式

通过 `plugin_manager install_bundle` 安装，profile 内写成软链依赖：
`/Users/liuyongze/.dsh/profiles/desktop/package.json` 第 48 行 `"dsh-superagent-lite": "link:/Users/liuyongze/plugins/dsh-superagent-lite"`。
安装回执为 `application: applied`，无阻塞警告（仅有一条与本插件无关的 experimental-inspector fetch accessor 警告）。

## 验证到哪一层

| 验证项 | 结果 |
|---|---|
| `node --check`（host + client） | 通过 |
| 冒烟测试 | 5/5 通过（导出/注册、程序层 Fast Path、订阅直通、batch 计量、HTTP 端点） |
| 安装 | applied |
| 工具调用 | 实际调用 `status` / `seed` / `decide`，返回结构正常 |
| HTTP | `GET /superagent/api/state` → 200，JSON 字段完整 |
| 槽位注册 | `conversation.view` occupant：`id=superagent-lite, order=6, active=true` |
| 真实视觉/交互 | **未验证**：本机无 playwright，profile 也无 react 可做服务端渲染（`Cannot find module 'react'`），只能证明注册成功且注册期未抛错 |

## 已知边界与后续入口

未实现（README 已列）：真实 Worker Session 调度、异步 ReAct `start()/stop()`、存算分离、Watchdog、真实 IM/GitHub 事件源。后续自然延伸方向：补 Watchdog 为第二模块；用 subagent 把 Worker 接成真实 Session 做跨会话调度；按皮肤调面板视觉。
