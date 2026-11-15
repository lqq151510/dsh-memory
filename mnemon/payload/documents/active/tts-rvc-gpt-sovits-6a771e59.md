---
id: "6a771e59-e1f5-4911-8d5b-87f7b50583f9"
title: "正太音（未变声少年男声）生成指南：TTS 提示词与 RVC/GPT-SoVITS 流程"
description: "把“正太音”翻译成声学参数的可复制 TTS 提示词模板、ElevenLabs 结构、B 站同款 RVC/GPT-SoVITS 训练与后期链、常见坑，以及 2026-11 核查的变声器与模型获取渠道。"
status: "active"
created_at: "2026-11-01T15:04:35.180Z"
updated_at: "2026-11-01T15:04:35.180Z"
content_hash: "9ff63ac726395b78745046ad3ec58bc0df0b656341c13aa143a62fb23c203b2d"
source_paths:
  []
session_ids:
  - "b38cfeba-0524-4cdf-9ab8-b5cc119ae6fe"
memory_body_ids:
  []
---

# 生成「正太音」（未变声少年男声）的提示词与工程化流程

面向目标：用 TTS / 音色设计模型或 RVC / GPT-SoVITS 得到接近 B 站 UP 主成品的清亮少年男声，而非伪音或变声器效果。

## 1. 关键认知：模型不吃「正太音」这个标签

音色设计类模型（ElevenLabs、MiniMax 音色设计、Fish Audio、CosyVoice2、Qwen3-TTS VoiceDesign）按声学参数建模，不理解「正太 / 少年音」这类二次元圈词汇。直接把这些词写进提示词，最常见的失败结果是生成**音调很高的女声**或**幼童奶音**。必须翻译成年龄、性别音质、音高、音色、共鸣位置、气息、语速、情绪这 8 个维度。

## 2. 声学维度 → 提示词对照

| 维度 | 正太音特征 | 英文提示词写法 |
|---|---|---|
| 年龄段 | 约 8～13 岁，未变声 | `10-year-old boy`, `pre-pubescent boy` |
| 性别音质 | 男童，不是女性 | `young boy`, `boyish`（禁用 female） |
| 音高 | 中高，不尖叫 | `medium-high pitch`, `light voice` |
| 音色 | 清亮、干净、略「薄」 | `clear, bright, light timbre` |
| 共鸣位置 | 靠前、口腔前部、轻微鼻音 | `forward resonance`, `slightly nasal` |
| 气息 | 短促、不拖、不过度气声 | `short breath support`, `not breathy` |
| 语速节奏 | 偏快、短句、顿挫明显 | `quick, bouncy rhythm`, `short sentences` |
| 情绪 | 略傲 / 软 / 元气 | `playful, slightly proud, soft` |

`pre-pubescent` 与 `no vocal rasp` 是核心否定约束，用于排除成年男性的低沉共鸣和沙哑感。

## 3. 提示词模板（ElevenLabs 官方推荐结构）

```
Native Chinese (Mandarin, standard). Male, 10–12 years old.
Persona: <2–5 词人设>. Emotion: <2–3 个情绪词>.
<1–2 句描述音色、语速、语气>
```

可直接用的三版：

**日常清爽型**
```
Native Chinese, standard Mandarin. Male, 10–12 years old, pre-pubescent.
Persona: curious young boy. Emotion: bright, relaxed, a little playful.
Clear, light, forward-placed boy's voice with medium-high pitch and a slightly
nasal, cheerful tone. Short breath support, no rasp, no adult male resonance.
Speaks at a brisk, natural pace with light ups and downs.
```

**元气傲娇型**
```
Native Chinese, standard Mandarin. Male, 9–11 years old, pre-pubescent.
Persona: proud little kid. Emotion: energetic, confident, slightly smug.
Bright and crisp childish male timbre, light and thin, forward in the mouth.
Quick, bouncy delivery with sharp short emphasis, ending sentences with a
confident lift. No breathiness, no rasp.
```

**冷淡安静型**
```
Native Chinese, standard Mandarin. Male, 11–13 years old, pre-pubescent.
Persona: quiet, aloof boy. Emotion: calm, detached, restrained.
Soft but clear young male voice, light timbre, medium-high pitch, minimal
breathiness. Slow, even pacing with flat intonation and short pauses.
```

**预览文本必须同步匹配角色**：ElevenLabs 文档明确说明提示词与预览台词冲突会导致跑偏——描述「安静少年」却用 `Hey! I can't stand this!!!` 这类兴奋台词，结果会变得奇怪。应改用「诶？你怎么知道这个……」等短台词。

## 4. 常见误区

- 直接把「萝莉 / 正太」当提示词：模型无对应锚点，常被拉向女声。
- 用 `female` / `woman` 想表达「细一点」：会真的生成女声。
- 滥用 FX 类词（`reverb`、`echo`、`phone`、`tape`）：ElevenLabs 文档指出会明显拉低音质。
- 用 `accent` 表达语调：它指口音，会触发方言漂移；语调要用 `intonation` 或 `delivery`。
- 年龄写到 5 岁以下：得到奶音 / 婴幼儿音，不是正太。
- 一次改多个变量：应先固定年龄与性别，再调音色，最后调节奏。
- 语言选择：中文模型（MiniMax 音色设计、Fish Audio、CosyVoice2、Qwen3-TTS VoiceDesign）用中文描述更准；ElevenLabs 用英文更准。

## 5. 为什么纯提示词到不了 B 站水准

B 站那类作品约 90% 靠「声线来源 + 表演工序 + 后期」，不是靠提示词。三条实际路线：

