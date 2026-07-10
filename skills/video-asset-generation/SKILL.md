---
name: video-asset-generation
description: Generate the building blocks of a short video — images, voiceovers, music, SFX, and image-to-video animation — using kie.ai (Nano Banana 2 + Wan 2.6), ElevenLabs (TTS + Music + Sound Generation), and Luna CDN for hosting. Use this skill whenever the user asks to generate any video asset programmatically — "generate image", "make a voiceover", "TTS", "create music", "add SFX", "animate this image", "Wan 2.6", "kie.ai", "ElevenLabs", "Antonio voice", "Camila voice", "Marto voice", "image-to-video", "papercut style", "cinematic realism style", "Luna CDN", "subir asset", "upload to CDN", or any combination. Also triggers on questions about pricing, voice selection, prompt patterns for these tools, character consistency between generated images, or how to avoid known anti-patterns (Mexican stereotypes in prompts, sepia tones, lip movement on silent videos). The five reference files cover each provider in depth — read the one that matches the task.
---

# Video Asset Generation

Programmatic generation of every asset needed for a short video — images, voice, music, SFX, animation — plus the CDN that hosts them when needed.

> **📹 ACTIVE-SKILL MARKER:** Prefija tu reply con 📹 **solo en turnos donde el trabajo toca el dominio de `video-asset-generation`** — generación de assets para video — render con kie.ai, prompts de imágenes-clave. La **capa/proyecto da igual** (frontend, backend, n8n, script local — todos valen): lo que importa es si *este turno* toca el dominio. En turnos que NO lo tocan (typecheck, build, deploy, git ops, edición o curl de otros dominios), **omite 📹** aunque la skill se haya cargado antes en la sesión. Si otras skills activas también aplican al mismo turno, **apila sus emojis** en el prefijo.

This skill is the **assets layer**. Composition (assembling the assets into a final video with HyperFrames) lives in `video-composition`. Captions live in `video-captions`. Setup lives in `video-edit-setup`.

---

## Providers covered

| Provider | What it does | Reference file |
|---|---|---|
| **kie.ai Nano Banana 2** | Text-to-image. Two validated styles: papercut, cinematic realism. | [IMAGES.md](IMAGES.md) |
| **ElevenLabs TTS** | Text-to-speech. Mexican-Spanish voices validated: Antonio, Camila, Marto, Jose. | [VOICES.md](VOICES.md) |
| **ElevenLabs Music** | Instrumental background music with controllable mood. | [MUSIC-SFX.md](MUSIC-SFX.md) |
| **ElevenLabs Sound Generation** | Single-purpose SFX (heartbeat, page-turn, chime). | [MUSIC-SFX.md](MUSIC-SFX.md) |
| **kie.ai Wan 2.6** | Image-to-video animation. 5/10/15s, 720p/1080p. | [ANIMATION.md](ANIMATION.md) |
| **Luna CDN** | File hosting with stable public URLs. | [LUNA-CDN.md](LUNA-CDN.md) |

---

## Decision matrix

