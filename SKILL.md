---
name: minimax-h3-prompt
description: MiniMax H3 (Hailuo 3.0) 视频提示词编写规范与可套模板。当需要为 MiniMax-H3 写 T2VA / I2VA(图生视频) / FL2VA(首末帧) / L2VA(尾帧) / Ref2VA 提示词时使用，覆盖官方三段式字段结构（带字段标签）、五种模式选择、帧对齐指令逐字模板、摄像机运动语法、多镜头时间戳、对话与声景写法、参考图角色分配、时序连接词、时长画幅甜区、对照范例、常见错误、起飞前检查单、一页速查。触发词：minimax h3、hailuo 3、图生视频提示词、i2v prompt、首帧对齐、三段式提示词、视频提示词规范、h3 提示词优化。
agent_created: true
updated: 2026-08-13
---

# MiniMax H3 视频提示词规范（优化版）

来源：官方仓库 `github.com/MiniMax-AI/MiniMax-H3` → `VIDEO_PROMPT_WRITING_GUIDE_base_en.md`；`huggingface.co/MiniMaxAI/MiniMax-H3`；社区实践（cdance.ai / photogptai.com / domoai.app / promptslove.com / morphed.app / atlabs.ai / jxp.com），经 2026-08-13 全网检索交叉验证。

开源规范与图文资产（引流用）：`github.com/minhchee/minimax-h3-prompt-spec`。

本 skill 的用法：**选模式 → 抄 §11 模板 → 过 §14 检查单出片**。规则讲完即给可直接套的成品。

## 0. 模型事实（写提示词前必须知道）

| 项 | 值 |
|---|---|
| 正式名 / 别名 | MiniMax H3 = MiniMax-H3（API model ID）= Hailuo 3.0 = Hailuo 03（消费端名）；Hailuo 是产品/App 名，H3 是模型。勿与 Hailuo 2.3 / 2.0 混淆 |
| 发布 | 2026-07-31，33B dense single-stream Omni Transformer |
| 音视频 | 原生同生（视频与音轨一次采样产出，非后期配音），立体声 32kHz |
| 帧率 / 时长 | 24fps / 4–15 秒（API 实测可 4s 起） |
| 分辨率 | 768p 基线；2K 需官方 H3-Regenerate-2K（开源权重未含 Context-IR 预处理，本地跑 ≠ 托管生成） |
| 提示词长度 | 上限约 7000 字符；**甜区 50–120 词**，超过约 120 词开始引入矛盾而非增加控制力 |
| 提示词语言 | 英文（官方 guide 全英文示例；中文可跑但对齐度下降） |
| 参考上限 | 单次生成 ≤9 图 + ≤3 视频 + ≤3 音频（合计 ≤12 文件）；**音频不能单独提交**，须至少配 1 图或 1 视频 |

关键结论：声音必须**显式写**——模型不会自动"猜"该有什么声音。

## 1. 核心结构：三段式字段（最高优先级）

最终提示词 = 三个**带标签**字段拼接，字段之间空一行——标签 `:` 必须写（官方格式，旧资料常漏）：

```
integrated_multimodal_description: <画面+动作+对话+摄像机，按时间顺序>

overall_soundscape: <环境声/物理动作声/非语言人声，1–4 句>

non_diegetic_music: <配乐乐器/节奏/音色，1–3 句，或 N/A>
```

| 字段 | 角色 | 写法 |
|---|---|---|
| `integrated_multimodal_description` | 画面+动作+镜头切换+对话+同期声，按播放顺序叙述 | 主体，放最前 |
| `overall_soundscape` | 1–4 句英文，总环境声、物理动作声、非语言人声（呼吸、脚步、衣料摩擦、餐具碰撞） | **不写台词内容** |
| `non_diegetic_music` | 1–3 句，描述配乐的乐器、节奏、音色 | **禁用抽象情绪词**（不写 "emotional"、"epic"、"melancholic"、"cinematic mood"）；不需要配乐写 `N/A` |

**为什么必须三段分离**：合并成一段 → 音轨质量塌陷，环境声缺失。这是最高优先级约束。

