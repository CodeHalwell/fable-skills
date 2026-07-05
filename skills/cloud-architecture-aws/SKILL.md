---
name: cloud-architecture-aws
description: Load when designing, reviewing, or debugging architectures on AWS — choosing compute (Lambda/Fargate/ECS/EC2), storage (S3/EBS/EFS/DynamoDB/RDS), VPC networking, IAM policies, multi-AZ/multi-region decisions, or diagnosing AWS bill shock and outage patterns. Also load when someone asks "which AWS service should I use for X".
---

# AWS Architecture Judgment

## Core mental model

- **Managed-ness is the axis; utilization is the crossover.** Duty cycle below ~20–30% → serverless wins; steady 24/7 → containers/EC2 with commitments (Lambda at constant high throughput costs 3–10× equivalent Fargate/EC2). Most teams misprice their own ops time and sit too low on the spectrum.
- **Data gravity dominates.** Egress, cross-AZ charges, and migration pain attach to data. Decide where data lives first; put compute next to it.
- **IAM is the actual architecture; the bill is a monthly design review.** If you can't predict your design's top-5 line items, you don't understand it yet.
- **AWS deprecates quietly.** Services drift into maintenance mode with a blog post, not a siren (App Runner in 2026; WorkMail; gp2 de facto). Any service recommendation recalled from memory rather than verified this year is a liability — this is the discipline this file exists to enforce.

## Compute selection

| Signal | Points to |
|---|---|
| Spiky, short, stateless, <15 min | Lambda |
| Containerized service, steady-ish | ECS on Fargate |
| "Just run my container from a repo" | **ECS Express Mode** (see below) |
| Needs K8s API/operators/mesh | EKS (the only justification for its ops surface) |
| GPU, licensed software, OS control, >10GB function-shaped memory | EC2 |
| Long batch, no latency SLO | Fargate tasks / AWS Batch |

Order the questions: duty cycle → unit-of-work duration (>15 min disqualifies Lambda) → first-request latency (async paths don't care about cold starts; don't pay to fix a non-problem) → per-instance state (WebSockets/loaded models → long-lived containers) → cost shape → "do you *need* Kubernetes, or containers?"

**The 2026 correction models get wrong:** AWS **App Runner is in maintenance mode and closed to new customers as of April 2026**. Pre-2026 knowledge says "App Runner is stagnant but alive; default to ECS/Fargate with your own ALB plumbing" — the current answer for "run my container from a repo" is **ECS Express Mode** (launched late 2025), which generates the ALB/scaling/ECS wiring for you. Don't propose App Runner for new builds, and don't hand-roll what Express Mode now generates.

## Storage selection — the non-obvious halves

- S3 lineup as of 2026: Standard, Intelligent-Tiering, Standard-IA, One Zone-IA, Express One Zone (~7× Standard $/GB, single-AZ performance product), Glacier Instant/Flexible/Deep Archive. Default Intelligent-Tiering for unknown patterns. Small churning objects are poison for archive tiers (40KB minimum billing, per-object transition fees, 90–180-day minimums) — aggregate before archiving; do the break-even arithmetic before adding any lifecycle rule.
- EBS = one instance's disk (AZ-scoped); EFS = shared POSIX at ~3× EBS cost — only for genuine shared *file semantics*; write-once/read-many objects = S3, full stop.
- **DynamoDB vs Aurora is "do I know all my access patterns now?"** — not SQL vs NoSQL. Young products with churning queries → Aurora Postgres (JSONB for the semi-structured parts). DynamoDB red flags: "we'll scan and filter," >~3 GSIs papering over relational patterns, hot tenant keys, a data team that will want SQL next quarter. Aurora Serverless v2 scales to zero (since late 2024), which killed DynamoDB's cost-only argument for spiky dev/test relational workloads.
- Messaging by delivery semantics: SQS (work queue) / SNS (fan-out) / EventBridge (content routing, SaaS events) / Kinesis (ordered replayable stream). Kinesis-as-job-queue and SQS-as-telemetry are category errors.

## Networking

- NAT gateway processing ($0.045/GB) is the #1 bill-shock source; **gateway endpoints for S3/DynamoDB are free — add them to every VPC** (ECR layers ride S3, so this also fixes image-pull-through-NAT). Interface endpoints pay for themselves once private subnets push real volume to any AWS API.
- SG-to-SG references for all segmentation; leave NACLs at default (stateless return-traffic bugs for zero benefit).
- Egress traps ranked: NAT processing on S3/ECR traffic → cross-AZ at $0.01/GB *each direction* → internet egress ~$0.09/GB (CloudFront in front; origin-to-CloudFront is free) → forgotten cross-region replication.
- One NAT per AZ for real HA — a shared NAT severs egress for *surviving* AZs when its AZ dies.
- Global Accelerator only for non-HTTP or static anycast IPs; CloudFront already covers plain websites.

## IAM

Roles never users; OIDC federation for CI; least privilege iterated from AccessDenied/Access Analyzer, not written perfect on day one; conditions (`aws:PrincipalOrgID`, `s3:prefix`, `aws:ResourceTag`) survive resource sprawl better than ARN enumeration; `iam:PassRole` scoped to specific role ARNs (with `Resource:"*"` whoever passes an admin role *is* admin); permission boundaries for delegation (effective = intersection); multi-account under Organizations beats within-account cleverness; cross-account debugging is usually the missing resource-policy half or an SCP deny.

