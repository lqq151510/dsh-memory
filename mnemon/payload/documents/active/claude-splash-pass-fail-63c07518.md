---
id: "63c07518-ac7e-4277-801f-e1d49a7b8911"
title: "Claude Splash：PASS/FAIL 音色重做与「动画消失」排障记录"
description: "claude-splash 单文件 client 插件的两项本轮结论：PASS/FAIL 音效从同模板改为一听可辨的两种构成，并把文案与音色统一到同一 verdict 函数；以及「开屏动画不见了」的只读排障路径（一次性生命周期、boot 日志位置、ESM 桩渲染复核）。"
status: "active"
created_at: "2026-10-30T11:51:27.335Z"
updated_at: "2026-10-30T11:51:27.335Z"
content_hash: "e8a8048be0d997b1cb706bed0f09001f4147a5759bfb20b8721d192d4af35720"
source_paths:
  - ".dsh/profiles/desktop/node_modules/@local/claude-splash/client.js"
session_ids:
  - "3e82ad8a-5939-44d0-9406-2658d6a31f3b"
memory_body_ids:
  []
---

# Claude Splash：PASS/FAIL 音色重做与「动画消失」排障记录

补充文档，只记录本轮新增结论。动画编排见 `b05ca3a1-8032-4e48-b292-8ac82c7cc546`，视觉与 Web Audio 信号链、白闪/夜幕、激光配色见 `a8ec51cb-79f4-4373-ada6-c9afeec3170c`。

- 生效实现：`.dsh/profiles/desktop/node_modules/@local/claude-splash/client.js`（经 `link:` 指向插件源码目录 `~/Desktop/dsh-plugins/claude-splash`，软链两端 sha 一致；本轮结束时 122,414 字节，sha `955d473d…`）。
- 注册：`shell.overlay` 槽，entryId `claude-splash`，order 1000；factory 只 `inject: ['slots','locale']`。

## 1. 「通过和报错的声音不同」——原来并不同

根因不是音量或时长，而是**同一个音色模板**：PASS 与 FAIL 都走同一个 `osc` 正弦模板，只把起点换成 1046 / 784 Hz 再各自上/下滑音，且都紧跟在同一个 boom 之后 ~20ms 内发声。频率接近 + 音色相同 + 时间重叠，听感上无法区分。

改法是让两条声音在**音色、走向、频段、混响量**四项上同时分开：

| 维度 | PASS（上行大三和弦） | FAIL（下行方波互打） |
|---|---|---|
| 主体 | 1046.5 / 1318.5 / 1568 Hz 正弦依次叠加上行（C6-E6-G6），逐层降音量 | 392→262 Hz 方波下行 |
| 关键特征 | 泛音层用三角波，更亮 | 叠一个 415→277 Hz 方波，与主体差 23 Hz 产生拍频「嗡嗡」声，主观即「不对劲」 |
| 低频/噪声 | 6.2k→15k 极短高频闪光（金属「叮」） | 131→92 Hz 低频闷响 + 180 Hz 以下低通噪声（砂纸感） |
| 混响 | 0.34，像玻璃风铃干净收尾 | 0.16–0.2，收得干脆发闷 |

判断标准是「不看画面也能分出哪个是报错」，不是靠响度差。

## 2. 顺带修的匹配 bug：文案与音色曾是两套逻辑

芯片文字按 `(i * 3 + 1) % 8` 从 8 条文案里选，音色却按 `i % 3 !== 1` 判成败，两者不保证一致——会出现屏上写 `✓ PASS` 却响失败音。

修法：把 `VERDICTS`、`verdictOf(i)`、`isFail`、`isPass` 提到模块作用域，画面与声音读同一函数。经验：**同一语义的随机/判定只能有一个来源**，视觉和音频各写一套取模必然漂移。

## 3. 「动画怎么没了」的排障结论

这是**一次性**组件，不是常驻层：挂载 → 播 8.82s → 收尾约 0.76s → `return null` 自我卸载。因此 `shell.overlay` 里 `claude-splash` 显示 `active: false` 且 cell 无节点，**是「已播完卸载」的正常态，不是加载失败**（同槽 `bili-upstreams`、`shortcuts` 等仍 `active: true`）。页面长时间未重载时动画早就播完了；渲染进程若是重连而非重载，插件模块不会重新求值，也就不会重播。⌘R 刷新即重播。

核实「插件到底有没有加载」的只读路径（按成本从低到高）：

1. **boot 日志**：`~/Library/Logs/DeepSeek Harness/crash-*-web-boot.log`。整插件失败时表现为 `Error: web boot: 1 entry did not activate` + `@local/claude-splash: import failed`，且 renderer console 会带同一条。注意日志时间戳与改文件的中间态对齐——本轮 18:39 / 18:44 两次失败正是编辑中间态，19:46 重启后无失败记录，说明当时是好的。
2. **槽位实况**：`cordis_inspect_query` 的 client `Slots` / `Service` provider 可列槽位 occupants（含 `active`），`plugin_manager` 列 entry 的 `fiberPhase`。
3. **ESM 桩渲染**：在 Node 里架 `window.__ModuleLoader__` + React 桩真实 import 模块，断言 factory keys、`inject`、locale key 数、注册项、渲染树 `csp-root`、子节点数、style 节点、定时器延迟（本轮 4242 / 8820 ms，均非 NaN）。
   - 坑：桩环境缺 DOM 时会出现 `EFFECT THROW: window.addEventListener is not a function`，这是桩的限制，**不是插件 bug**，别据此下结论。
   - 桩只能证伪「模块级抛错 / 注册丢失 / 首渲染 NaN」，证不了真实浏览器里的运行期错误；两边不一致时要 DevTools Console 的具体红字。

排查顺序总结：先看 boot 日志与槽位状态区分「没加载」和「播完卸载」，再用桩渲染缩小到模块层，最后才要控制台报错。

## 4. 校验基线

`node --check` 通过；validate 全绿：92 组 keyframes（声明/引用各 92）、125 个渲染 className 节点、总时长 8.82s、CSS 57,225 bytes；抽样 5 枚芯片 = 2 PASS + 3 FAIL，确认两种音色都有机会出现。音频仍受浏览器自动播放策略限制，需一次用户点击解锁。
