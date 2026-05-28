---
name: ripple-ts
description: Authoritative reference for the Ripple TS UI framework (ripple-ts) — covers `.ripple`/`.tsrx` components, `track`/`&[]` reactivity, template-only JSX rules, control flow, reactive collections, React compat, and common pitfalls. Use whenever the user works on Ripple code, edits a `.ripple` or `.tsrx` file, mentions Ripple, `track()`, `mount()`, `<Portal>`, `RippleArray`, the `@` sigil, the `&[]` destructure form, or anything that looks like Ripple syntax. Use proactively for any UI work in a Ripple project — Ripple looks like JSX but has stricter rules that LLMs routinely violate.
---

# Ripple TS

Ripple = fine-grain reactive TS UI framework. JSX-shaped but **not React**. Templates are statements inside `component` bodies, not values returned from functions. Reactivity = `track()` + `&[]` destructure (or `@` sigil in older `.ripple` projects).

This SKILL.md covers the rules you must not get wrong. For depth see `references/`.

## File extensions

- `.tsrx` — current official extension (per ripple-ts.com)
- `.ripple` — older/alt extension still widely used (e.g. this project)

Both compile through `@ripple-ts/vite-plugin`. Same syntax. Match what the project already uses — do not rename files.

## The non-negotiable rules

These are the rules LLMs break most often. Get them right first.

### 1. Components are declarations, not functions returning JSX

```ripple
component Button(props: { text: string; onClick: () => void }) {
  <button onClick={props.onClick}>{props.text}</button>
}
```

No `return <JSX/>`. No arrow components. Templates live **only** inside a `component` body. You cannot assign JSX to a variable from a regular function, and you cannot return JSX. The one exception is expression islands — see below.

### 2. Static text must be quoted; raw text is a syntax error

```ripple
<div>"Hello World"</div>     // ok — static text
<div>{greeting}</div>         // ok — dynamic
<div>Hello World</div>        // ERROR
```

Use `&quot;` / `&amp;` HTML entities inside the quoted string. Backslash escapes a literal quote.

### 3. Early return is a guard only

```ripple
if (!ready) {
  <p>"Loading…"</p>
  return;        // bare return only
}
```

Never `return <JSX/>`, never `return value`. To branch UI, write the JSX in both branches of an `if`/`else`.

### 4. Reactivity = `track()` + `&[]`

```ripple
import { track } from 'ripple';

let &[count] = track(0);
let &[double] = track(() => count * 2);   // derived
count++;                                   // direct read/write
```

`track()` cannot be called at module scope. Derived values pass a function. Need the underlying `Tracked` ref to pass around? Destructure both:

```ripple
let &[count, countRef] = track(0);
```

Hot paths / storage can use `.value`:

```ripple
const count = track(0);
count.value++;
```

**Legacy `@` sigil**: some `.ripple` projects use `@var` to read/write a tracked value declared as `const var = track(0)`. If you see that pattern in a file, match it; do not mix the two styles in one file.

### 5. Lazy destructure props with `&{ }` / `&[ ]`

```ripple
component Child(&{ count, className, children }: Props) {
  <button class={className}>{children}</button>
}
```

Eager destructure (`{ count }`) snapshots the value and breaks reactivity. `&{ }` keeps it live. Same for tracked-ref destructure: `&[count]`.

### 6. Control flow uses statements, not `.map()`

```ripple
for (const item of items; index i; key item.id) {
  <li>{item.text}" #"{i}</li>
}

if (x) { <A /> } else { <B /> }

switch (status) {
  case 'a':                 // fall-through ok
  case 'b': <P /> break;   // break is mandatory
  default: <Q />
}

try { <Risky /> } catch (e) { <div>"Err: "{e.message}</div> }
```

No `.map()` rendering. `key` is needed for plain-object arrays; `RippleArray` items are auto-keyed.

## Quick reference

### Mount
```ts
import { mount } from 'ripple';
const cleanup = mount(App, { target: document.getElementById('root')!, props: {} });
```

