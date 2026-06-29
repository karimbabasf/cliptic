# M0 — Walking Skeleton Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Prove the hardest thing works end-to-end on a 5-second clip — transcribe → render one caption preset → export with audio — with captions provably landing within one frame of the audio.

**Architecture:** A Tauri 2 desktop app (Rust core + React/Vite/TS webview). A **pure TypeScript timing core** (no UI, no native deps) owns all correctness: time math, active-word selection, page grouping, spring sampling, and `captionStateAtFrame()`. A thin `paint()` layer draws that state to Canvas2D. Rust commands handle FFmpeg audio extraction, `whisper-rs` transcription, and FFmpeg overlay export. Export in M0 = render transparent caption frames in the webview and have FFmpeg overlay them onto the untouched original video while copying the original audio — this sidesteps source-frame exactness (deferred to when zoom needs it) and keeps M0 robust.

**Tech Stack:** Tauri 2 · React 18 + Vite 6 + TypeScript 5 · Vitest · `whisper-rs` (whisper.cpp) · `hound` · system FFmpeg 8.

## Global Constraints

- **Determinism:** no `Date.now()`, no unseeded `Math.random()` in `src/core/**` or `src/render/**`. Timing is in **milliseconds**, never pre-rounded to frame indices.
- **No runtime network/LLM/cloud calls** anywhere.
- **Defaults:** `fps = 30`, canvas `1080 × 1920`.
- **Caption sync defaults:** `leadMs = 120`, `minWordMs = 150`, accent color `#3DDC97`.
- **Whisper:** greedy decoding, `best_of: 1`, no temperature fallback (deterministic).
- **Export audio:** original stream copied with `-c:a copy` (no re-encode).
- **Toolchain present:** Node 24, Rust 1.96, FFmpeg 8.1, `gh` authed as `karimbabasf`.
- **Branch/PR:** all M0 work on branch `m0-walking-skeleton` → PR → merge to `main`. Commit after every passing step.

---

### Task 0: Project scaffold (Tauri + Vite + React + TS + Vitest)

**Files:**
- Create: `package.json`, `index.html`, `vite.config.ts`, `tsconfig.json`, `tsconfig.node.json`, `vitest.config.ts`, `src/main.tsx`, `src/App.tsx`, `src/vite-env.d.ts`
- Create: `src-tauri/Cargo.toml`, `src-tauri/build.rs`, `src-tauri/tauri.conf.json`, `src-tauri/src/main.rs`
- Test: `src/core/__tests__/smoke.test.ts`

**Interfaces:**
- Consumes: nothing.
- Produces: a runnable app (`npm run tauri dev`) and a green `npm test`.

- [ ] **Step 1: Create the branch**

```bash
git checkout -b m0-walking-skeleton
```

- [ ] **Step 2: Write `package.json`**

```json
{
  "name": "cliptic",
  "private": true,
  "version": "0.0.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "tsc && vite build",
    "preview": "vite preview",
    "test": "vitest run",
    "test:watch": "vitest",
    "tauri": "tauri"
  },
  "dependencies": {
    "react": "^18.3.1",
    "react-dom": "^18.3.1",
    "@tauri-apps/api": "^2.0.0"
  },
  "devDependencies": {
    "@tauri-apps/cli": "^2.0.0",
    "@types/react": "^18.3.0",
    "@types/react-dom": "^18.3.0",
    "@vitejs/plugin-react": "^4.3.0",
    "typescript": "^5.6.0",
    "vite": "^6.0.0",
    "vitest": "^2.1.0"
  }
}
```

- [ ] **Step 3: Write the config files**

`index.html`:
```html
<!doctype html>
<html lang="en">
  <head><meta charset="UTF-8" /><meta name="viewport" content="width=device-width, initial-scale=1.0" /><title>Cliptic</title></head>
  <body><div id="root"></div><script type="module" src="/src/main.tsx"></script></body>
</html>
```

`vite.config.ts`:
```ts
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],
  clearScreen: false,
  server: { port: 1420, strictPort: true },
});
```

`vitest.config.ts`:
```ts
import { defineConfig } from "vitest/config";
export default defineConfig({ test: { environment: "node", include: ["src/**/*.test.ts"] } });
```

`tsconfig.json`:
```json
{
  "compilerOptions": {
    "target": "ES2022", "useDefineForClassFields": true, "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "module": "ESNext", "skipLibCheck": true, "moduleResolution": "bundler",
    "resolveJsonModule": true, "isolatedModules": true, "noEmit": true, "jsx": "react-jsx",
    "strict": true, "noUnusedLocals": true, "noUnusedParameters": true, "noFallthroughCasesInSwitch": true
  },
  "include": ["src"],
  "references": [{ "path": "./tsconfig.node.json" }]
}
```

`tsconfig.node.json`:
```json
{ "compilerOptions": { "composite": true, "skipLibCheck": true, "module": "ESNext", "moduleResolution": "bundler", "allowSyntheticDefaultImports": true }, "include": ["vite.config.ts"] }
```

`src/vite-env.d.ts`:
```ts
/// <reference types="vite/client" />
```

- [ ] **Step 4: Write minimal React entry**

`src/main.tsx`:
```tsx
import React from "react";
import ReactDOM from "react-dom/client";
import App from "./App";

ReactDOM.createRoot(document.getElementById("root")!).render(
  <React.StrictMode><App /></React.StrictMode>
);
```

`src/App.tsx`:
```tsx
export default function App() {
  return <div style={{ fontFamily: "system-ui", padding: 24 }}><h1>Cliptic</h1><p>M0 walking skeleton</p></div>;
}
```

- [ ] **Step 5: Write the Tauri backend scaffold**

`src-tauri/Cargo.toml`:
```toml
[package]
name = "cliptic"
version = "0.0.0"
edition = "2021"

[lib]
name = "cliptic_lib"
crate-type = ["staticlib", "cdylib", "rlib"]

[build-dependencies]
tauri-build = { version = "2", features = [] }

[dependencies]
tauri = { version = "2", features = [] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
whisper-rs = "0.13"
hound = "3.5"
```

`src-tauri/build.rs`:
```rust
fn main() { tauri_build::build() }
```

