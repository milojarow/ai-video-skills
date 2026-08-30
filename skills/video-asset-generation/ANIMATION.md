# Animation — kie.ai Wan 2.6 (image-to-video)

Animate a static image with cinematic motion. Used in short-02 for 3 segments. The single most expensive line item in a typical short — plan it carefully.

> **Check the other provider first.** OpenRouter exposes ~17 video models through one async
> endpoint, usually at 3-5× lower cost for equivalent 1080p i2v, and unlike kie.ai it has a
> real catalog endpoint you can query for current prices. See
> [OPENROUTER-VIDEO.md](OPENROUTER-VIDEO.md). This file remains accurate for Wan 2.6 on
> kie.ai; the prompt patterns and efficiency math below apply to both.

---

## API basics

**Endpoint:** `POST https://api.kie.ai/api/v1/jobs/createTask`
**Auth:** `Authorization: Bearer ${KIE_API_KEY}`

**Request body:**
```json
{
  "model": "wan/2-6-image-to-video",
  "input": {
    "prompt": "<motion description>",
    "image_urls": ["<public URL of the source image>"],
    "duration": "5",
    "resolution": "1080p",
    "nsfw_checker": false
  }
}
```

**Response:** `{ "data": { "taskId": "..." } }` — task is async, takes 1-5 minutes.

**Poll status:** `GET /api/v1/jobs/recordInfo?taskId=X`
- Same shape as Nano Banana 2 polling
- On success, `data.resultJson.resultUrls[0]` is the MP4 URL

---

## Does the model exist? Read *which* error you got

kie.ai answers with a `422` for two completely different problems, and only one of them means
the model name is wrong:

| Response `msg` | What it actually means |
|---|---|
| `The model name you specified is not supported. Please verify your input and use one of the supported models provided by KIE.` | **The model really doesn't exist.** Only this message proves it. |
| Any complaint about the input fields — `aspect_ratio is required for text-to-video`, `This field is required` | **The model exists.** Your input is malformed. A validation error is proof the model name passed the gate. |

**The trap that makes a real model look fake:** when the image field is missing or misspelled,
kie.ai classifies the job as *text-to-video* and replies `aspect_ratio is required for
text-to-video` — even though you asked for image-to-video. That reads like "it didn't recognise
your model", and it means the exact opposite: it recognised the model and couldn't find the
image.

**Rule — before declaring a kie.ai model nonexistent:**

1. Read the exact `msg`. Only `"model name ... not supported"` proves nonexistence.
2. Open `https://docs.kie.ai/market/<family>/<task>` before guessing field names. kie.ai has no
   catalog endpoint to enumerate models ([OPENROUTER-VIDEO.md](OPENROUTER-VIDEO.md)), but it
   does publish a page per model.

Guessing field names costs one billed call per attempt and doesn't converge; the doc page takes
30 seconds.

### Verified image-input field names

The image field is not spelled the same way across models — it isn't even the same *shape*:

| Model | Image input field |
|---|---|
| `wan/2-6-image-to-video` (kie.ai) | `image_urls` — **array**, max 1 URL |
| `minimax-h3/image-to-video` (kie.ai) | `first_frame_url` — **string**; optional `last_frame_url` pins the ending frame |
| OpenRouter video models | `frame_images[].image_url.url` + `frame_type: "first_frame"` — see [OPENROUTER-VIDEO.md](OPENROUTER-VIDEO.md) |

---

## Pricing (verified 2026-05-09)

| Resolution | 5s | 10s | 15s |
|---|---|---|---|
| **720p** | $0.35 | $0.70 | $1.05 |
| **1080p** | **$0.53** | $1.05 | $1.58 |

(Top-up bonuses bring effective prices down ~10%.)

**Standard on kie.ai: 1080p, 5s = $0.53 per video.** Note this is *not* the cheapest route to
1080p i2v — a comparable OpenRouter clip runs closer to $0.05/s. Price the job on both before
committing ([OPENROUTER-VIDEO.md](OPENROUTER-VIDEO.md)).

---

## Constraints (non-negotiable)

