---
id: "d723d537-13d1-494b-a535-4c42bf64da7a"
title: "dsh-computer-use：macOS 电脑操作插件实现与验证（2026-11-04）"
description: "dsh-computer-use 插件的架构（Swift 原生助手 + AX 树 + Vision OCR）、7 个工具语义、实测验证结果与踩坑修复，供后续扩展或排障参考。"
status: "active"
created_at: "2026-11-04T09:20:54.359Z"
updated_at: "2026-11-04T09:20:54.359Z"
content_hash: "e9c878b50a2c96928e718719c4db999c1077b724c73d8e50dc80f01b8ec8dde0"
source_paths:
  - "plugins/dsh-computer-use/README.md"
  - "plugins/dsh-computer-use/native/ComputerUse.swift"
  - "plugins/dsh-computer-use/lib/tools.js"
  - "plugins/dsh-computer-use/lib/screen.js"
  - "plugins/dsh-computer-use/lib/helper.js"
session_ids:
  - "f4e41702-10df-4fd8-9063-d382e467aca4"
memory_body_ids:
  []
---

# dsh-computer-use：macOS 电脑操作插件实现与验证

日期：2026-11-04。目标：让 Agent 不只会「截图猜坐标」，而是能读懂屏幕、按语义操作这台 Mac。

## 架构

- 源码 `/Users/liuyongze/plugins/dsh-computer-use`，以 link 方式装进 desktop profile（entryId `include:computer-use`）。
- 唯一原生部分是 `native/ComputerUse.swift`（约 700 行，只链 AppKit/CoreGraphics/Vision），首次使用时 `swiftc` 编译到 `~/.dsh/computer-use/bin/dsh-cu`，按源码哈希缓存，升级自动重编。
- 刻意不依赖 pyautogui / cliclick / robotjs；命令通过 argv 传 JSON、stdout 回 JSON。
- JS 侧：`lib/tools.js`（工具注册）、`lib/screen.js`（读屏/OCR 组织）、`lib/helper.js`（编译缓存与原生调用）。
- 输入是全局合成事件，发给当前前台窗口，插件不会自动切窗口；`blockedApps` 命中时拒绝发送输入。

## 工具语义（7 个）

| 工具 | 职责 |
| --- | --- |
| `computer_read` | 一次拿到辅助功能树（role/标题/坐标/index）+ OCR 文字行（带坐标），默认只读前台窗口 |
| `computer_act` | 按 index 或标题执行 `press` / `focus` / `set_value` / `click`，走 AX，后台窗口也能操作 |
| `computer_screenshot` | 全屏/窗口/区域截图，图片直接进模型上下文 |
| `computer_windows` | 列窗口、查前台 app、activate |
| `computer_mouse` | move / click / double_click / right_click / drag / scroll / position |
| `computer_keyboard` | type（中文、emoji）/ key / hotkey |
| `computer_permissions` | 检查辅助功能与屏幕录制授权，可打开设置面板 |

推荐工作流：`computer_read` 找元素 → `computer_act` 按 index/标题操作 → 再 `computer_read` 复核。坐标统一为屏幕 point 坐标，原点在主屏左上角；截图默认降采样到 point 尺寸，所以图上像素即点击 point。

## 踩坑与修复（有实测依据）

1. **AX path 漂移**：界面变化后固定 path 会指到别的控件（早期算 7×8 得到 89，实际点错按钮）。修复：每次操作携带「role + 标题 + 坐标」指纹，path 失效时在树中重新查找同一控件。
2. **打字丢字**：2ms 间隔分块太快，后续块被目标 app 丢弃。修复：单事件整串（≤16 字符）或 15ms 间隔分块；1500 字符逐字回读一致。
3. **OCR 语言顺序**：Vision `en-US` 排第一会让中文识别全乱且耗时约 14s。修复：`recognitionLanguages` 固定 `zh-Hans` 在前，原分辨率精确模式，窗口级读屏约 0.3–1s。
4. **swiftc 加载标准库失败**：改为通过 `xcrun swiftc` 调用。
5. **自绘界面**（Electron 画布/游戏）AX 元素稀少，需退回 OCR + 坐标点击。

## 验证结果

- 计算器 7×8：读屏取按钮 → 语义 press → 显示「7 × 8」结果为 56。
- 后台窗口 `set_value`：TextEdit 文本被改写，osascript 回读一致。
- 中文 + ASCII + emoji 打字：25/27/300/1500 字符逐字回读全部正确。
- 截图：1512×982 point，`imageAttached=true`。
- 授权：本机辅助功能 + 屏幕录制均已授予。

## 配置与运维

`cordis.patch.yml` 可调 `allowInput`、`resolution`（points 推荐）、`defaultDisplay`、`blockedApps`、`typeMaxChars`、`warmup`。截图落在 `$TMPDIR/dsh-computer-use/`，20 分钟自动清理。

改完 `lib/*.js` 需用 `plugin_manager set_plugin target=include:computer-use enabled=false` 再 `true` 重载（entryId 不是 `computer-use`）。卸载：`dsh plugin --profile desktop remove dsh-computer-use` + `rm -rf ~/.dsh/computer-use`。
