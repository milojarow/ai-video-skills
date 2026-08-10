# Music + SFX — ElevenLabs

Background music and one-shot sound effects. Both endpoints are part of ElevenLabs.

---

## Music

**Endpoint:** `POST https://api.elevenlabs.io/v1/music?output_format=mp3_44100_128`
**Auth:** `xi-api-key: ${ELEVENLABS_API_KEY}`

**Request body:**
```json
{
  "prompt": "<mood description>",
  "music_length_ms": 30000,
  "force_instrumental": true
}
```

**Response:** binary MP3 audio. Save with `-o music.mp3`.

**Latency:** 5-15 seconds.

**Pricing:** ~$0.05 for a 30-second instrumental track.

### `force_instrumental: true` — always

ElevenLabs Music can generate vocal melodies if the prompt invites them. **Always set `force_instrumental: true`** for video shorts — vocals would compete with the voiceover, sounding muddy and confusing.

### Prompt patterns

The prompt describes mood and instrumentation. Keep it focused — too many adjectives produce average results.

| Tone | Example prompt |
|---|---|
| **Calm/warm/emotional** | `"calm warm reflective acoustic instrumental, family theme, reassuring, light strings, gentle piano, no vocals, soft hopeful"` (used in pilot) |
| **Tense/serious-to-hopeful** | `"tense yet hopeful cinematic instrumental, contemplative acoustic, light strings, soft piano, building to reassuring resolution at the end, no vocals"` (used in short-02) |
| **High-energy/urgent** | `"upbeat instrumental, modern cinematic, driving rhythm, strings rising, urgency without panic, no vocals"` |
| **Minimalist/contemplative** | `"minimalist instrumental, sparse piano, pad textures, contemplative, no vocals"` |

### Length

The `music_length_ms` parameter takes integer milliseconds. **Generate slightly longer than your video** — 30s for a 22s short — so the composition can trim the extra at the end without cutting the music short. The render trims music with `data-duration` in HyperFrames.

---

## SFX (Sound Generation)

**Endpoint:** `POST https://api.elevenlabs.io/v1/sound-generation`
**Auth:** `xi-api-key: ${ELEVENLABS_API_KEY}`

**Request body:**
```json
{
  "text": "<sound description>",
  "duration_seconds": 1.0
}
```

**Response:** binary MP3.

**Pricing:** ~$0.01 per short SFX.

### When to use SFX

SFX should **support** the narrative — not compete with voice or music. They work for:

- **Page-turn / paper-rustle** between visual cuts (especially papercut style — pays off the aesthetic)
- **Heartbeat** under an emotional reveal (single thump on the line that lands)
- **Soft chime** at the logo / CTA reveal (closes the piece)
- **Whoosh** on dramatic transitions (use sparingly, easy to overdo)

**Production validated:**
- Pilot: page-turn between cuts 1→2 and 2→3, heartbeat at "tu capacidad de generar ingresos", chime at logo. ✓
- Short-02: heartbeat at "Pero qué pasaría..." (the pivot moment). ✓

### When NOT to use SFX

**A tick clock for "diciembre 2026" urgency was added to short-02 — it sounded discordant against the music + voice.** Lesson: if voice and music already convey urgency, an SFX competing for the same emotional beat creates noise.

**Rule of thumb:** the SFX should fill silence or punctuate a beat that isn't already carried by voice/music. If three layers are already shouting "urgent", the SFX adds nothing.

### Prompt patterns

Be concrete. The model is precise about literal description.

| Effect | Example prompt | Duration |
|---|---|---|
| Page turn | `"soft paper rustle, page turn, short, gentle"` | 0.4–0.6s |
| Heartbeat (single thump) | `"single soft heartbeat thump, deep, organic, brief moment of pause"` | 0.8–1.2s |
| Soft chime | `"soft warm wind chime, single bell, gentle, conclusive, brief"` | 1.2–1.8s |
| Whoosh | `"smooth whoosh transition, soft air movement, brief"` | 0.3–0.5s |
| Click/tap | `"light wood tap, single, soft"` | 0.2–0.3s |

### Tips

- **One SFX per moment.** Stacking SFX gets cluttered fast.
- **2-4 SFX per short, max.** A 22-second piece doesn't have room for more than a handful of accents.
- **Always normalize/check volume.** If an SFX renders louder than peers, lower its `data-volume` in the composition (typical: 0.5–0.7).

