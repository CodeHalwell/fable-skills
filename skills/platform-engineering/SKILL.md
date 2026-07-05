---
name: platform-engineering
description: Load when designing internal developer platforms, golden paths, or self-service infrastructure; evaluating IDP tooling (Backstage, Port, etc.); structuring platform vs product teams; measuring platform adoption/success; or advising whether an org should build a platform team at all.
---

# Platform Engineering

## Core mental model

- **A platform is a product whose customers can leave** — they route around platforms via shadow infra and "temporary" exceptions. Everything follows: user research, onboarding UX, versioning, deprecation policy.
- **Golden path, not golden cage**: the supported way wins by being easier, escape hatches remain for the real 5%. The test: remove the mandate tomorrow — would anyone still use it?
- **Cognitive load is the metric under all the metrics**: judge every decision by "what can a product team now safely not know?" and its inverse.
- **Design for the day the abstraction leaks**: progressive disclosure — inspectable generated manifests (`platform render`), raw logs, documented drop-down-one-layer hatches — or the first opaque "platform error 500" teaches helplessness, then hostility.
- **Leverage has a minimum scale.** Below ~4–5 stream-aligned teams, "the platform" is conventions + shared modules + CI templates owned part-time; a dedicated team is a cost center building for imaginary users. Revisit at ~8–10 teams.

## Should this org build a platform team? / where to start

Ask in order, willing to answer no: enough teams? repeated *convergent* toil (divergent needs → enabling team, not a platform)? leadership accepts product discipline (roadmap, research, saying no)? buy the commodity layers, build only the thin glue encoding *your org's* opinions — teams systematically overestimate their uniqueness.

Sequencing: **start where the pain is (almost always CI/CD templates + scaffold + deploy), never the portal** — a catalog of services nobody can easily create or deploy is a museum, and portal-first is the most common expensive IDP mistake. The portal comes last, seeded *mechanically* from scaffold metadata/repo scanning/cloud tags (hand-maintained catalogs are stale in a quarter, and a stale catalog is worse than none).

Portal selection when its time comes: self-hosted Backstage is a framework costing 2–4 dedicated engineers ongoing — below that staffing it becomes the abandoned-demo failure that dominates 2026 post-mortems; managed Backstage (Roadie) keeps the ecosystem and exit path; SaaS portals (Port, Cortex, OpsLevel) win time-to-value at per-seat cost and lock-in. Deep custom plugins → Backstage lineage; configuration-level customization → SaaS. Timebox the shootout — at "catalog + scaffold" requirements the choice matters less than teams think.

## The golden path — what "real" means

`create-service my-api` must produce, with zero follow-up tickets: repo + working versioned-reusable-workflow CI, a `service.yaml` deploy contract, a running dev/staging deployment by end of command, dashboards + paging alerts wired to `team:`, workload identity (zero copied credentials), structured logging/tracing on by default, runbook stub, catalog registration as a side effect. Any step that is "then file a ticket" is the roadmap item. Target: new service to prod < 1 day, measured quarterly by a *non-platform* engineer, counting queue/approval/onboarding time and reporting median *and* p90.

## Interface design

Expose intent, not implementation: named resource tiers (`resources: standard`) not raw requests/limits; `team:` as load-bearing metadata; everything omitted (probes, PDBs, TLS) is a deliberate "teams shouldn't need to know" with an inspectable default. The failure poles: wrapping kubectl flag-for-flag (no load removed, one more layer to debug) and "just push code" for genuinely varied workloads (exceptions eat the team). Abstract at the level where 80%+ of services are honestly identical; if no such level exists, the org isn't convergent enough for that layer yet.

## Escape hatches — the five rules

Documented and first-class; **partial, not total** (ejecting from deploy keeps CI, observability, identity — all-or-nothing hatches maximize shadow infra for minimum cause); cost stays visible (off-road teams own their toil — honest pricing, not punishment); instrumented (five teams off-road on one capability = your feature gap, discovered free); two-way (returning must be cheap or every divergence is permanent).

## Team topologies

