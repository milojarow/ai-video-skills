---
name: video-composition
description: Assemble a final short video by composing pre-generated assets (images, video clips, audio tracks, captions) into a HyperFrames composition with watermark, CTA, and platform-aware safe-zones. Use this skill whenever the user asks to "compose video", "assemble short", "armar el short", "componer", "build the composition", "render the short", "add watermark", "watermark rotando esquinas", "fade to black", "URL typewriter", "CTA típewriter", "WhatsApp Stories", "Instagram Reels", "1080x1920", "9:16 vertical", or asks how to combine images + audio + captions into a single video. Also triggers on questions about HyperFrames composition patterns, multi-track audio mixing, when to use `<video>` vs `<img>` element, how to truncate a Wan 2.6 video clip in the composition, and platform-specific safe-zones for caption placement (WhatsApp UI consumes bottom 15%). The four reference files cover composition patterns, watermark animation, ending patterns, and per-platform constraints.
---

# Video Composition

The **assembly layer**: take pre-generated assets (from `video-asset-generation`) and stitch them into a final MP4 with HyperFrames.

> **📹 ACTIVE-SKILL MARKER:** Prefija tu reply con 📹 **solo en turnos donde el trabajo toca el dominio de `video-composition`** — composición de video — HyperFrames, layout, timeline. La **capa/proyecto da igual** (frontend, backend, n8n, script local — todos valen): lo que importa es si *este turno* toca el dominio. En turnos que NO lo tocan (typecheck, build, deploy, git ops, edición o curl de otros dominios), **omite 📹** aunque la skill se haya cargado antes en la sesión. Si otras skills activas también aplican al mismo turno, **apila sus emojis** en el prefijo.

This is the last layer before render. Captions live in `video-captions` (called from this composition). Asset generation lives in `video-asset-generation`. Setup lives in `video-edit-setup`.

---

## What this skill covers

| Topic | Reference file |
|---|---|
| Multi-segment HyperFrames composition (img + video + audio multi-track) | [COMPOSITION.md](COMPOSITION.md) |
| Watermark with 4-corner rotation + compactar/descompactar animation | [WATERMARK.md](WATERMARK.md) |
| Ending patterns: URL typewriter + fade-to-black | [ENDINGS.md](ENDINGS.md) |
| Platform constraints: safe-zones, durations, aspect ratios for WhatsApp Stories / IG Reels / FB Stories | [PLATFORMS.md](PLATFORMS.md) |
| Multi-beat compositions: canonical root pattern + the audio/sub-composition freeze bug + workaround | [MULTI-BEAT.md](MULTI-BEAT.md) |
| Verifying a render before delivery (freezedetect, contact sheet, `hyperframes snapshot`) | [VERIFICATION.md](VERIFICATION.md) |
| Website-to-video pipeline: capture → beats fan-out → assembly (transcribe, fonts, sub-agent contract) | [WEBSITE-PIPELINE.md](WEBSITE-PIPELINE.md) |

---

## Stack

- **HyperFrames** (`hyperframes` npm package) — the composition + render engine. Composition is plain HTML with `data-*` timing attributes + a GSAP timeline.
- **GSAP 3.14.x** — drives every motion (image fades, watermark animation, caption transitions, URL typewriter, fade-to-black).
- **Node.js 22 LTS** — runs the build script that generates `index.html` from a JSON description of the composition.
- **FFmpeg** — invoked by HyperFrames internally during render.

For setup, see `video-edit-setup`.

---

## Workflow at the assembly layer

By the time you reach this skill, the assets exist:

```
~/video-lab/<topic>/<video-name>/<variant>/
├── voiceover.mp3              # ElevenLabs TTS
├── voiceover-scribe.json      # Scribe transcript with word timestamps
├── music.mp3                  # ElevenLabs Music
├── sfx-*.mp3                  # ElevenLabs SFX (per cue)
├── images/                    # 6-8 PNG from kie.ai
│   └── 0N-*.png
├── videos/                    # 0-N MP4 from Wan 2.6 (animated subset)
│   └── 0N-*.mp4
└── logo.png                   # client logo (raster, 4-channel grayscale alpha)
```

