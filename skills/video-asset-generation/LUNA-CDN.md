# Luna CDN — file hosting for video-lab

Public URL hosting for the assets that downstream APIs (like Wan 2.6 or an OpenRouter video
model) need to fetch.

---

## API basics

**Base URL:** `https://<your-luna-host>/api` — the host is per-deployment; keep it in
`LUNA_BASE_URL` rather than hardcoding it.
**Auth:** `X-API-Key` header. The key *is* the client identity — it selects which vault the
upload lands in and which processing pipeline runs.

---

## Keys and vaults

> **`LUNA_API_KEY_VIDEOLAB_RAW` does not exist.** Earlier revisions of this skill told you to
> use it "so PNG stays PNG". There is no such variable — exporting it gives you an empty
> string, and the upload goes out unauthenticated or falls back to whatever key the shell
> already had. The correct variable is **`LUNA_KEY_VIDEOLAB`**.

| Env var | Vault | Behavior |
|---|---|---|
| **`LUNA_KEY_VIDEOLAB`** | the shared **`video-lab` client** — **ephemeral** | **No transcoding** (PNG stays `.png`) and uploads **auto-delete after 24 h** |
| `LUNA_API_KEY` (per project, in that project's own env file) | that client's own vault | Images re-encoded to `.webp`; files are permanent |
| `LUNA_ADMIN_API_KEY` | admin scope | Vault administration — not for uploads |

**Verify what a key points at** before trusting it:

```bash
curl -sS "$LUNA_BASE_URL/me" -H "X-API-Key: $LUNA_KEY_VIDEOLAB"
```

The ephemeral client says so in its own response — "uploads bypass transcoding and
auto-delete after 24h". That single call settles both questions (format preservation and
retention) without a test upload.

---

## Decision rule: which vault for which asset

**Input frames for generation → the `video-lab` ephemeral vault (`LUNA_KEY_VIDEOLAB`).**

An image-to-video provider needs only one thing from you: a public URL it can fetch *right
now*. The ephemeral vault gives exactly that, and nothing more:

- **No format surprise** — the frame stays PNG, so format-strict models accept it.
- **No cross-vault contamination** — a frame derived from one client's material never lands
  in another client's vault; the one-vault-per-client rule stays intact.
- **No accumulated garbage** — 24 h retention means a real client vault doesn't slowly fill
  with throwaway intermediate frames.

**Final deliverables → the client's own vault (`LUNA_API_KEY`).** That's where permanence is
wanted. Note the asymmetry there: `.mp4` passes through untouched, but a `.jpg` poster
**is** converted to `.webp`. Irrelevant for a `<video poster=…>` (browsers accept webp), but
know it before you assume the extension you uploaded is the one you get back.

**Do not reach for a third-party paste host** (throwaway file-drop services) to serve a PNG a
model refused. It's an unnecessary exposure of client-derived material, and the ephemeral
vault already solves the problem.

---

## Why the format matters: webp breaks half the i2v catalog

| Model | webp input |
|---|---|
| Wan 2.6 (kie.ai) | ✅ accepted |
| Veo 3.1 (OpenRouter) | ❌ `Unsupported image format. Expected JPEG or PNG` |
| Kling 2.6 (kie.ai) | ❌ JPEG/PNG only |

The failure mode is late and confusing: the upload succeeds, the URL resolves, and the
*generation job* is what fails. Upload input frames with `LUNA_KEY_VIDEOLAB` and the whole
class of error disappears.

---

## Upload

```
POST /files/upload     # multipart/form-data
```

- Field `files` (can repeat — up to 20 files per request)
- Max **500 MB** per file

```bash
curl -sS -X POST "$LUNA_BASE_URL/files/upload" \
  -H "X-API-Key: $LUNA_KEY_VIDEOLAB" \
  -F "files=@image1.png" \
  -F "files=@image2.png" \
  -F "files=@image3.png"
```

**Response (HTTP 201):** JSON array, same order as input files:

