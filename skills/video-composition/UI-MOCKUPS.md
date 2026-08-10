# Imitating a Real UI Inside a Composition

Screen mockups (a chat, a feed, a terminal) are a common insert in a short. Copying the visual
design is the easy part; the part that decides whether it reads as real is the **fill direction**.

---

## Every UI has a fill direction, and it is not always top-down

- **Chats** grow **upward from the bottom** — the newest message rests on the compose bar.
- **Feeds** grow downward from the top.
- **Terminals** grow downward with the prompt pinned to the last line.

Copying the look but not the fill direction produces something that looks *almost* right and
reads as broken.

Measured on a full-screen chat mockup (1080×1920): header on top, four bubbles stacked downward
with `flex-direction: column`. Lint clean, layout clean, render successful — and the four bubbles
occupied ~560 px of a 1702 px body, so **67% of the frame was flat empty background below the
conversation**. It does not read as "a short chat", it reads as "the app failed to load". No
automatic gate flags it: technically everything is where it was told to be.

> **Smell to check by eye:** if a full-screen panel has more than half the frame in one flat
> colour, look at it before calling it good.

---

## The fix — which is also what the real app does

1. **Anchor the stack to the bottom.** The last message rests just above the input bar. The empty
   space ends up at the *top*, under the header, which is exactly where a real chat has it.
2. **Draw the input bar.** A chat without a compose field does not read as a chat. It costs one
   row and it anchors the composition.
3. **Dim older messages at the top**, some of them clipped by the top edge. That is what opening a
   conversation with history looks like, and it fills the upper void without inventing content.
4. **Animate the push.** Each new message pushes the stack up. It feels right, and it solves the
   fact that with fixed final positions the first messages would float mid-screen with a void
   underneath.

---

## Implementing it without a reflow mid-timeline

Absolute positions computed **once at load**, not animated flexbox layout:

```js
// measure each slot, stack them top-down, then shift everything so the last one
// lands on the resting line above the input bar
const total = /* accumulated height */;
const shift = bodyH - PAD_BOTTOM - total;
slots.forEach((ids, i) => ids.forEach(id => el(id).style.top = tops[i] + shift + "px"));
// how far to push the stack DOWN so slot i sits at rest
const offset = bottoms.map(b => total - b);
```

Then animate `y` on the stack container: `offset[k]` at each beat.

- The **typing indicator and the reply that replaces it share a slot** — so the reply appears
  exactly where the dots were.
- Remove the indicator with `autoAlpha`, **never** by collapsing `height` / `padding`: those are
  outside the animatable set and they force a reflow.
- If the stack ends up taller than the body, `shift` goes negative and the old messages spill past
  the top edge. That is correct, and it has to be declared with `data-layout-allow-overflow` or
  the layout check reports it as an overflow.

⚠️ Those slot heights are measured at load — if the mockup lives in a clip that is not active yet,
the measurement is garbage and every position is wrong. See [MEASURING.md](MEASURING.md).
