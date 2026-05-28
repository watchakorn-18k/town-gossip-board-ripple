# Ripple Cheatsheet

Compressed reference. For prose explanation see SKILL.md.

## Imports from `'ripple'`
`track, effect, tick, untrack, on, mount, hydrate, Context, Portal, RippleArray, RippleObject, RippleSet, RippleMap, RippleDate`

Types: `Children, Component, Tracked`

SSR: `import { render } from 'ripple/server'`

React compat: `import { RippleRoot, Ripple, reactCompat } from '@ripple-ts/compat-react'`

## Component
```ripple
component Name(props: Props) { … }                                  // standard
component Name(&{ a, b = 1, ...rest }: Props) { … }                 // lazy destructure
```

## Reactivity
```ripple
let &[x]      = track(0);                  // value
let &[x, ref] = track(0);                  // value + Tracked
let &[d]      = track(() => x * 2);        // derived
let &[v]      = track(0, getHook, setHook);// hooks
const r       = track(0); r.value++;       // .value form
```

## Templates — must be in `component` body
```ripple
<el attr="…" {shorthand} {...spread}>"static"{dynamic}</el>
<el class={{a: cond}}>…</el>
<el style={{color, 'background-color': 'red'}}>…</el>
```

### Expression island (JSX as value)
```ripple
const node = <tsrx>"x"</tsrx>;    // or <>…</>  or <tsx>…</tsx>
```

## Control flow (statements, in template)
```ripple
if (c) { … } else { … }
for (const i of arr; index n; key i.id) { … }
switch (k) { case 'a': … break; default: … }
try { <X/> } catch(e) { <Y/> }
return;                              // guard only
```

## Dynamic
```ripple
<@Cmp />              // dynamic component
<@tag>…</@tag>        // dynamic tag
```

## Events
`onClick`, `onClickCapture`, `onPointerMove`, … (React-style names but Ripple event system)

## Refs (3 forms)
```ripple
<el {ref local}>…</el>
<el ref={localTracked}>…</el>
<el namedRef={ref local}>…</el>
```

## Children + sub-components
```ripple
component Card(props: { children: Children; Footer?: Component }) {
  <div>{props.children}
    if (props.Footer) { <props.Footer /> }
  </div>
}
```

## Context
```ripple
const Ctx = new Context<T>(defaultValue);
Ctx.set(v);     // in setup, not handler
Ctx.get();      // in setup, not handler
```

## Effects
```ripple
effect(() => { … });                 // reactive
effect(() => on(window,'resize',h)); // returns cleanup
await tick();                         // wait for commit
untrack(() => x);                     // read without sub
```

## Mount
```ts
const cleanup = mount(App, { target: el, props: {} });
```

## Raw HTML / text
```ripple
<div>{html trusted}</div>
<div>{text raw}</div>
```

## Style scoping
- `<style>` auto-scoped per component
- `:global(.x) { … }` escapes scope
- `{style "name"}` passes scoped class across boundary
- `@keyframes -global-name { … }` makes global keyframe

## React compat
- `<tsx:react>` block = React JSX (use `className` here)
- Outside that block in `.ripple`/`.tsrx`, JSX is Ripple (use `class`)
- Wrap React tree with `<RippleRoot>` before using `<Ripple component={…} />`
- Configure with `reactCompat()` in `ripple.config.ts`

## File extensions
`.tsrx` (current) or `.ripple` (legacy/alt). Both work. Match the project.

## Top mistakes
1. `function Foo(){ return <…/> }` — should be `component Foo(){ … }`
2. Raw unquoted text in template
3. `return <…/>` instead of bare `return;`
4. `{ x } = props` instead of `&{ x } = props`
5. `.map()` instead of `for (… of …; key …)`
6. `className` outside `<tsx:react>`
7. `track()` at module scope
8. `Context.get/set` inside event handler
9. Mixing `&[]` and `@` sigil reactivity styles in one file
