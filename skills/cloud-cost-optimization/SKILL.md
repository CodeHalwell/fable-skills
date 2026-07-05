---
name: cloud-cost-optimization
description: Load when analyzing or reducing cloud spend — bill investigation, FinOps practices, reserved instances/savings plans strategy, spot/preemptible usage, egress and data-transfer costs, storage lifecycle, tagging and cost allocation, unit economics, or GPU/LLM cost management on AWS/Azure/GCP.
---

# Cloud Cost Optimization (FinOps Engineering)

## Core mental model

- **Compute mistakes self-correct when instances die; data mistakes compound monthly forever.** The creeping bill-killers are per-GB charges on *movement and retention* — costs proportional to data flows nobody drew on the diagram. Price every arrow.
- **Unit economics before optimization:** cost-per-request/customer/run against a written, never-silently-changed denominator. A rising bill with falling unit cost is a business succeeding; a flat bill with rising unit cost is a slow leak. Present every saving in both currencies ($/mo and unit-cost delta) — the first gets budget, the second proves it wasn't traffic decline.
- **Allocation before optimization** — an unattributed bill produces meetings, not savings.
- **The ladder, in order:** delete unused → right-size (p95 of CPU *and* memory over ≥2 weeks; install the memory agent first) → schedule off-hours → commitments → architecture. 60–80% of achievable savings sits in the boring rungs; commitments bought before cleanup lock in waste at a discount.
- **Savings that require vigilance don't persist.** Every cleanup ships with its prevention (policy, lifecycle rule, quota, schedule) or you rented the savings, not bought them.

## Rate anchors (order of magnitude, 2026 — reverify before quoting)

Internet egress ~$0.09/GB · cross-AZ (AWS) $0.01/GB each direction · NAT processing $0.045/GB + ~$32/mo hourly · log ingestion ~$0.50/GB · inter-region ~$0.02/GB · gp2→gp3 ~20% free win. The 20TB/mo-through-NAT pipeline = $900/mo; the S3 gateway endpoint fix = $0 and one route-table entry.

## Commitments

- Measure trailing 60–90 days of *hourly* spend; commit ~70–80% of the **floor** (P5–P10 hour, not the average); 70–85% coverage is the healthy band; alarm utilization <95%. Buy in tranches to ladder expiries. Effective discount = headline × utilization: a 72% instrument at 70% utilization (~60% effective) loses to 66% at 100% — flexibility beats rate when in doubt.
- Compute Savings Plans (~66%, cross-family/region, EC2+Fargate+Lambda) as the baseline layer; EC2 Instance SPs/RIs (~72%, family+region locked) only for named multi-year-stable fleets. **Savings Plans do not cover RDS/ElastiCache/OpenSearch/Redshift — those still need service RIs**, the most commonly forgotten 30–50% on managed-data spend.
- Azure: reservations + savings plan + **Hybrid Benefit** (check licenses first — often the largest lever) + Dev/Test subscription pricing. GCP: CUDs + automatic SUDs.
- A planned migration (Graviton, Fargate) is a reason to take the flexible instrument and re-measure the floor after it lands — family-locked commitments strand.

## Spot / preemptible

Notice: AWS 2 min, Azure/GCP ~30s. The 2026 risk is capacity reclaim (correlated waves per pool), not price. Mandatory engineering: interruption handler (drain/checkpoint/requeue), diversification across many pools with capacity-optimized allocation, on-demand floor mixed in. Track interruption rate per pool and evict bad pools. Never: stateful primaries, uncheckpointable long work, latency-SLO singletons.

## Storage lifecycle and data transfer

Every log/backup/artifact bucket gets tier + **expiry** rules at creation — the delete rule matters more than the tiering, and minimum-duration math (30d IA, 90–180d archive) can make tiered churning data cost *more*. On versioned buckets the two silently-hoarding gaps are `NoncurrentVersionExpiration` and `AbortIncompleteMultipartUpload` — invisible in the object list, fully visible on the bill. Bill-review transfer checklist: NAT processing (free gateway endpoints), cross-AZ chatter, internet egress (CDN in front; origin-to-CDN free), forgotten cross-region replication, log ingestion ("never expire" is the default and it is a trap).

## Analytics and data platforms

Scan-priced engines (Athena, BigQuery on-demand): the fix is physical — partitioning on filter columns, Parquet, pre-aggregated tables for anything a dashboard polls — not query politeness. Time-priced warehouses (Snowflake): aggressive auto-suspend in minutes, per-workload-class sizing, and hunt the scheduled job keeping an XL resumed 24/7 for one hourly query. Per-team cost-per-query turns "the platform is expensive" into three named dashboards fixable by Friday.

## Tagging, guardrails, Kubernetes

- Four enforced keys (`owner`/`service`/`env`/`cost-center`), deny-rules not wiki pages (unenforced standards plateau ~40–60% coverage); **activate cost-allocation tags in the AWS billing console the same week — activation is not retroactive**, the classic months-of-lost-data own-goal. Shared costs get a written split rule (proportional to direct spend is fine); allocate platforms partially — 100% cross-charging breeds shadow infrastructure that costs more in total.
- Budgets with *actions* (freeze sandbox creation), anomaly detection routed to the owning team's channel via tag, SCP/policy-blocked GPU SKUs and exotic regions, TTL reaper on ephemeral environments, and a 15-minute weekly top-5-movers review (monthly deep reviews die).
- Kubernetes waste order: over-set requests (dominant) → bin-packing → per-node overhead → missing autoscaler/spot. **Right-size requests before node optimization** — autoscaling against 4×-inflated requests institutionalizes the waste and compresses garbage efficiently. Charge teams requests × node unit price per namespace; it's the only mechanism that makes request inflation self-correcting. Scaling *minimums* are commitments in disguise — review them like reservations.

