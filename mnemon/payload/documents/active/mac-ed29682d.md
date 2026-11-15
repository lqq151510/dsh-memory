---
id: "ed29682d-08e9-426c-8aff-5a7ee76ade2d"
title: "开心问候守卫闸门：按发货实时钟点硬判静默窗口（含 Mac 睡眠迟到问题）"
description: "checkin-roll 挂 command 守卫而非 announce 作业的原因与设计：cron.skipMissedJobs 保留 at 作业补跑 + Mac DarkWake 迟到会撞进双倍计费窗口，故由 tools/checkin-fire.py 按发货实时钟点判定（静默窗口、迟到>60 分钟、早于 2 分钟均不发），放行才现挂 announce 并写入真实钟点；规则唯一处 tools/kaixin_time.py，test_guard.py 16 场景 + 1440 次抽样验证。"
status: "active"
created_at: "2026-10-31T12:27:36.956Z"
updated_at: "2026-10-31T12:27:36.956Z"
content_hash: "01bf446b3d1442aa26e21e801e61d903bdde5183a55be656be2bbf3449afc201"
source_paths:
  - "workspace/tools/checkin-fire.py"
  - "workspace/tools/kaixin_time.py"
  - "workspace/tools/checkin-roll.py"
  - "workspace/tools/test_guard.py"
  - "workspace/AGENTS.md"
  - "workspace/USER.md"
  - "workspace/MEMORY.md"
session_ids:
  - "008fa278-0430-4140-9b28-adbf89445dd1"
memory_body_ids:
  []
---

# 开心问候的守卫闸门：按发货实时钟点硬判静默窗口（2026-10-31 落地）

关联文档：`414a45c1-0caa-4ef0-b0eb-09ef98dc319b`（主动问候静默失败的旧根因：tools.allow 漏 message）。

## 问题

问候时刻是 checkin-roll 每天 00:05 摇号后挂**一次性 at 作业**。原设计靠「摇号窗口」在结构上避开 DeepSeek 双倍计费窗口（工作日 09:00-12:00、14:00-18:00 北京时间）。但两个事实推翻了这条保证：

- OpenClaw 2026.10.5 的 `cron.skipMissedJobs` 文档明确写「one-shot (at) jobs retain their normal catch-up behavior」——改配置救不了迟到补跑。
- `pmset -g log` 显示这台 Mac 整夜 DarkWake/Sleep 反复。实测有 07:42 的槽在 09:30 才补跑（真机数据里还出现 08:20 槽 09:10 才醒）。迟到会把问候直接推进静默窗口。

## 方案：守卫作业 + 双层判定

每个摇号时刻挂的不是 announce 作业，而是一条 **command 守卫作业**（payload 跑 `tools/checkin-fire.py`，cwd 是 workspace）。守卫**按发货那一刻的实时钟点**硬判，只有放行才现场用 CLI 挂一次性 announce 作业（`--announce --channel qqbot --to c2c:3D71F6B54EF7769731893C84CFFA8486 --best-effort-deliver --delete-after-run --tools "" --thinking off`），并把**真实钟点**写进提示词（否则模型会按预定钟点写「早上好」，而实际是晚上）。

判定规则（唯一实现处 `tools/kaixin_time.py`，禁止在其他脚本里复制规则）：

- 发货时刻落在工作日 09:00-12:00 或 14:00-18:00 → 整条不发。
- 比预定时刻**迟超过 60 分钟** → 不发。
- 比预定时刻**早 2 分钟以上** → 不发。
- 静默窗口外、迟到 ≤60 分钟 → 放行。

取舍：宁可今天少一条，也不在双倍计费窗口发。

## 文件与验证

- `tools/kaixin_time.py`（92 行）：静默窗口、问候窗口、法定节假日（`holidays-2026.txt`）规则的唯一定义处。
- `tools/checkin-fire.py`（140 行）：守卫。
- `tools/checkin-roll.py`：从「直接挂 announce」改成「挂 command 守卫」；补跑路径新增 `pick_catch_up`，在允许窗口内随机取点，不再用「现在+2 分钟」（可能撞进静默窗口）。补跑兜底：当天时刻全过就再补一条（commit 62cf878）。
- `tools/test_guard.py`（81 行，新增）：16 个判定场景（工作日到点/迟到/静默外、节假日、周日晚早跑、build_spawn 互斥开关）+ `pick_catch_up` 抽样 1440 次全部避开静默窗口，FAILS=0。另有 `tools/test_roll_live.py` 真机端到端（2 分钟后挂一条 announce 看能否发出）。
- 文档同步：AGENTS.md / USER.md / MEMORY.md 里「摇号窗口结构保证不会落进去」的旧口径全部改成守卫口径，并写明原因（Mac 睡眠让 at 作业迟到）。
- 提交：`8a09c84` 问候改守卫闸门（9 files, +405/-88）。

## 坑

- 改脚本前必须先 read 文件，否则写操作报 `file has not been read`。
- `cron_run_receipts` 表没有 `name` 列（只有 receipt_id / store_key / job_id / status / started_at_ms / delivery_attempt_state 等），按 name 查会 `no such column: name`；确认投递要看 `status` + `delivery_attempt_state`。

## 结论

静默窗口这层防护不再依赖摇号窗口的几何性质，也不依赖 cron 的准时性；是否发消息由发货瞬间的真实钟点决定。
