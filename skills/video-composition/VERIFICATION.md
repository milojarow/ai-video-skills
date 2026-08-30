# Verifying a Render Before Delivery

**`ffprobe` is not verification.** Container metadata (codecs, duration, size) says nothing about the actual pixels.

---

## The expensive lesson

An MP4 once shipped "verified" with ffprobe — codecs, duration, size all correct — and it had 16 frozen seconds. The container metadata looked perfect; it was caught downstream, not by our own checks. Always verify the actual content, not the wrapper.

---

## Minimum protocol before delivering any render

```bash
# 1. Frozen ranges (target, in seconds):
ffmpeg -i out.mp4 -vf "freezedetect=n=-60dB:d=0.4" -map 0:v -f null - 2>&1 | grep freeze_
# Any freeze > ~1s that isn't an intentional storyboard hold = broken render.

# 2. Contact sheet — and LOOK at it (Read the PNG), don't just generate it:
ffmpeg -y -i out.mp4 -vf "fps=1/2.5,scale=216:384,tile=7x2:padding=4:color=gray" -frames:v 1 contact.png
# Each column ≈ t = (n+0.5)*2.5s. Confirm each timestamp shows ITS expected content.

# 3. Spot frames at VO-synced moments (slams, counters, CTA):
ffmpeg -y -ss 2.98 -i out.mp4 -frames:v 1 frame-check.png
# The visual must match the transcript word at that timestamp.
```

---

## Snapshot before paying for a full render

`npx hyperframes snapshot . --at 2,7,13,20,27` uses the **same seek engine as the render** — it catches this class of bug in seconds without a 2-5 min render. Before each render: snapshots at 3-5 spread timestamps; if two consecutive ones are identical where there should be animation, there's a bug. (`hyperframes inspect` also seeks like the renderer.)

---

## Cheap post-render gates worth running on every mix

Next to the checks above, three cheap post-render controls catch what a visual review of the
timeline does not:

- **First frame is not black.**
- **Aspect ratio is exactly 0.5625** (1080×1920).
- **Voice entry is a jump, not a level** — see [COMPOSITION.md](COMPOSITION.md)'s "Voice entry
  timing" for the detection method (the largest step between consecutive volume windows, not a
  fixed dB threshold against a baseline — a threshold fires on the music ramping in instead of
  the voice).

---

## Gotcha: false negatives in chained checks

When chaining verifications with `&&` / `||`, a **missing file** makes ffmpeg fail to stderr and a trailing `|| echo "NO FREEZES"` fires a false negative. Always confirm the file exists first (`ls`) as an explicit, separate step.