## LLM / GPU cost (fast-moving — researched mid-2026; reverify rates at decision time)

- **Hyperscalers are not the cheap GPU option:** H100s ~$11–13/GPU-hr on-demand at AWS/Azure (8-GPU node granularity) vs ~$1.4–3.5/hr at neoclouds (CoreWeave, Lambda, marketplaces); H200s ~$3.8–4 cheap-provider vs $10+ hyperscaler; B200s ~$3–6 with spot near $2–3. The 3–5× spread is persistent and is the single biggest ML-infra lever; the premium buys data-locality with your stack and enterprise controls — make that an explicit purchase. The break-even is dominated by egress and integration effort, not the GPU rate.
- Utilization is the real GPU metric: a reserved H100 at 30% utilization costs 3.3× sticker per useful hour — worse than on-demand neocloud. Idle-kill notebooks (the #1 offender); reserve the floor, burst elsewhere.
- Inference ladder: per-token APIs → prompt caching + routing/cascades + batch APIs (~50% off) → provisioned throughput → self-hosted open-weights. The software rungs (caching, model right-sizing, token caps, trimmed contexts) routinely cut 60–80% and beat infrastructure heroics — exhaust them before renting a GPU. Training: spot + checkpoint cadence set by arithmetic (cadence = max loss per interruption).
- Token hygiene mirrors infra hygiene: per-feature metering, max-token caps, monthly cost-per-successful-task; one verbose feature is usually half the spend.

## How an expert thinks through this

*"Series-B SaaS, $48k → $71k/mo in six months, CFO wants 30%."*

Attribution first (20% untagged → fix in parallel, don't block). Denominator second: customers +25%, so unit cost rose ~23% — a real regression. Decompose the delta by service, then ladder: delete (14 hackathon environments, unattached volumes, 3 idle NATs ≈ $6k, one afternoon, no meetings) → right-size (RDS at 12% CPU sized "for Black Friday" → two sizes down with a scheduled scale-up for the actual event; fleet p95 22% → downsize + generation bump ≈ $7k) → schedule non-prod ($3k) → the $4k transfer growth is NAT processing from a new pipeline → free S3 gateway endpoint ($3k, the highest-ROI single action) → CloudWatch DEBUG bodies + never-expire ($2.5k). ≈$21.5k before any commitment; *then* buy Compute SPs at 75% of the cleaned floor (+$8k). Rejected: Fargate re-architecture (months of work the SP now partially captures — revisit next year); 3-year Instance SPs (mid-Graviton-migration; flexibility beats 6 points). Institutionalize: per-team budgets, anomaly alerts, SCP tag enforcement, unit-cost dashboard so the CFO conversation never restarts from zero.

## Failure modes & pitfalls (checklist)

- Commitments before cleanup; committing to the average instead of the floor; utilization unmonitored (prepaid waste *looks* like a discount and evades anomaly detection — it needs its own alarm).
- Right-sizing on CPU average → OOM after downsizing the memory-bound service nobody measured.
- Savings theater: celebrated cleanup, unchanged creation-side policy, six months later it's back.
- Egress-blind reviews: any design moving >1TB/mo gets a $/GB line item in the doc.
- Observability spend treated as fixed — ingestion routinely hits 10%+ of infra spend by *default settings*.
- Spot without interruption engineering: works in the pilot, dies in the first reclaim wave.
- GPU reservations sized to peak research demand → 30% utilization forever.
- Optimizing the interesting 5% (a week of Lambda tuning for $80/mo while zombie environments burn $6k) — rank by absolute dollars; the Pareto is brutal and boring.
- Denominator gamed mid-quarter; free-tier PoC linearly forecast to production (off by the entire bill).

## Verification / self-check

1. Savings stated as $/mo *and* unit-cost delta against a named baseline month.
2. Ladder order respected — commitments sized only after delete/right-size/schedule landed.
3. Every recommendation carries its risk label and its persistence mechanism.
4. Transfer and observability lines examined explicitly — compute-only analysis is half an analysis.
5. Fast-moving prices (GPU rates, discount instruments) verified this session; stale prices self-discredit the review.
Stopping rule: stop when the next rung's projected savings < the honestly-priced engineering cost to reach them; hand the remainder to automation and the quarterly cycle.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)

- Probed 14 claims: 14 baseline (cut/compressed), 0 partial, 0 delta — Opus cold reproduced the ladder, floor-commitment sizing, effective-discount arithmetic, NAT/endpoint fix, spot engineering, k8s requests-first ordering, versioned-bucket lifecycle gaps, tag-activation trap, LLM ladder, and utilization alarms at full specificity.
- File restructured to a compact anchor sheet: rate table, checklists, and the worked walkthrough retained as review scaffolding; all explanatory prose cut.
- Residual value is currency, not concepts: 2026 GPU rate anchors (Opus's figures matched but drift quarterly — the "reverify at decision time" discipline is the point), and Savings-Plans-don't-cover-RDS as the most-forgotten line.
