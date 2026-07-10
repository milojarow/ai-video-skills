# Captions Troubleshooting

Lookup index for every error encountered in development. If something breaks during the captions pipeline, search here first.

---

## Lint errors

### `overlapping_clips_same_track`

**Symptom**: hyperframes lint reports clips on track-index N overlap.

**Causes**:
1. Real overlap — two captions whose `start + duration > next.start`.
2. Floating-point precision — `30.0009999...s` reads as overlap with `30.001s`.
3. Phantom words from STT (`start == end`) — see below.

**Fixes**:
- Round timestamps to 3 decimals: `Number(x.toFixed(3))` everywhere.
- Enforce a 10ms gap: `end = Math.min(end, nextStart - 0.01)`.
- Accept short durations after clamping: `Math.max(0.001, end - start)`. Non-overlap takes priority over minimum duration.
- For phantom words specifically, pre-process via split-forward (see PIPELINE.md).

---

### `media_missing_data_start`

**Symptom**: `<video id="x"> has src but no data-start. HyperFrames cannot own playback for untimed media...`

**Cause**: A `<video>` or `<audio>` element is missing `data-start="0"`.

**Fix**: Add `data-start="0"` to every media element in the composition root, even if it's "obvious" the clip starts at zero. The default scaffolding from `hyperframes init` sometimes omits this on `<video>` — patch it in your build script.

---

### Composition warnings (file too large, track too dense)

**Symptom**: Warnings like "This HTML composition file has 600 lines" or "Track 1 has 148 timed elements".

**Cause**: Caption-style word-by-word naturally produces many spans (one per word) on one track. The lint warning is a recommendation, not an error.

**Fixes** (optional):
- Split scenes into sub-compositions via `data-composition-src`. Useful for multi-scene videos but overkill for a single 1-minute caption track.
- Just ignore the warning. Use `hyperframes lint` (without `--strict`) so warnings don't fail the run.

---

## Render errors

### `Video has sparse keyframes (max interval: N seconds)`

**Symptom**: Warning during render. Captions might appear correct in the output but the underlying `<video>` clip freezes intermittently.

**Cause**: Source video has keyframes too far apart for the seek-driven render. HyperFrames captures each output frame by seeking back to the source — without dense keyframes, the seek lands on nearest keyframe and replays forward, slowly.

**Fix**: Re-encode source before scaffolding the project:
```bash
ffmpeg -i source.mp4 \
  -c:v libx264 -r 30 -g 30 -keyint_min 30 \
  -movflags +faststart -c:a copy \
  source.fixed.mp4
```

Run this preprocessor on every source video as a habit. The warning is informational but the freezing is real.

---

### `BeginFrame auto-worker calibration timed out`

**Symptom**: WARN during render: `[Render] BeginFrame auto-worker calibration timed out; retrying calibration in screenshot capture mode.`

**Cause**: Chrome's BeginFrame protocol stalled during initial timing calibration. Render falls back to slower "screenshot capture" mode.

**Effect**: The render still succeeds, just slower (one frame at a time instead of pipelined). Worse on machines with slow disk or under heavy load.

**Fixes**:
- Reduce concurrent renders if running multiple at once: `--max-concurrent-renders 1`.
- Reduce workers: `--workers 1` or `--workers 2`.
- Free up RAM (close Chrome, etc.).
- Live with it — output quality is identical, only speed suffers.

---

### `[Browser:WARN] AudioContext was not allowed to start`

**Symptom**: Repeated browser warnings during render.

**Cause**: Chrome's autoplay policy. Cosmetic only — HyperFrames doesn't actually need WebAudio for the render (audio is mixed by FFmpeg later).

**Fix**: None needed. Ignore the warnings.

---

## Pipeline / runtime issues

### Captions appear consistently late

**Symptom**: A word is spoken in the audio at, say, 5.0s, but the on-screen caption pops at 5.15-5.2s. Affects all words, not just some.

**Cause**: Whisper-1 reports word timestamps ~150ms after the actual audio onset. Known model behavior.

**Fix**: Apply a global shift in the build script:
```javascript
const SHIFT = -0.15;  // shift all timestamps 150ms earlier
```

Scribe doesn't have this issue — leave SHIFT at 0 for Scribe.

If timestamps look off in a specific other way (advance, not lag), the heuristic doesn't apply — investigate per-word.

---

### Two words appear visually overlapping ("atropello")

**Symptom**: Word A is still visible/animating when word B pops in. Looks like they're crashing into each other.

**Causes**:
1. Pop animation duration (default ~150ms) is longer than the word's actual data-duration. Adjacent words start their pop before the previous one finishes.
2. Two words share the same `start` time after phantom-word merge.

**Fixes**:
- Cap pop duration at 80ms: `const pop = Math.min(0.08, w.dur);`. This trades some visual punch for never overlapping.
- Use split-forward on phantom words instead of merging text (see PIPELINE.md). Each word stays visually distinct.

---

### "Aguja" / "Abuja" instead of "Ahuja" (or any other proper noun)

**Symptom**: STT transcribes a proper noun as the closest phonetic match it knows.

**Causes**:
1. The STT model has never seen this word.
2. For Spanish, common Mexican surnames (Indian-origin like "Ahuja") are particularly affected — Whisper biases to capitals/cities.

**Fixes** (use both layers):

1. **At STT call time** (Whisper only — works well):
   ```bash
   curl ... -F "prompt=El locutor es <nombre completo del locutor>, agente de seguros..."
   ```
   The prompt biases the model toward this vocabulary.

