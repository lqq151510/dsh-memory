---
id: "638514ae-9218-46f5-9601-e27d760c2647"
title: "OpenBot（OpenMausBot 二开）重品牌与 macOS 打包交付"
description: "OpenBot 项目：以 Apache-2.0 上游源码 fork 而非反编译 asar 的取舍、全仓重命名范围、electron-builder arm64 dmg 构建命令与产物、本机环境坑（ELECTRON_RUN_AS_NODE、pnpm 版本、Google 镜像）、已验证/未验证层级与待决事项。"
status: "active"
created_at: "2026-11-08T08:04:05.610Z"
updated_at: "2026-11-08T08:04:05.610Z"
content_hash: "4a6b5d073a8ae9fa08321730ba33217d26d832243ff804f774d81b7379579207"
source_paths:
  - "Users/liuyongze/opensource/open-bot/release/OpenBot-0.1.0-arm64.dmg"
  - "Users/liuyongze/opensource/open-bot/release/OpenBot-installed-screenshot.png"
session_ids:
  - "10602087-e015-4b5d-a9b6-c0c3a0cff22a"
memory_body_ids:
  []
---

# OpenBot（OpenMausBot 二开）重品牌与 macOS 打包交付

时间：2026-11-08 · 项目根 `/Users/liuyongze/opensource/open-bot` · 机器：macOS arm64

## 目标与关键决策

用户给来一个 54,042,598 字节的 `cua.tar.gz`（sha256 `af23b26a6e2e1cd0917536244ca0050b1447f2d344c958cdb79bfb3342282268`），内为 `OpenMausBot.app`（Electron + TypeScript）。目标：做出自有品牌版本并可用。

- 决策：**不反编译 400MB 级 asar**，而是取上游 Apache-2.0 源码 fork（`milind-soni/OpenMausBot`）。理由：许可证允许、可维护、不残留他人品牌痕迹，改造成本低于逆向。
- 合规：仅在 `NOTICE`/`README` 保留上游 MIT/Apache 归属，其余全部替换。
- 身份：`name=openbot`、`appId=com.openbot.app`、协议 `openbot://`、更新源指向自有仓库 `lqq151510/open-bot`。
- 重品牌范围：`OpenMausBot→OpenBot` 覆盖 1516 个文件、19 个文件名/路径，含内部标识符（如 `mausBotWelcomeDue`）；图标重绘为深蓝底白色机器人头 + 青色天线，生成 `.icns`/`.png`/`.svg` 全套，替换原灰色三角。

## 构建与产物

- 工具链：electron-builder 26.15.3，Electron 43.7.9，arch arm64，pnpm 10.33.0。
- 命令（注意前面的环境变量）：
  ```
  PATH=/tmp/pnpm10/bin:$PATH CSC_IDENTITY_AUTO_DISCOVERY=false \
    ./node_modules/.bin/electron-builder --mac dmg --arm64 --publish never
  ```
- 产物：`release/mac-arm64/OpenBot.app`（797MB）、`release/OpenBot-0.1.0-arm64.dmg`（304MB）、`.dmg.blockmap`、`latest-mac.yml`（`EXIT=0`）。
- 打包内置：本地 server（`127.0.0.1:8799`，`OMB_PORT`/`OMB_HTTP_PORT`）、cua-driver 0.41.3（上游已有 0.43.2）、headless Chromium（`browser-engine`）、`android-platform-tools`、`cloudflared`。

## 环境坑（重建必看）

- 全局 `ELECTRON_RUN_AS_NODE=1` 会让 Electron 以 Node 模式启动，必须 `env -u ELECTRON_RUN_AS_NODE ./node_modules/.bin/electron .`。
- Homebrew 的 pnpm 12.9.1 缺原生二进制，改用 pnpm 10.33.0 包装器（`/tmp/pnpm10/bin`）。
- `dl.google.com`、`storage.googleapis.com` 在本机极慢/被重置；改用腾讯、npmmirror、gh-proxy 镜像后 Chromium 下载从 ~40KB/s 升到 ~700KB/s（103MB）。
- electron-builder 全程远超单次 120s 执行预算：必须以后台任务跑 + 轮询日志，前台 `run_code` 会 timeout 而进程仍在。
- 二次启动会打印 `[desktop] OpenBot is already running — focusing that window`，不是崩溃。

## 验证层级（已做 / 未做）

已做：dmg 构建成功 → `hdiutil attach` 挂载（卷 `/Volumes/OpenBot 0.1.0-arm64`，含 Applications 软链）→ 安装到 `/Applications/OpenBot.app`（768MB，机器上原先无同名 App）→ 启动进程 → 端口 8799 监听 → 窗口可见并截图：
- `release/OpenBot-packaged-screenshot.png`（打包版）
- `release/OpenBot-installed-screenshot.png`（安装版实跑）

未做：代码签名/公证（当前为 `adhoc` 签名，本机可用，分发需右键“打开”）；Git 提交与推送（工作区有 1566 个未提交改动，等待授权）；云端/手机连接仍指向占位域名 `openbot.com`。

## 待决事项

1. 界面机器人形象仍是上游灰色三角（约 30 多个表情状态的矢量状态机），是否按新图标重绘。
2. 是否做开发者签名 + 公证。
3. 是否把 1566 个改动提交并推送到 `lqq151510/open-bot`。
