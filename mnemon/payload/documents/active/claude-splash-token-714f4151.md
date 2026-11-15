---
id: "714f4151-3039-4e36-86c3-b880e625273b"
title: "Claude Splash：螃蟹变蓝的 token 覆盖根因与配色定位方法"
description: "claude-splash 开屏螃蟹/Claude 侧元素变蓝的根因链（--csp-accent 绑定宿主 --dsw-alias-brand-primary，皮肤配置 brand: deepseek 注入 #4d6bfe）、最小修复与派生色表，附像素蟹图宽度校验失败与 makeSoundtrack 未声明变量导致整树不渲染的两处连带修复及验证手段。"
status: "active"
created_at: "2026-10-30T12:28:17.581Z"
updated_at: "2026-10-30T12:28:17.581Z"
content_hash: "0e601da1757ae641956e37ee4996cb1dd09c8ec1601118e5a767b175aa32b8eb"
source_paths:
  - "Desktop/dsh-plugins/claude-splash/client.js"
session_ids:
  - "9b3f9072-eab2-4d0f-bb82-ac542c460010"
memory_body_ids:
  []
---

# Claude Splash：螃蟹变蓝的 token 覆盖根因与配色定位方法

补充文档，只记录本轮新增结论。动画编排见 `b05ca3a1-8032-4e48-b292-8ac82c7cc546`（Claude Splash 蟹鲸对决动画实现要点），白闪/暗夜幕/Web Audio 与双色激光见 `a8ec51cb-79f4-4373-ada6-c9afeec3170c`，「动画整段消失」的只读排障路径见 `63c07518-7e42-7f81-...`（Claude Splash：PASS/FAIL 音色重做与「动画消失」排障记录，id 以实际文档为准：`63c07518-7e42` 段请在搜索中复核）。

## 1. 症状与结论

- 症状：开屏里 Claude 一侧（螃蟹 + 光球 + 橙色激光）整体偏蓝，与鲸鱼撞色；用户先反馈「螃蟹颜色不对」，再指出「最开始的 claude 怎么是蓝色的」。
- 结论：不是绘制代码写错了色值，而是 **CSS 变量的 fallback 被宿主 token 顶掉**。

## 2. 根因链（可复用的排查顺序）

1. 实现里原本是：
   `--csp-accent: var(--dsw-alias-brand-primary, #d97757);`
   逗号后的 Claude 橙只是 fallback，**只有宿主完全没定义该 token 时才会生效**。
2. 宿主皮肤 `dsh-claude-style` 在 profile 的 `cordis.patch.yml` 中配置为 `brand: deepseek`、`paletteColors: deepseek`（`id: ui-skin-claude-style`），注入 `--dsw-alias-brand-primary: #4d6bfe`（DeepSeek 蓝）。
3. 螃蟹的壳/钳/腿全部由 `color-mix(in oklab, var(--csp-accent) …)` 派生；`.csp-figure--crab`、`.csp-orb--claude`、橙色激光的 `--lc` 也指向它 → 整个 Claude 侧被染成蓝色。
4. 鲸鱼侧用的是独立变量 `--csp-whale: #4d6bfe`，于是两边同色。

定位手法（有效且低成本的组合）：
- `Theme/listTokens` 输出确认宿主确实提供 `--dsw-alias-brand-primary`（14 个 alias token 列表）。
- 在皮肤 CSS 里 grep `--dsw-alias-brand-primary:` 的实际赋值，发现 `#4d6bfe` 与 `#d97757` 两种值并存 → 说明它随皮肤配置变化。
- grep 源码里对该 token 的引用点（20+ 处），确认只有 `--csp-accent` 这一处把美术配色绑到了宿主品牌色。

## 3. 最小修复

`client.js` 中把该行改为定值，只服从 Claude 皮肤色：

```
'  --csp-accent: ' + CLAUDE_INK + ';',     // CLAUDE_INK = '#d97757'
'  --csp-whale: ' + DEEPSEEK_INK + ';',    // DEEPSEEK_INK = '#4d6bfe'
```

要点：美术/品牌色是「插件自己的设计资产」，不该继承 `--dsw-alias-brand-primary`，否则皮肤换品牌（deepseek / brand / crab / deepy 多套 mascot 预设）就会串色。宿主 alias token 只适合用在需要跟随主题的表面色（bg / label / border / line）。

## 4. 派生色结果（accent = #d97757）

| 部件 | 派生公式 | 实际色 |
|---|---|---|
| 壳 `#` | accent 82% + `#c2410c` | `#d56d4a` |
| 钳 `[` `]` | accent 68% + `#b23a12` | `#cd6341` |
| 腿 `d` | accent 48% + `#7c2d12` | `#a95133` |
| 眼 `k` / 眼白 `w` | ink / 暖白 | `#141413` / `#fffdf7` |

验证方式：写临时脚本抽取 `CRAB_PALETTE` 的 `color-mix` 定义并按 oklab 近似计算，得到上述三档色；再离屏渲染成 560×560 JPG 目检（壳橙、钳略深、腿最深）。

## 5. 连带修复（同一轮）

- **像素蟹图宽度不一致**：`CRAB_ART` 第 3 行 `'[..+##########++..]'` 长 21，其余行 20 → `art-validation` 报 `1 row(s) with wrong width`，渲染时整只蟹错位。修成等宽后校验通过。校验脚本会打印每行 glyph 种类数（`CRAB glyph variety per row` / `WHALE glyph variety per row`）辅助人工核对。
- **`makeSoundtrack` 未声明变量导致整棵组件树不渲染**：函数内使用了未声明的 `sides`，在 `useEffect` 中同步抛 `ReferenceError`，React 直接卸载整段 splash（表现为「动画整段不出现」，而不是配色问题）。最小修复：
  `const sides = Array.isArray(plan && plan.sides) ? plan.sides : [];`
  排障教训：同一文件里既有「样式/配色问题」又有「组件完全不出现」时，先看渲染期异常，再谈观感。

## 6. 回归检查清单（本插件通用）

- `node --check client.js`
- `tools/validate.mjs`：括号配平、无 undefined/NaN、`@keyframes` 声明/引用数一致、class 全覆盖、节拍数、时间线、CSS 字节数
- 音频桩渲染（含真 `AudioContext` 桩）确认无 `EFFECT THROW`，节点数与时延正常
- 软链两端 sha 一致（`.dsh/profiles/desktop/node_modules/@local/claude-splash/client.js` 与 `Desktop/dsh-plugins/claude-splash/client.js`），改源文件即生效，无需重装
- 改配色后重新计算派生色并出图目检，避免只改一处却漏掉 color-mix 派生链
