---
id: "53636080-86e7-4b3b-9c57-5aa2af2c5a42"
title: "zebao-skin：DSH 浅色太空背景皮肤插件（结构、安装、验证）"
description: "纯背景皮肤插件的目录结构、desktop profile 三处安装改动、动态星空与玻璃参数设计、客户端 bundle 需 ⌘Q 重启才生效的环境特性，以及验证与回滚步骤。"
status: "active"
created_at: "2026-11-05T11:36:56.370Z"
updated_at: "2026-11-05T11:36:56.370Z"
content_hash: "07302491da4711519eab1b9f9917319ad3aadf99b3ff49b172e59d7e6f1a1356"
source_paths:
  - "plugins/zebao-skin/lib/client.src.js"
  - "plugins/zebao-skin/lib/skin.css"
  - "plugins/zebao-skin/README.md"
  - ".dsh/profiles/desktop/package.json"
session_ids:
  - "289bde2e-507f-42cf-ab64-7a88dc15531a"
memory_body_ids:
  []
---

# zebao-skin：DSH 原生 UI 之上的纯背景皮肤插件

浅色太空底皮肤插件，只叠加背景层，不改 DSH 任何原生组件。用户要求：保持原版 UI、浅色、太空风格 + DeepSeek 元素、可自定义透明度/磨砂/玻璃质感、可上传本机图片或视频、背景动态。

## 位置与产物

- 插件源码：`/Users/liuyongze/plugins/zebao-skin`
  - `lib/client.src.js`：源文件（分层挂载、IndexedDB 素材、设置面板、视差、浅色处理）
  - `lib/client.js`：构建产物（约 38 KB；`node --check` 通过）
  - `lib/skin.css`：舞台 / 磨砂 / 前景 / HUD 边框 / 面板全部样式（约 15 KB）
  - `lib/index.js`：host 侧入口（约 0.5 KB）
  - `README.md`：可调项、安装位置、开发与回滚
  - `preview/shot-light.png`、`preview/shot-panel.png`：模拟三栏预览
- 安装：`/Users/liuyongze/.dsh/profiles/desktop/node_modules/zebao-skin` → 软链到 `../../../../plugins/zebao-skin`

## 安装到 desktop profile 的三处改动

1. 在 `/Users/liuyongze/.dsh/profiles/desktop/package.json` 的 `dsh.profile.bundles` 数组里登记 `zebao-skin`（该数组当前含 dsh-base、dsh-web-app、beauticode-dsh、zebao-splash、dsh-pet 等）。
2. 在 `dependencies` 中写 `"zebao-skin": "link:/Users/liuyongze/plugins/zebao-skin"`（本机插件沿用 link: 约定）。
3. 在 `node_modules` 下建立同名软链。

插件自己的 `package.json` 只需 `dsh.bundle.patch: ./cordis.patch.yml` 与 `dsh.client.platform: web`；同机轻量插件（dsh-pet、beauticode-dsh）也是这个形状。

## 关键取舍与机制

- 纯皮肤：只创建固定定位的背景舞台并把 `#root` 抬到 z-index 1 且背景 transparent，不动任何组件 DOM；这是“保持原版 UI”要求的直接实现方式。
- 动态背景不依赖视频：程序化生成冷白星云 + 极光漂移 + 星点 / 轨道 / 行星地平线 / 鲸形星座（DeepSeek 元素）+ 流星。
- 玻璃：磨砂 0–30px、玻璃质感 0–100%，叠在星空之上；表面透明度滑杆默认 62%、磨砂默认 10px。
- 上传素材（图片或视频，视频自动静音循环铺满）存 IndexedDB，重启不丢；`清除` 回到程序化星空。
- 面板挂在右缘 `◈`，5 个滑杆即时生效并持久化；亦可用 `window.__zebaoSkin.set({ alpha: 80, blur: 6 })` 从控制台调。

## 已知环境特性（重要）

客户端 bundle 只在 DSH 启动时加载：改源码或首次安装后必须完全退出（⌘Q）再打开，单纯刷新网页不生效。这是 DSH 客户端插件的通用加载时机，不是本插件的 bug。

## 验证状态

- 已完成：`node --check`；headless Chrome 模拟三栏预览，确认图层顺序、磨砂、面板渲染（1440×900 下 1440px 有效宽）。
- 未完成：真实 DSH 进程内重启后的实际观感，需用户 ⌘Q 后确认。截图与模拟预览不能替代这一层验收。

## 回滚

设置 → 插件里禁用 `zebao-skin`；或按 README 步骤从 profile 的 bundles / dependencies 中移除并删除软链。

## 相关

- 同机另有 `zebao-splash`（`/Users/liuyongze/plugins/zebao-splash`，开机动画 + 已禁用的聊天壁纸半侧 `CHAT_WALLPAPER_ENABLED = false`），两者都靠层叠背景浮层实现，但 zebao-splash 保留设置页 UI、zebao-skin 不碰 UI。
- 机器上还装了 `beauticode-dsh`（`/Users/liuyongze/.dsh/profiles/desktop/node_modules/beauticode-dsh`），同样提供图片/视频背景能力；本插件是独立实现，与它无代码复用。
