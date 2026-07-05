---
name: platform-engineering
description: Load when designing internal developer platforms, golden paths, or self-service infrastructure; evaluating IDP tooling (Backstage, Port, etc.); structuring platform vs product teams; measuring platform adoption/success; or advising whether an org should build a platform team at all.
---

# Platform Engineering

## Core mental model

- **A platform is a product whose customers can leave.** Developers are not captive users; they route around platforms that slow them down (shadow infra, "temporary" exceptions that never die). Everything follows from this: you need user research, onboarding UX, docs, support SLAs, versioning, and a deprecation policy — because your alternative isn't "they comply," it's "they defect and now you have N+1 stacks."
- **Golden path, not golden cage.** The winning posture is a *paved road*: the supported way is so much easier that people choose it, while escape hatches remain for the 5% with real divergent needs. Mandates invert the incentive — teams spend effort escaping instead of adopting, and the platform team spends effort policing instead of improving. The test: if you removed the mandate tomorrow, would anyone still use it?
- **Cognitive load is the metric under all the metrics.** The platform's job is to shrink what a stream-aligned team must *know and hold in their heads* to ship: Kubernetes internals, IAM policy grammar, TLS renewal, artifact signing. Every platform decision should be judged by "what can a product team now safely not know?" — and its inverse, "what did we just force every team to learn?"
- **Abstractions leak on a schedule; design for the day they do.** Some day a team's deploy will fail *inside* your abstraction. If the platform is opaque, that team is helpless and files a ticket (you've become the bottleneck you were built to remove). Design **progressive disclosure**: simple interface on top, inspectable layers underneath (show the generated manifests, expose the raw logs, document the escape hatch to the layer below).
- **Platforms are leverage, and leverage has a minimum scale.** A platform team is an investment that pays off multiplied across many product teams. Below that multiplier — few teams, pre-product-market-fit, one product — the platform is a cost center building for imaginary future users.

## Should this org build a platform (team) at all? — the reasoning chain

Ask in order; be willing to answer "no":

1. **How many stream-aligned teams exist?** Under ~4–5 teams, "the platform" is a wiki page, some shared Terraform modules, and CI templates owned part-time. A dedicated platform team this early burns senior engineers building for users who don't exist yet — the premature-platform trap. Pre-PMF startups need product iteration speed, not internal products.
2. **Is there repeated, painful, *convergent* toil?** Platform opportunity = many teams solving the same problem badly and similarly. If every team's needs genuinely diverge, a platform would be a straitjacket; fix that with an enabling team (coaching) instead.
3. **Does leadership accept product discipline for it?** A "platform team" that's actually a ticket-driven ops team with a rebrand fails predictably. If there's no appetite for roadmaps, user research, and saying no, don't start.
4. **Build vs buy the foundation?** Default: buy/adopt the commodity layers, build only the thin opinionated glue that encodes *your* org's choices. Your platform's differentiation is its opinions, not its plumbing.

## The capability stack — build vs buy per layer (as of 2026)

Order of typical value delivery (start at the top):

| Layer | What good looks like | Build/buy judgment |
|---|---|---|
| CI/CD templates | Versioned reusable pipelines (e.g. GitHub reusable workflows) teams adopt in <1 day | Build the templates on a bought CI system; this is usually the highest-leverage first product |
| Scaffolding | `create-service` generating repo + pipeline + deploy + observability, working deploy in <1 hour | Build the templates; the generator itself comes with your portal or a simple CLI |
| Deployment abstraction | A `service.yaml`-grade interface hiding k8s/cloud specifics, with generated output inspectable | Build thin over bought runtime (managed k8s, or PaaS-like layers); this is where opinions live. Alternatively buy the whole runtime (internal PaaS products) if your needs are vanilla |
| Observability defaults | Dashboards, alerts, SLOs auto-provisioned per service; logging/tracing on by default | Buy the stack (Datadog/Grafana/etc.), build the per-service auto-provisioning |
| Secrets & identity | Workload identity (OIDC, no long-lived keys), secret manager integration in scaffold | Buy entirely; build only the paved-road wiring |
| Portal/catalog | Service ownership, docs, self-service actions in one place | **Backstage** (CNCF) if you can staff it — realistic cost is 2–4 dedicated engineers ongoing, which is why managed Backstage (e.g. Roadie) and SaaS portals (Port, Cortex, OpsLevel) have taken large shares of this market as of 2026. Buy unless portal customization is genuinely strategic |

