---
name: deployment-strategies
description: Load when planning how to ship a change safely — choosing rolling vs blue/green vs canary, designing feature-flag rollouts, sequencing database schema migrations (expand/contract), deciding rollback vs fix-forward, or reviewing deploy pipelines and release processes for blast-radius and reversibility problems.
---

# Deployment Strategies

## Core mental model

- **Deploy ≠ release.** Deploy = new code is running in production. Release = users experience new behavior. Every mature shipping practice — flags, canaries, dark launches — is a way of pulling these apart so that the risky event (code lands) and the observable event (behavior changes) can be controlled independently. If your only way to change user-visible behavior is a deploy, every product decision is coupled to your riskiest operation.
- **Optimize for MTTR, not for never failing.** You cannot test your way to zero bad deploys; you can engineer the time-to-undo down to minutes. Every deployment decision should be scored on three axes: blast radius (how many users see the failure), detection time (how fast you notice), and reversal time (how fast you can undo). A strategy that halves blast radius but makes rollback take an hour is usually a bad trade.
- **The database is the hard part; everything else is a routing problem.** Stateless code can run two versions side by side and switch traffic freely. State cannot — a schema, a queue message format, a cache entry outlives the deploy that created it. Whenever a plan looks clean, ask "and what about the data?" That question kills more naive deploy plans than any other.
- **Rollback must survive the rollout.** The rule that makes everything else work: at every moment during a rollout, the *previous* version must still function against the *current* state of the world (schema, config, queued messages, cached objects). N and N-1 compatibility is not a nicety; it is what makes rollback a button instead of an incident.
- **Config changes are deploys.** A huge share of major outages are triggered by configuration, not code — as of 2025–2026 the pattern is vivid: Cloudflare's Nov 18, 2025 global outage came from a database permission change that doubled a bot-management config file's size; its Dec 5, 2025 outage from a global config kill-switch flip hitting a latent proxy bug; Azure's multi-day East US2 outage from a networking config change. Config gets none of code's safeguards by default (no canary, no CI, instant global propagation) while having the same power to take everything down. Give config the full pipeline: version control, review, staged rollout, automatic validation of size/shape before propagation, and one-command revert.

## Choosing a strategy: the reasoning chain

Ask these questions in order; each answer prunes options.

