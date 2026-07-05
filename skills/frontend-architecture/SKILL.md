---
name: frontend-architecture
description: Load when designing or reviewing React-era frontend architecture at scale — state management choices, re-render performance, Server Components boundaries, component API design, useEffect misuse, framework selection (React/Vue/Svelte/Solid), or micro-frontend proposals.
---

# Frontend Architecture

## Core mental model

1. **Most state isn't yours.** The majority of "state management" pain in React apps is server data being manually cached in client stores. Server data belongs in a server-cache layer (TanStack Query, SWR, RSC, or a sync engine) — not in Redux/Zustand. Once you subtract server state, URL state, and form state, the truly-client state left over is usually tiny (a modal flag, a theme, a multi-step wizard's progress) and rarely justifies a library at all.
2. **Re-renders are cheap until they aren't; reconciliation is not the DOM.** A re-render is a function call producing a virtual tree; the DOM only changes where the diff differs. Performance problems come from *wide* renders (a context change re-rendering 500 subscribers) or *expensive* renders (heavy computation per render), not from renders per se. Optimize by narrowing what re-renders (state colocation, composition) before memoizing.
3. **The server/client boundary is an architectural line, not a performance trick.** With React Server Components, `"use client"` marks the *entry point* to client territory — everything imported below it ships to the browser. Design the tree so interactivity lives in leaf islands and data access lives above them.
4. **Component APIs are contracts; design for the caller.** A component's props are its public API. Prefer composition (children, slots) over configuration (a `variant`+12 boolean props matrix). Every boolean prop is a fork you must test in combination.
5. **Effects are an escape hatch for synchronizing with external systems** — DOM APIs, subscriptions, analytics, non-React widgets. If an effect only touches React state derived from props/state, it's almost always wrong: derive during render, or handle it in the event handler that caused the change.

## The state location decision chain

Ask these questions **in order** — the first "yes" decides:

1. **Does the server own the truth?** (user profile, list of orders, anything fetched)
   - → Server-cache layer: TanStack Query / SWR on the client, or fetch in Server Components.
   - Never copy it into a client store; you'd own invalidation, dedup, and staleness by hand — exactly what these libraries exist to solve.
   - `useEffect`+`setState` fetching is the pathology: no dedup, no cache, race-prone, waterfall-prone.
2. **Should a shared link or the back button reproduce it?** (filters, tab, pagination, search query, selected item)
   - → URL state (`useSearchParams`, or `nuqs` for typed search params in Next.js).
   - Teams habitually put filter state in `useState` and then discover users can't share filtered views — this is a rewrite, so ask early.
   - Corollary: if the back button should *not* restore it (a half-typed search box), keep the pending value local and commit to the URL on apply/debounce.
