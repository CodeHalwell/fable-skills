---
name: web-performance
description: Load when optimizing web page speed or Core Web Vitals (LCP, INP, CLS), debugging slow loads or laggy interactions, deciding bundle/code-splitting strategy, image/font loading, caching headers, or setting up performance measurement and budgets.
---

# Web Performance

## Core mental model

1. **Optimize the user-centric metrics, in field data, at p75.** Core Web Vitals (as of 2026, unchanged canonical thresholds): **LCP ≤ 2.5s** (loading), **INP ≤ 200ms** (interactivity; replaced FID in March 2024), **CLS ≤ 0.1** (stability) — each at the 75th percentile of real users. Lab scores (Lighthouse) are a debugging tool, not the target; a Lighthouse 100 with failing CrUX is still failing. Ignore SEO-blog claims of changed thresholds unless web.dev/Chrome announces them.
2. **The load is a dependency waterfall.** Every resource has a discovery time (when the browser learns it exists) and a priority. Most LCP problems are *discovery* problems (image behind CSS background / JS render / client-side fetch chain), not bandwidth problems. Draw the chain: HTML → CSS/JS → (fetch → JSON) → image. Each arrow you remove is usually worth more than compressing any node.
3. **JavaScript's cost is paid three times**: download, parse/compile, execute — and then a fourth, forever: every byte of framework component code is re-executed as hydration and re-render cost. Bundle size is a proxy; main-thread blocking time is the real currency of INP.
4. **The fastest resource is one already decided.** Reserve space (no layout shift), preload what's critical, cache immutably, and precompute (SSR/SSG) what doesn't need the client. Most wins are *removals and reorderings*, not clever code.
5. **Third parties are unbounded liabilities.** A tag manager is an arbitrary-code-execution platform for the marketing team. Contain, defer, or facade them — you cannot optimize what you don't control.

## The metric → cause → fix decision chains

**LCP** (usually the hero image or heading). Decompose into its four parts (TTFB → resource load delay → resource load time → render delay); the fix differs per part:
- TTFB slow (>800ms) → server/edge problem: cache the HTML, move rendering closer (edge/ISR), fix backend. No frontend trick beats a slow first byte.
- Load *delay* dominant (most common) → discovery problem: LCP image must be in the initial HTML as `<img>` (not CSS background, not client-rendered), with `fetchpriority="high"`; add `<link rel="preload" as="image">` only if it can't be in early HTML. Never lazy-load the LCP image — `loading="lazy"` on it is the single most common self-inflicted LCP wound.
- Load *time* dominant → size/format problem: AVIF/WebP, correct `srcset` sizing, CDN.
- Render delay → blocking CSS/JS or client-side rendering: inline critical CSS, remove render-blocking scripts, SSR the hero.

**INP** (worst interaction latency, full page lifetime). Decompose: input delay → processing → presentation delay.
- Input delay → main thread busy when the user clicked: long tasks from hydration, third parties, or rendering. Break up long tasks (`scheduler.yield()` in Chromium with a `setTimeout` fallback, or `await`-chunked loops); code-split so less runs at all.
- Processing → your handler is slow: do the minimum synchronously, paint, then continue (update state, `startTransition` for non-urgent React updates, move pure computation to a worker).
- Presentation delay → the re-render after the handler is huge: too much DOM updated per interaction (virtualize lists, narrow re-renders — see `frontend-architecture`), or synchronous layout thrash (interleaved reads/writes of `offsetHeight` etc.).
- Prior: on framework apps the top INP offenders are hydration bursts, giant list re-renders, and third-party scripts — in that order of tractability.

**CLS** — almost always one of four: (1) images/embeds without reserved space → set `width`/`height` attributes or `aspect-ratio`; (2) web font swap reflow → see fonts below; (3) late-injected banners/ads → reserve slots with fixed `min-height`; (4) animating layout properties → animate only `transform`/`opacity`. Note CLS counts shifts across the whole lifetime, session-windowed — a "stable load" can still fail from an accordion pushing content on scroll.

## The waterfall toolbox (what actually reorders loading)

- `<link rel="preload">` — "you'll need this soon but won't discover it": fonts referenced in CSS, LCP background image. Misuse is common: preloading things the preload-scanner already finds just steals bandwidth; every preload competes with HTML-discovered resources. Rule: add one, verify the waterfall improved, else remove.
- `fetchpriority="high|low"` — reprioritize within discovered resources: high on the LCP image, low on carousel images 2..n.
- `rel=preconnect` — only for origins critical in the first seconds (font CDN, image CDN); each costs a socket.
- `rel=prefetch` / Speculation Rules API (Chromium) — next-navigation warmup; `<script type="speculationrules">` with `"eagerness": "moderate"` prefetches/prerenders links on hover/pointerdown. Verify current cross-browser status before relying on it beyond Chromium; treat as progressive enhancement.
- `defer` for all classic scripts (ordered, after parse), `async` only for truly independent ones (analytics); module scripts defer by default. A synchronous `<script src>` in `<head>` blocks parsing — this is still found in the wild, still the first thing to check.
- CSS blocks *rendering*, not parsing; JS after CSS blocks on the CSS (it might read styles). Keep head CSS lean; split non-critical CSS via `media="print"` tricks only with measurement, prefer just shipping less.

