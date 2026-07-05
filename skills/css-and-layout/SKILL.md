---
name: css-and-layout
description: Load when writing or debugging nontrivial CSS — layout choices (flexbox vs grid, intrinsic sizing), container queries, cascade layers, custom properties/design tokens, z-index/stacking bugs, centering/overflow/height traps, scroll-driven animations, view transitions, or choosing between Tailwind/CSS-in-JS/CSS Modules.
---

# CSS and Layout

## Core mental model (anchors)

1. Every box is laid out by exactly one formatting context; name the algorithm before proposing a fix. Flexbox is content-out, grid is container-in; "these should line up across rows" is the tell for grid/subgrid.
2. Sizing is a min-content/max-content negotiation; most overflow bugs are the `min-width: auto` default on flex/grid items — check it *first* for any horizontal overflow.
3. `z-index` competes only among siblings in one stacking context; the fix lives at the ancestor that created the context (or in the top layer — `<dialog>`/`popover` make most z-index wars obsolete).
4. Decide layer order once (`@layer reset, tokens, base, components, utilities`) and specificity wars end; custom properties are a runtime token-delivery mechanism, not find-and-replace.

## Current state (verified July 2026) — where cold answers are stale

- **Scroll-driven animations (`animation-timeline: scroll()/view()`) are supported across current Chrome, Firefox, AND Safari** — cold answers still say "Safari hasn't shipped"; Safari 26 shipped them. Usable as progressive enhancement without polyfills for evergreen audiences; still gate with `@supports (animation-timeline: view())` for long-tail. They run off the main thread — prefer them over JS scroll handlers.
- **Same-document view transitions are Baseline (2025)**; cross-document (MPA) view transitions are newer and not uniformly supported — verify before promising.
- **Style queries on custom properties (`@container style(--card: featured)`) went cross-browser in 2025**; the non-custom-property `style()` forms remain limited. Check the project's matrix before leaning on them.
- `interpolate-size: allow-keywords`/`calc-size()` (native height-auto animation) — still effectively Chromium-only; ship the grid `0fr → 1fr` pattern as baseline and layer this on top.
- `@scope` and stylable `<select>` (`appearance: base-select`): verify current support; historically uneven.
- Landscape: runtime CSS-in-JS is legacy (styled-components maintenance mode; RSC-incompatible); Tailwind v4 (CSS-first `@theme`), zero-runtime (vanilla-extract/Panda/StyleX), and CSS Modules are all defensible — the real decision inputs are RSC compatibility, build-time extraction, and token enforcement.

## Decision anchors (compressed)

- Card grid without media queries: `repeat(auto-fill, minmax(min(14rem, 100%), 1fr))` — the inner `min()` is the sub-14rem-viewport overflow guard.
- `flex: 1 1 0` divides ignoring content; `flex: 1 1 auto` distributes leftover — choosing wrong is why "equal columns aren't equal."
- Container queries for reusable components, media queries for page-level shifts; a size-queried container can't size from its contents in the queried axis; `cqi` units for fluid component type (`clamp(1rem, 3cqi, 1.5rem)`).
- Logical properties by default (`margin-inline`, `inline-size`, `text-align: start`); physical only for genuinely physical geometry. RTL becomes nearly free.
- `:where()` when you intend to be overridable, `:is()` takes highest argument specificity; `!important` inverts layer order — one more reason to reserve it.
- Two-tier tokens (primitive → semantic); components consume semantic only; `@property` to type+animate custom properties; `light-dark()` (needs `color-scheme: light dark`).

## Pitfalls checklist (one-liners — cold answers reproduce the mechanisms)

`height: 100%` needs a definite parent chain — or `min-height: 100dvh` (never `vh` under mobile toolbars); `min-width: 0` on the overflowing flex/grid item + `overflow-wrap: anywhere`; margin collapse is flow-only (flex/grid/`flow-root` stop it); `overflow-x: hidden` computes the other axis's `visible` to `auto` — you get a scrollbar, not visible overflow; any non-visible overflow creates a scroll container that breaks descendant `position: sticky`; sticky-not-sticking → check ancestors for overflow first; percentage padding/margin (including top) resolve against *inline* size — `aspect-ratio` replaced the hack; `gap` between siblings, margins between sections; absolutely-positioned grid child with `inset: 0` covers the whole container unless given a `grid-area`; `scrollbar-gutter: stable` stops scrollbar-appearance jumps; `text-wrap: balance` (headings) / `pretty` (orphans); IACVT — invalid `var()` substitution resets to inherited/initial, it does NOT fall back to your earlier declaration; stacking-context creators to walk for: transform/translate/filter/backdrop-filter/opacity<1/will-change/contain/isolation/fixed/sticky; `position: fixed` inside a transformed ancestor positions to that ancestor, not the viewport.

Expanded — judgment the checklist can't carry:

- **Height-auto disclosure, the robust cross-browser shape**: wrapper `display: grid; grid-template-rows: 0fr → 1fr; transition`, inner `min-height: 0; overflow: hidden`. Prefer transform/opacity for everything else — layout-property animation reflows every frame.
- **Prevention beats diagnosis for stacking**: `isolation: isolate` per app-level region + a documented z-scale in tokens; but overlays that can use the top layer (`<dialog>.showModal()`, `popover`) should — that ends the genre.
- **The debugging discipline**: before touching z-index, walk ancestors and *name* the context creator; the deep element's own z-index is irrelevant — the fight resolves at the nearest common ancestor. And before z-index at all, check whether the "overlap" is actually a negative margin or DOM-order interleave (transform-created contexts with `z-index: auto` interleave with later siblings).

## Verification / self-check

- Name the formatting context of the buggy box; any overflow fix checked `min-width: auto` first; any z-index fix identified the exact context-creating ancestor and considered the top layer.
- Responsive claims verified at 320px, an intermediate width, and 200% zoom; new components stress-tested with 3x content, empty state, RTL where applicable.
- Feature-support claims for anything in the "verify" list above: check caniuse/MDN against the project's browser matrix — don't recall.
- Motion wrapped in `@media (prefers-reduced-motion: no-preference)` (see `web-accessibility`).

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)

- Probed 15 claims: 13 baseline (cut/compressed), 1 partial (style-query support sharpened), 1 delta (expanded).
- The delta: Safari support currency — cold answers say Safari lacks scroll-driven animations (it shipped them in Safari 26) and hedge on style queries (custom-property form went cross-browser 2025). Everything mechanical (min-width:auto, stacking contexts, IACVT, layers/!important inversion, grid idiom, 0fr trick, dvh, :is/:where) Opus produced cold and exactly.
- Remaining value: support-status currency, the "check min-content floor first / walk ancestors first" diagnostic ordering, and the compressed decision anchors.
