# Watermark — Rotating 4 Corners with Compactar/Descompactar

Brand identification overlay that **rotates between the 4 corners** of the canvas during the video, with a "compactar (in) / descompactar (out)" animation between corner moves. Validated as the standard for Acme Seguros shorts (pilot v3, short-02).

---

## What it looks like

- A small rounded box in one corner showing the logo + URL text (`acmeseguros.com`).
- After a dwell time (4-4.5s), the box scales down + fades out (compactar)
- Reappears in the next corner, scales up + fades in (descompactar)
- Cycle: **TR → BR → BL → TL → TR (closes back to start)**

The rotation prevents the watermark from feeling like a static UI overlay — it acquires presence by moving.

---

## HTML element

```html
<div id="watermark" class="clip"
     data-start="0" data-duration="22.5" data-track-index="3"
     style="top: 40px; right: 40px;">
  <img src="logo.png" class="wm-logo" />
  <span class="wm-text">acmeseguros.com</span>
</div>
```

- `data-start="0"` and `data-duration="<TOTAL_DURATION>"` — the watermark exists for the entire composition
- Initial position (TR corner) set via inline `style="top: 40px; right: 40px;"`
- GSAP changes top/right/bottom/left at each transition

---

## CSS

```css
#watermark {
  position: absolute;
  display: flex;
  align-items: center;
  gap: 12px;
  z-index: 50;
  background: rgba(15, 34, 55, 0.55);  /* deep-blue at 55% opacity */
  padding: 8px 14px;
  border-radius: 8px;
  backdrop-filter: blur(4px);
  transform-origin: center center;
}

.wm-logo {
  height: 60px;
  width: auto;
  filter: brightness(0) invert(1);    /* logo is grayscale; invert to white */
}

.wm-text {
  font-family: system-ui, -apple-system, sans-serif;
  font-size: 22px;
  font-weight: 600;
  color: #f7f5f0;
  letter-spacing: 0.5px;
}
```

**Why each rule:**

- `position: absolute` — anchored to the canvas, not affected by other layout
- `z-index: 50` — sits above the background segments but below caption (track 2 captions are at higher z via `.kw` rules) and below fade-out (z-index 200)
- `background: rgba(15, 34, 55, 0.55)` — deep navy with transparency. Without a background, the white text disappears against bright video frames
- `backdrop-filter: blur(4px)` — frosted glass effect; readable on any background
- `padding 8/14` and `border-radius: 8px` — pill-like shape
- `filter: brightness(0) invert(1)` on the logo — the logo PNG is grayscale (dark gray on transparent). This filter turns the entire logo white, so it reads against the dark watermark background
- `transform-origin: center center` — when GSAP scales the watermark for compactar, it shrinks toward the center of itself, not toward a corner

---

## Animation pattern (GSAP)

The 4-corner rotation is built as a sequence of `compactar → reposition → descompactar` cycles:

```javascript
const WM_INSET = 40;             // distance from canvas edges (px)
const WM_DWELL = 4.5;            // seconds in each corner
const WM_SHRINK = 0.25;          // animation duration (compactar OR descompactar)

const WM_CORNERS = [
  { name: "tr",  top: WM_INSET, bottom: "auto",  left: "auto",     right: WM_INSET },  // 0–4.5
  { name: "br",  top: "auto",   bottom: WM_INSET, left: "auto",    right: WM_INSET },  // 4.5–9.0
  { name: "bl",  top: "auto",   bottom: WM_INSET, left: WM_INSET,  right: "auto" },    // 9.0–13.5
  { name: "tl",  top: WM_INSET, bottom: "auto",   left: WM_INSET,  right: "auto" },    // 13.5–18.0
  { name: "tr2", top: WM_INSET, bottom: "auto",   left: "auto",    right: WM_INSET },  // 18.0–end
];
```

