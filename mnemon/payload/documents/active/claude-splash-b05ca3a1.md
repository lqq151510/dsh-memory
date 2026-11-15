---
id: "b05ca3a1-8032-4e48-b292-8ac82c7cc546"
title: "Claude Splash 蟹鲸对决动画实现要点"
description: "claude-splash 整屏动画的随机对决编排、鲸鱼镜像朝向、换位/瞬移特效、末日层挂载与单文件组件回归验证方法。"
status: "active"
created_at: "2026-10-30T11:23:10.732Z"
updated_at: "2026-10-30T11:23:10.732Z"
content_hash: "0ab2ffb6b61ab759cb3b357fb869ac6cd0637348341a0f874bd205814260ec93"
source_paths:
  []
session_ids:
  - "093793f0-9b8a-449b-8590-92d5c9f07667"
memory_body_ids:
  []
---

# Claude Splash 蟹鲸对决动画实现要点

## 范围与形态

- `claude-splash` 注册在 `shell.overlay`（order 1000，active），是整屏 splash；实现为单文件 `client.js`（本轮结束时 95,361 字节，sha `b4e04a2…`）。
- 组件 `Splash` 带 `leaving` / `gone` 状态，整段约 8.82s；主体是螃蟹与鲸鱼在竞技场对决，外围有星点、网格、冲击波、烟尘、碎片、灰烬等装饰层。
- 样式与动作数据都在同一文件内：`STYLE` 数组拼 CSS，`makeDuelPlan()` 生成对决关键帧，JSX 通过 `el(...)` 构建。

## 每挂载随机编排

- 不要在源码里写死一条时间线；`makeDuelPlan()` 在挂载时现算事件序列。事件类型三种：`ram`（冲撞）、`swap`（换位）、`blink`（瞬移），每事件随机起始边、时长与间隔。
- 换位在事件中点翻边；瞬移在中点消失、两侧各放一根光柱后翻边；冲撞产生命中点，供 `.csp-hit`、裂纹和闪白使用。
- 生成器带重掷：保证至少 1 次换位、1 次瞬移、4 次以上撞击，最多重掷 18 轮。实测 10 次挂载为 4–6 次撞击、1–3 次换位、1–2 次瞬移，7/10 形状不同。
- 左右两侧分别生成 `@keyframes csp-duel-l` / `csp-duel-r`；关键帧按事件序列展开，百分比按最大时间归一化，最后一帧落在 100%。

## 朝向与换位方向

- 鲸鱼原图本身朝左；换到左半场若保持原样会背对螃蟹。修复：按当前 x 计算 `face = x < 0 ? -1 : 1`，换边时施加 `scaleX(-1)`，且翻面发生在穿过中心那一帧，形成“冲过去—半路扭身—继续扑”的观感。
- 螃蟹是对称正面造型，不做整体翻转；改为向对手倾身（`-sign(x) * |rot| * 0.55`）。
- 两人交错时垂直错开约 ±7.5vmin（螃蟹在上、鲸鱼在下），避免穿模；“交换位置”是否自然，关键就在翻面时机与这个垂直错位。

## 换位、瞬移与末日层

- 换位：中间补两帧穿过中心（x 从 ±11 经 ∓3.5 到 ∓25），同步闪横向白光 `.csp-swapfx`。
- 瞬移：消失前/重现后各一根光柱 `.csp-tele`，用 `--tx` 定位，配两帧 opacity 0。
- 末日层曾只有 CSS 和数据未挂 DOM；本轮补挂：暗红 multiply 罩层、底部火光、7 团烟、18 块坠落碎片、22 粒灰烬、按每次撞击逐条长出的屏幕裂纹（SVG `pathLength=1` + dashoffset）、2.46 拍中心裂口、撕裂光带、每下撞击的全屏白闪。
- 离场靠 `.is-leaving` 统一触发 `csp-vanish`，并停掉各层动画。

## 回归教训（重要）

- 在一次“切代码块”的编辑中，`bokeh / rain / eq / spokes / bolts / comets` 等构建数组被误删，但 JSX 仍在引用，组件渲染时直接 `ReferenceError`，页面完全看不到变化。`node --check` 只能查语法，查不出这类运行时未定义引用，必须用渲染 smoke test 覆盖。
- 做渲染测试时，React stub 要补齐 `useState / useCallback / useRef / useEffect` 等，否则会先报 stub 缺方法，掩盖真正的组件错误。
- 这个单文件组件改动后至少检查：语法、括号平衡、`undefined/NaN` 字符串、`@keyframes` 声明数与引用数一致、class 全覆盖、节拍数不超过上限、总时长不变、软链两端 sha 一致；再跑一次“挂载 N 次统计事件形状”的随机性检查。
- 本轮验证结果：`@keyframes` 87/87，渲染 className 节点 116，节拍 105（上限 3.96），时间线 8.82s，构建产物 sha `b4e04a2…`，`shell.overlay` occupant `claude-splash` order 1000 active。
- 本轮还删除了代码标点层（14 个 `{}`、`<>`、`/;=*` 上浮字符）及其 CSS、构建器、keyframes，避免与“随机、干净”的视觉目标冲突。

## 后续调参入口

- 想提高换位频率：调 `mood` 权重与事件类型比例；想让瞬移更“炸”：在 `blink` 前后加残影或拉伸帧。
- 想整体加快/放慢：调 `pace`（约 0.82–1.24）与总时长；保留重掷的最小事件数，别让随机退化。
