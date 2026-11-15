---
id: "32c13a81-47d3-48ca-bc7a-1427243cff68"
title: "awesome-dsh-plugin 收录提交流程与门禁规则（dsh-computer-use → PR #11963）"
description: "把一个本地 DSH 插件提交进 awesome-dsh-plugin 的完整流程、条目格式、CI 检查项与「仓库年龄 1 天」门禁的自愈行为，含本次 dsh-computer-use 的实例证据。"
status: "active"
created_at: "2026-11-04T10:30:36.292Z"
updated_at: "2026-11-04T10:30:36.292Z"
content_hash: "c3b4bdc376aad1c750a6d339224c71b5bef8b0e012fb08e88aa4f0eee29393d2"
source_paths:
  - "plugins/dsh-computer-use/README.md"
  - "plugins/dsh-computer-use/package.json"
session_ids:
  - "1b83a669-0b1f-4439-82b2-e250a00c75a6"
memory_body_ids:
  []
---

# 把 DSH 插件提交进 awesome-dsh-plugin（2026-11-04 实例）

实例：本地插件 `plugins/dsh-computer-use` 提交为 PR #11963（https://github.com/awesome-dsh-plugin/awesome-dsh-plugin/pull/11963），仓库 https://github.com/lqq151510/dsh-computer-use。插件本身的实现与验证见文档 `d723d537-13d1-494b-a535-4c42bf64da7a`，本文只记「收录提交」这条链路。

## 提交前硬性条件（从 contributing.md 与 CI 脚本核实）

- 仓库必须**公开**，且 `package.json` 任意位置声明 `dsh.bundle`（monorepo 可在 `packages/`、`plugins/` 等子目录）；只声明 `dsh.client` 不算可安装。
- 仓库加 `dsh-plugin` topic（写 topic 的 API 偶发 502，失败可重试或用 `gh` 再打一次）。
- 市场卡片图标要**GitHub 托管的位图**：`icon.svg` 会被拒，需光栅化成 `assets/icon-180.png` 之类并写进 `package.json` 的 `files`。
- README 建议补一段英文说明，维护者审仓库时不必依赖中文内容。
- 自己新建的仓库默认分支可能为空，`git push -u origin main` 后再动 PR。

## 条目格式与落点

- 条目文件放 `data/plugins/`、后缀 `.yml`（CI 有专门检查：entry 必须 `.yml`、必须在 `data/plugins/`、README 行由生成器产出，手改 README 会被「READMEs match data/plugins」检查挡下）。
- 实际字段：`url`、`name`（`owner/repo`）、`category`、`description.en` / `description.zh`、`icon`（raw.githubusercontent 位图地址）。分类选 `tools` 时描述里只写能对着代码核实的能力，不写「更快更强」这类无法验证的话。
- 流程：fork → 新建分支 `add/<owner>-<repo>` → 只新增自己那一个条目文件（不动他人条目）→ 推分支 → 开 PR。

## CI 两个阶段与验收含义

- **PR check**（fork 上下文、无 token）：manifest / awesome-lint / README 重生成 / 站点构建 / 各类回归测试。本次 15m42s 全绿；轮询 `gh pr checks` 或 run 的 steps 直到 `check pass`。
- **Submission gate**（`workflow_run` 在 base 仓库跑，带 token）：查 `dsh.bundle`、仓库年龄、提交数。`check-submission.mjs` 里 `MIN_AGE_DAYS = 1`，本次报 "repository is 0.0 days old (needs 1)"，属于**会自愈**的红灯——gate 会按计划自动重跑，不需要重提交、推空提交或关 PR 重开（约 24h 内清除）。
- 因此「PR check 绿 + gate 仅年龄红」= 提交本身合格，等时间即可；不要为年龄去伪造提交历史或改仓库创建时间。

## 踩坑记录

- `gh repo create` 返回 422 "name already exists"：同名仓库可能已建（含空仓库），先 `gh repo view` 确认再决定是否复用。
- GitHub GraphQL / forks / topics 接口本次多次 502；REST 查询 `mergeable`、checks、runs 更稳，失败就换接口而不是反复重试同一端点。
- 分支 push 成功后 GitHub 会直接给出开 PR 的 URL，可直接用它开 PR。

## 后续可选动作

PR 仍 OPEN，等 gate 转绿后进维护者队列；需要的话可再补 `screenshots.json` 或发布 npm 包（`private: true` 需先改）。
