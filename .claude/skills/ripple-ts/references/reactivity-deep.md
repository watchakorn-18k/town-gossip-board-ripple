# Ripple Reactivity — Deep Dive

## `track()` shapes

```ripple
import { track } from 'ripple';

// 1. Plain reactive value
let &[count] = track(0);
count++;                       // read/write directly

// 2. Derived (function arg → recomputes when deps change)
let &[double] = track(() => count * 2);

// 3. Get both the live binding AND the Tracked ref
let &[count, countRef] = track(0);
//     ^^^^^         ^^^^^^^^
//     direct access  to pass to plain functions

// 4. Get/set hooks (interceptors)
let &[name] = track(
  '',
  (current) => { /* read hook */ return current; },
  (next, prev) => {                       /* write hook */
    if (typeof next === 'number') next = String(next);
    return next;                          // returned value becomes the new state
  },
);

// 5. .value form — for hot paths or storing in plain objects
const ref = track(0);
ref.value++;
```

**Cannot call `track()` at module top-level.** Wrap in a function/component, or use `Context` for app-wide reactive state.

## Transport reactivity — passing reactivity into plain functions

A plain function (not a `component`) can receive a tracked ref and derive from it. The caller passes the `&[name]` destructure form so the function gets a *live* reference, not a snapshot.

```ripple
function createDouble(&[count]) {
  const &[double] = track(() => count * 2);
  return double;                  // returns a Tracked
}

// Caller side:
let &[n] = track(3);
const &[d] = createDouble(track(() => n));  // wrap as Tracked first
```

Inside the function, `count` reads the live tracked value. If you destructure eagerly, you snapshot and lose reactivity.

## `effect`, `tick`, `untrack`

```ripple
import { effect, tick, untrack, on } from 'ripple';

// Re-runs whenever any tracked dep read inside changes.
effect(() => {
  console.log('count is', count);
});

// Resolve after the next DOM commit.
await tick();
// el.offsetHeight is now correct

// Read a tracked value WITHOUT subscribing to it.
effect(() => {
  console.log(visible, untrack(() => debugLabel));
});

// Native event listeners with auto-cleanup via effect return value.
effect(() => on(window, 'resize', handler));
```

`on()` returns a removal function. Returning it from an `effect` lets Ripple clean it up automatically when the effect re-runs or the owner unmounts.

## When to use `.value` instead of `&[name]`

| Situation | Form |
|---|---|
| Local component state | `let &[x] = track(0)` |
| Derived value | `let &[d] = track(() => …)` |
| Storing many refs in an object/array | `track(0).value++` |
| Hot path where you want to bypass the getter overhead a tiny bit | `.value` |
| Passing to a non-Ripple function | `track(0)` (pass the Tracked itself), then `&[x]` on the receiving side |

## Reactive collections vs primitives

`track(arrayLiteral)` makes the *binding* reactive but mutations to the array (push/pop) are not. To get reactive mutations, use the built-in collections:

```ripple
import { RippleArray, RippleObject, RippleSet, RippleMap, RippleDate } from 'ripple';

const items = new RippleArray(1, 2, 3);
items.push(4);                   // reactive — re-renders consumers
items.length;                    // reactive read

const user = new RippleObject({ name: 'Ada' });
user.age = 30;                    // reactive even though `age` wasn't there before

const tags = new RippleSet([ 'a', 'b' ]);
tags.has('a');                    // reactive

const counts = new RippleMap([['a', 1]]);

const today = new RippleDate();   // reactive getters
```

You can mix and match: store a `RippleArray` inside a tracked value, etc. The collection handles the inner reactivity; the tracked ref handles "is this the same collection or a brand-new one?".

## Gotchas

- Reading `count` inside an event handler is fine. Calling `Context.set/get` inside an event handler is not — it must run during component setup.
- Derived `track(() => …)` re-runs on every dep change; keep the body pure and cheap. For expensive derivations, gate with a coarser dep or precompute.
- Eager destructure inside a component body (`const { x } = props`) snapshots props at first run. Use `const &{ x } = props` to keep `x` live.
- `effect` runs after the first render and on every subsequent dep change. Use `tick()` to await DOM commit if you need to measure layout.
