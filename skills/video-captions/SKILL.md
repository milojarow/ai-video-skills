---
name: video-captions
description: Generate word-synchronized closed captions for videos using HyperFrames + cloud STT. Two production-ready styles documented and tuned: TikTok-pop (one word visible at a time, scale-pop animation — see PIPELINE.md) and karaoke (full phrase visible, current word lights up cyan with glow + scale-up — see KARAOKE.md). Use this skill whenever the user asks for captions, subtitles, closed captions, "burn captions", "TikTok captions", "Reels-style captions", "karaoke captions", "lyrics-video style", "highlight the word being spoken", "word-by-word captions", or any flow that turns video audio into on-screen synchronized text. Also use when the user mentions OpenAI Whisper, ElevenLabs Scribe, transcribing audio for video, or hits HyperFrames lint errors like `overlapping_clips_same_track`, phantom words, or frame freezing during render. Covers the full pipeline (STT → JSON → HTML → render), provider comparison and decision criteria, both styling variants with full final config, dynamic top/bottom positioning when source video has overlays, and the common errors discovered the hard way. Assumes HyperFrames is already installed (see video-edit-setup if not).
---

# Video Captions

Generate TikTok/Reels-style word-by-word closed captions for any video using HyperFrames + cloud STT (Whisper or Scribe). One word visible at a time, animated pop-in, accurate timestamps.

> **📹 ACTIVE-SKILL MARKER:** Prefija tu reply con 📹 **solo en turnos donde el trabajo toca el dominio de `video-captions`** — captions/subtítulos — ElevenLabs TTS, sincronización, SRT/VTT. La **capa/proyecto da igual** (frontend, backend, n8n, script local — todos valen): lo que importa es si *este turno* toca el dominio. En turnos que NO lo tocan (typecheck, build, deploy, git ops, edición o curl de otros dominios), **omite 📹** aunque la skill se haya cargado antes en la sesión. Si otras skills activas también aplican al mismo turno, **apila sus emojis** en el prefijo.

**Final-delivery resolution:** PIPELINE.md's examples render at 1024×576 (the captioning experiment's source). For a social short, match the platform target instead — usually **1080×1920 (9:16)** for WhatsApp Stories / Reels — and when the captioned clip feeds a vertical composition, see `video-composition/PLATFORMS.md` for safe-zones.

---

## When to use

This skill applies whenever video gets text overlaid synced to spoken audio. Two styles are fully documented with battle-tested config:

- **TikTok-pop** — one word visible at a time, scale-pop animation when each word is spoken. Best for fast-paced social-feed content. Full implementation in [PIPELINE.md](PIPELINE.md).
- **Karaoke** — full phrase visible (3-4 words), the current word lights up cyan with a glow effect and scales up while spoken. Best for narrations where the viewer benefits from seeing what's coming next, or for music/lyrics content. Full implementation in [KARAOKE.md](KARAOKE.md).

A third style — classic Netflix-style block subs (full phrase, fade in/out) — is straightforward to derive from the karaoke build script (just remove the per-word state transitions and keep the line-level fade) but isn't separately documented yet.

The STT pipeline is the same across all styles; only the rendering layer (line grouping, span structure, GSAP timeline) differs. **Pick the style first, then jump to the matching reference file.**

---

## Step 1 (always do this first): Show the user the menu of available styles

When this skill triggers, your job before any technical work is to **present the menu of caption styles this skill knows how to produce**. Like a waiter showing a menu — you do this regardless of whether the user already had something in mind on the way in. They can't pick what they don't know exists.

### Why this is mandatory, not optional

Users don't memorize internal style names. Someone who says "ponle captions estilo TikTok" might not know the karaoke option even exists, and might prefer it once they see the description. The menu serves three jobs at once: it confirms intent, exposes capabilities the user might not have known about, and locks in the choice with short visual hints so they can pick with confidence — not by name guessing.

The cost of skipping is real: a wrong-style render is 2-3 minutes wasted plus the conversational friction of "ah, no, quería el otro". A 30-second menu pays for itself.

### The exact options to present

