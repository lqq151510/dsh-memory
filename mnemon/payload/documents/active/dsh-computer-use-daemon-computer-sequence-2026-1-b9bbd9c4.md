---
id: "b9bbd9c4-2b5f-4136-be3f-e8e68dbcacce"
title: "dsh-computer-use 游戏级响应优化：低延迟输入 daemon 与 computer_sequence（2026-11-05）"
description: "dsh-computer-use 游戏模式增量：CUInput 常驻 daemon 架构、实测延迟数据、computer_sequence 工具、TextEdit AXDialog unicode 排障结论与交付状态。"
status: "active"
created_at: "2026-11-05T08:37:47.111Z"
updated_at: "2026-11-05T08:37:47.111Z"
content_hash: "ccc8055a4cc9a48ef4a1ef255b37ba309b812fe40c99b323cf8db88f995984e2"
source_paths:
  - "plugins/dsh-computer-use/native/CUInput.swift"
  - "plugins/dsh-computer-use/lib/helper.js"
  - "plugins/dsh-computer-use/lib/tools.js"
  - "plugins/dsh-computer-use/README.md"
session_ids:
  - "55e23f3b-be52-4ba6-9449-677669b7940d"
memory_body_ids:
  []
---

# dsh-computer-use 游戏级响应优化交付（低延迟输入 daemon + computer_sequence）

日期：2026-11-05。承接《dsh-computer-use：macOS 电脑操作插件实现与验证（2026-11-04）》（id `d723d537-13d1-494b-a535-4c42bf64da7a`，7 工具版本架构与踩坑），本文只记录其后新增的游戏模式层。源码 `/Users/liuyongze/plugins/dsh-computer-use`。

## 核心改动：常驻输入 daemon

- 新增 `native/CUInput.swift`（约 137 KB，只链 CoreGraphics）：鼠标键盘事件改走常驻 daemon（unix socket），空闲 900 秒自退；替代此前「每次调用 fork+exec 胖助手」的通道。
- 首次调用含拉起 daemon 约 325 ms，之后走 socket。

## 实测延迟（真机）

| 操作 | 之前（每次起进程） | 现在（daemon） | 提升 |
| --- | --- | --- | --- |
| move | 36 ms | 0.16 ms | ~225× |
| key 单键 | 91 ms | 10.6 ms | ~8.6× |
| click 单击 | 158 ms | 5.2 ms | ~30× |

key 10.6 ms 已到 CGEvent 可靠下限（内部只剩 2+4+2 ms 投递间隔）。

## 工具能力扩展（现为 8 个 computer_* 工具）

- `computer_mouse`：move 支持 dx/dy 相对位移（游戏视角转动）；click 连发 repeat/intervalMs/downMs；新 action=hold 按住鼠标键；humanize 拟人轨迹（贝塞尔、自然按压、微抖动）；game 低延迟标志。
- `computer_keyboard`：key 加 holdMs（游戏漏检按键时调大到 16–30 ms）和 game 标志。
- `computer_sequence`（第 8 个新工具）：steps[] 每步 move/click/hold/drag/scroll/key/type/wait + afterMs，一次调用跑完整连招，返回每步 tookMs。

## 修复的两个隐藏 bug

1. Node Socket 只读 `pending` 属性冲突 → 改名 `cuPending`。
2. `net.connect` 只试一次 → 改成轮询重试。

## 真机验收证据

- 相对移动 dx=100：光标 990.82→1090.82，精确落点。
- TextEdit 连招（按住 w 300 ms → 连招打字 → 回车）一次调用全对，步骤耗时 361/19/38 ms。
- 连发 5 次@30ms=164 ms；按住 500 ms 实测 504 ms；拟人移动 378 ms。

## 排障结论：TextEdit AXDialog 窗口吞 unicode 合成事件（可复用教训）

现象：keycode-0 合成的 unicode 打字在某个 TextEdit 窗口不落盘。对照实验定位：

- Terminal 在拼音 IME 下 unicode 打字正常 → 不是 IME 吞事件；
- 切 ABC 输入源后该 TextEdit 仍不通 → 不是输入源；
- 同一 TextEdit 新建文档窗口立即正常 → 根因是旧文档窗口处于异常状态（subrole=AXDialog 的 AXWindow），与插件、IME、系统无关。

结论：遇到「合成 unicode 事件不落盘」先怀疑目标窗口状态（AXDialog subrole），换新窗口即可验证，不要先改输入源。

## 边界声明

humanize 做的是正当的拟人输入手感，用于让操作观感接近人手；不用于绕过 CAPTCHA 或任何平台反作弊检测。

## 交付状态

- commit `98682d6`（feat: low-latency input daemon, game mode and computer_sequence，7 文件 +1119/−59）已推 `lqq151510/dsh-computer-use` 与 `dsh-computer-use-0.1.0`（v010 镜像）。
- 收录 PR awesome-dsh-plugin#11963 描述已更新为 8 工具；仓库年龄闸 2026-11-05 约 18:10 后自动转绿。
- 改 `lib/*.js` 后的热重载流程见运行时记忆条目（plugin_manager set_plugin target=include:computer-use false→true）。
