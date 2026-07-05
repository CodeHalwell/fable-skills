---
name: cloud-architecture-aws
description: Load when designing, reviewing, or debugging architectures on AWS — choosing compute (Lambda/Fargate/ECS/EC2), storage (S3/EBS/EFS/DynamoDB/RDS), VPC networking, IAM policies, multi-AZ/multi-region decisions, or diagnosing AWS bill shock and outage patterns. Also load when someone asks "which AWS service should I use for X".
---

# AWS Architecture Judgment

## Core mental model

- **Managed-ness is the axis, and you pay for it twice.** Every AWS choice sits on a spectrum from "AWS runs it" (Lambda, DynamoDB, Aurora Serverless) to "you run it on their metal" (EC2, self-managed Postgres). Managed costs more per unit but less per engineer-hour; the crossover point is utilization. Steady 24/7 load pushes you down the spectrum; spiky, low-duty-cycle load pushes you up. Most teams misprice their own ops time and sit too low.
- **Data gravity dominates.** Compute is cheap to move and swap; data is not. Egress fees, cross-AZ charges, and migration pain all attach to data. Decide where the data lives first, then put compute next to it — never the reverse.
- **Everything fails at the AZ boundary; design there.** AWS's real reliability contract is "an AZ can vanish." Multi-AZ is table stakes and mostly free in managed services. Region-level failure is rare enough that most multi-region designs are wasted premium (see below).
- **IAM is the actual architecture.** Services are commodity; the permission graph is what makes a design secure or a breach. Reason about "which principal can do what to which resource" before drawing boxes.
- **The bill is a design review you get monthly.** NAT gateway data processing, cross-AZ chatter, unattached EBS volumes, and CloudWatch ingestion are where designs leak. If you can't predict the top 5 line items of your design, you don't understand it yet.

## Compute selection — the questions in order

Ask these in sequence; each answer prunes the tree:

1. **Is the load spiky or steady?** Duty cycle below ~20–30% → serverless (Lambda) wins on cost. Steady 24/7 → containers/EC2 with commitment discounts win; Lambda at constant high throughput costs 3–10× equivalent Fargate/EC2.
2. **How long does one unit of work run?** >15 minutes hard-disqualifies Lambda. Minutes-long batch → Fargate tasks or AWS Batch. Milliseconds-to-seconds request/response → Lambda or Fargate services.
3. **How latency-sensitive is the first request?** Hard p99 latency SLO with idle periods → Lambda cold starts need mitigation (SnapStart or provisioned concurrency) or you go to always-on Fargate/EC2. Async/queue-driven work doesn't care about cold starts — don't pay to fix a non-problem.
4. **Is there per-instance state?** WebSockets held open, in-memory caches, sticky sessions, GPU model weights loaded → long-lived containers (ECS/EKS on Fargate or EC2). Stateless request handling → anything.
5. **What's the cost shape you want?** Per-request billing (Lambda) makes cost track revenue — good for uncertain products. Per-hour billing (Fargate/EC2) is cheaper at scale but is a fixed cost you must right-size.
6. **Do you need Kubernetes specifically?** Only if you already have K8s expertise, need its ecosystem (operators, Helm, service mesh), or run multi-cloud. Otherwise ECS is the same outcome with a fraction of the operational surface. EKS is a commitment to running Kubernetes, not just running containers.

**As of 2026:** AWS App Runner is in maintenance mode (closed to new customers April 2026). Don't propose it for new builds; the successor for "just run my container from a repo" is **ECS Express Mode** (launched late 2025), which generates the ALB/scaling/ECS plumbing for you. EC2 remains for: OS control, GPUs with specific drivers, licensed software, extreme steady-state cost optimization, and anything needing >10 GB memory in a function-shaped unit.

## Storage selection