Use AskUserQuestion (or your harness's equivalent) with these options verbatim. Don't paraphrase — labels and descriptions have been tuned with a real user. They include enough visual hint that someone who has never seen either rendered output can pick correctly.

```
Question: "¿Qué estilo de captions quieres?"
Header:   "Estilo CC"

Options:
  1. label: "TikTok-pop — palabra por palabra (Recommended)"
     description: "Una sola palabra visible a la vez, animación scale-pop al
                   pronunciarse. Estilo Reels/TikTok. Energético, ideal para
                   contenido corto y de feed social."

  2. label: "Karaoke — frase completa, palabra activa brilla"
     description: "Toda la frase visible (3-4 palabras), la que se está
                   pronunciando se ilumina en cian con glow y crece. Ideal para
                   narraciones donde ayuda ver lo que viene, o contenido tipo
                   lyrics."

  3. label: "Block / Netflix — frase completa, fade in/out"
     description: "Bloque de 1-2 líneas abajo, aparece y se va por frase. No
                   tiene animación palabra-por-palabra. Sin documentación
                   completa todavía — derivable de karaoke pero requiere
                   adaptar el build script."
```

After the pick:
- TikTok-pop → jump to [PIPELINE.md](PIPELINE.md)
- Karaoke → jump to [KARAOKE.md](KARAOKE.md)
- Block/Netflix → tell the user this style needs custom work (tractable but not a one-shot copy-paste yet) and ask whether they want to (a) wait while you adapt the karaoke script, or (b) pick one of the documented styles instead.

### The only legit exceptions to showing the menu

Two narrow cases where you can skip:

1. **Continuation of an in-flight task in the same session.** The user already picked a style for this video, you rendered, they're now asking for tweaks ("hazlo más rápido", "cambia la fuente"). Don't re-present the menu mid-iteration.
2. **Explicit opt-out from the user.** They say something like "no me presentes opciones, usa karaoke directo" or "skip the menu, just do TikTok-pop".

Outside those two cases, **show the menu every time** — even if the user said "ponle karaoke" in the triggering message. They might change their mind seeing the descriptions, and the 30 seconds is cheap insurance against a wasted render. The menu is part of what this skill does, not a step you optimize away.

---

## The pipeline

```
source.mp4
   │
   ▼
[1] STT call ── Whisper API (curl)  ────►  transcript JSON
              └ Scribe API (curl)
   │
   ▼
[2] Build script (Node) ── reads JSON, normalizes shape, resolves
                            phantom words, applies optional time
                            shift, applies vocab fixes, generates
                            index.html with <span class="word clip">
                            elements + GSAP timeline
   │
   ▼
[3] hyperframes lint ────►  must be 0 errors (warnings about file
                             size and track density are OK)
   │
   ▼
[4] hyperframes render ──►  output.mp4 with captions burned in
```

Each step is detailed in [PIPELINE.md](PIPELINE.md) including ready-to-copy build script.

---

## Provider choice (Whisper vs Scribe)

**Default: ElevenLabs Scribe.** It produces more accurate word-level timestamps out of the box (no shift needed), and didn't emit phantom words in our test. The trade-off is cost: ~$0.04/min vs ~$0.006/min for Whisper.

**Pick Whisper when:**
- Cost dominates (large batches, or ongoing automation at scale)
- A 150ms global shift heuristic is acceptable
- You can tolerate occasional phantom-word post-processing

**Pick Scribe when:**
- Final-delivery quality matters
- Word timestamps must be precise (TikTok pop animation feels off when timestamps lag the audio)
- Per-word `logprob` confidence values are useful (Whisper doesn't expose them)

Both providers fail at proper-noun transcription — both wrote `Abuja`/`Aguja` instead of `Ahuja` for the same audio. Both need `VOCAB_FIXES` post-processing. Whisper accepts a `prompt` parameter that helps; Scribe's `biased_keywords` parameter did not help in our test. Details in [PROVIDERS.md](PROVIDERS.md).

---

## Composition rules (HyperFrames)

These are non-negotiable — break any one and either lint fails or render produces garbage:

1. **Root** has `data-composition-id`, `data-width`, `data-height`. The width/height **match the source video resolution** — don't upscale a 1024×576 source to 1920×1080 by setting larger composition dims and `object-fit: cover`. You gain nothing and lose quality.

2. **Every timed element** carries `data-start`, `data-duration`, `data-track-index`, **and** `class="clip"`. Without `class="clip"` the runtime can't manage visibility lifecycle.

3. **Video element is `muted`.** Audio goes in a separate `<audio>` element with the same source. HyperFrames mixes audio at the encoding stage; the muted flag prevents double-audio.

4. **GSAP timeline is `paused: true`** and registered on `window.__timelines["composition-id"]`. The renderer drives playback by seeking that timeline — if it's auto-playing or unregistered, captures will be wrong.

5. **No `Math.random()`, no `Date.now()`, no `fetch()` during setup.** The renderer is deterministic and seek-driven. Anything wall-clock-dependent breaks reproducibility.

6. **Clips on the same `data-track-index` cannot overlap.** Add a 10ms gap between adjacent caption words and clamp `data-duration` so `start + duration < next.start`. Floating-point overlaps (`30.001s` vs `30.000s`) read as overlap to the linter — round to 3 decimals via `toFixed(3)`.

---

## Style 1 — TikTok-pop (one word visible)

Each word is a span with its own timing and a GSAP scale-pop animation:

```html
<span id="w42" class="word bottom clip"
      data-start="3.42" data-duration="0.18" data-track-index="1">palabra</span>
```

```javascript
// Pop in: scale 0.7 → 1.05 → 1.0, opacity 0 → 1
tl.fromTo("#w42", { scale: 0.7, opacity: 0 },
                  { scale: 1.05, opacity: 1, duration: 0.04, ease: "back.out(2)" }, 3.42)
  .to("#w42", { scale: 1.0, duration: 0.04, ease: "power2.out" }, 3.46);
```

**Pop duration ≤ 80ms.** A 150ms pop visually atropella adjacent short words (Spanish hablado rápido has many sub-100ms words). 80ms hits a sweet spot: snappy enough to feel TikTok-like, short enough to not overlap with the next word's pop.

CSS for the position class is in PIPELINE.md.

## Style 2 — Karaoke (full phrase visible, current word lights up)

Each line is a `<div class="caption-line">` containing 3-4 word spans. The line is the timed clip; words inside it transition through three color states (future grey → active cyan + glow + scale → past white) driven by their own absolute timestamps in the master GSAP timeline.

Key differences from TikTok-pop:

| | TikTok-pop | Karaoke |
|---|---|---|
| Visible at once | 1 word | 3-4 words (full phrase) |
| Track elements | One span per word (~150 spans for a 1-min video) | One div per line (~30-60 lines) |
| Active-word treatment | Scale-pop animation only | Cyan + glow + scale 1.15 + thin grey outline |
| Line structure | None — words live directly in root | Words grouped by punctuation + soft/hard word caps |
| Best for | Fast social content, Reels/Shorts | Narrations, lyrics, anything where the viewer benefits from preview |

The karaoke style went through 5 rounds of real-user iteration before settling. Don't redesign — apply the values verbatim and only deviate when the user explicitly asks. The full final config (line grouping algorithm, GSAP state transitions, CSS, complete build script) is in [KARAOKE.md](KARAOKE.md).

---

## Dynamic positioning (top vs bottom)

Source videos sometimes have **lower-third overlays** (presenter name, role, channel branding). Captions fixed at `bottom: 80px` cover those overlays. Solution: a TOP_WINDOW based on transcript word timestamps that switches captions to `top: 30px` while the overlay is visible.

```javascript
// Anchor the window to known words in the transcript, not raw seconds.
// This survives video re-edits — the words still mark the boundary.
const TOP_WINDOW = { start: 0.7, end: 5.5 }; // computed from word starts
const isTopWord = (w) => w.start >= TOP_WINDOW.start && w.start <= TOP_WINDOW.end;
```

CSS:
```css
.word { position: absolute; left: 50%; transform: translateX(-50%); /* shared */ }
.word.bottom { bottom: 80px; }
.word.top    { top: 30px; }
```

**Top position must avoid the presenter's face.** For 1024×576 video with presenter centered, `top: 30px` stays clear (face starts ~y=180). For other resolutions, leave at least ~120px between the caption baseline and where the face begins.

---

## Vocabulary fixes

Both providers mistranscribe proper nouns Whisper has never seen (e.g., `Ahuja` → `Abuja`/`Aguja`). Two layers of fix:

1. **Whisper `prompt` parameter** at API call time:
   ```
   prompt="El locutor es <nombre completo del locutor>, agente de seguros..."
   ```
   The model uses this as a hint. Recovers `Ahuja` reliably.

2. **`VOCAB_FIXES` post-processing in the build script** (works for any provider):
   ```javascript
   const VOCAB_FIXES = { "aguja": "Ahuja", "abuja": "Ahuja" };
   const fixWord = (text) => {
     const lower = text.toLowerCase().replace(/[.,!?;:¿¡]/g, "");
     return VOCAB_FIXES[lower] ?? text;
   };
   ```

Use both. The prompt approach catches it at the source for Whisper; `VOCAB_FIXES` is the safety net and the only option for Scribe.

For final delivery, also add a **human-review step** of the transcript JSON before render — STT will silently swallow words ("ahorro" disappeared in Whisper, "y" disappeared in Scribe in our test), and only a human catches that.

---

## Phantom words (zero-duration tokens)

Whisper API occasionally emits words where `start == end`. Don't pass them straight to HyperFrames — they cause overlap errors and visual atropello.

**Strategy: split-forward.** The phantom word takes the first half of the *next* word's duration:

```javascript
const isPhantom = (w.end - w.start) < 0.001;
if (isPhantom && i + 1 < words.length) {
  const next = words[i + 1];
  const nextDur = next.end - next.start;
  if (nextDur >= 0.1) {
    const mid = next.start + nextDur / 2;
    resolved.push({ word: w.word, start: w.start, end: mid });
    work[i + 1] = { ...next, start: mid };
  } else {
    // not enough next-word time to split: merge text
    resolved.push({ word: `${w.word} ${next.word}`, start: w.start, end: next.end });
    i++;
  }
}
```

Scribe didn't emit phantom words in our test. Whisper emitted ~10 in a 51s clip.

---

## Source video preprocessing

If you see this warning during render:

```
Video has sparse keyframes (max interval: 8.33s). This causes seek failures and frame freezing.
```

The source `.mp4` has keyframes too far apart for seek-driven render. Fix before rendering:

```bash
ffmpeg -i source.mp4 \
    -c:v libx264 -r 30 -g 30 -keyint_min 30 \
    -movflags +faststart -c:a copy \
    source.fixed.mp4
```

`-g 30 -keyint_min 30` forces a keyframe every 30 frames (1s @ 30fps). Recommended for all sources before they enter the pipeline; the warning is informational but the freezing it predicts is real.

---

## Performance: prime-run + --gpu (NVIDIA hosts)

On systems with PRIME offloading (laptops with dual GPU integrated + NVIDIA), prefix the render with `prime-run` and add `--gpu`:

```bash
prime-run ./node_modules/.bin/hyperframes render --gpu --output out.mp4 --quality high
```

`prime-run` makes Chrome and FFmpeg see the discrete NVIDIA GPU. `--gpu` activates NVENC for hardware H.264 encoding.

Speedup is workload-dependent:
- Draft renders (`--quality draft`): ~12% (modest — ~88% of time is serial frame seek)
- Final renders (`--quality high`, x264 slow preset): substantial — encoding dominates
- WebGL/Three.js compositions (future Phase 4): notable
- Long videos: scales linearly with duration

NVENC produces files ~6% larger than x264 ultrafast at draft for the same nominal CRF. Not a quality loss — different bit allocation. Use `--video-bitrate` if size matters.

**When to adopt:** always for `--quality high`, optional for draft iteration.

---

## Common errors

Quick lookup of every error we hit during the experiment:

- `overlapping_clips_same_track` → 10ms GAP between clips, `toFixed(3)` on timestamps
- `media_missing_data_start` → add `data-start="0"` on every `<video>` and `<audio>`
- Sharp install fails → see `video-edit-setup` skill
- Captions appear late → Whisper-only, apply `SHIFT = -0.15` global
- Captions cover lower-third overlay → dynamic `top` class with TOP_WINDOW
- Pop animation atropello → cap pop duration at 80ms
- Frame freezing in render → re-encode source with `-g 30 -keyint_min 30`
- "Aguja"/"Abuja" instead of "Ahuja" → `VOCAB_FIXES` map + Whisper `prompt` param
- Whisper MCP returns text only (no timestamps) → use curl directly with `response_format=verbose_json` and `timestamp_granularities[]=word`
- ffmpeg times out burning several caption cards via chained `overlay` filters → build one alpha-channel (`yuva420p`) video track and apply a single overlay instead

Full diagnostic recipes in [TROUBLESHOOTING.md](TROUBLESHOOTING.md).

---

## Reference files

For depth beyond this overview:

- **[PIPELINE.md](PIPELINE.md)** — TikTok-pop variant: full build script (Node) ready to adapt, HTML template, lint and render commands. Copy-paste starting point for one-word-at-a-time captions.
- **[KARAOKE.md](KARAOKE.md)** — Karaoke variant: full build script with line grouping algorithm, GSAP per-word state transitions, the final config that survived 5 rounds of user iteration, and karaoke-specific pitfalls table. **Apply verbatim** unless the user requests deviation.
- **[PROVIDERS.md](PROVIDERS.md)** — Whisper vs Scribe deep dive. Endpoints, parameters, response shapes, decision matrix, known quirks per provider. Karaoke benefits more from Scribe's accurate timestamps than TikTok-pop does — pick Scribe for karaoke unless cost forces Whisper.
- **[TROUBLESHOOTING.md](TROUBLESHOOTING.md)** — generic caption errors with cause + fix. Look here first when something breaks across either style.

---

## Workflow checklist

When tasked with adding captions to a video:

1. **Confirm style with the user** — see "Step 1" section above for the exact menu to present. Don't pick yourself unless the user already named one explicitly.
2. **Pre-flight**: source video has dense keyframes? If not, re-encode with `-g 30`.
3. **Workspace**: `~/video-lab/<topic>/<video-name>/<variant>/` exists with HyperFrames installed.
4. **Scaffold**: `hyperframes init <variant> --video <source> --example blank --non-interactive`.
5. **Transcribe**: curl directly to chosen provider's STT endpoint with word-level timestamps. Pass relevant proper nouns via `prompt` (Whisper). For karaoke, prefer Scribe — the accurate timestamps matter more.
6. **Build**: Use the matching reference's complete build script:
   - TikTok-pop → [PIPELINE.md](PIPELINE.md)
   - Karaoke → [KARAOKE.md](KARAOKE.md)
   The script reads the JSON, applies VOCAB_FIXES, resolves phantoms, applies SHIFT if Whisper, and generates `index.html`.
7. **Lint**: `hyperframes lint` — must be 0 errors. Fix any `overlapping_clips_same_track` by tuning gap or `toFixed`.
8. **Render**: `hyperframes render --output <name>.mp4 --quality draft` first (fast iteration), then `--quality high` for delivery (use `prime-run` + `--gpu` here).
9. **Review**: open the MP4. Check sync, vocab fixes landed, no atropello, overlay window respected, style-specific concerns (active word visible above neighbors for karaoke; pop animation crisp for TikTok-pop). Iterate if needed.

---

## Out of scope (future phases)

This sub-skill covers Phase 1: closed captions only. Future phases will be documented in their own sub-skills:

- **Phase 2** — Audio/music: background music with ducking under voice
- **Phase 3** — B-roll / image overlays during narration (kie.ai or other providers)
- **Phase 4** — Motion effects: zoom in/out, slow/fast motion, scene transitions

Some Phase 4 effects are pure GSAP (zoom = scale animation on a track). Others use HyperFrames blocks from `/catalog/blocks` (shaders, transitions). When that phase arrives, this skill will reference its sibling.
