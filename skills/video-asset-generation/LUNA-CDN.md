# Luna CDN — file hosting for video-lab

Public URL hosting for the assets that downstream APIs (like Wan 2.6) need to fetch.

---

## API basics

**Base URL:** `https://luna.solutions45.com/api`
**Auth:** `X-API-Key` header (per-key client identification)

**The two key types:**

| Env var | Behavior | Used by |
|---|---|---|
| `LUNA_API_KEY` (...4243) | **Converts images to .webp** on upload | a client web admin panel (multi-poster) |
| `LUNA_API_KEY_VIDEOLAB_RAW` (...8b07) | **Preserves PNG / JPG** as uploaded — no webp conversion | video-lab uploads, especially for downstream models that don't accept webp |

Both point to the same Luna vault for the client; the second one is configured to skip the encoder.

**Decision rule:**
- If the next consumer of the upload accepts webp (Wan 2.6, Sora 2, Grok Imagine all do) → either key works.
- If the next consumer only accepts JPG/PNG (Kling 2.6 only accepts JPG/PNG) → use **RAW**.
- **Default in the video-lab pipeline: RAW**, for forward compatibility with any provider you might add later.

---

## Upload

```
POST /files/upload     # multipart/form-data
```

- Field `files` (can repeat — up to 20 files per request)
- Max **500 MB** per file

```bash
curl -sS -X POST https://luna.solutions45.com/api/files/upload \
  -H "X-API-Key: $LUNA_API_KEY_VIDEOLAB_RAW" \
  -F "files=@image1.png" \
  -F "files=@image2.png" \
  -F "files=@image3.png"
```

**Response (HTTP 201):** JSON array, same order as input files:

```json
[
  { "cdn_url": "https://cdn.solutions45.com/76bc307d-2adf-422e-8c9a-00efd94047c3.png" },
  { "cdn_url": "https://cdn.solutions45.com/478ca451-1490-4e92-b90b-b589600dd6a5.png" },
  { "cdn_url": "https://cdn.solutions45.com/95576ef0-08b0-4747-b5eb-b679b13d6dd5.png" }
]
```

If a single file fails, that item carries `{ "error": "...", "original_name": "..." }` and the others succeed. **Always filter by presence of `cdn_url`** before assuming success.

---

## URL behavior

The returned `cdn_url`:

- Lives at `cdn.solutions45.com` (separate from the upload endpoint)
- **No auth.** Anyone with the URL can fetch.
- **Stable for life** of the file (UUID-based; never rotates)
- **Cached for 1 year** (`Cache-Control: public, max-age=31536000, immutable`)
- **CORS open** (`*`)
- **Supports range requests** — streaming and seek work

For the video-lab pipeline, this means:
- Upload once, reuse the URL
- Submit to Wan 2.6 with the URL — kie.ai fetches it directly
- The URL stays valid as long as the file isn't deleted

---

## Re-encoding behavior (default key only)

The default `LUNA_API_KEY` re-encodes uploads:

| Upload format | Stored as |
|---|---|
| jpg, jpeg, png, gif, webp, avif, heic, heif | `.webp` |
| mp4 | `.mp4` (passthrough) |
| mov, webm, mkv | `.mp4` (re-encoded H.264/AAC) |
| pdf, txt, others | raw, untouched |

**The `LUNA_API_KEY_VIDEOLAB_RAW` key skips the image re-encoding step**, so PNG stays PNG and JPG stays JPG. Video re-encoding is unaffected — both keys produce mp4 from mov/webm/mkv input.

**Don't infer the file extension from the upload name.** Always read `cdn_url` from the response — that's the canonical URL.

---

## Replace / delete

### Replace (keeps UUID and URL)

```
POST /files/<id>/replace      # multipart, field `file` (singular)
```

```bash
curl -X POST "https://luna.solutions45.com/api/files/$ID/replace" \
  -H "X-API-Key: $LUNA_API_KEY_VIDEOLAB_RAW" \
  -F "file=@new-image.png"
```

Response: `{ "cdn_url": "..." }` — same URL, new content.

⚠ **Cache busting:** the CDN serves with `max-age=31536000, immutable`. Browsers that already cached the old version keep showing it. Append a `?v=<timestamp>` query parameter in your `<img src>` / `<video src>` to bust.

### Delete

```
DELETE /files/<id>
```

Only the client owning the API key can delete a given file. Cross-client deletes return 403.

---

## Production usage example (short-02)

Upload three PNG images (segments 4, 5, 6 to be animated):

```bash
LUNA_KEY=$(grep '^LUNA_API_KEY_VIDEOLAB_RAW=' ~/.secrets/environment.d/11-secrets.conf | cut -d= -f2-)
WS=~/video-lab/<topic>/<video-name>/<variant>

curl -sS -X POST https://luna.solutions45.com/api/files/upload \
  -H "X-API-Key: $LUNA_KEY" \
  -F "files=@$WS/images/04-reflexiva.png" \
  -F "files=@$WS/images/05-facturas.png" \
  -F "files=@$WS/images/06-protegida.png"
```

Response (real, from short-02 production):
```json
[
  {"cdn_url":"https://cdn.solutions45.com/76bc307d-2adf-422e-8c9a-00efd94047c3.png"},
  {"cdn_url":"https://cdn.solutions45.com/478ca451-1490-4e92-b90b-b589600dd6a5.png"},
  {"cdn_url":"https://cdn.solutions45.com/95576ef0-08b0-4747-b5eb-b679b13d6dd5.png"}
]
```

These URLs are then passed verbatim to Wan 2.6's `image_urls` parameter.

---

## What Luna **doesn't** offer

- No `GET` for listing your uploaded files. The client app's database is the source of truth — persist `cdn_url` in your records when you upload.
- No URL rotation (the UUID is permanent).
- No folders or buckets — all files live flat by UUID.

---

## Common errors

| Status | Cause |
|---|---|
| 400 | malformed request (missing field, unsupported format) |
| 401 | missing or invalid `X-API-Key` header |
| 403 | trying to modify/delete another client's file |
| 404 | file doesn't exist |
| 413 | file >500 MB |

---

## Quick reference

```bash
# Upload (single image, RAW key for video-lab)
curl -sS -X POST https://luna.solutions45.com/api/files/upload \
  -H "X-API-Key: $LUNA_API_KEY_VIDEOLAB_RAW" \
  -F "files=@photo.png"

# Delete
curl -X DELETE "https://luna.solutions45.com/api/files/$UUID" \
  -H "X-API-Key: $LUNA_API_KEY_VIDEOLAB_RAW"
```

For the full Luna onboarding doc (covers branded covers, advanced endpoints), see `~/pond/luna-cdn-client-onboarding.md`.
