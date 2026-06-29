# Research 3 — Non-Caption Motion / Effects & Overlays (2026 Reels/TikTok/Shorts)

Goal: rebuild the "pro" feel of modern short-form motion as deterministic code presets. All timings normalized to **frames @ 30fps** (and 60fps where it matters). Magnitudes given as scale multipliers, easing as cubic-bezier or spring (stiffness/damping/mass), sync as ±frames.

Conversion: 1 frame @30fps = 33.3ms. 1 frame @60fps = 16.7ms. To convert a duration in seconds to frames: `frames = seconds * fps`.

---

## 1. ZOOM / PUNCH-IN

### 1a. Slow continuous push-in ("Ken Burns")
The instructional/cinematic baseline: an almost-imperceptible drift that adds life to a static or talking-head shot without the viewer noticing the move.

- **Scale range:** start 1.00 → end **1.06–1.10** (6–10% total). Cloudinary/CapCut tutorials cite 8–15% as the "push-in that feels instructional rather than disorienting." Keep under ~1.10 for talking heads; up to ~1.20 only for hero stills you want to feel dramatic.
- **Duration:** spans the whole shot. Practical numbers from FCP/Cloudinary: cinematic 7–10s, energetic 4–5s, **social-media sweet spot 3–5s**. At 30fps that's **90–150 frames** for a typical social push.
- **Easing:** FCP auto-smooths Ken Burns so it "accelerates slowly at the start and decelerates slowly at the end" → a symmetric **ease-in-out**. For code: `cubic-bezier(0.45, 0, 0.55, 1)` (gentle in-out) OR purely linear is acceptable because the move is so small. Most important: NO hard stop — always decelerate into the end frame.
- **Pan option:** combine the scale with a small position drift (e.g. shift the center by 3–6% of frame width over the same span) for the true Ken Burns "pan + zoom." Direction should vary shot-to-shot.
- **Rule:** one direction per shot; never reverse mid-shot (reads robotic). Vary direction/intensity slightly across consecutive shots.

Sources: cloudinary-ken-burns, submagic-capcut-zoom, FCP pan-and-zoom (Apple).

### 1b. Snap / punch-in zoom (emphasis on a keyword or beat)
The aggressive "camera lunges in" on a word/beat. This is the workhorse emphasis move.

- **Scale jump magnitude:** +10% to +30%. CapCut tutorials: 100% → **110–130%**. Common defaults: punch to **1.15** (subtle), **1.20** (standard), **1.30** (hard, max before black borders appear on a 1.0-fit clip). HARD CAP: keep scale **below 130%** or over-zoom exposes the frame edge / black borders.
- **Duration of the snap:** fast — **2–5 frames @30fps** (≈67–167ms) for the punch itself. The whole point is that it's near-instant. (4 frames @30fps ≈ 8 frames @60fps.)
- **Easing:** **ease-OUT** (fast start, decelerate into the held scale) so it "lands." `cubic-bezier(0.16, 1, 0.3, 1)` (a strong ease-out / "easeOutExpo"-ish) or `cubic-bezier(0.22, 1, 0.36, 1)` (easeOutQuint).
- **Hold vs return — two patterns:**
  1. **Hold (sustained punch):** snap to 1.20 and STAY there for the rest of the sentence/clip; cut out of it on the next edit. Use when the new scale becomes the shot.
  2. **Pulse / return (accent):** snap to 1.12–1.20 then ease back to 1.0 over **6–10 frames**. Use to "tap" a single word without committing to a tighter frame. This is the most reusable preset.
