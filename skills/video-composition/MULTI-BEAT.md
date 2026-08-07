# Multi-Beat Compositions with Sub-Compositions

Patterns — and a hard-won bug — for HyperFrames 0.6.x compositions built from multiple sequential "beats", each its own sub-composition embedded in a root `index.html`.

---

## The audio + multi-sub-composition freeze bug

**Symptom:** the rendered MP4 shows static visuals from 0 → ~16s, then normal animation after; audio is complete and correct. The Studio preview looks perfect — and that is exactly why it's dangerous.

**Why Studio doesn't catch it:** Studio mounts **one iframe per composition** and orchestrates them live — it never plays the single compiled document. The render compiles everything into one document and captures frame-by-frame with `beginFrame` (virtual clock). Two different worlds.

**Mechanism** (verified by bisection + inspecting the compiled document in Chrome):

- Each sub-composition that embeds its own `<script src=gsap>` creates a **distinct GSAP instance** in the compiled document (observed 6 different Timeline constructors). The runtime nester doesn't reliably adopt timelines from foreign instances; they stay un-paused, running on the virtual clock (orphans).
- **Without audio:** orphans advance in sync with captured frames (virtual rAF = 1 tick per frame) → output is correct by deterministic luck.
- **With audio:** the init's audio pipeline burns ~16s of virtual clock **before frame 0** → every orphan timeline has already finished by the time capture starts → frozen on its final state.

**Bisection matrix** (each cell = render + freezedetect): 1 beat no audio ✓ · 1 beat + audio ✓ · 5 beats no audio ✓ · 5 beats + a single `<audio>` ✗ (full pattern). Workers, streaming-encode (`PRODUCER_ENABLE_STREAMING_ENCODE=0`), quality and file order are **not** the variable — each was ruled out one by one.

---

## Production workaround (validated)

1. Back up `index.html`, strip the `<audio>` elements temporarily (regex over `<audio[^>]*></audio>`), render video-only, restore the index — all in one command with a guaranteed restore.
2. Mix the audio offline with ffmpeg, replicating the root's mix: VO `volume=1.0`; music `atrim=<lead>,asetpts=PTS-STARTPTS,volume=0.3`; each SFX with `adelay=<data-start in ms>` and its volume; then `amix=inputs=N:duration=longest:normalize=0,alimiter=limit=0.98,atrim=0:<dur>`.
3. Mux without re-encode: `ffmpeg -i video.mp4 -i mix.m4a -c:v copy -c:a copy -movflags +faststart final.mp4`.
4. Sync stays exact because the mix uses the same transcript timestamps that fed the compositions (verified frame-by-frame: the keyword lands on the frame where it's spoken).

**Prevention:** the canonical root pattern below avoids orphan timelines in the first place — prefer it over the workaround for new projects.

---

## Canonical root `index.html` for multi-beat

Derived from comparing a project that renders clean against one that doesn't, plus reading the runtime visibility code (`He()`: toggles every element carrying a `data-start` by the window `[data-start, data-start + data-duration]`).

**Root rules:**

1. **Sequential beats on ONE track**, with `data-duration` that tiles EXACTLY: each beat's end = the next beat's `data-start`. Zero overlaps, zero alternating tracks, zero z-index between hosts.
2. **Host `data-duration` == the inner template div's `data-duration`** (same number in `index.html` and in the composition file).
3. **Coverage transitions between beats live in the ROOT timeline**, as root overlay elements (sheets/wipes that cover the cut and reveal the next beat) — NOT as "cover entrances" inside the incoming beat. Pattern: `fromTo(sheet, {y:2050}, {y:0, 0.5s power2.in})` landing ON the cut, `to(sheet, {y:-2050, 0.6s power2.out})` right after, `set(sheet, {visibility:"hidden"})` at the end.
4. **The root ALWAYS registers its timeline and it is NEVER empty:** without registration the render aborts with `Composition has zero duration. Runtime ready: false`; with an empty timeline the master assembly misbehaves. The wipe tweens give it natural content.
5. Exception to rule 3: a beat whose local t=0 state covers the frame IN POSITION (e.g. stagger-blocks flying in) can self-cover its entry cut — but only with the exact tiling of rule 1.

**Anti-pattern (tried, breaks the render):** beats overlapping by 0.45s on alternating tracks + staggered z-index + internal covers per beat + empty root timeline. Looks perfect in Studio (isolated iframes); in render it produces beats visible outside their window and undirected timelines.

**Audio in the root:** every `<audio>` needs an EXPLICIT `data-duration` — without it the preview/lint treats it as "until the end" and reports false overlaps on the same track (and playback can overlap). `data-media-start` works to trim the music intro (e.g. skip 2s so the resolved ending lands on the video's close) — though pre-processing the file with ffmpeg (a pre-faded copy) is the most bulletproof.

---

## The last beat needs a tail AFTER the last word

Complements [ENDINGS.md](ENDINGS.md), which documents the fade-to-black itself. This is **why an ending still feels abrupt even when the fade is already in place** in a multi-beat build.

**The mechanism:** the natural thing is to set each beat's `data-duration` to the duration of its voiceover line. That's correct for every beat **except the last** — there the video ends exactly on the final syllable, so the fade either overlaps the last word or gets cut off. The fade wasn't missing; it had **nowhere to live**.

**The fix:** last beat `data-duration` = voiceover duration **+ ~1.0s of tail**. Every other beat keeps measuring its own line.

```
01 = 6.362   ← voice
02 = 3.344   ← voice
...
06 = 2.579   ← 1.579 voice + 1.000 tail
```

In that tail the brand sits alone on screen and the fade runs to completion. One second sounds like nothing, and it is exactly what separates a closing from a cut.

**Verify the tail actually exists** by measuring audio decay at three points, not by watching:

```
30.2 s  −18.6 dBFS   brand alone, audio still present
31.2 s  −45.7 dBFS   mid-decay
31.7 s  −56.2 dBFS   last frame, effectively silent
```

Three descending numbers = there is a tail. If the last frame is still near programme level, the video cuts off no matter what the black overlay says.

**5-second check:** `MP4 total` − `sum of voiceover durations` ≈ the tail you intended. If it comes out ~0, the last beat is glued to the last word.