## Multi-AZ vs multi-region

Prior: multi-AZ yes, multi-region almost always no — 1.5–2× cost plus a permanent engineering tax, and untested failovers mostly fail when invoked. Justifications in frequency order: latency to distant users (that's multi-region *serving* — CloudFront/read replicas — a far easier problem than failover; don't let it smuggle in active-active writes), data residency (region-pinned data, not duplication), downtime genuinely costing millions/hour (then fund *quarterly-exercised* failover or it's theater). Otherwise: cross-region backups + IaC that rebuilds in hours + restore drills ≈ 5% of the cost, covering the realistic disasters (corruption, account compromise, evacuation with notice).

## How an expert thinks through this

*"Document-processing API: PDF uploads 1–50MB, OCR + extract, results queryable later; 50k docs/day, bursty, 30–120s per doc."*

Data first: presigned S3 uploads (never proxy 50MB through compute). Results → Aurora Postgres, rejecting DynamoDB explicitly: access patterns unknown and 50k writes/day is nothing — flexibility, not scale, is the constraint. Processing: under Lambda's cap; container image (OCR deps won't fit zip); Lambda-from-SQS unless GPU is proven by benchmark (then ECS on GPU EC2 with queue-depth scaling). API: Lambda+API Gateway or one small Fargate service — pick by team shape and *say so*; reject EKS (no K8s need = a second job). Networking: Lambda-in-VPC → add the S3 gateway endpoint immediately (otherwise 50k × 25MB/day ≈ $56/day of NAT waste) and RDS Proxy in front of Aurora — the 9am burst exhausting Postgres connections is the outage this design ships with otherwise. Resilience: multi-AZ everything, multi-region no — an hour's delay of document processing hurts nobody; write that reasoning down so no one "upgrades" it without new facts.

## Bill-shock and outage priors

Bill jumped? In frequency order: (1) NAT processing; (2) cross-AZ transfer; (3) CloudWatch Logs ingestion ($0.50/GB × DEBUG × never-expire default); (4) forgotten environments, unattached storage, idle NAT/ALBs; (5) oversized on-demand databases and expired savings plans. Per-service pricing subtleties only after these.

Fell over? (1) connection/thread-pool exhaustion (serverless burst × relational DB); (2) retry storms (no jitter, timeouts inverted); (3) a zonal resource everyone believed was regional; (4) quota hits during scale-out (Lambda account concurrency is shared — a runaway consumer throttles *unrelated* functions; ENI/IP exhaustion in small Lambda subnets); (5) expired things (certs, presigned URLs, role sessions inside long jobs). Genuine AWS regional failure sits below all of these.

## Failure modes & pitfalls (checklist)

- Lambda → RDS without RDS Proxy: a 500-concurrent burst exhausts `max_connections`.
- Fargate pulling multi-GB images through NAT: S3 gateway + ECR interface endpoints.
- Any DB/cache SG with a CIDR ingress instead of an SG reference is a finding.
- gp2 burst-credit exhaustion (mystery 3pm crawl) → gp3; no reason to create new gp2 in 2026.
- DynamoDB hot partition (one tenant = 80% traffic): shard the key or rethink the model.
- Multi-AZ RDS standby is **not readable**; read replicas are a separate feature (Aurora blurs this).
- Presigned URLs die when the signing role's *session* expires, regardless of stated expiry — long-fuse links need CloudFront signed URLs or longer-lived credentials.
- SQS visibility timeout ≥ 6× p99 processing time (or heartbeat-extend), else mid-processing redelivery.
- ALB idle timeout (60s) vs backend keep-alive: backend keep-alive must *exceed* the ALB's or you get unexplainable 502s.
- S3 + CMK without **Bucket Keys**: ~100× the KMS request charges; one checkbox.
- RDS storage autoscaling ratchets up, never down — snapshot-restore to reclaim.
- Cost-allocation tags must be *activated* in the billing console (not retroactive) — do it the week the tagging standard ships.
- Timeout inversion: Lambda timeout > API Gateway's 29s cap → 504s + client retries + duplicated work; every timeout shorter than its caller's.

## Verification / self-check

1. Every component labeled zonal/regional; the AZ-loss story written, including NAT and LB placement.
2. Top-5 bill lines predicted, data transfer estimated explicitly.
3. No static access keys; policies name resources or binding conditions.
4. Connection math done for anything serverless → relational.
5. Retry/timeout chain: decreasing downstream, bounded, jittered.
6. Service status verified this year for anything recalled from memory (App Runner-class quiet deprecations).
Stopping rule: when the design survives "an AZ dies at peak," "traffic 10×es," and "a skeptical CFO reads the bill line-by-line," ship it and let metrics adjudicate.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)

- Probed 14 claims: 13 baseline (cut/compressed), 0 partial, 1 delta (expanded).
- The one real gap: Opus doesn't know App Runner **closed to new customers April 2026** or that **ECS Express Mode** (late 2025) is the successor — it recommends hand-rolled ECS/Fargate+ALB and calls App Runner "stagnant but alive."
- Everything else probed — Lambda/Fargate cost math, NAT/endpoint economics, presigned-URL session expiry, ALB keep-alive 502s, Glacier small-object arithmetic, Bucket Keys, PassRole escalation, multi-region skepticism, bill/outage priors — Opus produced cold at full specificity; this file is now mostly a verification checklist plus the deprecation-currency discipline.
