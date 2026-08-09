# MiniMax H3 Video Prompt Specification (Open Productivity Edition)

> A community spec that turns MiniMax's dense official guide into copy-paste productivity. Read the rules → copy a template → ship a prompt.
> License: CC-BY-4.0 · Unofficial community productivity layer, not affiliated with MiniMax.

---

## 0. What this is / why it exists

MiniMax's official `VIDEO_PROMPT_WRITING_GUIDE` is knowledge-dense but leaves the translation into usable prompts to you. This spec does three things:

1. Compresses the rules into **copy-paste templates** (one per mode, fill the blanks).
2. Ships **worked examples** (good / bad / before-after) so you write without stepping on landmines.
3. Ships a **prompt builder** (see `prompt-builder.html`) that outputs a copy-ready prompt from a form.

**Scope**: MiniMax H3 (Hailuo 3.0) text-to-video / image-to-video / first-last-frame / last-frame / reference-to-video.

**Sources**: MiniMax official repo `MiniMax-AI/MiniMax-H3` (`VIDEO_PROMPT_WRITING_GUIDE_base_en.md`); HuggingFace `MiniMaxAI/MiniMax-H3`; community practice (ambienceai / genie11 / rundiffusion). This file is a community re-organisation and does not replace the official guide.

---

## 1. Model facts (know before you write)

| Item | Value |
|---|---|
| Architecture | 33B dense single-stream Omni Transformer |
| Audio+Video | Native co-generation (video + audio sampled in one pass, not post-dubbed) |
| Framerate / Audio | 24fps / 32kHz stereo |
| Duration | 4–15 seconds |
| Resolution | 768p base; 2K needs official H3-Regenerate-2K (open weights lack Context-IR preprocessing) |
| Prompt language | **English** (official examples are English; Chinese runs but alignment drops) |

Key takeaway: audio must be **explicitly written** — the model will not "guess" what sound should exist.

---

## 2. The 3-field structure (all modes, top priority)

Final prompt = three fields joined with **a blank line between each**:

```
<integrated_multimodal_description>

<overall_soundscape>

<non_diegetic_music>
```

| Field | Role | How to write |
|---|---|---|
| `integrated_multimodal_description` | visuals + action + dialogue + camera, narrated in time order | the main body, goes first |
| `overall_soundscape` | 1–4 English sentences, total environment sound, physical action sound, non-verbal voice (breath, footsteps, fabric, cutlery) | **do not write dialogue lines** |
| `non_diegetic_music` | 1–3 sentences, describe instruments, tempo, timbre | **no abstract mood words** (no "emotional" / "epic" / "melancholic"); if none, write `N/A` |

**Why three separate fields**: collapsing into one block → audio track quality collapses, ambience missing. This is the highest-priority constraint.

---

## 3. Mode selector (pick right, then write)

| Mode | Input | When to use | Alignment line |
|---|---|---|---|
| **T2VA** | text only | no reference image, describe from scratch | none (straight to 3-field) |
| **I2VA** | 1 first-frame image | image-to-video, lock the opening | first-frame alignment (see §4) |
| **L2VA** | 1 last-frame image | image-to-video, lock the ending | last-frame alignment (see §4) |
| **FL2VA** | first + last 2 images | continuous interpolation between two frames, single shot | give both first + last alignment |
| **Ref2VA** | ≤9 img + ≤3 vid + ≤3 audio | multiple references define appearance, no frame alignment | rewrite `is referenced for the appearance of ...` |

**FL2VA key**: it tends to generate a **single-shot continuous interpolation** — do not cram multi-shot cuts inside FL2VA.
**Ref2VA key**: no frame alignment; use `is referenced for the appearance of ...` to describe which reference decides "whose appearance".

---

## 4. Frame-alignment lines — verbatim templates (I2VA / L2VA / FL2VA go at the very top)

I2VA / L2VA / FL2VA must put this line at the **very top of the prompt**, followed by a blank line, then the 3 fields:

```
For the target video, at 0.00 seconds into the target video, <Picture 1> (from [Shot 1]) is fully referenced.
```

Other modes:

| Mode | Alignment line (copy verbatim, replace `<Picture N>`) |
|---|---|
| I2VA first frame | `at 0.00 seconds into the target video, <Picture 1> (from [Shot 1]) is fully referenced.` |
| L2VA last frame | `at the last frame of the target video, <Picture 1> (from [Shot 1]) is fully referenced.` |
| FL2VA first+last | give `<Picture 1>` @0.00s AND `<Picture 2>` @last frame |

`<Picture 1>` = a short pointer to your first-frame image (e.g. `the woman in red coat`, `the cafe interior`).

Missing the alignment line → first frame unlocks, face swaps.

---

