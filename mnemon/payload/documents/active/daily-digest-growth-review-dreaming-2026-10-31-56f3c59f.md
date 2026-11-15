---
id: "56f3c59f-9f9f-4562-97e7-d1513ed9dc5f"
title: "夜间维护流水线：daily-digest / growth-review / dreaming（2026-10-31 时间前移）"
description: "OpenClaw 每晚记忆闭环的当前排程（22:20 摘录 / 22:30 复盘 / 07:00 dreaming）、前移原因（娘子 Mac 凌晨睡眠）、三作业 id 与提示词要点、用户作业与插件托管作业两种改法、核查与回滚路径及 sqlite/写竞态坑。"
status: "active"
created_at: "2026-10-31T13:50:40.743Z"
updated_at: "2026-10-31T13:50:40.743Z"
content_hash: "6c6ea801180b88966ef560fdf6bfbce69135c549566bfb1efea96a1c1dea3d25"
source_paths:
  - "workspace/AGENTS.md"
  - "workspace/OPS.md"
  - "workspace/tools/daily-digest.py"
  - "workspace/DREAMS.md"
  - "openclaw.json"
session_ids:
  - "3461c0dd-6fba-4cf9-885f-a4af402a802e"
memory_body_ids:
  []
---

# 夜间维护流水线：daily-digest → growth-review → dreaming（2026-10-31 时间前移）

本机 OpenClaw（`/Users/liuyongze/.openclaw`，2026.10.5）每晚一条记忆闭环，2026-10-31 按娘子要求整体前移。

## 当前时间表（2026-10-31 起）

| 阶段 | 时间 | 作业 | 产物 |
|---|---|---|---|
| daily-digest | 22:20（原 23:20） | `607e2c96-f9d9-4fd8-bd3c-e592dd7c7bb1` | `workspace/memory/YYYY-MM-DD.md` |
| growth-review | 22:30（原 23:30） | `79eec495-e827-4a23-b29e-8e1d79584207` | `GROWTH.md` / `USER.md` / `MEMORY.md` |
| dreaming | **07:00**（原 03:00，同日 21:49 再改） | `67c8f7cb-15ec-4910-bfdb-cf7d9eb0a356` | `MEMORY.md` 提升 + `DREAMS.md` |

## 为什么这么排（附原话与机制）

- 娘子原话：「23点的整理提前到22点30」→ 摘录与复盘一起前移到 22:20 / 22:30，保证**先摘录再复盘**（复盘要读当天摘录）。
- 娘子原话：「3点我电脑在休息」「怎么也得7点开始吧」→ dreaming 从 03:00 挪到 07:00。**她的 Mac 凌晨在睡眠**，原先 3 点档等于空跑；03:00 是插件默认值 `DEFAULT_MEMORY_DREAMING_FREQUENCY = "0 3 * * *"`。
- 复盘**不能挂回 main 会话**：会撞正在跑的一轮，历史上 `cron: job execution timed out`（durationMs 849,000）。所以固定 `isolated` + `delivery.mode: none` + `lightContext` + `timeoutSeconds: 1200`。
- 副作用取舍：dreaming 07:00 会改 MEMORY.md，早上第一句（07–09 问候窗）会少一点前缀缓存命中；它本身不发消息，不会重复吵人。

## 各作业要点

