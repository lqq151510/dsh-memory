---
id: "795dc10f-579a-4525-b6ac-0e46176c190b"
title: "zebao-skin v1.2：真实时间昼夜循环 + 海洋层（实现与验证）"
description: "zebao-skin 的昼夜循环/海洋/天体层实现：真实日出日落与月相引擎（坐标已按用户所在地切到广州）、昼夜自适应配色、设置项与 data 属性、harness 预览参数、-rgb 变量教训与验证门禁。"
status: "active"
created_at: "2026-11-07T09:48:57.229Z"
updated_at: "2026-11-07T09:48:57.229Z"
content_hash: "501a7823e23ebf97940c6a5b1c00ebec0a718ea9b01529bd1c3451bf8f0d3d98"
source_paths:
  - "plugins/zebao-skin/lib/client.src.js"
  - "plugins/zebao-skin/lib/skin.css"
  - "plugins/zebao-skin/tools/validate.mjs"
  - "plugins/zebao-skin/preview/harness.html"
  - "plugins/zebao-skin/README.md"
session_ids:
  - "9fe9acf3-2bd7-4bb8-8b68-25cee50608e6"
memory_body_ids:
  []
---

# zebao-skin v1.2：星空 × 海洋 × 真实时间昼夜循环

基础结构、desktop profile 安装位置、⌘Q 重启才生效的 bundle 特性见既有文档 53636080-86e7-4b3b-9c57-5aa2af2c5a42。本文记录 v1.2 增量（用户需求：模仿开屏动画，把星空、海洋、DeepSeek 结合，加动态效果，日月交替按真实情况）。

## 昼夜时间引擎（lib/client.src.js）

- 太阳位置按当天真实日出→日落弧线计算，含经度差与均时差修正；每 30s 跟随时钟刷新。
- 坐标按用户实际所在地：**当前为广州 23.13°N / 113.26°E**（2026-11-07 由郑州 34.75°N / 113.62°E 切换；用户说"我现在在广州"，可能是短期停留，回郑州需切回，或可做成"城市跟随"设置项）。验证：HUD 显示 ☀ 06:41 · ☾ 17:40，与广州当日真实日出日落一致。
- 月相按真实朔望月（29.53 天）算照度：满月全亮、弦月半亮；**照度 <0.2 直接隐藏月亮**（现实新月不可见，不渲染灰饼）。
- HUD 右下角实时显示 ☀ 日出 · ☾ 日落 时刻。

## 视觉层（lib/skin.css + builders）

- 天体层：太阳/月亮沿真实弧线运动、银河带；夜晚月亮挂高位。
- 海洋层：海面、三层波浪、日/月光碎光带、海中两头 DeepSeek 鲸剪影游动、夜光浮游。
- 昼夜自适应：天空/星空/代码雨/流星/鲸星座配色随太阳高度连续过渡；夜晚面板自动加实（最多 +22%）保正文对比度；行星大弧夜里退成淡光、波浪白条夜间收敛、星星夜间增益提亮。
- 磨砂层 #zebao-skin-frost 的白色渐变在夜空下会显灰，夜间模式需降低其不透明度。

## 设置项与 data 属性

- 设置 → 背景皮肤新增：「昼夜循环」四档（真实时间/白天/黄昏/夜晚）+「海洋浓度」滑杆。
- 卸载清理属性：data-zs-fx / data-zs-ds / data-zs-ocean（validate 门禁逐一验证 set/remove）。

## 调试教训（可复用）

- 皮肤 CSS 中以 `-rgb` 结尾的变量（如 --zs-star-rgb）必须写**裸 r,g,b** 值；JS 若写入已带 `rgb(...)` 包装的字符串会双重包装，整条声明失效、元素全透明（曾导致星星整夜不可见）。用截图取 median bg / bright px 统计定位。

## 验证流程

- `node --check` + `tools/validate.mjs`：括号配平、CSS 类名/@keyframes 覆盖、三个旋钮（fx/ds/ocean）defaults/range/rows/cssvar 闭环、data 属性 set/remove。
- preview/harness.html 预览参数：`?sky=day|dusk|night|auto` 截各时刻、`?bare=1` 看裸皮肤（已独立生效）、`?stats=1` 输出节点/动画数、`?probe=1` 派发 pointer 事件并测层遮挡（期望 OVERLAP {}）。
- Edge headless 截图存 preview/shot-sky-*.png；改完用层遮挡探针确认新层不挡字。

## 改动文件

lib/client.src.js（时间引擎+海洋/天体 builders）、lib/skin.css（昼夜变量+海洋样式）、lib/client.js（构建产物）、tools/validate.mjs、preview/harness.html、README.md。
