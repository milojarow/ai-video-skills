# Website-to-Video Pipeline (end-to-end)

Notes from running a full website-to-video pipeline: site capture → DESIGN → SCRIPT → STORYBOARD → ElevenLabs VO + Scribe → parallel sub-agents building beats → root assembly → render. These cover gaps an upstream website-capture skill leaves open for HyperFrames 0.6.88.

---

## `hyperframes transcribe` needs `whisper-cpp` on PATH

If `whisper-cpp` isn't installed, ElevenLabs Scribe is a drop-in replacement (same cents-level cost): `POST /v1/speech-to-text` with `model_id=scribe_v1`, `language_code=spa` (ISO-639-3, **not** `"es"`), `tag_audio_events=false`. Convert to the pipeline's `transcript.json` format with:

```bash
jq '[.words[] | select(.type=="word") | {text,start,end}]'
```

---

## Capture paths changed for 0.6.88

Older capture-skill examples use `../capture/assets/` from `compositions/`, but the compiler serves everything root-relative → paths must be `capture/assets/` (the linter flags the old form as `invalid_capture_path`). The old install check `node_modules/sharp/build/Release/*.node` no longer applies either — modern sharp doesn't leave the `.node` there; verify with `node -e "require('sharp').versions"`.

---

## Next.js captured fonts

Capture downloads woff2 with hashed names; `capture/extracted/fonts-manifest.json` maps file → family → weight → axes. For families outside the renderer's auto-embedded list, generate `@font-face` pointing at the manifest's woff2, with `font-weight: 100 900` if it has a `wght` axis. Note: Next may name a family with a size suffix internally (e.g. "… 9pt") — the `@font-face` can declare the clean name.

---

## Sub-agent fan-out per beat

One sub-agent per beat works very well (fresh context per beat, parent only orchestrates). Each agent's contract must include:

- explicit vertical format,
- word timings **relative to its beat's start**,
- root-relative asset paths,
- the transition rule (no exits; the next beat or the root covers — see [MULTI-BEAT.md](MULTI-BEAT.md)).

On receipt, verify: grep the selectors, count `gsap.timeline(`, and confirm the file starts with `<template>` at byte 0.

---

## Animation anti-bug checklist (validated)

- Zero loose `gsap.to/from` — everything mounted on the timeline with an explicit position.
- Zero CSS `@keyframes` for content.
- One `gsap.timeline({paused:true})` per composition, registered and never overwritten.
- 100% synchronous construction — no `DOMContentLoaded` / `fonts.ready` / `setTimeout`.
- Finite repeats.
- `tl.fromTo` instead of `tl.from`.
- Never two transform tweens on the same element (wrapper for entry, child for ambient).
