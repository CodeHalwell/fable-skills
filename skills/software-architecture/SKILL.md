---
name: software-architecture
description: Load when decomposing a system into modules or services, deciding monolith vs microservices, drawing module/team boundaries, evaluating or writing architecture decision records, choosing dependency directions, or judging whether an abstraction/layer is worth its cost. Also load for "should we split this service" and "how do we structure this codebase" questions.
---

# Software Architecture

## Core mental model

- **Coupling vs cohesion is the only real rule; everything else is a corollary.** Every named principle (SRP, layers, hexagonal, microservices) is a strategy for one goal: things that change together live together; things that change independently can be changed independently. When a pattern and this goal conflict, the pattern loses.
- **Draw boundaries along rate-of-change and reason-for-change lines, not noun lines.** The classic mistake is decomposing by data shape (UserModule, OrderModule) when the actual change vectors are "pricing rules change weekly, tax logic changes yearly, UI changes daily." A boundary is good iff a typical change lands on one side of it. Test any proposed decomposition by replaying the last 10 real changes: how many would have touched one module vs three?
- **Dependencies must point from volatile to stable.** Business rules shouldn't import the database driver; the database adapter should import the business rules' interfaces. Dependency inversion is not ceremony — it's the mechanism that lets the stable core survive infrastructure churn. But invert only at boundaries that plausibly churn: wrapping your language's stdlib behind interfaces is cargo cult.
- **Architecture is a bet on which changes will come.** You cannot make everything easy to change; every flexibility purchased in dimension A adds indirection cost in dimensions B–Z. State the bet explicitly ("we expect to swap payment providers, we do not expect to swap databases") — that sentence is the architecture; the boxes are its consequence.
- **The unit of decomposition cost is a network hop and a team boundary.** A function call is nanoseconds and refactorable by an IDE. A service call is milliseconds, partial failure, versioned contracts, distributed tracing, and a meeting to change. Never pay service prices for module benefits.

## Decision frameworks

### Monolith vs microservices economics
- **Default: modular monolith.** One deployable, strict internal module boundaries (enforced by build tooling — import linting, visibility rules — not convention). You get boundary benefits at function-call prices, and correct boundaries can later be extracted cheaply; wrong boundaries in a monolith cost a refactor, wrong boundaries across services cost a distributed refactor plus data migration plus API deprecation.
- Microservices pay only when a *specific* forcing function exists, per service:
  | Forcing function | Why it pays |
  |---|---|
  | Independent scaling with 10x+ asymmetric load (e.g., ingest vs admin UI) | Scaling the monolith means paying for the max of all needs |
  | Genuinely different runtime needs (GPU inference, different language, real-time constraints) | One deployable can't serve both |
  | Organizational: >~4 teams shipping on conflicting cadences, deploy queue is the bottleneck | Conway's law is real; deploy independence is the actual product of microservices |
  | Hard fault isolation / blast-radius or compliance boundary (PCI scope) | Process boundary is the requirement itself |
- "It's more scalable" is not a reason — monoliths scale horizontally fine until state says otherwise. "Cleaner separation" is not a reason — that's what module boundaries are for. Team of <10 asking for microservices: the answer is no, with the ADR explaining the revisit trigger.
- **Extraction order when you do split:** extract the thing with the fewest synchronous call edges and clearest data ownership first (often async workers: email, exports, media processing). Never extract the entity that everything joins against (usually "user" or "order") first — you'll turn every request into a distributed join.

### Where to put a boundary — quick tests
- **Rate-of-change test:** list the last 10 changes; a good boundary means ≥8 touched one side only.
- **Interface-to-implementation ratio:** a module boundary is real if its interface is much smaller than its implementation (deep module). A "layer" where every method is a one-line pass-through to the next layer down is negative-value — delete it.
- **Data ownership test:** exactly one module writes each table/aggregate. Two services writing one table is not a boundary, it's a shared mutable global with extra steps.
- **Circular check:** if A needs B and B needs A, they are one module; merge them or extract the shared piece C that both depend on. Never "fix" a cycle with events-as-disguised-calls (A emits event that only exists so B can call back).

