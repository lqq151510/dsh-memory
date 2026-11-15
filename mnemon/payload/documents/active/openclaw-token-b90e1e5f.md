---
id: "b90e1e5f-08c3-4d96-9a7f-075e33de2c97"
title: "OpenClaw 提示词瘦身：技能目录才是 token 大头（实测配置与坑）"
description: "降低 OpenClaw 每轮提示词体积的实测结论：技能目录占 17,948 字符、工具 schema 固定 6,236 且不受 tools.profile 影响；skills.limits.maxSkillsInPrompt=0 使 promptTokens 降 32%；agents.defaults.tools 为非法键。含备份、验证与回滚。"
status: "active"
created_at: "2026-10-31T01:20:52.527Z"
updated_at: "2026-10-31T01:20:52.527Z"
content_hash: "dff55021bff49cf055dd31533fcd1f552dc61e814a9d73bf03266edfe198b429"
source_paths:
  - "openclaw.json"
  - "openclaw.json.bak-skills-091848"
  - "openclaw.json.bak-toolsfix-091720"
session_ids:
  - "b2b7acaa-81f0-4411-a0da-8c52c0afe486"
memory_body_ids:
  []
---

# OpenClaw 提示词瘦身：技能目录才是 token 大头

## 背景与目标
用户要求"精简其他工具"，目标是降低 OpenClaw（2026.10.5，本机安装于 `/Users/liuyongze/.openclaw`）每轮注入的提示词体积，尤其是工具相关的开销。

## 测量方法
用每次会话的诊断数据对比（`promptTokens`、`systemPrompt.chars`、`skills.promptChars`、`tools.schemaChars`、`tools.entries`），并结合网关日志中的 `tool-search: cataloged N tools behind compact prompt surface`。

## 关键结论
- **技能目录是主要开销，不是工具 schema**：改动前每轮注入 131 条技能，占 17,948 字符；系统提示总计 52,602 字符。
- **工具面已很小且与 `tools.profile` 无关**：实际只有 11 个工具（`ls`/`read`/`edit`/`write`/`apply_patch`/`exec`/`process`/`sessions_yield`/`tool_search`/`tool_describe`/`tool_call`），schema 共 6,236 字符；把 `tools.profile` 从 `minimal` 切到 `full` 后 schemaChars 仍是 6,236，说明在此版本下改 profile 收益为零。
- **`agents.defaults.tools` 不是合法键**：配置校验报 `Invalid config ... agents.defaults: Unrecognized key: "tools"`，会把整个配置判为非法。tools 策略需放在 agents 条目/其它允许的作用域（如 `agents.entries.growth.tools` 是合法的）。

## 实际改动（`openclaw.json`）
1. 删除非法的 `agents.defaults.tools`；备份 `openclaw.json.bak-toolsfix-091720`。
2. 关闭模型侧技能清单注入：
   ```json
   "skills": { "limits": { "maxSkillsInPrompt": 0, "maxSkillsPromptChars": 0 } }
   ```
   备份 `openclaw.json.bak-skills-091848`。

## 验证结果
- 系统提示 52,602 → **33,973** 字符；技能条目 131 → **0**（`skills.promptChars` = 0）；工具仍为 11 个 / 6,236 字符。
- 每轮 `promptTokens` 18,791 → **12,746**（约省 32%）。
- 网关健康 `{"ok":true,"status":"live"}`，配置热加载生效（日志 `config hot reload applied (tools)`）。
- 四个自动化作业保持原状：`heartbeat-growth`（agent=growth）、`checkin-qq`（tools 仅 `message`、thinking off）、`growth-review`（agent=growth）、`Memory Dreaming Promotion`。

## 代价与回滚
代价：代理不再看到技能清单，需要按名调用技能；普通聊天无影响。回滚只需删除 `skills.limits`（或恢复上述备份文件）。

## 相关可调项（schema 中已确认存在）
`skills.allowBundled`（仅影响 bundled 技能）、`skills.load.extraDirs/watch`、`skills.install.preferBrew/nodeManager`、`skills.limits.maxSkillsLoadedPerSource`、`skills.workshop.autonomous.mode`（`off`/`auto`）。
