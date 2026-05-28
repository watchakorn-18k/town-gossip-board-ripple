# Ripple ↔ React Compat

Package: `@ripple-ts/compat-react`. **Client-only — no SSR.**

## Config (`ripple.config.ts`)
```ts
import { reactCompat } from '@ripple-ts/compat-react';
import { defineConfig } from '@ripple-ts/vite-plugin';

export default defineConfig({
  compat: { react: reactCompat() },
});
```
Without `reactCompat()`, `<tsx:react>` blocks won't compile.

## Embedding React inside Ripple — `<tsx:react>` block

Inside `<tsx:react>` you are writing **React JSX** (so `className`, React events, React components):
```ripple
component MyView() {
  <div class="ripple-wrap">           // ripple side → `class`
    <tsx:react>
      <div className="react-side">    // react side → `className`
        <ReactDatePicker />
      </div>
    </tsx:react>
  </div>
}
```

Reactive values flow into `<tsx:react>` and trigger React re-renders the same way Ripple updates its own DOM.

## Embedding Ripple inside React

```tsx
// In a .tsx React file
import { createRoot } from 'react-dom/client';
import { RippleRoot, Ripple } from '@ripple-ts/compat-react';
import MyRippleComponent from './MyRippleComponent.ripple';

createRoot(document.getElementById('root')!).render(
  <RippleRoot>
    <Ripple component={MyRippleComponent} props={{ name: 'world' }} />
  </RippleRoot>,
);
```

- `RippleRoot` must wrap the tree. Every `<Ripple/>` traverses up to find it.
- `props={...}` is the only way to pass props to the embedded Ripple component.

## Crossing the boundary

- **React Context** is visible from inside a `<tsx:react>` block to descendant React components, and from Ripple-wrapped React trees into nested React. Ripple's `Context` does **not** cross into React.
- **try/catch in Ripple templates** catches errors raised by React components rendered through `<Ripple/>` too.
- **Refs** can be forwarded across in either direction by passing the ref into the embedded component's props.

## Defining React components

A `.ripple` / `.tsrx` file's JSX is *always* Ripple JSX, even outside `<tsx:react>`. To author a React component you must either:

- put it in a `.tsx` file (preferred), or
- use `jsx()` from `react/jsx-runtime` directly.

Do not try to write `function ReactThing() { return <div className="…"/> }` inside a `.ripple` file — it will be parsed as Ripple JSX and break.

## Common compat mistakes

- Using `className` on a Ripple element (outside `<tsx:react>`). → Use `class`.
- Forgetting `RippleRoot` when embedding Ripple in React. The `<Ripple>` will not mount.
- Forgetting `reactCompat()` in `ripple.config.ts`. `<tsx:react>` will fail to compile.
- Trying to read Ripple `Context` from inside a React component embedded via `<Ripple/>`. Use React Context or pass the value through props instead.
- Writing React JSX outside a `<tsx:react>` block and expecting React semantics.
