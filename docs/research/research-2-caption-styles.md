# Caption Styles Teardown + Deterministic Timing Foundation (2025–2026)

Research for a caption-first, deterministic (no-cloud) video editor. Goal: rebuild market-leading animated caption styles as code presets, driven by frame-accurate word timing.

---

# PART A — CAPTION STYLE CATALOG

## How the market organizes styles (tool taxonomy)

- **CapCut** caption template families: Glow, Trending, Word, Frame, Aesthetic, Monoline, Multiline; plus presets: karaoke highlight, pop-by-word, bounce, typewriter, TikTok classic. Access: Captions → Templates.
  Source: https://www.capcut.com/resource/caption-template , https://www.capcut.com/resource/caption-style , https://www.capcut.com/resource/types-of-captions
- **Submagic** library categories: Trending, New, Emoji, Premium, Speakers. One-click creator presets inspired by Hormozi, MrBeast. Was first to ship a one-click "Hormozi style." 4 distinct Hormozi sub-styles.
  Source: https://www.submagic.co/blog/how-to-make-alex-hormozi-captions , https://max-productive.ai/ai-tools/submagic/
- **Captions.ai**: 30+ preset templates; per-word controls = underline / emphasize / **supersize** a word; "AI emphasis" auto-picks impactful words; "active word background transition" and separate "captions transition"; auto-emoji.
  Source: https://captions.ai/help/docs/captions/styles , https://captions.ai/features/add-captions-to-videos , https://captions.ai/help/guides/engagement/caption-styles-by-platform
- **OpusClip**: preset categories = "Bold Statement" (60–80pt, heavy weight, minimal animation, Montserrat Bold/Bebas/Impact, upper or center third, 5–8 word lines) and "Dynamic Word-by-Word / karaoke" (color shift white→yellow/accent preferred over underline/box; highlight 50–100 ms BEFORE the word is spoken). Brand kit saves font/colors/position/logo.
  Source: https://www.opus.pro/blog/best-caption-presets-styles-boost-retention , https://www.opus.pro/captions , https://www.opus.pro/blog/tiktok-caption-subtitle-best-practices
- **VEED** "dynamic subtitles" themes: Glide, Pulse, Handwritten, Whisper, Fusion. **Animation taxonomy (very useful as a preset list):** Box Highlight, Karaoke, Impact, Impact Pop, Reveal, Float In, Scale In, Drop In, Colour Highlight, Rotate & Flip, Rotate & Highlight, Stomp. Plus "AI Emphasis" auto-highlight.
  Source: https://www.veed.io/tools/add-subtitles/animated-subtitles , https://www.veed.io/tools/auto-subtitle-generator-online/dynamic-subtitles , https://support.veed.io/en/articles/12000003-how-to-use-dynamic-subtitles
- **Zubtitle**: template styling + extras = progress bar, supertitles (headline overlay), auto-resize per platform.
  Source: https://zubtitle.com/features/animated-captions
- **Vizard**: adjustable caption style, 16 languages.
  Source: https://vizard.ai/tools/auto-subtitle-generator-online/video-caption-generator

## Font → creator → color → animation map (from Blitzcut "exact fonts" teardown)
Source: https://blitzcutai.com/blog/best-caption-fonts-tiktok

| Font | Niche / creator | Case | Color/highlight | Outline | Animation |
|---|---|---|---|---|---|
| **Montserrat Bold/Black/ExtraBold** | Business/education = the "Hormozi style"; default for CapCut/Submagic/Opus/Riverside/Flixier Hormozi presets | UPPERCASE | white + single keyword **yellow #f7c204** (also red, green) | thick black outline | word-by-word pop |
| **The Bold Font** (free) | Closest match to what Hormozi actually used | UPPERCASE | white, yellow keyword | thick black | word-by-word pop |
| **Anton** (free Google) | Hormozi alternative; condensed, fits more text | UPPERCASE | white, yellow keyword | yellow stroke (new Hormozi) | pop-in bottom→top |
| **Bebas Neue** | Fitness/motivation/sports/hype | ALL CAPS (condensed) | bright (yellow, red) | — | fast bounce |
| **Poppins** | Lifestyle/fashion/beauty/travel | mixed | white + subtle shadow | none | fade, no bounce |
| **Proxima Nova / SF Pro** | Tech/startup/SaaS | mixed | white | thin black | minimal/none |
| **Roboto** | News/explainer/data | mixed | white on dark | — | minimal |
| **Inter** | Minimalist/art/design | mixed | white | minimal/none | simple fade |
| **Impact** | Classic meme/bold statement | UPPERCASE | white | black | minimal |

