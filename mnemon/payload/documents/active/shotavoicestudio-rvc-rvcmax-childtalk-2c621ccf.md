---
id: "2c621ccf-ebba-47c4-9913-1c9f3f8b4e7c"
title: "正太音模拟器（ShotaVoiceStudio）自训 RVC：RVCMAX 模型实测、打分口径与 ChildTalk 语料落地"
description: "正太音桌面 App 的 MLX/RVC 工程文件、RVCMAX 六个模型的客观实测（全部 <85 分、谱质心 439–857Hz→必须自训）、measure_voice 验收口径、ChildTalk 语料选型与 21.8GB 下载/训练落地计划，含 HF SSRF 与代理下载踩坑。"
status: "active"
created_at: "2026-11-15T09:41:47.810Z"
updated_at: "2026-11-15T09:41:47.810Z"
content_hash: "1ba37135cc909d62f7c58452acd2b7a97a50298dae6e37d81c896ac730275cbc"
source_paths:
  - "script/rvc_engine.py"
  - "script/rvc_worker.py"
  - "script/train_rvc.py"
  - "script/measure_voice.py"
  - "script/import_rvcmax_model.py"
  - "script/prepare_elra_kids.py"
  - "script/build_and_run.sh"
  - "docs/男童声线语料参考.md"
session_ids:
  - "mnemon-run-v1-eb267ef4-f6e0-4623-bb00-edec4d963a70"
memory_body_ids:
  []
---

# 正太音模拟器（ShotaVoiceStudio）自训 RVC：模型实测结论 + ChildTalk 语料落地路径

项目目录：`/Users/liuyongze/Documents/正太音`。macOS 桌面变声 App「正太音模拟器 RVC 优化版」（bundle `dev.kaixin.shotavoicestudio.rvc.smooth`，Mach-O 进程名 `ShotaVoiceStudio`，产物 `dist/正太音模拟器-RVC优化版.app`），前端 Swift 外壳 + Python/MLX 推理引擎。

关联文档（同一用户的既有实测，方法论可互相印证，但项目目录不同）：
- `0836109c-fd49-460d-a2d1-91559498d5c5`《正太音本地流水线实测：TTS+Applio/RVC 架构、F0 标定表与 mozi/conan 模型对比》
- `6a771e59-e1f5-4911-8d5b-87f7b50583f9`《正太音（未变声少年男声）生成指南：TTS 提示词与 RVC/GPT-SoVITS 流程》

## 1. 关键工程文件（本机已跑通）

`/Users/liuyongze/Documents/正太音/script/` 下：

| 文件 | 作用 |
| --- | --- |
| `rvc_engine.py` | 推理核心：`INPUT_RATE`/`MODEL_RATE`/`MODEL_HOP`、`SpeechFeatures`、`load_weights`、`generate`/`synthesize_model`、`pitch_bins`，`VoiceConverter(assets, model, index, index_rate, formant_scale, ...)`，子命令 `file`（`source`/`destination` + `--pitch`/`--source-pitch`/`--detail-formant-scale`/`--source-dynamics`）与 `stream` |
| `rvc_worker.py` | 常驻 worker，给 Swift 端做流式/文件转换 |
| `train_rvc.py` | MLX 微调：从 40 kHz 底模训说话人向量，`slices()` 读 `*.wav` → soxr 重采样到 `MODEL_RATE` → 峰值归一（`min(0.74/max_abs, 3)`）→ 再降到 `INPUT_RATE` 取 HuBERT 特征 + F0 |
| `measure_voice.py` | 客观打分/验收（见第 3 节），输出 f0 / centroid / score / 逐项 ok-weight |
| `import_rvcmax_model.py` | 第三方 RVCMAX `.pth` → 本项目 `.npz`：`--source` `--output` `--runtime` `--hubert-source` `--hubert-output`（默认写 `.build/rvc/assets/hubert-v1.npz`） |
| `convert_rvc_assets.py` / `convert_faiss_index.py` | 资产与检索索引格式转换 |
| `prepare_elra_kids.py` | ELRA 付费儿童语料的预处理入口（备用路线） |
| `build_and_run.sh` | 一键构建 + 启动 App |

