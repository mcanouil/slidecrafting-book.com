---
name: "animejs-revealjs"
description: "Lessons and patterns for driving anime.js animations from Quarto reveal.js fragments and slide events. Use this skill whenever the user wants a JS-driven animation library (anime.js) synced to reveal.js fragment/slide navigation — cursor-style demos, loops that must survive layout shifts, typewriter effects, or any timeline that needs to stop/restart cleanly on back-navigation. Covers coordinate-space conversion, fragment lifecycle gotchas, the self-cancelling loop pattern, and cleanup on stop."
---

# anime.js + Reveal.js

References:
- https://slidecrafting-book.com/animejs
- https://animejs.com/documentation/
- https://revealjs.com/fragments/

Notes distilled from building a collaborative-editing cursor animation for a Quarto reveal.js talk. This skill is about wiring an external animation library (anime.js) to reveal.js's fragment and slide lifecycle — not about CSS-only fragments (see the `quarto-revealjs-fragment` skill for those).

## Loading anime.js

Load it directly in an `include-after-body` file:

```html
<script src="https://cdn.jsdelivr.net/npm/animejs@3.2.2/lib/anime.min.js"></script>
```

## Coordinate spaces

Reveal.js scales the entire slide with a CSS transform to fit the viewport. Consequences:

- Screen pixels ≠ slide coordinate pixels.
- `getBoundingClientRect()` returns screen-space values.
- Convert cursor/mouse positions: `slideCoord = screenPixels * (1280 / section.getBoundingClientRect().width)`.
- `translateX`/`translateY` on child elements are already in the slide's internal coordinate space (e.g. 1280×720) — that's what you want to animate to.

## Measurements go stale

`getBoundingClientRect()` is a snapshot. If layout changes afterward — a fragment fires, a CSS transition runs, an element resizes — cached positions are wrong. Remeasure after the change settles, not before.

For fragment-triggered layout shifts, wait for the CSS transition to finish before remeasuring: `setTimeout(runAnim, 700)` after a 0.6s CSS transition, for example.

## Divide ownership between CSS transitions and anime.js

Two animation systems on the same property is undefined behavior.

- Let CSS `transition` own layout properties (`padding-left`, `width`) used for structural shifts.
- Let anime.js own `transform` and `opacity` on the animated elements.
- Never put a CSS transition on a property anime.js will also animate.

## Fragment events fire immediately

`fragmentshown` fires at the moment of trigger, not after reveal.js's own fragment animation completes. If you need to measure something that changes when the fragment appears, add a small delay before measuring.

`fragmenthidden` is the mirror event for back-navigation. Easy to forget — if you only handle `shown`, state gets out of sync when the user steps backward.

## Stopping vs. pausing

anime.js timelines use absolute time offsets. If layout shifts while a timeline is mid-play, the remaining keyframes target positions that no longer exist. There's no clean "pause here, resume after layout settles" — stop completely and restart fresh with new measurements.

## The self-cancelling loop pattern

Design animation loops so all state (translateY, width, text content) returns to its original value after one full cycle:

- No reset animation needed — the loop restarts seamlessly.
- You can stop and restart at any point without serializing "what phase was it in."
- Fragment transitions become trivial: stop, shift, restart.

Track state variables (`ty`, `imgCSS`, header text) as the timeline is built. Each move snapshots values before mutating state, so targets are computed correctly even though the timeline plays later.

## Build the timeline upfront, snapshot values early

anime.js timelines are built synchronously then played asynchronously. If target values are computed lazily (inside callbacks), they reflect state at play time, not build time — wrong if earlier moves already ran.

Snapshot any values needed before calling state-mutating functions:

```js
const snapHandleY = hy(blockIdx);  // snapshot before ty changes
tl.add({ targets: el, translateY: snapHandleY + dragDY, ... });
ty[blockIdx] += dragDY;  // mutate after
```

## Animating plain objects for text effects

anime.js can animate properties on plain JS objects, not just DOM elements — useful for typewriter effects:

```js
const state = { n: 0 };
tl.add({
  targets: state,
  n: text.length,
  duration: text.length * 65,
  easing: 'linear',
  update: () => { el.textContent = text.slice(0, Math.round(state.n)) + '|'; },
});
```

