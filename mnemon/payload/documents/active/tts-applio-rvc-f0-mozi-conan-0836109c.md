---
id: "0836109c-fd49-460d-a2d1-91559498d5c5"
title: "正太音本地流水线实测：TTS+Applio/RVC 架构、F0 标定表与 mozi/conan 模型对比"
description: "本机实测跑通的正太音生成流水线：edge-tts 成年男声底音 + RVC 音色转换 + -16 LUFS 后期，含 F0 标定表、谱质心选型依据、Applio CLI 与 NO_PROXY 踩坑，及 mozi-v2 与 conan 模型结论。"
status: "active"
created_at: "2026-11-02T06:12:34.431Z"
updated_at: "2026-11-02T06:12:34.431Z"
content_hash: "5356e2ec4ba0c2a4310037095de5ff590384791cd7c93d032ea0dc8e7c743dc2"
source_paths:
  - "Documents/python/voice/zhengtai_rvc.py"
  - "Documents/python/voice/README.md"
  - "Documents/python/voice/samples"
session_ids:
  - "c8f74745-c031-4dc5-bf22-f4f9744cb11f"
memory_body_ids:
  []
---

# 正太音本地生成流水线（TTS + RVC/Applio）实测落地

项目目录：`/Users/liuyongze/Documents/python/voice`（脚本 `zhengtai_rvc.py`、说明 `README.md`、成品在 `samples/`）。
关联文档：6a771e59-e1f5-4911-8d5b-87f7b50583f9《正太音（未变声少年男声）生成指南：TTS 提示词与 RVC/GPT-SoVITS 流程》——本篇只记录本机实测的流水线实现与调参结论，不重复其提示词/渠道部分。

## 架构

```
文本 --edge-tts--> 成年男声干声 --RVC(Applio CLI)--> 少年音 --后期(提亮+响度标准化)--> samples/*.wav
        (zh-CN-YunxiNeural 180Hz)        (mozi-v2 / conan)
```

模块划分：TTS 只负责吐字，RVC 只负责音色，后期只负责「亮」与响度。三者互不越界，换模型不动 TTS，换文本不动后期。

- 一键：`Applio/.venv/bin/python zhengtai_rvc.py "文本"`；自有录音：`--input 录音.wav --pitch N -o out.wav`。
- 底音坚持用成年男声（云希 180Hz）而非现成少年音（云夏 296Hz）：云夏已贴近上限，再升调直接飞出童声区。

## 关键结论：童声感来自声道（谱质心），不是音高

用户三次反馈把根因逼出来了（"成品2就是声音太高了" → "太成年人了" → "像男娘"）。

| 底音/模型 | pitch | F0 中位数 | 谱质心 | 判断 |
|---|---|---|---|---|
| 云夏 296Hz | +8 | 457 Hz | — | ❌ 已进幼儿/女声区，不是正太 |
| mozi-v2 | 6 | 242 Hz | 938 | 模型本身是 15 岁变声末期，往下像成年、往上像男娘 |
| conan | 4 | 225 Hz | 976（3kHz+ 能量 8.5%） | ✅ 童声自然，新默认 |

同 F0（约 240Hz）下 conan 谱质心显著高于 mozi —— 这才是"童声"的客观差别。**音高是必要不充分条件**，只用降/升调解决不了"太成年 / 像男娘"。

mozi-v2：作者 15 岁变声前训练，GPL-3.0，可商用可二创，pth+index 齐；候选下载渠道里国内公开渠道无免费中文正太音，B 站唯一真实链接指向 HF 英文男童（julesbrown，UP 自承换中文会"扁"），6 个对口中文正太音模型在 klrvc.com（50–80 元、需登录）。
conan（`rvc_models/conan/ConanEdogawa_e250_s4250.pth` + `added_IVF419_Flat_nprobe_1_ConanEdogawa_v2.index`，48k / v2 / f0=1）为童声底，音色更合适。

## 实测参数表（云希 180Hz 底音，F0 中位数）

- pitch 4 → 219 Hz（偏低，接近少年后期）
- pitch 6 → 239 Hz（原默认，仍被评价"太成年"）
- pitch 8 → 262 Hz（更亮、略嫩）
- pitch 10 → 281 Hz（偏嫩）
- pitch 12 → 291–327 Hz（听感偏高、容易假）
- 女声约 180–250Hz；"未变声男童 250–330Hz"是生理区间，**说话不需要贴着上沿**——偏高不等于更像童声。

脚本默认：`--model` 默认 `conan`，且 `pitch = 4 if model == "conan" else 6`（`zhengtai_rvc.py` 内 MODELS 字典 + argparse 分支）。

## 后期与响度

- 后期链含 deesser 等处理，目标是"更亮、不闷"，正太音的关键不是更尖。
- 响度标准化到 **-16 LUFS**（B 站标准）。未标准化时实测仅 -21.9 LUFS，偏低约 6dB。

## 两个踩过的坑（已写进 README）

1. **Applio 首次启动会下载 1.69GB 底模，但 RVC 推理只需要 contentvec + rmvpe** —— 不要等 Web UI，直接走 CLI 推理；探测用的 7899 端口 Web 服务用完即停。
2. **`NO_PROXY` 中的 `[::1]` 会让 httpx 抛 `Invalid port: ':1]'`** —— 脚本里主动清除该环境变量规避。

另：Applio 的 `--formant-shifting` / `--formant-qfrency` / `--formant-timbre` 实测**无收益**（谱质心 905 → 861，不升反降），已判定不必投入。

## 后续可选方向

- 继续下探 pitch -2 / -4 会进入变声进行时音区（F0 < 186Hz），可能又偏成年，平衡点靠耳朵定。
- 愿意付费：klrvc.com 中文正太音（id=453、127 性价比最好）。
- 自训：mozi 开源训练集，5–10 分钟干净干声即可；**不要混入成年男声**，否则会把音高范围拉低。