**心智模型（可选，帮助不漏维度）**：最强 H3 提示词读起来像一份简短制作计划，而非一堆形容词。主体（2–3 具体外貌/材质细节）+ 动作（按时间、具体动词）+ 环境（时间/天气/场景）+ 视觉风格（只选一种并坚持）+ 摄像机（一个运镜，含类型+幅度+速度）+ 音频（H3 原生出声，不写就是模型替你选）。

## 2. 五种模式与选择

先决定"从什么出发"，选错模式是提示词掉价的最常见原因：

| 模式 | 提供的输入 | 提示词第一任务 | 对齐行 |
|---|---|---|---|
| T2VA | 无 | 从文本建立完整视听场景 | 无（直奔三段式） |
| I2VA | 1 张首帧 | 完全锚定首图，再写"之后发生什么" | 首帧对齐（§3） |
| L2VA | 1 张末帧 | 倒推 plausible 的早前序列，落在这张终帧 | 尾帧对齐（§3） |
| FL2VA | 首帧+末帧 | 两张图之间连续插值，单镜头 | 同时给首+末对齐 |
| Ref2VA | ≤9 图 + ≤3 视频 + ≤3 音频 | 给每个素材分配角色（见 §8） | 改写 `is referenced for the appearance of ...` |

**FL2VA 关键点**：倾向生成**单镜头连续插值**，不要在 FL2VA 里塞多镜头切换。
**Ref2VA 关键点**：不做帧对齐；用 `is referenced for the appearance of ...` 描述哪张参考决定"谁的长相"。

## 3. 帧对齐指令 —— 逐字模板

I2VA / L2VA / FL2VA 必须把这行放在**提示词最开头**（在 `integrated_multimodal_description:` 之前），后接空行：

```
For the target video, at 0.00 seconds into the target video, <Picture 1> (from [Shot 1]) is fully referenced.
```

其他模式：

| 模式 | 对齐行（逐字复制，替换 `<Picture N>`） |
|---|---|
| I2VA 首帧 | `at 0.00 seconds into the target video, <Picture 1> (from [Shot 1]) is fully referenced.` |
| L2VA 尾帧 | `at the last frame of the target video, <Picture 1> (from [Shot 1]) is fully referenced.` |
| FL2VA 首末帧 | 同时给 `<Picture 1>` @0.00s 与 `<Picture 2>` @last frame |
| Ref2VA | 不做帧对齐，改写 `<Picture N> is referenced for the appearance of ...`，≤9 图 + ≤3 视频 + ≤3 音频 |

`<Picture 1>` = 指向首帧图的短描述（如 `the woman in red coat`、`the cafe interior`）。

缺对齐行 → 首帧不锁，人物换脸。

## 4. 图生视频核心规则 + 时序连接词（核心技巧）

1. **首帧已定义外观，提示词只写"之后发生什么"**。重复描述图里已有的细节属浪费 token 且可能引发冲突。
2. 只在需"锁住"时复述身份特征（同一张脸/同件衣服/同光线），一句带过。
3. **禁止与首帧冲突**（图是白天写 night、图是坐姿写 standing → 直接崩）。
4. 单镜头 5 秒内安排 **2–3 个动作 beats**，多了变加速播放。
5. 声音必须显式定义，模型不会自动"猜"该有什么声音。
6. 负向不靠 negative prompt，靠正向写清"保持不变的东西"。

**时序连接词是关键**（旧 Hailuo 02 指南不强调，H3 必加）：用 `First, then, as, finally` 等连接词给 H3 显式 beat 顺序。不写则模型默认"同时发生"，产出糊成一团的 everything-at-once 片段。

> 例：先她抬起杯子，*then* 蒸汽卷起捕捉窗光，*as* 她转头，*finally* 杯子落回桌面。

**结尾状态必须显式指定**：写清镜头应在哪停下（"ends with the cup lowered to the table, gaze still off-frame"）。不写则模型随意收尾。

## 5. 摄像机运动语法

H3 **弃用** Hailuo 02 的方括号标签堆叠（`[Zoom in][Pan left]`）。**方括号在 H3 只用于 `[Shot N]`**。