Pass `fromText`/`toText` explicitly when building the timeline — don't read from the DOM inside the timeline build loop, since the DOM hasn't changed yet (the animation hasn't played).

## Splitting entry animation from loop start

If fragments should control distinct phases (e.g. "cursors appear" vs. "loop starts"), split into two functions:

- `appearAnim()` — creates elements, runs the entry animation, sets `currentTl`.
- `runAnim()` — pauses `currentTl`, snaps elements to idle, starts the loop immediately.

`runAnim` must pause any in-progress `currentTl` before starting. If `appearAnim`'s entry timeline is still playing when `runAnim` fires (user advanced the fragment before it finished), the two animations fight over positions, corrupting any layout measurements a later step relies on.

```js
function runAnim() {
  if (running) return;
  running = true;
  if (currentTl) { currentTl.pause(); currentTl = null; } // kill appearAnim if still running
  // snap elements to idle, then doLoop()
}
```

## Restoring state after a layout shift

After a layout shift (fade out, then call `runAnim`), the loop restarts without an entry animation. Two things must happen at the top of `runAnim`:

1. **Snap opacity to 1** — if the shift faded elements to 0 and the loop just starts, they stay invisible.
2. **Snap positions to idle** — the loop was interrupted mid-move; elements are wherever the timeline left them. Snapping to a known idle position makes the loop restart clean, since later moves may compute deltas from that idle position.

```js
anime.set(alice, { opacity: 1, translateX: aliceSnap.x, translateY: aliceSnap.y });
```

## Cleaning up mid-animation state on stop

An effect that mutates DOM state incrementally (e.g. a typewriter appending `|` to `textContent`) will leave that state behind if the animation is paused mid-way (fragment back-navigation, layout shift, slide change) — the `complete` callback that would normally clean up never fires.

Fix: a dedicated cleanup helper, called from every stop path:

```js
function cleanHeader() {
  const h = document.getElementById('cblock-header');
  if (h && h.textContent.endsWith('|')) h.textContent = h.textContent.slice(0, -1);
}
```

## Fragment-gated animation phases

Gate distinct animation phases behind separate reveal.js fragments using invisible fragment spans:

```html
<span class="fragment" id="appear-fragment" style="position:absolute;pointer-events:none"></span>
<span class="fragment" id="start-fragment"  style="position:absolute;pointer-events:none"></span>
```

Wire each phase in `fragmentshown`/`fragmenthidden`:

```js
Reveal.on('fragmentshown',  ({ fragment }) => {
  if (fragment.id === 'appear-fragment') appearAnim();
  if (fragment.id === 'start-fragment')  runAnim();
});
Reveal.on('fragmenthidden', ({ fragment }) => {
  if (fragment.id === 'start-fragment')  stopLoop();    // keep elements, stop loop
  if (fragment.id === 'appear-fragment') stopAnim(true); // fly elements out
});
```

Back-navigation also fires `slidechanged`. On arrival at the slide, check which fragments are already visible to restore the right state:

```js
Reveal.on('slidechanged', ({ currentSlide }) => {
  if (currentSlide?.id === 'my-slide') {
    const af = document.getElementById('appear-fragment');
    const sf = document.getElementById('start-fragment');
    if (sf?.classList.contains('visible'))      runAnim();
    else if (af?.classList.contains('visible')) appearAnim();
  } else {
    stopAnim();
  }
});
```

## Detecting slides by class, not id

Avoid putting ids on slides for JS targeting. Drop an empty marker element inside the slide instead:

```markdown
::: {.my-slide}
:::
```

Detect it in `slidechanged` via `currentSlide.querySelector('.my-slide')`, and find the section from any DOM query via `.closest('section')`:

```js
const section = document.querySelector('.my-slide')?.closest('section');
```

For CSS, use `:has()` to style the section:

```css
section:has(.my-slide) {
  overflow: hidden;
}
```

This keeps slide ids out of JS and CSS, and works naturally with Quarto's `:::` div syntax.

**Trap:** checking whether a section itself carries the marker class with `section.querySelector('.my-slide')` returns `null` even when the class is on the section — `querySelector` only walks descendants. Use `section.classList.contains('my-slide')` or `section.matches('.my-slide')` instead.

## Reusable behavior with a factory

When multiple slides need an independently-running instance of the same behavior, wrap the state and loop in a factory instead of top-level variables:

```js
function makeFloatBehavior(defs) {
  let running = false;

  function floatOne(el) {
    if (!running) return;
    anime({ targets: el, translateX: x, translateY: y, ..., complete: () => floatOne(el) });
  }

  return {
    start(section) {
      if (running) return;
      running = true;
      ensureElements(section, defs);
      defs.forEach(({ id }) => { ... floatOne(el); });
    },
    stop() {
      running = false;
      anime.remove(...); anime({ targets, opacity: 0, ... });
    },
  };
}

const floatBehavior = makeFloatBehavior(FLOAT_DEFS);
// floatBehavior.start(currentSlide) / floatBehavior.stop()
```

Each factory call gets its own `running` flag, so state doesn't leak between slides.

Extract the "create if missing, append to section" pattern too, since it recurs wherever elements might not exist yet (entry animation, loop restart after a layout shift):

```js
function ensureElements(section, defs) {
  defs.forEach(({ id, ... }) => {
    if (!document.getElementById(id)) section.appendChild(makeElement(id, ...));
  });
}
```

Define element lists as constants and pass them to `ensureElements`, the factory, and any stop/fly-out logic, to avoid the same array hardcoded in three places.

## Edge-biased random positions

For floating elements that should avoid the slide center (e.g. to leave room for text), sample from one of four edge strips instead of the full slide area:

```js
function edgePos() {
  const region = Math.floor(Math.random() * 4);
  switch (region) {
    case 0: return { x: Math.random() * 250 + 30,   y: Math.random() * 660 + 30  }; // left
    case 1: return { x: Math.random() * 250 + 1000, y: Math.random() * 660 + 30  }; // right
    case 2: return { x: Math.random() * 1220 + 30,  y: Math.random() * 130 + 30  }; // top
    default: return { x: Math.random() * 1220 + 30, y: Math.random() * 100 + 590 }; // bottom
  }
}
```

Tune the strip widths (250, 130, 100) to give the center content more or less breathing room.

## Overlap via return values

Control how much animations overlap by what time value a builder function returns. Returning earlier than the true end time lets the next move start while the current one is still finishing:

```js
return startAt + 2100; // next move starts here even though the element settles at 2830
```

Tune up (less overlap) or down (more overlap) without touching any individual animation.

## Reveal.js lifecycle events don't replay

`Reveal.on('ready', cb)` only fires for *future* `ready` events. If the listener is registered after Reveal has finished initializing — common when scripts are injected via `include-after-body` — the callback never runs.

Fix: register `slidechanged`/`fragmentshown`/`fragmenthidden` listeners unconditionally as soon as `Reveal` exists. Those handlers fire on future user navigation regardless of whether `ready` already passed, so they don't need `ready`-gating.

```js
function onReveal(cb) {
  function go() {
    if (typeof Reveal === 'undefined' || !Reveal.on) {
      setTimeout(go, 50);
      return;
    }
    cb(); // register listeners now; do not wait for 'ready'
  }
  if (document.readyState === 'loading') {
    document.addEventListener('DOMContentLoaded', go);
  } else {
    go();
  }
}
```

Only the boot-time "did we land directly on slide X?" check actually needs to know Reveal is ready (so `Reveal.getCurrentSlide()` returns a real value). For that, poll briefly or accept that direct-deep-link entry won't auto-trigger an ambient animation until the user navigates once.

## Verify shared helpers empirically before building on them

If N slide modules all call into one shared helper (a `registerSlide`-style function, an element factory, etc.), a bug in the helper silently breaks every dependent feature at once. Cheap surface checks — HTTP 200, syntax passes, classes present in rendered HTML — won't catch behavioral bugs in the helper. Only running the deck in a real browser does.

When the unit of failure is the helper, the smallest viable test is "does anything trigger at all?" — there's no narrower test that's useful. Run that before assuming the foundation is sound; skipping it to save time costs more time when every module has to be re-investigated to localize the bug.

## Gotchas summary

- Convert screen pixels to slide pixels before feeding anime.js any measured coordinate.
- Never animate the same property with both CSS transitions and anime.js.
- Handle `fragmenthidden`, not just `fragmentshown`.
- Design loops to be self-cancelling so stop/restart never needs saved phase state.
- Snapshot values before mutating state when building a timeline synchronously.
- Clean up incremental DOM mutations (like a trailing typewriter cursor) on every stop path, not just on natural completion.
- Register lifecycle listeners unconditionally; don't gate them behind `ready`.