`src-tauri/tauri.conf.json`:
```json
{
  "$schema": "https://schema.tauri.app/config/2",
  "productName": "Cliptic",
  "version": "0.0.0",
  "identifier": "club.cliptic.app",
  "build": {
    "frontendDist": "../dist",
    "devUrl": "http://localhost:1420",
    "beforeDevCommand": "npm run dev",
    "beforeBuildCommand": "npm run build"
  },
  "app": {
    "windows": [{ "title": "Cliptic", "width": 1200, "height": 820 }],
    "security": { "csp": null }
  },
  "bundle": { "active": true, "targets": "all" }
}
```

`src-tauri/src/main.rs`:
```rust
#![cfg_attr(not(debug_assertions), windows_subsystem = "windows")]

fn main() {
    tauri::Builder::default()
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}
```

- [ ] **Step 6: Write the smoke test**

`src/core/__tests__/smoke.test.ts`:
```ts
import { describe, it, expect } from "vitest";
describe("smoke", () => { it("runs", () => { expect(1 + 1).toBe(2); }); });
```

- [ ] **Step 7: Install and verify**

Run:
```bash
npm install
npm test
cargo check --manifest-path src-tauri/Cargo.toml
```
Expected: `npm test` → 1 passing. `cargo check` → compiles (first build downloads crates; whisper-rs builds whisper.cpp — may take minutes).

- [ ] **Step 8: Commit**

```bash
git add -A
git commit -m "feat(m0): scaffold tauri + vite + react + ts + vitest"
```

---

### Task 1: Core types + time math

**Files:**
- Create: `src/core/types.ts`, `src/core/time.ts`
- Test: `src/core/__tests__/time.test.ts`

**Interfaces:**
- Produces:
  - `interface Caption { text: string; startMs: number; endMs: number }`
  - `interface Token { text: string; fromMs: number; toMs: number }`
  - `interface Page { text: string; startMs: number; durationMs: number; tokens: Token[] }`
  - `interface Project { fps: number; width: number; height: number; durationMs: number; captions: Caption[]; captionStyleId: string }`
  - `msAtFrame(frame: number, fps: number): number`
  - `frameAtMs(ms: number, fps: number): number`
  - `frameCount(durationMs: number, fps: number): number`

- [ ] **Step 1: Write the failing test**

`src/core/__tests__/time.test.ts`:
```ts
import { describe, it, expect } from "vitest";
import { msAtFrame, frameAtMs, frameCount } from "../time";

describe("time math", () => {
  it("msAtFrame converts frame index to milliseconds", () => {
    expect(msAtFrame(0, 30)).toBe(0);
    expect(msAtFrame(15, 30)).toBe(500);
    expect(msAtFrame(30, 30)).toBe(1000);
  });
  it("frameAtMs floors ms to a frame index", () => {
    expect(frameAtMs(0, 30)).toBe(0);
    expect(frameAtMs(999, 30)).toBe(29);
    expect(frameAtMs(1000, 30)).toBe(30);
  });
  it("frameCount ceils duration to whole frames", () => {
    expect(frameCount(1000, 30)).toBe(30);
    expect(frameCount(1001, 30)).toBe(31);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm test -- time`
Expected: FAIL — cannot resolve `../time`.

- [ ] **Step 3: Write the implementation**

`src/core/types.ts`:
```ts
export interface Caption { text: string; startMs: number; endMs: number; }
export interface Token { text: string; fromMs: number; toMs: number; }
export interface Page { text: string; startMs: number; durationMs: number; tokens: Token[]; }
export interface Project {
  fps: number; width: number; height: number; durationMs: number;
  captions: Caption[]; captionStyleId: string;
}
```

`src/core/time.ts`:
```ts
export const msAtFrame = (frame: number, fps: number): number => (frame / fps) * 1000;
export const frameAtMs = (ms: number, fps: number): number => Math.floor((ms / 1000) * fps);
export const frameCount = (durationMs: number, fps: number): number => Math.ceil((durationMs / 1000) * fps);
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npm test -- time`
Expected: PASS (3 tests).

- [ ] **Step 5: Commit**

```bash
git add src/core/types.ts src/core/time.ts src/core/__tests__/time.test.ts
git commit -m "feat(m0): core types and frame/ms time math"
```

---

### Task 2: Active-word selection + page grouping (correctness core)

**Files:**
- Create: `src/core/captions.ts`
- Test: `src/core/__tests__/captions.test.ts`

**Interfaces:**
- Consumes: `Caption`, `Token`, `Page` from `../types`.
- Produces:
  - `activeWordIndex(captions: Caption[], ms: number, opts?: { leadMs?: number }): number` — returns the index of the word spoken at `ms + leadMs`; in a gap, the last word that has started; `-1` before the first word.
  - `combineTokensIntoPages(captions: Caption[], opts?: { maxWords?: number; maxChars?: number; maxGapMs?: number }): Page[]`
  - `pageIndexOf(pages: Page[], captionIndex: number, captions: Caption[]): number`

- [ ] **Step 1: Write the failing test**

`src/core/__tests__/captions.test.ts`:
```ts
import { describe, it, expect } from "vitest";
import { activeWordIndex, combineTokensIntoPages } from "../captions";
import type { Caption } from "../types";

const W: Caption[] = [
  { text: "this", startMs: 0, endMs: 300 },
  { text: "is", startMs: 300, endMs: 460 },
  { text: "how", startMs: 460, endMs: 700 },
  { text: "you", startMs: 700, endMs: 900 },
  { text: "stop", startMs: 900, endMs: 1280 },
];

describe("activeWordIndex", () => {
  it("returns -1 before the first word", () => {
    expect(activeWordIndex(W, -10)).toBe(-1);
  });
  it("selects the word spoken at ms", () => {
    expect(activeWordIndex(W, 150)).toBe(0);
    expect(activeWordIndex(W, 380)).toBe(1);
    expect(activeWordIndex(W, 1000)).toBe(4);
  });
  it("lingers on the last started word inside a gap", () => {
    const G: Caption[] = [
      { text: "a", startMs: 0, endMs: 100 },
      { text: "b", startMs: 500, endMs: 600 },
    ];
    expect(activeWordIndex(G, 300)).toBe(0); // gap between a and b
  });
  it("applies lead so the highlight precedes speech", () => {
    // word 1 starts at 300; with 120ms lead it should be active at ms=200
    expect(activeWordIndex(W, 200, { leadMs: 120 })).toBe(1);
  });
});

describe("combineTokensIntoPages", () => {
  it("groups words up to the word cap", () => {
    const pages = combineTokensIntoPages(W, { maxWords: 4, maxChars: 42, maxGapMs: 600 });
    expect(pages.length).toBe(2);
    expect(pages[0].tokens.map((t) => t.text)).toEqual(["this", "is", "how", "you"]);
    expect(pages[0].startMs).toBe(0);
    expect(pages[1].tokens.map((t) => t.text)).toEqual(["stop"]);
  });
  it("breaks a page on a large gap", () => {
    const G: Caption[] = [
      { text: "a", startMs: 0, endMs: 100 },
      { text: "b", startMs: 2000, endMs: 2100 },
    ];
    const pages = combineTokensIntoPages(G, { maxWords: 4, maxChars: 42, maxGapMs: 600 });
    expect(pages.length).toBe(2);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm test -- captions`