1. **daily-digest（command payload）**：`/usr/bin/python3 /Users/liuyongze/.openclaw/workspace/tools/daily-digest.py`，isolated + `--no-deliver`。只读 `~/.openclaw/agents/{growth,main}/agent/openclaw-agent.sqlite` 的 `transcript_events`，只收 session_key `^agent:(growth|main):main$` 的直聊；跳过「现在北京时间」定时提示、「[OpenClaw heartbeat poll]」、复盘提示词与 `NO_REPLY`。输出「娘子说的 / 我回的」两段，每条 `HH:MM` + 320 字截断（`MAX_CHARS = 320`），覆盖写可重跑；当天无对话退昨天，两天都无则只打印 `DIGEST none` 不写文件。**摘录是原始流水，复盘时提炼，不要改这个文件。**
2. **growth-review（agentTurn）**：读 MEMORY/USER/GROUTH + 当天摘录；GROWTH 末尾追加 1–2 条 `- YYYY-MM-DD：经历 / 学到什么`，不编不重复；USER.md 只收「明确说过 / 反复出现 / 影响相处」的**稳定**偏好，祈使句 40 字内、带 `<!-- observed: YYYY-MM-DD | status: active -->`、查重 + 旧条标 superseded + 每天最多 2 条 + 全文 ≤3500 字符；**禁止改 SOUL / AGENTS / 摘录文件**，提示词第一句写死「后台维护任务，绝不给任何人发消息」。
3. **dreaming（memory-core 插件托管）**：`declarationKey: memory-core:memory-dreaming-promotion`，描述 `[managed-by=memory-core.short-term-promotion]`，把加权短期回忆提升进 MEMORY.md（limit=10, minScore=0.750, minRecallCount=3, minUniqueQueries=3, recencyHalfLifeDays=14, maxAgeDays=30），并写 DREAMS.md（纯文本，不进日常提示词）。

## 改排程的两条路径

- **用户作业**（daily-digest / growth-review）：`openclaw automations get|update|runs <id>`，`list --all` 才含已禁用项。
- **插件托管作业**（dreaming）：`openclaw automations` 改不动 system-owned 作业，只能改配置 `plugins.entries.memory-core.config.dreaming.frequency = "0 7 * * *"`（保留 `enabled: true`）。流程仍按技能「OpenClaw 配置开关与瘦身」：python3 改 JSON（`ensure_ascii=False, indent=2`）→ 备份 → lint（任一 openclaw 命令不得出现 `Unrecognized key`）→ 日志确认 `config hot reload applied (plugins.entries.memory-core.config.dreaming.frequency)` → `curl -s -m 5 http://127.0.0.1:18789/health` 返回 `{"ok":true,"status":"live"}` → `automations list` 里 cron 变 `0 7 * * *`。
  本次备份：`/Users/liuyongze/.openclaw/openclaw.json.bak-dreaming-7am-20261031214917`（回退：覆盖回 `openclaw.json` 后重跑 lint + health）。
- 改动后同步文档三处：`workspace/AGENTS.md`（第 11、15、45 行附近的路径说明与维护窗）、`workspace/OPS.md`（第 13、17、60 行的作业条目与纪律）、技能库；提交记录：`75be821` 夜间整理提前到 22:20/22:30、`d8ab61c` dreaming 改到早上 7 点。

## Pitfalls

- **改 `workspace/AGENTS.md` 会撞写竞态**：夜间作业正在写 GROWTH/USER/MEMORY，编辑报 `cannot edit ".../workspace/AGENTS.md": file changed since it was read — re-read the file, then retry`；先重读再改，别盲重试。
- **夜间维护窗随 dreaming 走**：五件套（AGENTS/SOUL/IDENTITY/USER/MEMORY）只在「22:20 摘录之后到 dreaming 之后」批量改，白天要改先记 `memory/YYYY-MM-DD.md` 或 OPS.md。
- **sqlite 表结构坑**：`cron_jobs` 只有 `job_json`（无 `cron` 列）；`cron_run_receipts` 的列是 `receipt_id, store_key, job_id, config_revision, agent_id, request_run_id, delivery_attempt_state, status, owner_pid, owner_start_time, started_at_ms, finished_at_ms, error_text`——**没有 `job_name` / `duration_ms`**，查名字要 join `cron_jobs` 或解析 `job_json`。
- **命令里带 `<!-- … -->` 注释会卡 300 秒**（见技能）：写 `observed` 注释要落文件，别塞单行 `python3 -c`。

## 关联修复（同日）

问候守卫闸门解析带秒的时间戳报错，导致 21:24 与 20:55 两条问候漏发；已修（提交 `0aeabc8`，`workspace/tools/test_guard.py` 场景 16 → 18），21:28 手动补发。守卫闸门本身的规则见文档 `ed29682d-08e9-426c-8aff-5a7ee76ade2d`；配置瘦身与开关表见 `b90e1e5f-08c3-4d96-9a7f-075e33de2c97`。