## JS cost strategy

- **Split by route first** (framework default), then by *interaction* (modal, editor, chart loaded on intent — `import()` on hover/click). Splitting below ~10–20KB chunks adds request overhead for nothing.
- **Tree-shaking failure taxonomy**: barrel files (`index.ts` re-exporting 200 modules pulls a large graph into every importer — import from the concrete module or use `optimizeDeps`/`modularizeImports`-style config); packages without `"sideEffects": false`; CommonJS deps (opaque to the bundler — prefer ESM builds); accidental top-level side effects (`const x = registerThing()`). Verify with a bundle analyzer (`rollup-plugin-visualizer`, `next build --analyze` variants, or `bundlephobia`/`pkg-size` per-dep) — never assert shaking worked without looking.
- **Hydration cost is real**: SSR HTML is fast to *show* and slow to *awaken*; a huge hydrating page produces long tasks that eat INP right when users first interact. Mitigations, in preference order: ship less JS at all (Server Components — zero client JS for non-interactive parts), islands (Astro's model: hydrate only interactive widgets, `client:visible` to defer offscreen ones), progressive/lazy hydration, `<Activity>`/deferred rendering for hidden UI.
- Selective replacement: heavy dependency on a light page (moment→`Intl`/date-fns, lodash→es-toolkit or native, charting lib → server-rendered SVG) is often the single biggest bundle win. Check the analyzer before assuming.

## Images

Decision chain: (1) format — AVIF first, WebP fallback, via `<picture>` or an image CDN that content-negotiates (universally supported as of 2026); SVG for icons/diagrams; (2) responsive sizing — `srcset` + `sizes`, where wrong `sizes` silently downloads desktop images on mobile (audit with DevTools' actual-vs-intrinsic size); (3) loading policy — LCP/above-fold: eager + `fetchpriority="high"`; below-fold: `loading="lazy"`; *near*-fold: eager (lazy-loading just-below-fold images delays them behind IntersectionObserver and hurts perceived speed); (4) always `width`/`height` or `aspect-ratio` for CLS; (5) `decoding="async"` is default-adjacent, don't cargo-cult it.

## Fonts

- Self-host WOFF2 with `font-display: swap` (or `optional` for truly cosmetic fonts — no swap-shift at all), `<link rel="preload" as="font" type="font/woff2" crossorigin>` for the 1–2 critical faces (crossorigin required even same-origin — fonts are CORS-fetched; omitting it double-downloads).
- Kill swap-CLS by metric-matching the fallback: `@font-face { font-family: "Inter-fallback"; src: local("Arial"); size-adjust: 107%; ascent-override: 90%; ... }` — tools like `fontaine` or Next.js `next/font` compute these automatically (and `next/font` self-hosts Google fonts, removing the third-party hop).
- Subset aggressively (unicode-range, variable font with only needed axes). Four weights of a family is a smell; variable font or fewer weights.

## Caching per asset class

- **Hashed immutable assets** (JS/CSS/images with content-hash filenames): `Cache-Control: public, max-age=31536000, immutable`.
- **HTML**: `no-cache` (revalidate every time; ETag) or short `s-maxage` + `stale-while-revalidate` at the CDN for SSG/ISR. Never long-cache un-hashed HTML — you'll strand users on old asset references.
- **API responses**: per-endpoint; `stale-while-revalidate` where eventual freshness is fine; `private` for personalized.
- **Fonts**: immutable (hashed or versioned path).
- Service worker: powerful, but a mis-scoped cache-first SW is how sites serve year-old bundles; use Workbox recipes, version caches, and have a kill switch. Don't add one for "performance" alone — add it for offline/installability.

## Measuring correctly

- **Field (RUM)**: CrUX (BigQuery / CrUX API / PageSpeed Insights "field" section) for Chrome-population p75; your own RUM (`web-vitals` library posting to analytics) for all browsers, per-page, per-segment, with attribution (`onINP(({attribution}) => ...)` tells you *which element*). Field data is the verdict.
- **Lab**: Lighthouse/DevTools traces to reproduce and diagnose what field data flagged. Throttle honestly (mobile CPU 4x, slow 4G) — your M-series laptop is a lie machine.
- The classic mismatch: lab LCP fine, field LCP bad → real users hit cold caches, slower devices, longer RTTs, or a different page state (logged-in, ads). Lab INP barely exists — INP needs real interactions, so RUM is mandatory for INP work.
- **Performance budget discipline**: budgets only work as CI gates, not aspirations — e.g., Lighthouse CI assertions or `size-limit` per-entry-point (fail PRs over e.g. 200KB gz route JS). Budget the *deltas* conversation: any PR adding >10KB to a route needs a stated reason. Without enforcement, budgets decay in one quarter.

## Third-party containment

Priors: tag managers, chat widgets, A/B testing scripts, and session replay are the usual INP/LCP top offenders. Strategy ladder: (1) remove — audit quarterly, most orgs run zombie tags; (2) facade — load a static placeholder, real widget on interaction (`react-live-chat-loader` pattern for chat, lite-youtube-embed for video); (3) defer — load after `load` event or first interaction; (4) isolate — Partytown (runs scripts in a worker) where compatible; (5) at minimum `async` + `preconnect` and a documented owner per tag. A/B scripts that anti-flicker-hide the page cap your LCP at their timeout — push for server-side experimentation.

## How an expert thinks through this

*Scenario: "PageSpeed says our product page LCP is 4.1s (field). Team wants to buy a faster CDN."*

Before spending: which part of LCP? Pull the p75 breakdown from RUM attribution. TTFB is 600ms — fine, so the CDN pitch is treating the wrong term; park it. Load delay is 2.4s — discovery problem, dominant term. Look at the page: the hero is a client-rendered `<img>` inside a React carousel, so the browser can't see it until the bundle downloads, hydrates, and renders — the image starts at ~3s. Considered `rel=preload` for the image: rejected as primary fix because the URL is computed client-side per variant (can't statically preload the right one) — and even if we could, we'd still pay render delay. Correct fix: SSR the first carousel slide as a plain `<img fetchpriority="high">` in the HTML (later slides stay client-rendered and lazy). Check format while here: 380KB JPEG at 2x the rendered size → image CDN with AVIF + proper `sizes`, saving ~250KB. Considered inlining as data URI: rejected — 30KB+ inline bloats HTML for every visitor including cached ones. Predicted result: load delay → ~200ms, load time halved; verify in lab trace, then wait ~28 days for CrUX to confirm (field data windows lag — tell stakeholders this upfront). Stopping rule: p75 LCP under 2.5s in field with margin; further hero micro-optimization is waste — next dollar goes to INP, which attribution shows failing on the size-selector interaction.

## Worked micro-example

The correct LCP hero, in HTML:

```html
<link rel="preconnect" href="https://img.example-cdn.com">
<picture>
  <source type="image/avif"
          srcset="https://img.example-cdn.com/hero.avif?w=800 800w,
                  https://img.example-cdn.com/hero.avif?w=1600 1600w"
          sizes="(min-width: 64rem) 50vw, 100vw">
  <img src="https://img.example-cdn.com/hero.jpg?w=1600"
       srcset="https://img.example-cdn.com/hero.jpg?w=800 800w,
               https://img.example-cdn.com/hero.jpg?w=1600 1600w"
       sizes="(min-width: 64rem) 50vw, 100vw"
       width="1600" height="900" alt="…"
       fetchpriority="high" decoding="async">
</picture>
```

RUM with attribution:

```js
import { onLCP, onINP, onCLS } from 'web-vitals/attribution';
const send = (m) => navigator.sendBeacon('/rum', JSON.stringify({
  name: m.name, value: m.value, rating: m.rating,
  target: m.attribution?.interactionTarget ?? m.attribution?.element,
  page: location.pathname,
}));
onLCP(send); onINP(send); onCLS(send);
```

Long-task chunking for INP:

```js
async function processAll(items) {
  for (const item of items) {
    process(item);
    if ('scheduler' in window && 'yield' in scheduler) await scheduler.yield();
    else await new Promise(r => setTimeout(r, 0));
  }
}
```

## Verification / self-check

- Every recommendation must name the metric and the *sub-part* it improves (e.g., "LCP load delay"); if you can't, you're guessing.
- Fix validated in a throttled lab trace *and* an explicit plan to confirm in field data (RUM or next CrUX window). State the lag.
- Bundle claims verified with an analyzer output, not package-size intuition. Preloads verified to improve the waterfall, not just added.
- Check you didn't trade metrics: lazy-loading that fixed bandwidth but pushed LCP; skeleton screens that fixed perceived speed but added CLS; SW caching that will strand deploys.
- Stopping rule: p75 green on all three CWV with margin, and the top RUM-attributed element/interaction addressed. Chasing lab 100s beyond that is decoration.