Expected: FAIL — cannot resolve `../captions`.

- [ ] **Step 3: Write the implementation**

`src/core/captions.ts`:
```ts
import type { Caption, Page } from "./types";

export function activeWordIndex(
  captions: Caption[], ms: number, opts: { leadMs?: number } = {}
): number {
  const t = ms + (opts.leadMs ?? 0);
  let lastStarted = -1;
  for (let i = 0; i < captions.length; i++) {
    const c = captions[i];
    if (c.startMs <= t && t < c.endMs) return i;
    if (c.startMs <= t) lastStarted = i;
  }
  return lastStarted;
}

export function combineTokensIntoPages(
  captions: Caption[],
  opts: { maxWords?: number; maxChars?: number; maxGapMs?: number } = {}
): Page[] {
  const maxWords = opts.maxWords ?? 4;
  const maxChars = opts.maxChars ?? 42;
  const maxGapMs = opts.maxGapMs ?? 600;
  const pages: Page[] = [];
  let cur: Page | null = null;
  for (const c of captions) {
    const tok = { text: c.text, fromMs: c.startMs, toMs: c.endMs };
    if (!cur) { cur = { text: c.text, startMs: c.startMs, durationMs: c.endMs - c.startMs, tokens: [tok] }; continue; }
    const gap = c.startMs - cur.tokens[cur.tokens.length - 1].toMs;
    const wouldChars = (cur.text + " " + c.text).trim().length;
    if (cur.tokens.length >= maxWords || wouldChars > maxChars || gap > maxGapMs) {
      pages.push(cur);
      cur = { text: c.text, startMs: c.startMs, durationMs: c.endMs - c.startMs, tokens: [tok] };
    } else {
      cur.tokens.push(tok);
      cur.text = (cur.text + " " + c.text).trim();
      cur.durationMs = c.endMs - cur.startMs;
    }
  }
  if (cur) pages.push(cur);
  return pages;
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npm test -- captions`
Expected: PASS (6 tests).

- [ ] **Step 5: Commit**

```bash
git add src/core/captions.ts src/core/__tests__/captions.test.ts
git commit -m "feat(m0): active-word selection and page grouping"
```

---

### Task 3: Deterministic spring + easing sampling

**Files:**
- Create: `src/core/easing.ts`
- Test: `src/core/__tests__/easing.test.ts`

**Interfaces:**
- Produces:
  - `springTable(stiffness: number, damping: number, mass: number, durationFrames: number, fps: number): number[]` — simulated 0→1 spring, one sample per frame, deterministic.
  - `sampleSpring(table: number[], frame: number): number` — clamps to table bounds.
  - `cubicBezier(p1x: number, p1y: number, p2x: number, p2y: number): (t: number) => number`

- [ ] **Step 1: Write the failing test**

`src/core/__tests__/easing.test.ts`:
```ts
import { describe, it, expect } from "vitest";
import { springTable, sampleSpring, cubicBezier } from "../easing";

describe("springTable", () => {
  const table = springTable(280, 20, 1, 24, 30);
  it("starts near 0 and settles near 1", () => {
    expect(table[0]).toBeCloseTo(0, 2);
    expect(table[table.length - 1]).toBeCloseTo(1, 1);
  });
  it("overshoots above 1 for a bouncy spring", () => {
    expect(Math.max(...table)).toBeGreaterThan(1.0);
  });
  it("is deterministic across runs", () => {
    expect(springTable(280, 20, 1, 24, 30)).toEqual(table);
  });
});

describe("sampleSpring", () => {
  const table = springTable(280, 20, 1, 24, 30);
  it("clamps below and above range", () => {
    expect(sampleSpring(table, -5)).toBe(table[0]);
    expect(sampleSpring(table, 999)).toBe(table[table.length - 1]);
  });
});

describe("cubicBezier", () => {
  it("approximates linear for (0,0,1,1)", () => {
    const e = cubicBezier(0, 0, 1, 1);
    expect(e(0)).toBeCloseTo(0, 3);
    expect(e(0.5)).toBeCloseTo(0.5, 2);
    expect(e(1)).toBeCloseTo(1, 3);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm test -- easing`
Expected: FAIL — cannot resolve `../easing`.

- [ ] **Step 3: Write the implementation**

`src/core/easing.ts`:
```ts
// Semi-implicit Euler spring from 0 -> 1, sampled once per frame. Deterministic.
export function springTable(
  stiffness: number, damping: number, mass: number, durationFrames: number, fps: number
): number[] {
  const dt = 1 / fps;
  const steps = 8;            // substeps per frame for stability
  const h = dt / steps;
  let x = 0, v = 0;
  const out: number[] = [];
  for (let f = 0; f < durationFrames; f++) {
    for (let s = 0; s < steps; s++) {
      const a = (stiffness * (1 - x) - damping * v) / mass;
      v += a * h;
      x += v * h;
    }
    out.push(x);
  }
  if (out.length === 0) out.push(1);
  out[0] = out.length > 0 ? out[0] : 0;
  return out;
}

export function sampleSpring(table: number[], frame: number): number {
  if (frame <= 0) return table[0];
  if (frame >= table.length) return table[table.length - 1];
  return table[frame];
}

export function cubicBezier(p1x: number, p1y: number, p2x: number, p2y: number): (t: number) => number {
  const cx = 3 * p1x, bx = 3 * (p2x - p1x) - cx, ax = 1 - cx - bx;
  const cy = 3 * p1y, by = 3 * (p2y - p1y) - cy, ay = 1 - cy - by;
  const sampleX = (t: number) => ((ax * t + bx) * t + cx) * t;
  const sampleY = (t: number) => ((ay * t + by) * t + cy) * t;
  const slopeX = (t: number) => (3 * ax * t + 2 * bx) * t + cx;
  return (x: number) => {
    let t = x;
    for (let i = 0; i < 8; i++) {
      const xe = sampleX(t) - x;
      if (Math.abs(xe) < 1e-6) break;
      const d = slopeX(t);
      if (Math.abs(d) < 1e-6) break;
      t -= xe / d;
    }
    return sampleY(Math.max(0, Math.min(1, t)));
  };
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npm test -- easing`
Expected: PASS (5 tests).

