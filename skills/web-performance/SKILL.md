---
name: web-performance
description: Load when optimizing web page speed or Core Web Vitals (LCP, INP, CLS), debugging slow loads or laggy interactions, deciding bundle/code-splitting strategy, image/font loading, caching headers, or setting up performance measurement and budgets.
---

# Web Performance

## Core mental model (anchors)

1. Optimize field p75 CWV: LCP ≤ 2.5s, INP ≤ 200ms, CLS ≤ 0.1 (thresholds unchanged as of 2026 — ignore SEO-blog claims otherwise). Lab is diagnosis; CrUX is the verdict; INP work is RUM-mandatory because lab barely has interactions.
2. Most LCP problems are *discovery* problems in the dependency waterfall, not bandwidth problems — each removed arrow beats compressing any node.
3. JS is paid four times: download, parse, execute, then forever as hydration/re-render. Main-thread blocking is INP's currency; bundle size is a proxy.
4. The fastest resource is one already decided: reserve space, cache immutably, precompute on the server. Most wins are removals and reorderings.
5. Third parties are unbounded liabilities; a tag manager is an arbitrary-code-execution platform for the marketing team.

## Decision chains (kept as checklists; cold answers reproduce the mechanisms)

**LCP by sub-part**: TTFB >800ms → server/edge, no frontend trick beats it; load *delay* dominant (most common) → LCP image in initial HTML as `<img fetchpriority="high">`, never CSS-background/client-rendered/lazy; load *time* → AVIF/WebP + `srcset`/`sizes` (wrong `sizes` silently downloads desktop images — the browser picks before layout); render delay → blocking CSS/JS, SSR the hero.

**INP by sub-part**: input delay (long tasks — `scheduler.yield()` + `setTimeout` fallback, chunk ~every 50ms); processing (minimum sync work, then paint); presentation (too much DOM per interaction — virtualize; layout thrash). Framework-app offenders in tractability order: hydration bursts, giant list re-renders, third parties.

**CLS**: unreserved images/embeds (`width`/`height` or `aspect-ratio`), font swap, late banners (reserve `min-height`), layout-property animation. Session-windowed across page lifetime — an accordion on scroll can fail a "stable" load. Skeletons sized differently from real content convert a perceived-speed win into CLS.

**Caching**: hashed assets `public, max-age=31536000, immutable`; HTML `no-cache` (never `no-store` — kills bfcache; never long-cache un-hashed HTML); SSG/ISR `s-maxage` + `stale-while-revalidate` at CDN; personalized `private`. Service worker only for offline/installability — a mis-scoped cache-first SW serves year-old bundles.

**Fonts**: self-host WOFF2, `font-display: swap`/`optional`, preload needs `crossorigin` even same-origin (else double-download), metric-matched fallback via `size-adjust`/`ascent-override` (fontaine, `next/font`). >2 families×weights → variable font or fewer.

**Third parties, escalation ladder**: remove (audit quarterly — zombie tags are real) → facade (lite-youtube-embed pattern) → defer to interaction → Partytown → at minimum async + preconnect + a named owner per tag. Anti-flicker A/B snippets set your LCP floor to their timeout (2–4s) — push for server/edge experimentation.

## Sharpened numbers and judgment

- Splitting below ~10–20KB chunks adds request overhead for nothing; split by route, then by interaction (`import()` on hover/click).
- Tree-shaking failures: barrel files, missing `"sideEffects": false`, CJS deps, top-level side effects — verify with an analyzer, never assert shaking worked without looking. Selective dependency replacement (moment→`Intl`, lodash→es-toolkit) is often the single biggest bundle win.
- Preload is a scalpel: 1–2 resources, each verified to improve the waterfall, else removed. Five preloads demote each other and the HTML-discovered critical path.
- Lazy-load policy: never the LCP image; *near*-fold images eager too (IntersectionObserver gating hurts perceived speed); offscreen iframes lazy. Cap DPR at 2x — 3x is invisible-difference bandwidth.
- Budgets only exist as CI gates (Lighthouse CI assertions, `size-limit` per entry): ~100–200KB gz JS/route, ≤50KB critical CSS, ≤2 font files, LCP resource ≤150KB, third-party blocking ≈ 0; any PR adding >10KB to a route states a reason. Unenforced budgets decay in one quarter.
- Field-data lag: tell stakeholders upfront that CrUX confirmation takes up to ~28 days after the fix ships.

## Pitfalls checklist (one-liners)

bfcache breakers (`unload` handlers — use `pagehide`; `no-store` HTML) silently double real page loads — check `notRestoredReasons` in RUM; CSS `@import` chains invisible to the preload scanner — bundle them; `document.write` ad tags — refuse or sandbox; measuring dev builds overstates JS cost multiples — profile production with source maps and honest throttling (4x CPU, slow 4G); giant DOM — `content-visibility: auto` + `contain-intrinsic-size` for long offscreen sections; compression audit (Brotli for text at the CDN, zstd increasingly supported; never recompress images/woff2) — five minutes, regularly finds uncompressed JSON APIs; `decoding="async"` is cargo cult; Speculation Rules (`"eagerness": "moderate"`) is Chromium-only progressive enhancement; the `load` event is not a UX moment — report CWV to stakeholders.

## Worked shape: RUM with attribution (the part teams skip)

```js
import { onLCP, onINP, onCLS } from 'web-vitals/attribution';
const send = (m) => navigator.sendBeacon('/rum', JSON.stringify({
  name: m.name, value: m.value, rating: m.rating,
  target: m.attribution?.interactionTarget ?? m.attribution?.element,  // WHICH element/interaction
  page: location.pathname,
}));
onLCP(send); onINP(send); onCLS(send);
```
Attribution turns "INP is failing" into "the size-selector click is failing" — without it you're guessing at what to fix.

## Verification / self-check

- Every recommendation names the metric AND sub-part it improves ("LCP load delay") — can't name it, you're guessing.
- Validated in a throttled lab trace + explicit plan (and stated lag) for field confirmation.
- Check you didn't trade metrics: lazy-loading that pushed LCP; skeletons that added CLS; SW caching that strands deploys.
- Stop at p75 green with margin and the top RUM-attributed element addressed; chasing lab 100s beyond that is decoration.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)

- Probed 14 claims: 14 baseline, 0 partial, 0 delta — the most baseline-saturated skill in the batch; restructured to a compact checklist per the ≥80% rule.
- Opus cold nails everything probed: CWV thresholds/INP-replaced-FID dates, four-part LCP decomposition, lazy-hero mistake, bfcache breakers + notRestoredReasons, scheduler.yield, font crossorigin + metric overrides, preload discipline, tree-shaking taxonomy, cache headers, third-party ladder + anti-flicker harm, Speculation Rules, content-visibility, 28-day lag, CI-gated budgets.
- Remaining value: compactness, the "name the sub-part" discipline, trade-check list, and a few judgment thresholds (10–20KB chunk floor, DPR 2x cap, >10KB-per-PR rule).
