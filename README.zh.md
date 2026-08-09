# MiniMax H3 视频提示词规范（开源生产力版）

> 一份把官方密集规则「翻译成生产力」的开源规范。读规则 → 抄模板 → 出成品。
> License: CC-BY-4.0 · 非正式官方文档，属社区生产力层。

---

## 0. 这是什么 / 为什么另写一份

MiniMax 官方 `VIDEO_PROMPT_WRITING_GUIDE` 知识密度高，但拿到后要自己翻译成可用提示词。本规范做三件事：

1. 把规则压缩成**可抄的模板**（每种模式一份，填空即用）。
2. 给**对照范例**（好 / 坏 / 对齐前后），照着写不踩坑。
3. 配一份**交互式生成器**（见 `prompt-builder.html`），填表即出可复制提示词。

**适用范围**：MiniMax H3（Hailuo 3.0）文生视频 / 图生视频 / 首末帧 / 尾帧 / 参考图视频。

**来源**：MiniMax 官方仓库 `MiniMax-AI/MiniMax-H3`（`VIDEO_PROMPT_WRITING_GUIDE_base_en.md`）；HuggingFace `MiniMaxAI/MiniMax-H3`；社区实践（ambienceai / genie11 / rundiffusion）。本文件为社区二次整理，不替代官方指南。

---

## 1. 模型事实（写之前必须知道）

| 项 | 值 |
|---|---|
| 架构 | 33B dense single-stream Omni Transformer |
| 音视频 | 原生同生（视频与音轨一次采样产出，非后期配音） |
| 帧率 / 音频 | 24fps / 32kHz 立体声 |
| 时长 | 4–15 秒 |
| 分辨率 | 768p base；2K 需官方 H3-Regenerate-2K（开源权重未含 Context-IR 预处理） |
| 提示词语言 | **英文**（官方示例全英文；中文可跑但对齐度下降） |

关键推论：声音必须**显式写**，模型不会自动「猜」该有什么声。

---

## 2. 三段式字段结构（所有模式通用，最高优先级）

最终提示词 = 三个字段，**字段之间空一行**拼接：

```
<integrated_multimodal_description>

<overall_soundscape>

<non_diegetic_music>
```

| 字段 | 作用 | 写法要点 |
|---|---|---|
| `integrated_multimodal_description` | 画面 + 动作 + 对话 + 摄像机，按时间顺序叙述 | 主体部分，放最前 |
| `overall_soundscape` | 1–4 句英文，总环境声、物理动作声、非语言人声（呼吸、脚步、衣料摩擦、餐具碰撞） | **不写台词内容** |
| `non_diegetic_music` | 1–3 句，描述配乐乐器、节奏、音色 | **禁用抽象情绪词**（不写 emotional / epic / melancholic）；不需要写 `N/A` |

**为什么必须分三段**：合并成一整段 → 音轨质量塌陷、环境声缺失。这是最高优先级约束。

---

## 3. 模式选择（先选对，再写）

| 模式 | 输入 | 何时用 | 对齐行 |
|---|---|---|---|
| **T2VA** | 纯文本 | 无参考图，从零描述 | 无（直接三段式） |
| **I2VA** | 1 张首帧图 | 图生视频，锁定开头画面 | 首帧对齐行（见 §4） |
| **L2VA** | 1 张尾帧图 | 图生视频，锁定结尾画面 | 尾帧对齐行（见 §4） |
| **FL2VA** | 首帧 + 尾帧 2 张图 | 两段之间连续插值，单镜头 | 同时给首 + 末对齐行 |
| **Ref2VA** | ≤9 图 + ≤3 视频 + ≤3 音频 | 多参考决定外观，不做帧对齐 | 改写 `is referenced for the appearance of ...` |

**FL2VA 关键**：它倾向生成**单镜头连续插值**，不要在 FL2VA 里塞多镜头切换。
**Ref2VA 关键**：不做帧对齐，用 `is referenced for the appearance of ...` 描述每张参考图决定「谁的外观」。

---

## 4. 帧对齐指令 —— 逐字模板（I2VA / L2VA / FL2VA 必放最开头）

I2VA / L2VA / FL2VA 必须把这行放在**提示词最开头**，后接空行，再写三段式：

```
For the target video, at 0.00 seconds into the target video, <Picture 1> (from [Shot 1]) is fully referenced.
```

其他模式对齐行：

| 模式 | 对齐行（逐字复制，替换 `<Picture N>`） |
|---|---|
| I2VA 首帧 | `at 0.00 seconds into the target video, <Picture 1> (from [Shot 1]) is fully referenced.` |
| L2VA 尾帧 | `at the last frame of the target video, <Picture 1> (from [Shot 1]) is fully referenced.` |
| FL2VA 首末帧 | 同时给 `<Picture 1>` @0.00s 与 `<Picture 2>` @last frame |