## 5. Image-to-video core rules (I2VA / L2VA / FL2VA)

1. **The first frame already defines appearance — the prompt only writes "what happens after".** Re-describing details already in the image wastes tokens and can cause conflict.
2. Only re-state identity features when you need to "lock" them (same face, same clothes, same light) — one sentence.
3. **Never contradict the first frame** (image is day but you write night, image is seated but you write standing → it breaks).
4. In a single 5s shot, schedule **2–3 action beats**; more becomes fast-forward mush.
5. Sound must be explicitly defined.
6. Negatives are not done via negative prompt — write what must "stay unchanged" in the positive.

---

## 6. Camera motion grammar

H3 dropped Hailuo 02's bracket-tag stacking (`[Zoom in][Pan left]`). **Brackets in H3 are only for `[Shot N]`**.

Write as: `motion type + amplitude + speed`, folded into a natural sentence.

- Amplitude: `small amplitude` / `large amplitude`
- Speed: `slow speed` / `fast speed`
- Type: Zoom in/out, Push in, Pull out, Pan left/right, Truck left/right, Tilt up/down, Pedestal up/down, Arc, Tracking shot, Static shot, Handheld shake, POV, Roll

Example: `The camera performs a slow-speed, small-amplitude push in toward her face.`

Use camera motion (not "cut") for distance or slight angle changes only.

---

## 7. Multi-shot timestamps

```
[Shot 1] (no timestamp, defaults to start at 0)
[Shot 2] At 00:03.500, the camera cuts to a close-up of ...
[Shot 3] At 00:07.200, ...
```

Timestamps must strictly increase, format `MM:SS.mmm`. Non-increasing → shot order scrambles.

---

## 8. Dialogue syntax

- Speaker tag: `(S1)` `(S2)` `(S1,S2)`
- Line wrap: `<d>[English] Are you serious right now?</d>`
  - bracket holds only the language tag; line is the original text, **not translated**
- Voiceover: `says in an off-screen voiceover ... while his lips remain completely closed`
- Density: ~**20 words dialogue / 15 seconds**, ~10 words per line. Over → speech-speed distortion

---

## 9. Length guide

| Scenario | Suggested length |
|---|---|
| single 5s I2VA | 80–150 words (description part) |
| multi-shot 10–15s | 200–350 words |
| soundscape | 1–4 sentences |
| music | 1–3 sentences or N/A |

---

## 10. Copy-paste templates (productivity core)

### 10.1 I2VA (image-to-video, most common)

```
For the target video, at 0.00 seconds into the target video, <Picture 1> (from [Shot 1]) is fully referenced.

[Shot 1] <one sentence locking identity & scene unchanged> Then <action beat 1>. <action beat 2>. <action beat 3>. The camera performs a <speed>-speed, <amplitude>-amplitude <motion type>. <ongoing light/material/grain description>.

<environment sound 1–4 sentences, incl. physical action sound & non-verbal voice>

<music 1–3 sentences, concrete instruments & tempo, or N/A>
```

### 10.2 T2VA (text-to-video)

```
<subject & scene in one sentence> <action beat 1>. <action beat 2>. <action beat 3>. The camera performs a <speed>-speed, <amplitude>-amplitude <motion type>.

<environment sound 1–4 sentences>

<music 1–3 sentences, or N/A>
```

### 10.3 FL2VA (first+last frame, single-shot interpolation)

```
For the target video, at 0.00 seconds into the target video, <Picture 1> (from [Shot 1]) is fully referenced.
At the last frame of the target video, <Picture 2> (from [Shot 1]) is fully referenced.

[Shot 1] <continuous action between first and last, no cuts> The camera performs a <speed>-speed, <amplitude>-amplitude <motion type>.

<environment sound 1–4 sentences>

<music 1–3 sentences, or N/A>
```

### 10.4 L2VA (last-frame alignment)

```
For the target video, at the last frame of the target video, <Picture 1> (from [Shot 1]) is fully referenced.

[Shot 1] <opening> <action beats evolving toward the last-frame state>. The camera performs a <speed>-speed, <amplitude>-amplitude <motion type>.

<environment sound 1–4 sentences>

<music 1–3 sentences, or N/A>
```

### 10.5 Ref2VA (multi-reference, no frame alignment)

```
<Picture 1> is referenced for the appearance of <subject A>. <Picture 2> is referenced for the appearance of <subject B>.

[Shot 1] <action beats>. The camera performs a <speed>-speed, <amplitude>-amplitude <motion type>.

<environment sound 1–4 sentences>

<music 1–3 sentences, or N/A>
```

---

## 11. Worked examples

### Example A: I2VA bad vs good

