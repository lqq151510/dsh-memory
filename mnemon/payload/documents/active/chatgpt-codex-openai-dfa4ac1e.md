---
id: "dfa4ac1e-558b-4124-a946-9b0a2bd15b11"
title: "ChatGPT/Codex 桌面端断流排查：区分本地代理故障与 OpenAI 上游故障"
description: "ChatGPT/Codex「Stream Disconnected」的分层取证顺序（OpenAI 状态页 → bili /__bili/overview alerts → codex 端点流式探针 → 桌面端 7896 连接数），本机 Clash Verge 代理事实（混合端口 7896、TUN/系统代理开启）与常见误判反例。"
status: "active"
created_at: "2026-11-18T12:36:34.189Z"
updated_at: "2026-11-18T12:36:34.189Z"
content_hash: "450491fb58fd07d5e0516d49eb0bde312a846c238ff4d578f6b7326ca1be34a4"
source_paths:
  - ".dsh/skills/dsh-bili-upstream-proxy-fix/SKILL.md"
  - "Library/Application Support/io.github.clash-verge-rev.clash-verge-rev/verge.yaml"
session_ids:
  - "mnemon-run-v1-51d34620-10ee-48c9-ae0b-da6e895cccac"
memory_body_ids:
  []
---

# ChatGPT/Codex 桌面端「Stream Disconnected」断流：本地代理 vs OpenAI 上游故障的分层判定

结论先行：2026-11-18 20:19 那次断流**本地无可修项**。报错时间窗正好落在 OpenAI 官方状态事件内（`Elevated conversation errors for Free and Go users`，状态 `identified`，创建于 2026-11-18T12:15:02Z = 本地 20:15:02，组件 `Conversations` = `degraded_performance`）；同时本机经 bili 转发口对流式端点连打 3 轮全部 HTTP 200 且每轮都有 `event: response.completed`。此时换节点、重装客户端、改 Clash 规则都是把外部故障误判成配置问题。

## 取证顺序（由便宜到贵，先外部后本机）

1. **OpenAI 状态页**：`status.openai.com` 的公开状态 API（summary / incidents 两类 JSON），看 `incidents[].status/impact/created_at/incident_updates` 与 `components[].status`。判定要点是**事件时间窗是否覆盖报错时间**，而不是是否存在历史 incident。实例抓取结果：incidents 25 条，仅 1 条未 resolved，组件 `Conversations`、`Compliance API` 为 `degraded_performance`。
2. **bili 侧健康**：`GET /__bili/overview` 的 `alerts[]` 是唯一可信源（`/__bili/upstreams` 可能返回空数组、`/__bili/circuits` 在 0.1.210 直接报 `no such management endpoint`）；`providers` 里 `chatgpt.com` 与 `api.openai.com` 都指向 `http://127.0.0.1:7896`，说明没有被判直连。横幅判读细则见文档 `c2665f41-c969-4053-bb1a-974d89149a67`。
3. **端点探针（关键）**：`GET https://chatgpt.com/backend-api/codex/responses` 返回 **405 属正常**（该端点只收 POST），不能当故障。真正的判定是带鉴权的 POST 流式请求，看是否出现 `event: response.created` 与 `event: response.completed`。实例：3/3 HTTP 200，ttfb 1.30–3.30s，总耗时 4.18–6.43s，每轮约 56.5KB。
4. **桌面端是否真的走代理**：按 pid 统计 ChatGPT 的已建立 TCP 连接，实例中 58 条指向 `127.0.0.1:7896`。只有这一步能证明"客户端在走代理"，仅看系统代理开关不够。
5. **底层连通性**：`api.openai.com/v1/models` 不带 key 经代理返回 **401 且 1.4s 返回**，说明网络层通、缺的只是鉴权。

## 本机环境事实

- Clash Verge Rev 配置目录：`/Users/liuyongze/Library/Application Support/io.github.clash-verge-rev.clash-verge-rev/`，内核 `verge-mihomo`，生效配置为目录下的 `clash-verge.yaml`（另有 `config.yaml`、`profiles/`）。
- `verge.yaml`：`enable_tun_mode: true`、`enable_system_proxy: true`、`verge_mixed_port: 7896`、`enable_external_controller: false`、`enable_proxy_guard: false`、`enable_builtin_enhanced: true`。外部控制器关闭意味着不能靠 `/configs` 之类的 REST 口改配置，只能改文件后 reload。
- AI 分流：`DOMAIN-SUFFIX,chatgpt.com,🤖 AI 服务`、`DOMAIN-SUFFIX,openai.com,🤖 AI 服务`；`🤖 AI 服务` 是手动 `select` 组，成员含日/美/新/台四组与 `🔰 节点选择`。
- 其它相关规则：`DOMAIN-KEYWORD,google,🔰 节点选择`、`DOMAIN-SUFFIX,gstatic.com,🔰 节点选择`、`MATCH,🐟 漏网之鱼`。
- 订阅更新日志在 `logs/latest.log`，成功标志为 `[Validate] 验证成功` + `[订阅更新] 更新订阅配置成功`；该文件里的订阅 URL 带 token，按敏感信息处理，不要外传或写入文档。

## 反例：容易误判成"线路坏了"的现象

- **首页 200 ≠ 端点可用**：`chatgpt.com` 首页体积 61KB–735KB、总耗时 5.8–15s，多次出现 `curl: (28) Operation timed out after 15003 milliseconds with 327730/536221 bytes received`——大数据量静态资源下载慢，与流式对话链路无关。
- **偶发抖动**：gstatic 首轮 10 次有 3 次失败，复测 10/10 全 204（0.68–3.32s）。单轮失败率不能作为换节点依据。
- **一次 ECONNRESET 不等于配置错**：`[conn#2383] tcp closed reason=error(ECONNRESET) age=7259ms reqs=1` 是同窗口内被上游重置的单条连接，需与状态页时间窗对齐判读。
- **带宽偏低不是断流原因**：首页下载 20–35 KB/s，但流式对话带宽需求低、3 次探针稳定，不能据此改节点。

## 判定口径

- 本地可疑：`alerts[]` 非空且指向 chatgpt/openai provider；探针 0/3 或稳定 ECONNRESET；桌面端没有 7896 连接；订阅更新失败。
- 上游可疑：状态页 incident 覆盖报错时间 + 探针多数 200 + 桌面端连接正常 → **等缓解后重试，不改本地配置**。

## 相关

- Skill：`/Users/liuyongze/.dsh/skills/dsh-bili-upstream-proxy-fix/SKILL.md`（bili 取证与定向代理的通用流程，与上文的 `/__bili/overview` 部分重叠，端点探测与桌面端连接核验是本档补充）。
