# Cliptic — Design Spec

- **Date:** 2026-06-28
- **Status:** Design approved → pending implementation plan
- **Working name:** Cliptic (changeable)
- **Authors:** Karim, with Claude (Opus 4.8)

---

## 1. Summary

Cliptic is a **local-first, deterministic** desktop app (Tauri) for making **caption-first vertical short-form video** (Reels / TikTok / Shorts). You import a talking-head clip; it auto-transcribes **on-device** (no cloud, no API, free), produces **word-level timestamps**, and renders **research-tuned animated caption presets** that map to the audio **frame-accurately**. It also provides **zoom/punch motion presets** and a **"sticky-note + sound-effect" overlay system**. Export is a **visually-lossless master** plus a **ready-to-post MP4**. There are **no LLM calls at runtime** — all behavior is deterministic code.

The product's reason for existing is **timing correctness**: captions and animations that land on the audio, every time, with the polish of current top-tier short-form.

## 2. Goals / Non-goals

**Goals (v1):**
- Auto-caption a single vertical clip on-device with accurate word-level timing.
- 6 caption presets (below), switchable live, faithful to 2026 craft.
- Editable transcript: fix a misheard word without losing timing.
- Zoom/punch motion presets + a sticky-note image-with-SFX system, both timed to spoken words.
- A "restraint" rate-limiter so effects never feel amateurish (overuse is the #1 tell).
- Deterministic preview that equals the export, frame-for-frame.
- Export: ProRes 422 HQ master (audio passthrough) + high-bitrate social MP4.
- Respect platform safe zones and readability limits by default.

**Non-goals (v1):**
- Multi-track / multi-clip timeline editing beyond trimming one clip.
- Music library, AI b-roll generation, auto-reframing, green-screen.
- Collaboration, cloud sync, accounts, or any network feature.
- Mobile/web build (desktop Tauri only).
- Any runtime LLM/cloud API call.

## 3. Core principle — correctness via determinism

The entire system rests on one rule: **time is stored in milliseconds, never in frames.**

1. Whisper yields each word's `startMs`/`endMs`.
2. Those ms values are the **single source of truth**. We never pre-round them to frame indices — rounding compounds and drifts.
3. On render, every frame computes `ms = (frame / fps) * 1000` and asks *"which word/animation state is active at this ms?"*. State is **always derived from the clock**, never the reverse.
4. `drawFrame(timeline, frameNumber)` is a **pure function**: no `Date.now()`, no unseeded `Math.random()`, no external mutable state. Same inputs → identical pixels.
5. Therefore the **live preview and the export are the same code path** → guaranteed WYSIWYG and repeatability.

**Sync rules baked in (from research):**
- Captions **lead audio by ~120 ms** (reading lags listening).
- **Minimum word display ~150 ms** so short words don't flash.
- On silence/gaps beyond a threshold, **clear captions** (or extend last word) rather than freeze.
- Unalignable tokens (e.g. `"£13.60"`, `"2014."`) **interpolate** timing from neighbors.

**Milestone 0 is a walking skeleton** that *measures* sync end-to-end (transcribe → render one preset → export → verify captions hit the right frames) before any large build. If whisper.cpp's built-in word timing is too loose, we add a forced-alignment pass **then**.

## 4. Users & UX

Target user: a solo creator making talking-head Reels who wants captions that look pro without learning an NLE. **Simplicity is a hard requirement.**

**Three-step flow:**
1. **Import** a vertical clip.
2. **Auto-caption & pick a style** — transcript appears; choose a preset from a gallery of live thumbnails (like the brainstorming playground widget).
3. **Tweak & export** — fix any word, drop punch-ins / stickers on chosen words, export.

UI surfaces: a 9:16 preview, a single readable timeline (audio waveform + caption blocks + effect markers), a preset gallery, a transcript editor, an export panel. No flight-deck of tracks. CapCut-simple.

## 5. Architecture & data flow

```
┌─────────┐   ┌──────────────┐   ┌────────────────────┐   ┌──────────────────────┐
│ video   │──►│ extract audio │──►│ Whisper (Rust,     │──►│ words + ms timings   │
│ (import)│   │ (FFmpeg)      │   │ whisper.cpp, local)│   │                      │
└─────────┘   └──────────────┘   └────────────────────┘   └──────────┬───────────┘
                                                                       │ editable transcript
                                                                       ▼
        ┌──────────────────────────── Timeline (project state) ───────────────────────────┐
        │  clip · captions+timings · caption preset · effects (zoom/punch) · stickers+SFX  │
        └───────────────────────────────────────┬─────────────────────────────────────────┘
                                                 │
                                   drawFrame(timeline, frameN)   ◄── ONE pure function
                                          │                          │
                                   live preview (webview)      export: render frames
                                                                ─► FFmpeg sidecar
                                                                ─► ProRes master (audio copied)
                                                                ─► social MP4 (H.264 high-bitrate)
```

**Components:**
- **Rust core (Tauri backend):** file I/O, FFmpeg invocation (audio extract, encode/mux), `whisper-rs` transcription, frame piping on export.
- **Webview frontend (React/Vite/TS):** the editor UI + the `drawFrame` compositor (Canvas2D) for both preview and export frame generation.
- **FFmpeg sidecar:** bundled native binary; encode + mux only (no transcode of untouched streams).

## 6. Data model (conceptual)

```ts
Project { fps: number; width: number; height: number; clip: Clip; captionStyleId: string;
          captions: Caption[]; effects: Effect[]; stickers: Sticker[]; }

Clip    { src: string; durationMs: number; trimInMs: number; trimOutMs: number; }

// Whisper output, normalized (Remotion-style Caption model)
Caption { text: string; startMs: number; endMs: number; }          // one per word
Page    { text: string; startMs: number; durationMs: number;       // grouped for line presets
          tokens: { text: string; fromMs: number; toMs: number }[]; }

Effect  { type: 'kenBurns'|'punchPulse'|'punchHold'|'whip'|'shake';
          startMs: number; params: object; }                       // zoom/punch on the video layer

Sticker { src: string|IconRef; atWordIndex: number;                // image/sticker
          enterMs: number; dwellMs: number; exitMs: number;
          posX: number; posY: number; tiltDeg: number; sfx?: SfxCue; }

SfxCue  { src: string; peakOffsetMs: number; gainDb: number; }     // SFX synced to visual peak
```

Pages are produced by a `combineTokensWithinMs(captions, threshold)` segmenter: low threshold → word-by-word; high → full lines. All presets read the same `captions` + `pages`; they differ only in how they *draw* the active token.

## 7. Transcription pipeline (deterministic, local)

1. **Extract audio** from the clip via FFmpeg (to 16 kHz mono WAV for Whisper).
2. **Transcribe** with `whisper-rs` (whisper.cpp), **greedy decoding, temperature 0, no temperature fallback** → deterministic on a given machine/build. Model: ggml `base`/`small` by default (downloaded once, free); larger optional.
3. **Word timing** via whisper.cpp's built-in **DTW token-level timestamps** (`token_timestamps` / `dtw`). No extra model. If too loose (validated in M0), add a **wav2vec2 CTC forced-alignment** refinement pass (WhisperX-style) against the transcript.
4. **Normalize** to the `Caption` model (per-word `text/startMs/endMs`); tokens preserve leading spaces.
5. **Group** into `Page`s for line-based presets.
6. **Editable transcript:** user edits change `text` only; timings are preserved (re-align just the edited word if length changes materially).

## 8. Caption readability & placement defaults (from research)

| Rule | Default | Range / note |
|---|---|---|
| Reading speed | 3.5 words/sec | 3–4 wps; 17 CPS comfort, 20 ceiling |
| Chunk size | 2–4 words | held ~700 ms each |
| Line limits | ≤42 chars/line, ≤2 lines | hard cap |
| Lead | +120 ms ahead of audio | tightens the feel |
| Min word duration | 150 ms | avoid flashing |
| Font | bold sans, ~6.5% frame-width cap-height (~60–66pt @1080w) | white + black stroke |
| Case | mixed-case default | not ALL CAPS (uppercase reserved for specific presets) |
| Vertical position | ~58–70% down frame | lower-middle / center |
| Safe zones | TikTok: bottom 25% / right 13% / top 7%; Meta Reels: bottom 35% / sides 6% / top 14% | guard captions out of these |

## 9. Caption preset catalog (6 + decorator)

All presets render from the same word timings; they differ in granularity, highlight mechanic, and entrance.

1. **TikTok classic** — per line; white + 2px black stroke; 200 ms fade in; no highlight. The neutral baseline.
2. **Karaoke highlight** — line stays; active word recolors white→accent; no layout shift = **most sync-robust**; education/music.
3. **Word-by-word pop** — 1–3 words; newest springs in (`stiffness 280, damping 20, mass 1`, ~120–220 ms, overshoot ~1.12); accent on active word; the core talking-head look.
4. **Hormozi big bold** — 4–6 words / 2 lines; UPPERCASE heavy (Anton/Montserrat Black); 8–12px stroke; one keyword in yellow `#FFD23F`; block pops in. **Benchmark, not flagship** (now generic in 2026).
5. **Big active word** — exactly one giant centered word; UPPERCASE; **extruded hard-offset depth** (stacked offset shadow, no blur) or inverse accent box; spring overshoot 0.7→1.0(→1.1); auto-scales so the longest word fits ~90% safe width; hard-replace per word. The screenshot reference; highest-impact hook.
6. **Active-word pill** — line; an accent **pill glides** to the active word, text inverts to dark; the clean podcast alternative to Hormozi.

**Auto-emoji decorator** — optional layer over any preset; adds a relevant emoji on emphasized words (rate-limited).

Default accent is a **teal `#3DDC97`**, deliberately distinct from the generic Hormozi yellow. Per-niche packs: Educational (clean word-by-word) vs Hype/Finance/Fitness (bold-statement, beat-synced).

## 10. Motion / effect presets

Driven by the same ms clock; applied to the **video layer only** (captions stay anchored as an overlay).

- **`kenBurns`** — scale 1.0→1.08 over the shot (90–150f), `cubic-bezier(0.45,0,0.55,1)`, optional 4% pan, one direction per shot.
- **`punchPulse`** — snap 1.0→1.20 in ~4f ease-out `cubic-bezier(0.16,1,0.3,1)`, return to 1.0 over ~8f. Keyword/beat emphasis. The most reusable.
- **`punchHold`** — snap 1.0→1.20 in ~4f, hold until cut.
- **`whip` / `shake`** — fast directional slide / decaying jitter (3–6f) + motion blur; reserved for reveals/drops.

**Timing cheat-sheet (30fps; ×2 for 60fps) — implementation defaults:**
```
KEN_BURNS_FRAMES     = 120     # 90–150
KEN_BURNS_SCALE_END  = 1.08    # 1.06–1.10
PUNCH_SCALE          = 1.20    # 1.15 subtle … 1.30 hard cap (above = black borders)
PUNCH_SNAP_FRAMES    = 4       # 2–5, ease-out
PUNCH_RETURN_FRAMES  = 8       # 6–10
STICKER_ENTER_FRAMES = 8       # 6–12
STICKER_OVERSHOOT    = 1.20    # peak 1.1–1.3
STICKER_SPRING       = {stiffness:280, damping:20, mass:1}
STICKER_DWELL_FRAMES = 36      # 24–60 (0.8–2.0s)
STICKER_EXIT_FRAMES  = 6       # 4–8
STICKER_TILT_DEG     = 5       # ±3–8
OVERLAY_LEAD_FRAMES  = 4       # land 3–6f (100–200ms) BEFORE the word
SFX_PRE_PEAK_FRAMES  = 2       # start SFX 1–3f before the visual peak
SFX_GAIN_VS_VO_DB    = -4      # SFX a few dB under voice
VO_TARGET_LUFS       = -16     ; MIX_PEAK_DB = -6..-3
CUE_CADENCE_FRAMES   = 50      # visual change ~45–60f; floor 36f (never sustain <1.2s)
POP_BEZIER           = cubic-bezier(0.34, 1.56, 0.64, 1)
```
Springs are simulated once and sampled per-frame (a cubic-bezier can't reproduce a real spring); store as a `linear()`-style point table for deterministic sampling.

## 11. Sticky-note + SFX system (key feature)

A small image / sticker / "sticky note" that pops onto screen with a sound effect exactly when you say a word.

- **Entrance:** scale 0→1.2→1.0 over 6–12f (spring above), variants: rotate-in (−12°→settle), slide-from-behind-speaker, drop+micro-shake.
- **Timing:** land the visual **on or 1–2f before** the word onset (never lag). Dwell 24–60f. Exit 4–8f (faster than entrance).
- **SFX sync:** align the SFX's **loudest transient to the visual's overshoot-peak frame**; start the clip ~1–3f before the peak; mix ~4 dB **under** the voice.
- **Treatment:** soft separation, resting tilt ±3–8°, paper/sticker border, placed in the upper-third or side gutter **opposite the speaker's face** — never over mouth or caption band.

## 12. Restraint / rate-limiter

The loudest research finding: the amateur tell is **overuse**, not a missing effect. The engine enforces:
- No two cues within ~0.8–1.2s; visual change cadence ~45–60f, floor 36f.
- Don't punch every line; emoji/sticker accents are scarce relative to zooms.
- Warn in-UI if an SFX lands >3f after its visual, or if cadence is too dense.
The "pro look" target: **fast attack + ease-out landing + at most one small overshoot + restraint.**

## 13. Rendering & export

- **Compositor:** `drawFrame(timeline, frameN)` draws to Canvas2D in the webview — video layer (with zoom transform), then captions, then stickers, then emoji.
- **Source frames on export:** primary path **WebCodecs `VideoDecoder`** for exact frames (validated on macOS WKWebView in M0); fallback = FFmpeg (Rust) decodes source frames and pipes them to the compositor. Compositing stays in one place either way (preview == export).
- **Encode/mux (FFmpeg sidecar):**
  - **Master:** ProRes 422 HQ, **audio `-c:a copy` (passthrough, no re-encode)**. Visually lossless.
  - **Social:** H.264 high-bitrate (1080×1920, ~CRF 18 / 12–20 Mbps), AAC 256–320k. (The one place audio is re-encoded — convenience MP4 only.)
- Optional **seamless-loop mode** (match first/last frame + audio crossfade) for sub-15s clips.

## 14. Tech stack

| Layer | Choice |
|---|---|
| Shell | Tauri 2 (Rust core + system webview) |
| Frontend | React + Vite + TypeScript |
| Transcription | `whisper-rs` (whisper.cpp), Metal on macOS, greedy/temp-0 |
| Compositor | Canvas2D (`drawFrame`), simulated springs sampled per-frame |
| Export | bundled FFmpeg sidecar (encode + mux) |
| State | a single serializable `Project` object (undo/redo via snapshots) |

## 15. Milestones

- **M0 — Walking skeleton (spike).** 5s clip → extract audio → whisper word timings → render ONE caption preset via `drawFrame` → export → **measure** captions land on the right frames. Decide WebCodecs vs FFmpeg frame source; decide whether forced-alignment is needed. *Exit criterion: caption-to-audio drift < 1 frame across the clip.*
- **M1 — Transcription + transcript editor.** Import, audio extract, whisper integration, editable transcript with preserved timing, the preview shell.
- **M2 — Caption presets.** The 6 presets + page segmenter + accent system + safe-zone guards + readability defaults.
- **M3 — Effects + stickers.** Zoom/punch presets, sticky-note + SFX system, audio mixing, the restraint rate-limiter.
- **M4 — Export.** ProRes master (audio passthrough) + social MP4; deterministic frame render pipeline; progress UI.
- **M5 — UX polish.** Preset gallery thumbnails, onboarding, keyboard shortcuts, accessibility, reduced-motion, performance.

Each milestone = its own branch → PR → review → merge into `main`.

## 16. Risks & mitigations

| Risk | Mitigation |
|---|---|
| WebCodecs incomplete in macOS WKWebView → inexact export frames | M0 validates early; FFmpeg-decode fallback keeps one compositing path |
| whisper.cpp word timing too loose for "on-time" feel | M0 measures; add wav2vec2 CTC forced-alignment pass if drift > 1 frame |
| FFmpeg sidecar bundling / licensing / size | use a documented Tauri sidecar; LGPL build; fetch at setup, gitignored |
| Font licensing (Anton/Montserrat) | ship only OFL/free fonts; document sources |
| Determinism broken by float/timer leaks | lint for `Date.now`/`Math.random` in render; seed any noise; ms-only timing |
| Whisper output varies across hardware | acceptable (single-user machine repeatable); transcript is editable |

## 17. Open questions

- Final product name (working: Cliptic).
- Default whisper model size (speed vs accuracy) — decide in M1 with real clips.
- Bundle a small royalty-free SFX pack vs user-supplied only.

## 18. Research provenance

Built from three research sweeps (full notes + source URLs in [`docs/research`](../../research)):
- `research-1-craft-2026.md` — short-form craft, hooks, caption readability numbers, safe zones, pacing.
- `research-2-caption-styles.md` — caption style teardown (12 styles), the deterministic timing pipeline, the 6-preset shortlist.
- `research-3-effects-overlays.md` — zoom/punch, sticky-note + SFX specs, easing/spring numbers, the timing cheat-sheet.

Strongest sources include OpusClip (captions/presets/hooks/trends), Kreatli and behaviour.digital (safe zones), Remotion `@remotion/captions`, WhisperX, Submagic (Hormozi teardown), VEED/Captions.ai (style taxonomy), and Finchley (SFX sync). Caveat: percentage-lift claims come from vendor blogs (directional); the pixel safe-zones, subtitling reading-speed standards, and effect-timing numbers are the safest to hard-code.