- **Sync:** the punch should hit ON the stressed syllable (frame-aligned to the word's start, or 1 frame early). On a music edit, align the keyframe to the beat/drop.

Sources: submagic-capcut-zoom, poindeo-capcut-zoom, capcut-zoom-shake-2025.

### 1c. Shake / whip / bounce zooms
- **Zoom + shake (impact):** on the punch frame, add 2–4 tiny position keyframes that jitter the frame a few px in alternating directions over 3–6 frames, then settle. CapCut guide: if it "looks too shaky → reduce position shift or add motion blur"; if it "feels robotic → vary directions and intensity." Add **motion blur** to sell it. Magnitude: a few % of frame width, decaying. Use for beat drops / reveal hits / "wait WHAT" moments — sparingly.
- **Whip / directional zoom:** combine a fast scale-up with a horizontal/vertical position slide + heavy motion blur to fake a whip-pan transition between two clips. ~4–6 frames.
- **Bounce zoom:** the punch-in but with overshoot — scale shoots past target then settles (see §4 overshoot). E.g. 1.0 → 1.25 → settle 1.18. Reads "springy/fun" vs the clean ease-out which reads "clean/serious."
- **Don't overdo it:** a zoom on every sentence stops registering as emphasis. Reserve the punch for genuine keywords/beats; the slow push is the default ambient motion. (Same overuse warning the b-roll sources give: below a certain cadence the brain treats it as noise and tunes out.)

Sources: capcut-zoom-shake-2025, submagic-capcut-zoom.

---

## 2. "STICKY NOTE" / IMAGE POP-UP WITH SOUND EFFECT (the key request)

The meme/educational pattern: a small image / sticker / "sticky note" snaps onto screen with a sound exactly when the creator says a word.

### 2a. Entrance animation
The signature is **scale-from-0 with overshoot** ("pop"):
- **Canonical keyframe shape** (straight from creators / GSAP thread):
  - `scale: 0` at 0%
  - `scale: 1.2` at ~80% (the overshoot peak)
  - `scale: 1.0` at 100% (settle)
- **Duration of entrance:** snappy — **6–12 frames @30fps** (≈200–400ms). Tighter (6–8f) reads punchy/meme; looser (10–12f) reads softer/educational.
- **Overshoot amount:** peak **1.1–1.3×** final size (1.2 is the default). For a sticker, +15–25% overshoot feels lively without looking broken.
- **Spring equivalent (preferred for code):** a snappy spring with one small overshoot — stiffness ~260–320, damping ~18–22, mass 1 (Framer-Motion-style), or Remotion `spring({config:{stiffness:200, damping:14}})`. SwiftUI named equivalent: `.snappy` (duration 0.5, small bounce) or `.spring(duration:0.4, bounce:0.25)`.
- **Variants:**
  - **Rotate-in:** add a rotation that springs back — e.g. start rotation −8° to −15°, overshoot past 0, settle at the resting tilt (see treatment below). Apple "Overshoot behavior" applied to Rotation = rotate then spring back.
  - **Slide-in:** translate from off-screen / from behind the speaker, same overshoot envelope, ~8–12 frames.
  - **Drop + micro-shake:** the CapCut "object pops up with a little shake" — pop in, then 2–3px jitter for 3–4 frames, settle. Pairs with a "boing."

### 2b. Timing to the spoken word
- **Lead:** the visual should land **ON or 1–2 frames BEFORE** the spoken word's onset. Reading/perception is faster than the audio; captions are recommended to lead audio by **100–200ms (~3–6 frames @30fps)**, and the same lead reads well for a pop-up so the image is "already there" as the word hits. Never lag behind the word (feels late/amateur); >300ms early feels disconnected.
- **Dwell (on-screen hold):** keep it up long enough to read — roughly **0.8–2.0s (24–60 frames)**. Match the duration the creator is still talking about it. Educational call-outs lean longer; meme gags can be as short as ~0.7s.
- **Exit:** quick — **4–8 frames**. Either scale-back-to-0 (ease-in, the reverse of entrance), a pop-out, or a slide-off. Exit should be faster than entrance (don't linger on the way out). Often paired with a softer "whoosh" or no SFX.

### 2c. Sound-effect conventions
- **Common SFX by feel:**
  - **Pop / "text pop"** — the default for a small image/sticker/sticky note appearing. Crisp, short.
  - **Whoosh / swoosh** — movement/slide-in or transition; "instantly grabs attention, signifies movement."
  - **Ding / chime** — a correct/positive reveal, a checkmark, a stat.
  - **Boing** — bouncy/comedic pop (pairs with overshoot/shake).
  - **Ka-ching** — money/price reveal.
  - **Riser/impact** — building to a big reveal (riser leads in, impact lands).
- **Sync of SFX to the visual:** align the **attack/transient (the loudest peak) of the SFX waveform to the visual's impact frame** — i.e. the moment scale hits its overshoot peak, NOT scale-start. Because most SFX have a tiny pre-attack, the audio clip usually starts **1–3 frames BEFORE** the peak so the transient lands on the peak. Pro guidance: align audio peaks to key visual moments frame-by-frame; aim to be accurate **within 100–200ms**, ideally within ½–¼ frame for paired transients to avoid "flam" (a sloppy double-hit). Practical preset: **SFX start = visual entrance start, minus ~1 frame**, so transient ≈ overshoot peak.
- **Loudness relative to voice:** **Dialogue/VO is the anchor**; everything else sits relative to it. Set a reference level for VO, mix SFX under it. Concrete: normalize individual clips toward a consistent target (e.g. ~**-12dB** peak), apply gentle **2:1–4:1** compression on the VO so it stays on top, keep final mix peaking between **-6dB and -3dB** for headroom. VO integrated loudness commonly **~-16 LUFS** (dialogue), platform delivery often **~-14 LUFS**. A pop SFX should be *audible and punchy but never bury the word* — a few dB under the VO peak. A loud SFX that "drowns out dialogue" is the #1 cited amateur mistake.

### 2d. Visual treatment
- **Drop shadow:** soft shadow to lift the sticker off the footage (separation from background). Subtle, not heavy.
- **Tilt/rotation:** a slight resting tilt, **±3° to ±8°**, makes a "sticker / sticky note" feel hand-placed and casual (vs a sterile 0°). Combine with rotate-in that overshoots and settles at the tilt.
- **Paper/sticker border:** white die-cut "sticker" stroke or a paper/sticky-note texture sells the metaphor.
- **Positioning:** place in a **safe zone that avoids the speaker's face and the caption band**. Smart caption systems auto-avoid faces/key elements; do the same for stickers — typically upper-third or a side gutter, opposite the speaker's head. Never cover the mouth or the caption.

Sources: gsap-overshoot-pop, capcut-zoom-shake-2025, sfxengine-tiktok-sfx, sfxengine-2026-mistakes, soundstripe-audio-levels, epidemic-tiktok-sfx, opus caption-timing (100–200ms lead), Apple Motion overshoot.

---

## 3. EMOJI / STICKER / B-ROLL CUE CONVENTIONS (when to drop one)

The decision rule for *intentional* cutaways/emoji, not just cadence-filling.

- **Anchor to meaning, not a timer.** "Anchor cuts to emotional beats, not time. Each edit happens at a point of emotional shift (reveal, realization, reflection)." A cue feels intentional when it lands on a **noun you can show, a punchline, a number/stat, or an emotional spike** — not on an arbitrary clock tick.
- **Pacing target (the ambient floor):** **a visual change every 1.5–2.0s** is the 2026 short-form target for sub-60s video; vertical/short can go tighter **~0.8–2.0s**. **Below ~1.2s the brain treats the pacing as noise and tunes out** — so faster is NOT always better. (Longer-form: a change every 15–25s.)
- **Emoji/text highlights:** "sprinkle in text or emoji highlights **only for big emotions or punchlines**." Scarcity preserves impact; spamming them oversaturates and they stop reading as emphasis.
- **B-roll overuse warning:** "you risk overusing b-roll if you constantly cut to it to meet a timing statistic… when you finally need it for something important, the viewer won't appreciate it." Treat the per-1.5–2s cadence as a *ceiling for visual variety*, satisfied by ANY change (angle, zoom, b-roll, cutaway, sticker) — not a mandate to cut to literal b-roll that often.
- **Implication for presets:** a good auto-cue engine fires overlays on (a) keyword/entity matches in the transcript, (b) detected emphasis/loudness peaks, or (c) sentence/beat boundaries — then RATE-LIMITS so no two fire within ~0.8–1.2s and so emoji/sticker accents stay rare relative to zooms.

Sources: aibrify-2026-pacing, air.io retention editing, visla b-roll.

---

## 4. EASING & TIMING LANGUAGE (snappy/modern vs cheap)

What separates "pro snappy" from "cheap linear/default-ease": **fast attack + ease-out landing**, optional **single small overshoot**, and **short durations**. Avoid symmetric `ease`/linear on accents (reads default/cheap); avoid >2–3 visible bounces (reads toy-like) except for deliberately playful content.

### 4a. Cubic-bezier menu (drop-in)
- **Snappy ease-out (lands hard, no overshoot)** — punch-in, UI pops:
  `cubic-bezier(0.16, 1, 0.3, 1)` (easeOutExpo) or `cubic-bezier(0.22, 1, 0.36, 1)` (easeOutQuint).
- **Slight overshoot ("pop", physical):** `cubic-bezier(0.34, 1.56, 0.64, 1)` — the back-out; control point >1 makes it overshoot the target. This is THE go-to "designed spring" approximation (Carmen Ansio's `--motion-spring` token).
- **Gentle in-out (Ken Burns, ambient):** `cubic-bezier(0.45, 0, 0.55, 1)`.
- **Standard ease-out (general UI):** `cubic-bezier(0.0, 0, 0.2, 1)` (Material decelerate).

### 4b. Spring params (stiffness / damping / mass) — perception table
From Carmen Ansio's measured table (mass=1 unless noted):
| Want | Stiffness | Damping | Mass | Result |
|---|---|---|---|---|
| Snappy, no bounce | 300 | 30 | 1 | Fast settle, near-critical |
| Light bounce | 200 | 15 | 1 | One small overshoot |
| Playful, bouncy | 180 | 8 | 1 | 2–3 bounces, fun |
| Heavy object | 120 | 14 | 2 | Slow, weighty, one overshoot |
| Instant, decisive | 400 | 40 | 1 | Very snappy |

Rules of thumb: **stiffness and damping move together** — doubling stiffness without raising damping makes it bouncier; raise damping to keep settle speed but kill oscillation. **Mass is the slow-mo knob** (higher = heavier, same shape). For interactive/feedback elements use stiffness **200–300**. Critically damped ≈ stiffness 200 / damping 28 / mass 1 = fastest settle, zero overshoot.

- **Framer Motion defaults:** stiffness 100, damping 10, mass 1 (bouncy-ish). For pops, override to stiffness ~260–300, damping ~20.
- **Remotion `spring()` defaults:** `{ damping: 10, mass: 1, stiffness: 100 }`. Set `damping: 100` (or `overshootClamping: true`) to remove bounce. Has `durationInFrames` to stretch the spring to an exact frame budget — ideal for deterministic video.
- **SwiftUI named springs (iOS 17):** `.smooth` (duration 0.5, extraBounce 0.0, no bounce), `.snappy` (duration 0.5, small bounce), `.bouncy` (duration 0.5, large bounce). Custom: `.spring(duration: 0.5, bounce: 0.2)` — bounce 0.15 = subtle, 0.3 = lively, 0.5 = very bouncy/overshoots. Maps cleanly to "duration + bounce" preset knobs.

### 4c. Overshoot/oscillation math (for a code spring without a physics lib)
Exponentially decaying sine (motionscript / AE-expression form), settling at the target:
```
value = target + amp * sin(t * freq * 2π) / exp(t * decay)
// motionscript starting values: amp = 80 (in its px example), freq = 1, decay = 1
```
- `amp` = initial overshoot magnitude (in your units; for scale, e.g. 0.2 → 20% past target). With overshoot you have a **harmonic oscillation: frequency stays constant while amplitude decays** (unlike a bounce, where frequency also speeds up).
- `freq` ≈ 1–3 for a natural settle (higher = more wiggles per second). `decay` ≈ 4–8 for a quick 1–2-overshoot settle in short-form (their 1 is slow/demonstrative; push higher for snappy).
- For a clean "one overshoot then stop," prefer a spring config (stiffness 200 / damping 15) over hand-tuned sine.

### 4d. `linear()` springs (modern CSS, frame-baked tables)
Comeau (Oct 2025, upd. May 2026): a 4-point cubic-bezier *cannot* faithfully reproduce a real spring; the modern approach bakes a spring into a many-point `linear()` timing function (e.g. ~20–40 sampled points) so it can overshoot (values outside 0–1) and wobble correctly. He records the source `stiffness/damping` in a comment (e.g. "stiffness: 235, damping: 10" → `--spring-smooth: linear(...)`). For a CODE video harness this is exactly the right model: **simulate the spring once, sample it per-frame into a value table, render deterministically.** Damping=0 → infinite oscillation (clamp duration); low mass+stiffness makes damping weak — keep stiffness ≥ ~120.

### 4e. Frame counts cheat (the "feel" durations)
- Micro UI pop / sticker entrance: **6–12f @30fps** (12–24f @60fps).
- Punch-in snap: **2–5f**; pulse-return: **6–10f**.
- Word-pop overshoot settle: **~8–10f**.
- Exit/pop-out: **4–8f** (faster than entrance).
- Ken Burns push: **90–150f** (3–5s).
- Caption/overlay lead before audio: **3–6f (100–200ms)**.
- SFX transient offset: start SFX **~1–3f before** the visual peak.
- Cutaway/visual-variety cadence: a change every **~45–60f (1.5–2s)**, floor ~24f (0.8s); never under ~36f (1.2s) sustained.

Sources: ansio-spring-physics, comeau-springs-css, heckel-spring-physics, getstream-spring-params, remotion-spring-docs, motionscript-bounce-overshoot, gsap-overshoot-pop.

---

## SOURCE URLs
- CapCut Zoom+Shake 2025: https://vediting.home.blog/2025/11/01/how-to-create-a-zoom-shake-effect-in-capcut-complete-2025-guide/
- Submagic — Zoom in CapCut 2025: https://www.submagic.co/blog/how-to-zoom-in-on-capcut
- Poindeo — CapCut zoom: https://poindeo.com/blog/zoom-in-on-capcut
- Cloudinary — Ken Burns guide: https://cloudinary.com/guides/image-effects/ken-burns-effect-complete-guide-and-how-to-apply-it
- Apple FCP — Pan & zoom: https://support.apple.com/guide/final-cut-pro/pan-and-zoom-clips-verb8e5de9c/mac
- Aibrify — 2026 Pacing Model (b-roll/cadence): https://aibrify.com/blog/short-form-video-editing-captions-b-roll-guide
- AIR — Retention editing (anchor cuts to beats): https://air.io/en/youtube-hacks/advanced-retention-editing-cutting-patterns-that-keep-viewers-past-minute-8
- GSAP — Simple overshoot/pop (scale 0→1.2→1): https://gsap.com/community/forums/topic/29105-simple-overshootpop-animation/
- Apple Motion — Overshoot behavior: https://support.apple.com/guide/motion/overshoot-behavior-motn0c60c821/mac
- motionscript — Realistic Bounce and Overshoot (decaying-sine math): https://motionscript.com/articles/bounce-and-overshoot.html
- Carmen Ansio — Spring Physics in CSS (perception table): https://www.carmenansio.com/articles/spring-physics-css/
- Josh Comeau — Springs/Bounces in native CSS (linear() ): https://www.joshwcomeau.com/animation/linear-timing-function/
- Maxime Heckel — Physics behind spring animations: https://blog.maximeheckel.com/posts/the-physics-behind-spring-animations/
- GetStream — SwiftUI spring cheat sheet (.smooth/.snappy/.bouncy): https://github.com/GetStream/swiftui-spring-animations
- Remotion — spring() docs (defaults, durationInFrames): https://www.remotion.dev/docs/spring
- SFX Engine — TikTok sound effects tips: https://sfxengine.com/blog/tiktok-sound-effects-tips
- SFX Engine — 8 sound design mistakes 2026 (levels/sync): https://sfxengine.com/blog/common-sound-design-mistakes-in-video-editing
- Soundstripe — Audio levels (headroom/anchor): https://www.soundstripe.com/blogs/how-to-perfect-your-audio-levels
- Epidemic Sound — TikTok SFX library: https://www.epidemicsound.com/tiktok/tik-tok-sound-effects/