| Constraint | Value | Implication |
|---|---|---|
| Duration | enum `'5'`, `'10'`, `'15'` (seconds) | **5 seconds is the minimum.** No 3s, no 4s. If your composition segment is shorter, you pay for unused video. |
| Resolution | enum `'720p'`, `'1080p'` | 1080p is the right call for WhatsApp Stories. 720p only if budget is tight and quality target is lower. |
| `image_urls` | max 1 | One source image per video. Multi-image input is not supported by this model. |
| Image format | JPEG, PNG, **or WEBP** | Wan accepts webp. Kling 2.6 (kie.ai) and Veo 3.1 (OpenRouter) accept only JPEG/PNG — upload input frames as PNG and the question never comes up. See [LUNA-CDN.md](LUNA-CDN.md). |
| Image size | min 256×256, max 10 MB | All papercut/cinematic images from Nano Banana 2 are well within bounds. |
| Image source | Public URL | **Must be reachable from kie.ai.** Local PNG paths don't work. Use Luna CDN. |

---

## Disabling lip-sync (critical for our use case)

Wan 2.6 has **native audio with lip-sync** as an advertised feature — it can synthesize speech and animate the mouth. **We never want this** in our pipeline because the voice comes from ElevenLabs (Marto, Camila, etc.) and lip-sync from a different model wouldn't match.

**How to disable:**

1. Don't include any audio-enabling parameter in the request body. The body shape above (no audio param) is correct — audio is off by default.
2. **Add to the prompt explicitly:**
   ```
   No lip movement. Subjects remain silent throughout. People do not speak.
   ```
   This counters the model's tendency to animate mouths anyway, since the source image may show people who *could* speak.

Validated in short-02 — three videos with people, none of them moved their mouths. ✓

---

## Never animate a frame that has text baked in

Feeding a finished poster (art + headline) to any i2v model mangles the letters — the model
re-synthesises every pixel it animates, and glyphs have no special status in that process. This
holds even when the prompt says, verbatim, that all text must stay pixel-identical and must
never be re-rendered, re-lettered, warped or restyled.

**The pipeline that works:**

1. **Remove the text from the base frame first** (image-to-image inpainting — see
   [IMAGE-EDITING.md](IMAGE-EDITING.md)). Erasing it locally isn't enough when the background
   has gradient *and* structure — a flat rectangle patch is visible immediately.
2. **Animate the text-free frame.** Nothing left with a right answer for the model to break.
3. **Generate the text separately as its own layer**, then composite it back on top of the
   rendered animation.

**Generating the text layer:** ask a text-capable image model for the copy on **pure flat
black, nothing else in frame** — no gradient, texture, or vignette. The alpha then falls out of
luminance with no thresholding argument, and the cut-out has hard edges.

Don't default to typographic composition (drawing the text with a font at render time) just
because "a model can't spell" — measured on Spanish copy with accents and a mixed-weight
editorial layout, the image model wrote it correctly (accented vowels and `ñ` included) and
matched the poster's own typographic voice, which a font substitution would not.

**Keep the text layer at native resolution.** Composite it at the size the shot needs by
downscaling the generated layer, never by upscaling it. If a camera move changes the apparent
size over time, scale from the high-resolution source on every frame — never from an
already-flattened low-res copy.

---

## Prompt patterns

The prompt describes **motion**, not the scene (the scene is the input image). Be specific about *what* moves.

### Good motion patterns

- **Camera motion**: "slow camera pull back from medium close-up to medium shot", "subtle camera dolly forward", "soft focus pull from foreground to background"
- **Subject motion**: "person's pose shifts slightly with breath", "head slowly turns toward the window", "hand places another bill on the stack"
- **Environment motion**: "calendar pages flip slowly one by one", "soft natural light gently shifts", "leaves on background trees move slightly in the breeze"
- **Mood descriptors**: "quiet ambient atmosphere", "documentary editorial style", "cinematic realism"

### Required clauses

Always include:

- `"No new elements appear."` — counters the model's tendency to introduce things not in the source image (extra people, props, lens flares)
- `"No lip movement. Subjects remain silent throughout."` — only when humans are visible
- `"Vertical 9:16 cinematic framing."` — locks the orientation

### Objects that must stay recognisable: forbid rotation *by name*

i2v models read "rotating gently on its own axis", "slowly" and "subtle" as permission to spin.
Measured on one first frame animated for 8 s, the objects completed a full 360°: their
silhouette dropped to **23% of the starting width**, and **44% of the frames** sat below the
legibility threshold. When the object *is* the message — a product, one icon per service — that
is a **content** failure, not a style one: for half the loop the hero subject is a shapeless
smear.

Softness words don't reach. Name the prohibition:

> The objects inside DO NOT ROTATE and DO NOT TURN. Each keeps exactly the same orientation as
> in the first frame, facing the camera, fully recognisable at every single moment.

Then state what *is* allowed to move, and nothing else:

> Extremely subtle motion only. The `<subjects>` float and bob very gently in place. […]
> What moves, and nothing else: the gentle vertical bob, soft specular highlights sliding
> across the glass, and the caustic shadows shifting on the floor.

