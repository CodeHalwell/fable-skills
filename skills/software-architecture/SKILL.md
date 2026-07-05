---
name: software-architecture
description: Load when decomposing a system into modules or services, deciding monolith vs microservices, drawing module/team boundaries, evaluating or writing architecture decision records, choosing dependency directions, or judging whether an abstraction/layer is worth its cost. Also load for "should we split this service" and "how do we structure this codebase" questions.
---

# Software Architecture — expert-correction sheet

The cold baseline already covers the standard material at full specificity: monolith-vs-microservices forcing functions and the "no" to small teams, entity-service critique, extract-leaf-workers-first ordering, availability math of sync chains, dual-write/transactional-outbox, async-edge semantics, wrong-abstraction-vs-duplication cost model, shared-domain-types-in-common-lib critique, and per-ecosystem fitness-function tooling (import-linter, ArchUnit, dependency-cruiser, Nx boundaries, Tach). Do not re-derive those at length; state them and move on. This sheet contains only what the baseline omits or under-specifies.

## Sharpenings the baseline misses

### Boundary tests beyond the standard set
The reflex list is change-locality, data-ownership, chatty-call, independent-deploy, transaction-boundary. Add these three, which the standard answer omits:
- **Deletion test:** could you `rm -rf` the module and know exactly what breaks from its interface alone? If answering requires reading its internals, the boundary leaks.
- **Deep-module ratio:** a boundary is real only if its interface is much smaller than its implementation. A layer where every method is a one-line pass-through to the layer below is *negative* value — delete it, don't document it.
- **Cycle rule with the event trap:** if A needs B and B needs A, they are one module — merge, or extract the shared piece C. Never "fix" a cycle with an event that exists only so B can call back into A: that's events-as-disguised-calls; the cycle survives, now invisible to the dependency graph and the linter.
- **Replay threshold, made exact:** test a proposed decomposition by replaying the last 10 real changes; a good boundary puts ≥8 of them entirely on one side. Argue boundaries from the change log, not from the domain diagram — the diagram produces noun boundaries (the entity-service failure) even when everyone knows better.

### ADRs: the field everyone omits is the revisit trigger
Context / options-with-real-rejections / decision / consequences / status is the standard template — and it still produces decisions that get re-litigated quarterly. Add a **falsifiable revisit trigger** as required content: "if p99 exceeds X," "if team count exceeds N," "if we add a second tenant type." It converts "no for now" into a decision that self-updates, and it is how you tell a small team "no microservices" exactly once. A proposal that can't state its revisit trigger isn't ready to be decided.
- Second tell: a consequences section with only upsides means the analysis wasn't done. Every real decision worsens something (a split worsens latency/debuggability/local dev; a cache adds an invalidation bug class; a queue adds redelivery semantics). Produce the costs column yourself before agreeing.

### Deferral is bought with seams, not mechanisms
The cheap way to keep an option open is cohesion — the candidate module's code lives together and communicates through a narrow neck — not building the plugin system / multi-backend mechanism now. A seam costs nothing until used; a mechanism costs maintenance forever. Corollary for dependency inversion: invert only at the 2–4 boundaries named in your explicit architecture bet ("we expect to swap payment providers; we do not expect to swap databases" — that sentence *is* the architecture); concrete calls everywhere else. Interfaces-with-one-implementation-forever wired by DI are pure reading tax.

## Failure modes the baseline doesn't volunteer

It will name distributed monolith, entity services, shared common-lib coupling, and big-bang rewrite unprompted. It usually won't name these:

- **Cache as architectural glue.** Service A reads service B's data via a shared cache or replica "to avoid coupling." That is an undocumented, unversioned API with no owner whose schema is B's internals; when B refactors storage, A breaks and nobody knows why. Data crosses ownership boundaries only through contracts: API, published events, or replicated read models with a schema.
- **Saga sprawl is a boundary symptom, not a fact of life.** If most workflows need cross-service sagas with compensation logic, the boundaries cut *through* the workflows. The fix is not better saga tooling — redraw so each workflow's happy path lives inside one service; reserve sagas for the few genuinely cross-organizational flows.
- **Event-driven as internal default.** Making in-process module communication event-based "for decoupling" trades a legible call graph for an invisible one: nobody can answer "what happens when an order is placed" without grepping subscribers, and at-least-once/ordering bugs appear where none were needed. Rule: commands/direct calls for workflows that must complete (one readable orchestration function); events only for genuinely open-ended fan-out where the emitter must not know its consumers (analytics, cache invalidation, third-party integrations).
- **Flag debt as shadow architecture.** Long-lived feature flags compound until the deployed system is one of 2^N configurations, none tested. Flags are scaffolding: owner and removal date assigned at creation; flag cleanup goes in the definition of done for the launch it guarded.
- **Coincidental duplication merged into real coupling.** Same-shaped code with *different reasons to change* (identical-looking validation on two unrelated forms) is not duplication — merging it welds unrelated change vectors together. Abstract on the third *identical-for-the-same-reason* duplicate, not the third lookalike.

## Self-check before presenting a recommendation

- Change-vector bet stated explicitly, every boundary traceable to it; "for cleanliness" boundaries cut.
- Replay 5–10 recent changes: median modules/services touched per change must be 1.
- Costs column present for every element (what it makes worse; hop arithmetic; carrying cost if the anticipated change never comes).
- One-way doors (datastore, public API shape, service split) get ADRs with revisit triggers; two-way doors get decided fast — a month spent deciding a two-way door is itself an architecture failure.
- Survives team doubling *and* halving? (3 people / 12 services and 12 engineers / 1 hot module both fail.)
- Every stated rule has a nameable CI enforcement; unenforced rules erode silently within months.
- Exactly one writer per store; any shared-write store is an undeclared merge.
- Every async edge: at-least-once → idempotent consumers, reordering tolerance, eventual consistency for readers — written down, each handled.
- Feature locations answerable from the directory listing alone (package by feature, not by technical layer).
- Checked against: distributed monolith, entity services, shared common-lib, event-default, big-bang rewrite, cache-as-glue, saga sprawl. Most bad proposals are one of these wearing a new name.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)
- Probed 12 claims: 10 baseline (cut/compressed), 2 partial (sharpened), 0 delta.
- Biggest baseline gaps found:
  - ADR template: Opus gives context/options/decision/consequences/status but omits the revisit trigger entirely — the one field that stops re-litigation.
  - Boundary tests: Opus's set lacks the deletion test, the deep-module interface-to-implementation ratio, and the "never fix a cycle with events-as-disguised-calls" rule.
  - Everything else — microservices economics, outbox/dual-write, async semantics, wrong-abstraction model, extraction order, fitness tooling, and the full worked 6-engineer scenario (where Opus even added characterization tests) — was reproduced cold at full specificity, so the v2 survey content was cut.
