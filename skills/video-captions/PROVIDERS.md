# STT Providers — Whisper vs Scribe

Deep reference for choosing and using each provider. Read this when deciding which to use for a project, or when integrating a new one.

---

## First: is this audio yours? Then don't transcribe it at all

**If you synthesized the audio with ElevenLabs TTS, the alignment already exists — asking an STT model for it is asking a second model to guess what the first one said.**

Use the timestamped TTS endpoint instead of generating and then transcribing:

```
POST https://api.elevenlabs.io/v1/text-to-speech/{voice_id}/with-timestamps
```

Same request body as normal TTS. The response is JSON carrying `audio_base64` **plus** `alignment`:

```json
{
  "audio_base64": "...",
  "alignment": {
    "characters": ["T","u"," ","n","e","g","o"],
    "character_start_times_seconds": [0.0, 0.081, 0.139],
    "character_end_times_seconds":   [0.081, 0.139, 0.174]
  }
}
```

That is the character-by-character alignment **of the audio just synthesized** — the data the model generated the audio from, not an estimate of it.

**Why this is strictly better, not just shorter:** the round-trip through STT costs an extra call, adds latency, and *can mishear* precisely where it hurts most — figures, brand names, spelled-out acronyms. A karaoke caption misaligned on the brand name is the worst possible place for an error.

### Characters → words

A word is a run of non-space characters: it spans from the start of the first to the end of the last.

```python
def words_from_characters(alignment, base=0.0):
    chars = alignment["characters"]
    starts = alignment["character_start_times_seconds"]
    ends   = alignment["character_end_times_seconds"]
    words, current, t0, prev_end = [], "", None, 0.0
    for c, a, b in zip(chars, starts, ends):
        if c.strip() == "":
            if current:
                words.append({"text": current, "start": base + t0, "end": base + prev_end})
                current, t0 = "", None
            continue
        if not current:
            t0 = a
        current += c
        prev_end = b
    if current:
        words.append({"text": current, "start": base + t0, "end": base + prev_end})
    return words
```

`base` is the beat's offset within the global timeline, when synthesizing line by line.

This yields two things at once and for free: the caption groups **and** the animation cues tied to words ("this element enters when the voice names it" — see `video-composition/MULTI-BEAT.md`).

### The routing rule

| Audio origin | Use |
|---|---|
| **Your own TTS** (synthesized in this pipeline) | `/v1/text-to-speech/{voice_id}/with-timestamps` |
| **Anyone else's audio** — real recordings, interviews, a human voice actor, archive material | Whisper or Scribe, below. No prior alignment exists; transcription is the only option. |

**Plumbing note:** the HyperFrames Python bridge to ElevenLabs requires the `elevenlabs` package installed. This endpoint over plain REST + `curl` does the same with zero dependencies — and additionally exposes the alignment, which the bridge does not.

---

## OpenAI Whisper API

### Endpoint
```
POST https://api.openai.com/v1/audio/transcriptions
```

### Authentication
`Authorization: Bearer $OPENAI_API_KEY`

### Required parameters for word-level captions

| Parameter | Value | Why |
|---|---|---|
| `file` | `@video.mp4` | The audio source. Accepts video files directly — no need to extract audio first. Max 25MB. |
| `model` | `whisper-1` | The only model that exposes word-level timestamps via this endpoint. Newer models (`gpt-4o-transcribe`, `gpt-4o-mini-transcribe`) don't. |
| `language` | ISO-639-1 (`es`, `en`, ...) | Improves accuracy. Required when audio language is non-English to avoid translation. |
| `response_format` | `verbose_json` | Default `json` returns plain text. Verbose includes timestamps, language, duration. |
| `timestamp_granularities[]` | `word` | Without this, timestamps are by segment (~5-10s blocks), not by word. |
| `prompt` | (optional) | Free-text hint of vocabulary. Use for proper nouns: `"El locutor es <nombre completo del locutor>, agente de seguros..."`. Whisper uses it as a contextual nudge — not a hard constraint. |

### Response shape

```json
{
  "task": "transcribe",
  "language": "spanish",
  "duration": 51.43,
  "text": "...",
  "words": [
    { "word": "Hola", "start": 0.32, "end": 0.51 }
  ],
  "usage": { ... }
}
```

### Quirks

1. **Timestamps lag the audio onset by ~150ms.** Words appear consistently late. Apply a global `SHIFT = -0.15` in post-processing. Empirically validated on Spanish, may differ for other languages.

2. **Phantom words (zero duration).** Occasionally `start == end` for a real word. ~10 in a 51s clip in our test. Resolve via split-forward (see PIPELINE.md).

3. **Proper nouns it has never seen are wrong.** An unfamiliar surname comes back as a similar-sounding word it does know (`<surname>` → a near-homophone) without a prompt; with the prompt, it recovers. The prompt is the cleanest fix for known proper nouns.

4. **The `mcp__whisper__transcribe_audio` MCP server (the usual shell wrapper) does NOT pass `response_format` or `timestamp_granularities`.** It only returns plain text. For word-level captions, **call the API directly with curl**, not through the MCP. (Future improvement: patch the MCP server to accept these parameters.)

### Cost
~$0.006 USD/min. Cheapest production-grade option.