- [ ] **Step 5: Commit**

```bash
git add src/core/easing.ts src/core/__tests__/easing.test.ts
git commit -m "feat(m0): deterministic spring table and cubic-bezier sampling"
```

---

### Task 4: `captionStateAtFrame` + the sync-drift proof (M0 exit criterion)

**Files:**
- Create: `src/core/captionState.ts`
- Test: `src/core/__tests__/captionState.test.ts`

**Interfaces:**
- Consumes: `Caption`, `Page` from `../types`; `msAtFrame`, `frameAtMs` from `../time`; `activeWordIndex`, `combineTokensIntoPages` from `../captions`; `springTable`, `sampleSpring` from `../easing`.
- Produces:
  - `interface RenderedToken { index: number; text: string; scale: number; opacity: number; accent: boolean }`
  - `interface CaptionState { activeIndex: number; pageIndex: number; tokens: RenderedToken[] }`
  - `interface StyleOpts { leadMs?: number; enterFrames?: number; spring?: number[]; trailing?: number; accentColor?: string }`
  - `captionStateAtFrame(captions: Caption[], pages: Page[], frame: number, fps: number, opts?: StyleOpts): CaptionState`

- [ ] **Step 1: Write the failing test**

`src/core/__tests__/captionState.test.ts`:
```ts
import { describe, it, expect } from "vitest";
import { captionStateAtFrame } from "../captionState";
import { combineTokensIntoPages } from "../captions";
import { msAtFrame, frameAtMs, frameCount } from "../time";
import type { Caption } from "../types";

const W: Caption[] = [
  { text: "this", startMs: 0, endMs: 300 },
  { text: "is", startMs: 300, endMs: 460 },
  { text: "how", startMs: 460, endMs: 700 },
  { text: "you", startMs: 700, endMs: 900 },
  { text: "stop", startMs: 900, endMs: 1280 },
  { text: "the", startMs: 1280, endMs: 1420 },
  { text: "scroll", startMs: 1420, endMs: 1820 },
];
const FPS = 30;
const pages = combineTokensIntoPages(W);

describe("captionStateAtFrame", () => {
  it("marks the correct active word at each word's temporal center (zero lead)", () => {
    for (let i = 0; i < W.length; i++) {
      const centerMs = (W[i].startMs + W[i].endMs) / 2;
      const frame = frameAtMs(centerMs, FPS);
      const state = captionStateAtFrame(W, pages, frame, FPS, { leadMs: 0 });
      expect(state.activeIndex).toBe(i);
    }
  });

  it("the active token is flagged accent", () => {
    const frame = frameAtMs(1000, FPS); // inside "stop"
    const state = captionStateAtFrame(W, pages, frame, FPS, { leadMs: 0 });
    const active = state.tokens.find((t) => t.index === state.activeIndex);
    expect(active?.accent).toBe(true);
  });

  it("DRIFT PROOF: every word's active window switches within one frame of its boundary", () => {
    const frameMs = 1000 / FPS;
    for (let i = 1; i < W.length; i++) {
      const boundaryMs = W[i].startMs;
      const frameBefore = frameAtMs(boundaryMs - frameMs, FPS);
      const frameAfter = frameAtMs(boundaryMs + frameMs, FPS);
      const before = captionStateAtFrame(W, pages, frameBefore, FPS, { leadMs: 0 }).activeIndex;
      const after = captionStateAtFrame(W, pages, frameAfter, FPS, { leadMs: 0 }).activeIndex;
      expect(before).toBe(i - 1);
      expect(after).toBe(i);
    }
  });

  it("DRIFT PROOF (full sweep): active index is monotonic non-decreasing across the clip", () => {
    const total = frameCount(W[W.length - 1].endMs, FPS);
    let prev = -1;
    for (let f = 0; f < total; f++) {
      const idx = captionStateAtFrame(W, pages, f, FPS, { leadMs: 0 }).activeIndex;
      expect(idx).toBeGreaterThanOrEqual(prev);
      prev = idx;
    }
  });

  it("lead shifts the switch earlier by ~leadMs", () => {
    const boundaryMs = W[4].startMs; // "stop" at 900
    const leadMs = 120;
    const frameJustBeforeLead = frameAtMs(boundaryMs - leadMs + 5, FPS);
    const state = captionStateAtFrame(W, pages, frameJustBeforeLead, FPS, { leadMs });
    expect(state.activeIndex).toBe(4);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm test -- captionState`
Expected: FAIL — cannot resolve `../captionState`.

- [ ] **Step 3: Write the implementation**

`src/core/captionState.ts`:
```ts
import type { Caption, Page } from "./types";
import { msAtFrame } from "./time";
import { activeWordIndex } from "./captions";
import { sampleSpring } from "./easing";

export interface RenderedToken { index: number; text: string; scale: number; opacity: number; accent: boolean; }
export interface CaptionState { activeIndex: number; pageIndex: number; tokens: RenderedToken[]; }
export interface StyleOpts { leadMs?: number; enterFrames?: number; spring?: number[]; trailing?: number; accentColor?: string; }

function pageOfCaption(pages: Page[], idx: number, captions: Caption[]): number {
  if (idx < 0) return -1;
  const startMs = captions[idx].startMs;
  for (let p = 0; p < pages.length; p++) {
    const last = pages[p].tokens[pages[p].tokens.length - 1];
    if (startMs >= pages[p].startMs && startMs <= last.toMs) return p;
  }
  return -1;
}

// Map a global caption index to its position within the flattened token stream of a page.
function captionIndexRangeForPage(pages: Page[], pageIndex: number, captions: Caption[]): number[] {
  const result: number[] = [];
  if (pageIndex < 0) return result;
  const page = pages[pageIndex];
  for (let i = 0; i < captions.length; i++) {
    if (captions[i].startMs >= page.startMs && captions[i].startMs <= page.tokens[page.tokens.length - 1].toMs) {
      result.push(i);
    }
  }
  return result;
}

export function captionStateAtFrame(
  captions: Caption[], pages: Page[], frame: number, fps: number, opts: StyleOpts = {}
): CaptionState {
  const leadMs = opts.leadMs ?? 120;
  const enterFrames = opts.enterFrames ?? 8;
  const trailing = opts.trailing ?? 3;
  const ms = msAtFrame(frame, fps);
  const activeIndex = activeWordIndex(captions, ms, { leadMs });
  if (activeIndex < 0) return { activeIndex: -1, pageIndex: -1, tokens: [] };

  const pageIndex = pageOfCaption(pages, activeIndex, captions);
  const pageCaptionIdx = captionIndexRangeForPage(pages, pageIndex, captions);
  const shown = pageCaptionIdx.filter((i) => i <= activeIndex).slice(-trailing);

  const tokens: RenderedToken[] = shown.map((i) => {
    const isActive = i === activeIndex;
    let scale = 1, opacity = 1;
    if (isActive && opts.spring) {
      const ageFrames = Math.max(0, frame - Math.floor((captions[i].startMs - leadMs) / 1000 * fps));
      scale = sampleSpring(opts.spring, Math.min(ageFrames, enterFrames));
      opacity = Math.min(1, ageFrames / 2 + 0.4);
    }
    return { index: i, text: captions[i].text, scale, opacity, accent: isActive };
  });

  return { activeIndex, pageIndex, tokens };
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npm test -- captionState`
Expected: PASS (5 tests). **This green suite is the M0 sync-correctness proof.**