1. **真人声源克隆**：录或取得正太声线干声，喂 GPT-SoVITS 训练，之后任意合成。
2. **RVC 音色转换**：正常真人朗读 / 演唱，再转换音色；AI 翻唱基本全走这条。
3. **直接用现成模型**：下载他人训练的 .pth + .index 推理。

提示词零样本路线的致命问题是**每句一致性差、表演和语流平**，成不了作品，只适合用来「探音色」，遇到满意的立刻保存。

## 6. 工程化流程（B 站同款）

**第 1 步：素材**（决定上限）
- GPT-SoVITS 路线：1～5 分钟单人、无 BGM、无混响、无变声器痕迹的干声；儿童音源信噪比差，必须降噪 + 切分。
- RVC 路线：先录本人干净干声，音色靠转换解决。
- 合法性：自己的录音、获授权的配音片段或开源声库；**不要拿真人声优声音训练后商用**，B 站对 AI 内容侵权投诉严格。

**第 2 步：训练 / 转换**
- RVC 关键参数：成年男声转童声 `pitch shift +8～+12` 半音；成年女声转正太 `+4～+6` 更自然；`index rate 0.5～0.75`（音色像且不僵硬）。
- GPT-SoVITS：V2 / V3 / V4 均可；先只训 SoVITS 看音色，再训 GPT 层管韵律。

**第 3 步：表演**（像不像 B 站的分水岭）
- 按**短句**分段合成，不整段丢进去。
- 台词加语气词：`诶？`、`啊……`、`唔`、`真的假的`。
- 句尾明确处理：上扬=疑问，下落=平静，拖长=撒娇。
- 长句或激烈情绪最易「掉回成年男声」，掉音就拆句重来。

**第 4 步：后期链**
```
降噪 → 高通 80–120Hz 切低频轰鸣 → EQ 提亮 3–5kHz
→ 轻压缩 2:1~3:1 → 去齿音 → 响度归一 -16 LUFS（B 站标准）
```
混 BGM 时人声通常比背景高 6～10 dB。

**排障口诀**：音色不对 → 改模型或素材；音色对但很假 → 改表演分句；都对了但塑料 → 改后期。

## 7. 正太音专属坑

- 不用变声器 / 伪音处理过的素材训练：音质劣化会被模型学进去，推理输出抖动。
- 不用「夹子音」素材直接训：喉部紧绷的压嗓感会被学成「憋着说话」。
- 素材里不混成年男声：哪怕一句，音高范围会被拉低。
- 儿童音源常带房间混响，训练前必须去混响，否则生成声永远像在厕所。
- 唱与说是两个任务：翻唱走 RVC，说话走 GPT-SoVITS，同一套模型很难都做好。

## 8. 现成工具与模型生态（2026-11 核查）

**实时变声（“变声器”）**
- `w-okada/voice-changer`（VCClient）⭐约 21.3k，2026-09 仍在更新；v2.2.2-beta 起按 edition 区分支持模型。**onnx edition 在 mac(M 系) 上支持 Beatrice + RVC**，是 Mac 上用 RVC 实时变声的正路；std edition 仅 Beatrice（内置音色不可换）。Windows/mac 版从 Hugging Face 仓库 `wok000/vcclient000`、`wok000/light_vcclient_beatrice` 下载。
- 商业预设：Voicemod（Tuna 社区有 Kid Voice / Little Kid Voice）、iMyFone MagicMic（标称 500+ 音色）。只能选不能训，音色偏「幼童」而非「少年」，实际听感较假。

**训练 / 推理框架**
- `RVC-Project/Retrieval-based-Voice-Conversion-WebUI` ⭐约 39.0k，MIT，≤10 分钟语音可训出不错模型。
- `RVC-Boss/GPT-SoVITS` ⭐约 64.5k，MIT，1 分钟语音可训 TTS（少样本克隆）。
- `IAHispano/Applio` ⭐约 3.9k，MIT，2026-10 仍更新，比 RVC 原版 WebUI 更易用，适合 Mac 上做推理/训练。

**模型获取渠道**：B 站（搜「正太音 RVC 模型」，下载链接常在简介/评论）、妙音 `klrvc.com`、`voice-models.com`（3 万+ 模型）、Hugging Face、GitHub 分享仓库（如 `Shinowing/Shinowing_AI_model`，B 站 UP 主独孤欲雪的模型库——实测目录只有洛琪希、Saber、菲兹等二次元角色，无正太音）。

**核查结论**：没有专门为正太音做的现成变声器；工具通用、模型散落、效果靠调。现成正太音模型多为二次元少年角色训练，泛化到日常说话易崩，稳定方案仍是自训。

**取证注意**：`weights.gg` 已失效/跳转（重定向到 weights.com，搜索页 404），不要再作为模型来源推荐；`voice-models.com` 的 `/api/search` 与 `/api/models/search` 均 404，只能用网页搜索页。

## 9. 兼容性与验收提醒

- **模型与变声器版本要匹配**：RVC v1 模型在 v2 界面可能加载失败，下载时看说明。
- **`.pth` 与 `.index` 成对**：只给 pth 也能用但音色会飘，index 是稳定器。
- 抽取 / 训练出音色后，**保存并在完整台词里连续听 3～5 句**验收，重点听是否中途掉回成年男声。
- 角色配音场景要把「音色描述」和「表演情绪指令」分两段写，不要混在一句里（Claude/Gemini/GPT 系 TTS 支持自然语言情绪控制）。
- 正太音属「稀缺音色」，纯提示词抽中概率不高；先在音色库按 `young boy` / `child` 标签找接近的现成音色再微调，比从零抽稳定。
