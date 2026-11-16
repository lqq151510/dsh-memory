---
id: "efd777e4-d249-4d06-a85b-c58f4d7b51dc"
title: "正太音 RVC/MLX 引擎保真度实测：电音非移植所致，9.5 半音拉音与共振峰旋钮失效"
description: "同一模型同一输入的 MLX vs Applio 逐频段对照（±2 dB 内）、共振峰 formantScale 失效证据（F1/F2 实测）、去刺EQ/索引/f0平滑三组消融、A1/P1b/P2b 拉音幅度对照与「语速太快」踩坑、谱质心评分卡作废的教训。"
status: "active"
created_at: "2026-11-16T00:35:21.707Z"
updated_at: "2026-11-16T00:35:21.707Z"
content_hash: "0b36e79956fa8a204490ac6ff81db5d26d0716497ed455d79e88b57176987742"
source_paths:
  - "zhengtai_rvc.py"
  - "demo/listen/A1-220-f100.wav"
  - "demo/listen/P1b-preshift-220-keep.wav"
  - "demo/listen/E2-noindex.wav"
session_ids:
  - "mnemon-run-v1-ebc7a382-fa66-4df6-b1ff-8950f06c1dd6"
memory_body_ids:
  []
---

# 正太音：RVC/MLX 引擎保真度实测 —— 电音不是移植的锅，是 9.5 半音拉音

项目：`/Users/liuyongze/Documents/正太音`（macOS 桌面 App「正太音模拟器」，MLX 版 RVC 推理 + 实时流式转换）。
关联文档：`8ce69397-7978-4b9b-a99d-15e8ae7c2da1`（自训 shotaboys 模型、1.5 kHz +6 dB 搁架、A/B/C/D 打分）、`2c621ccf-ebba-47c4-9913-1c9f3f8b4e7c`（measure_voice 打分口径与谱质心陷阱）、`0836109c-fd49-460d-a2d1-91559498d5c5`（TTS+Applio 流水线与 NO_PROXY 踩坑）。

## 1. 结论（先给）

1. **「机械感/电音味」不是 MLX 移植造成的。** 同一输入、同一 `shotavoicedao.pth`，MLX 引擎 vs 官方 PyTorch 实现（Applio）逐频段长期平均谱差值全部在 **±2 dB 以内**，客观指标基本重合。换回官方实现会听到同样的电。
2. **根因候选是拉音幅度**：输入 127 Hz 成年男声 → 目标 220 Hz = **+9.5 半音**，远超 RVC 约 **±5 半音** 的干净工作区；超出后模型只能靠 f0 条件「重新合成」，合成痕迹就是细碎的金属感。属于物理性越界，不是参数没调好。
3. **共振峰旋钮基本是安慰剂**（见 §3），所以「调参数」这条路本身无效。

## 2. 移植保真度证据（同一句、同一模型）

| 指标 | MLX 引擎（A1） | 官方 Applio 参考 | 判读 |
|---|---|---|---|
| f0 中位数 | 215 Hz | 226 Hz | 同一音区（参考链路多一级处理） |
| jitter（相邻帧音高抖动） | 0.20 半音 | 0.20 半音 | 完全相同 |
| 频谱通量 flux_med | 4.96 | 4.70 | 同量级 |
| flux_p99 | 14.64 | 14.87 | 参考略高 |
| 包络调制峰 buzz_prom | 62.8 | 57.7 | 同量级 |
| 长期平均谱逐频段差 | — | — | 全带 **±2 dB 内** |

逐频段（A1 为 0 参考，Applio 差值，dB）：0–125 **+1.97**，125–250 +0.07，250–500 +0.14，500–750 −0.49，750–1000 +0.48，1000–1500 +0.29，1500–2000 +0.56，2000–3000 −0.46，3000–4000 −0.97，4000–6000 +0.27，6000–8000 **+1.71**，8000–12000 +1.38。

## 3. 共振峰通道实测失效（关键机制）