写法：`运动类型 + 幅度 + 速度`，融进自然句。

- 幅度词：`small amplitude` / `large amplitude`（省略=中）
- 速度词：`slow speed` / `fast speed`（省略=正常）
- 类型词：Zoom in/out、Push in、Pull out、Pan left/right、Truck left/right、Tilt up/down、Pedestal up/down、Arc、Tracking shot、Static shot、Handheld shake（slightly/strongly）、POV、Roll（clockwise/counterclockwise）

示例：`The camera performs a slow-speed, small-amplitude push in toward her face.`

若需锁定画面不动，必须显式写 `holds a static shot`，否则模型会漂移。仅有距离/微角度变化用运镜描述，**不要写成 cut**。社区方括号语法（最多叠 3 个）未经官方确认，默认用自然语言。

## 6. 多镜头时间戳

```
[Shot 1] （不带时间戳，默认从 0 开始）
[Shot 2] At 00:03.500, the camera cuts to a close-up of ...
[Shot 3] At 00:07.200, ...
```

时间戳必须严格递增，格式 `MM:SS.mmm`；范围顺序无间隙无重叠；首镜 00:00.000 起；末镜止于总时长。每镜 2–5 秒，短于 1.5s 不给模型足够帧数；多镜=更多跨切不一致风险。

## 7. 对话语法

- 说话者标注：`(S1)` `(S2)` `(S1,S2)`
- 台词包裹：`<d>[English] Are you serious right now?</d>`
  - 方括号内只写语言标签，台词写原文，**不做翻译**
- 画外音：`says in an off-screen voiceover ... while his lips remain completely closed`
- 密度：约 **20 词对话 / 15 秒**，单行约 10 词。超量导致语速失真

## 8. 参考图角色分配（Ref2VA）

每个上传文件须在提示词正文里**用自然语言声明角色**（不是 metadata）。不分配角色的参考素材会被模型自行决定影响什么，这是身份漂移的主因。

| 角色 | 含义 | 类型 |
|---|---|---|
| identity | 角色面容/外观锁定 | 图 |
| wardrobe | 要套用的服装 | 图 |
| style | 颜色/纹理/美学方向 | 图 |
| environment | 背景环境 | 图 |
| motion | 要复刻的编舞或机位路径 | 视频 |
| edit_target | 要修改的已有片段 | 视频 |
| voice | 要克隆的嗓音特征 | 音频 |
| music | 供视觉同步的节奏结构 | 音频 |

示例：`Image 1 = character identity reference. Video 1 = camera movement reference. Audio 1 = background music.`

反模式：把多个文件一股脑丢进去不给角色——输出不可预测。

## 9. 时长 / 画幅甜区

| 用途 | 建议时长 | 画幅 | 避坑 |
|---|---|---|---|
| 短视频/社媒 | 5–6s | 9:16 | 主体太小、广角构图 |
| 产品物理镜头 | 5–8s | 1:1 或 16:9 | 让产品在不同房间来回移动 |
| 电影感场景 | 8–12s | 16:9 或 21:9 | 堆一堆风格形容词 |
| 锁定身份（参考模式） | 6–10s | 任意 | 让每个参考都控制一切 |

先用最短时长传达想法；动作需清晰起承转合才加时。画幅按投放地选：9:16（抖音/IG）、16:9（YouTube）、1:1（LinkedIn/推特/FB/Pinterest）。

**成本分层渲染**（生产建议）：① 768p 草稿测 prompt+参考组合 → ② 锁定创意方向 → ③ 2K 终渲 → ④ 指令编辑做定点修正。2K 约 $0.08–0.13/秒，先 768p（约 60% 成本）可省大量迭代费。

## 10. 字数指引

| 场景 | 建议长度 |
|---|---|
| 单镜头 5s I2VA | 80–150 词（description 部分） |
| 多镜头 10–15s | 200–350 词 |
| 甜区上限 | **~120 词**（再多开始引入矛盾） |
| soundscape | 1–4 句 |
| music | 1–3 句或 N/A |

## 11. 可套模板（5 模式全，直接抄）

