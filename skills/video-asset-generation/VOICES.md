# Voices — ElevenLabs TTS

Voiceover generation. Mexican-Spanish voices validated with real audio in production.

---

## API basics

**Endpoint:** `POST https://api.elevenlabs.io/v1/text-to-speech/{voice_id}?output_format=mp3_44100_128`
**Auth:** `xi-api-key: ${ELEVENLABS_API_KEY}` header

**Request body:**
```json
{
  "text": "<el guion completo>",
  "model_id": "eleven_multilingual_v2",
  "voice_settings": {
    "stability": 0.5,
    "similarity_boost": 0.75
  }
}
```

**Response:** binary MP3 audio. Save with `-o voiceover.mp3`.

**Latency:** 2-5 seconds for a typical 20-30s voiceover.

---

## The four Mexican-Spanish voices in this project

These are the professional voices in the operator's ElevenLabs library — they resolve inside the account whose `ELEVENLABS_API_KEY` you use. Filter your selection by the use case, not just availability.

| Voice ID | Name | Description | Best for | Validated in |
|---|---|---|---|---|
| `htFfPSZGJwjBv1CL0aMD` | **Antonio** | Confident, gentle, clear; young | Pilots and intro shots; warmth + clarity for "vende tranquilidad"-style messages | Pilot short (`short-01`) |
| `spPXlKT5a4JMfbhPRAzA` | ~~**Camila**~~ DEPRECATED | Inviting, rich, warm; young; narrative_story | ⚠️ **DEPRECATED 2026-05-10** — voice volume too low; even with music ducked to 0.30 (-12dB), background music drowns her out. Loses energy after 20s anyway. **DO NOT USE** until replaced with another Spanish-female voice. | Pre-tested + used in batch-10 shorts 01/03 (renderized but deprecated post-review) |
| `ojGau167OE7nUATM1xDa` | **Marto** | Crisp, conversational; middle-aged; Mexican | Authority + sustained articulation for longer scripts (>20s). Mature voice that doesn't tire. | Short-02 (full 20.85s VO) |
| `y2ijeTfmnXjzheHO6zeN` | **Jose** | Pleasant announcer's voice; confident; young | High-energy / urgent CTAs. Not yet used. | None yet |

The voice IDs are stable — drop them straight into the API URL. They're not secrets, but they only work with the owning account's API key.

---

## Voice selection — decision framework

Ask three questions about the script:

1. **Need a female voice?** Camila is DEPRECATED (volume too low). Pre-test a fresh Spanish-female voice from ElevenLabs Voice Library before production. Filter by Spanish (Mexico) + Female + Confident/Warm.
2. **Length?** If >20s, choose a voice that holds energy (Marto validated for this).
3. **Tone needed?** Warm/empathic → Antonio (or new female voice once selected). Authoritative/sustained → Marto. High-energy/urgent → Jose.
4. **Variety across the campaign?** If the previous short used voice X, use a different voice for variety. Different gender works especially well for cross-short identity.

**Production examples:**
- Pilot used Antonio (warmth + confidence).
- Short-02 about invalidity used Marto (heavier theme, longer script, sustained articulation).
- Future shorts could use Jose for urgent/promotional, or a fresh Spanish-female voice for emotional pieces (Camila is DEPRECATED — pre-test a replacement first).

---

## Casting a new voice — measure to shortlist, one file for the human to decide

Picking a replacement voice (or a voice for a new campaign) is two steps that get confused for one: **Claude narrows the field by measurement, the human picks by ear.** Both steps have a documented failure mode.

### Step 0 — enumerate BOTH libraries, or the casting is invalid

There are two places a voice can live, and searching only one silently loses the best candidate:

| Source | How to reach it |
|---|---|
| The **account's own library** — already curated, the voices someone deliberately saved | `GET /v1/voices` with the account's `ELEVENLABS_API_KEY` |
| The **shared/community Voice Library** | the library search endpoint |

A casting run that queried only the shared library produced six candidates and missed the voice that was ultimately chosen — it lived in the account's own library, correctly labelled for the use case (`female · mexican · advertisement · professional`). **Enumerate `/v1/voices` before synthesizing a single candidate.**

### Step 1 — measure, because the catalog label doesn't decide

ElevenLabs has no "energetic" descriptor for es-MX, so the catalog blurb is not a filter. Synthesize **two** lines per candidate — a short hook *and* a ~10s enumeration, because the fatigue defect only shows up on the long one — and measure four things:

| Metric | What it catches |
|---|---|
| Pitch range (semitones, p10→p90) | Flat voice = boring |
| Syllables / second | Actual energy, not an adjective |
| Peak RMS | **The known failure mode**: quiet voice gets drowned by the music bed |
| Duration | How much the timeline moves if the voice is swapped later (see `video-composition/MULTI-BEAT.md`) |

Shape of a real six-candidate run (long line):

```
  cand-A    10.40s  4.2 syl/s   8.9 st   RMS -13.5 dB
  cand-B     9.01s  4.9 syl/s   9.0 st   RMS -19.6 dB
  cand-C     9.20s  4.8 syl/s  13.4 st   RMS -22.1 dB
  cand-D     7.71s  5.7 syl/s   5.1 st   RMS -19.8 dB
  cand-E    10.40s  4.2 syl/s  11.6 st   RMS -20.3 dB
  cand-F    12.03s  3.7 syl/s   6.1 st   RMS -24.9 dB
```

Peak RMS is the column that matters most here: the deprecation above happened because a voice was too quiet to survive the music bed. A candidate ~10 dB below its peers will reproduce that exact problem.

**The measurement does not choose — it filters.** Six candidates down to three or four finalists; the decision is still by ear.

### Step 2 — deliver ONE audition file, not a pile of attachments

