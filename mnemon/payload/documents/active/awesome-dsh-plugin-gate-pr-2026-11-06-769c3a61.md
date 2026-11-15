---
id: "769c3a61-4d2e-4ac9-bd39-d46c28c186be"
title: "awesome-dsh-plugin gate 触发机制修正：空闲 PR 不会自动转绿（2026-11-06）"
description: "修正《awesome-dsh-plugin 收录提交流程与门禁规则》中「年龄闸约 2 小时自愈」的说法：gate 由 workflow_run 触发、只评估刚跑完检查的批次，空闲 PR 需推空提交刷新；含 PR #11963 的实证与刷新步骤。"
status: "active"
created_at: "2026-11-06T10:40:01.371Z"
updated_at: "2026-11-06T10:40:01.371Z"
content_hash: "9da366e312ee206afaf569386cb8ffdd4d7dd31dcfe0fb5c42bf30f3a5962a79"
source_paths:
  []
session_ids:
  - "2d86d995-daa9-4018-9a57-b171e45f92bc"
memory_body_ids:
  []
---

# awesome-dsh-plugin Submission gate 触发机制修正（2026-11-06）

承接《awesome-dsh-plugin 收录提交流程与门禁规则（dsh-computer-use → PR #11963）》（id `32c13a81-47d3-48ca-bc7a-1427243cff68`）与《dsh-computer-use 游戏级响应优化：低延迟输入 daemon 与 computer_sequence（2026-11-05）》（id `b9bbd9c4-2b5f-4136-be3f-e8e68dbcacce`）。本文只修正其中「仓库年龄闸会自己重跑、约 2 小时后转绿」这一条。

## 触发机制（实测修正）

- `Submission gate` 由 `workflow_run` 事件触发：只评估**当批刚跑完检查**的 PR，不是按 PR 定时轮询。
- 所以一条长时间没有新 check-run 的 PR，gate 结论会一直停在旧记录。PR #11963 的 gate 始终停在 `2026-11-05T08:47:49Z` 的 failure，输出仍是「repository is 0.9 days old (needs 1)」，即使仓库年龄早已满 1 天。
- 同期同仓库其它 PR 的 gate 在 2026-11-06 10:21/10:24/10:27 连续 success，证明 gate 服务本身正常，只是不回头重评空闲 PR。
- 结论：**年龄达标不等于自动转绿**；官方那句「this check re-runs by itself, about 2h」只在后续有其它活动把它重新纳入评估时成立。
- 排查时容易被 PR 上 `check` 全绿 + `mergeable: MERGEABLE` 误导，必须单独拉 `Submission gate` 的 check-run 时间戳，看它是不是旧的。

## 刷新办法

1. 在 PR 分支推一个空提交 / chore 提交（本次 fork `lqq151510/awesome-dsh-plugin` 分支 `add/lqq151510-dsh-computer-use`，commit `2d517a4`）。
2. 该推送触发 `PR check`，其完成后再触发 `Submission gate` 重跑并重新计算仓库年龄。
3. `PR check` 单次约 15 分钟（上次 15m42s），期间状态为 pending；用 `gh pr checks 11963 --watch` 或 GitHub API 轮询。
4. 空提交不改业务代码，维护者看到的是条目 yml 更新加一个 chore 提交，不构成额外审阅负担。

## 已核实的旁证

- `check`（文件位置、yml 格式、awesome-lint、站点构建）对 #11963 已长期 SUCCESS，PR 状态 OPEN、MERGEABLE、无 `held: author` 标签、无维护者评论。
- 仓库 `https://github.com/lqq151510/dsh-computer-use` 创建于 2026-11-04 前后，到 2026-11-06 时年龄已满 1 天，年龄本身不再是阻塞项。
- gate 失败文案里的「No need to resubmit, push, or close and reopen」是针对年龄不足场景的，不适用于「检查记录过期」——这两种失败长得一样，处理方式相反。
