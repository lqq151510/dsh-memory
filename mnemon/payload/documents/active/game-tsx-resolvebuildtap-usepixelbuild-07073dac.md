---
id: "07073dac-4a50-49fa-8342-afed2c0dd1a4"
title: "开心小镇：Game.tsx 搭建子系统抽取（resolveBuildTap / usePixelBuild）"
description: "kaixin-world 2026-11-06 重构交接：把自由搭建的点击守卫抽成纯函数 resolveBuildTap，再把整个搭建状态机搬进 usePixelBuild，Game.tsx 679→478 行；含 hook 接口形状、git apply --cached 单侧提交做法、buildTileArt.ts 死代码与并行线 play/store.test.ts 失败两个待定项。"
status: "active"
created_at: "2026-11-06T04:04:02.644Z"
updated_at: "2026-11-06T04:04:02.644Z"
content_hash: "66397d7fd6d5eacb8f6769cd9d66b89d6fddb51adfa653899f1ef9ad20295047"
source_paths:
  - "src/components/Game.tsx"
  - "src/hooks/usePixelBuild.ts"
  - "src/gameplay/pixelBuildTap.ts"
  - "src/gameplay/pixelBuildTap.test.ts"
  - "data/pixelBuilding.ts"
  - "src/gameplay/buildTileArt.ts"
  - "src/gameplay/play/store.test.ts"
session_ids:
  - "bdaa9bae-d6a8-486e-abc8-3ac0693460d9"
memory_body_ids:
  []
---

# kaixin-world：Game.tsx 搭建子系统抽取（resolveBuildTap + usePixelBuild）

时间：2026-11-06。用户以「后拆分」「继续优化」「可以」连续批准把 [Game.tsx](src/components/Game.tsx) 里最复杂的自由搭建逻辑逐层抽离。本记录是这次重构的实现交接，含两个仍未拍板的遗留项。

## 结论

- 搭建状态机已整块搬出组件：[src/hooks/usePixelBuild.ts](src/hooks/usePixelBuild.ts)（295 行），Game.tsx 679 → 478 行（−201），行为不变。
- 点击守卫先一步抽成纯函数：[src/gameplay/pixelBuildTap.ts](src/gameplay/pixelBuildTap.ts) 的 `resolveBuildTap`，只返回四种结果（忽略 / 记起点 / 拒绝并给原因 / 提交），判断顺序与原分支逐条对齐；[src/gameplay/pixelBuildTap.test.ts](src/gameplay/pixelBuildTap.test.ts) 用 7 个用例钉住每条分支，包含「暂停时也能先选起点」这类易改坏的细节。
- `PixelBuildMode` 从组件类型下沉到 [data/pixelBuilding.ts](data/pixelBuilding.ts)，gameplay 层不再反向依赖组件类型。

## 架构要点（为什么这样切）

1. **判断与提交分离**：`changeBuildCell` 原来在一个函数里做选起点、未继续世界、高价、预检失败、拆除空格等一串守卫再异步提交。抽成纯函数后守卫可单测、无 React/Convex 依赖，组件只负责异步提交。
2. **hook 承接副作用**：usePixelBuild 接走模式/材料/工具状态、选区预览、点击提交（铺设与拆除）、撤销、以及「开心自由发挥」的自动排程（低价时段判定、`document.visibilityState` 判断、1 秒节拍、90 秒节流、并发上限 3、低价恢复延迟任务 15 秒重试）。Esc / R 键盘处理也移入 hook，组件只传 `panelOpen` 与 `onEscapePanel`。
3. **hook 接口形状**：输入 `{ worldId, worldStatus, game, connected, isRunning, panelOpen, onEscapePanel }`；返回 `buildState / buildMode / buildKind / buildTool / blueprintId / rotation / buildAnchor / buildHover / buildPending / buildNotice / peak / costNow / creativeActive / buildPreview` 及一组 setter 与动作（`changeMode`、`changeTool`、`rotateBlueprint`、`cancelSelection`、`changeBuildCell`、`undoLastBuilding`、`toggleCreative`）。面板用到的 `costNow` 仍由 hook 提供，玩趣面板的 `now={costNow}` 继续可用。
4. **类型修正**：Convex `worldStatus` 可能返回 `null`，hook 接受 `PixelBuildWorldStatus | null | undefined`（即 `{ status: string; runUntil?: number }`），这是抽取后出现的唯一类型报错点。

## 验证

- `tsc --noEmit` 通过；`vite build` 通过。
- 测试：81/82 套件通过、758 用例全绿；唯一失败是另一条工作流的 `src/gameplay/play/store.test.ts`（见遗留 2）。

## 提交与工作区隔离

- `8c213ac` refactor(build): move the tap guard out of Game.tsx into a tested pure function
- `60dcaae` refactor(build): move the pixel-build state machine out of Game.tsx（2 files, +334 / −240）

同一 worktree 里另有并行会话在加「玩趣」子系统（`src/components/play/`、`src/gameplay/play/`、docs/PLAY-PACK-*.md），且已把 PlayPanel 相关 hunk 写进 Game.tsx。提交时用 `git apply --cached` 只暂存自己的 hunk（12 个 hunk 保留 7 个），排除 PlayPanel 的 4 个 hunk，保证提交的 Game.tsx 不依赖未跟踪文件、单独可编译；对方改动原样留在工作区。这是多会话共用工作区时可复用的提交做法。

## 遗留（需用户拍板，未处理）

1. **死模块**：[src/gameplay/buildTileArt.ts](src/gameplay/buildTileArt.ts) 有 634 行，全仓库 0 引用、0 测试，`PixelBuildOverlay` 仍用手写的 `drawBuilding`。疑似那次功能提交里没接上的渲染层，接进渲染会改变视觉、需截图验收；或先删除待用时再取回。
2. **另一条线的测试失败**：`src/gameplay/play/store.test.ts` 整套件起不来 —— `ReferenceError: exports is not defined`，指向 `store.ts` 在 ts-jest（ESM）下 import `react` 的 `useSyncExternalStore` 造成的模块格式问题；`store.ts` 自身无 `exports` 语句，包是 `"type": "module"`。属并行工作流文件，未经许可不主动修改。
3. 已观察到的并行改动：`Game.tsx` 的 PlayPanel 接入、`docs/PLAY-PACK-CONTRACT.md`、`.cursor/`、`.playwright-mcp/` 均未跟踪/未提交，后续大改 carry 前需确认两边不互相覆盖。

## 下一步候选

- 把 hook 内「开心自由发挥」排程拆成 `useKaixinCreative`（约 120 行），进一步缩小 usePixelBuild。
- 或先处理 buildTileArt.ts 死代码。

相关项目记录：[镜头 2.4 / 茶亭 / 到访判定浮点修复](documents/active/2-4-71d81705.md)（id 71d81705-24fc-44ab-bb18-47670a0d2a0a）。
