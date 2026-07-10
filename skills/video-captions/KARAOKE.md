# Karaoke Style — Reference

Word-by-word karaoke captions: full phrase visible, current word lights up cyan with a glow and scale-up. Distinct from TikTok-pop (one word at a time). When the user says "karaoke", "karaoke captions", "highlight the word being spoken", or "lyrics-video style", use this configuration.

This document captures the final tuning after iterating with a real user (5 rounds of feedback). Don't redesign — apply this verbatim and only deviate if the user explicitly requests something different.

---

## Visual at a glance

```
              palabra1   palabra2   PALABRA3   palabra4
              ────────   ────────   ════════   ────────
              gris       blanca      CYAN       gris
                                    + glow
                                    + scale
              future     past       active     future
```

Three states per word:
- **future** (not yet spoken): light grey `#c8c8c8`
- **active** (currently spoken): **cyan `#00ffff`**, glow, scale 1.15, thin dark-grey outline
- **past** (already spoken): white `#ffffff`

The line as a whole is centered, anchored bottom (or top during lower-third overlay window — same dynamic positioning as TikTok-pop).

---

## Final config — apply these values verbatim

These are the values that survived 5 rounds of iteration. Each one solves a real problem that surfaced during review.

### Line grouping

```javascript
const MAX_SOFT_WORDS = 3;   // split here when you hit a comma
const MAX_HARD_WORDS = 4;   // never exceed; force-split at longest pause
```

**Why 3-4 and not more**: longer lines run off the canvas (1024×576 with 42px font and active-word at scale 1.15 quickly exceeds the visible width). The user's first complaint was lines of 9-21 words spilling off-screen. 3-4 is the sweet spot — readable but always fits.

Splitting algorithm:
1. Sentence-ending punctuation (`.`, `!`, `?`) → always split
2. Comma after `MAX_SOFT_WORDS` → split at the comma
3. Hit `MAX_HARD_WORDS` → split at the **longest inter-word pause** within the current group (most natural breath)
4. No internal pause found → split at the hard limit anyway (rare)

### Line timing pad

```javascript
const LINE_LEAD_IN  = 0.2;   // line fades in 200ms before first word
const LINE_LEAD_OUT = 0.3;   // line stays 300ms after last word
```

Lead-in lets the viewer preview the upcoming phrase. Lead-out prevents the line from disappearing the instant the last word ends (visually jarring).

### Active word state — full GSAP block

```javascript
// In the build script. OUTLINE_FULL applies to past + future words; OUTLINE_THIN
// applies to the active word (less competition with the cyan glow).
const OUTLINE_FULL =
  '-2px -2px 0 #000, 2px -2px 0 #000, -2px 2px 0 #000, 2px 2px 0 #000, 0 0 8px rgba(0,0,0,0.7)';
const OUTLINE_THIN =
  '-1px -1px 0 #2a2a2a, 1px -1px 0 #2a2a2a, -1px 1px 0 #2a2a2a, 1px 1px 0 #2a2a2a';

// Per word, three timeline events:
tl.set("#wX", {
    color: "#c8c8c8",                              // future: grey
    scale: 1,
    filter: "drop-shadow(0 0 0 transparent)",
    textShadow: OUTLINE_FULL,                      // full black outline
    zIndex: 1
  }, 0)
  .to("#wX", {
    color: "#00ffff",                              // active: cyan
    scale: 1.15,                                   // visual grow (no reflow)
    filter: "drop-shadow(0 0 14px #00ffff) drop-shadow(0 0 6px #00ffff)",
    textShadow: OUTLINE_THIN,                      // thin dark-grey, NOT black
    zIndex: 100,                                   // paint above neighbors
    duration: 0.06,
    ease: "power2.out"
  }, w.start)
  .to("#wX", {
    color: "#ffffff",                              // past: white
    scale: 1.0,
    filter: "drop-shadow(0 0 0 transparent)",
    textShadow: OUTLINE_FULL,                      // outline back
    zIndex: 1,
    duration: 0.1,
    ease: "power2.in"
  }, w.end);
```

**Why each setting matters:**

- **`color: "#00ffff"`** — cyan reads as "lit up" against most video backgrounds. Tested vs gold (`#ffd60a`); user explicitly preferred cyan + glow over gold.
- **`scale: 1.15`** — `transform: scale()` does NOT reflow flex neighbors (the bounding box stays at scale 1.0). The word grows visually but the line stays anchored. `font-size` would reflow and make the whole line jitter — don't use it.
- **`filter: drop-shadow(...) drop-shadow(...)`** — stacked drop-shadows produce a halo glow around the cyan letterforms. Two shadows give depth: 14px wide, 6px tighter. Single shadow looks flat.
- **`textShadow: OUTLINE_THIN`** — the active word doesn't wear the full black outline that past/future words wear. The black outline competes with the cyan glow visually (looks muddy). Thin 1px dark-grey (`#2a2a2a`) gives just enough contrast without fighting the glow.
- **`zIndex: 100`** + `position: relative` on `.kw` (CSS) — the scaled-up active word **paints above its neighbors**. Without this, the next word in DOM order paints over the active word's overflow region, making the active word look like it's hiding behind the next one.

