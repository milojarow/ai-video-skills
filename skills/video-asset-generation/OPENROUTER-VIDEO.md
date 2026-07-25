# Animation — OpenRouter video models (image-to-video)

The second image-to-video path, alongside [ANIMATION.md](ANIMATION.md) (kie.ai Wan 2.6).
OpenRouter fronts ~17 video models through one dedicated endpoint and one key you probably
already have. For the same 1080p output it is typically **3-5× cheaper** than the kie.ai
default — the cheapest tier lands near **$0.05/s** against Wan 2.6 i2v's **$0.15/s**.

---

## API basics

Video generation is **asynchronous** and lives on its own endpoint — it is *not* the chat
completions API.

**Endpoint:** `POST https://openrouter.ai/api/v1/videos`
**Auth:** `Authorization: Bearer ${OPENROUTER_API_KEY}`

```json
{
  "model": "google/veo-3.1-lite",
  "prompt": "<motion only — see ANIMATION.md prompt patterns>",
  "duration": 8,
  "resolution": "1080p",
  "aspect_ratio": "16:9",
  "generate_audio": false,
  "frame_images": [
    {
      "type": "image_url",
      "image_url": { "url": "<public URL of the source image, PNG or JPEG>" },
      "frame_type": "first_frame"
    }
  ]
}
```

**Response:** `{ id, status: "pending", polling_url }`

**Poll:** `GET /api/v1/videos/{id}` until `status` is `completed`
**Fetch:** `GET /api/v1/videos/{id}/content?index=0` (or read `unsigned_urls[0]`)

### `frame_images` is the image-to-video control

- `frame_type: "first_frame"` — **this is i2v.** The generated motion starts from your image.
- `frame_type: "last_frame"` — pins the *ending* frame instead. Useful for a clip that has to
  land on a known composition.
- Some models additionally support reference-to-video (style/subject reference rather than a
  literal first frame).

---

## 🔑 Pricing lives on its own endpoint — the model catalog lies

The general model list (and the MCP surface over it) reports
`pricing: { prompt: "0", completion: "0" }` for every video model, because they aren't billed
per token. **That zero is not a price.** The real numbers come from:

```
GET https://openrouter.ai/api/v1/videos/models
→ per model: supported_resolutions, supported_aspect_ratios, pricing_skus
```

**Query it before every model choice.** The list and the prices move.

### Snapshot — 16:9, 1080p, no audio (verified 2026-07-25)

| Model | $/sec | 8 s clip |
|---|---|---|
| **`google/veo-3.1-lite`** | **0.05** | **$0.40** |
| `minimax/hailuo-2.3` | 0.082 | $0.65 |
| `google/veo-3.1-fast` · `alibaba/wan-2.7` | 0.10 | $0.80 |
| `alibaba/wan-2.6` i2v | 0.15 | $1.20 |
| `google/veo-3.1` | 0.20 | $1.60 |
| `x-ai/grok-imagine-video-1.5` | 0.25 | $2.00 |
| `openai/sora-2-pro` | 0.50 | $4.00 |

### Reading that table

- **Default to the cheap tier for subtle motion.** A camera push, breathing, a light shift —
  that's what most short-video segments and web heroes actually need. The premium tiers price
  in acting and complex physics you aren't using. Veo 3.1 Lite is ~3× cheaper than Wan 2.6
  i2v for the same 1080p.
- **The most-talked-about model is not the cheap one.** `grok-imagine-video-1.5` is the
  priciest of the realistic tier at 5× the cheap tier, and its catalog row doesn't even
  declare aspect ratios — verify before you plan a vertical around it.
- **Avoid token-priced video models** (e.g. Seedance-class models billed in `video_tokens`)
  unless you specifically need one: cost is unpredictable *before* you generate, which
  defeats the point of budgeting a short.

---

## Choosing between OpenRouter and kie.ai

| Consideration | OpenRouter | kie.ai |
|---|---|---|
| Price for equivalent 1080p i2v | Lower, often 3-5× | Higher |
| Model discovery | `GET /api/v1/videos/models` — real catalog with resolutions, ratios, prices | **No public catalog endpoint.** `/api/v1/jobs/models`, `/api/v1/models`, `/api/v1/jobs/modelList` all 404 |
| Balance check | Account dashboard / usage API | `GET https://api.kie.ai/api/v1/chat/credit` |
| Model breadth | ~17 video models, one contract | Whatever the docs happen to mention |

You can't discover what kie.ai offers programmatically; you can on OpenRouter. That alone
justifies keeping both documented — but reach for OpenRouter first and let price decide.

---

## Input format: PNG or JPEG only

Veo rejects webp outright:

```
Unsupported image format. Expected JPEG or PNG
```

The upload succeeds and the URL resolves — it's the *generation job* that fails, which makes
this an annoying bug to chase. Upload input frames with the ephemeral `video-lab` vault
(`LUNA_KEY_VIDEOLAB`), which doesn't transcode. See [LUNA-CDN.md](LUNA-CDN.md).

---

## Prompting

Identical rules to [ANIMATION.md](ANIMATION.md) — describe **motion, not the scene**; include
`"No new elements appear."`; add the anti-lip-sync clauses when people are visible.

Two things worth knowing:

- **`"No new elements appear."` is a strong hint, not a hard constraint.** Observed on a Veo
  clip that otherwise honored every clause: dust/lens-particle motes drifted in from around
  the 4-second mark. They looked good, so it wasn't worth a regeneration — but don't assume
  the clause is absolute, and inspect long clips before delivery.
- **Set `generate_audio: false`** whenever the voiceover comes from a TTS provider. Same
  reasoning as never enabling Wan's native audio: a second model's audio (and any lip-sync it
  implies) won't match the real narration.

---

## Cost sanity check against the composition

The efficiency math in [ANIMATION.md](ANIMATION.md) still applies, with one difference:
OpenRouter's `duration` is a **number of seconds**, not Wan's `'5' | '10' | '15'` enum, so
there's no forced 5-second minimum to waste. Ask for the duration the segment actually needs.

Pair that with the ping-pong loop trick (see the `video-composition` skill) and an 8-second
generation covers 16 seconds of seamless looping playback — for web heroes that's the
cheapest second you'll ever buy.