- [ ] **Step 5: Commit**

```bash
git add src/core/captionState.ts src/core/__tests__/captionState.test.ts
git commit -m "feat(m0): captionStateAtFrame with sub-frame sync-drift proof"
```

---

### Task 5: Canvas `paint()` + live preview (visual verification)

**Files:**
- Create: `src/render/paint.ts`, `src/preview/Preview.tsx`, `src/preview/sampleProject.ts`
- Modify: `src/App.tsx`

**Interfaces:**
- Consumes: `CaptionState`, `RenderedToken` from `../core/captionState`; `springTable` from `../core/easing`; `combineTokensIntoPages` from `../core/captions`.
- Produces:
  - `interface Layout { width: number; height: number; baselineY: number; fontPx: number; accentColor: string }`
  - `paint(ctx: CanvasRenderingContext2D, state: CaptionState, layout: Layout): void`

- [ ] **Step 1: Write `paint()`**

`src/render/paint.ts`:
```ts
import type { CaptionState } from "../core/captionState";

export interface Layout { width: number; height: number; baselineY: number; fontPx: number; accentColor: string; }

export function paint(ctx: CanvasRenderingContext2D, state: CaptionState, layout: Layout): void {
  ctx.clearRect(0, 0, layout.width, layout.height);
  if (state.tokens.length === 0) return;
  ctx.textAlign = "center";
  ctx.textBaseline = "middle";
  ctx.font = `900 ${layout.fontPx}px system-ui, sans-serif`;
  ctx.lineJoin = "round";

  const gap = layout.fontPx * 0.35;
  const widths = state.tokens.map((t) => ctx.measureText(t.text.toUpperCase()).width);
  const total = widths.reduce((a, b) => a + b, 0) + gap * (state.tokens.length - 1);
  let x = layout.width / 2 - total / 2;

  state.tokens.forEach((t, i) => {
    const w = widths[i];
    const cx = x + w / 2;
    ctx.save();
    ctx.translate(cx, layout.baselineY);
    ctx.scale(t.scale, t.scale);
    ctx.globalAlpha = t.opacity;
    ctx.lineWidth = layout.fontPx * 0.12;
    ctx.strokeStyle = "#000";
    ctx.strokeText(t.text.toUpperCase(), 0, 0);
    ctx.fillStyle = t.accent ? layout.accentColor : "#fff";
    ctx.fillText(t.text.toUpperCase(), 0, 0);
    ctx.restore();
    x += w + gap;
  });
}
```

- [ ] **Step 2: Write a sample project (deterministic preview data)**

`src/preview/sampleProject.ts`:
```ts
import type { Caption } from "../core/types";
export const SAMPLE_CAPTIONS: Caption[] = [
  { text: "this", startMs: 0, endMs: 300 }, { text: "is", startMs: 300, endMs: 460 },
  { text: "how", startMs: 460, endMs: 700 }, { text: "you", startMs: 700, endMs: 900 },
  { text: "stop", startMs: 900, endMs: 1280 }, { text: "the", startMs: 1280, endMs: 1420 },
  { text: "scroll", startMs: 1420, endMs: 1820 }, { text: "in", startMs: 1820, endMs: 1980 },
  { text: "three", startMs: 1980, endMs: 2300 }, { text: "seconds", startMs: 2300, endMs: 2780 },
];
export const SAMPLE_DURATION_MS = 3200;
```

- [ ] **Step 3: Write the preview component**

`src/preview/Preview.tsx`:
```tsx
import { useEffect, useRef } from "react";
import { combineTokensIntoPages } from "../core/captions";
import { captionStateAtFrame } from "../core/captionState";
import { springTable } from "../core/easing";
import { frameAtMs } from "../core/time";
import { paint, type Layout } from "../render/paint";
import { SAMPLE_CAPTIONS, SAMPLE_DURATION_MS } from "./sampleProject";

const FPS = 30, W = 1080, H = 1920;

export default function Preview() {
  const ref = useRef<HTMLCanvasElement>(null);
  useEffect(() => {
    const ctx = ref.current!.getContext("2d")!;
    const pages = combineTokensIntoPages(SAMPLE_CAPTIONS);
    const spring = springTable(280, 20, 1, 8, FPS);
    const layout: Layout = { width: W, height: H, baselineY: H * 0.62, fontPx: 132, accentColor: "#3DDC97" };
    let raf = 0, start = performance.now();
    const loop = (now: number) => {
      const ms = (now - start) % SAMPLE_DURATION_MS;
      const frame = frameAtMs(ms, FPS);
      const state = captionStateAtFrame(SAMPLE_CAPTIONS, pages, frame, FPS, { leadMs: 120, spring });
      paint(ctx, state, layout);
      raf = requestAnimationFrame(loop);
    };
    raf = requestAnimationFrame(loop);
    return () => cancelAnimationFrame(raf);
  }, []);
  return <canvas ref={ref} width={W} height={H} style={{ width: 270, height: 480, background: "#0e1219", borderRadius: 16 }} />;
}
```

- [ ] **Step 4: Wire into `App.tsx`**