Most-common highlight hex: **yellow #f7c204**; Hormozi palette also cites **green #02fb23** and white. (Hormozi exact: bold geometric sans ≈ Montserrat Black, all-caps, thick black stroke; on 1080×1920 stroke ≈ **8–12 px**; covers ~10–15% screen height, lower-middle.)
Source (Hormozi specifics): https://blitzcutai.com/blog/best-caption-fonts-tiktok , aggregated WebSearch on Hormozi style.

---

## THE DISTINCT STYLES (rebuild specs)

### 1. Karaoke Highlight (color-fill / progressive color)
- **Granularity:** per-line/phrase shown; highlight advances **per word**.
- **Highlight mechanic:** color swap of the current word (base white → highlight yellow/accent). Sometimes a left-to-right fill of the line. No layout change → most stable.
- **Entrance:** whole line fades/slides in once (~150–250 ms); per-word change is just the color crossfade (~60–120 ms).
- **Type:** heavy sans (Montserrat/Poppins), often Title or UPPERCASE, thin–medium black outline, soft drop shadow.
- **Color logic:** base white, active = accent (#f7c204). Spoken-past words may stay accent or revert to white.
- **Layout:** 1–2 lines, 3–6 words, lower-middle third.
- **Niche:** music/lyrics, language learning, general "clean" educational. (OpusClip notes color-shift karaoke = highest retention for education; ~70% of top educational creators use a word-by-word variant.)

### 2. Word-by-Word Pop (scale/spring reveal)
- **Granularity:** per-word reveal — words appear one at a time.
- **Highlight mechanic:** the act of appearing IS the emphasis; current word often also accent-colored.
- **Entrance:** per word scale 0.6–0.8 → 1.0 with spring overshoot (~1.1). Snappy TikTok feel ≈ **spring damping 10, mass 0.5** (Remotion-tuned). Duration ≈ 120–220 ms (≈ 4–7 frames @30fps).
- **Type:** heavy sans, UPPERCASE common, thick outline.
- **Layout:** 1–3 words on screen, centered.
- **Niche:** TikTok talking-head, hooks, fast educational.

### 3. Hormozi / Big Bold (keyword highlight on all-caps block) — the benchmark
- **Granularity:** phrase block (max **4–6 words across 2 lines**, per Hormozi's editor), with **one keyword** colored or enlarged per block.
- **Highlight mechanic:** keyword **color change** (yellow #f7c204 / green #02fb23) AND/OR keyword **scale-up** vs the rest. Submagic's documented method: split the caption exactly at the keyword time and enlarge the keyword in the second half.
- **Entrance:** OLD Hormozi = TheBoldFont, UPPERCASE, drop shadow, **NO stroke**, words appear in sync. NEW Hormozi = **Anton**, UPPERCASE, slight shadow, **yellow stroke**, **pop-in from bottom→top**.
- **Type:** Montserrat Black / The Bold Font / Anton, UPPERCASE, thick black outline 8–12 px @1080w, drop shadow.
- **Color logic:** base white; keyword accent. Optionally per-keyword size bump (~1.15–1.4×).
- **Layout:** large, lower-middle, ~10–15% screen height, 2 lines.
- **Niche:** business/education/motivation. Documented **+15% engagement** lift; every major tool ships a "Hormozi preset."
- Source: https://www.submagic.co/blog/how-to-make-alex-hormozi-captions

### 4. BIG ACTIVE WORD (single huge word, inverse/contrast, depth) — special focus
The "one giant word centered over a talking head" look (e.g. "CAPTIONS" rendered huge). Mechanics to rebuild:
- **Granularity:** strictly **one word on screen at a time** (occasionally 1 word + a small line beneath).
- **Highlight mechanic:** the single word IS the focus. Emphasis via (a) **inverse/contrast color** — accent fill or accent box behind the word with text knocked out, or white-on-accent vs accent-on-white alternating; (b) **scale/depth** — large scale plus a **layered look**: a slightly offset duplicate behind (hard drop shadow at e.g. 6–10 px offset, or a second copy in accent color) giving a 3D/sticker depth.
- **Entrance per word:** scale-in with spring overshoot (0.7→1.0, overshoot ~1.08–1.15) + optional tiny rotation (±2–4°) or a vertical "stomp" (drop in + settle). Duration ~150–250 ms. Each new word replaces the previous (hard cut or quick crossfade ≤80 ms).
- **Active-word mark:** because only one word shows, "active" = "the one rendered." If a base line is also shown, the big word is the current token pulled out and enlarged 2–4× over it.
- **Type:** maximal heavy display — Anton / Bebas Neue / Montserrat Black / The Bold Font / Impact, UPPERCASE, thick outline, **hard offset shadow for depth** (not soft blur). Optional dual-layer: front white + back accent.
- **Color logic:** high contrast — white word + black stroke + accent shadow OR accent word + black stroke; "inverse" variant = accent-filled rounded box with white knocked-out text.
- **Layout:** dead-center (or center-upper), single word fills 40–70% of frame width, max ~1 line. Auto-scale font so the longest word fits a max width (e.g. 90% safe width).
- **Niche:** high-energy hooks, MrBeast-adjacent, motivational, meme/punchline emphasis, "stomp/impact" edits.
- Built from: VEED "Impact / Impact Pop / Stomp" + Captions.ai "supersize word" + Hormozi keyword-enlarge mechanic + Bebas "fast bounce."

### 5. Active-Word Box / Pill (background highlight)
- **Granularity:** per-word reveal or per-line with moving box.
- **Highlight mechanic:** a **filled rounded rectangle (pill)** behind the current word; box color = accent, text often inverts to dark/white for contrast. Box can animate width to wrap the word (spring) or hard-swap per word.
- **Entrance:** word fades/pops in; box scale-in (~120 ms) and translates to follow the active word.
- **Type:** medium-heavy sans, often Title case, minimal outline (box gives the contrast).
- **Color logic:** base white/light; active word text dark on accent pill (inverse trick).
- **Layout:** 1 line, 3–5 words; box hugs one word.
- **Niche:** "clean creator," podcasts, talking-head where a softer look than Hormozi is wanted. (VEED "Box Highlight," Captions.ai "active word background transition.")

### 6. TikTok Classic (native style)
- **Granularity:** per-line (sentence chunked into short lines), no per-word animation by default.
- **Highlight mechanic:** none (or optional active-word color).
- **Entrance:** line appears (cut or quick fade).
- **Type:** TikTok default = **white font, black stroke**, Proxima-Nova-like (TikTok's "Classic" font), Title/sentence case.
- **Color logic:** white + black outline; max readability.
- **Layout:** lower-middle, 1–2 lines, centered.
- **Niche:** default/native, broad. Strong baseline contrast (white + black stroke recommended universally).
- Source: https://www.opus.pro/blog/tiktok-caption-subtitle-best-practices

### 7. One-Line Clean / Minimalist
- **Granularity:** per-line.
- **Highlight mechanic:** none or subtle weight/opacity shift.
- **Entrance:** simple fade in/out (~150–250 ms), no bounce.
- **Type:** Inter / Poppins / SF Pro, mixed case, minimal or no outline, subtle shadow.
- **Color logic:** white (or brand), low chroma.
- **Layout:** single line, small–medium (~8–10% height), often lower third.
- **Niche:** premium/brand, design/art, corporate, documentary.

### 8. Typewriter
- **Granularity:** per-character reveal (within a line), timed by word boundaries.
- **Highlight mechanic:** characters appear sequentially; optional blinking caret.
- **Entrance:** char-by-char, ~20–40 ms/char (or interpolate chars across the word's start→end).
- **Type:** mono or heavy sans; UPPERCASE optional.
- **Layout:** 1 line.
- **Niche:** storytelling, "texting"/chat vibe, suspense hooks.

### 9. Bounce / Stomp (impact entrance)
- **Granularity:** per-word or per-line.
- **Highlight mechanic:** exaggerated entrance is the emphasis; often paired with accent color on key words.
- **Entrance:** drop-in + overshoot + settle ("stomp"), or strong bounce (scale 0.5→1.15→1.0). Often beat-synced. ~200–300 ms.
- **Type:** Bebas Neue / Anton / Impact, ALL CAPS.
- **Color logic:** bright (yellow/red), thick outline.
- **Layout:** centered, 1–3 words.
- **Niche:** fitness, sports, hype, gaming, beat-synced edits. (VEED "Stomp," Bebas "fast bounce.")

### 10. Reveal / Wipe / Mask
- **Granularity:** per-word or per-line.
- **Highlight mechanic:** word revealed via a mask wipe (left→right) or slide-from-behind a bar.
- **Entrance:** clip-path / mask animation ~150–250 ms; can pair with a moving accent underline.
- **Type:** heavy sans; UPPERCASE common.
- **Niche:** sleek/promo, brand intros, lower-thirds. (VEED "Reveal," "Float In.")

### 11. Gradient / Neon Glow
- **Granularity:** per-line or per-word.
- **Highlight mechanic:** gradient fill or glow on active word (text-shadow glow, accent gradient).
- **Entrance:** fade/scale with glow pulse.
- **Type:** heavy sans; saturated gradient or neon accent; outline optional.
- **Niche:** music, nightlife, gaming, aesthetic. (Submagic "Neon Glow," CapCut "Glow.")

### 12. Emoji-Augmented (overlay variant, not a base style)
- Any of the above + auto-injected emoji on emphasized words (Captions.ai/Submagic auto-emoji). Treat as a **decorator layer**, not a separate timing model.

---

# PART B — TECHNICAL TIMING FOUNDATION (deterministic, no cloud)

## B1. Getting per-WORD start/end timestamps

### Whisper word timestamps via cross-attention DTW
- Vanilla Whisper (and OpenAI's `word_timestamps=True`) infers **token-level** timestamps from the decoder **cross-attention** weights, aligned with **Dynamic Time Warping (DTW)** to find word boundaries. It's *unsupervised* (no extra model) but **less accurate** than external phoneme alignment and prone to drift, especially on disfluent/verbatim speech.
  Source: https://arxiv.org/html/2408.16589v1 (CrisperWhisper) , https://arxiv.org/html/2505.15646v1
- **whisper-timestamped** (linto-ai): adds word-level timestamps + confidence to OpenAI Whisper using **DTW on cross-attention**, *no extra model/inference*. Cross-platform (PyTorch). Good when you want self-contained Whisper without a separate aligner.
  Source: https://github.com/linto-ai/whisper-timestamped
- **whisper.cpp WASM** (ggml): runs fully **in the browser**, audio never leaves the machine; **Greedy sampling only** in the WASM example → deterministic. Best fit for a no-cloud, in-browser editor.
  Source: https://github.com/ggml-org/whisper.cpp/tree/master/examples/whisper.wasm
- **transformers.js**: Hugging Face JS, runs Whisper via WASM/WebGPU in-browser; supports `word_timestamps`; greedy/temperature-0 = deterministic.
  Source: https://www.assemblyai.com/blog/offline-speech-recognition-whisper-browser-node-js
- **faster-whisper** (CTranslate2): fast CPU/GPU Whisper, `word_timestamps=True`, `beam_size` configurable; backend used by WhisperX. Deterministic with greedy + temperature 0 + disabled temperature fallback.
- **CrisperWhisper / faster_CrisperWhisper**: retrains attention with a timestamp loss → markedly better DTW word timing on verbatim speech.
  Source: https://huggingface.co/nyrahealth/faster_CrisperWhisper

### Forced phoneme alignment (more accurate word/char timing)
- **WhisperX** pipeline (recommended balance of accuracy + determinism):
  1. **VAD** (pyannote, or silero-vad) segments speech → cuts hallucination, enables batching with no WER loss.
  2. **Batched faster-whisper** transcription (`--without_timestamps True`, 1 forward pass/sample) → 70× realtime w/ large-v2.
  3. **wav2vec2 phoneme CTC forced alignment** maps transcript to audio at the frame level → **word-level (and optional character-level)** timestamps. API: `whisperx.load_align_model(language_code=...)` then `whisperx.align(segments, model_a, metadata, audio, device, return_char_alignments=False)`.
  - Default align model e.g. `WAV2VEC2_ASR_LARGE_LV60K_960H`; bigger align model not very helpful. Language-specific wav2vec2 model required.
  - Borrows the PyTorch torchaudio forced-alignment routine.
  - **Gotcha:** tokens lacking chars in the align dictionary (e.g. `"2014."`, `"£13.60"`) **cannot be aligned → no timing**; must fall back to interpolation.
  - Accuracy caveat: phoneme alignment beats cross-attention DTW, but vs **Montreal Forced Aligner**, WhisperX scored ~**52.7% word acc @20ms tolerance** on TIMIT — MFA higher. Good enough at ~50–80 ms tolerance for captions; not phonetics-grade.
  Source: https://github.com/m-bain/whisperX , https://www.robots.ox.ac.uk/~vgg/publications/2023/Bain23/bain23.pdf , https://deepwiki.com/m-bain/whisperX/3.3-forced-alignment-system , https://github.com/m-bain/whisperX/issues/1247
- **Forced alignment from a KNOWN transcript** (deterministic alternative; great when script is authored):
  - **Montreal Forced Aligner (MFA)** — HMM-GMM (Kaldi), word + phoneme alignment, gold-standard accuracy, fully offline & deterministic. Needs pronunciation dictionary + acoustic model per language. Heavier to integrate.
  - **gentle** — Kaldi-based, robust to transcript mismatches, returns word timings; offline.
  - **CTC aligners** (wav2vec2 / torchaudio `forced_align`, Charsiu text-independent) — modern, deterministic given fixed weights.
  Source: https://arxiv.org/html/2406.19363v1 , https://arxiv.org/pdf/1811.05553

### Determinism summary
- Greedy decoding + **temperature 0** + **disable temperature fallback** → same output every run. (Whisper docs: temperature 0 = always same result.)
- Forced alignment (CTC/MFA/gentle) is deterministic given fixed model weights and input.
- Cross-attention DTW is deterministic per run but more error-prone in placement.
- **Most deterministic + accurate for a no-cloud editor:** known/edited transcript → CTC/wav2vec2 forced alignment (or MFA). If transcribing from scratch in-browser: whisper.cpp WASM (greedy) for tokens, then optional wav2vec2 CTC align for tightening.

## B2. Rendering frame-accurately (Remotion model)

### Data model: Caption → pages → tokens
- `@remotion/captions` `Caption` type (v4.0.216+):
  ```ts
  type Caption = { text: string; startMs: number; endMs: number; timestampMs: number | null; confidence: number | null }
  ```
  Source: https://www.remotion.dev/docs/captions/caption
- `createTikTokStyleCaptions({ captions, combineTokensWithinMilliseconds })` → `{ pages }`.
  - `combineTokensWithinMilliseconds`: words closer than this merge onto one page. **High = many words/page; low = word-by-word.**
  - **Whitespace-sensitive:** each token's `text` must include its **leading space** (spaces are the delimiter) or everything merges.
  - Returned `TikTokPage`:
    ```ts
    type TikTokPage = {
      text: string;
      startMs: number;
      durationMs: number;        // from v4.0.261
      tokens: { text: string; fromMs: number; toMs: number }[];  // absolute ms, for per-word animation
    }
    ```
  Source: https://www.remotion.dev/docs/captions/create-tiktok-style-captions , https://www.remotion.dev/docs/captions/api

### frame → time → active word
- Remotion: `frame = useCurrentFrame()`, `{fps} = useVideoConfig()`. Convert: `ms = (frame / fps) * 1000`.
- A token is **active** when `token.fromMs <= ms < token.toMs`.
- Per-word entrance: `framesSinceStart = frame - (token.fromMs/1000)*fps`; drive `spring({frame: framesSinceStart, fps, config:{damping:10, mass:0.5}})` for the snappy TikTok scale (0.7→1.0, slight overshoot); `interpolate()` for color/opacity.
  Source: https://www.remotion.dev/docs/spring , https://www.remotion.dev/docs/interpolate , https://crepal.ai/blog/aivideo/blog-how-to-create-tiktok-style-captions-remotion/ , https://github.com/orgs/remotion-dev/discussions/356

## B3. Sync gotchas (the loose ends)

1. **Frame vs ms rounding.** Render is frame-quantized; always derive active word from `ms = frame/fps*1000` and compare to token ms. Don't pre-round token times to frames (compounds error). At 30 fps a frame = 33.3 ms, so up to ~17 ms quantization either way — fine for captions.
2. **Lead-in / lead-out padding.** Highlight a word **~50–100 ms BEFORE** it's spoken (reading lags listening) — OpusClip's documented timing trick. Add small lead-out so the last word lingers.
3. **Minimum word display duration.** Very short words (<~120 ms) flash illegibly — clamp each token to a floor (e.g. ≥150–200 ms), borrowing time or holding past `toMs`.
4. **Gaps / silence.** Optionally extend a caption to the next start (CapCut/Netflix "close gaps" within a threshold) or **clear captions during long silence** and extend the last caption before silence. Don't leave a stale word frozen for seconds.
5. **Grouping into readable pages/lines.** Cap words per line (Hormozi: 4–6 words / 2 lines; TikTok: short lines). Use `combineTokensWithinMilliseconds`, max-chars-per-line, and prefer breaking at sentence/clause boundaries. Keep a sentence in one segment when it fits.
6. **Unalignable tokens.** Numbers/symbols (`"2014."`, `"£13.60"`) get **no timing** from CTC aligners → interpolate their ms from neighbors so they don't collapse to 0 duration.
7. **Frame-rate mismatch.** Caption timings are absolute ms; if project fps ≠ source fps the mapping still holds via ms — but never assume "1 caption = N frames" at a fixed fps. Common real-world bug: inconsistent fps between video and caption track, plus platform re-compression shifting A/V.
   Source: https://support.syncwords.com/hc/en-us/articles/200879287-Advanced-Caption-Output-Settings , https://help.captions.ai/docs/captions/adjust-timing
8. **Safe areas (9:16).** Keep captions clear of TikTok/Reels/Shorts UI (right rail + bottom caption bar). Lower-middle/center is the documented sweet spot; budget ~10–15% height.

---

## Recommended deterministic pipeline (synthesis)

1. **Transcript**: edited/known script if available; else transcribe in-browser with **whisper.cpp WASM** (greedy, temp 0) or **faster-whisper** (greedy, temp 0, no fallback) → deterministic.
2. **Word timing**: run **wav2vec2 CTC forced alignment** (WhisperX `align`, or torchaudio `forced_align` / MFA for gold) against the transcript → per-word `startMs/endMs`. Interpolate any unalignable numeric/symbol tokens from neighbors.
3. **Normalize to the Caption model** (`text/startMs/endMs/timestampMs`), tokens carry **leading spaces**.
4. **Group** into pages via a `createTikTokStyleCaptions`-style segmenter (`combineTokensWithinMilliseconds` + max chars/line + sentence-boundary preference). Clamp min word duration; close small gaps; clear on long silence.
5. **Render** frame-accurately: `ms = frame/fps*1000`; active token where `fromMs≤ms<toMs`; per-word spring (damping 10, mass 0.5) + color/opacity interpolate; apply lead-in ~70 ms.
6. Style = a **preset descriptor** (font, weight, case, fill, highlightColor, stroke px, shadow, box on/off, entrance type+duration, words-per-page, position, max width) consumed by one renderer.

## 6 presets to build first
1. **TikTok Classic** — white + black stroke, sentence-chunked lines, fade in. The safe, universal baseline; everything else is a delta from it.
2. **Karaoke Highlight** — phrase on screen, current word recolors white→#f7c204; no layout shift = most robust sync; best for education/music.
3. **Word-by-Word Pop** — one word at a time, spring scale-in (damping 10/mass 0.5), accent active word; the core TikTok talking-head look.
4. **Hormozi Big Bold** — Montserrat Black/Anton, UPPERCASE, 4–6 words/2 lines, thick black stroke 8–12px @1080w, one keyword in yellow (or enlarged); the market benchmark, +15% engagement.
5. **BIG ACTIVE WORD** — single giant centered word, hard offset shadow / dual-layer depth, optional inverse accent box, spring overshoot + stomp; the highest-impact hook style; auto-scale to fit max width.
6. **Active-Word Box/Pill** — rounded accent pill behind the current word with inverse text, box translates per word; the "clean creator/podcast" alternative to Hormozi.

(Decorator layer to add after: **auto-emoji on emphasized words**, reusable across all six.)
