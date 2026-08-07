# Endings — URL Typewriter + Fade-to-Black

The closing 3-5 seconds of a short are where the call-to-action lives. Two patterns combine to produce a professional ending: an animated URL appearing on top of the final scene, and a clean fade to black.

Both validated in pilot and short-02.

---

## Pattern 1 — URL typewriter (CTA)

The brand URL appears character-by-character, synchronized with the voiceover saying it. Feels deliberate, retro-modern, and emphasizes the URL as the action item.

### HTML

```html
<div id="cta-url" class="clip"
     data-start="18.35"
     data-duration="3.15"
     data-track-index="4">
  <span id="cta-text"></span><span id="cta-cursor">|</span>
</div>
```

- `data-track-index="4"` — separate from the watermark (track 3) and segments (0/1)
- `data-duration` extends to the end of the composition (covers the fade-to-black)

### CSS

```css
#cta-url {
  position: absolute;
  top: 60%;
  left: 50%;
  transform: translate(-50%, -50%);
  font-family: system-ui, -apple-system, sans-serif;
  font-weight: 700;
  font-size: 72px;
  color: #f7f5f0;
  letter-spacing: 0.5px;
  text-shadow:
    -2px -2px 0 #1B3A5C, 2px -2px 0 #1B3A5C,
    -2px  2px 0 #1B3A5C, 2px  2px 0 #1B3A5C,
    0 0 16px rgba(0,0,0,0.7);
  white-space: nowrap;
}

#cta-cursor {
  animation: blink 0.6s infinite;
}

@keyframes blink {
  50% { opacity: 0; }
}
```

The text-shadow uses `#1B3A5C` (deep-blue brand color) as the outline — gives the white text a branded edge. Plus a soft black shadow for legibility.

The cursor has a CSS-only blink (no GSAP needed for that).

### GSAP — typewriter sequence

```javascript
const URL_TEXT = "acmeseguros.com";
const URL_TYPE_DURATION = 1.5;            // total time to type all chars
const URL_TYPE_BEGIN = 18.60;             // ⚠️ bound to the CURRENT voice — derive, don't type

const urlChars = URL_TEXT.split("");
const charDur = URL_TYPE_DURATION / urlChars.length;

urlChars.forEach((c, i) => {
  const t = URL_TYPE_BEGIN + i * charDur;
  // Update the visible text at each tick
  tl.call(() => {
    document.getElementById("cta-text").textContent = URL_TEXT.slice(0, i + 1);
  }, null, t);
});
```

Each character is appended in sequence. `tl.call(...)` registers a function call at a specific timeline time — when the renderer seeks to that time, the function runs and updates the DOM.

### Tuning

| Knob | Default | Notes |
|---|---|---|
| `URL_TYPE_DURATION` | 1.5s | Should match the voiceover's pronunciation speed of the URL. Speed up (1.2s) if the voice says it fast |
| `URL_TYPE_BEGIN` | sync with the VO timestamp where the brand/URL is spoken | **Derive it from the word's onset in the TTS alignment — don't type the second.** A hand-typed value silently breaks on any voice change; see `MULTI-BEAT.md`. |
| Font size | 72px | Big enough for phone screens but not overwhelming. 60-80px range |
| Top position | 60% (slightly below center) | Avoids covering the bottom-left/right where watermark may be |

### Why character-by-character (not just fade-in)

A character-by-character reveal makes the URL feel like it's *being told to the viewer*. A fade-in feels passive ("look at this URL"). The typewriter feels active ("type this in"). Higher conversion in user research for CTA elements (general industry wisdom; not measured in this project).

---

## Pattern 2 — Fade-to-black ending

The composition ends with a 1-second fade to black. This prevents the abrupt cut that feels amateur and creates a clean visual full-stop.

### HTML

```html
<div id="fade-out" class="clip"
     data-start="20.5"
     data-duration="1.0"
     data-track-index="5"></div>
```

- `data-start = TOTAL_DURATION - 1.0` (the fade starts 1 second before the end)
- `data-track-index="5"` — separate from everything else
- The element exists only during the fade (no need to be present earlier)