Two details that made it hold on the re-run:

1. **Enumerate object by object what each one does not do** — "the phone stays flat and front
   facing", "the globe does not spin". A model generalises badly from a single abstract
   prohibition.
2. **Shorten the clip** (8 s → 6 s). Less room to drift.

Same model, same first frame, prompt-only change:

| Run | Glass / refraction | Worst object silhouette | Output |
|---|---|---|---|
| `minimax-h3/image-to-video` (kie.ai), "subtle motion / gently" | better | ❌ 23% — 44% of the loop illegible | 2944×1248 |
| A second-provider i2v model, same frame | paler edges | ✅ 89% | 2176×928 |
| `minimax-h3/image-to-video` + anti-rotation prompt | better | ✅ **99%** | 2944×1248 |

**The fix is the prompt, not a different model.** The model with the better glass was also the
one that spun the objects; with rotation forbidden it won both columns. Write the prohibition on
the *first* attempt — i2v retries are billed, and a run can die mid-iteration on
`{"code":500,"msg":"Credits insufficient"}`.

### The prohibition does NOT extend to "it turns, but only within an arc"

Forbidding rotation by name works for a piece that must not turn **at all**. It does
not transfer to the other case — a needle, a hand, a dial that *does* rotate, but must
stay inside a bounded arc. Measured twice on `wan/2-6-image-to-video`, 1080p, 5 s,
animating a gauge needle:

| Attempt | Prompt | Needle's angular travel |
|---|---|---|
| v1 | describes the desired motion (tachometer sweep with bounce), forbids rotation on the tile and the ring | **356.7°** out of 360 |
| v2 | + an explicit, named angular limit: "NEVER completes a circle, NEVER spins, NEVER passes 12 o'clock, NEVER passes 6 o'clock, its whole travel stays between 7 and 1 o'clock" | **352.1°**, and the needle **fragments** and detaches from its pivot |

The model treats any articulated piece as a clock hand. The explicit prohibition does
not shorten the travel — on the second run it additionally degraded the integrity of
the piece. Cost of that lesson: 4 renders × $0.53 = $2.12.

**The rule, sibling of "a generative model cannot land a move on the exact second":**

> If the value of the shot is an object reaching a **specific angle**, or staying
> inside a **bounded arc**, the generative i2v model is the wrong tool for that
> motion — exactly as when what matters is the INSTANT. It is material, not time and
> not geometry.

Rotate the piece deterministically from the flat source art instead — see
[DETERMINISTIC-ROTATION.md](DETERMINISTIC-ROTATION.md).

#### Measuring angular travel (a 9-frame contact sheet lies)

Per-frame angle series plus `np.unwrap`, so the ±180° wrap does not fake the range:

```python
# per frame: color-mask the piece, tip = pixel farthest from the pivot
tip = max(pts, key=lambda p: (p[0]-cx)**2 + (p[1]-cy)**2)
ang = math.degrees(math.atan2(cy-tip[1], tip[0]-cx))
...
u = np.degrees(np.unwrap(np.radians(np.array(angs))))
travel = np.nanmax(u) - np.nanmin(u)      # >=330 => it went all the way around
loop   = abs(u[0] - u[-1])                # how cleanly the loop closes
```

**Mandatory positive control before believing the meter:** build a static video from
the source image (`ffmpeg -loop 1 -i still.jpg -t 2`) and run it through the same
meter. It must report `travel = 0.0°`. Without that control a "bounded arc" is not
evidence — the mask may simply not be finding the piece, and an empty range reads as a
small one.

### Measuring rotation without eyeballing it

A 4-frame contact sheet only hints; this proves it. Crop the object's region, count the width of
its silhouette frame by frame, compare against frame 0 — **an object that turns gets narrower.**
Below 80% of the starting width it already reads badly.

```python
cols = [x for x in range(W) if min(px[x, y] for y in range(0, H, 3)) < 150]
width = max(cols) - min(cols)      # as a % of the same measure on frame 0
```

That gives the exact percentage of the loop that's ruined instead of a judgement call.

### Real prompts from short-02

**Seg 4 — protagonist alone, contemplative:**
```
Cinematic realism. The man sitting still slowly turns his head toward the window with subdued thoughtful expression. Soft natural light gently shifts through the window. Slow camera pull back from medium close-up to medium shot. Quiet ambient atmosphere. No new elements appear. Person does not speak. No lip movement. Subjects remain silent throughout. Vertical 9:16 cinematic framing.
```

