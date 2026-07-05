---
name: web-accessibility
description: Load when building or reviewing UI for accessibility — semantic HTML choices, ARIA usage, keyboard navigation and focus management (SPA routes, dialogs, menus), screen-reader behavior, color contrast, motion preferences, a11y testing strategy, or WCAG conformance questions.
---

# Web Accessibility

## Core mental model (anchors)

1. Semantics are 90% of the work; the `<div onclick>` is four bugs in one element (no role, name, keyboard activation, focusability).
2. ARIA is a promise, not an implementation — it changes what's *announced*, never what's *operable*; wrong ARIA makes AT lie to users.
3. Debug via the accessibility tree (role, name, state) like you debug layout via the box model.
4. Keyboard is the contract: everything reachable, conventional keys, focus never vanishes — that's most of WCAG's operable half plus switch/voice users.

## Conformance target and law (verified July 2026 — the currency block)

- Target **WCAG 2.2 AA**. Post-cutoff fact cold answers miss: **WCAG 2.2 is now the ISO standard — ISO/IEC 40500:2025** (cold answers cite the 2012 edition, which was WCAG 2.0).
- EU: EAA enforceable since June 2025, via EN 301 549 (being updated toward 2.2; 2.1 AA is the ratified floor). US: **ADA Title II formally requires WCAG 2.1 AA** — compliance deadlines April 2026 (≥50k population) / April 2027 (smaller); Section 508 still cites 2.0 AA. Building to 2.2 AA satisfies every current legal reference.
- WCAG 3.0: Working Draft, Candidate Recommendation projected 2027+ — never cite it as a requirement.

## Decision frameworks (compressed — cold answers reproduce the details)

- **ARIA chain**: native element exists → use it. Genuinely composite (combobox, live regions, tabs, grid, application menus, slider, dialog-when-`<dialog>`-can't) → APG pattern with full keyboard behavior, or better a maintained headless library (React Aria, Radix, Base UI) that already fixed the AT quirks. Otherwise you're decorating: `aria-label`/`aria-describedby`/`aria-current` at most.
- **Anything dynamic changed — who gets told?** Focus moved to it → done. Didn't move → live region or move focus deliberately. Neither → AT users don't know; that's the bug. (Priors: submit errors, filter counts, "added to cart," route changes.)
- **SPA route change**: focus the new main heading/wrapper (`tabindex="-1"` + `.focus()`), update `document.title`. Routers don't do this by default. Skip link stays required.
- **Overlays**: `<dialog>.showModal()` for trap/Escape/top-layer natively; restore focus to the invoker on close (the most common dialog bug); non-`<dialog>` overlays `inert` the rest, never hand-rolled traps.
- **Roving tabindex** for toolbars/tabs/menus/grids (one Tab stop, arrows within); `aria-activedescendant` when DOM focus must stay in an input (comboboxes).
- Don't hand-roll comboboxes when a vetted primitive exists — it's the hardest pattern and `aria-activedescendant`/live-region behavior differs across NVDA/VoiceOver; async result counts go to a pre-rendered `role="status"` node, debounced to settled results, never per-keystroke.

## Numbers (AA)

Text 4.5:1 (3:1 for ≥24px or 18.7px bold); non-text UI 3:1 (1.4.11). WCAG 2.2 additions: focus not fully obscured (2.4.11), targets ≥24×24 px or spaced (2.5.8), no drag-only (2.5.7), no cognitive-test logins (3.3.8). Reflow at 320px/400% zoom (1.4.10); text-spacing overrides must not break content (1.4.12) — both fail on fixed-height + `overflow: hidden`.

## Testing reality

Automated (axe in CI) catches ~30–40% by volume — a floor, not a proof. Highest value per minute: the keyboard-only pass. SR smoke tests on real *pairs* (VoiceOver+Safari, NVDA+Chrome/Firefox — live regions and `aria-activedescendant` are the pair-divergent features). Review priors: WebAIM Million's stable top-5 — low contrast, missing alt, missing link/button names, missing input labels, empty links — check these first; they're cheap and cover most user harm.

## Pitfalls checklist (one-liners — mechanisms are cold-recoverable)

`aria-label` overrides inner text and breaks voice control (2.5.3: visible text must be in the accessible name); names need roles — `aria-label` on a bare `<div>` does nothing; icon buttons: `aria-label` on button + `aria-hidden focusable="false"` on svg; placeholder is not a label; live regions render-empty-then-populate (injecting pre-populated `role="alert"` is unreliable); `assertive` only for urgent; toasts persist or duplicate in-page; `aria-hidden` wrappers must contain no focusables (ghost tab stops); positive `tabindex` never; `:focus-visible` rings, 3:1 indicator contrast, never bare `outline: none`; unset `<button>` type defaults to `submit`; nav is `<nav><ul><a>` — `role="menu"` promises application keyboard behavior; `role="application"` disables browse mode, almost never wanted; disabled-until-valid submits hide the feedback path — enabled submit + focus-the-first-error (or `role="alert"` summary); `fieldset`/`legend` for radio/checkbox groups; `<th scope>` + `<caption>` for data tables; decorative `alt=""` (omitted alt reads the filename); headings by structure not size; `title` attribute is not a label; `user-scalable=no` fails 1.4.4 and iOS ignores it — remove; infinite scroll needs a skip path; hover-only truncation reveals are unreachable on touch/keyboard; `autofocus` only inside just-opened dialogs; motion behind `prefers-reduced-motion: no-preference` (opt-*in* to motion), reduced ≠ none.

Support-currency note: stylable `<select>` (`appearance: base-select`) — verify cross-browser status before recommending over a headless listbox.

## Micro-example: the one pattern worth keeping verbatim

Async feedback + field error (render-empty live region; focus moves to the error):

```tsx
<input id="email" aria-invalid={!!error} aria-describedby={error ? 'email-err' : undefined} />
{error && <p id="email-err">{error}</p>}
<p role="status" className="sr-only">{status}</p>  {/* rendered up-front, empty; set text after async settles */}
// on failure: set error AND document.getElementById('email')?.focus()
```

## Verification / self-check

1. axe clean (necessary, nowhere near sufficient) → 2. keyboard-only pass of changed flows → 3. accessibility-tree spot check (role + non-empty name + state per interactive element) → 4. one SR pass on the primary flow; live regions actually announce → 5. contrast/motion/320px checks.
Cite specific success criteria for disputed items. Beyond AA, invest in testing with disabled users, not AAA checkbox-chasing.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)

- Probed 14 claims: 12 baseline (cut/compressed), 2 partial (sharpened), 0 hard delta beyond standards currency.
- Opus cold nails: Label in Name, WCAG 2.2 SC numbers, dialog focus restoration, live-region injection, roving tabindex vs activedescendant, WebAIM top-5, contrast ratios, combobox pattern. Restructured to a correction sheet.
- Remaining value: ISO/IEC 40500:2025 currency (cold cites the 2012/2.0 edition), precise EAA/ADA regulatory mapping (this pass also corrected the skill: ADA Title II legally requires 2.1 AA, not 2.2), WCAG 3.0 CR ~2027+ framing, and the compressed review priors.