### Line container CSS

```css
.caption-line {
  position: absolute;
  left: 50%;
  transform: translateX(-50%);
  display: flex;
  flex-wrap: nowrap;
  /* 0.7em (not 0.35em) so active word at scale 1.15 — extending ~7.5% per side —
     doesn't touch its neighbors. The first version used 0.35em and the active
     word visibly clipped into the next word. */
  gap: 0.7em;
  font-family: system-ui, -apple-system, "Segoe UI", sans-serif;
  font-weight: 800;
  font-size: 42px;
  white-space: nowrap;
  letter-spacing: 0.3px;
  pointer-events: none;
  max-width: ${W - 80}px;     /* 80px breathing room */
}
.caption-line.bottom { bottom: 70px; }
.caption-line.top    { top: 30px; }
```

### Per-word CSS

```css
.kw {
  position: relative;          /* enables z-index */
  z-index: 1;                  /* baseline; active word jumps to 100 via GSAP */
  display: inline-block;
  color: #c8c8c8;              /* matches GSAP "future" state */
  transform-origin: center bottom;   /* word grows up, not toward baseline */
  text-shadow:
    -2px -2px 0 #000, 2px -2px 0 #000,
    -2px  2px 0 #000, 2px  2px 0 #000,
    0 0 8px rgba(0,0,0,0.7);
  will-change: transform, color, filter, text-shadow, z-index;
}
```

`transform-origin: center bottom` keeps the baseline anchored — the active word grows upward and outward, not into the line's vertical space.

---

## Complete build script

Save as `build-captions-karaoke.mjs` in the variant directory. Reads `scribe-response.json` (Scribe is the recommended provider — see PROVIDERS.md). For Whisper, change the input filename and add the `SHIFT = -0.15` (see PIPELINE.md for the Whisper adapter shape).