| Need | Provider | Notes |
|---|---|---|
| Static image (background, scene, CTA card) | kie.ai Nano Banana 2 | `aspect_ratio: 9:16`, `resolution: 1K`, output PNG |
| Animated scene (motion in a static image) | kie.ai Wan 2.6 | Requires public URL of the source image, `duration: '5'`, `resolution: '1080p'` |
| Spoken narration | ElevenLabs TTS | `eleven_multilingual_v2`, `language_code: es` for Mexican Spanish |
| Background music | ElevenLabs Music | Always `force_instrumental: true` so it doesn't fight the voice |
| One-shot SFX (heartbeat, click, chime) | ElevenLabs Sound Generation | Single sound, brief (0.5-2s) |
| Public URL for an image (because Wan 2.6 needs it) | Luna CDN | Use `LUNA_API_KEY_VIDEOLAB_RAW` to preserve PNG (don't convert to webp) |

---

## Quick-start workflow (typical short)

1. **Write the script** (text file with the full voiceover).
2. **Pre-test brand name pronunciation** with the chosen voice (~$0.005, see VOICES.md). Critical step — saves $0.05+ later.
3. **Generate VO + music + SFX in parallel** (~3-10s for VO, up to ~30s for music).
4. **Transcribe the VO with Scribe** to get exact word timestamps (informs the timing of every visual segment).
5. **Generate static images with kie.ai** (one prompt per visual segment, in parallel).
   - Draft each prompt with the `ai-video-skills:scene-prompt-smith` subagent (give it the scene concept + style + protagonist clause; it applies the validated prefix and dodges the anti-patterns).
   - **QA before continuing:** dispatch one `ai-video-skills:asset-qa-validator` per generated image, in parallel (give each the image path + the prompt used + the protagonist clause). Regenerate anything it returns `REGEN` on before step 6. Worth it with several images; for a single image, just eyeball it.
6. **Decide which segments to animate with Wan 2.6** (only segments ≥4s long — see ANIMATION.md for the efficiency math).
7. **Upload images-to-be-animated to Luna CDN** (preserve PNG with the RAW key).
8. **Submit Wan 2.6 jobs in parallel**, poll to completion, download MP4s.
9. Hand off to `video-composition` for assembly.

---

## Subagents (Meeseeks)

This skill ships two project-scoped subagents (in `agents/`, namespaced `ai-video-skills:*`). They **travel with the plugin** — install it on another machine and they come along, no extra setup. Dispatch them via the Agent tool while producing a short.

| Agent | Does | How to dispatch |
|---|---|---|
| `ai-video-skills:scene-prompt-smith` | Drafts a kie.ai prompt for a scene (validated prefix, no anti-patterns) | Once — give it the scene concepts + style + protagonist clause, get the prompts back |
| `ai-video-skills:asset-qa-validator` | Inspects ONE generated image, returns `OK` / `REGEN: <reason>` | One per image, **in parallel**, after a batch is generated |

The QA validator pays off as **fan-out**: with several images, parallel validators run concurrently and keep the image binaries out of the main context. For a single image, validate inline — don't spawn.

---

## Pricing snapshot (verified 2026-05-09 / 2026-05-10)

| Item | Cost | Notes |
|---|---|---|
| kie.ai image (Nano Banana 2, 1K) | ~$0.02 | Per image |
| Wan 2.6 image-to-video (1080p, 5s) | $0.53 | 5s minimum |
| Wan 2.6 image-to-video (1080p, 10s) | $1.05 | |
| Wan 2.6 image-to-video (1080p, 15s) | $1.58 | |
| ElevenLabs TTS (per character) | varies by plan | ~$0.005 for short pre-tests |
| ElevenLabs Music (30s instrumental) | ~$0.05 | |
| ElevenLabs Sound Gen (1-2s) | ~$0.01 | |
| Luna CDN upload | $0 | Already paid via storage |

**Real cost reference** — a typical 22s short with 8 images + 3 Wan animations + VO + music + 2 SFX: **~$1.80-2.00 USD**.

---

## Anti-patterns ledger (lessons from production)

These are mistakes already made and fixed — don't repeat them:

| Anti-pattern | Why it fails | Fix |
|---|---|---|
| Prompt with "Mexican person/family/home" | Triggers Hollywood/Coco stereotype output (huipil, agave, sombrero) | Use "contemporary universal setting" + describe specific traits if needed |
| "Warm sunlight / golden hour / warm directional lighting" in image prompt | Produces a sepia filter on everything | "Natural neutral lighting" / "soft directional daylight" |
| Wan 2.6 with audio enabled + person in image | Generates lip-sync that doesn't match the external voiceover | Don't include audio param; explicitly add to prompt: "No lip movement. Subjects remain silent throughout." |
| Generate Wan video without checking script timing | If segment is <5s, you waste 33%+ of the cost | Either rewrite script segments to be ~4.5-5s, or use `data-media-start` to pick a different slice |
| Use `LUNA_API_KEY` for Wan input | Webp conversion may break providers that only accept PNG/JPG | Use `LUNA_API_KEY_VIDEOLAB_RAW` for downstream tools that need original format |
| Skip pre-test of brand-name pronunciation | Antonio likes "AcmeSeguros" together; Camila prefers "Acme Seguros" with space — assumption costs a re-render | Always pre-test 2 variants ($0.005), pick the natural one |
| Use Camila for VOs >20 seconds | Voice goes "lazy" toward the end (validated short-02) | Marto for longer VOs; Camila for shorter (<15s) ones where her warmth shines |

---

## Common errors

| Error | Cause | Fix |
|---|---|---|
| `401 Unauthorized` from kie.ai | API key not in env, or IP whitelist enabled with your IP not in the list | Check `KIE_API_KEY` env var; in kie.ai panel either remove whitelist or add your current outbound IP |
| `413 Payload Too Large` from ElevenLabs | Audio file >25MB | Extract audio with ffmpeg first: `ffmpeg -i video.mp4 -vn -acodec libmp3lame -q:a 4 audio.mp3` |
| Wan job stays in `waiting` for >10 min | Provider queue overload or bad image input | Re-submit; verify image URL is reachable from outside (curl -I should return 200) |
| Image URL `dailyportraitmag.com/...` (or similar) returns 404 | Hosting not actually public, or rate-limited | Use Luna CDN — UUIDs are stable for life and the CDN is public |
| Generated image is much smaller than peer images (e.g., 100KB vs 1.5MB) | kie.ai sometimes returns a low-res/low-quality variant | Regenerate that single prompt — usually the next attempt is normal |

---

## Reusable templates

Real-world reference projects in `~/video-lab/`:

- **Pilot (papercut)**: `~/video-lab/<topic>/<video-name>/<variant>/` — 8 papercut images, voice Antonio, all static
- **Short-02 (cinematic + Wan animation)**: `~/video-lab/<topic>/<video-name>/<variant>/` — 8 cinematic-realism images, voice Marto, 3 segments animated with Wan 2.6, watermark rotation, fade-to-black

Both have working `build-composition.mjs` scripts that generate `index.html` from the assets — copy as starting point.

---

## When in doubt

- Check the specific reference file ([IMAGES.md](IMAGES.md), [VOICES.md](VOICES.md), [ANIMATION.md](ANIMATION.md), [MUSIC-SFX.md](MUSIC-SFX.md), [LUNA-CDN.md](LUNA-CDN.md))
- For composition assembly, pivot to `video-composition` skill
- For caption styling, pivot to `video-captions` skill
- For workspace setup or Node 22 + sharp issues, pivot to `video-edit-setup` skill
