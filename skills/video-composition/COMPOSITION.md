# Composition — HyperFrames Patterns

The structural patterns for assembling images, video clips, and audio tracks into a single HyperFrames composition. Validated in pilot and short-02.

---

## Composition root

```html
<div
  id="root"
  data-composition-id="main"
  data-start="0"
  data-duration="22.5"
  data-width="1080"
  data-height="1920"
>
  <!-- everything goes inside here -->
</div>
```

- `data-composition-id` — required identifier (matches the timeline registration in `window.__timelines`)
- `data-duration` — total length of the rendered MP4 in seconds (plus a small lead-out for fade-to-black)
- `data-width` / `data-height` — output resolution. For WhatsApp Stories: 1080×1920 (9:16)

---

## Segment elements: `<img>` vs `<video>`

A segment is a visual block on the timeline. Two element types depending on whether the segment is animated:

### Static segment (`<img>`)

```html
<img id="seg-3"
     class="bg-media clip"
     src="images/03-hogar.png"
     data-start="3.65"
     data-duration="3.00"
     data-track-index="0" />
```

- `class="clip"` — required for HyperFrames visibility management
- `class="bg-media"` — shared CSS class (background image cover, see CSS section below)
- `data-track-index` — alternates 0/1 across adjacent segments so cross-fades work

### Animated segment (`<video>`)

```html
<video id="seg-4"
       class="bg-media clip"
       src="videos/04-reflexiva.mp4"
       muted playsinline
       data-start="6.65"
       data-duration="4.10"
       data-track-index="1"></video>
```

- `muted` — **required.** HyperFrames manages audio in `<audio>` tags; a sounding video would compete with the voiceover.
- `playsinline` — required so the video plays inline (not full-screen) on mobile preview
- `data-duration` may be **shorter than the video file's actual length** — HyperFrames trims the end. This is how you fit a 5s Wan video into a 4s segment.

### Tracks alternating 0/1

For a sequence of segments, alternate `data-track-index` between 0 and 1 so adjacent segments can overlap their fade frames without lint errors:

```
Seg 1  track=0
Seg 2  track=1   ← can overlap Seg 1 by 0.2s for cross-fade
Seg 3  track=0
Seg 4  track=1
…
```

Captions go on track 2. Watermark on track 3. CTA URL on track 4. Fade-to-black on track 5. Audio tracks at 10+. Tracks are identifiers, not z-order — the visual stacking comes from element CSS `z-index`.

---

## Background CSS — the `.bg-media` class

```css
.bg-media {
  position: absolute;
  inset: 0;
  width: 100%;
  height: 100%;
  object-fit: cover;
  opacity: 0;          /* GSAP fades it in */
}
```

`object-fit: cover` handles the few-pixel mismatch between Wan 2.6 outputs (1076×1926) and the composition canvas (1080×1920) — the video is up-sampled invisibly.

---

## Cross-fade pattern (GSAP)

Each segment's GSAP block:

```javascript
tl.set("#seg-3", { opacity: 0 }, 0)                                      // initial: hidden
  .to("#seg-3", { opacity: 1, duration: 0.2, ease: "power1.out" }, 3.65) // fade in at start
  .to("#seg-3", { opacity: 0, duration: 0.2, ease: "power1.in" }, 6.45); // fade out before next
```

`CROSSFADE = 0.2` seconds is the validated value. Shorter (0.1s) feels jumpy; longer (0.4s) blurs the message.

For adjacent segments to fade smoothly, they should **overlap in time** by `CROSSFADE`:

```
Seg 3: data-start=3.65, data-duration=3.00 → ends at 6.65
Seg 4: data-start=6.65, data-duration=4.10
```

Both visible (one fading out, one fading in) at t=6.65. Their `track-index` values are different (0 and 1) so the lint doesn't reject them.

---

## Audio tracks

Three to five `<audio>` elements per composition, each on its own `data-track-index`:

```html
<audio id="vo"     src="voiceover.mp3"      data-start="0"     data-duration="20.90" data-volume="1.0"  data-track-index="10"></audio>
<audio id="mus"    src="music.mp3"          data-start="0"     data-duration="21.50" data-volume="0.30" data-track-index="11"></audio>
<audio id="sfx-hb" src="sfx-heartbeat.mp3"  data-start="6.66"  data-duration="1.00"  data-volume="0.7"  data-track-index="12"></audio>
```

### Volume guidelines

| Layer | `data-volume` | Comment |
|---|---|---|
| VO (voiceover) | `1.0` | Reference level |
| Music (ducked) | `0.30` (≈ -12dB) | Validated balance — see `video-asset-generation/MUSIC-SFX.md` |
| SFX (each) | `0.5–0.7` | Per-cue, depends on the SFX's natural punch |

### Tracks must be unique per audio element

If two `<audio>` elements share the same `data-track-index`, the lint flags `overlapping_clips_same_track`. Always one audio per track. Use 10, 11, 12, … for clarity (skip the visual track range).

### Voice entry timing

A voiceover starting at `data-start="0"` sounds like the file began mid-sentence. Give it
**750–900ms of runway** so the music and first frame establish before anyone speaks — set the
VO's `data-start` to that value instead of `0`, rather than trimming silence into the MP3
itself. Keeping it as a `data-start` offset (not baked into the file) keeps the delay visible
and adjustable in the mix graph.