默认面板 `formantScale=1.18` 时用 LPC 量共振峰：**F1 531 → 562 Hz、F2 1282 → 1193 Hz**，几乎不向上移（F2 反而降了）。
原因：引擎里 `shift_formants` 是倒谱包络沿频轴插值，又被 **250–5300 Hz 的 taper** 和 **±3 dB 上限**夹住，所以幅度被吃掉。
⇒ 「共振峰微调」当前是安慰剂旋钮；真正能拧的只有 EQ，而 EQ 只会让声音变亮变薄 —— 这正是「怎么调都做不好」的结构性原因。

## 4. 已做过的对照实验（均已量化）

- **E1-nodeharsh**（关掉去刺 EQ，即 `converted -= 0.23 * voiced * highpass(4200 Hz)`）：flux 4.91、frozen 0.213、buzz_prom 62.4 —— 与 A1（4.96 / 0.214 / 62.8）几乎无差。去刺 EQ 不是电音来源。
- **E2-noindex**（关掉检索索引）：flux 4.96、frozen 0.208、buzz_prom 59.2 —— 同样无实质变化。index-rate 不是电音来源。
- 两个开关以环境变量形式保留、默认未改：`RVC_DEHARSH`（当前 0.23）、`RVC_STABILIZE`（当前开）。
- **S1-nostab**（关 f0 平滑 `stabilize_f0`）：frozen 0.224、jump1 0.029 —— 与默认差异很小。

## 5. 拉音幅度对照（同一句、同时长、同音高对齐）

底音输入 127 Hz，输出时长统一 11.8 s。

| 文件 | 链路 | RVC 实际拉音 |
|---|---|---|
| A1-220-f100 | 127 → 220（当时默认） | +9.5 半音 |
| P1b-preshift-220-keep | 前置升到 180 → 220 | +3.5 半音 |
| P2b-preshift-200-keep | 前置升到 180 → 200 | +1.9 半音 |

**踩坑记录**：首版用重采样做前置升调，同时把时长压短 1.42 倍（11.8 s → 8.3 s），被误判成「语速太快」。前置升调必须保时长；产品里计划改用 App 已有的 WORLD 路径（相位声码器只是方向验证，带涂抹感）。
实时模式的代价：前置升调要多约 **30 ms** 延迟（先升调再送 RVC），需在界面标注。

## 6. 度量口径教训（避免重犯）

此前自建的「谱质心 1000–1600 Hz + 3 kHz 能量」评分卡把「亮」当「幼」：A 版本仅 11 分，B/C/D 均 100 分，但用户**一个都不要**，只认 A。该评分卡已作废，**不应再用于选型决策**；用户偏好无法由谱质心代理。（`measure_voice` 的数值仍可用于监控，但不能当目标函数。）

## 7. 待定（下一轮的唯一分叉）

用户听 A1 / P1b / P2b 后给「哪个最不电」：
- P1b/P2b 明显更干净 → 引擎改成「前置升调 + 小步长拉音」标准链路（实时 +~30 ms 延迟）。
- 三个一样电 → 不是拉音幅度，而是 kazuki 模型本身的合成音色，转向换模型（goku/boruto 可一键切，代码已支持）。

## 8. 实现旁注（读代码时确认的事实）

- 引擎常量：`INPUT_RATE=16000`，`MODEL_RATE=40000`，hop 10 ms；`pitch_bins` 用 mel 刻度 1–255 共 254 档；`F0_MIN, F0_MAX = 50, 1100`。
- `_retrieve` 用 index 近邻加权混合，`mix = index_rate * where(f0>0, 1, 1-consonant_protection)`，即清音帧受 `consonant_protection` 保护少混。
- 推理链：`features.extract` → `stabilize_f0` → baseline 自适应（0.95 旧 + 0.05 新中位数，比值夹在 0.5–4）→ `_retrieve` → `enc_p`（content prior + log_scale + mask）→ `flow` 反向 → NSF HiFi-GAN `dec`。
- 输出保护：非有限值或 |x|>0.995 占比 >20% 抛 `RVC returned invalid audio`；最终 `clip(-0.95, 0.95)`。
- 旧脚本 `zhengtai_rvc.py` 走 Applio CLI（`core.py infer`），需剔除 `NO_PROXY`/`no_proxy`（httpx 解析 `[::1]` 报 Invalid port），并设 `PYTORCH_ENABLE_MPS_FALLBACK=1`、`PYTORCH_MPS_HIGH_WATERMARK_RATIO=0.0`。