1. **"What happens to state during and after?"** (Ask this FIRST, not last.) If there's a schema change, message-format change, or cache-format change, the deploy strategy is constrained by the expand/contract sequence below regardless of what you pick for traffic-shifting.
2. **"How fast do I need to undo, and how much traffic can see a failure?"**
   - Internal tool, low stakes → plain rolling update. Kubernetes gives it to you free (`maxSurge`/`maxUnavailable`); rollback = redeploy previous image, minutes. Its weakness: during rollout both versions serve traffic anyway (so you need N/N-1 compat regardless — you never escape that requirement), and blast radius grows linearly until someone notices.
   - Need instant, total reversal (payments cutover, big-bang framework upgrade) → blue/green. Two full environments, flip the router, flip it back in seconds. Costs: 2× capacity during the window, and it's all-or-nothing — 100% of users hit the new version at once, so it has the *worst* blast radius if the flip is wrong and the *best* reversal time. Also: long-lived connections and in-flight jobs don't "flip" — you need draining.
   - Want failures caught on a sliver of traffic → canary. Route 1% → 5% → 25% → 100% with evaluation gates between steps. Best blast-radius control; slowest to complete; requires per-version metrics (you must be able to compare canary vs baseline — if your metrics can't be split by version, a canary teaches you nothing).
3. **"Can I judge the canary?"** A canary needs enough traffic to produce a statistically meaningful signal in an acceptable time. 1% of 100 rps is 1 rps — a rare error mode won't show up for hours. Low-traffic services often do better with blue/green plus good smoke tests than a canary that provides false confidence.
4. **"Is the risky part actually the code path, or the behavior?"** If the risk is a new *feature's* behavior (not infra-level correctness), don't encode the rollout in the deploy topology at all — deploy dark behind a feature flag and do the percentage rollout at the flag layer, where reversal is a config flip measured in seconds and targetable per-user/per-tenant.

What changes the decision: session affinity requirements push toward blue/green; expensive stateful warm-up (big JVM heaps, local caches) pushes away from blue/green's instant flip; multi-region setups usually canary by region ("one region is the canary") because region-sized blast radius with region-sized rollback is a good trade.

## Feature flags: decoupling done with discipline

- Flags move release control from the deploy pipeline to a runtime config plane: percentage rollouts, targeting (employees → beta cohort → 1% → 50% → 100%), and instant kill switches. As of 2026, OpenFeature (CNCF incubating) is the vendor-neutral SDK standard — prefer coding against it over a vendor SDK directly so the provider (LaunchDarkly, Unleash, Flagsmith, Flipt) is swappable.
- **Every flag is debt with a deletion date.** A flag is a fork in your codebase tested in one branch. Ten stale flags = up to 2^10 configuration states nobody has ever run together. Discipline: every flag gets an owner and a removal ticket *at creation*; release flags die within weeks of 100% rollout; distinguish release flags (short-lived, delete after rollout) from ops kill-switches (long-lived, deliberately kept, periodically *tested* — an untested kill switch is a rumor) from entitlement flags (permanent, but those are product configuration, not flags — move them out of the flag system).
- Bucketing must be sticky (hash of user ID, not random per request) or users flap between behaviors and your metrics are garbage. Evaluate flags server-side for anything security- or correctness-relevant; client-side evaluation is a UX optimization, never an authorization boundary.
- Flag *changes* are config deploys (see above): audit-logged, reviewable, and ideally staged. "Someone flipped a flag to 100% at 4:55pm Friday" is a classic incident opener.

## Database migrations: expand / migrate / contract

The iron law: **the application must work with both the old and new schema at every step**, because deploys are not atomic (old and new code run concurrently during any rollout) and because rollback means old code returns against the new schema.

Renaming `users.email` → `users.email_address`, done properly:

1. **Expand** — add `email_address` (nullable, no default that rewrites the table). Deploy code that *writes both* columns, *reads old*. Old code still works: the new column is invisible to it. Rollback-safe.
2. **Backfill** — copy old → new in small batches (thousands of rows, sleep between batches, key-range iteration — never one giant `UPDATE`, which locks and bloats). Idempotent and resumable: it WILL be interrupted. Throttle on replication lag.
3. **Migrate reads** — deploy code that reads new, still writes both. Verify with a comparison sample (read both, log mismatches) before trusting.
4. **Contract** — only after the previous step has soaked and no rollback to old-reading code is plausible: stop writing old, then drop the column *in a later deploy still*. Contract is the only irreversible step, so it goes last and alone.

Each step is separately deployable and separately rollback-able. Yes, it's five deploys instead of one. That is the price of never being in a state where rollback is impossible.

Corollaries:
- **You can roll back code; you cannot roll back data.** Writes made by the new version don't un-happen. If v2 wrote rows in a new format and you roll back, v1 must tolerate them (ignore unknown fields, handle new enum values) or you've converted a bad deploy into data corruption. This is why "just restore the backup" is not a rollback strategy — it's data loss for everything written since.
- Same law applies to **queues and events**: a consumer must be deployed to accept format N+1 *before* any producer emits it. Schema-registry compatibility modes (e.g., Avro/Protobuf `BACKWARD` in Confluent Schema Registry) mechanize this; without a registry, enforce it in review.
- Watch for migrations that lock: adding an index without `CONCURRENTLY` in Postgres, adding a column with a volatile default on older MySQL, any `ALTER` that rewrites a huge table. Test migration duration against production-sized data, not the 50-row dev DB.

## Canary analysis: what gates promotion

- Gate on a small set of symptoms compared *canary vs. baseline at the same time* (never canary-now vs. last-week): error rate (5xx and app-level), latency p95/p99, and 1–3 domain metrics that would catch a silently-wrong result (orders created, login success rate, messages delivered). Resource metrics (CPU, memory) are secondary — include memory growth to catch leaks that symptoms won't show in a 30-minute window.
- Run a **baseline control**: route the same small slice of traffic to a fresh deployment of the *old* version alongside the canary. This cancels out "new pods are cold / on different nodes" effects that otherwise produce false alarms. This is how automated canary judges (e.g., Spinnaker's Kayenta lineage, Argo Rollouts `AnalysisTemplate`s querying Prometheus) are designed to work.
- Automated analysis beats eyeballs for *not skipping the check at 6pm*, but automated judges are conservative pattern-matchers: they catch regressions in the metrics you named, and nothing else. Keep a human on the hook for novel weirdness, and keep the metric set small — a judge watching 40 metrics fails a healthy canary constantly (multiple-comparisons problem) and gets turned off, which is worse than no judge.
- Minimum soak per step: long enough to cover the slowest feedback loop you care about (cache TTLs expiring, hourly crons, memory leak slope). A 5-minute canary catches crashes, not leaks.

## Rollback-first culture

The decision rule under pressure: **if a rollback is available and not ruled out, roll back first and diagnose second.** Rollback is a known-good state reachable by a rehearsed mechanical action; fix-forward is an unrehearsed change written by a stressed engineer with partial understanding — the base rate of first-attempt fixes actually fixing the issue is poor, and each failed fix-forward resets the clock. Choose fix-forward only when rollback is genuinely impossible (irreversible data written, security fix that must stay, the bad deploy is 3 days old and 40 changes are stacked on it — which is an argument for deploying more often, not for fix-forward culture).

Know your rollback-killers *before* you need the button:
- **Schema**: contract steps taken too early; new code wrote data old code can't read.
- **Queues**: new-format messages already enqueued that old consumers will poison-loop on.
- **Caches**: new code cached objects in a new shape with long TTLs; old code deserializes garbage. (Version your cache keys — `user:v2:{id}` — so rollback simply misses instead of exploding.)
- **Clients**: mobile apps and third-party integrations that saw the new API can't be rolled back at all. Anything client-visible is effectively a one-way door; flag it server-side instead.
- **Never rolled back = can't roll back.** Rehearse: roll back a healthy deploy quarterly and time it.

## How an expert thinks through it

*"We need to ship the new pricing engine — new `price_quotes` table, new calculation logic, replaces the inline calculation in checkout."*

First question: state. New table, and checkout writes quotes. So old and new code will coexist against the same DB during any rollout — fine, new table is additive (expand). Rollback story: if we revert, orphaned rows in `price_quotes` — harmless, clean up later. Good.

Real risk: not crashes — *wrong prices*. A canary on error rate won't catch a 10%-too-cheap quote; nothing 500s. Reject "canary and watch the dashboards" — the failure mode is silent. So: dark launch. Deploy the engine behind a flag in *shadow mode* — compute both old and new price on every checkout, serve old, log disagreements with inputs. Zero user risk, production-realistic comparison data. Considered running it only in staging and rejected it: pricing bugs live in weird real-world carts, coupons, currencies that staging doesn't have.

After a week: mismatches down to a known, accepted set (rounding rule change, approved by finance). Now flip *serving* via the flag: employees → 1% sticky by user → watch conversion rate and average order value against control, not just errors → 25% → 100%. Deploy strategy for the code itself: plain rolling — the code was already live in shadow, the deploy is not the risky event anymore; the flag flip is, and it reverses in seconds. That inversion — moving risk from the deploy to a reversible flag — was the whole design.

Stopping rule: shipped at 100%, business metrics stable for two weeks → delete the old calculation path and the flag (removal ticket was filed the day the flag was created). Leaving both paths "just in case" means every future pricing change is implemented twice or diverges silently.

## Failure modes & pitfalls

- **Running `DROP COLUMN`/`RENAME` in the same deploy as the code change.** During the rollout, old pods query the missing column and 500. Rename is the worst: there is no moment when one name satisfies both versions. Always expand/contract; never rename in place — add new, backfill, drop old.
- **Backfill as one big `UPDATE users SET ...`.** Locks the table (or creates massive MVCC bloat/replication lag), times out, and is non-resumable. Batch by primary-key range, sleep between batches, make it idempotent, monitor replica lag while it runs.
- **Adding a Postgres index without `CONCURRENTLY`** (or doing it *inside a transaction*, where `CONCURRENTLY` is disallowed) — takes a lock that blocks writes for the whole build on a big table. Migration frameworks default to transactional migrations; you must explicitly opt out for concurrent index builds (e.g., `disable_ddl_transaction!` in Rails, `atomic = False` in Django).
- **Canary judged against last week's baseline** instead of a live control at the same traffic — every diurnal pattern becomes a false regression or masks a real one.
- **A 1% canary for a change whose failure mode is aggregate** (connection-pool exhaustion at the DB, cache stampede, thundering-herd on a dependency). The canary passes at 1% because the damage scales with total load; it detonates at 100%. For capacity-shaped risks, canary the *load* (region by region) not the request percentage, and watch the shared dependency's metrics, not the service's.
- **Feature flag as authorization.** Flag says premium users get the export endpoint; the endpoint checks the flag but not the entitlement; flag misconfiguration = free features for everyone (or worse, data exposure). Flags gate *availability*, authz gates *permission* — separate checks.
- **Kill switch that was never exercised.** The day you need it, it turns out the "off" path was broken by a refactor eight months ago. Any long-lived operational flag must be flipped in production (briefly, deliberately) on a schedule, or in game days.
- **Blue/green with a shared database treated as "fully isolated environments."** The DB is shared state; a green-version migration that breaks blue kills your instant-rollback story — expand/contract discipline still applies. Blue/green isolates *compute*, nothing else.
- **Draining ignored.** Flipping the LB while websockets/long polls/in-flight jobs are live on blue severs them all at once. Set `terminationGracePeriodSeconds` and preStop hooks realistically; drain queue consumers (stop-consume, finish in-flight, then terminate) before killing pods.
- **Config propagated globally in one shot.** Code gets a canary; then someone pushes a routing rule / WAF rule / flag to every region simultaneously. Stage config like code: one region → soak → rest. And validate config *shape and size* at generation time — Cloudflare's Nov 2025 outage was a config file that silently doubled in size past a consumer's limit.
- **Deploy freezes as a safety strategy.** Long freezes batch up changes, and change size is the dominant risk factor — the first deploy after the freeze is the year's riskiest. Prefer small, frequent, always-rollbackable deploys with good gates; use short freezes only around genuinely critical windows (Black Friday), paired with a fast-track exception process, because a freeze with no exception path just means undocumented hotfixes.
- **"We'll fix forward, rollback loses today's data."** Interrogate this claim: usually only the *schema* moved forward, and code rollback is fine against the expanded schema. People conflate "roll back the code" with "restore the database" — the whole point of expand/contract is that you almost never need the latter.

## Verification / self-check

Before blessing any deploy plan, answer these; a blank is a gap:
1. During rollout, both versions run concurrently — does each work against the shared DB, queues, caches? Name the specific mixed-version interaction you checked.
2. What is the *rehearsed* undo action, who can execute it, and how long does it take? ("Redeploy previous image, anyone on-call, ~4 min" is an answer; "revert the PR and wait for CI" is a 45-minute answer pretending to be one.)
3. What metric distinguishes healthy from broken, is it split by version, and would it catch a *silently wrong* result — not just errors?
4. Is anything a one-way door (dropped column, new-format messages emitted, client-visible API change)? Each one-way door goes in its own late-stage deploy.
5. Does every new flag have an owner and a removal ticket?

Stopping rule: a deploy is *done* when it's at 100%, the soak window covering your slowest feedback loop has passed, the old path and release flags are deleted, and the contract migration has run. Until the cleanup ships, you're still mid-deploy — schedule it, don't hope for it.
