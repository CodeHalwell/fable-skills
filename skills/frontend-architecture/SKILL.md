---
name: frontend-architecture
description: Load when designing or reviewing React-era frontend architecture at scale — state management choices, re-render performance, Server Components boundaries, component API design, useEffect misuse, framework selection (React/Vue/Svelte/Solid), or micro-frontend proposals.
---

# Frontend Architecture

## Core mental model (anchors)

1. Most state isn't yours: server data → query layer/RSC; shareable view state → URL; forms → form library/uncontrolled; what remains as true client state is usually tiny and rarely justifies a store.
2. Optimize by narrowing what re-renders (colocation, composition) before memoizing; a component re-renders because of its own state, a consumed context, a subscribed store, or its parent — never "props changed" directly.
3. `"use client"` is a module-graph boundary, not a tree position: everything *imported* below it ships to the browser; pass server-rendered content through client wrappers as children/props.
4. Effects synchronize with external systems; if you can't name the external system, the logic belongs in render, an event handler, or the data layer.

## Current state (verified July 2026)

- **React Compiler 1.0** (Oct 2025) is production-stable, back-compatible to React 17 via `react-compiler-runtime`. On compiler-enabled codebases stop hand-writing `useMemo`/`useCallback`; it does NOT fix architectural width (a per-keystroke context still re-renders every subscriber). Structure first, compiler second.
- **`useEffectEvent` is stable as of React 19.2** — cold answers still call it experimental/canary; it shipped. Use it for "trigger on X, read latest Y" instead of lying to the deps lint.
- Next.js 16: `cacheComponents` + `use cache` directive (+ `cacheLife`/`cacheTag`), dynamic-by-default, Partial Prerendering. The Next 13/14 "fetch is cached by default" model is gone — don't reason from it. Async `params` since Next 15.
- RSC outside Next (React Router v7, TanStack Start) exists but is younger — verify support before assuming.
- Virtualize lists past ~a few hundred rows (`virtua`, TanStack Virtual); `nuqs` for typed URL search params.

## The state location decision chain (kept as the checklist; first "yes" decides)

1. Server owns the truth → query layer (TanStack Query/SWR) or RSC; never copy into a client store. Set `staleTime` deliberately — default-0 causes refetch storms.
2. Should a shared link or back button reproduce it → URL state. Ask early: retrofitting filters from `useState` to URL is a rewrite. Corollary: half-typed pending input stays local; commit to URL on apply/debounce.
3. Form → react-hook-form's register model, or platform `<form>` + Actions/`useActionState` (React 19).
4. Needed by distant components → small store (Zustand/Jotai) *only now*; context only for rarely-changing values; `useSyncExternalStore` for non-React sources.
5. Otherwise `useState`, colocated. Lifting is a cost, not a virtue.

Derived state is never stored; undo/history wants a reducer with an event log; optimistic/offline wants a sync engine (see `realtime-web`).

## Judgment calls and sharpened specifics

- **Diagnostic order for slow interactions:** Profiler with "record why each component rendered" first — the prior is one wide subscription at the top, not a thousand small ones. Then structural fixes (push state down; lift content up — children passed as props don't re-render with the parent's own state; narrow selectors) before any `memo`. If renders are cheap but interaction lags: layout thrash in effects, unvirtualized lists, or urgent/non-urgent mixing (`useTransition`/`useDeferredValue`).
- **useEffect review priors, most common first:** (1) fetching that belongs in the query layer/RSC; (2) deriving state ("sync A into B") that belongs in render — reset-on-prop-change wants `key`, not an effect; (3) watching a flag to react to a user event — put it in the handler; (4) legitimate external sync — add cleanup, and `useEffectEvent` where deps would lie.
- **Micro-frontends:** the honest trigger is independent deployment by autonomous teams with incompatible release cadences — an org-chart problem. Single team → monorepo with enforced module boundaries. If forced: Module Federation with strictly-singleton React; two React copies break context/events in ways that surface months later.
- **Server Actions are public POST endpoints**: authenticate, authorize, and schema-validate *inside every action*; closed-over values round-trip through the client — treat as untrusted, never secret.

## Pitfalls checklist (one-liners — cold answers reproduce the mechanisms)

Fetch-in-effect pathology (no dedup, races, waterfalls); context value as fresh object literal (memoize or split by change-rate); `"use client"` at layout root; non-serializable props across the RSC boundary; `memo` defeated by inline object/`children` props; index-as-key on reorderable lists; hydration mismatch from non-deterministic render (two-pass pattern; `suppressHydrationWarning` only for expected cases like timestamps); client-side waterfalls → route-level `prefetchQuery`/`ensureQueryData`; StrictMode double-invoke exposes non-idempotence — fix the code, never remove StrictMode; prop-drilling three levels is fine — composition removes most of it without a store; Zustand store with hand-written loading/error flags per resource is 2018 Redux rebuilt.

## Signal-based frameworks (Vue 3 / Svelte 5 runes / Solid)

Re-render-width reasoning and the memo discipline are React-specific taxes — don't port `useCallback` habits. What carries over: the state-location chain, server-cache principle (adapters exist; Pinia Colada for Vue), URL rule, component-API guidance. Derived state is first-class (`computed`/`$derived`/`createMemo`) — use it instead of effects always. Choose on ecosystem/hiring, not benchmarks.

## Micro-example: the RSC boundary + authorized action (the whole shape in 12 lines)

```tsx
// page.tsx — Server Component: data access, zero client JS
export default async function OrderPage({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;                 // async params, Next 15+
  const order = await getOrder(id);            // direct service call, no API hop
  return <article><OrderTimeline events={order.events} /><AddNoteButton orderId={order.id} /></article>;
}                                              // AddNoteButton is the only "use client" island

// actions.ts
'use server';
export async function addNote(raw: unknown) {
  const session = await auth();                          // authorize INSIDE
  if (!session) throw new Error('unauthorized');
  const input = NoteInput.parse(raw);                    // validate INSIDE
  await assertCanEditOrder(session.user.id, input.orderId);
  await db.notes.create({ ...input, authorId: session.user.id });
  revalidatePath(`/orders/${input.orderId}`);
}
```

## Verification / self-check

- Every piece of state: name its category (server/URL/form/client-local/client-global) and tool; >1 global client store item → re-justify each.
- Re-render claims verified with a Profiler recording, naming the component and why.
- RSC: trace one interactive feature — the `"use client"` file imports nothing server-only and sits at a leaf; every action authorizes.
- Version-gate compiler/Next-caching/`useEffectEvent` advice against the project's actual React/Next versions.
- Stop when state categories are assigned, boundaries drawn, and the two most expensive interactions profiled.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)

- Probed 14 claims: 13 baseline (cut/compressed), 0 partial, 1 delta (expanded).
- The delta: `useEffectEvent` stability — cold answers say "experimental/canary only"; it's stable in React 19.2. Everything else (Compiler 1.0 status, Next 16 `use cache` model, children-as-props mechanics, Server Action threat model, signals-framework contrast) Opus produced cold, often with more detail than the skill had.
- Remaining value: the stability correction, decision-chain compactness, review priors ordering, and version-gating discipline.