3. **Is it a form?**
   - → Form state library or uncontrolled inputs (react-hook-form's register model; or platform `<form>` + Actions and `useActionState` in React 19).
   - Controlled-everything forms re-render the entire form per keystroke; that's the main reason "our form is slow."
4. **Is it needed by distant components?**
   - → Small client store (Zustand, Jotai) *only now*.
   - Context is fine for rarely-changing values (theme, auth identity); it's wrong for frequently-changing values because every consumer re-renders on any change.
   - If subscribing to a non-React external source (browser API, third-party store), use `useSyncExternalStore`, not effect+state mirroring — it's tearing-safe under concurrent rendering.
5. **Otherwise** → `useState`/`useReducer`, colocated as close to usage as possible. Lifting state up is a cost, not a virtue — lift only as far as the lowest common ancestor.

What changes the answer:
- Optimistic updates or offline requirements → consider a sync engine (see `realtime-web`).
- Undo/history requirements → reducer with an explicit event log, not snapshot-copying components.
- State that's purely derivable → never store it; compute it during render, `useMemo` only if measurably expensive. Storing derived state creates the synchronization bugs the effect section below describes.

## Rendering performance reasoning

The expert's diagnostic order when "the app feels slow on interaction":

1. **Find what re-renders** (React DevTools Profiler, "record why each component rendered"). The prior: it's usually one wide subscription — a context or store selector at the top — not a thousand small ones. A component re-renders because: its state changed, its parent re-rendered, or a context it consumes changed. Props changing is *not* on that list — parent re-render is what actually triggers it, which is why `memo` (opt into prop comparison) exists at all.
2. **Narrow before memoizing.** Three structural fixes beat `memo`:
   - *Push state down*: if only the search box needs the query string, don't hold it in the page component.
   - *Lift content up*: `<Parent>{expensiveChildren}</Parent>` — children passed as props don't re-render when Parent's own state changes, because the element was created by the grandparent.
   - *Subscribe narrowly*: Zustand/Jotai selectors, or split one context into several by change-frequency.
3. **Memoize what remains.** `memo` on the expensive subtree boundary; `useMemo`/`useCallback` only to preserve referential identity for that `memo` or for effect deps. Memoizing everything by hand adds noise and breaks silently (one non-memoized object prop defeats the whole chain).
4. **The compiler era (as of 2026):** React Compiler 1.0 shipped October 2025 and is production-stable; it auto-memoizes components and hooks (compatible back to React 17 via `react-compiler-runtime`). On compiler-enabled codebases, stop hand-writing `useMemo`/`useCallback` for render performance — write idiomatic code and follow the Rules of React (the compiler skips components that break them; `eslint-plugin-react-hooks` latest flags violations). The compiler does *not* fix architectural width problems: a context that changes every keystroke still re-renders every subscriber. Structure first, compiler second.
5. **If renders are cheap but interaction still lags**, the cost is elsewhere: synchronous layout thrash in effects, huge lists without virtualization (use `virtua` or TanStack Virtual past ~a few hundred rows), or non-urgent updates blocking urgent ones (`useTransition`/`useDeferredValue` for filter-as-you-type over large data).

## Server Components and the boundary (state as of 2026)

- RSC is production-mainstream via Next.js App Router (Next.js 16 current); React 19.x has first-class Server Components, Actions, `use`. Non-Next adoption (React Router v7, TanStack Start) exists but is younger — verify current support before assuming RSC outside Next.
- **Decision:** does this component need interactivity, browser APIs, or hooks with state? No → keep it a Server Component (default). Yes → it's a client component, but push the `"use client"` boundary to the smallest subtree. Pass Server Components *as children/props into* client components to avoid pulling static content into the bundle — the boundary is about module imports, not tree position.
- Data flows down as serializable props; mutations flow up as Server Actions (`"use server"`). Actions are POST endpoints in disguise: validate and authorize inside every action as you would any API route — the function-call syntax hides the network but not the threat model.
- Avoid server-side request waterfalls: two independent `await fetch(...)` lines in one Server Component serialize. Start promises together (`Promise.all`, or kick off both and `await` late), or push each fetch into the component that needs it and let Suspense parallelize the streams.
- Caching (Next.js 16): `cacheComponents` + the `use cache` directive replace the old fetch-cache heuristics and `unstable_cache`; dynamic-by-default, opt into caching per page/component/function, with Partial Prerendering serving a static shell while dynamic parts stream. Don't reason from Next 13/14-era "fetch is cached by default" memories — that model is gone.
- RSC replaces the *server-read* half of TanStack Query for initial data; client-side interactive refetching (infinite scroll, polling, optimistic mutation) still belongs to a client cache. Many apps legitimately use both (hydrate the query cache from server data, then let the client layer own updates).

## Effects: the escape-hatch test

Before writing `useEffect`, name the *external system* being synchronized (DOM measurement, subscription, `document.title`, analytics, a map/chart widget, a timer). If you can't name one, the effect is wrong — the logic belongs in render (derivation), an event handler (user-caused changes), or the router/data layer. Priors on `useEffect` review, most common first:
1. Data fetching that belongs in the query layer/RSC.
2. State derivation ("sync state A into state B") that belongs in render.
3. Reacting to a user event indirectly (`useEffect(..., [submittedFlag])`) — put the logic in the submit handler; effects that watch flags create ordering puzzles and double-fires.
4. Legitimate external sync — keep it, add cleanup, and check the dependency list isn't lying (`useEffectEvent` for "trigger on X, read latest Y").

## Component API design

- Start from the call sites you *want*, then implement backwards. If usage requires reading docs to order 8 props, redesign.
- Compound components for coordinated pieces (`<Tabs><Tabs.List/><Tabs.Panel/></Tabs>`) beat a `tabs={[{label, content}]}` config prop — callers regain JSX composition, styling access, and insertion points.
- Accept `children` over `renderX` props where possible; accept a `ReactNode` over a string for anything that might need markup later.
- Forward the platform: spread `...rest` onto the underlying element, forward `ref` (a plain prop in React 19; `forwardRef` no longer needed there), respect `className`/`style` merging. Wrappers that swallow DOM props are the #1 source of "I can't attach my tooltip/test-id/aria-label."
- Prefer controlled+uncontrolled duality (`value`/`defaultValue` + `onChange`) for inputs-like components; document which mode wins.
- Use existing headless primitives (Radix, React Aria, Base UI) for anything with keyboard/focus semantics rather than reimplementing — see `web-accessibility`.

## How an expert thinks through this

*Scenario: "Our dashboard is slow and state is a mess — we're planning to move everything into Redux Toolkit and wrap components in memo."*

Internal monologue: First, inventory the state. The dashboard shows org metrics (fetched), a date-range filter, a chart-type toggle, and a settings modal. Metrics are server state — the current code fetches in `useEffect`, stores in Redux, and invalidates via a `refreshFlag` counter. That's a hand-rolled cache with no dedup; moving it to RTK doesn't fix that, RTK Query or TanStack Query does. Reject "everything into Redux": it centralizes the wrong thing. Date range and chart type — a teammate will share a link to "last quarter, bar view" eventually; that's URL state, `useSearchParams`. Modal flag: `useState` in the dashboard shell. Remaining global client state: nothing. So the plan needs *zero* store.

Now performance. Profile first — hypothesis: the filter lives in a top-level context, so every keystroke in the date input re-renders all 12 chart cards. Profiler confirms. Considered `memo` on each card: rejected as first move, because cards receive an `options` object literal rebuilt each render — memo would silently no-op until we also memoize the object, and the compiler will eventually own that. Structural fix instead: move the pending input value into the filter component (local state), commit to the URL on apply/debounce; cards subscribe to the committed URL value only. Re-render width drops from 12 cards/keystroke to 0. Then enable React Compiler and delete the existing scattered `useCallback`s in a follow-up. Stopping rule: interaction trace shows <50ms scripting per keystroke and the Profiler shows only the intended components rendering — further memoization is waste.

## Failure modes & pitfalls

- **Fetching in `useEffect` and mirroring into state** — no dedup across components, StrictMode double-invocation confusion, race conditions on fast param changes (fix requires an ignore flag or AbortController; better: don't hand-roll — use a query library or RSC).
- **Deriving state with effects**: `useEffect(() => setFullName(first + last), [first, last])` renders twice and can flicker. Derive in render. Same for "reset child state when prop changes" — use a `key` on the child, not an effect.
- **`useEffect` with an object/array dep rebuilt each render** → runs every render. Either primitive deps, or memoize the input, or restructure. Conversely, suppressing the lint rule with `eslint-disable-next-line react-hooks/exhaustive-deps` almost always hides a stale-closure bug; for "react to X but read latest Y", use `useEffectEvent` (stable as of React 19.2).
- **Context as a store**: one big `AppContext` whose value is a fresh object literal `{user, theme, setTheme, cart, ...}` re-renders every consumer on any change *and* on every provider render (new object identity). Split by change-rate and memoize the value — or use a store with selectors.
- **`"use client"` at the layout root**, dragging the whole app into the client bundle because one header button needed `onClick`. Isolate the button.
- **Passing non-serializable props across the RSC boundary** (functions, class instances, Dates pre-React-19-serialization support for some types) — fails at runtime; restructure so the client component owns the callback.
- **Server Action without auth check** because "it's just a function" — it's a public endpoint; check session inside the action.
- **`memo` defeated invisibly**: any inline object/array/function prop, or `children` (always a fresh element), makes `memo` compare-and-fail every time. If you must memo a component that takes children, memoize at the caller or restructure.
- **Index-as-key on reorderable lists** — state (input values, animations) sticks to positions, not items. Stable IDs, always.
- **Hydration mismatch from non-deterministic render**: `Date.now()`, `Math.random()`, locale-dependent formatting, or `typeof window` branches that change markup produce the "hydration failed" error and a full client re-render. Compute such values in effects/state after mount, pass them from the server as props, or use `suppressHydrationWarning` only for genuinely-expected mismatches like timestamps.
- **Client-side data waterfalls**: parent fetches, renders child, child fetches — serial round trips. Hoist queries to route level (router loaders + `queryClient.prefetchQuery`/`ensureQueryData`), or fetch in parallel and let components `useQuery` the already-warm cache.
- **StrictMode-double-render "bugs"**: dev-only double invocation of render and effects exposes non-idempotent code. The fix is making the code idempotent (cleanup functions, abortable fetches), never removing StrictMode.
- **Prop-drilling overcorrection**: three levels of prop passing is fine and explicit; reaching for context/store to avoid it trades greppable data flow for hidden coupling. Component composition (pass the composed child down) removes most drilling without either.
- **Zustand store holding fetched data** with hand-written `loading`/`error` flags per resource — you've rebuilt 2018 Redux. Query library, or keep it and accept you now own cache invalidation forever.
- **Micro-frontends adopted for code organization**: the honest trigger for micro-frontends is *independent deployment by autonomous teams with incompatible release cadences* — an org-chart problem. If one team owns the app, a monorepo with enforced module boundaries (Nx/Turborepo + lint rules) gives the modularity without duplicate framework payloads, version-skew matrices, cross-app routing hacks, and design drift. If genuinely needed, prefer build-time composition or Module Federation with a *strictly shared* singleton React — two React copies on one page breaks context and events in ways that surface months later.

## When Vue/Svelte/Solid change the calculus

- Their fine-grained/signal-based reactivity (Vue `ref`/`computed`, Svelte 5 runes, Solid signals) means *component-level re-render reasoning mostly disappears*: updates target the exact DOM bindings that read a signal, so the memo/context-width material above is largely React-specific. Don't port `useCallback` habits into signal frameworks — referential identity of callbacks doesn't cause re-render cascades there.
- What carries over unchanged: the state-location chain, the server-cache principle (TanStack Query ships Vue/Svelte/Solid adapters; Pinia Colada for Vue), the URL-state rule, form-state reasoning, and all component-API guidance.
- What differs: derived state is a first-class primitive (`computed`/`$derived`/`createMemo`) — use it instead of effects *always*; SSR stories differ (Nuxt and SvelteKit have mature server rendering; RSC-style server/client module boundaries remain React-specific).
- Choose frameworks on ecosystem depth, hiring pool, and team experience more than benchmarks — the runtime performance differences are real but rarely the binding constraint; the component-library and tooling gaps are.

## Worked micro-examples

**1. Server state + URL state + minimal client state, TanStack Query v5 style:**

```tsx
// queries.ts — key factory + queryOptions (v5 pattern)
import { queryOptions } from '@tanstack/react-query';

export const metricsQuery = (orgId: string, range: string) =>
  queryOptions({
    queryKey: ['metrics', orgId, { range }],
    queryFn: ({ signal }) =>
      fetch(`/api/orgs/${orgId}/metrics?range=${range}`, { signal })
        .then(r => { if (!r.ok) throw new Error(r.statusText); return r.json(); }),
    staleTime: 60_000, // metrics don't change per-second; stop default-0 refetch storms
  });

// Dashboard.tsx
function Dashboard({ orgId }: { orgId: string }) {
  const [params, setParams] = useSearchParams();          // URL state: shareable
  const range = params.get('range') ?? '30d';
  const { data, isPending } = useQuery(metricsQuery(orgId, range));
  const [settingsOpen, setSettingsOpen] = useState(false); // true client state: local
  // ...
}
```

Invalidation after mutation: `onSettled: () => queryClient.invalidateQueries({ queryKey: ['metrics', orgId] })` — invalidate the prefix, let active queries refetch.

**2. RSC boundary + authorized Server Action (Next.js App Router):**

```tsx
// app/orders/[id]/page.tsx — Server Component: data access, zero client JS
import { AddNoteButton } from './add-note-button';
export default async function OrderPage({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;                      // async params in Next 15+
  const order = await getOrder(id);                 // direct DB/service call, no API hop
  return (
    <article>
      <h1>Order {order.number}</h1>
      <OrderTimeline events={order.events} />       {/* stays server-rendered */}
      <AddNoteButton orderId={order.id} />          {/* the only client island */}
    </article>
  );
}

// app/orders/actions.ts
'use server';
import { z } from 'zod';
const NoteInput = z.object({ orderId: z.string().uuid(), body: z.string().min(1).max(2000) });
export async function addNote(raw: unknown) {
  const session = await auth();                     // authorize INSIDE the action
  if (!session) throw new Error('unauthorized');
  const input = NoteInput.parse(raw);               // validate INSIDE the action
  await assertCanEditOrder(session.user.id, input.orderId);
  await db.notes.create({ ...input, authorId: session.user.id });
  revalidatePath(`/orders/${input.orderId}`);
}
```

The button component is the whole `"use client"` surface; the page, timeline, and data layer never ship to the browser.

## Verification / self-check

- For every piece of state in the design, name its category (server/URL/form/client-local/client-global) and the tool; if >1 global client store item exists, re-justify each.
- Re-render claim? Verify with the Profiler recording, not intuition — name the component that re-renders and why.
- RSC design: trace one interactive feature and confirm the `"use client"` file imports nothing server-only and sits at a leaf; confirm every Server Action authorizes.
- Version-sensitive advice (compiler, Next caching, `useEffectEvent`) — confirm the project's actual React/Next versions before prescribing.
- Stopping rule: architecture review is done when state categories are assigned, boundaries drawn, and the two most expensive interactions profiled; further abstraction ("what if we need X later") is speculation — stop.
