---
name: css-and-layout
description: Load when writing or debugging nontrivial CSS — layout choices (flexbox vs grid, intrinsic sizing), container queries, cascade layers, custom properties/design tokens, z-index/stacking bugs, centering/overflow/height traps, scroll-driven animations, view transitions, or choosing between Tailwind/CSS-in-JS/CSS Modules.
---

# CSS and Layout

## Core mental model

1. **CSS is a set of layout algorithms, not a bag of properties.** Every element is laid out by exactly one formatting context (flow, flex, grid, table, positioned). Most "CSS is unpredictable" complaints come from applying property intuitions from one algorithm inside another (`width` behaves differently in flow vs flex vs grid; `margin: auto` centers in flex/grid but only horizontally in flow). First question when debugging: *which algorithm is laying out this box?*
2. **Sizing is a negotiation between container and content.** Every box has a `min-content` size (longest unbreakable word/item), a `max-content` size (never wrap), and a constraint from outside. Modern layout is choosing where on that spectrum each track/item sits (`fit-content`, `minmax()`, `flex-basis`, `auto`). Overflow bugs are almost always this negotiation going wrong — usually a forgotten `min-width: auto` default in flex/grid items.
3. **The cascade is an architecture layer, not an enemy.** With `@layer`, `:where()`, and scoped custom properties, specificity conflicts are a design choice you failed to make, not fate. Decide the layer order once (`reset, tokens, base, components, utilities, overrides`) and specificity wars end.
4. **Custom properties are a runtime, not find-and-replace variables.** They inherit, cascade, can be invalid-at-computed-value-time, and can be redefined per subtree — which makes them a design-token *delivery mechanism* (theming, container-responsive values, component APIs) that preprocessor variables never were.
5. **Stacking contexts are scopes for z-index.** `z-index` competes only among siblings within the same stacking context. Every "z-index: 99999 doesn't work" bug is an element trapped in a context created by an ancestor (`transform`, `filter`, `opacity < 1`, `position: fixed/sticky`, `contain`, `will-change`, ...).

## Layout algorithm selection — the reasoning chain

Ask in order:

