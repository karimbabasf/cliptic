# Cliptic

Caption-first vertical video editor for Reels, TikTok, and Shorts — **deterministic, local, and free.**

> Status: pre-alpha, in active development. Built design-first from a teardown of how high-performing 2026 short-form video is actually made (see [`docs/research`](docs/research)).

Cliptic auto-transcribes your talking-head clips entirely on-device (no cloud, no API keys, no LLM calls at runtime), pins every word to the audio with millisecond timing, and renders animated caption presets that actually land on the beat. Add punch-in zooms and "sticky-note + sound-effect" pop-ups, then export a clean master and a ready-to-post MP4.

## Why it's different

- **Correct by construction.** All timing lives in milliseconds, and a rendered frame is a pure function of `(timeline, frameNumber)`. The live preview and the exported file run the *same* render code, so what you see is exactly what you get — repeatably.
- **Local and free.** On-device Whisper (`whisper.cpp` via Rust) transcribes with word-level timestamps. Nothing is uploaded; nothing costs money.
- **Research-tuned presets.** Caption styles, timing, and effects come from current short-form research, not guesswork. We deliberately ship cleaner styles than the now-generic "Hormozi" look.
- **No quality crunch.** Export a visually-lossless ProRes master with your original audio passed through untouched, plus a high-bitrate MP4 for upload.

## Caption presets

TikTok classic · Karaoke highlight · Word-by-word pop · Hormozi big bold (benchmark) · Big active word · Active-word pill — plus a reusable auto-emoji decorator.

## Effects

Ken Burns push · Punch-pulse and punch-hold zoom · Sticky-note image pop with a synced sound effect · whip/shake transitions — all governed by a restraint rate-limiter (the number-one amateur tell is overuse, so the engine refuses to over-animate).

## How it works

```
video ─► extract audio ─► Whisper (Rust, local) ─► words + ms timings ─► editable transcript
      ─► Timeline (clips · captions · effects · stickers)
      ─► drawFrame(timeline, frameN) ──► live preview  +  FFmpeg export (ProRes master · social MP4)
```

## Stack

Tauri (Rust core) · React + Vite + TypeScript · `whisper-rs` (whisper.cpp, Metal-accelerated) · Canvas2D compositor · bundled FFmpeg sidecar.

## Roadmap

- **M0** — walking skeleton: prove transcribe → render → export sync on a 5-second clip
- **M1** — transcription + transcript editor
- **M2** — the six caption presets
- **M3** — zoom + sticky-note/SFX + restraint limiter
- **M4** — export (ProRes master + social MP4)
- **M5** — UX polish

Full design: [`docs/superpowers/specs/2026-06-28-cliptic-design.md`](docs/superpowers/specs/2026-06-28-cliptic-design.md).

## License

MIT © 2026 Karim. Not affiliated with any platform; preset styles are inspired by, not copied from, existing tools.