```json
[
  { "cdn_url": "https://<your-luna-cdn-host>/<uuid-1>.png" },
  { "cdn_url": "https://<your-luna-cdn-host>/<uuid-2>.png" },
  { "cdn_url": "https://<your-luna-cdn-host>/<uuid-3>.png" }
]
```

If a single file fails, that item carries `{ "error": "...", "original_name": "..." }` and the
others succeed. **Always filter by presence of `cdn_url`** before assuming success.

These URLs are passed verbatim to the provider's image input (Wan 2.6 `image_urls`, or an
OpenRouter `frame_images[].image_url.url`).

---

## URL behavior

The returned `cdn_url`:

- Lives on the CDN host (separate from the upload endpoint)
- **No auth.** Anyone with the URL can fetch.
- **Stable for life** of the file (UUID-based; never rotates)
- **Cached for 1 year** (`Cache-Control: public, max-age=31536000, immutable`)
- **CORS open** (`*`)
- **Supports range requests** — streaming and seek work

Caveat for the ephemeral vault: "stable for life" is bounded by the 24 h auto-delete. Fine
for a generation job that runs in minutes; **never** link an ephemeral URL from a delivered
page.

---

## Re-encoding behavior (client vaults)

A normal client key re-encodes uploads:

| Upload format | Stored as |
|---|---|
| jpg, jpeg, png, gif, webp, avif, heic, heif | `.webp` |
| mp4 | `.mp4` (passthrough) |
| mov, webm, mkv | `.mp4` (re-encoded H.264/AAC) |
| pdf, txt, others | raw, untouched |

The **`video-lab` ephemeral client skips the whole encoder**, so PNG stays PNG and JPG stays
JPG.

**Don't infer the file extension from the upload name.** Always read `cdn_url` from the
response — that's the canonical URL.

---

## Replace / delete

### Replace (keeps UUID and URL)

```
POST /files/<id>/replace      # multipart, field `file` (singular)
```

```bash
curl -X POST "$LUNA_BASE_URL/files/$ID/replace" \
  -H "X-API-Key: $LUNA_API_KEY" \
  -F "file=@new-image.png"
```

Response: `{ "cdn_url": "..." }` — same URL, new content.

⚠ **Cache busting:** the CDN serves with `max-age=31536000, immutable`. Browsers that already
cached the old version keep showing it. Append a `?v=<timestamp>` query parameter in your
`<img src>` / `<video src>` to bust.

### Delete

```
DELETE /files/<id>
```

Only the client owning the API key can delete a given file. Cross-client deletes return 403.
Ephemeral uploads need no delete — they expire on their own.

---

## What Luna **doesn't** offer

- No `GET` for listing your uploaded files. The consuming app's database is the source of
  truth — persist `cdn_url` in your records when you upload.
- No URL rotation (the UUID is permanent).
- No folders or buckets — all files live flat by UUID.

---

## Common errors

| Status | Cause |
|---|---|
| 400 | malformed request (missing field, unsupported format) |
| 401 | missing or invalid `X-API-Key` header — **including the empty-string case from exporting a variable that doesn't exist** |
| 403 | trying to modify/delete another client's file |
| 404 | file doesn't exist — or an ephemeral upload aged past 24 h |
| 413 | file >500 MB |

---

## Quick reference

```bash
# Which vault am I about to write to?
curl -sS "$LUNA_BASE_URL/me" -H "X-API-Key: $LUNA_KEY_VIDEOLAB"

# Upload an input frame for generation (ephemeral, PNG preserved)
curl -sS -X POST "$LUNA_BASE_URL/files/upload" \
  -H "X-API-Key: $LUNA_KEY_VIDEOLAB" \
  -F "files=@frame.png"

# Upload a final deliverable (client vault, permanent)
curl -sS -X POST "$LUNA_BASE_URL/files/upload" \
  -H "X-API-Key: $LUNA_API_KEY" \
  -F "files=@final.mp4"

# Delete
curl -X DELETE "$LUNA_BASE_URL/files/$UUID" \
  -H "X-API-Key: $LUNA_API_KEY"
```
