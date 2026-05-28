# AGENTS.md

Guidance for any AI coding agent (Claude Code, Cursor, Copilot, Codex, etc.) operating in this repo. Mirrors [CLAUDE.md](CLAUDE.md) — read both.

## Commands

- `npm run dev` — Vite dev server on port 3000 ([vite.config.js](vite.config.js))
- `npm run build` — production build (base path `/town-gossip-board-ripple/` for GitHub Pages)
- `npm run serve` — preview built bundle
- `npm run lint` — ESLint (`@ripple-ts/eslint-plugin` recommended)
- `npm run format` / `npm run format:check` — Prettier (tabs, single quotes, 100-col)

No tests configured.

## Stack snapshot

- **Ripple** (`ripple-ts`) SPA — NOT React, NOT Svelte. `.ripple` SFCs compiled by `@ripple-ts/vite-plugin`.
- Tailwind v4 + daisyUI + **RPGUI** (CDN in [index.html](index.html)).
- TypeScript with `jsxImportSource: "ripple"`.

## Ripple idioms you must use

- Components: `export component Name(props) { ... }` in `.ripple` files.
- State: `let x = track(initial)` then read/write with `@`: `@x = ...`, `for (const i of @x)`. Bare `x` is the proxy, not the value.
- Template loops: `for (const item of @list) { <div>{item.name}</div> }` — no `.map()`.
- Mount: `mount(App, { target })` in [src/index.ts](src/index.ts).
- `<Portal target={document.body}>`, `<head>`, scoped `<style>` (use `@keyframes -global-foo` for global).

## Architecture rules

- [src/components/main_board.ripple](src/components/main_board.ripple) owns state. `rumorsAll` = master, `rumors` = filtered view.
- Filters re-derive from the static `rumorData` import (NOT from `@rumors`) so they don't compound.
- Mutations (create/delete) **must** update both `@rumors` AND `@rumorsAll`. This is load-bearing.
- Dialogs: native `<dialog>` opened via `document.getElementById('id').showModal()` from inline `onclick` strings. Inputs read via `getElementById`, not bound state. Keep this pattern.
- RPGUI requires the `rpgui-content` wrapper. Tailwind utilities often need `!` to override (e.g. `!text-center`, `!p-4`).
- [src/models/rumor.ts](src/models/rumor.ts) declares `vote: number` but mock data omits it — voting UI not implemented yet.

## Deployment

`vite.config.js` hardcodes `base: '/town-gossip-board-ripple/'` for GitHub Pages. Production builds break if served elsewhere.

## Skill routing (Claude Code only)

If you are Claude Code, follow the skill rules in [CLAUDE.md](CLAUDE.md):
1. **Every turn** → invoke `karpathy-guidelines` first.
2. **UI work** → also invoke one of `redesign-existing-projects` / `industrial-brutalist-ui` / `minimalist-ui` / `high-end-visual-design` (default fallback) / `brandkit` per the priority table.

Other agents without a skill system: apply the same principles manually — surgical changes, surface assumptions, match the existing RPGUI + Tailwind + daisyUI aesthetic.

## Don't

- Don't add a test framework, state management lib, or routing without being asked.
- Don't rewrite `.ripple` components into React/JSX patterns.
- Don't introduce build steps outside Vite + the Ripple plugin.
- Don't change `base` in `vite.config.js` — it's tied to the GH Pages deploy URL.
- Don't mutate the seed `rumorData` import; treat it as immutable.
