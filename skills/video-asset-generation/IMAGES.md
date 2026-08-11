# Images — kie.ai Nano Banana 2

Static image generation for video segments. The standard model used in this project.

> **Editing an existing photo instead of generating one?** See
> [IMAGE-EDITING.md](IMAGE-EDITING.md) — image-to-image uses a different input
> field per model family, and the raw output is never used as-is.

---

## API basics

**Endpoint:** `POST https://api.kie.ai/api/v1/jobs/createTask`
**Auth:** `Authorization: Bearer ${KIE_API_KEY}` (sourced from your secrets store)

**Request body:**
```json
{
  "model": "nano-banana-2",
  "input": {
    "prompt": "<prompt text>",
    "aspect_ratio": "9:16",
    "resolution": "1K",
    "output_format": "png"
  }
}
```

**Response:** `{ "data": { "taskId": "..." } }` — task is async.

**Poll status:** `GET /api/v1/jobs/recordInfo?taskId=X`
- Response includes `data.state` (`waiting` → `success` / `fail`)
- On success, `data.resultJson` (parse as JSON) contains `resultUrls[0]` — the temporary URL of the generated image
- Download the URL within minutes (it's on `tempfile.aiquickdraw.com` and expires)

**Typical latency:** 15-60s per image. Submit all in parallel, then poll.

**Pricing:** ~$0.02 per image at 1K resolution (verified 2026-05).

**Output dims:** ~768×1376 for 9:16 at 1K (not exactly 1080×1920 — the 1K refers to long-edge target, not exact output). For composition at 1080×1920 use `object-fit: cover`.

---

## Two validated styles

### Style 1 — Papercut

**When to use:** abstract concepts, warm emotional pieces, brand storytelling, when literal realism would feel cheesy or stocky. The pilot short used this style.

**Prompt prefix (apply to every image to maintain consistency):**

```
Papercut paper-craft style illustration, layered cut paper textures, contemporary universal setting, neutral balanced lighting, no sepia tone or warm grading, no traditional ethnic clothing, no cacti or agave or palm trees, paleta amber + ivory + deep navy blue used as papercut color system not as warm filter, vertical 9:16 composition,
```

**Why each part of the prefix:**
- "Layered cut paper textures" — the defining feature of the style
- "Contemporary universal setting" — counters Hollywood-Mexico clichés
- "Neutral balanced lighting, no sepia tone" — counters the warm-orange filter that the model defaults to with paper-craft
- "Amber + ivory + deep navy blue used as a papercut color system not as warm filter" — explicit instruction that the warm tones are *layers of paper*, not a sepia overlay

**Per-image prompt structure:**

```
<PREFIX> + <subject description with simple action>
```

Example (from pilot):
> *"...vertical 9:16 composition, a hand holding a brand new smartphone, isolated background"*

### Style 2 — Cinematic realism

**When to use:** relatable scenarios where the viewer should picture themselves in the scene; documentary-feeling pieces. Short-02 used this style.

**Prompt prefix:**

```
Cinematic realism, documentary editorial photography style, contemporary universal setting, natural neutral lighting (no warm golden grading, no sepia), no traditional ethnic clothing, vertical 9:16 composition,
```

**Why each part:**
- "Documentary editorial photography style" — the visual reference; counters generic stock-photo-ness
- "Natural neutral lighting (no warm golden grading, no sepia)" — explicit. Without this clause, the model returns warm-graded everything, dragging back to sepia tone.

**Per-image prompt structure:**

```
<PREFIX> + <protagonist description if person> + <action / setting>
```

If you need character consistency across multiple images, use a **stable protagonist clause**:

```
an adult man around 35 years old with short dark hair and a light trimmed beard, wearing simple casual clothing (plain sweater or button-up shirt)
```

Repeat this clause verbatim in every image where the protagonist appears. Variations break consistency.

---

## Anti-patterns to avoid

These are documented mistakes from production:

### ❌ "Mexican family / Mexican person / Mexican home"

The word "Mexican" in a prompt triggers Hollywood/Coco stereotype output from the model:
- huipil bordado on women
- mustaches and serapes on men
- adobe houses with palms and cacti
- sunset sepia tone

The visual is "how Hollywood pictures Mexico" — caricaturesque and not what real Mexican audiences identify with. Even if the audience is Mexican, the scene should look like contemporary life, not picture-book.

**Fix:** describe the scene without nationality. If specific traits matter ("dark hair", "olive skin", "casual clothing"), describe them directly. Skip ethnicity/nationality words entirely.

### ❌ "Warm sunlight / golden hour / warm directional lighting"

Default lighting in cinematic realism prompts. Sounds nice, but the model interprets it as "everything has a sepia overlay" — the entire image gets a warm color grading filter that's the visual equivalent of a 1990s travel magazine cover.

**Fix:** "Natural neutral lighting" / "soft directional daylight" / "documentary editorial style" — specifically push the model toward editorial-photography mood, where light is just light, not a coloring agent.

**Sweet food pulls hardest.** Anything with honey, syrup, caramel or batter drags the whole frame to orange even when the prefix never asks for warmth — the subject's own colour is read as a grading cue. On desserts state all three clauses explicitly: `no warm golden grading, no sepia tone, no orange filter`. With them the frame comes back neutral on the first try; without them an amber filter over the entire image is the default.

### ❌ Stock-photo settings

Without explicit "documentary editorial" or "contemporary universal", outputs default to overly clean stock-photo aesthetics. Real lives have texture — kitchens with magnets on the fridge, offices with personal items on desks, bedrooms with rumpled blankets.

**Fix:** explicitly specify "contemporary universal setting" and add 1-2 specific details (a calendar with crossings, plants on a shelf, a child's backpack on a chair) — the small touches read as real life, not curated stage set.

### ❌ Forgetting character consistency

When 3-5 images share a protagonist, the model treats each prompt independently and gives you 3-5 different people. The viewer perceives this as broken continuity.

**Fix:** copy a literal stable description of the protagonist into every prompt that includes them. The prompt prefix doesn't enforce consistency — only the description does.

---

## If the copy states a NUMBER, the composition has to let you count it

A product shot for an item sold as "2 pancakes with butter and honey" was prompted with
the count stated **twice**, once positively and once as a negation:

```
...exactly two round golden-brown pancakes stacked one on top of the other on a plain
white ceramic plate... exactly two pancakes only, no third pancake...
```

It returned **three**. And not visibly so: at thumbnail size the pile reads fine, and you
have to zoom into the edge of the stack and count golden rims to catch it. A photo showing
three where the price tag charges for two is false advertising, not a styling detail.

**Stacking hides the count.** In a pile each unit contributes only a few pixels of visible
rim, so nothing in the image strongly separates "two layers" from "three" — not for the
model, and not for the viewer either. Repeating the number in the prompt does not fix it:
the failure is in the **composition**, not in the instruction.

**What worked first try** — pieces side by side, slightly overlapping, from a high
three-quarter angle so each unit shows its whole face:

```
a pair of two round golden-brown pancakes lying flat side by side and slightly overlapping
on a plain white ceramic plate, seen from a high three-quarter angle so both pancake faces
are fully visible and countable, ... only two pancakes in the entire image
```

Every unit is a separate, complete disc — countable at a glance. A second variant (a stack
shot at plate level so the silhouette shows exactly two layers) also came back with two, but
it needs a second look to confirm; it lost on legibility.

### The rule

- **If the product copy carries a number, never ask for a stack.** Ask for the pieces
  separated or slightly overlapping. Applies to anything stackable — pancakes, tacos,
  wings, donuts, slices.
- **Count the units in the delivered image before it ships**, zooming into the edge if
  anything is piled. A count that does not match the copy is a REGEN criterion.
- Regenerating costs ~$0.02. Publishing a photo that promises more than the product
  delivers costs the client's trust.

---

## Curation: when to regenerate

After downloading images, do a visual scan. Regenerate any that:

- Are **dramatically smaller** than peers (e.g., 100KB while others are 1.5MB) — quality variance
- Show **literal text on the wrong year/word** (e.g., calendar showing 2024 when the campaign is 2026)
- Show a **different protagonist** than the consistency clause specifies
- Default into stereotype tropes despite the anti-pattern prefix
- Have visible **artifacts** (extra fingers, distorted faces, weird text)
- **Show a different number of units than the prompt or the product copy declares** — count them, zooming into the edge of any stack

Regeneration cost: ~$0.02 per image. Cheap insurance against shipping bad output.

---

## Background removal eats transparent subjects — if you know the silhouette, build the alpha

**Symptom:** assets cut out of a single frame come back carrying a near-transparent square background that covers the others. The subject is supposed to be the shape alone, but in composition each asset occludes whatever is behind it.

Measured on six glass spheres (each with an object inside) cut from one frame and passed through `hyperframes remove-background` — this applies to any subject-matting model:

| Assets | Result |
|---|---|
| 3 of 6 | correct |
| one | **0.1% opaque pixels** — erased everything |
| one | **2.3% opaque** — left a ghost |
| one | kept ONLY the inner object and threw away the sphere |

The pattern: **the model reads glass as background.** The smaller the encapsulated object is relative to the sphere, the more the whole sphere looks like "background" to it. And because it returns a PNG with a *valid* alpha channel, everything downstream accepts it — you get near-transparent squares that occlude their neighbours.

### The fix: don't ask a model for an alpha you can derive

If the subject's silhouette is a **known shape**, construct it. For a glass sphere the reasoning closes on itself: the glass rim **is** the circle's edge, so the sphere's bounding box is the crop's bounding box. The alpha is a filled circle at the inscribed radius, with ~1px feather so it isn't aliased:

```python
from PIL import Image, ImageDraw, ImageFilter
n = img.size[0]                       # square crop
mask = Image.new("L", (n*4, n*4), 0)  # 4× supersample for antialiasing
ImageDraw.Draw(mask).ellipse([0, 0, n*4-1, n*4-1], fill=255)
mask = mask.resize((n, n), Image.LANCZOS).filter(ImageFilter.GaussianBlur(1))
img.putalpha(mask)
```

Milliseconds, deterministic, and structurally incapable of eating the subject.

### The referee (cheap and decisive)

**Percentage of pixels with alpha > 0, per asset.** Every asset in the same family must yield **the same number**. In the run above all six landed at **64.8%** (a full inscribed circle would be π/4 = 78.5%; 64.8% because the sphere doesn't fill the frame). An asset that falls outside the family is an asset the matting destroyed — visible as a number, without opening a single image.

### Related trap: `PIL.Image.crop()` pads with BLACK

Anything outside the source image's bounds comes back **black**, not transparent. If an object's crop runs off the edge and you paste it as-is, every asset carries a black frame — and with a geometric alpha on top, that black is invisible in the thumbnail but reappears when composited over a light background. Clamp the crop to the real bounds and paste at the corresponding offset.

### The rule

Before sending an asset to a background remover, ask: **do I know the silhouette?** If yes (circle, rectangle, frame, a tile cut from a regular grid), build the alpha. Matting models are for silhouettes you can't describe — a person, an irregular product, hair.

---

## Real prompts from production

### Papercut (pilot — short-01)

```
Papercut paper-craft style illustration, layered cut paper textures, warm amber and ivory palette, deep navy blue accents, soft directional lighting, vertical 9:16 composition, a hand holding a brand new smartphone, isolated background
```

(Pilot was generated before we documented the "no warm sunlight" rule. Future papercut prompts should add `neutral balanced lighting, no sepia tone or warm grading` to that prefix.)

### Cinematic realism (short-02)

Protagonist clause:
```
an adult man around 35 years old with short dark hair and a light trimmed beard, wearing simple casual clothing (plain sweater or button-up shirt)
```

Eight image prompts (paraphrased structure):

1. *"<PREFIX> <protagonist> waking up alone in bed, soft morning daylight through window blinds, hand reaching toward phone alarm on nightstand, contemporary bedroom interior, gentle expression"*
2. *"<PREFIX> <protagonist> focused at a modern office desk working on a laptop, hands on keyboard, dual monitors with spreadsheets visible, contemporary office setting, neutral daylight"*
3. *"<PREFIX> a family morning routine in a contemporary kitchen, <protagonist> as the father with his wife and child, breakfast on table, child holding backpack ready for school, the man reaching for car keys, warm yet neutral interior light"*
4. *"<PREFIX> <protagonist> sitting alone on the edge of a bed in a bedroom, looking out the window with a subdued thoughtful contemplative expression, a wooden crutch leaning against the wall beside him, natural daylight, quiet introspective atmosphere"*
5. *"<PREFIX> a contemporary home desk close-up, stack of paper bills and invoices accumulating one on top of another, a wall calendar visible showing 'OCTOBER 2026' with days crossed out hanging behind, an adult mans hand holding a pen hesitating over a bill, subtle worried gesture, soft daylight"*
6. *"<PREFIX> <protagonist> seated at home in a comfortable living room with his family beside him (his wife and child), calm peaceful expression with a slight smile, his wife's hand reassuringly on his shoulder, soft natural daylight from a window in background, sense of safety and relief, contemporary interior"*
7. *"<PREFIX> an extreme close-up of a wall calendar page showing the month of December 2026, the date 31 highlighted with a red mark, dramatic side-lighting creating urgency, contemporary clean wall behind"*
8. *"<PREFIX> a minimal contemporary background with neutral cream and deep blue tones, central composition with empty space designed for a logo overlay, soft directional editorial lighting, clean professional editorial photography composition, no people"*

Refer to `~/video-lab/<topic>/<video-name>/<variant>/images/` for the actual outputs.

---

## Quick reference

```bash
# Submit
curl -sS https://api.kie.ai/api/v1/jobs/createTask \
  -H "Authorization: Bearer $KIE_API_KEY" \
  -H "Content-Type: application/json" \
  -d "$(jq -nc --arg p "$PROMPT" '{model:"nano-banana-2",input:{prompt:$p,aspect_ratio:"9:16",resolution:"1K",output_format:"png"}}')"

# Poll
curl -sS "https://api.kie.ai/api/v1/jobs/recordInfo?taskId=$TID" \
  -H "Authorization: Bearer $KIE_API_KEY"

# Extract URL from successful response
URL=$(jq -r '.data.resultJson | fromjson | .resultUrls[0]' response.json)

# Download
curl -sS -L "$URL" -o image.png
```

### Poll to a FILE — don't pipe the response through `echo`

The response carries the original request in `param` as **nested, escaped JSON** (`\\\"`).
A polling loop written as

```bash
R=$(curl -sS ".../recordInfo?taskId=$TID" -H "Authorization: Bearer $KIE_API_KEY")
echo "$R" | jq -r '.data.state'          # ← breaks
```

loses backslashes in `echo` (zsh's builtin interprets them), so `jq` fails with
`Invalid numeric literal ...` on **every** iteration. The loop then looks exactly like
"the API never finishes" — while `state` was already `success` on the first pass.

Write the body to a file (`curl -o response.json` and `jq ... response.json`) as the Quick
reference above does, or use `printf '%s' "$R" | jq`. The file is not incidental style;
it is what keeps the polling loop from lying to you. Don't "simplify" it into a pipe.