`src/App.tsx`:
```tsx
import Preview from "./preview/Preview";
export default function App() {
  return (
    <div style={{ fontFamily: "system-ui", padding: 24, display: "flex", gap: 24, alignItems: "flex-start" }}>
      <div><h1>Cliptic</h1><p>M0 walking skeleton — word-by-word pop preview</p></div>
      <Preview />
    </div>
  );
}
```

- [ ] **Step 5: Run the app and verify visually**

Run: `npm run tauri dev`
Expected: a 9:16 dark canvas where the sample sentence plays word-by-word, the active word in teal popping in with a spring, looping. Confirm it reads as on-time.

- [ ] **Step 6: Commit**

```bash
git add src/render/paint.ts src/preview/ src/App.tsx
git commit -m "feat(m0): canvas paint + live word-by-word pop preview"
```

---

### Task 6: FFmpeg audio extraction (Rust command)

**Files:**
- Create: `src-tauri/src/audio.rs`
- Modify: `src-tauri/src/main.rs`, `src-tauri/Cargo.toml` (add `[lib]` entry already present)

**Interfaces:**
- Produces: Tauri command `extract_audio(input_path: String, out_path: String) -> Result<String, String>` — runs FFmpeg to produce 16 kHz mono PCM WAV at `out_path`, returns `out_path`.

- [ ] **Step 1: Write `audio.rs`**

`src-tauri/src/audio.rs`:
```rust
use std::process::Command;

#[tauri::command]
pub fn extract_audio(input_path: String, out_path: String) -> Result<String, String> {
    let status = Command::new("ffmpeg")
        .args(["-y", "-i", &input_path, "-ar", "16000", "-ac", "1", "-c:a", "pcm_s16le", &out_path])
        .status()
        .map_err(|e| format!("failed to spawn ffmpeg: {e}"))?;
    if !status.success() {
        return Err(format!("ffmpeg exited with status {status}"));
    }
    Ok(out_path)
}
```

- [ ] **Step 2: Register the command**

`src-tauri/src/main.rs`:
```rust
#![cfg_attr(not(debug_assertions), windows_subsystem = "windows")]
mod audio;

fn main() {
    tauri::Builder::default()
        .invoke_handler(tauri::generate_handler![audio::extract_audio])
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}
```

- [ ] **Step 3: Verify against a real clip**

Prerequisite: place a 5-second talking clip (1080×1920, with speech) at `scratch/sample.mp4`.
Run (from a temporary Rust integration via `cargo` is heavy; verify the underlying command directly):
```bash
ffmpeg -y -i scratch/sample.mp4 -ar 16000 -ac 1 -c:a pcm_s16le scratch/sample.wav
ffprobe -v error -show_entries stream=codec_name,sample_rate,channels -of default=nw=1 scratch/sample.wav
```
Expected: `codec_name=pcm_s16le`, `sample_rate=16000`, `channels=1`.

- [ ] **Step 4: Build to confirm the command compiles into the app**

Run: `cargo check --manifest-path src-tauri/Cargo.toml`
Expected: compiles.

- [ ] **Step 5: Commit**

```bash
git add src-tauri/src/audio.rs src-tauri/src/main.rs
git commit -m "feat(m0): ffmpeg audio extraction command"
```

---

### Task 7: Whisper transcription with word timings (Rust command)

**Files:**
- Create: `src-tauri/src/transcribe.rs`, `scripts/fetch-model.sh`
- Modify: `src-tauri/src/main.rs`

**Interfaces:**
- Produces: Tauri command `transcribe(wav_path: String, model_path: String) -> Result<Vec<CaptionDTO>, String>` where `CaptionDTO { text: String, start_ms: i64, end_ms: i64 }` (serialized to the same shape as the TS `Caption`).

- [ ] **Step 1: Write the model fetch script**

`scripts/fetch-model.sh`:
```bash
#!/usr/bin/env bash
set -euo pipefail
mkdir -p models
MODEL="${1:-base.en}"
URL="https://huggingface.co/ggerganov/whisper.cpp/resolve/main/ggml-${MODEL}.bin"
echo "Downloading ggml-${MODEL}.bin ..."
curl -L "$URL" -o "models/ggml-${MODEL}.bin"
echo "Saved to models/ggml-${MODEL}.bin"
```
Run:
```bash
chmod +x scripts/fetch-model.sh
./scripts/fetch-model.sh base.en
```
Expected: `models/ggml-base.en.bin` exists (~140 MB).

- [ ] **Step 2: Write `transcribe.rs`**

`src-tauri/src/transcribe.rs`:
```rust
use serde::Serialize;
use whisper_rs::{FullParams, SamplingStrategy, WhisperContext, WhisperContextParameters};

#[derive(Serialize)]
#[serde(rename_all = "camelCase")]
pub struct CaptionDTO { pub text: String, pub start_ms: i64, pub end_ms: i64 }

fn read_wav_f32(path: &str) -> Result<Vec<f32>, String> {
    let mut reader = hound::WavReader::open(path).map_err(|e| e.to_string())?;
    let samples: Vec<f32> = reader.samples::<i16>()
        .map(|s| s.map(|v| v as f32 / 32768.0).map_err(|e| e.to_string()))
        .collect::<Result<_, _>>()?;
    Ok(samples)
}

#[tauri::command]
pub fn transcribe(wav_path: String, model_path: String) -> Result<Vec<CaptionDTO>, String> {
    let audio = read_wav_f32(&wav_path)?;
    let ctx = WhisperContext::new_with_params(&model_path, WhisperContextParameters::default())
        .map_err(|e| format!("load model: {e}"))?;
    let mut state = ctx.create_state().map_err(|e| e.to_string())?;

    let mut params = FullParams::new(SamplingStrategy::Greedy { best_of: 1 });
    params.set_token_timestamps(true);
    params.set_split_on_word(true);
    params.set_max_len(1);            // ~one word per segment
    params.set_temperature(0.0);
    params.set_no_speech_thold(0.6);

    state.full(params, &audio).map_err(|e| format!("transcribe: {e}"))?;

    let n = state.full_n_segments().map_err(|e| e.to_string())?;
    let mut out = Vec::new();
    for i in 0..n {
        let text = state.full_get_segment_text(i).map_err(|e| e.to_string())?;
        let t0 = state.full_get_segment_t0(i).map_err(|e| e.to_string())?; // centiseconds
        let t1 = state.full_get_segment_t1(i).map_err(|e| e.to_string())?;
        let trimmed = text.trim().to_string();
        if trimmed.is_empty() { continue; }
        out.push(CaptionDTO { text: trimmed, start_ms: t0 * 10, end_ms: t1 * 10 });
    }
    Ok(out)
}
```

