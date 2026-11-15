---
id: "a8ec51cb-79f4-4373-ada6-c9afeec3170c"
title: "Claude Splash 视觉与音效打磨：去白闪、暗夜幕、合成音效与双色激光"
description: "claude-splash 对决动画的观感/听感打磨记录：全屏白光根因与边缘发光替代、is-doom 夜幕混合与文字可读性、Web Audio 分层合成信号链与自动播放解锁、PASS/FAIL 代码芯片、按阵营配色的远程激光，以及 crispEdges 像素闪烁的合成层修复与校验手段。"
status: "active"
created_at: "2026-10-30T11:39:53.650Z"
updated_at: "2026-10-30T11:39:53.650Z"
content_hash: "854aef7a52e2b788773ff7a2bc122eded9879b4af84a76eff3de0701496b0a31"
source_paths:
  - ".dsh/profiles/desktop/node_modules/@local/claude-splash/client.js"
session_ids:
  - "2812b1b6-6915-4ef8-b5e2-93058618fa4a"
memory_body_ids:
  []
---

# Claude Splash 视觉与音效打磨：去白闪、暗夜幕、合成音效与双色激光

补充文档，聚焦「观感/听感」这一层；对决编排、鲸鱼镜像朝向、换位/瞬移特效与末日层挂载见既有文档
`b05ca3a1-8032-4e48-b292-8ac82c7cc546`（Claude Splash 蟹鲸对决动画实现要点）。

## 范围与产物

- 唯一实现文件：`client.js`（单文件 React 组件，注册在 `shell.overlay`，order 1000）。
- 实际生效路径是被 `@local/claude-splash` 软链的桌面 profile 副本：
  `/Users/liuyongze/.dsh/profiles/desktop/node_modules/@local/claude-splash/client.js`。
- 本阶段字节数演进（可作为改动量参考）：104,884 → 110,328（换音轨）→ 111,179 → 116,290（代码芯片）→ 120,369 → 121,405（激光）→ 121,516；sha 前缀 `1a966d5…` → `4c2eb8be0409`。
- 校验基线：时间线 8.82s（进入离开前）、`@keyframes` 声明数 = 引用数（88 → 90 → 92）、渲染 className 节点从 117 增长到 125、CSS 52,522 → 57,225 bytes。

## 1. 全屏白闪：根因与替代方案

- 根因：`.csp-hit` 用「全屏 `background:#fff` + `mix-blend-mode:screen`」做命中反馈，screen 混合下整屏被提亮成白色闪光弹。
- 修复：改成**只亮边缘的冲击光**——`radial-gradient` 中心 34% 全透明 → 外圈 76% 半透明 → 边缘 100% 最亮，中心保持透明，文字与精灵始终可读。
- 同时清掉其他纯白光源：瞬移光柱 `#FFF6E4` → 琥珀 `#FFC47E`；换位横光 `#FFF0D4` → `#FFB066`；镜头光带中心与彗星头 `#fff` → 暖金；字标高光改 `rgba(255,226,192,.8)`。
- 峰值透明度从 `.9` 降到 `.66`，最后一击仍 170ms；代价是「打击感」要靠边缘扩散而不是整屏爆白。

## 2. is-doom 下白字不可见

- 根因：`.is-doom` 把文字换成暖白 `#fff6ea`，但夜幕层用 `multiply` 且中心仅 34% 红透明度，屏幕中部实际仍是奶油色，浅底白字对比度不足。
- 修复：夜幕从 `multiply` 浅红改为**普通混合深色罩**（中心 `rgba(26,9,5,.74)` → 边缘 `rgba(5,3,2,.97)`，再叠 `rgba(255,116,44,.24)` 暖光），从「粉色褪色」变成暗红熔炉。
- 文字补 `text-shadow:0 1px 3px rgba(0,0,0,.72)` 作描边，保证背景未完全变暗的过渡帧也可读；文字切暖白时机从 1.92 拍推迟到 2.02 拍，先暗后换色。

## 3. 音效：从干声到分层合成信号链

纯 Web Audio 现场合成（无音频素材文件），关键分层：