2. **In post-processing** (always):
   ```javascript
   const VOCAB_FIXES = {
     "aguja": "Ahuja",
     "abuja": "Ahuja",
   };
   const fixWord = (text) => {
     const lower = text.toLowerCase().replace(/[.,!?;:¿¡]/g, "");
     return VOCAB_FIXES[lower] ?? text;
   };
   ```

   For Scribe specifically, `biased_keywords=Ahuja:5` did **not** work in our test. Don't rely on that parameter.

For each new project, build up `VOCAB_FIXES` for known names (people, places, brands) before the first STT call. It's a 30-second investment that saves a re-transcribe.

---

### A whole word is missing from captions

**Symptom**: The locutor clearly says a word, but no caption appears for it.

**Cause**: STT silently dropped the word. Both Whisper and Scribe do this at low but non-zero rates. Especially common with conjunctions ("y", "o", "de") in fast speech.

**Fix**: There's no automated fix. Two options:

1. **Post-process review**: have a human read the transcript JSON before render and add missing words manually. Worth it for client deliverables.

2. **Try the other provider**. Whisper's omissions and Scribe's omissions don't usually overlap. If a word is missing in one, it's often present in the other.

For automated production at scale, run **both** providers, diff, and flag mismatches for human review.

---

### Captions cover the source video's lower-third overlay

**Symptom**: The video has a graphic at the bottom (presenter name, channel logo, subscribe button) and the captions overlap it.

**Cause**: Caption position is fixed at `bottom: 80px`.

**Fix**: Use a TOP_WINDOW. While the overlay is visible, render captions at the top instead. CSS:
```css
.word.bottom { bottom: 80px; }
.word.top    { top: 30px; }
```

JS:
```javascript
const TOP_WINDOW = { start: 0.7, end: 5.5 }; // anchored to known transcript words
const isTopWord = (w) => w.start >= TOP_WINDOW.start && w.start <= TOP_WINDOW.end;
const cls = isTopWord(w) ? "top" : "bottom";
```

**Important**: anchor TOP_WINDOW to **transcript words**, not raw seconds. If the source video gets re-edited, the words still mark the same boundaries; raw seconds drift.

For top position, ensure `top: 30px` doesn't intrude on the presenter's face. For 1024×576 with centered presenter, ~30px is safe. For other framing, leave at least 120px between caption baseline and the face.

---

## Hyperframes setup issues

### `Cannot find module 'sharp'` or sharp postinstall errors

**Symptom**: `npm install hyperframes` aborts mid-way; `node_modules/sharp/build/Release/*.node` is missing.

**Cause**: Sharp's prebuilt binary isn't compatible with your system (rolling-release distros), and the source build fails because `node-addon-api` and `node-gyp` aren't in your project's deps.

**Fix**: Use the recipe from the `video-edit-setup` skill — install deps in this exact order:
```bash
SHARP_FORCE_GLOBAL_LIBVIPS=1 npm install node-addon-api
SHARP_FORCE_GLOBAL_LIBVIPS=1 npm install node-gyp
SHARP_FORCE_GLOBAL_LIBVIPS=1 npm install hyperframes
```

Verify with `ls node_modules/sharp/build/Release/*.node` — must exist.

---

### `node: undefined symbol`

**Symptom**: `hyperframes` crashes immediately on launch.

**Cause**: Sharp's compiled binary was built against a different Node ABI than the one running. Common after switching Node versions.

**Fix**:
```bash
nvm use 22                       # or your target version
rm -rf node_modules
SHARP_FORCE_GLOBAL_LIBVIPS=1 npm install node-addon-api node-gyp hyperframes
```

---

## STT call issues

### Curl returns HTTP 401

**Cause**: API key not exported in the environment.

**Fix**:
```bash
echo $OPENAI_API_KEY      # for Whisper — must be non-empty
echo $ELEVENLABS_API_KEY  # for Scribe — must be non-empty
```

If empty, export the key in your shell environment (or source your local secrets file). Or restart Claude Code so systemd-user environment hydration runs.

---

### Curl returns HTTP 413 (Payload Too Large)

**Cause**: Source file > 25MB (Whisper limit).

**Fix**: Extract audio first:
```bash
ffmpeg -i source.mp4 -vn -acodec libmp3lame -q:a 4 audio.mp3
# then upload audio.mp3 instead
```

Or trim the video to shorter segments.

---

### Whisper MCP returns plain text, no timestamps

**Symptom**: Calling `mcp__whisper__transcribe_audio` returns the transcribed text but no `start`/`end` fields per word.

**Cause**: The MCP wrapper at `~/.secrets/mcp/whisper.sh` (server in `~/.local/share/mcp/audio-transcriber/`) doesn't pass `response_format=verbose_json` or `timestamp_granularities[]=word` to the Whisper API.

**Fix**: For word-level captions, **call the API directly with curl**, not through the MCP. The MCP is fine for plain text transcription tasks.

**Future**: Patch the MCP server to accept these parameters as optional input. The server is a simple Node script — adding the params is ~10 lines.

---

## When you're stuck

The pipeline has several places where a small misconfiguration produces a confusing symptom. When something feels wrong:

1. **Open `index.html` in a regular browser** with DevTools. Visual problems often show clearly there before you realize they were timing problems in disguise.
2. **Inspect `transcript.json` directly** with `jq`. Look for words with weird timestamps:
   ```bash
   jq '.words[] | select(.end - .start > 2)' transcript.json   # absurdly long words
   jq '.words[] | select(.start == .end)' transcript.json     # phantom words
   jq '[.words[].start] | min,max' transcript.json            # range sanity check
   ```
3. **Run `hyperframes inspect`** for a visual audit of the rendered timeline at multiple sample points.
4. **Render with `--quality draft`** even when the final will be `--quality high`. Iterate fast, polish at the end.