Sequencing judgment: teams commonly start with the portal because it demos well — usually wrong. A catalog of services nobody can easily create or deploy is a museum. Start where the pain is (almost always CI/CD + deploy), add the portal when there's something to catalog.

**What one golden path concretely contains** (the checklist for "is our scaffold real?"): `create-service my-api --template=node-service` produces a repo with working CI (build/test/scan on PR, deploy on merge, all via the org's versioned reusable workflows), a `service.yaml` deploy contract, a running dev/staging deployment by the end of the command, dashboards and paging alerts wired to `team:`, workload identity to cloud resources (zero copied credentials), structured logging and tracing on by default, an on-call runbook stub, and catalog registration as a side effect. If any of those steps is "then file a ticket," the path isn't paved yet — the ticket is the roadmap item.

## Portal selection — the reasoning chain (as of 2026)

When the portal layer's time genuinely comes, ask in order:
1. **What must it do on day one?** Rank: ownership catalog, scaffolding ("create-service"), self-service actions, scorecards, docs (TechDocs-style). If the top need is catalog + scaffold, almost any option works and the decision matters less than teams think — timebox it.
2. **Can you staff self-hosted Backstage honestly?** It's a framework, not a product: React/TypeScript plugin development, realistic ongoing cost of 2–4 dedicated engineers. Below that staffing, self-hosted Backstage becomes an abandoned demo — the single most common portal failure as of 2026.
3. **If not: managed Backstage (Roadie-style) vs SaaS portals (Port, Cortex, OpsLevel)?** Managed Backstage keeps the open ecosystem and exit path; SaaS portals win on time-to-value and low-code self-service actions, cost per-seat money and lock-in. Deciding factor is usually whether you need deep custom plugins (→ Backstage lineage) or configuration-level customization (→ SaaS).
4. **Whatever you pick**: the catalog must be populated *mechanically* (from scaffold metadata, repo scanning, cloud tags) — a hand-maintained catalog is stale in a quarter, and a stale catalog is worse than none because people stop trusting all of it.

## Team topologies — who does what

Use the Team Topologies frame precisely, because conflating these roles breaks them:
- **Stream-aligned teams** own products/user journeys end to end. The platform exists to keep their cognitive load inside budget.
- **Platform team** builds and runs the internal product, interacting *as a service* (self-service consumption, docs, SLAs) — not as a gate in other teams' critical paths. If every deploy needs a platform-team human, you built a bottleneck, not a platform.
- **Enabling teams** coach and upskill (e.g. "help teams adopt tracing"), embedding temporarily and *leaving*. Standing "DevOps team that does the DevOps for you" is the anti-pattern both of these get confused with.
- Interaction modes evolve: new capabilities often start in *collaboration* mode with one pilot team, then harden to *X-as-a-Service*. A platform team permanently in collaboration mode with everyone is under-productized; permanently ticket-driven is an ops team.

## Escape hatches — designing the off-road

An escape hatch is a product feature, not a defeat. Design rules:
- **Documented and first-class**: "bring your own manifests" has a how-to page, not a shrug. Undocumented escapes still happen; they're just invisible and unsupported.
- **Partial, not total**: a team ejecting from the deploy abstraction should *keep* CI templates, observability defaults, and workload identity. All-or-nothing hatches force teams who need one divergence to abandon everything — maximum shadow infra for minimum cause.
- **Cost stays visible**: off-road teams own their extra operational load explicitly (their on-call, their upgrade toil). Not punishment — honest pricing; the paved road should win on economics, not access control.
- **Instrumented**: count hatch usage per capability. One team off-road is their special need; five teams off-road on the same capability is your feature gap, discovered for free.
- **Two-way**: returning to the paved road must be cheap, or every temporary divergence becomes permanent.

## Priors an expert carries into any platform conversation

- The stated problem ("we need a portal/mesh/multi-cloud") is downstream of the real one (slow provisioning, unclear ownership, deploy fear) most of the time — go find the timing data before accepting the framing.
- Adoption problems are product problems ~80% of the time and communication problems ~20%; they are almost never solved by mandate, and a mandate hides which one you had.
- The highest-ROI platform work is usually the least glamorous: CI templates, secrets wiring, one-command scaffolds. Distrust roadmaps that lead with catalogs and dashboards.
- Buy beats build for anything a vendor does at scale (portals, observability, secret stores); build wins only for the thin layer encoding org-specific opinions. Teams systematically overestimate their uniqueness.
- Every platform capability has an ongoing maintenance tax around 20–30% of build cost per year; a roadmap that assumes zero maintenance is fiction.

## Platform-as-product discipline

- **User research is mandatory**: watch developers onboard a new service; the friction log is your roadmap. Internal customers are polite to your face and defect quietly — instrument actual usage, don't rely on surveys alone.
- **Adoption is earned per team**: pilot with one friendly-but-real team, fix what bleeds, then expand. Big-bang mandated migrations create hostage users who evangelize against you.
- **Versioning and deprecation policy** are what distinguish a product from a pile of scripts: templates and abstractions carry versions; breaking changes ship with migration tooling (codemods, automated PRs into consumer repos — you have the leverage to run the migration *for* teams; use it) and a support window. A platform that breaks its consumers unannounced trains them to fork and pin — which is how paved roads die.
- **Say no in public**: a visible roadmap with explicit non-goals prevents the platform becoming a wish-list dumping ground. Every bespoke feature for one loud team is generic capacity taken from everyone.

## Measuring platform success

- **DORA metrics of your *consumer* teams** (deploy frequency, lead time, change-failure rate, time-to-restore) — the platform's outcomes live in its customers' numbers, not its own. Segment by adopters vs non-adopters for the most honest signal you can get.
- **Time-to-first-deploy** for a brand-new service (scaffold → prod): the sharpest single platform metric. Days → hours is the promise; measure it quarterly with a real engineer, not a demo.
- **Ticket/interrupt volume** to the platform team, classified: "how do I" (docs gap), "please do for me" (missing self-service), "it's broken" (reliability). Each class has a different fix; the aggregate should trend down per consumer team.
- **Voluntary adoption rate** where alternatives exist — the paved-road test made quantitative.
- Anti-metrics: platform team output (features shipped, services migrated by force) and vanity catalog completeness. A platform can hit 100% "adoption" by mandate while every one of its customers gets slower.

## Failure modes & pitfalls

- **The premature platform**: 20-engineer startup, 3 platform engineers building a Backstage instance and a multi-cluster service mesh. Correction: below ~5 teams, platform = conventions + a few shared modules, owned part-time; revisit at 8–10 teams.
- **The mandate substitute**: adoption enforced top-down because the product isn't good enough to win voluntarily. Symptom: exception requests pile up; teams keep shadow infrastructure "for now." Correction: treat every exception request as user research — it's a feature gap or a fit gap, and either way it's data.
- **The 80-variable abstraction**: the deployment interface grows a passthrough option for every k8s field consumers asked about, until it's kubectl with extra steps and none of the safety. Correction: an abstraction earns its existence by *refusing* to expose most of the layer below; divergent teams get the documented escape hatch (own manifests, platform still provides CI + observability), not another knob.
- **The opaque abstraction**: deploys fail with "platform error 500," no way to see the generated resources or underlying logs. Teams learn helplessness, then hostility. Correction: progressive disclosure as a design requirement — `platform deploy --show-manifests`, links to underlying cloud consoles/log queries in every error, and docs that explain the layer below rather than pretending it doesn't exist.
- **Portal-first theater**: a beautiful catalog, scorecards, and no self-service actions behind any of it. Correction: every portal page should let you *do* something (create, deploy, rotate, scale), not just view; otherwise ship a spreadsheet and spend the engineers on CI.
- **Snowflake golden paths**: five scaffolds (per language) that drift apart, each with its own CI shape. Correction: one paved road with per-language surface layers over shared pipeline/deploy/observability contracts; the contract is the product.
- **Ops team with a rebrand**: platform "team" spends 80% on interrupt tickets, builds nothing, burns out. Correction: measure interrupt load, convert top ticket classes into self-service features, and protect build capacity explicitly (rotation for interrupts, roadmap for the rest).
- **Ignoring the exodus signal**: a team quietly moves to their own AWS account and their velocity *improves*. That's not insubordination; it's a churn event with product feedback attached. Interview them like lost customers.
- **The platform-for-the-builders**: abstractions shaped by what's elegant to *implement* (a beautiful CRD hierarchy, a config DSL) rather than what's easy to *consume*. Tell: the platform team finds it intuitive and every consumer keeps a cheat sheet. Correction: the interface is designed from the consumer's vocabulary ("I have a web service that needs Postgres"), and consumers review interface changes before implementation starts.
- **The v2 rewrite trap**: platform v1 has warts; the team disappears for three quarters building v2 "properly" while v1 rots and trust drains. Correction: platforms earn change tolerance through continuous small migrations they run themselves; if a rewrite is truly needed, it ships strangler-style, one capability at a time, with v1 supported until the last consumer is moved *by the platform team*.
- **Support channel as documentation**: every question answered ad-hoc in Slack, nothing written; the same question costs a platform engineer 20 minutes weekly forever. Correction: answer-once policy — every non-trivial support answer becomes a docs PR or an error-message improvement the same day; track questions-answered-by-docs-link as a win.
- **Wrong-altitude abstraction**: wrapping `kubectl` flag-for-flag (too low — no cognitive load removed, one more layer to debug) or "just push code, we handle everything" for an org with genuinely varied workloads (too high — the exceptions eat the team). Correction: abstract at the level where 80%+ of services are honestly identical; if you can't find such a level, the org isn't convergent enough for that layer yet — paved-road the layers below it instead.

## Deprecation done right — the template

Deprecation policy is where platform-as-product gets tested for real. The sequence that keeps trust:
1. **Announce with a migration path**, not a deadline alone: what replaces it, why, and the automated migration (codemod, bot PR) the platform team provides.
2. **Ship the automation first**: open PRs against every consumer repo; teams review and merge rather than research and rewrite. If you can't automate most of the migration, the replacement may not be ready to deprecate *into*.
3. **Dashboards of remaining consumers**, public, with owners — social proof does most of the chasing.
4. **Support window proportional to blast radius** (a CI template: a quarter; a deploy contract: two-plus), and the old path *keeps working* — degraded velocity of new features, never sudden breakage.
5. **Hard cutoff only at the long tail**, coordinated individually with the stragglers. An org-wide breaking flag-day is a product failure being reframed as a compliance problem.

## Worked micro-examples

**A deployment abstraction contract that respects cognitive load (and shows its work):**

```yaml
# service.yaml — the whole interface a stream-aligned team must know
name: payments-api
team: payments            # drives ownership, alerts routing, cost attribution
runtime: nodejs22
port: 8080
resources: standard       # named tiers (standard/high-mem/burst), not raw requests/limits
scaling: { min: 3, max: 20, target: cpu:70 }
dependencies:
  - postgres: { tier: production }      # provisions DB + injects creds via workload identity
env:
  LOG_LEVEL: info
# escape hatch, by design — not a workaround:
# overlays/production/*.yaml is strategic-merge-patched onto generated manifests,
# and `platform render` prints exactly what will be applied.
```

Design notes an expert would defend: named resource tiers instead of raw numbers (the platform owns right-sizing; teams state intent), `team:` as load-bearing metadata (ownership is the catalog), one documented escape hatch with `render` for progressive disclosure — and everything omitted (probes, PDBs, TLS, sidecars) is a deliberate "teams shouldn't need to know," each with a platform-set default they can inspect.

**A cognitive-load audit, the 30-minute version:** list every question a developer must answer to get a new endpoint to production (Where does CI config come from? How do I get a DB? Who approves IAM? How do I see logs? What's the rollback command?). Sort into: *platform answers it* / *docs answer it* / *tribal knowledge*. The third column, weighted by how often each question recurs, is the roadmap. Rerun quarterly; the metric is the third column shrinking.

**Metrics starter set (pre-registered before building anything):**

| Metric | Source | Target trend |
|---|---|---|
| Time-to-first-deploy (new service, scaffold → prod) | quarterly timed run by a non-platform engineer | days → < 1 day |
| Voluntary adoption (services on paved road / total) | catalog metadata | up, without mandates |
| Platform interrupt tickets per consumer team per month | ticket labels: how-do-i / do-for-me / broken | down; "do-for-me" → self-service features |
| Consumer DORA: lead time & change-failure rate | CI/CD + incident data, adopters vs not | adopters better and improving |

## How an expert thinks through it: "leadership wants an IDP; 12 product teams; where do we start?"

First, discovery — not tool selection: interview 5 teams, run one real "new service to prod" timing. Findings: 3 weeks to first deploy (ticket-driven IAM, hand-rolled CI per repo, deploy via copied Helm charts), on-call quality wildly uneven, nobody can list service owners. Leadership's stated ask is "a portal like Spotify's"; the evidence says the pain is provisioning and deploy, and a catalog would just index the chaos. Plan: quarter one, ship the paved road for the dominant stack (one scaffold: repo + reusable CI workflow + deploy abstraction + default dashboards + OIDC-based cloud access), pilot with two volunteer teams, target time-to-first-deploy < 1 day. Quarter two, migrate willing teams with automated PRs; adopt a *managed* portal or SaaS (staffing 3 engineers on self-hosted Backstage plugins would consume the whole team — rejected on cost, not capability) seeded from the scaffold's metadata so the catalog is born accurate. Considered and rejected: buying a full internal-PaaS runtime (two teams have genuine low-latency/GPU needs the PaaS can't express — the escape hatch would become the main road); starting with a mandate ("all services in the catalog by Q2" — produces a stale catalog and resentment); service mesh rollout (nobody's pain, someone's fascination). Success metrics set *before* building: time-to-first-deploy, voluntary adoption count, platform ticket volume per team. Stopping rule for v1: two pilot teams shipping through the paved road without platform-team hand-holding for a full month.

## Verification / self-check

Before presenting a platform design or recommendation:
- Can you name the *specific pain*, per evidence (timings, friction logs, ticket classes) — not "developers need a platform" in the abstract?
- Does every abstraction come with its escape hatch and its progressive-disclosure story ("when this breaks, here's how a product dev sees underneath")?
- Is there a number for the leverage math: N consumer teams × hours saved vs platform headcount? If N < 5 or the math needs imaginary future teams, recommend conventions, not a platform team.
- Would a team choose this voluntarily over what they do today? If honestly unsure, the design needs a pilot, not more architecture.
- Does the plan state what the platform team will *not* do, and the deprecation/versioning policy for what it will?
- Stopping rule: the recommendation is done when it names the first paved road, the pilot teams, the buy-vs-build call per layer with reasons, and pre-registered success metrics. Tool shootout matrices beyond the top two candidates are procrastination.