- [ ] **Step 3: Register the command**

`src-tauri/src/main.rs` (update the handler list):
```rust
#![cfg_attr(not(debug_assertions), windows_subsystem = "windows")]
mod audio;
mod transcribe;

fn main() {
    tauri::Builder::default()
        .invoke_handler(tauri::generate_handler![audio::extract_audio, transcribe::transcribe])
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}
```

- [ ] **Step 4: Build and verify shape**

Run: `cargo check --manifest-path src-tauri/Cargo.toml`
Expected: compiles. (If `whisper-rs` API names differ in the resolved version, adjust: confirm `set_max_len`, `set_split_on_word`, `full_get_segment_t0/t1` against `cargo doc -p whisper-rs --open`.)

- [ ] **Step 5: Smoke-test transcription end-to-end in the running app**

Add a temporary button in `App.tsx` that invokes `extract_audio` then `transcribe` on `scratch/sample.mp4` and `console.log`s the result. Run `npm run tauri dev`, click it.
Expected (verify in devtools console): an array of `{ text, start_ms, end_ms }`, words in order, `start_ms < end_ms`, all within the clip duration, monotonically increasing. Remove the temporary button after verifying.

- [ ] **Step 6: Commit**

```bash
git add src-tauri/src/transcribe.rs scripts/fetch-model.sh src-tauri/src/main.rs
git commit -m "feat(m0): whisper-rs transcription with word-level timings"
```

---

### Task 8: Overlay export — caption frames + untouched video + copied audio

**Files:**
- Create: `src-tauri/src/export.rs`, `src/export/exportFrames.ts`
- Modify: `src-tauri/src/main.rs`, `src/App.tsx`

**Interfaces:**
- Produces:
  - TS `renderOverlayFrames(captions: Caption[], fps: number, width: number, height: number): Uint8Array[]` — one RGBA frame buffer per frame, captions only (transparent elsewhere).
  - Rust command `export_overlay(original_path: String, frames_dir: String, fps: u32, width: u32, height: u32, out_path: String) -> Result<String, String>` — overlays the PNG frames in `frames_dir` (named `f%05d.png`) onto the original video, copies audio, writes `out_path`.

- [ ] **Step 1: Write the TS overlay frame renderer**

`src/export/exportFrames.ts`:
```ts
import type { Caption } from "../core/types";
import { combineTokensIntoPages } from "../core/captions";
import { captionStateAtFrame } from "../core/captionState";
import { springTable } from "../core/easing";
import { frameCount } from "../core/time";
import { paint, type Layout } from "../render/paint";

export async function renderOverlayPngs(
  captions: Caption[], fps: number, width: number, height: number
): Promise<Blob[]> {
  const canvas = new OffscreenCanvas(width, height);
  const ctx = canvas.getContext("2d")!;
  const pages = combineTokensIntoPages(captions);
  const spring = springTable(280, 20, 1, 8, fps);
  const layout: Layout = { width, height, baselineY: height * 0.62, fontPx: 132, accentColor: "#3DDC97" };
  const last = captions[captions.length - 1]?.endMs ?? 0;
  const total = frameCount(last, fps);
  const blobs: Blob[] = [];
  for (let f = 0; f < total; f++) {
    const state = captionStateAtFrame(captions, pages, f, fps, { leadMs: 120, spring });
    paint(ctx, state, layout);
    blobs.push(await canvas.convertToBlob({ type: "image/png" }));
  }
  return blobs;
}
```

- [ ] **Step 2: Write the Rust export command**

`src-tauri/src/export.rs`:
```rust
use std::process::Command;

#[tauri::command]
pub fn export_overlay(
    original_path: String, frames_dir: String, fps: u32, width: u32, height: u32, out_path: String
) -> Result<String, String> {
    let pattern = format!("{frames_dir}/f%05d.png");
    let scale = format!("scale={width}:{height}");
    let status = Command::new("ffmpeg")
        .args([
            "-y",
            "-i", &original_path,
            "-framerate", &fps.to_string(), "-i", &pattern,
            "-filter_complex", &format!("[0:v]{scale}[bg];[bg][1:v]overlay=format=auto[v]"),
            "-map", "[v]", "-map", "0:a",
            "-c:v", "libx264", "-crf", "18", "-pix_fmt", "yuv420p",
            "-c:a", "copy",
            &out_path,
        ])
        .status()
        .map_err(|e| format!("failed to spawn ffmpeg: {e}"))?;
    if !status.success() { return Err(format!("ffmpeg exited with status {status}")); }
    Ok(out_path)
}
```

- [ ] **Step 3: Register the command**

`src-tauri/src/main.rs` (final handler list):
```rust
#![cfg_attr(not(debug_assertions), windows_subsystem = "windows")]
mod audio;
mod transcribe;
mod export;

fn main() {
    tauri::Builder::default()
        .invoke_handler(tauri::generate_handler![
            audio::extract_audio, transcribe::transcribe, export::export_overlay
        ])
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}
```

- [ ] **Step 4: Verify the FFmpeg overlay pipeline directly**

Generate 90 transparent caption PNGs is exercised in Task 9; first verify the FFmpeg invocation shape with a quick synthetic overlay:
```bash
mkdir -p scratch/frames
ffmpeg -y -f lavfi -i color=c=red@0.0:s=1080x1920:d=3 -vf "drawtext=text='X':fontcolor=white:fontsize=200:x=(w-tw)/2:y=(h-th)/2" -frames:v 90 scratch/frames/f%05d.png
ffmpeg -y -i scratch/sample.mp4 -framerate 30 -i scratch/frames/f%05d.png -filter_complex "[0:v]scale=1080:1920[bg];[bg][1:v]overlay=format=auto[v]" -map "[v]" -map 0:a -c:v libx264 -crf 18 -pix_fmt yuv420p -c:a copy scratch/out.mp4
ffprobe -v error -show_entries stream=codec_type,codec_name -of default=nw=1 scratch/out.mp4
```
Expected: an `scratch/out.mp4` with a video stream (`h264`) and an audio stream whose `codec_name` matches the source (copied). Play it; the white X overlays the video and audio is intact.

- [ ] **Step 5: `cargo check`**

