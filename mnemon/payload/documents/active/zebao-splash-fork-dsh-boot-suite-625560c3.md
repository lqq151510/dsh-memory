---
id: "625560c3-a139-4011-bb1e-eb874fb9be6e"
title: "zebao-splash：私有开屏插件 fork 与 dsh-boot-suite 移除记录"
description: "本机开屏动画改为私有 fork zebao-splash 后的差别清单：逐幕配音回到自带真声素材、终幕不叠 TTS、上游更新与模板商店入口置空、dsh-boot-suite 目录与锁文件条目全部清除，附可复用的预览验证方法。"
status: "active"
created_at: "2026-11-05T07:09:45.488Z"
updated_at: "2026-11-05T07:09:45.488Z"
content_hash: "9c625e0fbd722219fe921201df23fe58b4255a0f76eb64b2987986f0987a254d"
source_paths:
  - "plugins/zebao-splash/README.md"
  - "plugins/zebao-splash/lib/host-startup.js"
  - "plugins/zebao-splash/lib/client.js"
  - "plugins/zebao-splash/lib/splash.js"
  - "plugins/zebao-splash/templates/archive-warm/template.json"
session_ids:
  - "21e08c28-2394-4cd1-9d93-c14fd1968a57"
memory_body_ids:
  []
---

# zebao-splash：私有开屏插件 fork 与 dsh-boot-suite 彻底移除

本机 DSH 开屏动画的实现来源已从上游插件 `dsh-boot-suite` 切换为本地私有 fork `zebao-splash`，两者不再有任何代码或更新通道关联。本文记录该 fork 的差别、配音方案的最终决定、以及上游清理的范围与验证方式。

## 位置与形态

- 源码：`/Users/liuyongze/plugins/zebao-splash`（约 6.1M），核心文件 `lib/splash.js`、`lib/client.js`、`lib/host-startup.js`、`lib/index.js`、`cordis.patch.yml`、`package.json`。
- 安装：`~/.dsh/profiles/desktop/node_modules/zebao-splash` 软链到源码目录；desktop profile 的 bundles 列表里只剩 `zebao-splash`。
- 模板：`templates/archive-warm/template.json`（档案冷白 · 浅蓝科幻改版，`schemaVersion:1`，`id: archive-warm`）、`templates/whale-girl-scan/template.json`（第二幕）。
- 配置入口保持不变：`$DSH_HOME/dsh-startup.json`（宿主侧）+ 设置页里的启动动画面板。
- 身份默认写死为 刘勇泽 / 1510 / 权限 6；配色冷白底 + 浅蓝科幻，`accent: '#3f9cff'`，footer `POWERED BY DEEPSEEK`。

## 相对上游 2.16.5 的关键差别及理由

1. **逐幕配音回到自带真声素材**：`voiceStepEn` 置空（默认值与 `dsh-startup.json` 都清空），`stepLoop` 回落到 `voice.line(step.say, step.zh, step.audio)`，第 1–5 幕播 `voice/f/line-1..5.mp3`，终幕播 `voice/f/line-6.mp3`。原因：素材音色统一，原版固定英文台词；填了 `voiceStepEn` 才会改走系统 TTS，混念改写的科幻中文会很突兀。
2. **终幕只播 line-6，不叠系统 TTS**：素材念不出中文名字，名字只出现在画面上。想恢复「名字也念出来」时把 `voiceWelcomeZh` / `voiceWelcomeEn` 填回文案。
3. **堵死回灌上游**：`lib/host-startup.js` 里 `UPDATE_URL`（GitHub latest release）与 `STORE_INDEX_URL`（`raw.githubusercontent.com/LeeRolling/dsh-boot-suite-templates/main/index.json`）置空，`readLatestRelease()` / `readStore()` 直接抛「已停用：zebao-splash 是私有 fork」，检查更新与模板商店只在本地文件夹里找。原因：避免上游发行包覆盖本地改版。
4. 程序员特效默认开启；开头音频被浏览器挂起时会补播；终幕留白压缩（`welcomeDelay` 450ms，素材收尾后 60ms 接第一句）。
5. 24 处 `dsh-boot-suite` 字面量统一改名为 `zebao-splash`（client.js 4、host-startup.js 16、index.js 2、cordis.patch.yml 1、package.json 1）；README 与模板 `homepage` 有意保留 LeeRolling 原仓库地址作为 MIT 来源署名。

## dsh-boot-suite 的清理范围（2026-11-05）

- 删除目录：`~/.dsh/profiles/boot-dev`、`~/opensource/dsh-boot-suite`（82M 源码）、`~/.dsh/backups/boot-suite-uninstall-20261105`、诊断 HTML；`profiles/` 现仅剩 acp、bili-duty、desktop、node_modules、superagent 系列、web。
- 清清单条目：desktop 的 `pnpm-lock.yaml`、`node_modules/.pnpm/lock.yaml`、`node_modules/.modules.yaml`、`pnpm-workspace.yaml`、`.pnpm-workspace-state-v1.json`、`.dsh-market` 缓存里的 boot-suite 条目（remaining 0），改后用 YAML/JSON 解析校验通过；`package.json` 依赖与 bundles 中均不再有它。备份后缀 `.bak-delboot-20261105`。

## 验证方法（可复用）

- 起本地预览：`/Users/liuyongze/.dsh/splash-scifi-preview/harness-zebao.html` 通过仅回环 HTTP 服务（端口 8793）打开，用 ego-browser 注入探针统计 `speechSynthesis` 调用、网络 `voice/*.mp3` 请求与站点阶段文本。
- 通过标准：幕序 `00,01,01,02,03,04,05,06,06,done` 走完；请求只出现 `/dsh-startup/voice/f/line-1..6.mp3`；`tts` 仅一条预热空串，无系统 TTS 混入。
- 一次性生命周期：`sessionStorage['dsh-startup-screen:shown']`（client.js 在 `mode==='session'` 时写 1 并在已为 1 时跳过），排查「动画不见了」先清这个键。
- 改完 JS 用 `node --check` 全过，再重启 DSH（完全退出，不是刷新）才会加载新开屏。