```javascript
// build-captions-karaoke.mjs
// Generates index.html for karaoke-style captions from scribe-response.json.
// Run: node build-captions-karaoke.mjs

import fs from "node:fs";

// ──────── CONFIG ────────
const W = 1024, H = 576;                            // match source video
const VIDEO = "source.mp4";
const TRANSCRIPT = "scribe-response.json";
const OUT = "index.html";

const SHIFT = 0;                                     // 0 for Scribe; -0.15 for Whisper-1
const VOCAB_FIXES = { "aguja": "Ahuja" };           // post-process proper nouns
const TOP_WINDOW = { start: 0.7, end: 5.5 };        // anchor to lower-third overlay window

const MAX_SOFT_WORDS = 3;
const MAX_HARD_WORDS = 4;
const LINE_LEAD_IN  = 0.2;
const LINE_LEAD_OUT = 0.3;

// ──────── LOAD + NORMALIZE ────────
const data = JSON.parse(fs.readFileSync(TRANSCRIPT, "utf8"));
const isScribe = "audio_duration_secs" in data;
const duration = Number((isScribe ? data.audio_duration_secs : data.duration).toFixed(2));

const fixWord = (text) => {
  const lower = text.toLowerCase().replace(/[.,!?;:¿¡]/g, "");
  return VOCAB_FIXES[lower] ?? text;
};

const rawWords = (isScribe
  ? data.words.filter(w => w.type === "word").map(w => ({ word: fixWord(w.text), start: w.start, end: w.end }))
  : data.words.map(w => ({ word: fixWord(w.word), start: w.start, end: w.end }))
).map(w => ({
  word: w.word,
  start: Math.max(0, w.start + SHIFT),
  end:   Math.max(0.05, w.end + SHIFT),
}));

// ──────── GROUP WORDS INTO LINES ────────
function groupIntoLines(words, softMax, hardMax) {
  const lines = [];
  let current = [];
  const flush = (g) => { if (g.length > 0) lines.push(g); return []; };
  const findBestSplit = (group) => {
    if (group.length < 2) return -1;
    let bestIdx = -1, bestGap = 0;
    for (let i = 0; i < group.length - 1; i++) {
      const gap = group[i + 1].start - group[i].end;
      if (gap > bestGap) { bestGap = gap; bestIdx = i; }
    }
    return bestIdx;
  };
  for (const w of words) {
    current.push(w);
    const trailing = w.word.slice(-1);
    if (/[.!?]/.test(trailing)) {
      current = flush(current);
    } else if (current.length >= softMax && /,/.test(trailing)) {
      current = flush(current);
    } else if (current.length >= hardMax) {
      const splitAt = findBestSplit(current);
      if (splitAt >= 0) {
        flush(current.slice(0, splitAt + 1));
        current = current.slice(splitAt + 1);
      } else {
        current = flush(current);
      }
    }
  }
  flush(current);
  return lines;
}

// ──────── ASSEMBLE LINES WITH IDS + POSITIONING ────────
let wordIdx = 0;
const lines = groupIntoLines(rawWords, MAX_SOFT_WORDS, MAX_HARD_WORDS).map((group, i) => {
  const firstStart = group[0].start;
  const lastEnd    = group[group.length - 1].end;
  const lineStart  = Math.max(0, Number((firstStart - LINE_LEAD_IN).toFixed(3)));
  const lineEnd    = Number((lastEnd + LINE_LEAD_OUT).toFixed(3));
  const isTop      = firstStart >= TOP_WINDOW.start && firstStart <= TOP_WINDOW.end;
  const words      = group.map(w => ({
    id: `w${wordIdx++}`,
    text: w.word,
    start: Number(w.start.toFixed(3)),
    end:   Number(w.end.toFixed(3)),
  }));
  return {
    id: `line${i}`,
    start: lineStart,
    duration: Number((lineEnd - lineStart).toFixed(3)),
    pos: isTop ? "top" : "bottom",
    words,
  };
});

// ──────── ENFORCE NON-OVERLAP BETWEEN LINES (10ms gap) ────────
const GAP = 0.01;
for (let i = 0; i < lines.length - 1; i++) {
  const cur = lines[i], nxt = lines[i + 1];
  const maxAllowedEnd = nxt.start - GAP;
  if (cur.start + cur.duration > maxAllowedEnd) {
    cur.duration = Math.max(0.001, Number((maxAllowedEnd - cur.start).toFixed(3)));
  }
}

// ──────── BUILD HTML ────────
const escape = (s) => s.replace(/[<>&"]/g, c => ({
  "<": "&lt;", ">": "&gt;", "&": "&amp;", '"': "&quot;",
}[c]));

const lineDivs = lines.map(line => {
  const wordSpans = line.words
    .map(w => `        <span id="${w.id}" class="kw">${escape(w.text)}</span>`)
    .join("\n");
  return `      <div id="${line.id}" class="caption-line ${line.pos} clip"
           data-start="${line.start}" data-duration="${line.duration}" data-track-index="1">