1. **Is it one-dimensional content distribution** (a row/column of items whose sizes should come from their content — toolbar, tag list, nav, form row)? → **Flexbox**. Flexbox is content-out: items size themselves, then negotiate leftover space via `flex-grow/shrink`.
2. **Is it two-dimensional, or must items align across rows/columns, or should the *container* dictate sizes** (page shell, card grid, form label/input alignment, image mosaic)? → **Grid**. Grid is container-in: you define tracks, content slots in. The tell for grid-even-in-1D: "these should line up" across separate rows → shared column tracks (or `subgrid` for aligning nested items to the parent's tracks — Baseline since 2023).
3. **Is it text with things floating in it?** → flow + `float` (rare, legitimate: pull-quotes, drop caps).
4. **Does it overlay?** → grid with stacked cells (`grid-area: 1 / 1`) or positioned layout; prefer grid stacking for overlays that must size to content.

Key sizing tools and when each wins:
- `grid-template-columns: repeat(auto-fill, minmax(min(14rem, 100%), 1fr))` — the responsive card grid without media queries; the inner `min()` prevents overflow below 14rem viewports.
- `fit-content` / `width: max-content` — "shrinkwrap this block" (buttons-as-blocks, captions).
- `flex: 1 1 0` vs `flex: 1 1 auto`: basis `0` divides space *equally ignoring content*; basis `auto` distributes *leftover* space, so bigger content stays bigger. Choosing wrong is why "equal columns aren't equal."
- `aspect-ratio` for media boxes; stop padding-top hacks.

## Container queries vs media queries

- Media queries answer "how big is the viewport" — correct for page-level layout shifts (sidebar collapses, nav becomes drawer).
- Container queries answer "how big is the space *this component* got" — correct for any reusable component (card that goes horizontal when wide), because the same card renders in a 300px sidebar and a 900px main column on the same page. Baseline since 2023; use freely as of 2026.
- Mechanics: ancestor declares `container-type: inline-size` (and optionally `container-name`); component queries `@container (min-width: 30rem) { ... }`. Gotcha: a size-queried container cannot size itself from its contents in the queried axis — `inline-size` containment forces its inline size to come from context. Don't make everything a container reflexively.
- Container query *units* (`cqi`, `cqw`) enable fluid component typography: `font-size: clamp(1rem, 3cqi, 1.5rem)`.
- Style queries (`@container style(--card: featured)`) — check current support before relying on them; historically Chromium-first.

## Cascade layers and specificity architecture

- Declare order up front: `@layer reset, tokens, base, components, utilities;`. Later layers beat earlier ones **regardless of specificity** — a `.btn` in `utilities` beats `#app nav a.fancy` in `base`. Unlayered styles beat all layers (design decision: keep app overrides unlayered, or add an explicit final layer).
- Put third-party CSS in an early layer: `@import url(vendor.css) layer(vendor);` — ends the "fighting the library's selectors" genre of bug.
- Inside component styles, keep selector specificity flat: single class, `:where()` for anything structural. `:where(ul, ol) li` has the specificity of `li` alone; `:is(ul, ol) li` takes the *highest* specificity of its arguments. Use `:is()` for convenience grouping, `:where()` when you intend to be overridable.
- `!important` inverts layer order (earlier layers' importants win) — one more reason to reserve it for utilities-that-must-win, if at all.

## Custom properties as a token runtime

- Two-tier tokens: primitive (`--blue-600: oklch(0.55 0.15 250)`) → semantic (`--color-accent: var(--blue-600)`). Components consume only semantic tokens. Theming = redefining semantics on `:root[data-theme=dark]` or any subtree — dark mode, brand theming, and "this card is inverted" all become one mechanism.
- `@property` (Baseline 2024) registers type + initial value, enabling *animation* of custom properties (gradient angles, numeric tokens) and guarding against garbage values.
- Component prop surface: expose `--gauge-size`, `--gauge-color` with fallbacks (`var(--gauge-size, 3rem)`) — callers style without piercing internals.
- Pitfall: an invalid `var()` substitution doesn't fall back to the previous declaration — the property becomes its *initial/inherited* value ("invalid at computed-value time"). `color: var(--oops)` where `--oops: 12px` gives you inherited color, not your earlier `color: red` line.
- `prefers-color-scheme` + `light-dark()` (Baseline 2024) collapses many dark-mode overrides: `color: light-dark(#111, #eee)` (requires `color-scheme: light dark`).

## Modern selectors worth their complexity

- `:has()` (Baseline since Dec 2023) — parent/ancestor selection: `label:has(input:invalid)`, `.card:has(img)`, form-level state `form:has(.error) .submit`. It's a live query; keep argument selectors cheap on huge DOMs, but don't fear it.
- `:focus-visible` not `:focus` for focus rings (keyboard-only ring); `:focus-within` for container highlighting.
- `:nth-child(... of S)` syntax scopes counting: `:nth-child(2 of .visible)`.
- `:user-valid` / `:user-invalid` instead of `:valid`/`:invalid` for form styling — they wait for user interaction, so required fields don't render red on first paint (Baseline 2023+).
- Nesting is Baseline (2023+): use it, keep it ≤2 levels; nesting doesn't create specificity walls but it hides them.
- `@scope` (donut scoping: style between `.card` and `.card-slot` without leaking into slotted content) — powerful for design systems, but verify current cross-browser support before shipping; historically Firefox lagged here.

## Stacking contexts — the z-index bug taxonomy

Diagnosis chain for "element renders under something despite high z-index":
1. In DevTools, walk *ancestors* of the losing element looking for stacking-context creators: `transform`, `translate/scale/rotate`, `filter`, `backdrop-filter`, `opacity < 1`, `will-change: transform/opacity`, `contain: layout|paint|strict`, `isolation: isolate`, `position: fixed/sticky`, or positioned + z-index.
2. The fight is resolved at the *nearest common ancestor* level: compare the two ancestor branches' z-orders there. The fix is raising the ancestor's z-index or removing the accidental context — the deep element's own z-index is irrelevant.
3. `position: fixed` inside a `transform`ed ancestor positions relative to that ancestor, not the viewport — the classic "fixed header breaks after adding an animation."
4. Prevention: give each app-level region an explicit `isolation: isolate` and a documented z-scale (tokens: `--z-dropdown: 100; --z-modal: 200; --z-toast: 300`). Better: render overlays through the **top layer** — `<dialog>.showModal()` and `popover` (Baseline 2024) escape stacking contexts entirely; most z-index wars are obsolete for modals/menus that adopt them.

## Scroll-driven animations & view transitions (support as of mid-2026)

- Scroll-driven animations (`animation-timeline: scroll()/view()`) are supported across current Chrome, Firefox, and Safari — usable as progressive enhancement without polyfills for evergreen-browser audiences; still gate with `@supports (animation-timeline: view())` for long-tail browsers. They run off the main thread — prefer them over JS scroll handlers for parallax/reveal/progress.
- Reveal-on-scroll: `animation: fade-in both; animation-timeline: view(); animation-range: entry 0% entry 100%;`.
- Same-document view transitions (`document.startViewTransition`, `view-transition-name`) are Baseline (2025) — use for SPA state/route changes; frameworks (Next.js, React Router, Vue Router) have integrations. Cross-document view transitions (MPA) are newer and not yet uniformly supported — treat as enhancement and verify current support before promising them.
- Always wrap motion in `@media (prefers-reduced-motion: no-preference)` (see `web-accessibility`).

## Logical properties

Default to logical: `margin-inline`, `padding-block`, `inset-inline-start`, `border-start-start-radius`, `inline-size`/`block-size`, `text-align: start`. Physical properties are the correct choice only when the geometry is genuinely physical (a shadow direction, a map overlay). This makes RTL support nearly free instead of a retrofit.

## The classic traps (with the actual mechanism)

- **`height: 100%` does nothing**: percentage heights resolve against the parent's *definite* height; if the parent's height is auto, it's ignored. Fix at the root with `html, body { height: 100% }` chain, or skip it: `min-height: 100dvh` on the shell (use `dvh` not `vh` — `vh` overshoots under mobile dynamic toolbars), or grid: `body { min-height: 100dvh; display: grid; grid-template-rows: auto 1fr auto; }`.
- **Flex/grid item refuses to shrink / blows out the layout**: flex and grid items default to `min-width: auto` (= min-content). A long URL or `<pre>` sets a floor. Fix: `min-width: 0` on the item (or `overflow: hidden`). This is the single most common modern layout bug; check it *first* for any horizontal overflow inside flex/grid.
- **Centering**: one column of things → `display: grid; place-content: center;`. Single line of text → `line-height` trick is dead; use flex + `align-items: center`. Absolute-position centering → `inset: 0; margin: auto;` with fixed size, or `translate: -50% -50%` variant.
- **Margin collapse** surprises (first child's margin poking out of parent): only in flow layout, only block margins; flex/grid/`display: flow-root`/padding/border all stop it. If a parent "moves when the child has margin," this is it.
- **`overflow: hidden` on one axis isn't**: `overflow-x: hidden; overflow-y: visible` computes y to `auto` — you get a scrollbar, not visible overflow. Also `overflow` (any non-visible) creates a scroll container that clips `position: sticky` and absolutely-positioned descendants' visibility.
- **`position: sticky` not sticking**: an ancestor with `overflow: hidden/auto/scroll` becomes the scroll container it sticks within; or the sticky element already fills its container (nothing to stick within). Check ancestors for overflow first.
- **Animating `height: auto`**: transitions to/from `auto` historically don't interpolate. The robust cross-browser pattern is grid: wrapper `display: grid; grid-template-rows: 0fr;` → `1fr` on open, inner element `min-height: 0; overflow: hidden;` — smooth height animation with no JS measurement. (`interpolate-size: allow-keywords` / `calc-size()` solve this natively but were Chromium-only historically — verify current support before relying on them.) Prefer `transform`/`opacity` animations wherever possible; animating layout properties triggers reflow every frame.
- **Percentage padding/margin resolve against the *inline* size of the containing block** — including `padding-top: 100%`. That's why the old aspect-ratio hack worked, and why a percentage vertical margin "mysteriously" changes with width.
- **`gap` vs margins**: use `gap` for spacing between flex/grid siblings (no first/last-child exceptions, no margin collapse); reserve margins for spacing between *sections*. Mixed margin+gap spacing is why "the spacing doubles sometimes."
- **Absolutely-positioned children of a grid**: a `position: absolute` child of a grid container with `inset: 0` covers the whole container, not its assigned area — unless you also give it a `grid-area`, in which case its containing block is that area. Both behaviors are useful; know which you're getting.
- **Layout shift from scrollbars appearing**: `scrollbar-gutter: stable` on the scroll container (or `html`) reserves the gutter and stops the page "jumping" when content grows.
- **Ragged headlines/last lines**: `text-wrap: balance` for headings (line-count-limited), `text-wrap: pretty` for body-paragraph orphan control — both Baseline; zero-JS wins.

## CSS-in-JS vs utility classes (landscape as of 2026)

- **Runtime CSS-in-JS (styled-components, Emotion) is legacy for new work**: serialization cost per render, and incompatibility with Server Components (runtime style injection needs client execution) pushed it out. styled-components is in maintenance mode. Don't propose it for greenfield.
- **Tailwind v4** is the dominant default (CSS-first config via `@theme`, built on cascade layers and custom properties, Vite-native, no `tailwind.config.js` required). It wins on colocation, constraint enforcement, and dead-style elimination; costs are class-string readability and a shared-vocabulary learning curve.
- **Zero-runtime CSS-in-JS** — vanilla-extract, Panda CSS, StyleX (Meta) — compiles typed style objects to static/atomic CSS. Choose when you need typed tokens + dynamic-variant composition with RSC compatibility, or Meta-scale style dedup (StyleX).
- **CSS Modules** remain a perfectly good boring answer, especially paired with tokens-as-custom-properties.
- The reasoning: the real decision inputs are *RSC compatibility, build-time extraction, and how design tokens are enforced* — not aesthetics of syntax. Any of {Tailwind, CSS Modules, zero-runtime} is defensible; runtime injection is the only wrong answer for new SSR/RSC apps.

## How an expert thinks through this

*Scenario: card grid; one card's content overflows horizontally on mobile, and the "New" badge renders behind a later card's image.*

Which algorithm? Cards are in `grid-template-columns: repeat(auto-fill, minmax(280px, 1fr))`. On a 320px viewport minus padding, the available space is under 280px — `minmax`'s min wins and the track overflows. Considered a media query to change the min at small sizes: works, but it's treating the symptom; the robust idiom is `minmax(min(280px, 100%), 1fr)`. Inside the card, a long unbroken order-ID string still overflows its flex row — prior says `min-width: auto`; confirmed in DevTools (the text's min-content exceeds the row). `min-width: 0` on the flex child + `overflow-wrap: anywhere` on the ID. Now the badge: it's `position: absolute; z-index: 10` inside the card; the *next* card has a CSS `filter` on its image hover creating a stacking context... no — walk it properly: the badge's own card has `transform: translateY(...)` from an entrance animation, creating a stacking context whose z-index is auto, so it interleaves with later siblings in DOM order. Options: z-index on the *card* when hovered (fragile, N cards fighting), remove the transform after animation (`animation-fill-mode` juggling — brittle), or `isolation: isolate` + `z-index: 1` on the hovered/badged card. Actually the cleanest: the badge only needs to beat content *within its own card* — it already does; the visual bug is the next card overlapping due to a negative margin from the old design. Check that before touching z-index. It was the margin. Stopping rule: bug reproduced, mechanism named (track min, min-content floor, sibling overlap), fix verified at 320/768/1280 — done; no refactor of the whole grid.

## Worked micro-examples

**1. Layered architecture + tokens + container-responsive component:**

```css
@layer reset, tokens, base, components, utilities;

@layer tokens {
  :root {
    color-scheme: light dark;
    --color-surface: light-dark(#fff, #16181d);
    --color-accent: oklch(0.62 0.17 255);
    --z-modal: 200; --z-toast: 300;
  }
}

@layer components {
  .card-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(min(18rem, 100%), 1fr));
    gap: 1rem;
    container-type: inline-size;
    container-name: cards;
  }
  .card { display: flex; flex-direction: column; gap: .5rem; background: var(--color-surface); }
  .card > * { min-width: 0; }               /* kill the min-content floor once */
  @container cards (min-width: 40rem) {
    .card { flex-direction: row; align-items: center; }
  }
}
```

**2. Scroll-reveal + animated disclosure, no JavaScript:**

```css
/* Progressive-enhancement scroll reveal (compositor-thread, honors reduced motion) */
@media (prefers-reduced-motion: no-preference) {
  @supports (animation-timeline: view()) {
    .reveal {
      animation: reveal-in both;
      animation-timeline: view();
      animation-range: entry 0% entry 60%;
    }
    @keyframes reveal-in { from { opacity: 0; translate: 0 1rem; } }
  }
}

/* Height-auto disclosure via grid fraction trick */
.disclosure { display: grid; grid-template-rows: 0fr; transition: grid-template-rows .25s ease; }
.disclosure[data-open] { grid-template-rows: 1fr; }
.disclosure > .content { min-height: 0; overflow: hidden; }

/* Registered custom property → animatable gradient angle */
@property --angle { syntax: "<angle>"; inherits: false; initial-value: 0deg; }
.spinner-ring { background: conic-gradient(from var(--angle), var(--color-accent), transparent);
                animation: spin 1s linear infinite; }
@keyframes spin { to { --angle: 360deg; } }
```

## Verification / self-check

- Name the formatting context of the buggy box before proposing a fix; if you can't, open DevTools' layout panel first.
- Any horizontal-overflow fix: did you check `min-width: auto` on flex/grid items before adding `overflow: hidden`?
- Any z-index fix: did you identify the exact ancestor creating the stacking context, and consider `<dialog>`/`popover`/top layer instead?
- Responsive claims: verified at 320px, an intermediate width, and with 200% zoom (zoom reveals fixed-size assumptions).
- Feature-support claims (style queries, cross-document view transitions, newest features): check caniuse/MDN for the project's browser matrix — don't recall.
- New component checklist: works with 3x the expected content, works empty, works at 320px, focus-visible styles present, no physical properties without reason.
- Stopping rule: the bug's mechanism is named and the fix survives content stress (long words, many items, RTL if applicable). Refactoring unrelated CSS "while here" is scope creep.