**I2VA（图生视频，最常见）**
```
For the target video, at 0.00 seconds into the target video, <Picture 1> (from [Shot 1]) is fully referenced.

integrated_multimodal_description: [Shot 1] <一句锁定身份与场景不变> Then <动作 beat 1>. <动作 beat 2>. <动作 beat 3>. The camera performs a <speed>-speed, <amplitude>-amplitude <motion type>. <光线/材质/颗粒等持续性描述>. Ends with <ending state>.

overall_soundscape: <1–4 句环境声/物理声/非语言人声>

non_diegetic_music: <1–3 句具体乐器与节奏，或 N/A>
```

**T2VA（无参考，从文本建全场）**
```
integrated_multimodal_description: [Shot 1] Live-action, cinematic and photorealistic. <Subject, 2–3 concrete details> <Environment, time/weather>. First <action beat 1>, then <beat 2>, as <beat 3>. The camera performs a <speed>-speed, <amplitude>-amplitude <motion type>. <Lighting & style: direction + quality, not mood words>. Ends with <ending state>.

overall_soundscape: <1–4 句环境声/物理声/非语言人声>

non_diegetic_music: <1–3 句具体乐器与节奏，或 N/A>
```

**FL2VA（首+末帧，单镜头插值）**
```
For the target video, at 0.00 seconds into the target video, <Picture 1> (from [Shot 1]) is fully referenced.
At the last frame of the target video, <Picture 2> (from [Shot 1]) is fully referenced.

integrated_multimodal_description: [Shot 1] <首末帧之间的连续动作，无切镜> The camera performs a <speed>-speed, <amplitude>-amplitude <motion type>.

overall_soundscape: <1–4 句环境声/物理声/非语言人声>

non_diegetic_music: <1–3 句具体乐器与节奏，或 N/A>
```

**L2VA（尾帧对齐）**
```
For the target video, at the last frame of the target video, <Picture 1> (from [Shot 1]) is fully referenced.

integrated_multimodal_description: [Shot 1] <opening> <action beats evolving toward the last-frame state>. The camera performs a <speed>-speed, <amplitude>-amplitude <motion type>.

overall_soundscape: <1–4 句环境声/物理声/非语言人声>

non_diegetic_music: <1–3 句具体乐器与节奏，或 N/A>
```

**Ref2VA（多参考，无帧对齐）**
```
<Picture 1> is referenced for the appearance of <subject A>. <Picture 2> is referenced for the appearance of <subject B>.

integrated_multimodal_description: [Shot 1] <action beats>. The camera performs a <speed>-speed, <amplitude>-amplitude <motion type>.

overall_soundscape: <1–4 句环境声/物理声/非语言人声>

non_diegetic_music: <1–3 句具体乐器与节奏，或 N/A>
```

## 12. 对照范例（好 / 坏）

**A. I2VA 坏 vs 好**

坏（复述首帧 + 无声音 + 动作超量 + 方括号标签）：
```
A woman with long black hair standing in a red coat in a cafe. She walks, turns, laughs, drinks, looks at phone, opens door, smiles, leaves. [Zoom in]
```
问题：复述外观、无 soundscape/music、5 秒塞 8 个动作、方括号标签。

好：
```
For the target video, at 0.00 seconds into the target video, the woman in the red coat (from [Shot 1]) is fully referenced.

integrated_multimodal_description: [Shot 1] She remains seated by the window, then slowly brings the cup to her lips and takes a sip, sets it down, and glances toward the rain-streaked glass. The camera performs a slow-speed, small-amplitude push in toward her face. Warm afternoon light continues to fall across the table.

overall_soundscape: Soft cafe ambience with distant espresso machine hiss, the clink of a ceramic cup on saucer, and quiet rainfall against the window.

non_diegetic_music: A lone nylon-string guitar plays a gentle, mid-tempo lo-fi pattern with brushed percussion.
```