**Seg 5 — calendar + bills accumulating:**
```
Cinematic realism. On the desk, the wall calendar slowly flips to reveal more days passing. The mans hand places another paper bill on top of the existing stack of bills. Subtle camera dolly forward focusing on the growing pile of papers. Documentary editorial style. No new elements appear. Person does not speak. No lip movement. Subjects remain silent throughout. Vertical 9:16 cinematic framing.
```

**Seg 6 — protagonist with family, calm:**
```
Cinematic realism. The man at home with his family, his calm expression slowly growing into a slight peaceful smile. His wifes hand stays reassuringly on his shoulder. Soft natural daylight from the window in the background. Slow gentle camera movement. Documentary editorial style. No new elements appear. People do not speak. No lip movement. Subjects remain silent throughout. Vertical 9:16 cinematic framing.
```

The actual MP4s are at `~/video-lab/<topic>/<video-name>/<variant>/videos/` for reference.

---

## The base frame fixes the field of view

A camera move that "pulls back to reveal the whole composition" only works if the base frame
already contains what it pulls back to. The i2v job only ever has pixels for the crop it was
given — a zoom-out beyond that crop has nothing to reveal, no matter how the prompt words it.

**The rule:** decide the widest framing the finished piece will ever show, **then** build the
base frame with that margin, **then** animate. If the piece opens tight and settles wide, the
wide end is what the model must be given at submission time — that's fixed and no
post-processing recovers it.