The composition step:

1. Write a `build-composition.mjs` (Node) that:
   - Reads `voiceover-scribe.json` for word-level timestamps
   - Defines a `SEGMENTS` array (one entry per visual segment, type `"img"` or `"video"`, with `start` / `duration` / `track`)
   - Defines the watermark, CTA, fade-out
   - Defines audio tracks (VO + music ducked + SFX)
   - Generates `index.html` from JSX-like template literals
2. Run `node build-composition.mjs` → `index.html`
3. `npx hyperframes lint` (must be 0 errors)
4. `npx hyperframes render --output draft.mp4 --quality draft` for iteration
5. After approval, `prime-run npx hyperframes render --output final.mp4 --gpu --quality high`

---

## Reusable templates

Two real `build-composition.mjs` files in production:

- **`~/video-lab/<topic>/<video-name>/<variant>/build-composition.mjs`** — pilot (papercut, 7 static images, no Wan animation). Cleanest baseline.
- **`~/video-lab/<topic>/<video-name>/<variant>/build-composition.mjs`** — short-02 (cinematic realism, 5 static images + 3 Wan videos, watermark rotation, fade-to-black). Most complete reference.

Copy the closer one as starting point and adapt:
- Workspace path
- `TOTAL_DURATION` (from the Scribe duration of your VO, plus ~0.5-1s lead-out)
- `SEGMENTS` (with timestamps from your Scribe transcript)
- `AUDIO_CUES` (which SFX, when)

---

## Quick checklist before render

- [ ] All asset files exist in the workspace (run `ls` to verify)
- [ ] `voiceover-scribe.json` is the latest transcription (re-run Scribe after any VO regeneration)
- [ ] `SEGMENTS` timestamps match the transcript word starts (off-by-1s ruins sync)
- [ ] `data-track-index` alternates 0/1 across adjacent segments (so cross-fades work)
- [ ] Captions safe-zone respects the target platform (see [PLATFORMS.md](PLATFORMS.md))
- [ ] Watermark rotation interval × 4 corners ≤ TOTAL_DURATION (otherwise you waste corners)
- [ ] `prime-run` available if rendering high-quality with GPU encoding (see `video-edit-setup`)

---

## Common errors

| Error | Likely cause | Fix |
|---|---|---|
| `overlapping_clips_same_track` lint error | Two clips on the same `data-track-index` overlap in time | Move one to a different track, or adjust timestamps + add a 10ms gap |
| `media_missing_data_start` lint error | A `<video>` or `<audio>` is missing `data-start="0"` | Always set `data-start` explicitly on every element |
| `gsap_exit_missing_hard_kill` lint warning | A GSAP exit tween doesn't have a follow-up `tl.set` to lock the final state | Add `tl.set("#X", { opacity: 0 }, transitionTime)` after the exit `.to(...)` |
| Audio out of sync after a script change | Forgot to re-run Scribe after regenerating the voiceover | Re-transcribe → update timestamps in build script |
| Black flash mid-video | Cross-fade overlap timing wrong (a segment ends before the next fully fades in) | Make adjacent segments overlap by `CROSSFADE` (0.2s typical) |
| Watermark text reads tiny on phone | Font size too small for 1080×1920 | Bump to ≥22px; logo height ≥60px |
| Final render is much larger MB than draft | `--quality high` produces larger files (this is correct — keep it for delivery) | Acceptable for archival/delivery; not a bug |

---

## Where to go from here

- For the actual HyperFrames composition patterns (img/video/audio elements, GSAP timeline, cross-fade): [COMPOSITION.md](COMPOSITION.md)
- For the watermark rotation animation: [WATERMARK.md](WATERMARK.md)
- For ending patterns (URL typewriter + fade-to-black): [ENDINGS.md](ENDINGS.md)
- For platform-specific safe-zones and duration limits: [PLATFORMS.md](PLATFORMS.md)
- For caption rendering (karaoke / TikTok-pop), pivot to the `video-captions` skill — its `KARAOKE.md` and `PIPELINE.md` are the prescriptive references; this composition skill calls the same patterns.