### Latency
~5-7s per call for 51s of audio. Doesn't scale linearly — overhead-dominated for short clips.

---

## ElevenLabs Scribe

### Endpoint
```
POST https://api.elevenlabs.io/v1/speech-to-text
```

### Authentication
`xi-api-key: $ELEVENLABS_API_KEY`

### Parameters

| Parameter | Value | Why |
|---|---|---|
| `file` | `@video.mp4` | Audio source. Accepts video too. |
| `model_id` | `scribe_v1` | Verified working. `scribe_v2` may exist; not tested. |
| `language_code` | ISO-639-3 (`spa`, `eng`, ...) | **Three-letter codes**, not the two-letter ISO-639-1 Whisper uses. |
| `tag_audio_events` | `false` | Default `true` includes `[music]`, `[laughter]`, etc. Set `false` for clean captions. |
| `biased_keywords` | `<surname>:5` (optional) | Bias toward specific words. **Did not work for proper nouns in our test** — Scribe still output the near-homophone. Treat as unreliable. |
| `additional_formats` | `["srt"]`, etc. (optional) | Returns formatted exports alongside JSON. Useful only for direct SRT delivery. |
| `diarize` | `true` (optional) | Speaker diarization. Not relevant for single-speaker captions. |

### Response shape (different from Whisper!)

```json
{
  "audio_duration_secs": 51.43,
  "language_code": "spa",
  "language_probability": 0.99,
  "text": "...",
  "transcription_id": "...",
  "words": [
    { "text": "Hola", "start": 0.299, "end": 0.399, "type": "word", "logprob": -0.0006 },
    { "text": " ", "start": 0.399, "end": 0.459, "type": "spacing", "logprob": -0.0001 }
  ]
}
```

### Adapter notes

| Whisper field | Scribe field |
|---|---|
| `duration` | `audio_duration_secs` |
| `language` | `language_code` |
| `words[].word` | `words[].text` |
| (n/a) | `words[].type` — `"word"` or `"spacing"`. **Filter `type === "word"`**. |
| (n/a) | `words[].logprob` — confidence per word. Useful for flagging low-confidence words for human review. |

The build script in PIPELINE.md auto-detects which provider via shape.

### Quirks

1. **Timestamps are accurate.** First-word onset matches audio reality (~0.3s), not lazy 0.0s. **No SHIFT needed.**

2. **No phantom words.** In our test, Scribe emitted 0 zero-duration words vs Whisper's ~10. The phantom-resolution code is still good to have (defensive), but in practice triggers only for Whisper.

3. **`biased_keywords` doesn't fix proper nouns.** Tested by biasing an unfamiliar surname at weight 5 — Scribe still returned the near-homophone. Use post-processing `VOCAB_FIXES` instead.

4. **Skips words occasionally.** In our test, Scribe omitted the conjunction "y" from "planear tu retiro y disfrutar de una vejez". No general fix — needs human review of transcripts before final render. (Whisper had its own omission: "ahorro" disappeared in a different sentence. Both providers have this failure mode at low rates.)

5. **`logprob` is per-word.** Unlike Whisper, Scribe exposes confidence. Useful for automating review: words with `logprob < -0.5` (or whatever threshold) get flagged for human inspection.

### Cost
~$0.04 USD/min. ~6.7× more expensive than Whisper.

### Latency
~4-5s per call for 51s of audio. Slightly faster than Whisper in our test.

---

## Decision matrix

| Need | Pick |
|---|---|
| Final delivery, accuracy first | **Scribe** |
| Iterating, cost-conscious | Whisper |
| Bulk processing (10+ hours of audio/month) | Whisper, post-process aggressively |
| Audio in tonal language (Mandarin, Vietnamese, etc.) | Test both — neither documented well, empirical |
| Need per-word confidence scores | **Scribe** (only one that exposes `logprob`) |
| Need SRT/VTT output directly | Either (`additional_formats` on Scribe, or convert JSON to SRT in post for Whisper) |
| Need speaker diarization | Scribe (`diarize: true`) — Whisper doesn't expose it |

---

## Hybrid approach (worth considering)

For a production pipeline:

1. **First pass with Whisper** (cheap) to get the bulk transcript.
2. **Second pass with Scribe** on the same audio.
3. **Diff the two transcripts.** Words that appear in one but not the other go to a human-review flag list.
4. **Use Scribe's timestamps** (more accurate) but Whisper's text **only** when Scribe's `logprob` is below a threshold and Whisper's word matches expected vocabulary.

This costs ~$0.046/min total but produces near-zero omissions/mistranscriptions. Worth it for client-delivered work.

We didn't implement this for the experiment — it's noted here as the obvious next iteration.

---

## Future providers to evaluate

- **Deepgram Nova** — competitive on accuracy, has streaming, similar pricing to Scribe.
- **AssemblyAI** — has speaker diarization, sentiment, content moderation built-in.
- **whisper.cpp local** — free if compute is free, slower iteration. Useful for offline / privacy-sensitive jobs.
- **WhisperX** — Whisper with forced alignment. May fix the timestamp lag without a global shift heuristic.

When evaluating a new provider, the integration touchpoints are minimal: the build script's normalization step (filter, rename) is the only place that needs adapter code.
