# Measuring Geometry Inside a Composition

When a value has to be *derived* from the rendered text or artwork — an exact ink width, a
letter-spacing that hits a target, an offset that lines up with baked footage — measuring it at
page load is the reflex, and inside a composition it is wrong.

---

## A layout API always answers

`getBoundingClientRect()`, `offsetWidth`, `getComputedStyle().width` never return an error for an
element that is hidden, collapsed, or not yet in the flow. They return zeros or the container's
values — **numbers that look exactly like measurements**.

In a HyperFrames composition, an element that lives inside a clip with a future `data-start` is in
the DOM but **not active** when a load-time script runs. The API answers; the answer describes
nothing:

```js
const el = document.getElementById("fixtext");
el.style.letterSpacing = "0px";
const natural = el.getBoundingClientRect().width;   // ← a lie: the clip is not active yet
el.style.letterSpacing = (TARGET - natural) / (n - 1) + "px";
```

Measured: a line that had to be 537.7 px of ink came out at **177 px** — a factor of 3.

**No gate catches it.** Lint 0 errors, runtime 0, layout 0 findings, motion 0, contrast all
passing. From the checker's point of view the text is where it was told to be, at the size it was
told to be, with a perfectly valid CSS letter-spacing. The defect only exists relative to an
intent that lives *outside* the file.

Same family as measuring inside a zero-height box, or measuring before the web fonts load — the
fallback face has different metrics and its number also looks fine.

---

## The two ways out

### 1. Bake the measured value (default)

Set the property to 0, take a `hyperframes snapshot`, measure the real ink width **in the PNG**,
compute, and write the number into the CSS with a comment saying where it came from.
Deterministic, immune to load order, and the number stays auditable.

### 2. Measure on a probe outside the composition

Clone the node with the same styles into `document.body`, measure it, compute, apply to the real
element, delete the probe. Use it when the value must follow variable text — but it is still
racing font loading, so it needs `document.fonts.ready`, and that is **async**, which is exactly
what the timeline does not admit.

> **Rule:** fixed text → option 1, always. Option 2 only when the content arrives through
> variables.

---

## Patching over baked material: the axis belongs to the object, not the canvas

Adding a line on top of footage that is already rendered, the reference is the geometry of **what
is already there**, measured in the material's own pixels.

`left: 0; right: 0; text-align: center` centres on the *canvas* axis (540 of 1080). That is the
automatic reflex and it is wrong the moment the thing you are aligning to is not centred in the
frame — measured: the object's axis was at 758, because the mark sits right of centre with a
symbol occupying the left.

It is checked by **looking at the frame**: this error is obvious in an image and appears in no
number.
