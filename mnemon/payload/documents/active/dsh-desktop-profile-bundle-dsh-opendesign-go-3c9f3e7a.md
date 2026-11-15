---
id: "3c9f3e7a-dee8-4138-988c-f4c1406249c1"
title: "从 DSH desktop profile 彻底卸载 bundle 插件（dsh-opendesign-go 实录）"
description: "dsh-opendesign-go 左上角 amr-link 横幅来源，及五层卸载流程：cordis.patch.yml 条目、package.json 依赖、pnpm-lock 三处、.credentials.yaml refs、node_modules 与运行态目录；含备份/幂等脚本路径、验证证据、重启生效与运行时回写风险。"
status: "active"
created_at: "2026-11-02T15:55:14.252Z"
updated_at: "2026-11-02T15:55:14.252Z"
content_hash: "f5774bcfab8e94ba71ecb41411937115ca6fec4b34aff59ef61a2ceb11435762"
source_paths:
  - ".dsh/profiles/desktop/cordis.patch.yml"
  - ".dsh/profiles/desktop/package.json"
  - ".dsh/profiles/desktop/pnpm-lock.yaml"
  - ".dsh/.credentials.yaml"
  - ".dsh/backups/dsh-opendesign-go-20261102-234816/RESTORE.md"
  - ".dsh/backups/dsh-opendesign-go-20261102-234816/remove-opendesign-go.py"
session_ids:
  - "36ee31e1-2dcb-4945-a873-c22874eaeafd"
memory_body_ids:
  []
---

# 从 DSH desktop profile 彻底卸载一个 bundle 插件（dsh-opendesign-go 实操记录）

## 1. 触发与结论

- 触发：左上角出现 `amr-link.open-design.ai` 的异常横幅，用户问「左上角怎么回事」。实际来源是 **dsh-opendesign-go 插件自带的健康探测**，不是当前会话链路问题——本次会话走 `api.deepseek.com`。
- 探测记录（`~/.dsh/bili-duty/duty.log` 同类 health 记录）形态：`{"kind":"upstream-reset","host":"amr-link.open-design.ai","hint":"upstream/network reset before the response — check upstream and local network; a bounded transparent replay may be attempted (BILI_REPLAY_RETRY_MAX)","count":3}`。横幅在探测恢复正常后自行消失，与对话功能无关。
- 用户随后决定不再保留该插件（"彻底去掉 dsh-opendesign-go"），执行了**五层清理 + 备份 + 幂等脚本 + 重启**。结论：配置与文件层已清干净，**必须重启应用**才生效。

## 2. 配置分层前提（关键）

`~/.dsh/profiles/desktop/cordis.yml` 是空入口列表 `[]`，插件树由 `package.json` 的 `dsh.profile.bundles` + `cordis.patch.yml` 组合而成（文件头注释：`Edit cordis.patch.yml, not this file.`）。所以插件条目的增删**只能改 `cordis.patch.yml`**，改 `cordis.yml` 无意义。

插件包自身的 `package.json` 声明决定它如何注入宿主：

```json
"dsh": {
  "bundle": { "patch": "./cordis.patch.yml" },
  "client": { "inject": ["@deepseek-ai/dsh-client-ui-settings", "...", "@deepseek-ai/dsh-api-settings-controller"], "platform": "web" }
}
```

即条目来自 bundle patch，profile 只需保留一条 `- id: opendesign-go / name: dsh-opendesign-go / config: {...}`；卸载要同时去掉**依赖、patch 条目、lock 记录、凭据引用、磁盘目录**，否则 loader 或 settings 仍会引用到它。

## 3. 卸载清单（逐文件）

