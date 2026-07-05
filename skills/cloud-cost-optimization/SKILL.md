---
name: cloud-cost-optimization
description: Load when analyzing or reducing cloud spend — bill investigation, FinOps practices, reserved instances/savings plans strategy, spot/preemptible usage, egress and data-transfer costs, storage lifecycle, tagging and cost allocation, unit economics, or GPU/LLM cost management on AWS/Azure/GCP.
---

# Cloud Cost Optimization (FinOps Engineering)

## Core mental model

- **Compute is elastic; data costs are sticky and silent.** Everyone watches instance spend because it's the big visible line. The bill-killers that creep are **data transfer** (internet egress ~$0.09/GB on AWS, cross-AZ $0.01/GB *each way*, NAT gateway processing $0.045/GB stacked *on top of* egress) and **storage that only ever grows** (snapshots, logs, orphaned volumes). Compute mistakes self-correct when instances die; data mistakes compound monthly forever.
- **Unit economics, not total bill.** "Spend went up 40%" is meaningless without "requests went up 70%." The only defensible metrics are cost-per-request, cost-per-customer, cost-per-training-run, cost-per-GB-processed. A rising bill with falling unit cost is a business succeeding; a flat bill with rising unit cost is a slow leak. Establish the denominator before optimizing anything.
- **Allocation before optimization.** You cannot optimize what you cannot attribute. Tagging/labeling discipline (owner, service, environment, cost-center — enforced by policy, not memo) and account/subscription separation are the prerequisite for every other move. An unattributed bill produces meetings, not savings.
- **Optimization has an ROI ladder — work it in order.** Delete unused (infinite ROI, zero risk) → right-size (high ROI, low risk) → schedule off-hours (high ROI for non-prod) → commitment discounts (financial move, no engineering) → architectural change (highest ceiling, highest cost/risk). Teams jump to re-architecture because it's interesting; the boring rungs usually contain 60–80% of the achievable savings.
- **Savings that require sustained vigilance don't persist.** Prefer structural fixes (lifecycle policy, autoscaling floor, policy-blocked SKUs, gateway endpoints) over heroic cleanups. A one-time cleanup without the policy that prevents recurrence is renting the savings, not buying them.

## The cost map — where money hides, with numbers (AWS-flavored; Azure/GCP analogous)

| Line item | Rate (order of magnitude, 2026) | Why it surprises |
|---|---|---|
| Internet egress | ~$0.09/GB first tier | Priced per GB nobody estimated; CDN-out is the fix |
| Cross-AZ traffic (AWS) | $0.01/GB *each direction* | Invisible on diagrams; chatty services and replication pay it twice |
| NAT gateway processing | $0.045/GB + ~$32/mo each | Stacks *on top of* egress; free gateway endpoints often eliminate it |
| Log ingestion (CloudWatch/Log Analytics) | ~$0.50/GB order | DEBUG × never-expire retention is a choice made by default |
| Inter-region replication/pulls | ~$0.02/GB | Enabled once, billed forever |
| Snapshots/orphaned storage | per-GB-month, compounding | Only ever grows unless a policy shrinks it |

The pattern: per-GB charges attached to *movement and retention* — costs proportional to data flows nobody drew on the architecture diagram. Price every arrow.

## Unit economics discipline

- Build the metric as `allocated cost ÷ business denominator` per service: cost-per-1k-requests for APIs, cost-per-customer for tenancy decisions, cost-per-job/run for batch and ML. Pick the denominator the *business* already tracks, define it in writing, and never change it silently.
- Use it to route effort: a unit cost flat while traffic doubled means the architecture is scaling sub-linearly — leave it alone regardless of the total. A unit cost creeping 3%/month is the leak worth a project even if the total looks stable.
- Unit economics also answers pricing questions engineering gets dragged into: "can we offer a free tier?" is answerable only as marginal-cost-per-free-user.
- Present every optimization in both currencies: "$8k/mo" and "−22% cost-per-request" — the first gets budget approval, the second proves it wasn't just traffic decline.

## The optimization ladder — reasoning at each rung

