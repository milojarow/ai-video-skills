# Third-party / borrowed assets — nothing baked-in is neutral

Reusing an asset someone else made — footage from a provider or another account, a font
pulled from a build cache — carries decisions that person already made for a different
brand. Skip verifying them and you ship those decisions by default, which looks exactly
like having chosen them on purpose.

## Borrowed footage comes with narration baked in

If a third-party video clip isn't explicitly stripped of its own audio, that audio
publishes with it — and the client's ad ends up speaking in a voice, accent, or gender
that someone else picked for a different brand.

**Why it's easy to miss:** the instinct is to transcribe the clip to read what it says.
A transcript comes back as plain text — identical whether the voice is regional Spanish
or Mexican, male or female, a professional VO or cheap TTS. The script gets reviewed;
the voice never does. The defect only exists when you actually listen.

**Rule:** everything baked into a borrowed asset — voice, music, on-screen text, color
grade, a logo — was somebody else's decision for their brand. Each one gets accepted or
replaced *on purpose*. Accepting by default looks identical to having chosen it, so it
never shows up on a checklist.

**Verification — check the rendered output, not the source clip:** transcribe the audio
of the FINAL exported file and confirm the text is the intended VO. This only
discriminates if the two scripts diverge from the first sentence; if they're similar,
this check is useless and you need a different marker (a word that only exists in the
new script).

**Instrument trap measured the same day:** `ffmpeg -v error … -af volumedetect` prints
**nothing** on success. An empty result reads exactly like silence. Always run it with
`-hide_banner -nostats`, and validate the check itself against a clip you know has audio
(a positive control) before trusting a "silent" verdict on the real one.

## Removing an unwanted region from borrowed footage: measure the boundary per frame

Zeroing the alpha over a fixed rectangle — a logo box, a headline band — to strip what a
foreign studio baked into a borrowed clip can look correct at the first and last frame and
still be wrong throughout the middle: if the subject moves toward the killed band over the
clip, a fixed rectangle clips it wherever it grows into that region. The signature is a
perfectly straight horizontal or vertical edge cutting an organic subject — nature doesn't
produce straight lines; a rectangle does.

**The rule: measure the keep/kill boundary per frame, never hardcode it.**

Cheap and robust: label the connected components of the subject mask, take the one that
reaches the anchor (the part that's always present — a stem, a base, the bottom edge), and read
its topmost row per frame. Everything above that row is overlay; everything below is subject.
Label at quarter resolution to keep it fast, then gate the full-resolution alpha with the
upscaled result — the boundary can travel tens of pixels across a clip that a fixed rectangle
cannot follow.

**Two controls to run before trusting the boundary:**
- **No long component above the cut** — no component reaching the anchor may start above the
  measured boundary, or the subject is still being clipped.
- **Nothing of the overlay below it** — count pixels differing from the replacement background
  inside the overlay's own band, across every frame. Should be exactly zero.

## Font subset from a build cache: glyph count ≠ coverage

Pulling a typeface out of `next/font`'s cache (`.next/static/media/*.woff2`) can hand you
several subsets of the same family. The subset with the MOST glyphs is not necessarily
the one that covers your text — the largest file measured was `latin-ext` and it did not
contain a single accented vowel.

Pick the subset by the actual characters your composition needs to render, not by file
size or glyph count. Keep the wrong subset around as a negative control for whatever
checks your text rendering — it reproduces the missing-accent failure on demand.