推理默认参数（`rvc_engine.py` 的 argparse 默认值，改动前先看这里）：`--index-rate 0.08`、`--consonant-protection 0.9`、`--formant-scale 1.12`（1.0 = 模型原始共振峰）、`--latent-noise-scale 0.0`、`--pitch 275`；`--detail-formant-scale` 合法区间 1.0–1.2，`--source-dynamics` 0–1。

`train_rvc.py` 里的既有踩坑注释：`extract` 已为 HuBERT 补了 40 点，若在切片阶段再补会让「内容、基频、波形」三者错位。

## 2. RVCMAX 模型批量导入（已验证）

同一批 `.pth` 用 `import_rvcmax_model.py` 转成 npz，输出到 `.build/rvc/model/rvcmax-import-*/rvcmax-<name>.npz` + 同名 `.json` 配置：

- 40 kHz / 256-dim 体系：`kikiV1`、`keruanV1`、`guanguanV1`（三者均转换成功，353 个 tensor 级别的一致性在 youzhanv2xi 上验证为 "Converted 353 tensors."）
- 48 kHz / 768-dim 体系：`youzhanv2xi`（来自 `youzhanv2-xi.pth`）——说明导入器同时支持两套维度，切换模型时必须带上匹配的 `hubert` 资产

## 3. 客观打分口径（`measure_voice.py`）

及格线 **85 分**。逐项判据（含权重，来自实际输出）：

- 谱质心落在 **1000–1600 Hz**（权重 20；低于 1000 = 闷，高于 1600 = 刺）
- 基频 10 分位 **≥175 Hz**（权重 15，"不会掉回成年男声"）
- 1000–2000 Hz 能量 ≥ −12 dB（6）、2000–3000 Hz ≥ −16 dB（6）、3000–4000 Hz ≥ −19 dB（6）、4000–6000 Hz ≥ −20 dB（6）
- 浊音比例 ≥55%（权重 11，"不被吞音"）
- 另有权重 30 的项（音高/稳定性主项）

## 4. 实测结论：手上所有模型都不可用 → 只能自训

同一句测试音频、同为 250 Hz 目标、同一台机器的结果：

| 模型 | 客观评分 | 谱质心 | 基频中位 |
| --- | --- | --- | --- |
| kazuki（现役，220 Hz / 共振峰 1.0 参数组） | 26 ✗ | 466 Hz | 220 Hz |
| guanguanV1 | 47 ✗ | 857 Hz | 244 Hz |
| kikiV1 | 47 ✗ | 591 Hz | 243 Hz |
| youzhanv2-xi | 56 ✗ | 498 Hz | 246 Hz |
| keruanV1 | 41 ✗ | 439 Hz | 246 Hz |
| （参考）goku | 100 ✓ | 1180 Hz | 250 Hz |

其中 250 Hz 单模型渲染实测值：`guanguanV1` f0=244.2 / centroid=857.2 / 47；`keruanV1` f0=245.6 / centroid=439.0 / 41；`kikiV1` f0=242.8 / centroid=591.2 / 47；`youzhanv2xi` f0=245.6 / centroid=498.0 / 56。

**归因（本轮最重要的判断）**：这些模型的谱质心全部落在 439–857 Hz，2 kHz 以上几乎没有能量。这是模型训练数据本身的物理上限，不是 pitch/formant 参数没调好——所以「换模型、调参数」这条路已经排除，唯一出路是拿未变声男孩原声自训一个 RVC，再用 `measure_voice.py` 卡 85 分。

