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
