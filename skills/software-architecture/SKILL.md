---
name: software-architecture
description: Load when decomposing a system into modules or services, deciding monolith vs microservices, drawing module/team boundaries, evaluating or writing architecture decision records, choosing dependency directions, or judging whether an abstraction/layer is worth its cost. Also load for "should we split this service" and "how do we structure this codebase" questions.
---

# Software Architecture

## Core mental model

- **Coupling vs cohesion is the only real rule; everything else is a corollary.** Every named principle (SRP, layered, hexagonal, microservices) is a strategy for one goal: things that change together live together; things that change independently can be changed independently. When a pattern and this goal conflict, the pattern loses.
- **Draw boundaries along rate-of-change and reason-for-change lines, not noun lines.** The classic mistake is decomposing by data shape (UserModule, OrderModule) when the actual change vectors are "pricing rules change weekly, tax logic changes yearly, UI changes daily." A boundary is good iff a typical change lands on one side of it. Test any proposed decomposition by replaying the last 10 real changes: how many would have touched one module vs three?
- **Dependencies must point from volatile toward stable.** Business rules shouldn't import the database driver; the database adapter should implement interfaces the business rules own. Dependency inversion is not ceremony — it's the mechanism that lets a stable core survive infrastructure churn. But invert only at boundaries that plausibly churn: wrapping the language stdlib behind interfaces is cargo cult.
- **Architecture is a bet on which changes will come.** You cannot make everything easy to change; every flexibility purchased in dimension A adds indirection cost in dimensions B through Z. State the bet explicitly ("we expect to swap payment providers; we do not expect to swap databases") — that sentence *is* the architecture; the boxes are its consequence.
- **The unit of decomposition cost is a network hop plus a team boundary.** A function call is nanoseconds and refactorable by an IDE. A service call is milliseconds, partial failure, versioned contracts, distributed tracing, and a meeting to change anything. Never pay service prices for module benefits.
- **Good architecture maximizes decisions deferred.** The mark of a strong design is how much it *doesn't* decide yet: keep the core ignorant of delivery mechanism and storage so those choices stay cheap to revisit. Defer with cohesion (code that's easy to extract later), not with speculative mechanism (plugin systems for plugins that don't exist).

## Decision frameworks

### Monolith vs microservices economics
- **Default: modular monolith.** One deployable, strict internal module boundaries enforced by build tooling (import linting, visibility rules — not convention). You get boundary benefits at function-call prices. Correct boundaries can later be extracted cheaply; wrong boundaries in a monolith cost a refactor, while wrong boundaries across services cost a distributed refactor plus a data migration plus an API deprecation cycle.
- Microservices pay only when a *specific* forcing function exists, evaluated per service:

  | Forcing function | Why it pays |
  |---|---|
  | Independent scaling with 10x+ asymmetric load (e.g., ingest vs admin UI) | Scaling the monolith means paying for the max of all needs on every node |
  | Genuinely different runtime needs (GPU inference, different language, real-time constraints) | One deployable physically can't serve both |
  | Organizational: more than ~4 teams shipping on conflicting cadences; the deploy queue is the bottleneck | Conway's law is real; deploy independence is the actual product of microservices |
  | Hard fault-isolation or compliance boundary (PCI scope, tenant isolation) | The process boundary is itself the requirement |

- Non-reasons, and the rebuttals to give:
  - "It's more scalable" — monoliths scale horizontally fine until shared state says otherwise.
  - "Cleaner separation" — that's what module boundaries are for, at 1/10 the cost.
  - "Industry best practice" — the practice at 200-team scale is not the practice at 8 engineers.
  - A team under ~10 asking for microservices: the answer is no, recorded in an ADR with an explicit revisit trigger.
- **Extraction order when you do split:** extract the piece with the fewest synchronous call edges and clearest data ownership first — usually async workers (email, exports, media processing). Never extract the entity everything joins against (usually "user" or "order") first; you'll turn every request into a distributed join.

### Where to put a boundary — quick tests
- **Rate-of-change test:** list the last 10 changes; a good boundary means ≥8 touched one side only.
- **Interface-to-implementation ratio:** a boundary is real if its interface is much smaller than its implementation (a deep module). A "layer" where every method is a one-line pass-through to the layer below is negative value — delete it.
- **Data ownership test:** exactly one module writes each table/aggregate. Two services writing one table is not a boundary; it's a shared mutable global with extra steps.
- **Circular check:** if A needs B and B needs A, they are one module — merge them, or extract the shared piece C both depend on. Never "fix" a cycle with events-as-disguised-calls (A emitting an event that exists only so B can call back).
- **Deletion test:** could you `rm -rf` the module and know exactly what breaks from its interface alone? If the answer requires reading its internals, the boundary leaks.

### Abstraction cost model
- **Wrong abstraction > duplication (in cost).** Duplication is a linear tax you can pay down anytime; a wrong abstraction shared by N callers costs a parameter per divergence, each flag doubles the config space, and the callers get welded together. When you find yourself adding a boolean to a shared helper so caller 3 can behave differently: stop, inline the helper into the callers, let them diverge, re-abstract later from real duplicates.
- **Rule of three, with a twist:** abstract on the third *confirmed identical-for-the-same-reason* duplicate. Two pieces of code that look alike but change for different reasons (same-shaped validation on two unrelated forms) are *coincidental* duplication — merging them creates coupling between unrelated change vectors, the exact disease architecture exists to prevent.
- **Indirection budget:** each layer must earn its keep by absorbing a change you actually expect.
  - Earns it: repository interface over the ORM, if you fake storage in tests or plausibly swap engines.
  - Doesn't: `ServiceImpl` behind `ServiceInterface` with exactly one implementation forever, wired by a DI framework — pure reading tax; every "what does this call" question now needs the debugger.
- **Cheap seams over built mechanisms:** the low-cost way to keep options open is cohesion (the candidate module's code all lives together, communicates through a narrow neck) rather than building the plugin/multi-backend mechanism now. A seam costs nothing until used; a mechanism costs maintenance forever.

### ADRs (architecture decision records)
- Write one whenever a decision is expensive to reverse or will be re-litigated: datastore choice, sync vs async boundary, build-vs-buy, service split, auth model, framework adoption.
- Required content, in order of value:
  1. **Context** — the forces and constraints, with numbers (team size, traffic, latency budget).
  2. **Options considered, with the real reasons the losers lost.** An ADR without rejected options is a press release, not a record.
  3. **Decision** — one sentence, active voice.
  4. **Consequences — including what gets worse.** Every real decision makes something worse; a consequences section with only upsides means the analysis wasn't done.
  5. **Revisit trigger** — "if p99 exceeds X," "if team count exceeds N," "if we add a second tenant type."
- Mechanics: immutable, numbered, in-repo (`docs/adr/0007-use-postgres-outbox.md`). A superseding decision links back rather than editing. The archaeological payoff — "why is it like this?" answered in 2 minutes instead of 2 days — is the entire point.

### Evolutionary architecture
- Prefer reversible steps over big designs:
  - **Strangler fig** around legacy: route traffic incrementally to the new thing; never big-bang rewrite (the rewrite must chase a moving target while providing zero value until 100% done).
  - **Branch by abstraction** for in-place replacement: introduce a seam → move callers onto it → swap the implementation behind it → remove the seam. Trunk stays releasable throughout.
  - **Parallel run with diffing** before cutover for anything correctness-critical: run old and new, compare outputs on real traffic, cut over on measured agreement.
- Encode architectural rules as **fitness functions** — executable checks in CI: import-boundary linting (`import-linter` in Python, ArchUnit in Java, `dependency-cruiser` in JS), dependency-direction tests, "no module may import another's `internal/`." A rule not enforced by tooling erodes within months, and the erosion is silent until it's structural.

## Failure modes & pitfalls

- **Layering by technology instead of by domain.** `controllers/`, `services/`, `models/` as the top-level structure means every feature change touches every directory and no feature can be deleted as a unit. Correction: package by feature (`billing/`, `catalog/`), layers inside each if useful. You should be able to `rm -rf` a feature.
- **The distributed monolith.** Services that must deploy together, share a database, or call each other synchronously in chains (A→B→C→D per request) carry all microservice costs with no benefits. Detection questions: "Can I deploy service B alone on a Friday?" and "What's the availability math?" — four 99.9% services in a synchronous chain compose to ~99.6%, roughly 35 minutes/week worse than any one of them. Correction: merge them back, or make the edges async with each service owning its data.
- **Entity-service decomposition.** UserService, OrderService, ProductService — nouns, not capabilities. Every business operation now orchestrates 3+ services, and cross-service transactions appear ("create order + decrement inventory + charge card") requiring sagas and compensation logic for what a monolith did with `BEGIN...COMMIT`. Correction: decompose by business capability (Checkout, Fulfillment, Catalog), each owning *all* the data it needs to complete its job.
- **Shared "common" library as coupling superspreader.** A `common-utils` package that every service imports, containing domain types, grows until any change forces lockstep upgrades of everything — a compile-time distributed monolith. Correction: share only genuinely stable generic code (logging, tracing shims). Duplicate domain types per service and translate at the boundary — that duplication *is* the decoupling you paid for.
- **Premature dependency inversion everywhere.** Interfaces with one implementation, factories for everything, DI configuration longer than the code it wires. The tell: finding what actually runs requires the debugger. Correction: invert at the 2–4 boundaries named in your architecture bet; concrete calls everywhere else.
- **Event-driven as default instead of as tool.** Making internal module communication event-based "for decoupling" trades a legible call graph for an invisible one: nobody can answer "what happens when an order is placed" without grepping subscribers; ordering and at-least-once bugs appear; workflows have no single place to read. Correction: commands/direct calls for workflows that must complete (one readable orchestration function); events only for genuinely open-ended fan-out where the emitter must not know its consumers (analytics, cache invalidation, third-party integrations).
- **Deciding by resume or by conference talk.** The tell: the proposal names technologies before naming change vectors, or cites a FAANG practice without the FAANG constraint. Correction: enforce ADR discipline — context and rejected options first. A proposal that can't state its revisit trigger isn't ready to be decided.
- **Skipping the "what gets worse" analysis.** Splitting a service worsens latency, debuggability, and local dev setup. Adding a cache worsens consistency and adds an invalidation bug class. Adding a queue worsens end-to-end latency visibility and adds redelivery semantics. A proposal listing only benefits has been advocated, not analyzed — produce the costs column yourself before agreeing.
- **Confusing "we might need it" with "we will need it."** Plugin systems with one plugin, multi-tenancy scaffolding for one tenant, cloud-provider abstraction for a provider you'll never leave. Carrying cost is paid daily; payoff requires the future to cooperate. Correction: keep the seam cheap (cohesion, narrow interface) instead of building the mechanism now.
- **Letting the ORM/framework own the architecture.** When domain objects are ORM classes and business logic lives in controllers, the framework's layering *is* your architecture and its upgrade cycle is your migration cycle. Keep the core domain plain objects; adapt at the edges. This is the difference between "we use Django" and "we are Django."

## Worked micro-example: split or not?

Team: 6 engineers. Django monolith: e-commerce, p95 350ms, deploys 3×/week. Pains: (a) image-processing jobs (thumbnailing, ~40% of CPU) cause latency spikes; (b) "checkout code is spaghetti."

1. **Two complaints, different remedies — don't let one justify the other.** Spaghetti checkout is a *cohesion* problem; a network boundary would freeze the spaghetti behind an API contract and make refactoring harder, not easier. Fix: carve a `checkout/` module in-repo, enforce with an `import-linter` contract (only `checkout.api` importable from outside), refactor internally at IDE speed.
2. **Image processing has a real forcing function** (asymmetric resource profile, latency isolation) *and* the cheap-extraction shape: async, few call edges, owns no shared tables. But the minimal sufficient fix is cheaper than a service: move it to a worker queue (Celery + a separate worker fleet) in the same codebase. Separate scaling/deployment unit, same repo, zero new API contracts. Microservice benefits at ~10% of the cost.
3. **Availability math check:** the worker split adds no synchronous hop (queue in between), so request-path availability is untouched. A synchronous image-service call would have added one.
4. **ADR:** decision "extract image work to async workers, keep single codebase"; rejected option "image microservice" (new contract + repo + on-call surface for 6 people; no independent-team forcing function); consequences "worker deploys now separate; queue adds at-least-once semantics — thumbnailing must be idempotent"; revisit trigger "a second team owns media, or the work needs a runtime the monolith can't host (GPU)."
5. Outcome shape: latency spikes gone via isolation; checkout refactorable in-place; zero new network contracts. The expert move was noticing that "microservices?" was the wrong question for *both* pains.

## Self-check before presenting an architecture recommendation

- Did I state the change-vector bet explicitly ("we expect X to change, not Y"), and does every boundary trace back to it? Any boundary existing "for cleanliness" gets cut.
- Replay test: walk 5 recent (or plausible) changes through the proposed structure and count modules/services touched per change. Median >1 means the boundaries fight the change vectors.
- Costs column present? For each element: what it makes worse, its availability/latency arithmetic if it adds hops, and its carrying cost if the anticipated change never comes.
- Reversibility sort: is each decision a one-way door (datastore, public API shape, service split, language) or a two-way door (internal structure, naming, most library choices)? One-way doors get ADRs and extra scrutiny; two-way doors get decided fast and revisited cheaply. Spending a month deciding a two-way door is itself an architecture failure.
- Would this survive the team doubling *and* halving? Architectures requiring heroics (3 people running 12 services) or bottlenecking growth (one module all 12 engineers edit daily) both fail.
- Is every stated rule enforceable by a CI check I can name (`import-linter`, ArchUnit, `dependency-cruiser`, schema-compat linter)? If not, either name the tool or expect the rule to silently erode.
- Does the data-ownership map have exactly one writer per store? Any shared-write store is an undeclared merge of two "separate" components.