1. **Delete unused.** Unattached EBS volumes/managed disks, aged snapshots, idle load balancers, stopped-but-EBS-billing instances, unused elastic IPs, empty-but-provisioned databases, forgotten dev environments, orphaned NAT gateways ($32+/mo each doing nothing). Find via cost tools (Cost Explorer resource view, Azure Advisor, or Cloud Custodian rules). Expect 5–15% of the bill in a mature-but-unaudited account. This rung requires no design conversation — do it first, always.
2. **Right-size.** Compare provisioned vs used (CloudWatch/Azure Monitor 2-week p95, not average). The classic: instances at 8% CPU because someone sized for a launch spike in 2023. Downsize one size at a time; watch memory (the metric agents don't collect by default — install the agent before claiming a box is idle). Also: gp2→gp3, previous-gen → current-gen instance families (better price/perf for free), over-provisioned IOPS.
3. **Schedule off-hours.** Non-prod running 24/7 costs 3× its business-hours cost (168h vs ~50h). Instance schedulers, ACA/Cloud Run scale-to-zero, Aurora Serverless v2 min-capacity (as of 2026 it scales to zero), database stop schedules. Cultural note: this rung fails when stopping things is manual — automate the stop, make the *start* the manual/automatic-on-access step.
4. **Commitment discounts** — see next section. Financial engineering, zero code, but do it *after* rungs 1–3 or you'll commit to waste.
5. **Architectural change:** NAT → gateway endpoints, straight-S3 serving → CloudFront, chatty cross-AZ services → zone-aware routing, per-request FaaS → containers at sustained load (or the reverse at low duty cycle), Snowflake/BigQuery query hygiene, log-ingestion diets. Highest ceiling, but cost it like any project — engineering time is also money, and a $200/mo saving rarely justifies a quarter of platform work.

## Tagging and allocation — the unglamorous prerequisite

- Minimum viable schema, enforced not suggested: `owner` (team, not person), `service`, `env`, `cost-center` — four keys, controlled vocabularies. More keys at the start means more drift; add later from need.
- Enforcement is structural: tag policies + SCPs on AWS (deny `RunInstances` without required tags), Azure Policy `deny`/`modify` effects (inherit tags from resource group). A wiki standard without a deny rule achieves ~60% coverage decaying toward 40%.
- Activate cost-allocation tags in the billing console the same week (AWS tags don't appear in Cost Explorer until activated — a months-of-lost-data classic).
- Shared costs (NAT, support plans, observability, clusters) need an explicit split rule — proportional to direct spend is defensible and simple; per-request attribution is rarely worth its plumbing. What matters is that the rule is written down so teams stop litigating it.
- Kubernetes/multi-tenant platforms break resource-level allocation: adopt namespace/label-based cost tooling (OpenCost/Kubecost-style) and treat *requests*, not usage, as the allocated quantity — teams pay for what they reserve, which creates the right-sizing incentive exactly where the knowledge lives.
- The output that makes it real: a monthly per-team, per-service unit-cost report that a team lead can act on without asking FinOps to interpret it. Allocation nobody reads is compliance, not FinOps.

## Commitment strategy (verified current as of 2026)

- **Instruments (AWS):** Compute Savings Plans (up to ~66% off, applies across EC2 families/regions/Fargate/Lambda), EC2 Instance Savings Plans (up to ~72%, locked to family+region), Standard/Convertible RIs (RDS, ElastiCache, OpenSearch, Redshift still need RIs — Savings Plans don't cover them). Azure: reservations (VMs, SQL, Cosmos) + Azure savings plan for compute + Hybrid Benefit (bring your Windows/SQL licenses — often the largest single Azure discount). GCP: committed use discounts; sustained-use discounts apply automatically.
- **The layering that practitioners converge on:** measure the past 60–90 days of *hourly* on-demand spend; find the **floor** (minimum sustained $/hr, not the average); commit Compute Savings Plans to ~70–80% of that floor; add resource-specific commitments (EC2 Instance SPs, DB reservations) only for workloads with named multi-year stability; leave the top 20–30% on-demand/spot for churn.
- **The utilization-vs-flexibility tradeoff, quantified:** a commitment is only worth its discount × its utilization. A 72% discount instrument used at 70% ≈ a 66% instrument at 100% — with none of the flexibility. When in doubt, take the more flexible instrument at the slightly worse rate; architectures change faster than 3-year terms.
- **Coverage target ≠ 100%.** Chasing full coverage guarantees overcommitment after the next architecture change (that migration to Graviton/ARM or to Fargate you're planning changes what your commitments match). 70–85% coverage of steady-state is the defensible band; review quarterly, buy in tranches (monthly/quarterly purchases ladder the expiry dates and average out mistakes).
- Watch both dashboards: **utilization** (are we using what we bought) and **coverage** (what fraction of usage is discounted). Alarm on utilization < ~95% — that's money already spent, leaking.

## Spot / preemptible engineering

- **What tolerates interruption:** stateless web fleets behind LBs (with surplus capacity), batch/queue workers (work returns to queue), CI runners, big-data executors (Spark with decommissioning), ML *training with checkpointing*, rendering. **What doesn't:** databases and stateful primaries, long transactions that can't checkpoint, latency-SLO singletons, anything whose interruption cost (rework + human attention) exceeds the ~60–90% discount.
- **Engineering requirements, not optional:** handle the interruption notice (AWS: 2-minute warning via IMDS/EventBridge; Azure Spot: ~30s; GCP: ~30s) — drain connections, checkpoint, requeue; diversify across many instance types/AZs (capacity-optimized allocation) because single-pool spot evaporates in correlated waves; mix a spot base with an on-demand floor (e.g., ECS capacity provider strategy or ASG mixed-instances: 30% on-demand base, 70% spot).
- Spot prices are relatively stable in 2026 — the risk isn't price spikes, it's *capacity reclaim*. Treat interruption rate as a per-pool observable and evict bad pools from your mix.

## Storage lifecycle and data-transfer specifics

- Every log/backup/artifact bucket gets a lifecycle policy at creation: transition to IA/Cool after 30 days, archive tiers after 90, **expiry** after the retention requirement — the delete rule matters more than the tiering. Verify minimum-storage-duration math (30d IA / 90–180d archive tiers) against object churn before tiering; small hot-churning objects can cost *more* tiered.
- Intelligent-Tiering (AWS) as default for unknown patterns; it caps the downside of nobody-ever-audits-this.
- Snapshots: orphaned EBS/disk snapshot chains from deleted volumes are the classic ratchet — automate retention (Data Lifecycle Manager / Azure Backup policies).
- Data transfer checklist for any bill review: NAT gateway processing (add S3/DynamoDB gateway endpoints — free — and interface endpoints where volume justifies), cross-AZ chatter (co-locate chatty pairs, topology-aware routing, single-AZ for dev), internet egress (CloudFront/CDN in front — origin-to-CDN is free on AWS), cross-region replication you forgot about, and per-GB log ingestion (sample debug logs, set retention; "never expire" is the default and it is a trap).

Worked lifecycle policy (S3, the shape to copy — tier *and* expire, with the multipart cleanup everyone forgets):
```json
{ "Rules": [{
    "ID": "logs-standard-lifecycle", "Status": "Enabled",
    "Filter": { "Prefix": "logs/" },
    "Transitions": [ { "Days": 30, "StorageClass": "STANDARD_IA" },
                     { "Days": 90, "StorageClass": "GLACIER_IR" } ],
    "Expiration": { "Days": 365 },
    "AbortIncompleteMultipartUpload": { "DaysAfterInitiation": 7 },
    "NoncurrentVersionExpiration": { "NoncurrentDays": 30 }
}]}
```
The `AbortIncompleteMultipartUpload` and noncurrent-version rules are where versioned buckets silently hoard invisible gigabytes — invisible in the console object list, fully visible on the bill.

## Analytics and data-platform cost — the per-scan trap

- Scan-priced engines (Athena, BigQuery on-demand) bill per TB scanned: a dashboard auto-refreshing a `SELECT *` over an unpartitioned table every 5 minutes is a money printer in reverse. The fixes are physical, not query-side politeness: partition on the filter columns, columnar formats (Parquet), clustering, and materialized/pre-aggregated tables for anything a dashboard polls.
- Warehouse-compute engines (Snowflake, BigQuery capacity) bill for *time running*: auto-suspend aggressively (minutes, not hours), size warehouses per workload class instead of one XL for everything, and hunt the scheduled job that keeps the warehouse resumed 24/7 to serve one hourly query.
- The unit metric here is cost-per-query or cost-per-dashboard — surfacing it per team turns "the data platform is expensive" into three named dashboards someone can fix by Friday.
- Same prevention logic as everywhere: query/scan budgets and alerts per user or service account, because one analyst's cross-join should page them, not surprise finance.

## Guardrails and anomaly detection — savings that persist

- **Budgets with actions, not just emails:** per-team/per-account budget alerts at 80/100/forecast-120%, and for sandboxes, automated response (notify → freeze new resource creation) — a human reading a budget email three weeks later is not a control.
- **Anomaly detection on by default** (AWS Cost Anomaly Detection, Azure Cost Management anomaly alerts): they catch the recursive Lambda loop and the DEBUG logger the day it ships, not at month-end. Route alerts to the owning team's channel via the tag, not to a central FinOps inbox.
- **Deny the expensive mistakes structurally:** SCP/Azure Policy blocking GPU/metal SKUs and exotic regions in general-purpose accounts, quota caps on sandbox accounts, TTL-based auto-teardown for ephemeral environments (tag `expiry` + a reaper job).
- **Weekly cost review cadence, 15 minutes:** top-5 movers vs last week, anomalies triaged, one action item. Monthly deep reviews die; a short weekly loop with the delta view survives.
- Treat the monthly bill diff like a code diff: every unexplained line item gets an owner or a ticket. "Unexplained but small" compounds into "unexplainable and large."

## Kubernetes and container-platform cost specifics

- The waste hierarchy in clusters: (1) requests set far above usage (the dominant term — right-size requests before anything), (2) poor bin-packing from oversized/mismatched node shapes, (3) idle system overhead per node (fewer, larger nodes amortize daemonsets), (4) missing autoscaler + spot mix on stateless pools.
- Right-size requests from VPA/actual-usage percentiles, then let cluster autoscaler/Karpenter consolidate nodes; doing node optimization before request optimization compresses garbage efficiently.
- Charge teams by *requests × node unit price*, publish it per namespace — the only mechanism that makes request inflation self-correcting.

## LLM / GPU cost (fast-moving — researched mid-2026)

- **Hyperscalers are not the cheap option for GPUs.** As of mid-2026, H100s run ~$6–12/GPU-hr on-demand at AWS/Azure vs ~$1.4–3/hr at neoclouds (CoreWeave, Lambda, Fluidstack, marketplaces); H200s ~$3.8–4 at cheap providers vs ~$10+ hyperscaler; B200s roughly $3–6/hr with spot near $2–3. The spread is 3–5×, persistent, and the single biggest lever in ML infra budgets. The hyperscaler premium buys data-locality with your existing stack and enterprise controls — sometimes worth it, but make it an explicit purchase, not a default.
- **Utilization is the real GPU metric.** A reserved H100 at 30% utilization costs 3.3× its sticker per useful hour — worse than on-demand neocloud. Measure GPU-hours-per-training-run and $-per-training-run; hunt idle allocations (notebooks holding GPUs overnight is the #1 offender — idle-kill policies).
- **Inference cost ladder:** per-token APIs (zero commitment, best below sustained volume) → provisioned throughput / PTU-style reservations when steady → self-hosted open-weights on rented GPUs when volume is high and prompts are cacheable/batchable. Prompt caching, batching (batch APIs typically ~50% off), model right-sizing (a distilled/small model for the 80% easy cases, escalate the rest) usually beat infrastructure heroics; measure $-per-1k-requests before and after.
- Training: spot + aggressive checkpointing (every N minutes to object storage) makes 60–70% discounts survivable; the checkpoint cadence is your maximum loss-per-interruption — set it by arithmetic, not vibes.
- Token-spend hygiene mirrors infra hygiene: per-feature token metering (the tagging equivalent), max-token caps on every call, context windows trimmed to what the eval says is needed, and a monthly "cost per successful task" number — LLM bills have the same Pareto (one verbose feature is usually half the spend) and the same fix (measure, attribute, cap).
- Verify GPU rates at decision time, not from memory or this file — the market moves quarterly and a stale $/hr in a proposal invalidates the whole comparison.

## How an expert thinks through this

*"Series-B SaaS, AWS bill went $48k → $71k/mo in six months, CFO wants 30% off. Where do I start?"*

Not with architecture. First: attribution. Pull Cost Explorer grouped by service, then by tag; discover 20% is untagged → fix allocation in parallel (tag policy + backfill), but don't block on it. Second: the denominator — customers grew 25%, so unit cost rose ~23%; this is a real efficiency regression, not just growth. Now decompose the delta: EC2 +$9k, RDS +$5k, data transfer +$4k, CloudWatch +$3k, S3 +$2k.

Ladder rung 1 (delete): resource-level view shows 14 forgotten dev/staging environments from a hackathon, unattached volumes, 3 idle NAT gateways → ~$6k/mo, one afternoon, no meetings. Rung 2 (right-size): RDS primary at 12% CPU on a db.r6g.4xlarge "sized for Black Friday" → step down two sizes with a scheduled scale-up for the actual event; EC2 fleet p95 CPU 22% → downsize + one instance-family generation bump → ~$7k/mo. Rung 3: non-prod on schedules (they run 24/7) → ~$3k/mo. Data transfer: $4k growth is NAT processing — the new data pipeline pulls from S3 through NAT; add the free S3 gateway endpoint → ~$3k/mo for a one-line route-table change; this is the highest-ROI single action in the whole exercise. CloudWatch: new service logs full request bodies at DEBUG, retention "never" → sampling + 30-day retention → ~$2.5k/mo.

Running total ≈ $21.5k (30% hit) *without commitments yet*. Now, on the cleaned baseline, measure the 90-day hourly floor and buy Compute Savings Plans at 75% of it → another ~$8k/mo. Considered and rejected for now: migrating the monolith to Fargate ("it would right-size itself") — months of engineering for savings the SP now partially captures anyway; revisit next year. Rejected: 3-year all-upfront EC2 Instance SPs despite the better rate — they're mid-migration to Graviton, so instrument flexibility beats 6 extra points of discount. Last: institutionalize — budget alerts per team tag, weekly anomaly detection, tag policy enforced via SCP, and the unit-cost dashboard so the CFO conversation never restarts from zero.

## Failure modes & pitfalls

- **Buying commitments before cleaning up** — you lock in the waste at a discount and *guarantee* paying for deleted infrastructure. Ladder order is not optional.
- **Committing to the average instead of the floor** of hourly usage → utilization dips below 100% nightly; the "70% coverage of p10 floor" discipline exists because on-demand overflow is cheaper than idle commitment.
- **RI/SP utilization unmonitored:** an unused reservation makes the bill *look* fine (it's prepaid) while burning cash. Alarm at <95% utilization; exchange/resell (Convertible exchange, RI Marketplace) instead of riding it out.
- **Right-sizing on CPU average** → OOM kills after downsizing a memory-bound service the agent wasn't measuring. p95 of *both* CPU and memory over ≥2 weeks including month-end/seasonal peaks.
- **Savings theater:** celebrating a $40k cleanup while the creation-side policy is unchanged — six months later it's back. Every cleanup ships with its prevention (policy, lifecycle rule, quota, schedule).
- **Egress-blind architecture reviews:** approving a cross-region read replica, an inter-AZ Kafka mirror, or "we'll stream video from S3 directly" without a $/GB estimate. Any design moving >1TB/mo gets a transfer line item in the doc.
- **Log/observability spend treated as fixed:** CloudWatch/Datadog/Log Analytics ingestion routinely reaches 10%+ of infra spend; per-GB ingestion × DEBUG × never-expire is a choice, not a fact of nature.
- **Kubernetes bin-packing illusion:** cluster "needs" 40 nodes because requests are set at 4× actual usage; right-size *requests* (VPA recommendations) before nodes. Cost-per-pod without request-tuning is fiction.
- **Spot without interruption handling:** works in the pilot, then a capacity reclaim wave kills 60% of the fleet in one AZ-pool and the queue backs up into an SLA breach. Diversify pools + on-demand floor + drain hooks, or don't use spot.
- **GPU reservations sized to peak research demand** → 30% utilization forever. Reserve the floor, burst to on-demand/neocloud, idle-kill notebooks.
- **Optimizing the interesting 5%:** a week of Lambda memory-tuning saving $80/mo while 14 zombie environments burn $6k. Always rank by absolute dollars first; the Pareto in cloud bills is brutal and boring.
- **Unit-cost metric gamed by the denominator:** switching from cost-per-customer to cost-per-request mid-quarter to make a trend look good. Fix the denominator per service in writing; change it only with a restated history.
- **Auto-scaling as a cost strategy without a floor audit:** the fleet scales beautifully from a minimum of 20 instances that nobody ever justified. Scaling *minimums* are commitments in disguise — review them like reservations.
- **Cross-charging shared platforms at 100%** → product teams build shadow infrastructure to dodge the allocation, costing more in total. Subsidize platforms partially; the goal of allocation is behavior, not accounting purity.
- **Ignoring the free-tier cliff in forecasts:** the PoC that cost $0 on free tiers linearly forecast to production is off by the entire bill. Re-price at production volumes explicitly.

## Worked micro-example — commitment sizing from the hourly floor

90 days of hourly compute spend shows: overnight floor $41/hr, business-day plateau $68/hr, month-end peak $95/hr, and a planned migration that will move ~15% of EC2 to Fargate next quarter.
- Commit target: 75% of the *floor* → $31/hr of Compute Savings Plan (not EC2 Instance SP — the Fargate migration would strand family-locked commitments; flexibility beats the extra discount points here, and Compute SPs follow the workload to Fargate).
- Expected coverage: $31 of every hour discounted ≈ 55–75% coverage depending on time of day — inside the healthy band; the plateau/peak stays on-demand+spot by design.
- Buy in two tranches (now, +6 weeks) to ladder expiries; re-measure the floor after the migration lands before tranche three.
- Review loop: utilization alarm <95%, coverage reviewed quarterly. If utilization ever dips, the fix is *waiting* (growth catches up) or exchanging — never buying more to "average it out."

## Worked micro-example — the NAT tax, quantified

Pipeline pulls 20TB/mo from S3 into private-subnet workers through a NAT gateway:
- NAT processing: 20,000 GB × $0.045 = **$900/mo** (+ ~$32/mo/gateway hourly)
- Fix: S3 gateway endpoint = **$0**. One route-table entry.
Same pipeline later ships 5TB/mo of results to another region's bucket: 5,000 × $0.02 (inter-region) = $100/mo — fine, but now it's a *known* line item. The habit being taught: price every GB-moving arrow on the diagram; the ones nobody priced are where the $900s live.

## Verification / self-check

1. Savings claims are stated as $/mo *and* as unit-cost deltas, against a named baseline month.
2. Ladder order respected — commitments sized only after delete/right-size/schedule landed.
3. Every recommendation carries its risk label (zero-risk deletion vs performance-risk downsize vs term-risk commitment) and its persistence mechanism (what policy prevents recurrence).
4. Data-transfer and observability lines were examined explicitly — if the analysis only discusses compute, it's half an analysis.
5. Current prices/discount mechanics verified this session for anything fast-moving (GPU rates, new discount instruments); stale prices in a cost review are self-discrediting.
Stopping rule: stop optimizing when the next rung's projected savings < the engineering cost to reach them (price the engineers' time honestly), and hand the remainder to the quarterly review cycle — continuous vigilance belongs to automation, not heroes.