**用户决策（本轮）**：明确弃用悟空音（goku）——`不要这个悟空音`。已把默认音色换回 kazuki 版「カズキ · 自然男童音」（约 220 Hz），goku 模型文件保留但不再是默认。

## 5. 语料选型（这是唯一的卡点）

| 语料 | 情况 | 结论 |
| --- | --- | --- |
| **ChildTalk**（南开 NKU-HLT；HF `yujie-ovo/ChildTalk`） | 2–8 岁多方言真实亲子对话，文件名带年龄/性别（如 `..._child_8_M_...`），含 `information/speaker_info.csv`、`dialogue_info.csv`、`short/`+`long/`、`wav.scp`+文本 | ✅ 唯一可用；**CC-BY-NC-SA-4.0，仅限非商用** |
| BAAI/ChildMandarin | 3–5 岁（偏小）、gated=auto 需接受许可 | ✗ |
| MatrixStudio/TTS-SCFChilSC | 5 岁**女童**、仅 215 条/15 分钟，**CC-BY-NC-ND-4.0（禁衍生 → 不能拿来训练）** | ✗ |
| ELRA-S0496 / Chinese Kids Speech DB / Nexdata 41h | 质量最对口，但付费或需申请 | 备用（商用必走这条） |

ChildTalk 包体：**21 个分片 × 1 GB ≈ 21.8 GB**（`part-aa`…`part-au`），合并方式 `cat childtalk.tar.gz.part-* > childtalk.tar.gz` 再 `tar -zxvf`；HF 仓库里只有根目录 tar 分片，`data/` 目录树 API 会失败，别以为仓库是空的。

## 6. 网络/环境事实（会反复咬人）

- `huggingface.co` 在本机解析到 **198.18.0.217（私有段）**，导致内置 fetch/url 工具报 `SSRF blocked`；但 **`curl` 走环境代理是通的**。
- 代理环境：`http_proxy/all_proxy/HTTPS_PROXY = http://127.0.0.1:7896`，`NO_PROXY` 含 `localhost,127.0.0.1,::1,10.242.192.0/24`，`NODE_USE_ENV_PROXY=1`；macOS 系统代理同端口 7896。
- 实测 5 MB range 下载：`huggingface.co` 228 KB/s，**`hf-mirror.com` 478 KB/s**（约 2×），断点续传可用 → 大文件走 hf-mirror + 多路并行。
- 磁盘：`/System/Volumes/Data` 926 GiB，剩 333 GiB，装得下 21.8 GB + 解压。

## 7. 落地计划（已开工，未完成）

1. 21 分片 4 路并行下到 `.build/datasets/childtalk/shards/`（可续传，实测约 1.6 MB/s 合计，ETA 3–4 h；检查点时已下到 `part-aa` 起步阶段），下完自动合并解压。
2. 从 `speaker_info.csv` 挑 **8 岁男孩 / 单人 / 普通话干净**的说话人。
3. 从 `short/` 切片捞 **20–30 分钟干声**（去停顿拼接，44.1 kHz → 40 kHz 单声道）。
4. `script/train_rvc.py` 训练 → 导出 npz。
5. `script/measure_voice.py` 打分，**≥85 且谱质心进 1000–1600** 才写进 App 设为默认并重新构建——不合格不装机。

补充：仓库 `docs/男童声线语料参考.md` 已记录 ChildTalk 论文（aclanthology.org/2026.findings-acl.251/）作为参考；若后续要商用，必须换 ELRA 付费语料或自录。

## 8. 可复用教训

- 音色"不像童声"时，先量**谱质心 + 分频带能量**再谈参数：模型上限决定天花板，调 pitch/formant 只是在下限附近打转。
- 数据集许可证要先读：`CC-BY-NC-ND`（禁衍生）在训练场景里等同于「不能用」，比"非商用"限制更硬。
- 同一被测句 + 同一目标音高 + 同一评分器，才能横向比较不同来源的 RVC 模型；跨仓库的分数不可直接对比。