`<Picture 1>` 填你对首帧图的简短指代（如 `the woman in red coat`、`the cafe interior`）。

缺帧对齐首行 → 首帧不锁，人物换脸。

---

## 5. 图生视频核心心法（I2VA / L2VA / FL2VA）

1. **首帧已定义外观，提示词只写「之后发生什么」**。复述图里已有细节 = 浪费 token 且可能引发冲突。
2. 只在需「锁住」时复述身份特征（同一张脸、同一件衣服、同一光线），一句带过。
3. **禁止与首帧冲突**（图是白天写 night、图是坐姿写 standing → 直接崩）。
4. 单镜头 5 秒内安排 **2–3 个动作 beats**，多了变加速播放。
5. 声音必须显式定义。
6. 负向不靠 negative prompt，靠正向写清「保持不变的东西」。

---

## 6. 摄像机运动语法

H3 弃用 Hailuo 02 的方括号标签堆叠（`[Zoom in][Pan left]`）。**方括号在 H3 只用于 `[Shot N]`**。

写法：`运动类型 + 幅度 + 速度`，融进自然句。

- 幅度词：`small amplitude` / `large amplitude`
- 速度词：`slow speed` / `fast speed`
- 类型词：Zoom in/out、Push in、Pull out、Pan left/right、Truck left/right、Tilt up/down、Pedestal up/down、Arc、Tracking shot、Static shot、Handheld shake、POV、Roll

示例：`The camera performs a slow-speed, small-amplitude push in toward her face.`

仅有距离或微角度变化时用摄像机运动描述，**不要写成 cut**。

---

## 7. 多镜头时间戳

```
[Shot 1] （不带时间戳，默认从 0 开始）
[Shot 2] At 00:03.500, the camera cuts to a close-up of ...
[Shot 3] At 00:07.200, ...
```

时间戳必须严格递增，格式 `MM:SS.mmm`。不递增 → 镜头顺序错乱。

---

## 8. 对话语法

- 说话者标注：`(S1)` `(S2)` `(S1,S2)`
- 台词包裹：`<d>[English] Are you serious right now?</d>`
  - 方括号内只写语言标签，台词写原文，**不做翻译**
- 画外音：`says in an off-screen voiceover ... while his lips remain completely closed`
- 密度：约 **20 词对话 / 15 秒**，单行约 10 词。超量 → 语速失真

---

## 9. 字数指引

| 场景 | 建议长度 |
|---|---|
| 单镜头 5s I2VA | 80–150 词（description 部分） |
| 多镜头 10–15s | 200–350 词 |
| soundscape | 1–4 句 |
| music | 1–3 句或 N/A |

---

## 10. 可抄模板（生产力核心）

### 10.1 I2VA（图生视频，最常用）

```
For the target video, at 0.00 seconds into the target video, <Picture 1> (from [Shot 1]) is fully referenced.

[Shot 1] <一句锁定身份与场景不变> Then <动作 beat 1>. <动作 beat 2>. <动作 beat 3>. The camera performs a <speed>-speed, <amplitude>-amplitude <motion type>. <光线/材质/颗粒等持续性描述>.

<环境声 1–4 句，含物理动作声与非语言人声>

<配乐 1–3 句，具体乐器与节奏，或 N/A>
```

### 10.2 T2VA（纯文生视频）

```
<主体与场景一句话建立> <动作 beat 1>. <动作 beat 2>. <动作 beat 3>. The camera performs a <speed>-speed, <amplitude>-amplitude <motion type>.

<环境声 1–4 句>

<配乐 1–3 句，或 N/A>
```

### 10.3 FL2VA（首末帧连续插值，单镜头）

```
For the target video, at 0.00 seconds into the target video, <Picture 1> (from [Shot 1]) is fully referenced.
At the last frame of the target video, <Picture 2> (from [Shot 1]) is fully referenced.

[Shot 1] <首末之间连续发生的动作，不要切镜头> The camera performs a <speed>-speed, <amplitude>-amplitude <motion type>.

<环境声 1–4 句>

<配乐 1–3 句，或 N/A>
```

### 10.4 L2VA（尾帧对齐）

```
For the target video, at the last frame of the target video, <Picture 1> (from [Shot 1]) is fully referenced.

[Shot 1] <开场> <动作 beats 向尾帧状态演进>. The camera performs a <speed>-speed, <amplitude>-amplitude <motion type>.

<环境声 1–4 句>

<配乐 1–3 句，或 N/A>
```

### 10.5 Ref2VA（多参考决定外观，无帧对齐）