### Abstraction cost model
- **Wrong abstraction > duplication (in cost).** Duplication costs a linear tax you can pay down anytime; a wrong abstraction shared by N callers costs a parameter/flag per divergence, and each flag doubles the config space and welds callers together. When you find yourself adding a boolean to a shared helper so caller 3 can behave differently: stop, inline the helper into the callers, let them diverge, re-abstract later from real duplicates.
- **Rule of three, but with a twist:** abstract on the third *confirmed identical-for-the-same-reason* duplicate. Two pieces of code that look alike but change for different reasons (same-shaped validation for two unrelated forms) are *coincidental* duplication — merging them creates coupling between unrelated change vectors.
- **Indirection budget:** each layer must earn its keep by absorbing a change you actually expect. "Repository interface over the ORM" earns its keep if you swap or fake storage in tests; "ServiceImpl behind ServiceInterface with exactly one impl forever, injected by framework" is pure reading tax.

### ADRs (architecture decision records)
- Write one whenever a decision is expensive to reverse or will be re-litigated: datastore choice, sync vs async boundary, build-vs-buy, service split, auth model.
- Required content, in order of value: **context** (forces, constraints, numbers), **options considered with the real reasons the losers lost**, decision, consequences (including what gets *worse*), and **revisit trigger** ("if p99 exceeds X" / "if team count exceeds N"). An ADR without rejected options is a press release, not a record.
- Immutable + numbered + in-repo (`docs/adr/0007-use-postgres-outbox.md`). Superseding decision links back rather than editing. The archaeological value — "why is it like this" answered in 2 minutes instead of 2 days — is the entire point.

### Evolutionary architecture
- Prefer reversible steps over big designs: strangler-fig around legacy (route traffic incrementally, never big-bang rewrite), branch-by-abstraction for in-place replacement (introduce seam → move callers → swap impl → remove seam), parallel-run with diffing before cutover for anything correctness-critical.
- Encode architectural rules as **fitness functions** — executable checks in CI: import-boundary linting (`import-linter` in Python, ArchUnit in Java, `depcruise` in JS), dependency-direction tests, "no module may import `internal/` of another." A rule not enforced by tooling erodes in months; the erosion is silent until it's structural.

## Failure modes & pitfalls

