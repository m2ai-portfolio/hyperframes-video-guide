# HyperFrames Video Workflow (agent brief)

A working SOP for producing narrated, captioned videos from HTML using HyperFrames. Drop this in your project so your Claude follows the same pipeline and skips the mistakes we already paid for. Every command is `npx hyperframes ...`.

## What HyperFrames is

HyperFrames renders video **from HTML**. A "composition" is an HTML file where the DOM declares timing with `data-*` attributes, animation is one seekable (paused) GSAP timeline, and media playback is owned by the framework. The CLI seeks that timeline frame by frame in headless Chrome and encodes an MP4. Because rendering samples the timeline deterministically, the composition must be reproducible: no wall-clock, no unseeded `Math.random`, no network calls, no infinite loops.

## Prerequisites

- **Node.js 22+** and **FFmpeg** on PATH.
- **ElevenLabs** for narration (your account):
  - Voice: **"Dylan Platelle"**, voice ID `wJ4U84X4t4rzenAmCmRG`.
  - API key with TTS scope (a read-only key returns 401 on TTS). Export it in your shell env, e.g. `ELEVENLABS_API_KEY_FULL`.
  - Model `eleven_multilingual_v2`; sensible defaults: stability 0.5, similarity_boost 0.75, style 0, speaker_boost on, speed 1.0.
- `--json` is available on every command except `render` / `preview` / `play` if you want machine-readable output.

## The pipeline (idea to finished video)

**0. Storyboard first, before any HTML.** Write a `SCRIPT.md`: scene list with narration + on-screen text + capture cues. Authoring without a script drifts and reworks. On-screen text is the structural headline per beat, not the full narration.

**1. Scaffold.**
```bash
npx hyperframes init my-video                # interactive
npx hyperframes init . --non-interactive --example blank   # CI/agent mode
```
`init` refreshes the bundled skills from GitHub, keep that on. Gotcha: `init` refuses a non-empty directory, so move `SCRIPT.md` aside, init, move it back. For a video sourced from a live site, `npx hyperframes capture <url>` scaffolds from the captured page.

**2. Narration / TTS, do this EARLY (voiceover drives all timing).**
```bash
npx hyperframes auth status      # run first, confirm providers, then choose
npx hyperframes tts "clean narration text"
```
Clean the text before sending: strip em-dashes, expand abbreviations ("feat" to "feature branch"). TTS reads raw text literally. Provider chain is HeyGen then ElevenLabs then local Kokoro; set your ElevenLabs key + voice ID to route to Dylan Platelle.

**3. Author the composition HTML.** One full-frame `.scene .clip` per beat on a track, a lower-third caption per beat on its own track, audio clips on their own track. Every timed element needs `data-start`, `data-duration`, `data-track-index`, and `class="clip"`.

**4. Set timing FROM the measured voiceover.** Generate VO clips first, `ffprobe` each duration, then `scene_duration = clip + ~1.3s` (0.5s lead-in + 0.8s tail). Place each `<audio>` at `scene_start + 0.5`. Round every start and duration to a **1-decimal grid** so clip boundaries touch exactly.

**5. Music + SFX (optional).** Use a real no-copyright track, looped to length, normalized (`loudnorm=I=-19`), fade in 2s / out 3s, with an `acrossfade` at the loop seam. Music bed on its own track.

**6. Static gates, run after EVERY change (0 errors required).**
```bash
npx hyperframes lint       # missing ids, overlapping tracks, unregistered timelines
npx hyperframes validate   # headless Chrome: runtime errors + WCAG contrast
npx hyperframes inspect    # seeks timeline: text overflow, motion-intent checks
```

**7. Snapshot smoke test, REQUIRED if you use sub-compositions.** `lint`/`validate`/`inspect` evaluate each file in isolation and never mount `index.html`, so they miss cross-file mount failures. Only `snapshot` loads the project the way `render` does.
```bash
npx hyperframes snapshot --at <t1>,<t2>,<t3>   # midpoints per scene, eyeball each PNG
```

**8. Captions.** Transcribe from the RENDERED voiceover so captions are word-timed to what is actually spoken.
```bash
npx hyperframes transcribe --model small.en    # ALWAYS pass --model; the default silently translates non-English
```