**B. 多镜头时间戳**
```
[Shot 1] A lone figure walks across an empty station platform.
[Shot 2] At 00:03.500, the camera cuts to a close-up of his weary eyes reflecting the flickering fluorescent light.
[Shot 3] At 00:07.200, a wide shot reveals the approaching train's headlights piercing the tunnel dark.

overall_soundscape: Echoing footsteps on tile, a distant train rumble growing louder, fluorescent hum.

non_diegetic_music: A low sustained cello drone with a slow building timpani pulse.
```

## 13. 常见错误清单

| 错误 | 后果 |
|---|---|
| 三段式合并成一整段（无标签/无空行） | 音轨质量塌陷，环境声缺失 |
| 缺帧对齐首行 | 首帧不锁，人物换脸 |
| 用 `[Zoom in]` 方括号标签 | H3 不解析，被当普通文本 |
| music 写 "emotional / cinematic mood" | 官方明令禁止抽象情绪词 |
| 5 秒塞 5+ 个动作 | 动作加速、糊帧 |
| 对话超 20 词 / 15 秒 | 语速异常 |
| 描述与首帧矛盾 | 画面撕裂、身份漂移 |
| 多镜头时间戳不递增 | 镜头顺序错乱 |
| 缺时序连接词（First/then/as/finally） | everything-at-once 糊片段 |
| 不写结尾状态 | 模型随意收尾 |
| 参考素材不给角色 | 身份/风格漂移 |
| 提示词超 ~120 词 | 开始引入内部矛盾 |
| 中文写提示词 | 对齐度下降（官方示例全英文） |

## 14. 起飞前检查单（出片必过）

- [ ] 三段字段分离，彼此空一行
- [ ] 模式已选；I2VA/L2VA/FL2VA 最顶有逐字对齐行
- [ ] 图生视频不复述首帧外观（只写"之后发生什么"）
- [ ] 无任何描述与首帧冲突
- [ ] 单 5s 镜头 ≤ 3 个动作 beats
- [ ] 摄像机用"类型+幅度+速度"自然语言，无方括号标签
- [ ] soundscape 已写且不含台词
- [ ] music 写具体乐器/节奏，无抽象情绪词（或 N/A）
- [ ] 对话 ≤ 20 词 / 15 秒，台词包在 `<d>[lang]... </d>`
- [ ] 多镜头时间戳严格递增 `MM:SS.mmm`
- [ ] 整篇英文

## 15. 一页速查

```
结构: description (空行) soundscape (空行) music
对齐: I2VA 首帧 / L2VA 尾帧 / FL2VA 首+尾 / Ref2VA 改写
镜头: [Shot N] 仅方括号; 时间戳 MM:SS.mmm 递增
摄像机: 类型+幅度(small/large)+速度(slow/fast), 融进句子
对话: <d>[English] ...</d>, 20 词/15 秒
音乐: 乐器+节奏, 禁 emotional/epic/melancholic, 否则 N/A
字数: 5s→80-150 词; 15s→200-350 词; 上限~120
铁律: 三段不并 / 不冲突首帧 / 声音显式 / 全英文
```

## 16. 本地 ComfyUI 执行链路（本地环境）

执行部分已独立成 skill `comfyui-minimax-h3-i2v`（自带 UI→API 转换器资产与故障排查），本 skill 只管提示词。

- 工作流：`<你的 ComfyUI 工作流目录>/video_minimax_h3_i2v.json`（含 subgraph）
- 转换器脚本（固化版）：`<WORKBUDDY_SKILLS_DIR>/comfyui-minimax-h3-i2v/assets/gen_video.py`
- 模型：minimax_h3_fl2va_pruned_int8_convrot / qwen3vl_32b_minimax_h3_nvfp4_awq / minimax_h3_video_vae_fp16 / minimax_h3_audio_vae_fp32
- 输入图放 `<你的 ComfyUI input 目录>/your_input_image.png`
- 实测出片：576×768 / 124 帧 @24fps / ~5.2s / AAC 32kHz 立体声

> 注意：凡事实/版本/兼容性拿不准的，先交叉验证再下结论。本 skill 2026-08-13 已据全网官方+社区资料复核，并据开源规范 `minimax-h3-prompt-spec` 补齐 5 模式模板、对照范例、起飞前检查单与一页速查。