| 位置 | 处理 |
| --- | --- |
| `~/.dsh/profiles/desktop/cordis.patch.yml` | 删掉 `- id: opendesign-go` 整块（第 78–105 行，28 行：accounts 开心/`OPENDESIGN_ACCOUNT_1`、baseURL、vela 凭证开关、usageJson/reportJson、12 个模型配置）；再删 `ui-skin-claude-style` 的 `quickProviders` 中 `- dsh-opendesign-go` 一项（第 89 行） |
| `~/.dsh/profiles/desktop/package.json` | 移除依赖 `dsh-opendesign-go: 0.1.11`（第 43 行） |
| `~/.dsh/profiles/desktop/pnpm-lock.yaml` | 三处：importer stanza（44–47）、packages 块（1301–1332，含 peerDependencies/engines）、snapshots 块（3394–3396）；共 -38 行 |
| `~/.dsh/.credentials.yaml` | 删除 refs 中的 `OPENDESIGN_SESSION_TOKEN`、`OPENDESIGN_API_KEY`、`OPENDESIGN_ACCOUNT_1`（保留 `version / records / refs` 三段结构） |
| 磁盘目录 | `~/.dsh/profiles/desktop/node_modules/dsh-opendesign-go` 与运行态目录 `~/.dsh/dsh-opendesign-go` 整体移到备份区（后者含 `usage.jsonl`：3336 次调用、\$13.78 计费明细） |

改动规模：cordis.patch.yml -29 行、package.json -1 行、pnpm-lock.yaml -38 行、.credentials.yaml -4 行。

## 4. 备份与回滚

- 备份目录：`/Users/liuyongze/.dsh/backups/dsh-opendesign-go-20261102-234816`
  - `cordis.patch.yml` / `package.json` / `pnpm-lock.yaml` / `.credentials.yaml` 原件
  - `node_modules__dsh-opendesign-go`、`state__dsh-opendesign-go`（原目录平移）
  - `RESTORE.md`（回滚说明）、`remove-opendesign-go.py`（清理脚本）
- 清理脚本是**幂等**的：第二次运行输出 `RESULT: no-op, already clean`。回滚或重跑都是一条命令：
  `python3 ~/.dsh/backups/dsh-opendesign-go-20261102-234816/remove-opendesign-go.py`

## 5. 验证证据

- `cordis.patch.yml` 可正常 YAML 解析，插件条目数 **19**；`opendesign present: false`
- `package.json` 合法；`pkg dep present: false | bundles has it: false`
- `pnpm-lock.yaml` → `lock parses: true`，无 `dsh-opendesign-go` 节点
- `.credentials.yaml` → `refs` 只剩 DEEPSEEK_API_KEY、COMMANDCODE_API_KEY_2、DSH_WEIXIN_BOT_TOKEN_*、OPENAI_CODEX_SUBSCRIPTION_OAUTH
- 全 profile 配置 grep `opendesign` = 0；`node_modules/dsh-opendesign-go` 不存在
- 观察 ~90 秒后 patch 层未被运行时回写（`opendesign refs after wait: 0`）

## 6. 生效条件与风险

- 插件在**启动时装载**，改文件后当前进程仍持有旧状态：需 Cmd+Q 完全退出再打开。重启后三点验收：设置页无 Open Design 一栏；模型列表少了 `gpt-6-luna`/`mimo-v2.6-*`/`glm-5.3-*`/`hy4-preview`（commandcode 自带的 glm/kimi/claude 仍在）；左上角不再出现 `amr-link.open-design.ai` 横幅。
- 退出前运行中的进程可能把内存里的插件配置**写回 patch 层**；若重启后横幅/条目复现，重跑上面的幂等脚本再清一次即可。
- 旧会话若 pin 过 Open Design 模型（如某次用 `gpt-6-luna` 的会话），重开时需手动切回 `DeepSeek-V4.1-Flash`（deepseek-account，当前默认模型）。

## 7. 可复用的通用步骤

1. 在 `~/.dsh/profiles/<profile>/cordis.patch.yml` 定位 `- id: <plugin>` 块，连同其它插件对它的引用（如 quickProviders、preset 列表）一并删除；
2. 移除 `package.json` 依赖与 `pnpm-lock.yaml` 的 importer/packages/snapshots 三处；
3. 清理 `.credentials.yaml` 的 `refs` 中该插件专用凭据；
4. 把 `node_modules/<plugin>` 与 `~/.dsh/<plugin>` 运行态目录移到带时间戳的备份目录，附 `RESTORE.md` 与幂等清理脚本；
5. 校验：YAML/JSON/lock 可解析、条目计数、`grep -c` 为 0；
6. 完全退出应用再启动，逐项验收 UI，并留意运行时回写导致的复活。
