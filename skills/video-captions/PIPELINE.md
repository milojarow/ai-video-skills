# Captions Pipeline — TikTok-pop Style

Complete copy-paste implementation for the **TikTok-pop variant** (one word visible at a time, scale-pop animation per word). For karaoke style (full phrase visible, current word lights up cyan), see [KARAOKE.md](KARAOKE.md) instead — different line grouping, different GSAP transitions, different CSS.

Read this when you need the actual code for TikTok-pop captions, not just the concept.

---

## 1. STT call

### Whisper

```bash
curl https://api.openai.com/v1/audio/transcriptions \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -F "file=@source.mp4" \
  -F "model=whisper-1" \
  -F "language=es" \
  -F "response_format=verbose_json" \
  -F "timestamp_granularities[]=word" \
  -F "prompt=Contexto del audio + nombres propios separados por espacios" \
  -o whisper-verbose.json
```

`whisper-1` is the only OpenAI model that returns word-level timestamps via this endpoint. Other models (`gpt-4o-transcribe`) don't expose word granularity at the time of this writing.

### Scribe

```bash
curl https://api.elevenlabs.io/v1/speech-to-text \
  -H "xi-api-key: $ELEVENLABS_API_KEY" \
  -F "file=@source.mp4" \
  -F "model_id=scribe_v1" \
  -F "language_code=spa" \
  -F "tag_audio_events=false" \
  -o scribe-response.json
```

`tag_audio_events=false` strips noise like `[laughter]`, `[music]` from the output — wanted for clean captions, unwanted for accessibility-grade subtitles.

---

## 2. Build script (Node)

Save as `build-captions.mjs` in the variant directory. Generates `index.html` from the transcript JSON.

```javascript
// build-captions.mjs
// Usage: node build-captions.mjs
// Generates index.html for TikTok-style word-by-word captions.

import fs from "node:fs";

// ──────── CONFIG ────────
const W = 1024, H = 576;                            // match source video
const VIDEO = "source.mp4";                          // file in same dir
const TRANSCRIPT = "transcript.json";                // STT output
const OUT = "index.html";

// Whisper-1 reports timestamps ~150ms after the actual onset.
// Scribe is accurate, so SHIFT = 0 for it. Tune per-source.
const SHIFT = 0; // for Scribe; use -0.15 for Whisper-1

// Vocabulary fixes (Whisper/Scribe both miss proper nouns).
const VOCAB_FIXES = {
  "aguja": "Ahuja",
  "abuja": "Ahuja",
};

// Caption position window: while these timestamps are active,
// captions render at top (away from a lower-third overlay in source).
// Anchor to transcript word timestamps, not magic numbers.
const TOP_WINDOW = { start: 0.7, end: 5.5 };

// ──────── LOAD ────────
const data = JSON.parse(fs.readFileSync(TRANSCRIPT, "utf8"));

// Detect provider by shape: Whisper has `duration`, Scribe has `audio_duration_secs`
const isScribe = "audio_duration_secs" in data;
const duration = Number(
  (isScribe ? data.audio_duration_secs : data.duration).toFixed(2)
);

// ──────── NORMALIZE TO {word, start, end} ────────
const fixWord = (text) => {
  const lower = text.toLowerCase().replace(/[.,!?;:¿¡]/g, "");
  return VOCAB_FIXES[lower] ?? text;
};

let rawWords;
if (isScribe) {
  // Scribe: filter type==="word" (skip "spacing"), rename text→word
  rawWords = data.words
    .filter(w => w.type === "word")
    .map(w => ({ word: fixWord(w.text), start: w.start, end: w.end }));
} else {
  // Whisper: word field already named "word"
  rawWords = data.words.map(w => ({
    word: fixWord(w.word), start: w.start, end: w.end,
  }));
}

// ──────── RESOLVE PHANTOM WORDS (start == end) ────────
// Strategy: split-forward — phantom takes first half of next's duration.
// If next is too short to split, merge text.
const work = rawWords.map(w => ({ ...w }));
const resolved = [];
for (let i = 0; i < work.length; i++) {
  const w = work[i];
  const isPhantom = (w.end - w.start) < 0.001;
  if (isPhantom && i + 1 < work.length) {
    const next = work[i + 1];
    const nextDur = next.end - next.start;
    if (nextDur >= 0.1) {
      const mid = next.start + nextDur / 2;
      resolved.push({ word: w.word, start: w.start, end: mid });
      work[i + 1] = { ...next, start: mid };
    } else {
      resolved.push({ word: `${w.word} ${next.word}`, start: w.start, end: next.end });
      i++; // skip merged-in word
    }
  } else {
    resolved.push(w);
  }
}

// ──────── APPLY SHIFT ────────
const words = resolved.map(w => ({
  word: w.word,
  start: Math.max(0, Number((w.start + SHIFT).toFixed(3))),
  end: Math.max(0.05, Number((w.end + SHIFT).toFixed(3))),
}));

// ──────── ENFORCE NON-OVERLAP (10ms gap) ────────
const GAP = 0.01;
const safeWords = words.map((w, i) => {
  const next = words[i + 1];
  const start = Number(w.start.toFixed(3));
  let end = Number(w.end.toFixed(3));
  if (next) end = Math.min(end, Number(next.start.toFixed(3)) - GAP);
  const dur = Math.max(0.001, Number((end - start).toFixed(3)));
  return { word: w.word, start, dur, id: `w${i}` };
});

// ──────── BUILD HTML ────────
const escape = (s) => s.replace(/[<>&"]/g, c => ({
  "<": "&lt;", ">": "&gt;", "&": "&amp;", '"': "&quot;",
}[c]));

const isTopWord = (w) => w.start >= TOP_WINDOW.start && w.start <= TOP_WINDOW.end;

const spans = safeWords.map(w => {
  const pos = isTopWord(w) ? "top" : "bottom";
  return `      <span id="${w.id}" class="word ${pos} clip"
            data-start="${w.start}" data-duration="${w.dur}" data-track-index="1">${escape(w.word)}</span>`;
}).join("\n");

const gsapLines = safeWords.map(w => {
  const pop = Math.min(0.08, w.dur);
  return `      tl.fromTo("#${w.id}", { scale: 0.7, opacity: 0 }, { scale: 1.05, opacity: 1, duration: ${pop / 2}, ease: "back.out(2)" }, ${w.start})
        .to("#${w.id}", { scale: 1.0, duration: ${pop / 2}, ease: "power2.out" }, ${w.start + pop / 2});`;
}).join("\n");

const html = `<!doctype html>
<html lang="es">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=${W}, height=${H}" />
    <script src="https://cdn.jsdelivr.net/npm/gsap@3.14.2/dist/gsap.min.js"></script>
    <style>
      * { margin: 0; padding: 0; box-sizing: border-box; }
      html, body {
        margin: 0; width: ${W}px; height: ${H}px;
        overflow: hidden; background: #000;
      }
      .word {
        position: absolute;
        left: 50%;
        transform: translateX(-50%);
        font-family: system-ui, -apple-system, "Segoe UI", sans-serif;
        font-weight: 900;
        font-size: 56px;
        color: #fff;
        text-shadow:
          -3px -3px 0 #000, 3px -3px 0 #000,
          -3px  3px 0 #000, 3px  3px 0 #000,
          0 0 10px rgba(0,0,0,0.6);
        white-space: nowrap;
        letter-spacing: 0.5px;
        pointer-events: none;
      }
      .word.bottom { bottom: 80px; }
      .word.top    { top: 30px; }
    </style>
  </head>
  <body>
    <div
      id="root"
      data-composition-id="main"
      data-start="0"
      data-duration="${duration}"
      data-width="${W}"
      data-height="${H}"
    >
      <video
        id="a-roll"
        class="clip"
        src="${VIDEO}"
        muted
        playsinline
        data-start="0"
        data-duration="${duration}"
        data-track-index="0"
        style="position: absolute; inset: 0; width: 100%; height: 100%; object-fit: cover"
      ></video>
      <audio
        id="a-roll-audio"
        src="${VIDEO}"
        data-start="0"
        data-duration="${duration}"
        data-track-index="2"
        data-volume="1"
      ></audio>