### Sound Generation is bass-heavy — a phone cannot hear what it returns

Measured across six generations asking for a brand sting. The bias is systematic, not bad luck:

| Prompt | Energy below 300 Hz |
|---|---|
| `"bronze bell hit"` | 88.2% |
| `"two-note sonic logo"` | 97.5% |
| `"electronic brand ident"` | 67.6% — and peaked at 1.38, i.e. delivered clipped |

A phone speaker starts responding around **300 Hz**. A sting with 97% of its energy below that
**does not exist** on the device where the video is mostly watched. It sounds perfect on
headphones and like nothing on the handset — the failure mode you cannot detect listening at
the desk.

**The arbiter — run it before accepting any generated SFX:**

```python
X  = np.abs(np.fft.rfft(x * np.hanning(len(x))))
fr = np.fft.rfftfreq(len(x), 1/SR); T = X.sum()
low  = X[fr < 300].sum() / T
mid  = X[(fr >= 300) & (fr < 4000)].sum() / T
high = X[fr >= 4000].sum() / T
```

Practical floor: **>25% in 300–4000 Hz**, or the sound does not survive a phone speaker. Check
the peak in the same pass — `abs(x).max() > 1.0` means the model already handed you a clipped
file, and normalizing afterwards cannot recover what was cut off.

**Ask for brightness explicitly, by name.** The same six attempts, now requesting treble:

| Prompt | Result |
|---|---|
| `"small bright bronze bowl, crisp attack, upper midrange overtones, glassy, thin and bright not boomy, minimal bass"` | 2.2% low / 94.6% mid |
| `"tuned metal bell, glockenspiel, sharp transient, no low end, no rumble, no bass"` | 2.6% low / 35.3% mid / 62.2% high |

Words that moved it: *bright, crisp, upper midrange, glassy, thin not boomy, minimal bass, no
rumble, no low end, legible on small speakers*.

### Generate TEXTURE, synthesize MELODY

When the sound needs **specific pitches** — a note sequence, a chord, a recognizable motif —
Sound Generation is the wrong tool. It does not accept "four rising notes forming a major
chord"; it returns a texture. Forty lines of numpy give exact control over frequency, duration,
glissando and the millisecond of every note, come out deterministic (fixed seed for the noise),
and can be checked by an arbiter. **Generate for texture; synthesize for melody.**

---

## Audio mix in the composition

The voiceover, music, and SFX go into separate audio tracks in the HyperFrames composition. See `video-composition/COMPOSITION.md` for the full multi-track audio layout. Quick recap:

| Layer | Volume | Notes |
|---|---|---|
| Voiceover | 1.0 | Reference level — never lower this |
| Music | 0.30 (≈ -12 dB ducking) | Validated as right balance. 0.25 was too aggressive (pilot v1). |
| SFX | 0.5–0.7 | Per-SFX, depends on its punch |

### Ducking depth

**0.30 (≈ -12 dB) is the validated music ducking level** for a typical voiceover. The pilot v1 used 0.25 — Plan agent flagged it as "too aggressive", and the v2 fix to 0.30 resolved it.

If music feels too quiet under the voice → 0.35.
If music drowns the voice → 0.25.

This is **static ducking** (a single volume value). Sidechain-compressed dynamic ducking (the music dips automatically when voice talks) is more sophisticated but not implemented in the HyperFrames composition pipeline. For the typical 22s short, static is enough.

---

## Quick reference

```bash
# Music (30s instrumental, calm warm)
curl -sS "https://api.elevenlabs.io/v1/music?output_format=mp3_44100_128" \
  -H "xi-api-key: $ELEVENLABS_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"prompt":"calm warm instrumental, light strings, gentle piano, no vocals","music_length_ms":30000,"force_instrumental":true}' \
  -o music.mp3

# SFX (heartbeat, ~1s)
curl -sS "https://api.elevenlabs.io/v1/sound-generation" \
  -H "xi-api-key: $ELEVENLABS_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"text":"single soft heartbeat thump, deep, organic, brief moment of pause","duration_seconds":1.0}' \
  -o sfx-heartbeat.mp3
```

Both endpoints accept `Authorization: Bearer` as alternative auth, but the in-house pattern is `xi-api-key` header — keep consistency.