```
<Picture 1> is referenced for the appearance of <主体 A>. <Picture 2> is referenced for the appearance of <主体 B>.

[Shot 1] <动作 beats>. The camera performs a <speed>-speed, <amplitude>-amplitude <motion type>.

<环境声 1–4 句>

<配乐 1–3 句，或 N/A>
```

---

## 11. 对照范例

### 范例 A：I2VA 好 vs 坏

**坏（复述首帧 + 缺声音 + 动作超量）**
```
A woman with long black hair standing in a red coat in a cafe. She walks, turns, laughs, drinks, looks at phone, opens door, smiles, leaves. [Zoom in]
```
问题：复述图已有外观、无 soundscape/music、8 个动作塞 5 秒、用了方括号标签。

**好**
```
For the target video, at 0.00 seconds into the target video, the woman in the red coat (from [Shot 1]) is fully referenced.

[Shot 1] She remains seated by the window, then slowly brings the cup to her lips and takes a sip, sets it down, and glances toward the rain-streaked glass. The camera performs a slow-speed, small-amplitude push in toward her face. Warm afternoon light continues to fall across the table.

Soft cafe ambience with distant espresso machine hiss, the clink of a ceramic cup on saucer, and quiet rainfall against the window.

A lone nylon-string guitar plays a gentle, mid-tempo lo-fi pattern with brushed percussion.
```

### 范例 B：多镜头时间戳

```
[Shot 1] A lone figure walks across an empty station platform.
[Shot 2] At 00:03.500, the camera cuts to a close-up of his weary eyes reflecting the flickering fluorescent light.
[Shot 3] At 00:07.200, a wide shot reveals the approaching train's headlights piercing the tunnel dark.

Echoing footsteps on tile, a distant train rumble growing louder, fluorescent hum.

A low sustained cello drone with a slow building timpani pulse.
```

---

## 12. 常见错误清单

| 错误 | 后果 |
|---|---|
| 三段式合并成一整段 | 音轨质量塌陷，环境声缺失 |
| 缺帧对齐首行（I2VA/L2VA/FL2VA） | 首帧不锁，人物换脸 |
| 用 `[Zoom in]` 方括号标签 | H3 不解析，当普通文本 |
| music 写 "emotional / cinematic mood" | 官方明令禁止抽象情绪词 |
| 5 秒塞 5+ 个动作 | 动作加速、糊帧 |
| 对话超 20 词 / 15 秒 | 语速异常 |
| 描述与首帧矛盾 | 画面撕裂、身份漂移 |
| 多镜头时间戳不递增 | 镜头顺序错乱 |
| 中文写提示词 | 对齐度下降（官方全英文示例） |

---

## 13. 起飞前检查单（Pre-flight）

- [ ] 三段式分三段、段间空一行
- [ ] 选对模式，I2VA/L2VA/FL2VA 首行有逐字对齐句
- [ ] 图生视频未复述首帧已有外观（只写「之后发生什么」）
- [ ] 无与首帧冲突的描述
- [ ] 单镜头 5s 内动作 beats ≤ 3
- [ ] 摄像机运动用「类型+幅度+速度」，无方括号标签
- [ ] soundscape 写了，且不含台词
- [ ] music 写了具体乐器/节奏，无抽象情绪词（或 N/A）
- [ ] 对话 ≤ 20 词 / 15 秒，台词带 `<d>[lang]... </d>`
- [ ] 多镜头时间戳严格递增 `MM:SS.mmm`
- [ ] 全文英文

---

## 14. 一页速查表

```
结构：desc（空行）soundscape（空行）music
对齐：I2VA 首帧 / L2VA 末帧 / FL2VA 首+末 / Ref2VA 改写法
镜头：[Shot N] 仅此用方括号；时间戳 MM:SS.mmm 递增
运动：类型+幅度(small/large)+速度(slow/fast)，融进句子
对话：<d>[English] ...</d>，20词/15秒
音乐：写乐器+节奏，禁 emotional/epic/melancholic，无则 N/A
长度：5s→80-150词；15s→200-350词
铁律：三段不合并 / 不冲突首帧 / 声音显式写 / 英文
```

---

## 15. 许可证与署名

- **License**：CC-BY-4.0。可自由复制、改编、再分发，需署名来源。
- **署名**：本规范改编自 MiniMax 官方 `MiniMax-AI/MiniMax-H3` 视频提示词指南及社区实践，本文件为社区生产力整理层，非官方文档。
- **免责**：模型能力随版本演进，以官方最新指南为准。

---

### 配套工具

`prompt-builder.html`：单文件交互式提示词生成器。选模式 → 填字段 → 实时生成可复制提示词，内置错误校验（动作 beats 超量、对话超词、music 禁词）。无需联网、无依赖。