**Bad** (re-states first frame + no sound + too many actions + bracket tag)
```
A woman with long black hair standing in a red coat in a cafe. She walks, turns, laughs, drinks, looks at phone, opens door, smiles, leaves. [Zoom in]
```
Problems: re-states appearance, no soundscape/music, 8 actions in 5s, bracket tag.

**Good**
```
For the target video, at 0.00 seconds into the target video, the woman in the red coat (from [Shot 1]) is fully referenced.

[Shot 1] She remains seated by the window, then slowly brings the cup to her lips and takes a sip, sets it down, and glances toward the rain-streaked glass. The camera performs a slow-speed, small-amplitude push in toward her face. Warm afternoon light continues to fall across the table.

Soft cafe ambience with distant espresso machine hiss, the clink of a ceramic cup on saucer, and quiet rainfall against the window.

A lone nylon-string guitar plays a gentle, mid-tempo lo-fi pattern with brushed percussion.
```

### Example B: multi-shot timestamps

```
[Shot 1] A lone figure walks across an empty station platform.
[Shot 2] At 00:03.500, the camera cuts to a close-up of his weary eyes reflecting the flickering fluorescent light.
[Shot 3] At 00:07.200, a wide shot reveals the approaching train's headlights piercing the tunnel dark.

Echoing footsteps on tile, a distant train rumble growing louder, fluorescent hum.

A low sustained cello drone with a slow building timpani pulse.
```

---

## 12. Common mistakes

| Mistake | Consequence |
|---|---|
| 3 fields merged into one block | audio collapses, ambience missing |
| missing alignment line (I2VA/L2VA/FL2VA) | first frame unlocks, face swap |
| `[Zoom in]` bracket tag | H3 ignores it, treats as plain text |
| music writes "emotional / cinematic mood" | official ban on abstract mood words |
| 5+ actions in 5s | fast-forward, blurred frames |
| dialogue > 20 words / 15s | speech-speed distortion |
| description contradicts first frame |撕裂 / identity drift |
| multi-shot timestamps not increasing | shot order scrambles |
| Chinese prompt | alignment drops (official examples are English) |

---

## 13. Pre-flight checklist

- [ ] 3 fields separated, blank line between each
- [ ] mode picked; I2VA/L2VA/FL2VA has verbatim alignment line at top
- [ ] image-to-video does not re-state first-frame appearance (only "what happens after")
- [ ] no description contradicts the first frame
- [ ] single 5s shot has ≤ 3 action beats
- [ ] camera motion uses "type+amplitude+speed", no bracket tags
- [ ] soundscape written, contains no dialogue lines
- [ ] music writes concrete instruments/tempo, no abstract mood words (or N/A)
- [ ] dialogue ≤ 20 words / 15s, lines wrapped in `<d>[lang]... </d>`
- [ ] multi-shot timestamps strictly increasing `MM:SS.mmm`
- [ ] entire prompt in English

---

## 14. One-page cheat sheet

```
Structure: desc (blank) soundscape (blank) music
Alignment: I2VA first / L2VA last / FL2VA first+last / Ref2VA rewrite
Shot: [Shot N] brackets only; timestamp MM:SS.mmm increasing
Camera: type+amplitude(small/large)+speed(slow/fast), in a sentence
Dialogue: <d>[English] ...</d>, 20 words/15s
Music: instruments+tempo, ban emotional/epic/melancholic, else N/A
Length: 5s→80-150 words; 15s→200-350 words
Iron rules: 3 fields not merged / no first-frame conflict / sound explicit / English
```

---

## 15. Visual assets (图文素材 — for X / Reddit hooks)

Three ready-to-post vector cards (1080×1080) live in this repo, built to pull attention and drive repo visits. All CC-BY-4.0; repost freely with attribution.

| File | Use |
|---|---|
| `cheatsheet.svg` | One-page cheat sheet — the densest collection bait |
| `structure-contrast.svg` | Wrong vs right prompt wiring — concept hook |
| `before-after.svg` | Weak-prompt render vs structured-prompt render — proof hook (real frames embedded) |

Tip: browsers render SVG crisply; screenshot to PNG (or convert) before posting to X / Reddit. No AI-generated imagery is used — these are pure vector plus real renders, so text stays sharp.

---

## 16. License & attribution

- **License**: CC-BY-4.0. Free to copy, adapt, redistribute; attribution to source required.
- **Attribution**: This spec adapts MiniMax's official `MiniMax-AI/MiniMax-H3` video prompt guide and community practice. It is an unofficial community productivity layer.
- **Disclaimer**: Model capabilities evolve with versions; defer to the official latest guide.

---

### Companion tool

`prompt-builder.html`: a single-file interactive prompt builder. Pick mode → fill fields → get a copy-ready prompt live, with built-in validation (action-beat overflow, dialogue word count, music banned words). No network, no dependencies.
