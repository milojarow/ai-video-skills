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

## Synthesizing a sonic logo

### A brand sting that ENDS LOW reads as failure — and it is measurable

A four-note logo sting came out with good timbre, good level and a clean tail, and was
rejected on first listen as sounding like a *defeat* cue — the sound a game makes when the
player loses something. The timbre was not the problem. The pitches were:

    566 → 916 → 716 → 1150 → 322 Hz
                                ↑ the last one, nearly an octave BELOW all the others

**Landing on the lowest note is the sonic signature of loss** — it is what error chimes and
"womp womp" stings do. Achievement jingles do the opposite without exception: they ascend and
land at the top. Fixed with an ascending arpeggio resolving into a major chord:

    383 → 516 → 650 → 795 Hz   (a triad resolving on the major chord)

### The arbiter — automate it, don't listen for it

Segment by RMS envelope, take the dominant pitch of each segment, and require the last to be
the maximum. It is a binary gate; the script exits non-zero when it fails.

```python
m    = (fr > 250) & (fr < 2600)      # melodic range, above the sub
note = int(fr[m][np.argmax(X[m])])   # per segment, split at the attacks
assert notes[-1] == max(notes), "ends low = failure signature"
```

### Two traps this arbiter caught that the ear could not have explained

1. **A reinforcement layer an octave down eats the resolution.** Doubling the tonic below the
   final chord for body made the detector measure the piece landing at 533 Hz — the same pitch
   as the first note, i.e. no ascent at all. Put the low-end weight in a **sub outside the
   melodic range (55–65 Hz)**, where it cannot compete for being "the note".
2. **A single note sounds like a beep, not like a brand.** Resolving on a chord — three tones
   together — is what makes it read as a logo. Cheap: three more oscillators.

### Placing it in a mix

Two rules that apply the moment the sting sits over an existing bed:

- **Broadband RMS lies against a tonal element over a noise bed.** It averages energy
  concentrated in a few narrow bands against energy spread across the whole spectrum. In a real
  case the notes measured only ~3 dB over the ambience broadband — reads as "buried" — while a
  band split showed **+16 dB in 250–700 Hz**, where their fundamentals live. Measure the band
  of the fundamentals before adding gain to a tonal element.
- **Measure the WINDOW the element falls in, not the track.** A bed described as "living in the
  upper mids" — true for the full track — measured **94.7% below 300 Hz** in the exact 2.4 s
  window of the hit: a low-pulse section, no melody, zero overlap. A three-minute average does
  not describe two seconds.
- **A number that does not move proves nothing; only a marker that RISES is proof.** Adding the
  sting to a mix left its peak and mean unchanged to the decibel, because the programme peak
  lived in the spoken section far from the sting — which reads exactly like "the sting never
  landed". Confirm in the positive, by band: 300 Hz–3 kHz went from 5.3% to 17.6% in that
  window.

### Measure the file you WROTE, never the buffer you encoded

A synthesized signal normalized to **0.71 linear = −2.97 dBFS** and written with

```bash
ffmpeg -f f32le -ar 48000 -ac 1 -i -   -ar 48000 -ac 2 -y out.wav
```

was documented as "peaks at −3 dBTP". The delivered file peaks at **−5.99 dBFS**. Somebody
else caught it by measuring the file.

**Cause:** ffmpeg's mono→stereo upmix preserves **power**, not amplitude, and spreads −3 dB per
channel. Correct by design and completely silent — no warning, valid file, and the difference
only appears if you measure the result.

    mono signal before writing    −2.97 dBFS
    the written stereo WAV        −5.99 dBFS
    that same WAV folded to mono  −2.97 dBFS   ← the energy is there, split

−6 dBFS is fine headroom for a sting; the file was not wrong. **The claim** was. Whoever mixes
the asset derives their gain from the number you gave them, so a 3 dB error starts their
calculation crooked — and "fixing" the level after someone already mixed against the real file
changes the sound of their piece without anything failing.

```python
raw = subprocess.run(["ffmpeg","-v","error","-i",f,"-f","f32le","-ar","48000","-"],
                     capture_output=True).stdout
peak_db = 20*np.log10(abs(np.frombuffer(raw, dtype=np.float32)).max())
```

Between the buffer and the file there is a resample, a channel change, a quantization and
sometimes a dither — any of them moves the peak. And when the wrong number already went out to
third parties: **correct the documentation, not the file**, if anyone has already derived a mix
from it.

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