### CSS

```css
#fade-out {
  position: absolute;
  inset: 0;
  background: #000;
  z-index: 200;             /* above everything: segments, captions, watermark, URL */
  opacity: 0;               /* GSAP animates this to 1 */
  pointer-events: none;
}
```

`z-index: 200` is way above any other element (watermark at 50, captions ~100). The black overlay covers everything as it fades in.

### GSAP

```javascript
const FADE_OUT_DURATION = 1.0;
const FADE_OUT_START = TOTAL_DURATION - FADE_OUT_DURATION;

tl.set("#fade-out", { opacity: 0 }, 0)
  .to("#fade-out", { opacity: 1, duration: FADE_OUT_DURATION, ease: "power2.in" }, FADE_OUT_START);
```

`power2.in` is a slow-start, fast-end ease — the fade is gentle at first, then accelerates to full black. Feels more natural than linear or `power2.out`.

### Tuning

| Knob | Default | Notes |
|---|---|---|
| `FADE_OUT_DURATION` | 1.0s | Range: 0.6-1.5s. Longer = more dramatic; shorter = more abrupt |
| Ease | `power2.in` | Validated. `linear` feels mechanical; `power2.out` fades too fast at the start |
| Background color | `#000` (pure black) | Could be `#0F2237` (night-blue brand) for a branded fade-out — untested, may look like a glitch on dark videos |

### Why include the fade-out

Without it: the last frame holds at full brightness, then suddenly the video is over. On a phone, this feels like the video buffered or crashed.

With it: the fade signals "this is intentional" and gives the viewer 1 second to digest the URL/CTA before the closure. Validated as essential — review feedback consistently flags the un-faded version as ending too abruptly.

---

## Combining both patterns

In short-02, both patterns coexist in segment 8 (CTA segment):

```
Time    | Element                          | What's visible
--------|----------------------------------|---------------------------------
18.35s  | seg-8 (CTA background image)     | Background image fades in
18.50s  | (captions END)                   | Karaoke captions stop here
18.60s  | URL typewriter starts            | Characters appear one by one
20.10s  | URL fully visible + cursor blinks
20.50s  | fade-out starts                   | Black overlay fades in
21.50s  | (composition ends — total black)
```

The URL is visible for ~2 seconds before the fade starts, giving the viewer time to read it.

---

## Common pitfalls

| Symptom | Cause | Fix |
|---|---|---|
| URL appears all at once instead of typing | `tl.call(...)` not registering on the timeline | Make sure each `tl.call` is concatenated to the main `tl`, not a sub-timeline |
| Typewriter text gets cut off mid-character | `URL_TYPE_DURATION` too short for the URL length | Bump to 2s for longer URLs (>20 chars) |
| Fade-out happens but the screen flashes back to bright at the end | `data-duration` of `#fade-out` < `FADE_OUT_DURATION` | Make sure `data-duration` covers the full fade length |
| Fade-out covers the URL prematurely | URL ends at the same time the fade starts | Have the URL fully visible at least 0.5s before fade starts (i.e., URL ends ≤ FADE_OUT_START - 0.5s) |
| URL text overflows the canvas horizontally | Font-size too large for canvas width | Reduce font-size to 60-64px, or shorten the URL displayed (use `acme.mx/plan` instead of `acmeseguros.com`) |
| Fade-to-black has visible color tint | Browser/encoder gamma quirk | Stay with pure `#000`. If a tint persists, set `mix-blend-mode: normal` on the overlay |

---

## Why these are critical for delivery quality

A short without these patterns can have great content and still feel "off":
- No URL reveal animation: the CTA looks like a static caption
- Abrupt cut at the end: the video feels broken or unfinished

These patterns are cheap to add (zero asset generation cost; just GSAP + CSS) and noticeably improve the perceived production quality.

For the WhatsApp Stories audience, professional polish in the closing 3 seconds is what makes the difference between "this is a serious product" and "this is some random thing forwarded by a friend".