### Effects / scheduling
```ripple
import { effect, tick, untrack, on } from 'ripple';
effect(() => { console.log(count); });
tick().then(() => { /* after DOM commit */ });
untrack(() => count);
effect(() => on(window, 'resize', handler));   // on() returns its own cleanup
```

### Context (no get/set in event handlers or module scope)
```ripple
import { Context } from 'ripple';
const Theme = new Context<'light'|'dark'>('light');
component Parent() { Theme.set('dark'); <Child /> }
component Child()  { const t = Theme.get(); }
```

### Children + sub-components
```ripple
import type { Children, Component } from 'ripple';
component Card(props: { children: Children; Footer?: Component }) {
  <div>
    {props.children}
    if (props.Footer) { <props.Footer /> }
  </div>
}
```
Pass nested components as explicit props — don't nest `component` declarations.

### Dynamic component / tag
```ripple
let &[Cmp] = track(() => Child1);
<@Cmp />

let &[tag] = track('div');
<@tag class="x">"Hi"</@tag>
```

### Refs (three forms; named recommended)
```ripple
let el: HTMLDivElement | null | undefined;
<div {ref el}>"x"</div>
<input ref={inputTracked} />
<button anyName={ref (n) => console.log(n)}>"x"</button>
```

### Reactive collections (import from `'ripple'`)
`RippleArray`, `RippleObject`, `RippleSet`, `RippleMap`, `RippleDate`. Reactive `.length`, `.size`, mutating methods. Use these instead of plain arrays/objects whenever you need reactive collections.

### Portal
```ripple
import { Portal } from 'ripple';
<Portal target={document.body}><div class="modal">…</div></Portal>
```

### Raw HTML / explicit text
```ripple
<article>{html trustedString}</article>   // trusted only
<div>{text mightLookLikeHtml}</div>        // never parsed as HTML
```

### Expression islands (JSX as a value)
Only place a template can be a value. Useful for passing UI through plain JS:
```ripple
const title = <tsrx><span class="title">"Hello"</span></tsrx>;
const frag  = <>"shorthand"</>;
```

### Class / style helpers
```ripple
<div class={{ active: isActive, ghost: false }}></div>
<div class={['btn', { lg: big }, big && 'shadow']}></div>
<div style={{ color, fontWeight: 'bold', 'background-color': 'gray' }}></div>
```

### Style scoping
`<style>` is auto-scoped per component. Parent styles don't leak into children. To share a scoped class across a child boundary use `{style "name"}`. To break out, use `:global(...)`. To make a keyframe global: `@keyframes -global-name { … }`.

## When to read the reference files

- `references/react-compat.md` — using `@ripple-ts/compat-react`, embedding React JSX via `<tsx:react>`, mounting Ripple inside React, common compat mistakes.
- `references/reactivity-deep.md` — `track` get/set hooks, transport reactivity (passing tracked refs to plain functions), `effect` vs `tick` vs `untrack`, when to use `.value`.
- `references/cheatsheet.md` — compressed one-pager of every imported name, syntax form, and gotcha.

Read these only when the task touches that area. SKILL.md alone covers ~90% of edits.

## Common mistakes to avoid

- Writing components as `function Foo() { return <…/> }` — use `component Foo() { … }`.
- Leaving raw text unquoted inside an element.
- Returning a JSX value from `if`/early-return — only `return;` is allowed.
- Eager-destructuring props (`{ count }`) and losing reactivity — use `&{ count }`.
- Rendering lists with `.map()` — use `for (… of …; key …)`.
- Using `className` outside `<tsx:react>` — it's `class` in Ripple JSX.
- Calling `track()` at module scope.
- Calling `Context.get()`/`Context.set()` inside event handlers.
- Mixing `&[]` and `@` sigil styles in the same file — pick whatever the file already uses.

## Workflow

1. Before editing a `.ripple`/`.tsrx` file, skim it to detect which reactivity style it uses (`&[]` vs `@`) and match it.
2. State which rule(s) above apply to the change you're about to make (briefly — one line is fine).
3. Make the edit. Keep changes surgical (per Karpathy guidelines this repo runs first).
4. If the change touches reactivity, control flow, or refs, re-read the relevant section above before saving.