Platform team interacts *as a service*, not as a gate; enabling teams coach and *leave*; both get confused with the standing shared-services ticket queue. Permanently-in-collaboration-mode = under-productized (the platform is really just people and won't scale past its headcount); permanently ticket-driven = an ops team with a rebrand. New capabilities start in collaboration with one pilot, then harden to X-as-a-Service.

## Product discipline, measurement, deprecation

- Friction logs from watching real onboarding are the roadmap; internal customers are polite to your face and defect quietly — instrument usage, don't survey. Pilot with one friendly-but-real team, fix what bleeds, expand. Say no in public (visible non-goals).
- Metrics: consumer-team DORA segmented adopters vs non-adopters; time-to-first-deploy; interrupt volume classified (how-do-i = docs gap / do-for-me = missing self-service / broken = reliability) trending down per team; **voluntary adoption where alternatives exist — the sharpest single signal**. Anti-metrics: platform output, catalog completeness, portal logins, mandated-adoption %.
- Mandates suppress the market signal ("is it good?" becomes "were they forced?"), produce checkbox compliance plus underground shadow platforms, and atrophy product discipline; reserve them for genuine security/compliance floors only. A team that leaves and gets *faster* is a churn event with product feedback attached — interview them like lost customers; bring guardrails to their account rather than dragging them back.
- Deprecation sequence that keeps trust: replacement ships first at parity (dogfooded); automated migration ships next (bot PRs into consumer repos — the platform absorbs migration cost, always); public dashboard of remaining consumers; support window proportional to blast radius, old path degrading gracefully not breaking; hard cutoff only at the individually-coordinated long tail. An org-wide flag-day is a product failure reframed as compliance.
- Budget ~20–40% of build cost per year as maintenance tax per capability (higher on fast-moving substrates); a roadmap assuming zero maintenance is fiction, and killing capabilities is first-class platform work.

## Failure modes (checklist with tells)

- Premature platform: 3 platform engineers + Backstage + mesh at 20 engineers.
- Portal-first theater: beautiful catalog, no self-service actions behind any page.
- The 80-variable abstraction: kubectl with extra steps and none of the safety — divergent teams get the hatch, not another knob.
- Opaque abstraction: "platform error 500," learned helplessness.
- Snowflake golden paths: five per-language scaffolds drifting; the shared pipeline/deploy/observability *contract* is the product.
- Ops-team-with-a-rebrand: 80% interrupts, nothing built, burnout — convert top ticket classes to self-service, protect build capacity structurally.
- Platform-for-the-builders: elegant CRD hierarchy the platform team finds intuitive while every consumer keeps a cheat sheet — design from consumer vocabulary; consumers review interface changes first.
- v2 rewrite trap: three quarters of silence while v1 rots — strangler-style, one capability at a time, platform team moves the consumers itself.
- Support-channel-as-documentation: answer-once policy — every non-trivial answer becomes a docs PR or error-message fix same day.

## How an expert thinks through it: "leadership wants an IDP; 12 teams"

Discovery before tools: interview 5 teams, time one real new-service-to-prod (finding: 3 weeks — ticket IAM, hand-rolled CI, copied Helm). Leadership asks for "a portal like Spotify's"; evidence says the pain is provisioning/deploy, and a catalog would index the chaos. Q1: paved road for the dominant stack, two pilot teams, TTFD < 1 day. Q2: migrate willing teams via automated PRs; adopt *managed* portal/SaaS (self-hosted Backstage staffing would consume the team — rejected on cost, not capability), seeded from scaffold metadata. Rejected: full internal-PaaS runtime (two teams' GPU/latency needs would make the hatch the main road), catalog mandates, service mesh (nobody's pain, someone's fascination). Metrics pre-registered; v1 stopping rule: two pilot teams shipping unassisted for a month.

## Verification / self-check

- The specific pain is named with evidence (timings, friction logs, ticket classes) — not "developers need a platform."
- Every abstraction has its hatch and its progressive-disclosure story.
- Leverage math is real: N teams × hours saved vs platform headcount, no imaginary future teams.
- Would a team choose it voluntarily today? If unsure, that's a pilot, not more architecture.
- The plan states non-goals and the versioning/deprecation policy.
Stopping rule: first paved road named, pilot teams named, buy-vs-build per layer with reasons, metrics pre-registered. Tool matrices beyond two candidates are procrastination.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)

- Probed 14 claims: 14 baseline, 0 partial, 0 delta — Opus cold reproduced the premature-platform thresholds, Backstage staffing/failure-mode/market shift (Port/Cortex/Roadie), portal-last sequencing, voluntary-adoption-as-sharpest-metric, all five escape-hatch rules, the replacement-first deprecation sequence, intent-not-implementation interfaces, team-topologies distinctions, honest TTFD measurement, mandate consequences, and the exodus-as-churn-signal response.
- File restructured to a compact judgment sheet; explanatory prose cut. Residual value is the assembled review scaffolding (checklists, walkthrough, pre-registered metrics table) and the discipline of demanding evidence before accepting the stated problem — no individual fact here exceeds the Opus baseline.