**S3 classes (current lineup as of 2026: Standard, Intelligent-Tiering, Standard-IA, One Zone-IA, Express One Zone, Glacier Instant Retrieval, Glacier Flexible Retrieval, Glacier Deep Archive):**
- Default to **Intelligent-Tiering** for anything with unknown access patterns — the monitoring fee is trivial and it removes a whole class of "we forgot to lifecycle this" waste. Use Standard only when you *know* it's hot.
- IA classes have retrieval fees and 30-day minimum storage; Glacier tiers have 90–180-day minimums. Lifecycle rules that transition objects too early on churning data cost *more* than Standard — do the arithmetic on object age distribution first.
- **Express One Zone** (~7× Standard's storage price) is for single-digit-millisecond, high-request-rate workloads (ML training shuffle, interactive analytics) — it's a performance product, not a storage product.
- Small objects are poison for archive tiers: per-object transition requests and 40KB metadata overhead mean a billion 10KB objects should be aggregated (tar/parquet) before archiving.

**EBS vs EFS vs S3:** EBS is a disk for one instance (per-AZ, attachable to one instance barring io2 multi-attach niches). EFS is a shared POSIX filesystem — use it when multiple instances genuinely need shared file semantics, and expect ~3× the cost and higher latency than EBS. If the access pattern is write-once/read-many objects, it's S3, full stop — teams reach for EFS because it "feels like a filesystem" and pay 10× for the familiarity.

**DynamoDB vs RDS/Aurora — the reasoning, not the table:**
- The question is not SQL vs NoSQL; it's **"do I know all my access patterns now?"** DynamoDB demands you design the table around queries in advance; it rewards you with flat single-digit-ms latency at any scale and no connection limits. If the product is young and query patterns will churn (ad-hoc filters, reporting, joins), a relational store preserves optionality — pick Aurora Postgres.
- DynamoDB red flags in a design review: "we'll scan and filter" (scans are the anti-pattern), more than ~3 GSIs papering over relational access patterns, hot partition keys (single-tenant key getting most traffic).
- Aurora vs plain RDS: Aurora for HA-sensitive production (storage replicated 6-way across 3 AZs, fast failover, up to 15 read replicas); RDS for cheaper small instances, or engines/extensions Aurora lacks. Aurora Serverless v2 (which as of 2026 supports scale-to-zero) covers the spiky/dev-environment case that used to argue for DynamoDB on cost alone.
- Don't run databases on EC2 unless you have a DBA team and a specific reason (unsupported extensions, superuser needs, exotic tuning).

## Networking mental model

- **VPC = your private address space; subnets are AZ-scoped.** Public subnet = route to an Internet Gateway. Private subnet egress goes through a NAT gateway — which charges ~$0.045/GB processed on top of hourly cost. This is the single most common bill-shock source.
- **Security groups vs NACLs:** SGs are stateful, attached to ENIs, and support referencing other SGs ("allow from sg-app") — do all your segmentation here. NACLs are stateless subnet-level allow/deny lists; leave them at default except for coarse guardrails (block a CIDR). Teams that build fine-grained NACL rules create unstateful return-traffic bugs (ephemeral port ranges) for zero benefit.
- **VPC endpoints are the NAT-tax escape hatch.** Gateway endpoints for S3 and DynamoDB are *free* — there is almost never a reason not to add them to every VPC. Interface endpoints (PrivateLink) cost hourly + per-GB but keep traffic private and dodge NAT processing; they pay for themselves the moment a private subnet pushes real volume to any AWS API.
- **Egress cost traps, ranked by how often they bite:** (1) NAT gateway processing for traffic to S3/ECR that a free gateway endpoint would eliminate; (2) cross-AZ traffic at $0.01/GB *each direction* — chatty microservices and Kafka replication across AZs add up; enable AZ-aware routing/rack awareness; (3) internet egress ~$0.09/GB — serve large public assets via CloudFront (S3→CloudFront transfer is free) not straight from S3/ALB.

## IAM reasoning

- **Roles, never users, for workloads.** Long-lived access keys are the top real-world breach vector. EC2 instance profiles, ECS task roles, Lambda execution roles, IRSA/Pod Identity on EKS, OIDC federation for CI (GitHub Actions → `sts:AssumeRoleWithWebIdentity`). A design containing an access key in an env var is wrong until proven otherwise.
- **Least privilege is iterative, not aspirational.** Start narrower than you think and widen from observed AccessDenied; or start broad in dev and narrow using IAM Access Analyzer's policy generation from CloudTrail. Writing the perfect policy from first principles is how people end up with `"Action": "*"` "temporarily."
- **Scope by resource ARN and conditions, not just action lists.** `s3:GetObject` on `*` is not least privilege. Use conditions (`aws:SourceVpce`, `aws:PrincipalOrgID`, `s3:prefix`) to bind permissions to context.
- **Permission boundaries** are for delegation: they cap what a principal can grant/hold, letting platform teams allow dev teams to create roles without privilege-escalation risk. They don't grant anything — a boundary alone gives no access. SCPs (org-level) are the same idea for whole accounts.
- Multi-account is the real isolation primitive: separate prod/dev/security accounts under Organizations beats any amount of within-account IAM cleverness.

## Multi-AZ vs multi-region

Prior: **multi-AZ yes, multi-region almost always no.** Regional outages happen (roughly annually somewhere) but multi-region done properly costs 1.5–2× infra plus a permanent engineering tax (data replication conflicts, deployment complexity, drifted configs). Most "multi-region DR" setups fail their first real failover because it was never exercised — you paid double for a false sense of security.

Ask: (1) What does an hour of full downtime actually cost, in dollars? (2) Is there a regulatory mandate? (3) Do you need low latency for far-away users (that's multi-region *serving*, a different, easier problem — CloudFront + regional read replicas)? If the honest answers don't sum to "millions or mandate," spend the budget on: tested backups in another region (S3 cross-region replication, RDS snapshots copied cross-region), infrastructure-as-code that can rebuild the stack in a new region in hours, and game days. That's "pilot light on paper" and it covers the realistic disaster (data corruption, account compromise, region evacuation with hours of notice) at 5% of the cost.

Well-Architected pillars, applied not recited: use them as a review checklist by asking each pillar's sharpest question — Reliability: "what happens when this AZ disappears mid-write?"; Security: "which principal can exfiltrate the data store?"; Cost: "what are the top 5 line items and which are load-proportional?"; Operations: "how do we deploy a fix at 3am?"; Performance: "where's the first bottleneck as load 10×es?"; Sustainability rides along with cost.

## How an expert thinks through this

*"We're building a document-processing API: users upload PDFs (1–50MB), we OCR + extract, results queryable later. ~50k documents/day, bursty (9am spike), processing takes 30–120s per doc."*

Data first: PDFs land in S3 (presigned upload URLs — never proxy 50MB uploads through compute). Results are structured extractions, query patterns unclear this early → Aurora Postgres, not DynamoDB; product will want ad-hoc filters within months and JSONB covers the semi-structured parts. Reject DynamoDB: access patterns unknown, and 50k writes/day is nothing — scale isn't the constraint, flexibility is.

Processing: 30–120s per doc. Under Lambda's 15-min cap, so Lambda is *possible* — but check memory/CPU: OCR is CPU-heavy and may want >10GB or a GPU. If CPU-only and <10GB, Lambda from an SQS queue is the cheapest bursty option and scales to the 9am spike with zero tuning. If it needs GPU, Lambda is out → ECS on EC2 GPU instances with queue-depth autoscaling, and now cold capacity vs queue latency is the design conversation. Take Lambda (container image, since OCR deps are huge) unless the GPU requirement is proven.

API layer: spiky, stateless, sub-second handlers → Lambda + API Gateway fits; but if the team already ships one container, a single small Fargate service behind an ALB is operationally simpler than a lambda-per-route sprawl. Either is defensible; pick by team shape. Reject EKS outright: no K8s requirement, no K8s team.

Networking: Lambda in-VPC to reach Aurora → add S3 gateway endpoint (free, else NAT eats $0.045/GB × 50k×25MB ≈ $56/day just for uploads flowing to processing), RDS Proxy in front of Aurora because Lambda concurrency will exhaust Postgres connections at the 9am burst. That connection-exhaustion is the bug this design would have shipped with; the proxy is not optional.

Resilience: Aurora multi-AZ, SQS is regionally durable, S3 is regionally durable. Multi-region? An hour of downtime delays document processing — nobody dies, queue absorbs the backlog. No. Cross-region snapshot copies, yes.

## Failure modes & pitfalls

- **Lambda → RDS without RDS Proxy.** Each concurrent Lambda opens its own connection; a burst of 500 concurrents exhausts `max_connections` and takes down the DB for everything else. Use RDS Proxy or cap function concurrency deliberately.
- **NAT gateway processing on ECR/S3 pulls.** Fargate tasks in private subnets pulling multi-GB images through NAT on every scale-up event. Fix: S3 gateway endpoint (ECR layers come from S3) + ECR interface endpoints.
- **Security group referencing gone stale:** allowing `0.0.0.0/0` on 5432 "temporarily for debugging." Check: any DB/cache SG with a CIDR ingress instead of an SG reference is a finding.
- **gp2 volumes and burst-credit exhaustion.** gp2 IOPS scale with size and burst; a small busy volume runs out of credits and the DB mysteriously crawls. Migrate to gp3 (independent IOPS/throughput, cheaper) — as of 2026 there is essentially no reason to create new gp2.
- **DynamoDB hot partitions:** using `tenant_id` as partition key when one tenant is 80% of traffic → throttling at a fraction of provisioned capacity. Shard the hot key (`tenant_id#0..N`) or rethink the key.
- **S3 lifecycle to Glacier on small churning objects** — transition request fees plus 90-day minimum-storage charges exceed the savings. Compute break-even before adding the rule.
- **Multi-AZ RDS mistaken for a read-scaling feature.** The standby in Multi-AZ (non-cluster) is not readable. Read replicas are a separate thing. Aurora blurs this (replicas are both) — know which you're on.
- **CloudWatch Logs ingestion at $0.50/GB** quietly becoming a top-5 line item because a debug logger ships full request bodies. Set retention (default is *never expire*) and sample verbose logs.
- **IAM `iam:PassRole` with `Resource: "*"`** — the classic privilege escalation: anyone who can pass an admin role to a service they control is admin. Always scope PassRole to specific role ARNs.
- **Zonal services treated as regional:** an ALB with targets in one AZ, a single NAT gateway shared by all AZs (AZ outage severs egress for surviving AZs — one NAT per AZ for real HA), ElastiCache single-node. Walk every component and label it zonal/regional.
- **The retry storm outage:** downstream slows → Lambdas time out at 29s (API Gateway cap) → clients retry → concurrency limit exhausted → unrelated functions in the account throttle (account-level concurrency is shared). Set per-function reserved concurrency for anything critical, and timeouts shorter than your caller's.

## Worked micro-example — cost shape of one decision

API doing 2M requests/day, 150ms avg @ 512MB:
- Lambda: 2M × 30 days × ($0.20/1M) = $12 req + 60M × 0.15s × 0.5GB × $0.0000167/GB-s ≈ $75 compute ≈ **~$90/mo**, scales to zero at night.
- Fargate equivalent (2 × 0.5 vCPU/1GB, always on): ≈ **~$55/mo** + ALB ~$20 — cheaper *if* always right-sized, but doesn't absorb 10× spikes without autoscaling config, and you now own patching-adjacent config.
At 20M req/day the Lambda bill 10×es (~$900) while Fargate maybe 3×es — the crossover is real and worth computing per workload, not assumed.

## Verification / self-check

Before presenting an AWS design, confirm:
1. Every component labeled zonal/regional; the AZ-loss story is written down.
2. Predicted top-5 bill lines, with data transfer explicitly estimated (NAT, cross-AZ, egress).
3. No workload uses static access keys; every role's policy names specific resources.
4. Connection math done for anything serverless talking to anything relational.
5. Service names verified current — AWS deprecates quietly (App Runner 2026, gp2 de facto); if a service wasn't confirmed alive this year, check before recommending.
Stopping rule: when the design survives "an AZ dies at peak," "traffic 10×es," and "the bill is read line-by-line by a skeptical CFO," further architecture is speculation — ship and measure.
