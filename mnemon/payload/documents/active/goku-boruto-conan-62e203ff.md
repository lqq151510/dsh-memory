---
id: "62e203ff-43cf-45d5-ac8e-23c09189a833"
title: "正太音选型：谱质心优先于音调（goku/boruto/conan 模型对比与踩坑）"
description: "在 0836109c 流水线基础上的第二轮实测：幼态感由谱质心决定而非 pitch，五模型 F0/谱质心标定表、共振峰调整反效果、当前默认 goku p0、并行推理 SIGSEGV 等操作坑。"
status: "active"
created_at: "2026-11-02T06:29:07.872Z"
updated_at: "2026-11-02T06:29:07.872Z"
content_hash: "bc8d333065d968875aafc8c95181fe87a58f833d37083f70d304acc9645e4dcc"
source_paths:
  - "Documents/python/voice/zhengtai_rvc.py"
  - "Documents/python/voice/README.md"
  - "Documents/python/voice/samples"
session_ids:
  - "4508ee6b-c8cc-4b87-9820-6b4338d416b0"
memory_body_ids:
  []
---

# 正太音选型：音色模型比音调更关键（谱质心标定 + 模型对比）

项目目录：`/Users/liuyongze/Documents/python/voice`（脚本 `zhengtai_rvc.py`、说明 `README.md`、成品 `samples/`、模型 `rvc_models/`）。
关联文档：0836109c-fd49-460d-a2d1-91559498d5c5（流水线架构、F0 标定表、mozi-v2 与 conan 首轮结论）、6a771e59-e1f5-4911-8d5b-87f7b50583f9（正太音声学参数与提示词）。

## 结论先行

用户连续四轮听感反馈（“成品2就是声音太高了” → “太成年人了” → “像男娘” → “还是太成熟了幼态一点”）推翻了一个初始假设：**调高 pitch 就能更像孩子**。实际根因是模型音色（声道长度 → 高频含量）不够童，而非参数。

- **幼态感的主变量是谱质心与 3kHz+ 高频能量占比，不是 F0。** mozi（15 岁变声末期模型）与 conan 在相同 F0≈240Hz 时谱质心差 40 点以上，听感差异却是“成熟” vs “童声”。
- 只调 pitch 的两种失败模式：往下调（pitch 6 → F0 239Hz）显得成年；往上调（pitch 10–12 → F0 281–327Hz）音色跟不上，变成“男娘”。两种都指向换模型。
- **共振峰调整（formant shifting）在本流程中是反效果**：qfrency 升高后谱质心从 936 降到 706，3kHz+ 能量从 8.2% 降到 3.1%，越调越闷。不要在这条路上投入。

## 谱质心标定表（越高越幼/亮）

| 模型 (pitch) | F0 median | 谱质心 | 3kHz+ 能量% | 判断 |
|---|---|---|---|---|
| mozi-2 (p6) | 242 | 938 | 7.3 | 偏成熟，已弃用 |
| conan 柯南 (p4) | 225 | 976 | 8.5 | 中性，首轮默认 |
| miitomo 男孩 (p2/p6) | 211 / 250 | 564 / 532 | 2.7 / 1.5 | 闷，不可用 |
| **goku 小悟空 (p0)** | **184** | **1329** | **12.4** | **当前默认：幼且音高自然** |
| boruto (p0) | 208 | 1381 | 11.0 | 更幼更软，可能偏女孩 |
| 云夏底 + goku (p0) | 302 | 1418 | 10.8 | 最幼但音高最高，易触发“太高了” |

## 音高标定（底音=云希成年男声 180Hz）

pitch 0/2/4/6/8/10/12 → F0 约 186/203/219/239/262/281/327 Hz（mozi/conan 上实测，不同模型略有偏移）。
初始把目标定为“未变声男童 250–330Hz”，后被认定下沿其实是唱高音的区间：**说话用不到那么高，偏高不等于更像童声**，超过一定程度会飘成女声/幼儿。默认值因此从 10 降到 6，最终改为「模型 goku + pitch 0」。

## 当前默认（脚本已改）

```bash
cd ~/Documents/python/voice
Applio/.venv/bin/python zhengtai_rvc.py "要念的文本"                # goku, pitch 0 → F0 184Hz
Applio/.venv/bin/python zhengtai_rvc.py "文本" --model boruto       # 谱质心 1381，更软
Applio/.venv/bin/python zhengtai_rvc.py "文本" --pitch 4 -o 低.wav  # 229Hz，少年后期
```

`--model` 支持 conan / goku / boruto / miitomo / mozi / julesbrown；脚本按模型给默认 pitch（conan→4，其余→6 的旧逻辑已改为按模型选幼态档）。模型来自 HF / voice-models 等公开直链，解压后放 `rvc_models/<name>/`（.pth + added_*.index 成对）。

## 操作坑

- **多个 RVC 推理并行会 SIGSEGV**（MPS 后端，退出码 11，Command '... core.py infer ...' died with Signals.SIGSEGV）。串行跑，一次一个模型/档位。
- 首次启动 Applio 会下载 1.69GB 底模，推理不需要；直接走 CLI，不要等 Web UI。
- `NO_PROXY` 中的 `[::1]` 会让 httpx 抛 `Invalid port: ':1]'`，脚本会自动清除。
- 评估脚本用 F0 median/p10/p90 + 谱质心，比单听样本更快定位“偏高 vs 偏成熟”。

## 可继续的方向

- 用户偏好落点尚未最终确认：goku p0（184Hz/1329）与 boruto p0（208Hz/1381）之间的取舍需要耳朵定；若仍不够幼，下一步是云夏底 goku（1418，但 F0 302Hz）。
- 付费渠道仍有 klrvc.com 的中文正太音模型（id=453、127 性价比最好，50–80 元需登录）。
- 自训：mozi 开源训练集，5–10 分钟干净干声，**不要混入成年男声**，否则音高范围被拉低。