${spans}
    </div>

    <script>
      window.__timelines = window.__timelines || {};
      const tl = gsap.timeline({ paused: true });
${gsapLines}
      window.__timelines["main"] = tl;
    </script>
  </body>
</html>
`;

fs.writeFileSync(OUT, html);
console.log(`Wrote ${OUT}: ${safeWords.length} words, duration=${duration}s, dims=${W}x${H}`);
```

The script auto-detects Whisper vs Scribe from the JSON shape. Adjust `SHIFT` for Whisper-1 (`-0.15`) or leave at 0 for Scribe.

---

## 3. Lint

```bash
./node_modules/.bin/hyperframes lint
```

Expected outcome: **0 errors**, 2 warnings about file size and track density. The warnings recommend splitting into sub-compositions; for a single-pass captions experiment they're tolerable.

If you see errors, see [TROUBLESHOOTING.md](TROUBLESHOOTING.md).

---

## 4. Render

### Iteration (fast)

```bash
./node_modules/.bin/hyperframes render --output draft.mp4 --quality draft
```

~2 min for 50s of video at 1024×576 with 3 workers (default). Use this loop for tweaking caption style, timing, vocab fixes.

### Final delivery (slow + good)

```bash
prime-run ./node_modules/.bin/hyperframes render --output final.mp4 --gpu --quality high
```

`prime-run` activates discrete NVIDIA on dual-GPU systems. `--gpu` engages NVENC for H.264 hardware encoding. `--quality high` is x264 slow preset (CRF 15) for near-lossless quality.

---

## 5. Inspect (visual sanity)

```bash
./node_modules/.bin/hyperframes inspect
```

Headless Chrome seeks through the timeline at multiple sample points and reports text overflowing containers, words spilling off-canvas, etc. Useful especially for TikTok-style — large fonts can clip if a word is too long.

---

## File layout reminder

Inside the variant directory (e.g., `~/video-lab/captioning-experiments/foo-video/A-whisper/`), the working layout is:

```
A-whisper/
├── source.mp4                    ← copied here by `hyperframes init --video`
├── transcript.json               ← from STT call (whisper-verbose.json or scribe-response.json)
├── build-captions.mjs            ← the script above
├── index.html                    ← generated by build-captions.mjs
├── hyperframes.json              ← created by `hyperframes init`
├── meta.json                     ← created by `hyperframes init`
├── package.json                  ← if `node_modules` is here, otherwise above
└── final.mp4                     ← render output
```

`node_modules/` lives one level up (shared between variants — saves disk).