- **Layering by technology instead of by domain.** `controllers/`, `services/`, `models/` as top-level structure means every feature change touches every directory and no code can be deleted as a unit. Correction: package by feature (`billing/`, `catalog/`) with layers inside if needed. You should be able to `rm -rf` a feature.
- **The distributed monolith.** Services that must deploy together, share a database, or call each other synchronously in chains (A→B→C→D per request) have all microservice costs and no benefits. Detect it: ask "can I deploy service B alone on a Friday?" and "what's the availability math?" — four 99.9% services in a synchronous chain yield 99.6%, ~35 min/week worse. Correction: either merge them back or make edges async with owned data.
- **Entity-service decomposition.** UserService, OrderService, ProductService — nouns, not capabilities. Every business operation now orchestrates 3+ services, and cross-service transactions appear ("create order + decrement inventory + charge card") requiring sagas for what a monolith did with `BEGIN...COMMIT`. Correction: decompose by business capability (Checkout, Fulfillment), each owning all data it needs to complete its job.
- **Shared "common" library as coupling superspreader.** `common-utils` that every service imports, containing domain types, grows until any change forces lockstep upgrades of everything — a compile-time distributed monolith. Correction: share only truly stable, generic code (logging, tracing shims); duplicate domain types per service and translate at boundaries (yes, really — that duplication is the decoupling).
- **Premature dependency inversion everywhere.** Interfaces with one implementation, factories for everything, DI configuration longer than the code. The tell: to find what actually runs you need the debugger. Correction: invert at the 2–4 boundaries you named in your architecture bet; concrete calls everywhere else.
- **Event-driven as default instead of as tool.** Making internal module communication event-based "for decoupling" trades legible call graphs for invisible ones: now nobody can answer "what happens when an order is placed" without grepping subscribers, ordering bugs appear, and workflows have no single place to read. Correction: commands/calls for workflows that must complete (orchestration, one readable function); events only for genuinely open-ended fan-out where the emitter must not know consumers (analytics, cache invalidation, integrations).
- **Deciding architecture by resume or by conference talk.** The tell: the proposal names technologies before naming change vectors, or cites a FAANG practice without the FAANG constraint (their 200-team org problem is not your 8-person problem). Correction: force the ADR discipline — context and options first; a proposal that can't articulate its revisit trigger isn't ready.
- **Skipping the "what gets worse" analysis.** Every real architectural decision makes something worse (splitting a service worsens latency and debuggability; adding a cache worsens consistency and adds an invalidation bug class). A proposal that lists only benefits hasn't been analyzed, it's been advocated. Always produce the costs column yourself.
- **Confusing "we might need it" with "we will need it."** Speculative generality: plugin systems with one plugin, multi-tenancy scaffolding for one tenant, abstraction over cloud providers you'll never leave. The carrying cost is paid daily; the payoff needs the future to cooperate. Correction: make the *seam* cheap (keep the code cohesive so extraction stays easy) instead of building the *mechanism* now.

## Worked micro-example: split or not?

Team: 6 engineers. Django monolith: e-commerce, p95 350ms, deploys 3×/week, pain: image-processing jobs (thumbnailing, ~40% of CPU) cause latency spikes; also "checkout code is spaghetti."

Expert reasoning:
1. Two complaints, different remedies — don't let one justify the other. Spaghetti checkout is a *cohesion* problem; a network boundary would freeze the spaghetti behind an API and make refactoring harder. Fix: carve a `checkout/` module in-repo, enforce with `import-linter` contract (only `checkout.api` importable from outside), refactor internally.
2. Image processing has a real forcing function (asymmetric resource profile, latency isolation) *and* the cheap-extraction shape: async, few call edges, owns no shared tables. But the minimal fix is even cheaper — move it to a worker queue (Celery + separate worker fleet) in the same codebase. Separate *deployment/scaling unit*, same repo, no new API contract. Microservice benefits at ~10% of the cost.
3. ADR: decision "extract image work to async workers, keep single codebase"; rejected "image microservice" (new contract + repo + on-call for 6 people, no independent-team forcing function); revisit trigger "if a second team owns media, or worker code needs a runtime the monolith can't host (e.g., GPU)."
4. Outcome shape: latency spikes gone via isolation, checkout refactorable at IDE speed, zero new network contracts. The expert move was noticing that "microservices?" was the wrong question for both pains.

## Self-check before presenting an architecture recommendation

- Did I state the change-vector bet explicitly ("we expect X to change, not Y") and does every boundary trace to it? If a boundary exists "for cleanliness," cut it.
- Replay test: take 5 recent (or plausible) changes and walk each through the proposed structure — count modules/services touched per change. Median >1 means the boundaries fight the change vectors.
- Costs column present? For each element: what it makes worse, its availability/latency math if it adds hops, and its carrying cost if change never comes.
- Reversibility check: for each decision, is it a door that locks behind you (datastore, public API, service split) or a two-way door (internal structure)? One-way doors get ADRs and extra scrutiny; two-way doors get decided fast and cheap.
- Would this survive the team doubling *and* the team halving? Architectures that require heroics (a 3-person team running 12 services) or that bottleneck growth (one module all 12 engineers edit daily) fail this.
- Is every stated rule enforceable by a CI check I can name? If not, either name the tool or expect the rule to erode.
