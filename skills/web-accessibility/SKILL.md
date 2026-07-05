---
name: web-accessibility
description: Load when building or reviewing UI for accessibility — semantic HTML choices, ARIA usage, keyboard navigation and focus management (SPA routes, dialogs, menus), screen-reader behavior, color contrast, motion preferences, a11y testing strategy, or WCAG conformance questions.
---

# Web Accessibility

## Core mental model

1. **Semantics are 90% of the work.** Native HTML elements ship with role, name, state, keyboard behavior, and focus handling for free, tested across every browser/AT pair. `<button>`, `<a href>`, `<label>`, `<select>`, `<dialog>`, `<details>`, headings, lists, `<table>` — choosing them correctly does more than any amount of ARIA retrofitting. The `<div onclick>` is the root pathology: it has no role, no name, no keyboard activation, no focusability — four bugs in one element.
2. **ARIA is a promise, not an implementation.** `role="button"` tells the screen reader "this behaves like a button" but implements nothing — you now owe `tabindex="0"`, Enter *and* Space activation, and a name. Wrong ARIA is worse than none: it makes AT lie to users. First rule of ARIA: don't use it when HTML suffices. Second: `aria-*` changes what's *announced*, never what's *operable*.
3. **The accessibility tree is the real UI for AT users.** Browsers compute, per node: **role** (from element or ARIA), **name** (accessible-name computation: `aria-labelledby` > `aria-label` > native label/alt/caption > content > `title`), **value/state** (checked, expanded, invalid...). Debug a11y by inspecting this tree (DevTools Accessibility panel), the same way you debug layout with the box model.
4. **Keyboard is the contract; screen readers ride on it.** If every interactive element is reachable in a sensible order, operable with Enter/Space/arrows per convention, and focus never vanishes, you've satisfied most of WCAG's operable half — and most switch, voice, and power users too.
5. **Conformance target (as of 2026): WCAG 2.2 Level AA.** It's the ISO-standard (ISO/IEC 40500:2025) benchmark cited by the EU's EAA (enforceable since June 2025, via EN 301 549 updates), ADA title II rules, and Section 508. WCAG 3.0 is a Working Draft — nothing to conform to yet; don't cite it as a requirement (Candidate Recommendation projected 2027+).

## Decision frameworks

