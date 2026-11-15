---
id: "414a45c1-0caa-4ef0-b0eb-09ef98dc319b"
title: "OpenClaw 主动问候静默失败根因：tools.allow 白名单漏 message"
description: "12:30 定时问候静默失败的排查结论：并发会话写入 tools.allow 15 项白名单却漏掉 message 工具，导致 tool_call 报 Unknown tool id。含回退与验证、故障指纹，以及 message 工具需 action:\"send\" 的参数要点。"
status: "active"
created_at: "2026-10-31T10:26:45.361Z"
updated_at: "2026-10-31T10:26:45.361Z"
content_hash: "2f8efefc2f09b324def26a87e519f5e014d492476c8c8c2813085bd738d97ed5"
source_paths:
  - "openclaw.json"
  - "openclaw.json.bak-prompt-20261031014812"
session_ids:
  - "1c1d04d5-4706-4df7-9e48-8ea2657523bc"
memory_body_ids:
  []
---

# OpenClaw 主动问候静默失败根因：tools.allow 白名单漏 message

## 现象
2026-10-31 12:30 定时问候（checkin-qq）静默失败。会话记录显示：`tool_call {"id":"message",...}` 返回 `Unknown tool id: message`，`tool_search` 对 message / qqbot 发送查询也返回空；而 QQ 通道本身全程 `running/connected`。

## 根因（有证据）
不是提示词瘦身优化导致的，而是另一个并发会话在 09:48 又写了一次 `openclaw.json`，把 message 工具挡掉：
- 备份 `openclaw.json.bak-prompt-20261031014812` 里出现 `tools.allow` 15 项白名单（read/write/exec/…），**偏偏漏了 message**。
- 同一份配置把 `skills.limits` 从 40/18000 压到 6/900。
- 完整旁白记录在另一 DSH 会话目录（0b6050f1… / 5b46c301…），证实是它 09:48 做「再压一档 prompt 成本」时写入，并写了 bak-prompt 备份。

## 回退（已执行并验证）
| 项 | 改坏后 | 回退后 |
|---|---|---|
| `tools.allow` | 15 项白名单，漏 message | 删除该键，恢复默认全量工具目录 → message 回归 |
| `skills.limits` | 6 / 900 | 40 / 18000（turn-1 值） |

回退后与 turn-1 结束态逐字段比对零差异（`diff keys: NONE`）。手动跑一次问候任务实测：`18:01:56 status=ok delivered=True`，正文照发到 QQ。

## 故障指纹（下次先查这里）
再出现「突然不主动发消息」，先看 `openclaw.json` 是否又被加了 `tools.allow` 却没带 `message` —— 这是本次故障的指纹。

## message 工具参数要点
`openclaw:core:message` 需要 `action` 属性（如 `action: "send"`），缺省会报：
`Invalid arguments for tool "openclaw:core:message": action: must have required property 'action'`。
调用形态：`{"action":"send","channel":"qqbot","target":"<c2c id>","message":"..."}`。

## 保留未动
- `heartbeat.every="0"`（心跳关闭）保留 —— 与主动联系无关，只是每 30 分钟烧 token 换 NO_REPLY。
- 三个定时任务保留：checkin-qq（08:30/12:30/21:30）、growth-review（23:30）、梦境（03:00）。

## 相关文档
- 提示词瘦身与工具配置键实测结论见 b90e1e5f-08c3-4d96-9a7f-075e33de2c97（含 `tools.profile` 不影响固定 schema、`agents.defaults.tools` 为非法键等）。