${wordSpans}
      </div>`;
}).join("\n");

const OUTLINE_FULL = '-2px -2px 0 #000, 2px -2px 0 #000, -2px 2px 0 #000, 2px 2px 0 #000, 0 0 8px rgba(0,0,0,0.7)';
const OUTLINE_THIN = '-1px -1px 0 #2a2a2a, 1px -1px 0 #2a2a2a, -1px 1px 0 #2a2a2a, 1px 1px 0 #2a2a2a';

const gsapBlocks = lines.flatMap(line =>
  line.words.map(w =>
`      tl.set("#${w.id}", { color: "#c8c8c8", scale: 1, filter: "drop-shadow(0 0 0 transparent)", textShadow: "${OUTLINE_FULL}", zIndex: 1 }, 0)
        .to("#${w.id}", { color: "#00ffff", scale: 1.15, filter: "drop-shadow(0 0 14px #00ffff) drop-shadow(0 0 6px #00ffff)", textShadow: "${OUTLINE_THIN}", zIndex: 100, duration: 0.06, ease: "power2.out" }, ${w.start})
        .to("#${w.id}", { color: "#ffffff", scale: 1.0, filter: "drop-shadow(0 0 0 transparent)", textShadow: "${OUTLINE_FULL}", zIndex: 1, duration: 0.1, ease: "power2.in" }, ${w.end});`)
).join("\n");

const html = `<!doctype html>
<html lang="es">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=${W}, height=${H}" />
    <script src="https://cdn.jsdelivr.net/npm/gsap@3.14.2/dist/gsap.min.js"></script>
    <style>
      * { margin: 0; padding: 0; box-sizing: border-box; }
      html, body { margin: 0; width: ${W}px; height: ${H}px; overflow: hidden; background: #000; }
      .caption-line {
        position: absolute;
        left: 50%;
        transform: translateX(-50%);
        display: flex;
        flex-wrap: nowrap;
        gap: 0.7em;
        font-family: system-ui, -apple-system, "Segoe UI", sans-serif;
        font-weight: 800;
        font-size: 42px;
        white-space: nowrap;
        letter-spacing: 0.3px;
        pointer-events: none;
        max-width: ${W - 80}px;
      }
      .caption-line.bottom { bottom: 70px; }
      .caption-line.top    { top: 30px; }
      .kw {
        position: relative;
        z-index: 1;
        display: inline-block;
        color: #c8c8c8;
        transform-origin: center bottom;
        text-shadow:
          -2px -2px 0 #000, 2px -2px 0 #000,
          -2px  2px 0 #000, 2px  2px 0 #000,
          0 0 8px rgba(0,0,0,0.7);
        will-change: transform, color, filter, text-shadow, z-index;
      }
    </style>
  </head>
  <body>
    <div id="root" data-composition-id="main" data-start="0" data-duration="${duration}" data-width="${W}" data-height="${H}">
      <video id="a-roll" class="clip" src="${VIDEO}" muted playsinline data-start="0" data-duration="${duration}" data-track-index="0" style="position: absolute; inset: 0; width: 100%; height: 100%; object-fit: cover"></video>
      <audio id="a-roll-audio" src="${VIDEO}" data-start="0" data-duration="${duration}" data-track-index="2" data-volume="1"></audio>

${lineDivs}
    </div>

    <script>
      window.__timelines = window.__timelines || {};
      const tl = gsap.timeline({ paused: true });
${gsapBlocks}
      window.__timelines["main"] = tl;
    </script>
  </body>
</html>
`;

fs.writeFileSync(OUT, html);
const totalWords = lines.reduce((acc, l) => acc + l.words.length, 0);
console.log(`Wrote ${OUT}: ${lines.length} lines, ${totalWords} words, duration=${duration}s`);
```

---

## Common karaoke-specific pitfalls

| Symptom | Cause | Fix |
|---|---|---|
| Active word visually hidden behind the next word | No z-index on active state | Add `position: relative` to `.kw` CSS, GSAP sets `zIndex: 100` on active |
| Lines spill off-canvas right side | Too many words per line (>4) | `MAX_SOFT_WORDS = 3`, `MAX_HARD_WORDS = 4` |
| Active word touches/clips into neighbors | Flex `gap` too small for scale 1.15 | `gap: 0.7em` (was 0.35em — too tight) |
| Cyan word looks muddy / black outline competes with glow | Active word inherits `OUTLINE_FULL` | Active state uses `OUTLINE_THIN` (1px `#2a2a2a`), not the full black outline |
| Whole line jitters when active word changes | Used `font-size` change instead of `transform: scale` | Always use `scale` — `transform` doesn't reflow flex neighbors |
| Line appears exactly as first word starts (no preview) | `LINE_LEAD_IN = 0` | Use `LINE_LEAD_IN = 0.2` (~200ms) for preview feel |
| Line vanishes the instant last word ends | `LINE_LEAD_OUT = 0` | Use `LINE_LEAD_OUT = 0.3` (~300ms) for breathing room |
| 21-word lines appearing despite split logic | Sentence has no `.!?` and no commas | Hard cap at `MAX_HARD_WORDS` + split at longest internal pause |

---

## Tunables — when to deviate from the defaults

These values worked for one specific video (51s, single Spanish-speaking presenter, 1024×576). For different content, adjust:

- **Faster speech** (rap, energetic narration): drop `MAX_HARD_WORDS` to 3 — even 4 may feel cramped
- **Slower speech** (calm narration, audiobook): can go up to `MAX_HARD_WORDS = 5` — long phrases fit and read naturally
- **Vertical 9:16 (TikTok format)**: drop font-size to 36px; reduce `MAX_HARD_WORDS` to 3 (less horizontal space)
- **Larger canvas (1920×1080)**: bump font-size to 60-72px; `MAX_HARD_WORDS` can stay 4
- **Different brand color**: replace `#00ffff` (cyan) and the matching glow color in `drop-shadow(... #00ffff)` with the brand's primary. Keep `scale 1.15` and `OUTLINE_THIN` regardless.
- **Higher contrast preference**: bump `OUTLINE_THIN` from 1px `#2a2a2a` to 1.5px `#1a1a1a` — still subtle but reads on bright backgrounds.

---

## Related references

- [SKILL.md](SKILL.md) — overview, when to use captions at all, provider choice
- [PIPELINE.md](PIPELINE.md) — the TikTok-pop variant (one word at a time, scale-pop animation)
- [PROVIDERS.md](PROVIDERS.md) — Whisper vs Scribe (Scribe is the default for karaoke; its accurate timestamps matter more here than for pop)
- [TROUBLESHOOTING.md](TROUBLESHOOTING.md) — generic caption errors not specific to karaoke