For each transition (skip the first corner — it's set inline):

```javascript
WM_CORNERS.slice(1).forEach((corner, i) => {
  const transitionTime = (i + 1) * WM_DWELL;       // 4.5, 9.0, 13.5, 18.0
  const compactStart = transitionTime - WM_SHRINK;  // 4.25, 8.75, 13.25, 17.75

  // 1. Compactar (scale down + fade out)
  tl.to("#watermark",
        { scale: 0.3, opacity: 0, duration: WM_SHRINK, ease: "power2.in" },
        compactStart);

  // 2. Hard-kill: lock the invisible state at the moment of repositioning
  tl.set("#watermark",
         { opacity: 0, scale: 0.3 },
         transitionTime - 0.001);

  // 3. Reposition (instantaneous, while invisible)
  tl.set("#watermark",
         { top: corner.top, bottom: corner.bottom, left: corner.left, right: corner.right },
         transitionTime);

  // 4. Descompactar (scale up + fade in)
  tl.to("#watermark",
        { scale: 1, opacity: 1, duration: WM_SHRINK, ease: "power2.out" },
        transitionTime);
});
```

### The hard-kill step (why it matters)

Without the explicit `tl.set(..., { opacity: 0, scale: 0.3 }, transitionTime - 0.001)`, the HyperFrames lint flags `gsap_exit_missing_hard_kill`:

> *"GSAP exit on '#watermark' ends at the X.Xs clip start boundary without a matching tl.set hard kill. Non-linear seeking can land after the fade and leave stale visibility state."*

The hard-kill ensures that during random-access seeks (which the renderer performs frame-by-frame), the state is unambiguously "invisible" at the transition point — no half-faded ghosts.

---

## Helper for CSS values via GSAP

GSAP needs CSS values as strings — particularly `top: "auto"` vs `top: "40px"`. A small formatter:

```javascript
const formatPos = (v) => typeof v === "number" ? `"${v}px"` : `"${v}"`;
```

Then in the generated string:

```javascript
`tl.set("#watermark", {
  top: ${formatPos(corner.top)},
  bottom: ${formatPos(corner.bottom)},
  left: ${formatPos(corner.left)},
  right: ${formatPos(corner.right)}
}, ${transitionTime});`
```

This produces valid JS like `top: "40px", bottom: "auto", ...` instead of the broken `top: 40px, bottom: "auto"`.

---

## Tuning

| Knob | Default | What it changes |
|---|---|---|
| `WM_INSET` | 40 | Distance from canvas edges. Larger = more breathing room (looks calmer); smaller = closer to edge (more presence) |
| `WM_DWELL` | 4.5s | Time in each corner. Shorter = more visual movement (busy); longer = calmer, may feel static |
| `WM_SHRINK` | 0.25s | Animation duration. 0.15-0.30 is the comfortable range |
| `font-size` (`.wm-text`) | 22px | Smaller (18) for less prominent watermark; larger (26) when it's a key brand moment |
| `background` rgba alpha | 0.55 | Lower (0.4) = more transparent, less visible against dim backgrounds; higher (0.7) = more solid |

---

## Validated values (production)

These came from pilot iteration:

- **Pilot v1**: watermark fixed top-right (no rotation) → user feedback: *"se ve muy estática"*
- **Pilot v3**: 4-corner rotation with compactar/descompactar 250ms, dwell 4s — approved, used in published version
- **Short-02**: same pattern, dwell tweaked to 4.5s for the 22s composition (so corners cycle naturally over the duration). 4 corners × 4.5s = 18s, plus a 3.5s tail in the starting corner

For longer videos (>30s), increase dwell to 5-6s rather than adding more corners — the 4-corner cycle is iconic and adding more breaks the visual rhythm.

---

## Common pitfalls

| Symptom | Cause | Fix |
|---|---|---|
| Watermark visible in two corners simultaneously briefly | Reposition `set` happening while watermark is still partially visible | Tighten the hard-kill: `tl.set("#watermark", { opacity: 0, scale: 0.3 }, transitionTime - 0.001)` before the reposition `set` |
| Watermark "jumps" instead of compactar smoothly | GSAP didn't animate scale; check `transform-origin` is `center center` not `top left` | Ensure CSS has `transform-origin: center center;` |
| Logo invisible against dark video backgrounds | Logo's grayscale didn't get inverted | `filter: brightness(0) invert(1);` on `.wm-logo` |
| Watermark too small to read on phone | Font-size too low or canvas too large | Bump font-size; ensure canvas is 1080×1920 not lower |
| Lint error "gsap_exit_missing_hard_kill" | Missing `tl.set` after the exit `.to(...)` | Add the hard-kill (see Animation Pattern section) |