Sending N separate audio files is not an audition: it forces download, a player, remembering which was which, and comparing against the previous one from memory. The delivery that works:

- **One file, ~50s, that plays straight through.**
- Each finalist **says its own name** first, then reads the same line. That removes the mental bookkeeping — the answer comes back as "the second one" without re-opening a list.
- **Mixed with the music bed at production level, not dry.** Auditioning dry hides the exact defect being screened for (the voice being buried), which is the whole reason for the peak-RMS column.
- Include the command to play it.

**Cost:** six candidates × two lines ≈ $0.06. A full casting run costs less than regenerating one wrong voiceover — and far less than a voice swap after the composition is built.

---

## Pre-test pattern (mandatory)

**Always pre-test the brand name before generating the full voiceover.**

Why: voices interpret brand names differently, especially compounds like "AcmeSeguros". Some prefer separated forms ("Acme Seguros"), some prefer joined ("AcmeSeguros"). The wrong form sounds awkward — and at full-VO scale you've already paid $0.05+ on the wrong audio.

**Pre-test recipe:**

```bash
VOICE=<voice_id>

# Variant 1: spaced
curl -sS "https://api.elevenlabs.io/v1/text-to-speech/$VOICE?output_format=mp3_44100_128" \
  -H "xi-api-key: $ELEVENLABS_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"text":"Acme Seguros punto com.","model_id":"eleven_multilingual_v2","voice_settings":{"stability":0.5,"similarity_boost":0.75}}' \
  -o pretest-v1-spaced.mp3

# Variant 2: joined
curl -sS "https://api.elevenlabs.io/v1/text-to-speech/$VOICE?output_format=mp3_44100_128" \
  -H "xi-api-key: $ELEVENLABS_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"text":"AcmeSeguros.com.","model_id":"eleven_multilingual_v2","voice_settings":{"stability":0.5,"similarity_boost":0.75}}' \
  -o pretest-v2-joined.mp3
```

**Cost:** ~$0.01 total for both variants. Saves $0.05+ if you'd otherwise need to regenerate the full VO.

**Listen and pick.** The user does this — Claude can't audition audio. Hand it over as a single playable file, not as loose attachments: see "Casting a new voice" above for the delivery format.

**Production findings:**
- Antonio prefers **joined** ("AcmeSeguros")
- Camila prefers **spaced** ("Acme Seguros")
- The choice goes into the script.txt that's then sent to TTS for the full VO.

---

## Voice settings — what each parameter does

| Parameter | Range | Default | What it does |
|---|---|---|---|
| `stability` | 0.0 – 1.0 | 0.5 | How consistent the voice's tone stays. Higher = more monotone, lower = more emotional variance. **0.5 is the sweet spot.** |
| `similarity_boost` | 0.0 – 1.0 | 0.75 | How tightly the voice clings to its trained character. Higher = more faithful but can sound stiff. **0.75 works** for all four voices in this library. |
| `style` | 0.0 – 1.0 | 0 (off) | Adds expressiveness. Useful for dramatic scripts but can introduce inconsistency. Leave at 0 for narration. |
| `use_speaker_boost` | bool | true | Slight enhancement for the speaker's voice. Default on. |

**Tested combinations:**

- **Antonio @ 0.5/0.75**: confident gentle clear. Default — what we used in the pilot. ✓
- **Camila @ 0.5/0.75**: warm rich at start, fades on long scripts. Same — short-02 confirmed the warning.
- **Marto @ 0.5/0.75**: crisp throughout. ✓ — short-02 final.

If a voice sounds "off" in some way (lazy, robotic, rushing), the first lever to pull is `stability`:
- Sounds robotic / monotone → lower stability (try 0.3)
- Sounds inconsistent / wobbly → raise stability (try 0.7)

But change one variable at a time.

---

## Multilingual nuances

- `model_id`: **always** `eleven_multilingual_v2` for Spanish (Mexican). v1 is older, less stable.
- `language_code` (in TTS): not always required, but if you want to force it, use ISO-639-1 (`es`) for the `language` field if the API surfaces it. The TTS endpoint generally infers language from the text.
- **Scribe (transcription) uses ISO-639-3** (`spa`), which is different from ISO-639-1 (`es`). Don't mix them up.

---

## Common errors

| Error | Cause | Fix |
|---|---|---|
| `401 Unauthorized` | `ELEVENLABS_API_KEY` missing or wrong | Source it from your secrets store, verify with `echo $ELEVENLABS_API_KEY` |
| Voice sounds completely different from expected | Wrong voice_id | Check the URL — `voice_id` goes in the path, not body |
| Output very short (<5 KB) | Empty `text` field or filter rejection | Verify the JSON body has the text correctly |
| Pronunciation of brand name is awkward | Voice's tokenization of compound words | Pre-test pattern (see above), pick the natural form |

---

## Quick reference (Marto, validated for short-02)

```bash
VOICE=ojGau167OE7nUATM1xDa
SCRIPT=$(cat script.txt)

curl -sS "https://api.elevenlabs.io/v1/text-to-speech/$VOICE?output_format=mp3_44100_128" \
  -H "xi-api-key: $ELEVENLABS_API_KEY" \
  -H "Content-Type: application/json" \
  -d "$(jq -nc --arg t "$SCRIPT" '{text:$t,model_id:"eleven_multilingual_v2",voice_settings:{stability:0.5,similarity_boost:0.75}}')" \
  -o voiceover.mp3
```

**Don't transcribe your own voiceover.** To get word timestamps for caption alignment, synthesize through `/v1/text-to-speech/{voice_id}/with-timestamps` instead — it returns the audio *and* its character-level alignment in one call, exactly as generated rather than re-estimated. Scribe (`/v1/speech-to-text`) is for audio you did **not** synthesize. See `video-captions/PROVIDERS.md`.
