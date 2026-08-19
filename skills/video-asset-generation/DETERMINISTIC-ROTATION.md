# Rotating a flat art piece (needle, hand, dial) deterministically — and its 3 traps

The replacement for generative i2v when a piece must reach a **specific angle** or stay
inside a **bounded arc** (see [ANIMATION.md](ANIMATION.md) — the model cannot bound
angular travel). Costs $0, keeps the source art **pixel-identical**, loops exactly, and
the curve is editable in seconds. PIL + numpy only.

**Pipeline:** color-mask the piece → cut it out → fill the hole it leaves → rotate the
sprite around its pivot frame by frame → composite over the plate.

## Trap 1 — filling the hole by neighbour average leaves a dark GHOST

Filling iteratively with the mean of the known neighbours drags in the soft shadow that
surrounds the piece and leaves its silhouette legible. In **circular** art (a ring, a
dial, a watch face) the correct pixel is at the **same radius**, at another angle. And
taking the nearest valid angle is not enough: it cuts abruptly across the middle of the
filled band and breaks the angular gradient.

**What does work: angular interpolation between the two valid sides.** For each pixel at
`(r, θ)`, scan in `+θ` and in `−θ` until a valid pixel at the same radius is found, then
blend weighted by each side's angular distance. It preserves the gradient and leaves no
seam.

**And do not erase the pivot.** The axis never moves, and rotating a circle about its own
centre is a no-op: leave it on the plate (exclude a disc of radius ≈ 0.115 × the length
of the piece). Erase it and the radial fill degenerates near r≈0, leaving a black smudge
at the centre.

## Trap 2 — the color mask also catches TEXT of the same color

A mask for an orange needle also captured same-colored digits of the wordmark — false
positives the script treated as needles and set spinning. Objective discriminator by
elongation:

```python
elong = reach**2 / area      # needle ≈ 5.9-6.0 ; wordmark glyph ≈ 0.88-0.90
```

Huge separation; gate at 3.0. Finding the pivot falls out of the same analysis for free:
the axis is usually a **donut**, so its enclosed hole (flood-fill from the bbox edge —
whatever stays unreached is the hole) gives the exact centre.

## Trap 3 — measuring "does the piece leave the frame?" by marching a ray FROM the pivot

It returns 0 in every direction when the pivot pixel does not belong to the frame's
mask — and it does not, because the donut's hole is dark. The ray exits the frame at step
zero and reads as "it sticks out on all sides". Scan from the OUTSIDE inward, or fill
holes first.

Also, a frame mask built from distance-to-background-color **fails** wherever the art has
a dark gradient (that zone is confused with the background and eats the border). For a
known geometric figure, construct it instead: measure the ring's radius and derive the
rounded square by proportion (measured on iOS-icon-style art: `side = 3.263 × R_ring`,
`corner_r = 0.225 × side`, tile centre = pivot + `(0.003, −0.061) × R_ring`). Then clip
the rotated sprite against that mask: the piece stays at 100% length and never pokes
outside the icon.

## The curve: a drag launch, not a ramp

Idle with micro-jitter → `easeOutCubic` launch with **~6% overshoot** → damped bounce →
hold at redline with fine vibration → **gear-change drop** (~38% down) → second climb →
hold → `easeInOutCubic` deceleration back to idle. Make the first and last frames
coincide (measured: 0.4° apart) so the loop closes without a jump.

**And the direction matters:** a tachometer sweeps from rest at lower-left (7:30)
**clockwise, over the top** (9:00 → 12:00) to redline at upper-right (1:30). The other way
round (under the bottom and up the right) reads as a gauge going DOWN, and the bug is
invisible in a contact sheet — it is caught with the unwrapped angle series
([ANIMATION.md](ANIMATION.md#measuring-angular-travel-a-9-frame-contact-sheet-lies)).