**"Do I need ARIA here?"** — the chain:
1. Is there a native element with this semantics? → Use it, done. (Toggle → `<button aria-pressed>` or checkbox; disclosure → `<details>` or button+`aria-expanded`; modal → `<dialog>`; popup menu attached to a button → often a `popover` or a styled `<select>`, not a `menu` role.)
2. Is this genuinely one of the composite patterns HTML can't express? Legitimate list: **combobox/autocomplete** (`role="combobox"` + `aria-expanded` + `aria-controls` + `aria-activedescendant`), **live regions** (`aria-live`/`role="status"`/`role="alert"` for content that changes without focus moving: toasts, async results counts, chat), **tabs**, **grid/treegrid**, **menu (application-style, not nav)**, **slider (when `<input type=range>` can't work)**, **dialog (when `<dialog>` can't be used)**. Implement from the ARIA Authoring Practices Guide (APG) pattern, keyboard behavior included — or better, use a maintained headless library (React Aria, Radix, Base UI) that has already fixed the AT quirks.
3. Otherwise you're decorating: at most `aria-label` for name, `aria-describedby` for hints/errors, `aria-current` for nav state.

**"Anything dynamic changed — who gets told?"** Focus moved to it → nothing needed (focus announces). Focus didn't move → live region, or move focus deliberately. Neither → AT users don't know it happened; that's the bug. (Priors: form submit errors, filter result counts, "added to cart," route changes.)

**Focus management on SPA route changes** (the platform announces full page loads; SPAs must replicate): after navigation, move focus to the new main heading or a main-content wrapper (`tabindex="-1"` on it, then `.focus()`), and update `document.title`. Routers don't do this by default — check the app's router integration. Skip link (`<a href="#main">` first in DOM) remains required either way.

**Focus traps and restoration**: modal dialogs trap Tab within themselves while open, close on Escape, and **restore focus to the invoker on close** (forgetting restoration strands keyboard users at `<body>` — the most common dialog bug). `<dialog>.showModal()` gives you the trap, Escape, top-layer rendering, and `inert`-like backdrop behavior natively — use it (Baseline-stable for years now); add focus restoration if the invoker moved. For non-`<dialog>` overlays, make the rest of the app `inert` (the `inert` attribute is Baseline) instead of maintaining a hand-rolled trap.

**Roving tabindex** for composite widgets (toolbars, tabs, menus, grids): the widget is one Tab stop; arrows move within. Implementation: active item `tabindex="0"`, all others `tabindex="-1"`, arrow handlers move both focus and the 0. The alternative (`aria-activedescendant`, focus stays on the container) suits comboboxes where focus must remain in the input. A toolbar where every button is a Tab stop makes a 30-icon editor cost 30 Tabs to cross — that's the smell that triggers this pattern.

## Screen-reader mental model (enough to predict behavior)

- SR users operate in two modes: browse/virtual mode (reading the accessibility tree linearly, jumping by headings/landmarks/links — heading structure is *navigation*, not styling: one `<h1>`, no skipped levels) and focus/forms mode (interacting with widgets, where the widget's keyboard handling takes over). `role="application"` disables browse mode — almost never what you want.
- Landmarks are the page's SR-level layout: one `<main>`, `<nav>` (labelled if multiple: `aria-label="Breadcrumb"`), `<header>`/`<footer>`, `<aside>`, `<search>`. All content should live inside some landmark; SR users jump between them the way sighted users saccade.
- Name computation gotchas that produce real bugs:
  - `aria-label` on a `<div>` with no role does nothing reliable — names need roles.
  - `aria-label` *overrides* inner text: `<button aria-label="Close">Save</button>` announces "Close" — and fails WCAG 2.5.3 Label in Name for voice-control users who say "click Save". If there's visible text, the accessible name must contain it.
  - Placeholder is not a label (disappears on input, low contrast, not reliably announced).
  - An icon button with no name announces "button" — the void. A linked logo image with empty alt announces the raw URL.
  - `aria-labelledby` can compose names from multiple ids (`aria-labelledby="row-title col-title"`) — often better than duplicating text into `aria-label` that then drifts from the visible copy.
- `display: none`/`visibility: hidden`/`hidden` remove from the tree; `aria-hidden="true"` removes from the tree while staying visible (never put focusable elements inside it — "ghost" tab stops that announce nothing); visually-hidden CSS (clip-path/1px pattern, e.g. Tailwind `sr-only`) keeps it in the tree while invisible — for SR-only text.
- `aria-live="polite"` (waits for silence) vs `role="alert"`/`assertive` (interrupts — reserve for urgent errors). Live regions must exist in the DOM *before* content changes; injecting a node with `role="alert"` already populated is unreliable across AT — render the empty container upfront, then set its text.

## Contrast, motion, and visual requirements (AA numbers)

- Text contrast ≥ **4.5:1** (≥ 3:1 for large text: 24px, or 18.7px bold). Non-text UI (input borders, icons conveying info, focus indicators) ≥ **3:1** (WCAG 1.4.11). Placeholder-gray-on-white at 2.8:1 is the perennial failure. Don't convey state by color alone (add icon/text/underline).
- WCAG 2.2 additions to know: focus indicators must not be fully obscured (2.4.11), pointer targets ≥ 24×24 CSS px or spaced (2.5.8), no drag-only operations (2.5.7), no cognitive-test logins (3.3.8).
- `prefers-reduced-motion`: wrap nonessential animation in `@media (prefers-reduced-motion: no-preference)` (opt-*in* to motion — safer than opting out); in JS check `matchMedia('(prefers-reduced-motion: reduce)')` before smooth-scroll/parallax/autoplay. Reduced ≠ none: crossfades may replace slides.
- Respect zoom: layout must work at 400% zoom / 320px width (1.4.10 Reflow); text spacing overrides must not break content (1.4.12) — both fail on fixed-height containers with `overflow: hidden`.

## Testing strategy — the layered reality

- **Automated (axe-core via @axe-core/playwright, jest-axe, Lighthouse)** catches roughly 30–40% of issues by volume — the mechanical ones: contrast, missing names/alt, invalid ARIA, missing labels. Run it in CI as a *floor*; a clean axe run proves little.
- **Manual keyboard pass** (highest value per minute): unplug the mouse; Tab through every flow — everything reachable? visible focus at all times? logical order? Escape/arrows per convention? no traps? focus restored after overlays? This catches the operable half axe can't.
- **Screen reader smoke test** on real flows: VoiceOver+Safari (macOS/iOS), NVDA+Chrome or Firefox (Windows — NVDA is free), TalkBack (Android). Test the *pairs* users actually use; a pattern working in VoiceOver can fail in NVDA (particularly `aria-activedescendant` and live regions). You don't need fluency — need: does the widget announce name/role/state, can you operate it, do updates get spoken?
- **Structural checks**: DevTools accessibility tree inspection per new component; headings/landmarks audit (e.g., Accessibility Insights "FastPass").
- Priors for review: the top real-world failures are low contrast, missing image alt, missing link/button names, missing input labels, empty links (WebAIM Million's stable top-5 for years). Check these first — they're cheap and cover most user harm.

## How an expert thinks through this

*Scenario: "Add an autocomplete city picker to the checkout form."*

First instinct: can HTML do it? `<datalist>` — considered, rejected: styling is locked down, AT behavior inconsistent across pairs, and we need async results and grouped options. So this is a genuine combobox — the hardest ARIA pattern; do not hand-roll if a vetted primitive exists. The project uses React → React Aria's `ComboBox` or a Radix-based one; they handle `aria-activedescendant`, `aria-expanded`, listbox option roles, type-ahead, and the NVDA/VoiceOver quirks. Decision: wrap the design system around React Aria rather than build. If forced to hand-roll (no-framework constraint), follow the APG combobox pattern exactly: input `role="combobox" aria-expanded aria-controls`, popup `role="listbox"`, options `role="option" aria-selected`, arrows move `aria-activedescendant` while DOM focus stays in the input (rejected roving tabindex here — moving DOM focus out of the input breaks typing), Enter selects, Escape closes then clears. Async results: announce the count — a visually-hidden `role="status"` node rendered upfront, text set to "12 results" after fetch settles (rejected `assertive`: result counts aren't urgent; rejected announcing every keystroke: debounce to settled results). Error state: `aria-invalid="true"` + error text linked via `aria-describedby` — not a toast (toasts vanish and aren't associated with the field). Verify: keyboard-only run, then NVDA and VoiceOver: name ("City"), role (combobox), state (expanded), options announced as "Lisbon, 3 of 12". Stopping rule: axe clean, keyboard flow complete, both SR pairs operable — ship; exotic AT matrix testing beyond that is not the marginal-value move for a checkout field.

## Failure modes & pitfalls

- **`<div onClick>`**: needs `role="button"`, `tabindex="0"`, `onKeyDown` for Enter *and* Space (Space must also prevent page scroll), plus a name — or just use `<button>`. Same for `<a>` without `href` (not focusable, no link role).
- **`<button>` vs `<a>` swapped**: navigation → `<a href>` (announces "link", supports open-in-new-tab); action → `<button type="button">` (and inside forms, an unset `type` defaults to `submit` — the classic "clicking any button submits the form" bug).
- **Icon buttons without names**: `<button><svg…/></button>` announces "button". Fix: `aria-label` on the button + `aria-hidden="true" focusable="false"` on the svg.
- **`aria-label` sprinkled on non-interactive `<div>`/`<span>`** — ignored or inconsistently announced; names require roles.
- **Redundant/contradictory ARIA**: `role="button"` on `<button>` (harmless noise), `role="menu"` on site navigation (harmful — promises application-menu keyboard behavior that isn't there; nav links are `<nav><ul><a>`), `aria-selected` on things that aren't in a selection widget.
- **Disabled-button error hiding**: submit disabled until valid gives no feedback path; prefer enabled submit + validation messages, `aria-describedby`-linked, focus moved to the first error (or an error-summary `role="alert"`).
- **Focus outline removed** (`:focus { outline: none }`) with no replacement — WCAG 2.4.7 failure and the most common keyboard complaint. Use `:focus-visible` for keyboard-only rings; ensure 3:1 contrast for the indicator.
- **Positive `tabindex`** (`tabindex="3"`) — hijacks document order globally; only ever use `0` and `-1`.
- **Modal without inert background**: focus escapes into the page behind; or focus not restored on close. Use `<dialog>.showModal()`; if custom, `inert` the siblings.
- **Live region injected with content** (render-then-populate required); **overusing `assertive`** (interrupts mid-sentence); toast-only feedback with 3s auto-dismiss (fails 2.2.1-adjacent expectations; make toasts persistent or duplicated in-page).
- **`aria-hidden="true"` on a wrapper containing focusable elements** — Tab lands on invisible-to-AT controls.
- **Custom select rebuilt as div-soup** when `<select>` (now stylable with `appearance: base-select` in Chromium — verify current cross-browser status) or a headless listbox would do.
- **Alt text**: decorative images need `alt=""` (omitting `alt` entirely makes SRs read the filename); meaningful images need content, not "image of".
- **Auto-playing carousels/motion** with no pause control (2.2.2) and ignoring `prefers-reduced-motion`.
- **Headings chosen by font size**: `<div class="h2">` styled as a heading gives SR users nothing to navigate by; conversely `<h4>` used "because it's smaller" breaks the outline. Pick the level by structure, restyle with CSS.
- **`title` attribute as the only label/tooltip** — not reliably announced, invisible to touch and keyboard. Use visible text, `aria-label`, or a real tooltip pattern (focus/hover-triggered, `aria-describedby`-linked, dismissible per WCAG 1.4.13).
- **Radio/checkbox groups without a group name**: individual labels ("Yes", "No") announce without the question. Wrap in `<fieldset><legend>Question?</legend>…</fieldset>` (or `role="group"` + `aria-labelledby`).
- **Data tables without header associations**: use `<th scope="col|row">` (or `headers`/`id` for complex tables) and `<caption>`; a grid of `<div>`s styled as a table announces as word soup. Never use `role="presentation"` on a real data table.
- **`autofocus` on page load** (skips context for SR users, scrolls unexpectedly) — acceptable only inside just-opened dialogs where focus must move anyway.
- **`user-scalable=no` / `maximum-scale=1`** in the viewport meta — disables pinch-zoom, fails 1.4.4, and iOS ignores it anyway; just remove it.
- **Infinite scroll with no way past it**: keyboard users Tab through 500 loaded items to reach the footer. Provide a skip link past the feed, a "load more" button instead of auto-load, or both.
- **Ellipsis/truncation-only affordances**: hover-revealed full text is unreachable on touch and keyboard; ensure the full value is available in-flow, on focus, or via an accessible tooltip.

## Worked micro-examples

**1. Accessible async feedback + field error, React:**

```tsx
function SaveSettings() {
  const [status, setStatus] = useState('');       // live region text
  const [error, setError] = useState<string | null>(null);
  return (
    <form onSubmit={onSubmit} noValidate>
      <label htmlFor="email">Email</label>
      <input id="email" name="email" type="email"
             aria-invalid={!!error} aria-describedby={error ? 'email-err' : undefined} />
      {error && <p id="email-err">{error}</p>}
      <button type="submit">Save</button>
      {/* rendered up-front and empty; text set after async settles */}
      <p role="status" className="sr-only">{status}</p>
    </form>
  );
}
// onSubmit: on success setStatus('Settings saved'); on failure set field error
// AND move focus: document.getElementById('email')?.focus()
```

**2. SPA route-change focus (framework-agnostic):**

```ts
router.afterEach(() => {
  document.title = pageTitle;
  const main = document.getElementById('main'); // <main id="main" tabindex="-1">
  main?.focus({ preventScroll: false });
});
```

**3. Roving tabindex for a toolbar (one Tab stop, arrows within):**

```ts
const toolbar = document.querySelector('[role="toolbar"]')!;
const items = [...toolbar.querySelectorAll<HTMLButtonElement>('button')];
let active = 0;
const setActive = (next: number) => {
  items[active].tabIndex = -1;
  active = (next + items.length) % items.length;   // wrap
  items[active].tabIndex = 0;
  items[active].focus();
};
items.forEach((el, i) => { el.tabIndex = i === 0 ? 0 : -1; });
toolbar.addEventListener('keydown', (e) => {
  if (e.key === 'ArrowRight') { e.preventDefault(); setActive(active + 1); }
  if (e.key === 'ArrowLeft')  { e.preventDefault(); setActive(active - 1); }
  if (e.key === 'Home')       { e.preventDefault(); setActive(0); }
  if (e.key === 'End')        { e.preventDefault(); setActive(items.length - 1); }
});
// Also update `active` on click/focus so mouse use doesn't desync the roving 0.
```

**4. Icon button + native dialog with focus restoration:**

```tsx
function DeleteButton({ onConfirm }: { onConfirm: () => void }) {
  const dialogRef = useRef<HTMLDialogElement>(null);
  const openerRef = useRef<HTMLButtonElement>(null);
  return (
    <>
      <button ref={openerRef} type="button" aria-label="Delete item"
              onClick={() => dialogRef.current?.showModal()}>
        <TrashIcon aria-hidden="true" focusable="false" />
      </button>
      {/* showModal(): top layer, focus trap, Escape-to-close, ::backdrop — all native */}
      <dialog ref={dialogRef} aria-labelledby="del-title"
              onClose={() => openerRef.current?.focus()}>  {/* restore focus to invoker */}
        <h2 id="del-title">Delete this item?</h2>
        <p>This cannot be undone.</p>
        <button type="button" onClick={() => dialogRef.current?.close()}>Cancel</button>
        <button type="button" onClick={() => { onConfirm(); dialogRef.current?.close(); }}>
          Delete
        </button>
      </dialog>
    </>
  );
}
```

## Verification / self-check

Before declaring UI accessible:
1. axe run clean (CI) — necessary, nowhere near sufficient.
2. Keyboard-only pass of the changed flows: reachable, visible focus, conventional keys, no traps, restoration after overlays.
3. Accessibility-tree spot check: every interactive element has correct role + non-empty name + state.
4. One SR pass (VoiceOver or NVDA) on the primary flow; live regions actually announce.
5. Contrast of new colors checked (4.5:1 text / 3:1 UI), motion gated on `prefers-reduced-motion`, layout at 320px/400% zoom.
Stopping rule: WCAG 2.2 AA is the bar — cite specific success criteria for disputed items rather than vibes; beyond AA, invest in real-user testing with disabled users, not AAA checkbox-chasing.