| 层 | 实现 |
|---|---|
| 空间 | 卷积混响：现生成「噪声 × 指数衰减」1.9s impulse response，接 `ConvolverNode` |
| 总线 | `DynamicsCompressorNode` 压总线，避免多碰撞叠加爆音 |
| 撞击 | 三层：4kHz 瞬态 click + 低通噪声本体 + 34Hz 次低频 |
| 掠风 | 棕噪声（白噪声积分，比白噪声暖）→ 带通扫频 → `StereoPanner` 左右横移 |
| 瞬移 | 方波 + 96Hz FM 抖动，补高频火花噪声，上升/下落两种 zap |
| 铃音 | 4 分音（1 / 1.5 / 2.02 / 3.01）× 各 4 音分 detune，长尾混响 |
| 轰鸣 | 棕噪声低通 150→88Hz，缓起缓落 |
| 火 | 16 次随机位置/音高的噼啪声，铺在 2.2~3.8 拍 |

- 连击时撞击音高递减；换位声像随开火/移动方向走，耳机可听出交叉。
- 自动播放限制：监听 `pointerdown` / `keydown` 首次手势解锁 AudioContext，并**跳过已过期音效**，避免解锁瞬间一次性播放积压声音。

## 4. 程序员浪漫：PASS / FAIL 芯片与代码片段

- 每次碰撞生成 token 节点：`csp-token` 带 `--pass` / `--fail` / 代码态三种修饰类，另配 `csp-snip` 代码片段与 `csp-tokens` 容器（`aria-hidden`）。
- 设计意图：让「碰撞」语义化为编译/测试结果（通过=绿、报错=红、代码块=中性），比纯火花更贴开发者语境。
- 抽样渲染可验证数量：6 次碰撞样本下 4~5 个 token（2 pass + 2 fail 常态）、3 个 snip。

## 5. 远程激光（按阵营配色）

- 每个命中前双方各射一束：2px 细线 + `box-shadow` 外发光，`scaleX(0→1)` 约 46ms 甩出、340ms 消散；起点 10px 亮球 + 18px 外发光；命中端 12vmin 白色核心光斑炸开 1.5 倍。
- 颜色绑角色本体色：螃蟹 `#d97757`（橙）、鲸鱼 `#4d6bfe`（蓝）；**从各自当前所在半场发射**，换位后枪口位置随之交换，不是固定「左橙右蓝」。声像跟随开火方向。
- 碰撞命中光 `.csp-hit` 同样按奇偶在橙/蓝间交替（`.csp-hit--alt` 切到 `--csp-whale`）。
- 碎屑从 18 块增到 26 块以强化冲击。

## 6. 像素精灵「一直闪」：一个设计 + 一个真 bug

- 设计部分：瞬移时两帧 `opacity:0` 造成消失/出现；在连续变形与震屏背景下被眼睛读成「闪一下」。
- 真 bug：像素画用 `shape-rendering: crispEdges`，而螃蟹/鲸鱼持续做 transform 动画（震屏、冲锋、挤压、呼吸），浏览器逐帧重对齐像素栅格导致边缘像素跳帧，表现为身体抖动/闪烁；鲸鱼因像素块更大而不明显。
- 修复：瞬移两帧 `opacity:0` → `0.14` 残影（读作「瞬移虚影」）；每个精灵提升为合成层
  `.csp-figure svg{transform:translateZ(0);backface-visibility:hidden}`，一次栅格化后只做图层位移。

## 7. 回归与验证手段（可复用）

- 静态：`node --check client.js`；花括号/圆括号平衡；无 `undefined`/`NaN` 字面量。
- 结构校验脚本输出：beats 使用数、`@keyframes` 声明数 vs 引用数、渲染 className 节点数、CSS 字节数、时间线秒数。
- 桩渲染（低成本无浏览器 e2e 替代）：伪造 `window.__ModuleLoader__` + 极简 React（`createElement` 返回 `{t,p,c}` 树、hook 直通），`new Function` 加载源码拿 `mod.factory`，用假 `ctx`（effect/locale/slots）取回组件再递归遍历，统计 `csp-token`/`csp-snip`/`csp-hit`/`csp-clash`/`csp-tele`/`csp-swapfx`/laser 节点数，并检查随机编排两次渲染是否不同（`random per mount`）。
- 生效性：比对源文件与软链目标的 sha，确认改的是正在被加载的那份 `client.js`。
- 注意：脚本里用字符串锚点（如 `"    #ffd9a8 50%"`）做断言时，排版/缩进变动会 `count=0` 直接 AssertionError；锚点应选稳定的短片段。