**Consequence for resolution:** a zoom that starts at `z` and ends at `1.0` upscales the
opening frames by `z`. At 1080p output, `z = 1.78` means the opening shows only ~600px of real
detail. Two ways to keep it sharp: request a higher output tier so the tight end is still
oversampled, or pick a smaller `z` — measure what `z` actually reproduces the framing you want
to open on (it's an exact ratio of two crop widths, not a guess).

**Squares into 9:16:** a square artwork can't fill a 9:16 frame at full width without vertical
bands. Either extend the art before animating (see [IMAGE-EDITING.md](IMAGE-EDITING.md)'s
outpaint section), or accept the bands **inside the base frame** so the model animates them too
— bands added after the fact stay frozen while the centre moves, and the half-animated cells
straddling the seam are visible.

---

## Efficiency math (very important)

**Wan's minimum is 5 seconds, but your composition segment may be shorter.** Any time the segment is <5s, the unused portion of the video is cost wasted.

### Short-02 actuals

| Segment | Composition duration | Wan video paid for | Used / Paid |
|---|---|---|---|
| Seg 4 | 4.10s | 5s | 82% |
| Seg 5 | 3.00s | 5s | 60% |
| Seg 6 | 2.90s | 5s | 58% |
| **Total** | **10.0s shown** | **15s paid** ($1.59) | **67% efficient — $0.53 wasted** |

### Strategies to maximize efficiency

1. **Write the script so animated segments last ~4.5-5s.** That nearly maximizes use of each Wan video. Trim ~0.2-0.5s for the cross-fade tail (HyperFrames cross-fade eats the last 0.2s). 5s minimum implies you should target 4.5-4.8s segments to use 90%+.

2. **Use `data-media-start` to pick a different slice.** If the most interesting motion in your generated video is between seconds 1-5 instead of 0-4, set `data-media-start: 1` on the `<video>` element so the composition shows seconds 1-5 instead of 0-4. This is more advanced and not yet validated in production, but the parameter exists in HyperFrames.

3. **Reuse one Wan video at multiple `data-media-start` offsets.** Theoretically, one 10s Wan video could feed two segments using different slices — at $1.05 it's cheaper than two 5s videos at $1.06. **Untested in production.**

---

## A generative model CANNOT land a move on the exact second of a word

This question comes back every time a shot looks "really animated" — why not just use a text-to-video / image-to-video tool for the whole thing? The answer is not about quality. **It's about interface.**

**There is nowhere to put the time.** The input to a generative video model is: a prompt, optionally a first frame, a duration and an aspect ratio. It does not accept:

- an audio track to align against,
- a per-object identity ("this one is object 3"),
- a list of instants ("object 3 comes forward at 2.80s").

Ask for "each object comes forward in turn" and you get plausible motion at arbitrary instants. **And the instant IS the shot**: if an object arrives half a second early, the viewer sees one object approaching while the voice names a different one, and the shot stops meaning what it meant. Render quality is irrelevant once sync is lost.

### The split that works

| Piece | Who does it | Why |
|---|---|---|
| The objects (loose assets, clean alpha) | generative model | it's material, and the model is unbeatable at material |
| The motion and its instant | hand-authored timeline (GSAP + word-level alignment) | it's time, and time gets declared |

It's also cheaper: a good still costs cents and iterates fast; a generated clip costs an order of magnitude more and can't be corrected without regenerating the whole thing.

There's a benefit that only shows up later: **the cues stay editable.** When a voice swap forced a resync, repatching six numbers fixed the entire shot. A baked generated clip has no such repair — you throw it away and pay again (see `video-composition/MULTI-BEAT.md`).

### The rule

> If the value of the shot is in **WHEN** something happens relative to the audio, the generative model is the wrong tool for the motion — and the right tool for the material.

**Generator is right for:** textures, living backgrounds, an object floating, a camera push — any motion whose exact instant doesn't matter.
**Generator is wrong for:** hits on a beat, entrances on a word, counts, lyric sync.

---

## When to animate vs keep static

| Situation | Recommendation |
|---|---|
| Segment <2s | **Static.** Animation cost wasted, viewer doesn't have time to register motion. |
| Segment 2-3s, motion adds narrative | **Borderline.** Static usually wins — consider it carefully. |
| Segment 3-4s, motion adds narrative | **Animate.** Worth the spend. |
| Segment 4-5s | **Animate.** Sweet spot. |
| Segment 5-10s | Animate at 5s and use `data-duration < 5` to trim, or generate 10s for full coverage. |
| Static asset (CTA card, calendar close-up) | **Static.** Animation distracts from the message. |

---

## Hosting requirement: Luna CDN

Wan needs a public URL for the input image. Local files don't work. Luna CDN is the project's standard.

**Upload input frames with `LUNA_KEY_VIDEOLAB`** — the ephemeral `video-lab` vault. It skips transcoding (PNG stays PNG) and auto-deletes after 24 h, which is exactly the lifetime an input frame needs. A per-client key (`LUNA_API_KEY`) converts images to webp: Wan tolerates that, but Veo and Kling reject it outright, so the ephemeral vault is what buys you provider portability.

⚠ There is **no** `LUNA_API_KEY_VIDEOLAB_RAW`. Earlier revisions of this skill named it; it resolves to an empty string.

See [LUNA-CDN.md](LUNA-CDN.md) for the upload pattern and the vault decision rule.

---

## Common errors and fixes

| Error | Cause | Fix |
|---|---|---|
| `401 Unauthorized` | Missing or invalid `KIE_API_KEY`, or IP whitelist enabled with your IP not in it | Verify env var. In kie.ai panel, check API key whitelist: empty whitelist = "all IPs allowed", any IP = restrictive. |
| `502 Bad Gateway` (rare) | kie.ai upstream issue | Re-submit. Check kie.ai status. |
| Status stuck on `waiting` for >10 min | Provider queue overload, or invalid image URL | Test the URL with `curl -I` from another shell — should return 200. If it does, just wait or re-submit. |
| `400 Validation Error: image too small` | Source image <256×256 | Use a higher-resolution source. Nano Banana 2 outputs are 768×1376 and well above the minimum. |
| Generated MP4 includes elements not in source image | Prompt didn't constrain enough | Add `"No new elements appear."` to the prompt. Regenerate. |
| Person in video moves mouth | Lip-sync disable wasn't strong enough | Add stronger negation: `"No lip movement. No mouth animation. Subjects remain silent. People do not speak."` Multiple phrasings reinforce. |
| Output dimensions slightly off (e.g., 1076×1926 instead of 1080×1920) | Provider rendering quirk | Use `object-fit: cover` in the composition's `<video>` styling. The few-pixel difference is invisible after composition resize. |

---

## Polling pattern (Bash)

```bash
KIE_API_KEY="${KIE_API_KEY:?export it from your secrets store first}"
TID="<taskId from submit>"

# Poll
for try in 1 2 3 4 5 6 7 8 9 10; do
  RESP=$(curl -sS "https://api.kie.ai/api/v1/jobs/recordInfo?taskId=$TID" \
    -H "Authorization: Bearer $KIE_API_KEY")
  STATE=$(echo "$RESP" | jq -r '.data.state')
  if [ "$STATE" = "success" ]; then
    URL=$(echo "$RESP" | jq -r '.data.resultJson | fromjson | .resultUrls[0]')
    curl -sS -L "$URL" -o video.mp4
    echo "Downloaded video.mp4"
    break
  elif [ "$STATE" = "failed" ]; then
    echo "Generation failed"
    break
  fi
  sleep 20
done
```

Run multiple `submit` calls in series (each takes <2s), then poll all taskIds in parallel inside the loop.
