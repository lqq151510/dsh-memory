---
id: "7105e37d-b713-4447-8118-c8a460ce29bc"
title: "RVC 变声提亮配方（共振峰 1.30 + 1.5kHz/+6dB）与 ChildTalk 22GB 下载实测"
description: "kazuki 预置模型免重训过 85 分闸门的参数扫描（含 EQ 二维扫描表）、demo A/B 主观判据分叉、ChildTalk 22.02GB 的 aria2c 下载实测与「文件大小≠已下载字节」测量陷阱、以及下载后的训练流水线。承接 2c621ccf-ebba-47c4-9913-1c9f3f8b4e7c。"
status: "active"
created_at: "2026-11-15T12:18:12.058Z"
updated_at: "2026-11-15T12:18:12.058Z"
content_hash: "734635e82b35d1b4a1e7bdbad6f85b7d45e3c155a5dcf72ef07e23a5e8df5701"
source_paths:
  - "demo/A-现在这版.wav"
  - "demo/B-提亮版.wav"
  - "script/train_rvc.py"
  - "script/measure_voice.py"
session_ids:
  - "mnemon-run-v1-9a93fe8f-a603-42d9-bc78-924f47b007ee"
memory_body_ids:
  []
---

# RVC 变声「提亮」配方与 ChildTalk 下载实测（2026-11-15）

承接文档 `2c621ccf-ebba-47c4-9913-1c9f3f8b4e7c`（RVCMAX 六个模型全部 <85 分 → 结论是必须自训）。**本次实测修正了那条结论**：现成的 kazuki 预置模型在不重训的前提下可以过 85 分闸门，瓶颈是高频被压掉，不是模型本身。

## 1. 参数扫描（同一句成年男声输入，measure_voice 打分）

目标带：谱质心 1000–1600 Hz，验收线 85 分。

| 配置 | 谱质心 Hz | 分 |
|---|---|---|
| app 现默认（220 Hz / 共振峰 ×1.0） | 466 | 26 ✗ |
| 250 Hz / ×1.15 | 631 | 47 ✗ |
| 250 Hz / ×1.30 | 701 | 47 ✗ |
| 275 Hz / ×1.30 | 788 | 47 ✗ |
| **250 Hz / ×1.30 + 1.5 kHz 以上提 6 dB** | **1188** | **85 ✓** |

EQ 二维扫描（shelf 频率 × 增益）：

| shelf | +6 dB | +9 dB | +12 dB |
|---|---|---|---|
| 1.5 kHz | 1187.7 / **85** | 1763.8 / 65 | 2484.8 / 65 |
| 2.0 kHz | 1139.4 / 79 | 1712.9 / 65 | 2605.3 / 65 |
| 2.5 kHz | 1103.2 / 73 | 1637.9 / 59 | 2533.8 / 59 |

结论：**只有 1.5 kHz / +6 dB 这一格能过**。增益或频率再往上推，谱质心冲到 1637–2605 Hz，落到「刺」的区间，分数反而掉到 59–79。单独抬基频（242.8 → 267.8 Hz）对分数没有帮助（始终 47），说明幼态来自高频能量分布，不是 F0 时钟。

## 2. 待用户主观确认（决策分叉）

- `demo/A-现在这版.wav`：app 当前输出（11.8 s, 24000 Hz）
- `demo/B-提亮版.wav`：共振峰 ×1.30 + 1.5 kHz/+6 dB（11.8 s, 24000 Hz，同一句输入、同长度）

用户听完给一句话判据：
- 「更清亮的少年」→ 把链路固化为默认（共振峰 1.30 + 后级提亮，离线导出与实时变声两条路都加），不等任何下载。
- 「尖、刺、像电话」→ EQ 补不出幼态，回归自训路线（约 2 天）。

## 3. ChildTalk 语料与下载实测

- 来源：南开 NKH-HLT ChildTalk，2–8 岁真人童声，cc-by-nc-sa，是唯一免费、年龄段对得上的中文童声语料。
- 体积：HF 文件列表精确为每分片 1 GiB × 21 = **22.02 GB**。
- 加速器结论：curl 单连接被限速（4 路并行仅 111–118 KB/s，ETA ~50 h）；换 `aria2c`（6 文件并发 × 8 连接）。
- **测量陷阱（本次最大教训）**：aria2c 启动即把文件预分配到完整大小，所以 `ls`/`stat` 的文件大小 ≠ 已下载字节。曾据此误报「22 MB/s、22.02 GB 只需 10 分钟、110 秒下了 5.29 GB」，随后实测证伪。**可靠读法只有 aria2 RPC（`http://127.0.0.1:6800` 的 downloadSpeed / numActive / numWaiting）或 `.aria2` 控制文件残留判断。**
- 修正后真实吞吐：0.3–1.2 MB/s，ETA 5–9 小时（当天 19:39 时约 3.1 GB / 14%）。
- 环境侧坑：本机 `huggingface_hub` / `hf_transfer` 均未安装；多个探针返回 0 KB/s（mirror_proxy / mirror_direct / hf_proxy / hf_direct 全 0）；aria2 日志累计 114 条 errorCode=1；hf-mirror 直连绕代理比走代理快。备选镜像：ModelScope `Cloudbird/ai-toy-data-childtalk`（Apache-2.0 标注，下载量很小，需自备鉴权）。
- 持久化：`nohup` 起独立进程，关掉 DSH 窗口仍在跑；`.aria2` 控制文件保证断点续传。

## 4. 下载完成后的既定流水线

1. 校验 21 个分片数量 → `cat` 合并 → 解压 → 打印目录结构
2. 读 `speaker_info.csv`，筛 `child_<年龄>_M` 中 3–9 岁男童（2 岁咿呀、9 岁临界变声，单独标注）
3. 切 3–10 s 片段，剔掉含成人说话/噪音的
4. `script/train_rvc.py`（MLX，M5 上跑，直接产出 app 可用的 .npz；要求 ≥3 条干净目标说话人 WAV，帧数 <48 的片段会被丢弃）
5. `script/measure_voice.py` 打分，与基线对齐：kazuki 默认 26 分、提亮版 85 分

## 5. 复用要点

- 变声「不够幼」先量谱质心再动模型：高频 shelf +6 dB @1.5 kHz 是最便宜的一档，越过 1.6 kHz 质心就变「刺」。
- 任何预分配型下载器（aria2c、部分 bit-torrent 客户端）都不要用文件大小报进度。
