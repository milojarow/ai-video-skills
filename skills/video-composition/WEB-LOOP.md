# Seamless Web Loops (ping-pong) + hero embed

For video that plays on `loop` forever — a website hero, an ambient background, a long-scene
backdrop — rather than as a timed segment inside a composition. Different problem from
`data-media-start` (which trims a clip *inside* a composition); this is about the seam where
the clip meets itself.

---

## The problem: i2v output drifts, so `loop` visibly jumps

Image-to-video models **drift**. An 8-second camera push ends on a framing nothing like frame
0. Put that in a `<video loop>` and every lap snaps back with a hard visible cut. No amount of
prompt engineering reliably returns a clip to its own first frame.

---

## The fix: ping-pong, no extra generation

Play the clip forward, then in reverse. The ends match **by construction**, not by luck:

```bash
ffmpeg -i in.mp4 -filter_complex \
  "[0:v]split[a][b];[b]reverse[r];[a][r]concat=n=2:v=1:a=0[v]" \
  -map "[v]" -an -c:v libx264 -preset slow -crf 26 -pix_fmt yuv420p \
  -movflags +faststart out.mp4
```

Two things this buys you:

1. **A truly seamless loop** — zero generation spend, zero regeneration roulette.
2. **Double the duration for free** — 8 s generated becomes 16 s of playback.

**Why it doesn't read as "video played backwards":** for slow, ambient motion (a camera push,
breathing, shifting light) the reversal reads as *a camera that breathes*. The illusion breaks
for motion with a clear arrow of time — falling objects, a person walking through frame,
liquid pouring, anything that accumulates. Prompt for reversible motion when you know the
clip is destined to loop.

Reference measurement: 8 s of 1080p → 16 s out, **~3.7 MB at crf 26**.

---

## Corollary: ship two cuts, choose in the client

~3.7 MB is too much to push at a phone. Render a 720p cut alongside the 1080p:

```bash
ffmpeg -i out.mp4 -vf scale=1280:720 -an -c:v libx264 -preset slow -crf 28 \
  -pix_fmt yuv420p -movflags +faststart out-720.mp4     # ~1.26 MB
```

Then let the client pick — and make sure it picks **before** any bytes move:

- `preload="none"` on the `<video>`.
- **No `<source>` elements in the server-rendered markup.** A `<source>` list is resolved by
  the browser immediately; a phone would start pulling the desktop cut before your JS ever
  runs.
- `poster` = frame 0, extracted with `ffmpeg -ss 0 -frames:v 1`.
- Assign `src` on mount from `matchMedia('(max-width: 767px)')`.

That ordering is the whole point: the phone never begins downloading the desktop cut before
it's known to be a phone.

---

## Two obligations for any autoplaying hero video

Both are easy to miss and both leave a visibly broken control if you do.

### `prefers-reduced-motion: reduce`

Don't autoplay. Show the poster only, and have the control offer **play** rather than
**pause**. Respecting the media query but leaving a pause button on a video that never
started is worse than ignoring it.

### Autoplay can be refused

Low-power mode, data saver, and per-site settings all reject `el.play()`. **Handle the
rejected promise:**

```js
el.play().catch(() => {
  // poster stays; flip the control to "play"
  setPlaying(false)
})
```

Without the `.catch`, the poster sits there under a dead pause button that does nothing — the
only affordance on screen is a lie about the current state.

---

## Fit with the rest of the pipeline

- Generate the source clip with either i2v provider — see the `video-asset-generation` skill
  ([ANIMATION.md](../video-asset-generation/ANIMATION.md) for kie.ai Wan 2.6,
  [OPENROUTER-VIDEO.md](../video-asset-generation/OPENROUTER-VIDEO.md) for the cheaper
  OpenRouter route).
- Ping-pong changes the economics of duration: because playback is doubled, a hero that needs
  16 s only has to *generate* 8 s. On a per-second-priced model that halves the line item
  outright.
- This is a **standalone deliverable**, not a HyperFrames render — no composition, no lint, no
  `hyperframes render`. Just ffmpeg and the markup above.