**9. Preview in Studio (user-gated). Never auto-render.**
```bash
npx hyperframes preview     # opens the Studio timeline editor, the user can edit here
```
Pause and let the human approve before rendering.

**10. Render.**
```bash
npx hyperframes render --quality draft                     # iterate
npx hyperframes render --quality high --output out.mp4      # deliver
```
After exit 0, confirm the file exists and has plausible size (`[ -s out.mp4 ]`); on long renders sanity-check with `ffprobe -i out.mp4 -show_format`.

**11. Cloud render (only for long / large / 4K / batch).**
```bash
npx hyperframes lambda deploy
npx hyperframes lambda render ./my-project --width 1920 --height 1080 --wait
npx hyperframes lambda destroy
```
Stay on local `render` for the dev loop.

**12. Publish.**
```bash
npx hyperframes publish     # shareable link
```

Diagnostics: `npx hyperframes doctor --json` always exits 0, gate on the payload's `.ok` field, not the exit code.

## Golden rules (paid for in burned renders)

**Timing**
- DO let voiceover drive timing. DON'T pre-fix scene durations and trim narration to fit (clipped, rushed VO).
- DO round to a 1-decimal grid. DON'T use 2-decimal cumulative starts, they drift in float (a clip ends at `39.980000000000004`, the next starts at `39.98`, and the guard throws a false "overlapping clips" error).

**Audio**
- DO use a real looped music track, normalized, with a crossfade at the loop seam.
- DON'T synthesize a bed from lowpassed sines, it reads as rumble on real headphones and fails the moment anyone listens.
- DO measure bed level on a VO-free gap (`ffmpeg -af volumedetect`); target roughly -31 dB in gaps with VO peaking -4 to -6 dB. Never make the viewer the level tester.
- If VO sounds strained over the bed, don't just lower volume, sidechain-duck the music ~6 dB under VO and scoop 1 to 4 kHz.

**Captions**
- DO word-time captions from the rendered VO. DON'T let short captions paraphrase while the VO speaks a fuller script, the reader sees one thing and hears another.

**Visual / accessibility**
- DO use light-text-on-dark for labels/pills. DON'T use dark-text-on-gold, it is a real legibility problem AND the WCAG validator false-flags it.
- DO match capture framing: `object-fit: cover; object-position: top left` for wide content, `contain` for dialogs/modals.

**Counts (infographics / N-node diagrams)**
- DON'T trust a generator to hold an exact count. A brief said 7 nodes, the generator rendered 9, and it survived review because the image looked polished. Count rendered elements against the brief before shipping, or lock with a reference image.

**Process**
- DO storyboard first. DO run lint + validate + inspect after every change. DO snapshot-smoke any project with sub-compositions.

## Common pitfalls

- `init` refuses a non-empty dir: move files aside, init, move back.
- `transcribe` default model silently translates non-English audio: always pass `--model`.
- A standalone root must be a `<template>`-free, explicitly sized box. Tiny text piled in the top-left of a snapshot usually means a `<style>` left outside the `<template>`, or an unsized root collapsing a `100%` child.
- Sub-composition host id must exactly equal the inner template's `data-composition-id` and the `window.__timelines` key, no `-mount`/`-slot` suffix, or you get "timelines not registered after 45000ms".
- Duplicate `<video>`/`<img>` ids across assembled files render blank: prefix ids with the composition id.
- A full-screen background on the composition root can render black: put the fill on a full-bleed child (`position:absolute; inset:0`).
- `<video>`/`<audio>` must be a direct child of the host root, never inside a sub-comp `<template>`.
- Determinism bans (silent corruption lint won't catch): no render-time clocks, no unseeded random, no network, no `repeat: -1`, no animating `display`/`visibility`, animate transform aliases (`x`, `y`, `scale`, `rotation`) not layout (`width`/`top`/`left`).
- `npm run dev` is a long-running server, background it, never run it as a one-shot foreground command.
- Never auto-render. Render is user-gated, pause at preview.

## The one-line summary

Storyboard, generate the voiceover, time everything to the voiceover on a 1-decimal grid, gate every change with lint/validate/inspect (plus snapshot if you have sub-compositions), preview for a human, then render. Most of the quality comes from doing those gates every time instead of once at the end.