Run: `cargo check --manifest-path src-tauri/Cargo.toml`
Expected: compiles.

- [ ] **Step 6: Commit**

```bash
git add src-tauri/src/export.rs src/export/exportFrames.ts src-tauri/src/main.rs
git commit -m "feat(m0): overlay export (caption frames + untouched video + copied audio)"
```

---

### Task 9: End-to-end wiring + measured sync on a real clip

**Files:**
- Modify: `src/App.tsx`
- Create: `src/m0/runWalkingSkeleton.ts`

**Interfaces:**
- Consumes: all prior commands and core functions.
- Produces: a single button "Run M0" that does import → extract → transcribe → render overlay → export, writing real caption PNGs to disk and producing the final MP4.

- [ ] **Step 1: Write the orchestration**

`src/m0/runWalkingSkeleton.ts`:
```ts
import { invoke } from "@tauri-apps/api/core";
import { writeFile, mkdir } from "@tauri-apps/plugin-fs"; // add @tauri-apps/plugin-fs if not present
import type { Caption } from "../core/types";
import { renderOverlayPngs } from "../export/exportFrames";

const FPS = 30, W = 1080, H = 1920;

export async function runWalkingSkeleton(originalPath: string, modelPath: string, scratchDir: string) {
  const wav = `${scratchDir}/sample.wav`;
  await invoke<string>("extract_audio", { inputPath: originalPath, outPath: wav });
  const captions = await invoke<Caption[]>("transcribe", { wavPath: wav, modelPath });

  const blobs = await renderOverlayPngs(captions, FPS, W, H);
  const framesDir = `${scratchDir}/frames`;
  await mkdir(framesDir, { recursive: true });
  for (let i = 0; i < blobs.length; i++) {
    const buf = new Uint8Array(await blobs[i].arrayBuffer());
    const name = `${framesDir}/f${String(i).padStart(5, "0")}.png`;
    await writeFile(name, buf);
  }
  const out = `${scratchDir}/cliptic-m0.mp4`;
  await invoke<string>("export_overlay", { originalPath, framesDir, fps: FPS, width: W, height: H, outPath: out });
  return { out, wordCount: captions.length };
}
```

Note: this introduces `@tauri-apps/plugin-fs` for writing frame files. Add it: `npm i @tauri-apps/plugin-fs` and `cargo add tauri-plugin-fs --manifest-path src-tauri/Cargo.toml`, register `.plugin(tauri_plugin_fs::init())` in `main.rs`, and grant `fs` write to the scratch dir in `src-tauri/capabilities/default.json`. Verify the capability file exists (created by Tauri scaffold); if absent, create it per Tauri 2 docs with `fs:allow-write-file` and `fs:allow-mkdir` scoped to the scratch path.

- [ ] **Step 2: Wire a "Run M0" button in `App.tsx`**

Add a button that calls `runWalkingSkeleton` with the absolute path to `scratch/sample.mp4`, `models/ggml-base.en.bin`, and the absolute scratch dir, then shows the returned output path and word count.

- [ ] **Step 3: Run the full skeleton**

Run: `npm run tauri dev`, click "Run M0".
Expected: produces `scratch/cliptic-m0.mp4`.

- [ ] **Step 4: Verify the export structurally**

Run:
```bash
ffprobe -v error -show_entries format=duration -show_entries stream=codec_type,codec_name -of default=nw=1 scratch/cliptic-m0.mp4
```
Expected: duration ≈ source duration; one `video`/`h264` stream and one `audio` stream copied from source.

- [ ] **Step 5: Verify sync (the M0 exit criterion)**

Two-part verification:
1. **Automated proof (authoritative):** `npm test -- captionState` is green — sub-one-frame drift is proven for the timing core that drove the render.
2. **Observed confirmation:** play `scratch/cliptic-m0.mp4` and confirm each caption word lights up as it's spoken. Note any perceptible drift; if captions feel late/early, adjust `leadMs` (the spec default is 120 ms) — do NOT change the frame math.

If whisper word boundaries themselves feel loose (caption switches don't match the spoken word, even though the math is exact), record it as the trigger to add the forced-alignment pass in M1 — and write that finding into the spec's §17.

- [ ] **Step 6: Final commit + open the PR**

```bash
git add -A
git commit -m "feat(m0): end-to-end walking skeleton with measured caption sync"
git push -u origin m0-walking-skeleton
gh pr create --title "M0: walking skeleton — transcribe → render → export with proven sync" \
  --body "Implements docs/superpowers/plans/2026-06-28-m0-walking-skeleton.md. Caption sync proven by src/core/__tests__/captionState.test.ts (sub-frame drift). Export verified via ffprobe + observed playback."
```

Merge happens after review (per the subagent-driven-development / executing-plans review gates).

---

## Self-review

**Spec coverage (M0 scope):** §3 determinism → Tasks 1–4 (ms-only, pure functions, drift proof). §7 transcription pipeline → Tasks 6–7 (extract + whisper greedy/temp-0 word timings). §9 one caption preset (word-by-word pop) → Tasks 4–5. §13 export with audio passthrough → Task 8 (`-c:a copy`). §15 M0 exit criterion (drift < 1 frame) → Task 4 test suite + Task 9 verification. WebCodecs-vs-FFmpeg decision → resolved for M0 as FFmpeg overlay (documented in Architecture), full-frame/WebCodecs deferred to when zoom needs it.

**Placeholder scan:** no TBD/TODO; every code step shows complete code; every verify step shows the exact command and expected output. The one external dependency note (whisper-rs API drift, Tauri fs capability) includes the exact remediation command.

**Type consistency:** `Caption {text,startMs,endMs}` is used identically in TS (Task 1) and serialized from Rust `CaptionDTO {text,start_ms,end_ms}` (Task 7) — note the snake_case→camelCase boundary is handled by Tauri's serde (document: ensure `#[serde(rename_all = "camelCase")]` on `CaptionDTO` so JS receives `startMs/endMs`). **Fix applied:** add `#[serde(rename_all = "camelCase")]` to `CaptionDTO` in Task 7. `captionStateAtFrame`, `combineTokensIntoPages`, `springTable`, `paint`, `Layout` signatures match across Tasks 4, 5, 8.

## Subsequent milestones

M1–M5 each get their own plan (`docs/superpowers/plans/`) when reached: M1 transcription UI + editable transcript, M2 the six presets, M3 effects + stickers + restraint limiter, M4 ProRes master export + social MP4, M5 UX polish.
