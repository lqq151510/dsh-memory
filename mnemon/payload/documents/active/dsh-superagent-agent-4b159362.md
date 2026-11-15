---
id: "4b159362-5618-4c0a-a521-e157ffab0689"
title: "dsh-superagent 本地试用环境搭建与 agent 小号必要性依据"
description: "dsh-superagent 在 macOS 本地跑通的环境修补记录：tmux/dsh 工具链修复、独立 profile 托管启动、GitHub Source 未配置的中间态，以及为什么必须用独立 agent 小号（含源码判定依据）。"
status: "active"
created_at: "2026-11-01T06:05:59.627Z"
updated_at: "2026-11-01T06:05:59.627Z"
content_hash: "2363b05a08b14e642481b91cbb48493341d35135e231f9c11a100ae8dcd63c6e"
source_paths:
  - "opensource/dsh-superagent-prototype"
  - "anaconda3/envs/tmux/bin/tmux"
  - "opensource/dsh-superagent-prototype/.artifacts"
session_ids:
  - "0959a715-67e7-402c-8142-4c2604faa753"
memory_body_ids:
  []
---

# dsh-superagent 本地试用环境：搭建过程、环境修补与“小号”必要性依据

记录 2026-11-01 一次本地试用 dsh-superagent 的落地过程：环境已就绪到“托管启动成功、等待 GitHub 源配置”，并留下后续待办的明确入口。适用于同一台 macOS（arm64，用户 liuyongze）上继续调试 dsh-superagent，不适用于其他机器。

## 目标与当前状态

目标：在独立 profile 中跑起 dsh-superagent，用沙盒 GitHub 仓库验证“评论触发工单 → Executor 在 worktree 中改代码 → 开 PR”的闭环。

已就绪：

| 步骤 | 状态 |
| --- | --- |
| 仓库克隆到 `~/opensource/dsh-superagent-prototype` | 完成 |
| `pnpm install && pnpm build` | 完成（期间修过 rolldown 原生绑定未下载的问题） |
| 沙盒仓库 `lqq151510/superagent-sandbox`（私有） | 已创建 |
| 独立 profile `~/.dsh/profiles/superagent`（端口 3082，stateDir 已配，bundle 以 link 方式装入） | 实测可启动，管理 API 正常 |
| policy 允许名单、executor 的 worktree prepare 命令 | 已保存 |
| tmux | 已装（见下方 Homebrew 卡死绕行） |
| `~/.local/bin/gh-agent` 包装器 | 已写好，未登录 |
| 全局 `@deepseek-ai/dsh@0.2.1-rc.3` | 安装完成（修复了坏软链） |

卡住的唯一一环：GitHub Source 的 repository/cli 尚未填写，`scripts/dsh-superagent status` 退出码为 3，setup 停在 `repositoryUnset`（“The GitHub Source names no repository.”）。这是预期中间态，不是故障。

待用户完成：注册/提供 agent 小号账号名，并执行 `~/.local/bin/gh-agent auth login`（凭据隔离在 `~/.config/gh-agent`，不影响主号 gh 登录）；随后填入 `source:github` 的 repository 与 cli 路径，把该小号邀请进 `superagent-sandbox`（写权限），再复验 status 退出码为 0。

## 为什么必须用“agent 小号”而不是主号（源码依据）

主号 `lqq151510` 若同时充当 agent 账号，用户自己 @ 它发出的请求会被判成非人类消息，工单永远建不起来：

- `src/sources/github-trigger.ts:62` 的 `author()`：`human: isHumanLogin(user) && !sameName(user.login, agent)` —— 评论作者与 agent 账号同名时该消息被标记为 `human: false`。
- `src/core/assignments.ts:97,106`：准入只接受 human 的显式请求。

因此 agent 身份必须是另一个独立 GitHub 账号（建议命名形如 `<主号>-agent`），通过 `gh-agent` 独立登录。这是本次排查的关键结论，改动任何账号配置前先回到这两处源码确认。

## 环境中踩到的三个坑与修法

1. **Homebrew 装 tmux 卡死**：`HOMEBREW_API_DOMAIN` 被指向阿里云镜像后长时间停在 `==> Downloading Homebrew API data`（本地 `~/Library/Caches/Homebrew/api/internal/packages.arm64_golden_gate.jws.json.new` 持续增长但永不完成）。
   绕行：改用 conda-forge 安装 `tmux 3.8` 到 `~/anaconda3/envs/tmux/bin/tmux`，再软链到 `~/.local/bin/tmux`；`tmux -V` 返回 `tmux 3.8`。
2. **`~/.local/bin/dsh` 是死链接**：原指向已卸载的 Open Design 工具链。已备份为 `~/.local/bin/dsh.broken-opendesign`，改用全局安装的 `@deepseek-ai/dsh@0.2.1-rc.3`（fnm node `v24.22.0`，真实路径 `/Users/liuyongze/.local/share/fnm/node-versions/v24.22.0/installation/lib/node_modules/@deepseek-ai/dsh/lib/bin.js`），`dsh --version` 正常。
   注意：全局安装提示 `npm warn install-scripts`，若需要原生脚本再补 `--allow-scripts=@deepseek-ai/dsh-subprocess-local,koffi,node-pty,@google/genai,protobufjs`。
3. **托管启动与状态判读**：`scripts/dsh-superagent start` 用 tmux 会话 `dsh-superagent` 托管 `dsh --profile superagent --no-open`；启动日志在 `~/opensource/dsh-superagent-prototype/.artifacts/dsh-superagent-superagent-<时间戳>-<pid>.log`，面板 URL（含 token）打印在该日志中，本机地址为 `http://127.0.0.1:3082`。status 退出码语义：0 表示全部 setup 通过，3 表示仍有 setup 阻塞项。

## 已落盘的关键配置（未生效前不要当结论）

- `executor.directory` = `/Users/liuyongze/.dsh/state/superagent/work`。
- prepare 命令：`git -C /Users/liuyongze/src/superagent-sandbox fetch --no-tags https://github.com/...` 后建 `agent/<ticket>` worktree。
- `policy.sources.github.allowed` = `[{'id': 'lqq151510'}]`（准入允许名单，admission 为空）。

排障顺序建议：先看 `scripts/dsh-superagent status` 的 setup 项与退出码，再看后台服务日志，最后才去核对 profile 配置与 policy 文件；面板 401 属于正常鉴权，需带日志中的 token 访问。

## 可复用的方法性结论

- 该项目的状态检查用“退出码 + setup 阻塞项”表达 readiness，而不是单看进程存活：进程在跑但 setup 未完成时 status 仍非 0，验证时两者要分别报告。
- 账号类身份问题优先从源码定位判定条件，再决定注册新账号，避免配置反复试错。
- 国内镜像环境（HOMEBREW_API_DOMAIN / UV_INDEX_URL / PIP_INDEX_URL）常导致安装卡住而非报错，遇到长时间无输出的安装应换通道（conda-forge、官方源）而不是反复重试。