**Detecting the actual entry point, if it isn't already known:** the obvious check — "first
window whose mean volume exceeds base + N dB" — fires on the music ramping in, not on the
voice (measured: reported entry at 0.20s on a mix whose voice demonstrably started at 0.85s).
Measure the **jump between consecutive windows** instead — a voice entering over a steady bed
is a step, not a level; music fading up is a slope, and a slope never produces the largest
single-step delta:

```
for t in 0.0 .. 1.3 step 0.1:
    v[t] = mean_volume of the 100ms window at t
entry = argmax( v[t] - v[t-1] )
```

On the same file this reported 0.80s with a +14.2 dB jump — unambiguous. Worth running as a
gate on every mix — see [VERIFICATION.md](VERIFICATION.md).

---

## Build script structure (Node)

The validated pattern: a `build-composition.mjs` that reads the Scribe transcript and a `SEGMENTS` array, then emits `index.html`. Skeleton:

```javascript
import fs from "node:fs";

// CONFIG
const W = 1080, H = 1920;
const TOTAL_DURATION = 22.5;
const TRANSCRIPT = "voiceover-scribe.json";
const OUT = "index.html";

const SEGMENTS = [
  { id: "seg-1", type: "img",   file: "images/01-x.png", start: 0.00, duration: 1.85, track: 0 },
  { id: "seg-2", type: "video", file: "videos/02-y.mp4", start: 1.65, duration: 2.20, track: 1 },
  // …
];

const CROSSFADE = 0.2;

const AUDIO_CUES = [
  { id: "vo",  file: "voiceover.mp3", start: 0, duration: 20.9, volume: 1.0,  track: 10 },
  { id: "mus", file: "music.mp3",     start: 0, duration: 21.5, volume: 0.30, track: 11 },
  // …
];

// Load transcript
const data = JSON.parse(fs.readFileSync(TRANSCRIPT, "utf8"));
const rawWords = data.words.filter(w => w.type === "word");

// Group words into caption lines (skip — see video-captions/KARAOKE.md for the exact algorithm)

// Generate fragments
const segmentEls = SEGMENTS.map(s => {
  if (s.type === "video") {
    return `<video id="${s.id}" class="bg-media clip" src="${s.file}" muted playsinline
                   data-start="${s.start}" data-duration="${s.duration}" data-track-index="${s.track}"></video>`;
  }
  return `<img id="${s.id}" class="bg-media clip" src="${s.file}"
              data-start="${s.start}" data-duration="${s.duration}" data-track-index="${s.track}" />`;
}).join("\n");

const segmentGsap = SEGMENTS.map(s => {
  const fadeOutStart = s.start + s.duration - CROSSFADE;
  return `tl.set("#${s.id}", { opacity: 0 }, 0)
            .to("#${s.id}", { opacity: 1, duration: ${CROSSFADE}, ease: "power1.out" }, ${s.start})
            .to("#${s.id}", { opacity: 0, duration: ${CROSSFADE}, ease: "power1.in" }, ${fadeOutStart});`;
}).join("\n");

const audioEls = AUDIO_CUES.map(a => `<audio id="${a.id}" src="${a.file}"
            data-start="${a.start}" data-duration="${a.duration}"
            data-volume="${a.volume}" data-track-index="${a.track}"></audio>`).join("\n");

// Generate full HTML (omitted for brevity; see real templates in ~/video-lab/)
const html = `<!doctype html>...`;
fs.writeFileSync(OUT, html);
```

Real complete files (run as-is):
- `~/video-lab/<topic>/<video-name>/<variant>/build-composition.mjs`
- `~/video-lab/<topic>/<video-name>/<variant>/build-composition.mjs`

---

## Trimming a Wan video segment

Wan 2.6 produces 5s minimum. Your segment may need only 3-4s. **HyperFrames trims by setting `data-duration` < the video's actual length** — only the first N seconds are shown.

Example:
- Wan video: 5s long
- Segment in composition: 4.10s
- `data-duration="4.10"` on the `<video>` element
- Result: viewer sees seconds 0-4.10 of the Wan video; seconds 4.10-5.0 are discarded

If you want a different slice (e.g., seconds 1-5 instead of 0-4):

```html
<video data-start="6.65"
       data-duration="4.0"
       data-media-start="1.0"
       ... />
```

`data-media-start` offsets which point in the source video the playback starts at. Seconds 1.0-5.0 of the Wan video play during composition seconds 6.65-10.65.

This pattern is **untested** in production but the parameter exists in HyperFrames. Use it carefully — a 5s Wan generation could theoretically feed two segments at distinct offsets, doubling the value.

---

## Lint must be 0 errors

```bash
./node_modules/.bin/hyperframes lint
```

Expected output: `0 error(s), N warning(s)`. The warnings of "track too dense" / "file too large" are tolerable and don't block render.

If errors appear, see SKILL.md's "Common errors" table or the `video-captions/TROUBLESHOOTING.md` reference.

---

## Render

### Draft (fast iteration, ~2-3 min for 22s)

```bash
./node_modules/.bin/hyperframes render --output draft.mp4 --quality draft
```

### Final (high quality, slower, but cleaner output)

```bash
prime-run ./node_modules/.bin/hyperframes render --output final.mp4 --gpu --quality high
```

`prime-run` activates the discrete NVIDIA GPU (on systems with PRIME offloading) and `--gpu` enables NVENC encoding. Together they shave ~12-20% off render time at 1080p, and the difference grows with longer videos.

For details on prime-run + GPU acceleration, see `video-captions/SKILL.md` (the GPU section there documents the benchmark from the pilot).
