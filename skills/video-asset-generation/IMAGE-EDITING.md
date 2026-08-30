# Image editing (image-to-image) — surgical retouching of a real photo

Distinct from [IMAGES.md](IMAGES.md), which covers **text-to-image** generation
from scratch. This file covers the other case: changing **ONE region of a real
photograph** and leaving everything else untouched — retouching product photos,
swapping what a mirror/screen/window shows, cleaning a defect.

---

## Models and request shape (verified against the API, not from memory)

Same endpoint as generation — `POST /api/v1/jobs/createTask` — but **each family
names the input-image field differently**. Getting it wrong returns a 422 that
reads like "the model doesn't exist":

| model | input field | notes |
|---|---|---|
| `nano-banana-pro` | `image_input` (array, up to 8) | + `aspect_ratio` (use `auto`), `resolution` 1K/2K/4K, `output_format` |
| `gpt-image-2-image-to-image` | `input_urls` (array, up to 16) | `aspect_ratio`, `resolution` |
| `grok-imagine/image-to-image` | `image_urls` (array, **max 1**) | accepts webp, max 10 MB |

`seedream-4-5-image-to-image` does NOT exist under that name: it answers *"The
model name you specified is not supported"* — which is the only error that
actually proves non-existence (a 422 does not).

- **Credits:** `GET /api/v1/chat/credit` → `{"code":200,"data":<number>}`.
- The result URL lives on `tempfile.aiquickdraw.com` and **403s against
  `urllib`** — download with `curl -L -A "Mozilla/5.0"`.

---

## The finding: only one of the three respects the photo

Same prompt, same photo, same request ("change only the mirror"):

| model | pixels changed OUTSIDE the requested region |
|---|---|
| **nano-banana-pro** | **0.001 %** |
| gpt-image-2 | 0.72 % |
| grok-imagine | 1.32 % |

The last two **redraw the whole object and shift it a few pixels**: the
difference map lights up the entire perimeter (legs, mouldings, drawers,
handles). In a thumbnail it looks identical. For retouching a real photo,
nano-banana-pro is the only one of the three that serves.

---

## The rule that avoids the problem entirely: DON'T TRUST — COMPOSITE

Even with a good model, **you never use the full output.** Take **only the edited
region** and paste it over the original.

Measured on a photo with a busy background: even with nano-banana-pro and a
prompt that explicitly said "do not touch the background", a background blind
changed **17.8 %** of its pixels and a baseboard **18.7 %** — both looked intact
to the eye. With compositing that stops mattering: whatever the model invents
outside the region is discarded before anything touches disk.

**The verification that closes the matter, and it is binary: outside the mask,
the difference against the original must be exactly 0 pixels.**

---

## Trap when building the compositing mask

Deriving the mask from the **difference map** looks obvious and **leaves islands
of the old content**: where old and new have similar brightness the difference
never crosses the threshold, and that patch keeps the original pixel. Real
result: a "cleaned" mirror with a piece of the old object floating in the middle.

**Fix: use the rectangular envelope of each blob, not the blob.** Same principle
already documented in [IMAGES.md](IMAGES.md) for the spheres' alpha — *if you
know the silhouette, construct it; matting models are for silhouettes you cannot
describe*. A mirror, a picture frame, a screen, a window: all rectangles.

And if the input is a **PNG with transparency** (a cut-out with the background
removed), the model returns the frame filled in: **re-apply the original's alpha
channel** after compositing. Since nothing moved, the old alpha is still valid —
verified at 0 differing alpha pixels.

---

## The shape of the hole, and who respects it

The finding above (nano-banana-pro touches the photo least) holds, but is
incomplete. The axis that decides the result when the hole to fill is **not a
rectangle**:

| model | hole shape it produces | how much of the rest it touches |
|---|---|---|
| nano-banana-pro | **always rectangular** | almost nothing (0.001 %) |
| gpt-image-2 | **follows the real contour** | redraws the object (23 %) |

nano-banana-pro draws a rectangle even when the prompt says *"the top edge is cut
to follow the carved scalloped moulding, in a wavy line — do not draw a straight
horizontal top edge"*. It ignores it however many times you ask. The result reads
as a grey sticker pasted on top, leaving a strip of the original material between
the ornament and the glass.

**Neither one works alone.** The combination that does:

1. the **CONTENT** (the clean gradient) comes from gpt-image-2, which respects the contour
2. the **OBJECT** is the original photo, not one pixel touched
3. they are joined by a mask that is the **real silhouette, measured from the original photo**

### Getting the silhouette from the photo, not from the model

Inside the hole's bounding rectangle, the frame is saturated red wood and the
content is not. Separating by "redness" (`R − (G+B)/2`) gives the exact contour,
ornament included:

- measured percentiles: frame 55–75, glass −50…+10 → **threshold 10**, not 28
- at 28 it eats the ornament and the straight edge comes back
- then: opening to remove specks, largest connected component, hole filling
- **dilate** 2–8 px, never erode: eroding leaves a rim of the old content stuck
  to the frame, and it reads as tatters

Binary verification: outside the mask, the difference against the original must
be **0 pixels**. Anything else means the mask is wrong.

### Read the bounding rectangle off the photo with a grid

Reading coordinates "by eye" off a contact sheet fails — short on the left in one
image, short on the bottom in another, and each error shows up as a *different*
looking defect (reflection "squashed to one side", tatters at the edge). Draw a
labelled tenths grid over the image and read from it: one pass, done.

---

## Extending the canvas (outpaint): deterministic beats generative for a background

Handing a square render to an image-to-image model as a taller canvas — original centred,
top/bottom bands filled with a blurred hint — to extend it to 9:16 tends to return a
**different composition**: the subject re-posed, a different surface, different background
elements. As a standalone image it can look good; as an extension it's useless, because the
piece's identity lives in the original framing. It's the same failure "composite only the
edited region back over the original" already covers above — it just costs more here, because
once the whole geometry has moved there is no clean region left to composite back.

**For a background extension — gradient, sky, a table surface, out-of-focus bokeh — do it
deterministically instead of asking a model:**

- **Top band:** mirror the first N rows and blur. Mirroring is continuous at the seam by
  construction, so there's no edge to hide.
- **Bottom band:** stretch the last ~40 rows downward and blur lightly — on a receding surface
  this reads as perspective.
- **Blend both** into the sharp area over a ramp of ~100px so the transition reads as depth of
  field, not a pasted band.

Cost: zero, instant, and the original is untouched by definition.

**When a model IS the right tool:** when the extension needs new *content* — more of a
patterned floor, more objects, architecture — not just more of a smooth field. Then use it, and
composite only the new bands back over the untouched original (same compositing rule as above).

**The tell:** if the prompt is mostly a list of things that must not change, it's the wrong
tool — a prompt that's 80% prohibitions is a compositing job wearing a prompt's clothes.

## Method lesson: for a finite batch, the overlay plus your eyes wins

Two automatic arbiters were built to judge "did it touch what it shouldn't?" and
both failed:

- the first required declaring the region by hand, so it ended up measuring the
  operator's aim at reading coordinates — it failed eight edits that were fine
- the second, by connected components, **approved exactly the case that had to be caught**

What worked: **looking at the difference map painted over the original photo** —
four per sheet, two sheets for fourteen images. For a finite batch that happens
once, the overlay + eyes beats two rounds of hardening a detector.
