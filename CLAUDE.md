# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

- `npm run dev` — Vite dev server on port 3000 (see [vite.config.js](vite.config.js))
- `npm run build` — production build (base path `/town-gossip-board-ripple/` for GitHub Pages)
- `npm run serve` — preview built bundle
- `npm run lint` — ESLint with `@ripple-ts/eslint-plugin` recommended config
- `npm run format` / `npm run format:check` — Prettier (tabs, single quotes, 100-col, `@ripple-ts/prettier-plugin` for `.ripple` files)

No test runner configured.

## Architecture

This is a **Ripple** (`ripple-ts`) SPA — not React, not Svelte. Ripple has its own `.ripple` SFC format compiled by `@ripple-ts/vite-plugin`. Key Ripple idioms used here:

- `mount(App, { target })` in [src/index.ts](src/index.ts) bootstraps the app (no virtual DOM root component pattern).
- `.ripple` files declare `export component Name(props) { ...JSX-ish... }`. Template control flow uses `for (const x of @arr) { ... }` directly inside the markup — no `.map()`.
- **Reactive state**: `track(initialValue)` creates tracked state. Read/write with the `@` sigil: `@rumors = [...]`, `for (const r of @rumors)`. Without `@`, you get the proxy, not the value. See [src/components/main_board.ripple](src/components/main_board.ripple) for the canonical pattern.
- `<Portal target={document.body}>` is the standard way to escape the mount root.
- `<head>` inside a component sets document head (used for title).
- `<style>` blocks scope styles to the component; `@keyframes -global-foo` makes a global keyframe (see slideDownFade in main_board).

### Data flow

Single source of truth lives in [src/components/main_board.ripple](src/components/main_board.ripple):
- `rumorsAll` — the unfiltered master list (also used to populate the NPC dropdown in Filter).
- `rumors` — the currently displayed (filtered) list.
- Filter callbacks (`onChange`, `onChangeDate`) always re-filter from the static `rumorData` import, not from `@rumors`, so filters don't compound. Mutations (`onCreateNewRumor`, `onDeleteRumor`) must update **both** `@rumors` and `@rumorsAll` to stay consistent — this is load-bearing; the seed `rumorData` import is treated as immutable.
- New rumors get a generated `uuidv4()` id and a Dicebear pixel-art avatar URL derived from the NPC name.

### Dialog pattern

Dialogs are native `<dialog>` elements opened by `document.getElementById('create_rumor_dialog').showModal()` invoked from inline `onclick` strings (see main_board.ripple line ~104). Form inputs are read with `document.getElementById(...)` rather than bound state — keep this pattern when adding similar dialogs ([src/components/dialog/create_rumor_dialog.ripple](src/components/dialog/create_rumor_dialog.ripple)).

### Styling stack

Tailwind v4 (via `@tailwindcss/vite`) + daisyUI + **RPGUI** (RPG-themed CSS loaded from CDN in [index.html](index.html)). The `rpgui-content` wrapper must be present for `rpgui-container`, `rpgui-button`, `framed-*` classes to render. Many Tailwind utilities are forced with `!` to override RPGUI defaults (e.g. `!text-center`, `!p-4`).

### Models

`Rumor` interface in [src/models/rumor.ts](src/models/rumor.ts) declares `vote: number`, but the mock data in [src/mock_data/rumor.ts](src/mock_data/rumor.ts) and the create flow omit it. Treat the model as aspirational — voting UI is on the roadmap (see README "Vote / Verify"), not implemented.

## Always-On Skill: `karpathy-guidelines`

Invoke `karpathy-guidelines` via `Skill` at the **start of every prompt** in this repo — code, UI, refactor, debug, even a question. It is short, low-token, and biases toward surgical changes + surfaced assumptions. Call it first, before any other skill (including the UI router below) and before any Edit/Write. One invocation per user turn.

## Ripple Skill: `ripple-ts` (MANDATORY for any `.ripple` edit)

Whenever a prompt requires reading, editing, or creating a `.ripple` / `.tsrx` file — or discusses Ripple concepts (`track`, `@` sigil, `component`, `mount`, `Portal`, `RippleArray`, `effect`, `Context`, etc.) — invoke `ripple-ts` via `Skill` **before** the first Edit/Write. Stacks with `karpathy-guidelines` (always first) and with the UI router below if the change is visual. One invocation per user turn. Pure non-Ripple work (vite config, mock data types, README) does not need it.

## UI Skill Auto-Routing (MANDATORY for any UI change)

Whenever a prompt touches visual/UI work in this repo (any `.ripple` component, `index.css`, Tailwind classes, layout, color, typography, motion, dialog, card, filter, etc.), you MUST invoke exactly one design skill via `Skill` **before** editing. Pick using the decision table — keywords are matched case-insensitive against the user's prompt (TH or EN). If multiple match, use the priority order shown.

| Priority | Skill | Trigger keywords / intent |
|---|---|---|
| 1 | `redesign-existing-projects` | "redesign", "ปรับปรุง", "ยกระดับ", "upgrade UI", "make it premium", "audit design", "ทำให้สวยขึ้น", or any broad "improve the look" ask without a specific style direction |
| 2 | `industrial-brutalist-ui` | "brutalist", "terminal", "military", "blueprint", "tactical", "swiss", "data dashboard", "mono grid", "เท่ๆ ดิบๆ", "แนวทหาร" |
| 3 | `minimalist-ui` | "minimal", "editorial", "clean", "bento", "เรียบๆ", "โทนอบอุ่น", "monochrome", "notion-style" |
| 4 | `high-end-visual-design` | "awwwards", "agency", "premium", "cinematic", "apple-like", "linear-like", "หรูๆ", "high-end", "$150k" — and ANY UI change that doesn't match a more specific skill above (this is the **default fallback** for UI work) |
| 5 | `brandkit` | "logo", "brand kit", "identity", "brand board", "โลโก้", "แบรนด์" — image/brand asset generation, not component code |
| — | `design-taste-frontend-v1` | Only when user explicitly names it. Do not auto-select. |
| — | `full-output-enforcement` | Stack on top of a design skill when the user says "ทำให้ครบ", "อย่าตัด", "full file", "no placeholders" |

Rules:
- Exactly one design skill per UI task (priority 1–4). `brandkit` and `full-output-enforcement` may stack.
- Skill invocation happens **before** the first Edit/Write. State which skill you picked and why in one sentence.
- Pure logic / state / data changes (filter logic, mock data, model types) do **not** trigger this — only visual/layout/styling work.
- Respect the existing RPGUI + Tailwind + daisyUI stack; design skills augment style decisions, they do not replace the framework.

## Deployment

`vite.config.js` hardcodes `base: '/town-gossip-board-ripple/'` for GitHub Pages. Dev server ignores this; production builds will break if served from a different path.
