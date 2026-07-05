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
- **AWS deprecates quietly.** Services drift into maintenance mode with a blog post, not a siren (App Runner in 2026, WorkMail, gp2 de facto). Any service recommendation recalled from memory rather than verified this year is a liability.

## Compute selection — the questions in order

Ask these in sequence; each answer prunes the tree:

1. **Is the load spiky or steady?** Duty cycle below ~20–30% → serverless (Lambda) wins on cost. Steady 24/7 → containers/EC2 with commitment discounts win; Lambda at constant high throughput costs 3–10× equivalent Fargate/EC2.
2. **How long does one unit of work run?** >15 minutes hard-disqualifies Lambda. Minutes-long batch → Fargate tasks or AWS Batch. Milliseconds-to-seconds request/response → Lambda or Fargate services.
3. **How latency-sensitive is the first request?** Hard p99 latency SLO with idle periods → Lambda cold starts need mitigation (SnapStart or provisioned concurrency) or you go to always-on Fargate/EC2. Async/queue-driven work doesn't care about cold starts — don't pay to fix a non-problem.
4. **Is there per-instance state?** WebSockets held open, in-memory caches, sticky sessions, GPU model weights loaded → long-lived containers (ECS/EKS on Fargate or EC2). Stateless request handling → anything.
5. **What's the cost shape you want?** Per-request billing (Lambda) makes cost track revenue — good for uncertain products. Per-hour billing (Fargate/EC2) is cheaper at scale but is a fixed cost you must right-size.
6. **Do you need Kubernetes specifically?** Only if you already have K8s expertise, need its ecosystem (operators, Helm, service mesh), or run multi-cloud. Otherwise ECS is the same outcome with a fraction of the operational surface. EKS is a commitment to *running Kubernetes*, not just running containers — someone owns upgrades, CNI quirks, and node lifecycle.

| Signal | Points to | Because |
|---|---|---|
| Spiky, short, stateless, <15 min | Lambda | Pay-per-use dominates at low duty cycle |
| Containerized service, steady-ish | ECS on Fargate | Managed containers without cluster ops |
| "Just run my container from a repo" | ECS Express Mode | App Runner's successor (see below) |
| Needs K8s API/operators/mesh | EKS | Only justification for the operational surface |
| GPU, licensed software, OS control, steady max-scale | EC2 | Managed layers subtract control you need |
| Long batch jobs, no latency SLO | Fargate tasks / AWS Batch | Duration caps and cold starts irrelevant |

**As of 2026:** AWS App Runner is in maintenance mode (closed to new customers April 2026). Don't propose it for new builds; the successor for "just run my container from a repo" is **ECS Express Mode** (launched late 2025), which generates the ALB/scaling/ECS plumbing for you. EC2 remains the answer for: OS control, GPUs with specific drivers, licensed software, extreme steady-state cost optimization, and anything needing more memory than Lambda's 10GB cap in a function-shaped unit.

## Storage selection

**S3 classes (current lineup as of 2026: Standard, Intelligent-Tiering, Standard-IA, One Zone-IA, Express One Zone, Glacier Instant Retrieval, Glacier Flexible Retrieval, Glacier Deep Archive):**
- Default to **Intelligent-Tiering** for anything with unknown access patterns — the monitoring fee is trivial and it removes a whole class of "we forgot to lifecycle this" waste. Use Standard only when you *know* it's hot.
- IA classes carry retrieval fees and a 30-day minimum storage duration; Glacier tiers have 90–180-day minimums. Lifecycle rules that transition churning data too early cost *more* than Standard — do the arithmetic on object-age distribution first.
- **Express One Zone** (~7× Standard's storage price) is for single-digit-millisecond, very-high-request-rate workloads (ML training shuffle, interactive analytics). It's a performance product, not a storage product; nothing durable-only belongs there (single AZ).
- Small objects are poison for archive tiers: per-object transition request fees and ~40KB-scale metadata overhead mean a billion 10KB objects should be aggregated (tar/parquet) before archiving.
- Request pricing is real at scale: millions of small PUTs can exceed the storage cost. Batch writes, or use Kinesis Firehose to aggregate before landing.

**EBS vs EFS vs S3:**
- EBS is a disk for one instance — AZ-scoped, attached to a single instance (io2 multi-attach is a niche, not a sharing strategy). Snapshots are the durability story; the volume itself is not backup.
- EFS is a shared POSIX filesystem — use it only when multiple instances genuinely need shared *file semantics* (legacy apps, shared config, shared training checkpoints), and expect ~3× EBS cost and higher per-op latency.
- If the access pattern is write-once/read-many objects, it's S3, full stop. Teams reach for EFS because it "feels like a filesystem" and pay 10× for the familiarity.

**DynamoDB vs RDS/Aurora — the reasoning, not the table:**
- The question is not SQL vs NoSQL; it's **"do I know all my access patterns now?"** DynamoDB demands you design the table around queries in advance; it rewards you with flat single-digit-ms latency at any scale, no connection limits, and true serverless billing. If the product is young and query patterns will churn (ad-hoc filters, reporting, joins), a relational store preserves optionality — pick Aurora Postgres.
- DynamoDB red flags in a design review: "we'll scan and filter" (scans are the anti-pattern), more than ~3 GSIs papering over relational access patterns, hot partition keys (one tenant getting most traffic), or a data team that will want SQL next quarter.
- Aurora vs plain RDS: Aurora for HA-sensitive production (storage replicated 6-way across 3 AZs, sub-minute failover, up to 15 read replicas); RDS for cheaper small instances or engines/extensions Aurora lacks. Aurora Serverless v2 (which as of 2026 supports scale-to-zero) covers the spiky/dev-environment case that used to argue for DynamoDB on cost alone.
- Don't run databases on EC2 unless you have a DBA team and a specific blocker (unsupported extensions, superuser needs, exotic tuning). "We'll save money self-hosting Postgres" is usually mispriced ops time.

**Messaging selection (the glue decides the coupling):** SQS for point-to-point work queues (competing consumers, per-message retry, DLQ); SNS for fan-out to many subscribers; EventBridge for content-based routing, cross-account/SaaS events, and schema-registry discipline; Kinesis for ordered, replayable streams at volume. Choosing Kinesis for a job queue or SQS for telemetry streaming are both category errors — pick by delivery semantics (queue vs broadcast vs routed vs ordered stream), not by familiarity.

## Networking mental model

- **VPC = your private address space; subnets are AZ-scoped.** Public subnet = route to an Internet Gateway. Private subnet egress goes through a NAT gateway — which charges ~$0.045/GB processed on top of hourly cost. This is the single most common bill-shock source.
- **Security groups vs NACLs:** SGs are stateful, attached to ENIs, and support referencing other SGs ("allow 5432 from sg-app") — do all your segmentation here, expressing intent as SG-to-SG edges. NACLs are stateless subnet-level allow/deny lists; leave them at default except for coarse guardrails (block a hostile CIDR). Teams that build fine-grained NACL rules create stateless return-traffic bugs (ephemeral port ranges) for zero security benefit.
- **VPC endpoints are the NAT-tax escape hatch.** Gateway endpoints for S3 and DynamoDB are *free* — there is almost never a reason not to add them to every VPC. Interface endpoints (PrivateLink) cost hourly + per-GB but keep traffic off the public path and dodge NAT processing; they pay for themselves the moment a private subnet pushes real volume to any AWS API (ECR, CloudWatch, STS, Secrets Manager).
- **PrivateLink is also the cross-account/SaaS pattern:** expose a service to other VPCs via an endpoint service instead of peering — consumers get an ENI, you don't merge address spaces, and the blast radius stays one-directional.
- **Egress cost traps, ranked by how often they bite:**
  1. NAT gateway processing for traffic to S3/ECR that a free gateway endpoint would eliminate.
  2. Cross-AZ traffic at $0.01/GB *each direction* — chatty microservices, Kafka replication, and misconfigured load balancing across AZs add up; enable zone-aware routing where offered.
  3. Internet egress ~$0.09/GB (first 10TB tier) — serve large public assets via CloudFront (S3→CloudFront transfer is free), never straight from S3/ALB.
  4. Cross-region replication and pulls nobody remembers turning on.

**Edge and global layer:** CloudFront belongs in front of anything serving the public internet — not only for latency but because origin-to-CloudFront transfer is free (egress arbitrage), it absorbs L7 floods with AWS Shield Standard, and it's where WAF attaches cleanly. Route 53 health checks + failover routing is the cheap 80% of a DR story. Global Accelerator is for non-HTTP or when you need static anycast IPs — don't buy it for a plain website; CloudFront already does that job.

## IAM reasoning

- **Roles, never users, for workloads.** Long-lived access keys are the top real-world breach vector. EC2 instance profiles, ECS task roles, Lambda execution roles, EKS Pod Identity/IRSA, OIDC federation for CI (GitHub Actions → `sts:AssumeRoleWithWebIdentity`). A design containing an access key in an env var is wrong until proven otherwise.
- **Least privilege is iterative, not aspirational.** Start narrower than you think and widen from observed AccessDenied errors; or run broad in dev and narrow using IAM Access Analyzer's policy generation from CloudTrail. Writing the perfect policy from first principles is how people end up with `"Action": "*"` "temporarily," forever.
- **Scope by resource ARN and conditions, not just action lists.** `s3:GetObject` on `*` is not least privilege. Conditions (`aws:SourceVpce`, `aws:PrincipalOrgID`, `s3:prefix`, `aws:ResourceTag`) bind permissions to context and survive resource sprawl better than enumerating ARNs.
- **Permission boundaries** are for delegation: they cap what a principal can hold or grant, letting a platform team allow product teams to create roles without privilege-escalation risk. A boundary alone grants nothing — effective permissions are the *intersection* of boundary and policy. SCPs are the same idea at the org/account level.
- **Multi-account is the real isolation primitive.** Separate prod/dev/security/log-archive accounts under Organizations beats any amount of within-account IAM cleverness — blast radius, billing attribution, and quota isolation all come free with the split.
- **Resource policies complete the graph:** S3 bucket policies, KMS key policies, and SQS queue policies are evaluated *with* identity policies. Cross-account access needs both sides; debugging "AccessDenied despite the role allowing it" is usually a missing resource-policy half or an explicit deny in an SCP.

## Multi-AZ vs multi-region

Prior: **multi-AZ yes, multi-region almost always no.** Regional outages happen (roughly annually, somewhere) but multi-region done properly costs 1.5–2× infrastructure plus a permanent engineering tax: data replication conflicts, doubled deployment pipelines, drifted configs, and a failover that decays unless exercised. Most "multi-region DR" setups fail their first real failover because it was never tested — the company paid double for a false sense of security.

The questions that justify multi-region, in order of frequency:
1. **Latency to distant users** — that's multi-region *serving* (CloudFront, regional read replicas, Global Accelerator), a much easier problem than multi-region *failover*. Don't let it smuggle in active-active writes.
2. **Regulatory mandate / data residency** — non-negotiable, but usually means region-pinned data, not full duplication.
3. **Downtime cost genuinely in the millions per hour** — then fund active-passive with automated, *quarterly-exercised* failover, or active-active with per-key home regions. Anything less than funded game days is theater.

If none apply, spend the budget on: cross-region backup copies (S3 replication, RDS snapshot copy), IaC that can rebuild the stack in a fresh region in hours, and restore drills. That "pilot light on paper" covers the realistic disasters — data corruption, account compromise, region evacuation with notice — at ~5% of the cost.

**Well-Architected pillars, applied not recited:** use them as a review checklist by asking each pillar's sharpest question. Reliability: "what happens when this AZ disappears mid-write?" Security: "which principal can exfiltrate the data store, and would we see it?" Cost: "which top-5 line items are load-proportional vs fixed?" Operational excellence: "how do we ship a fix at 3am, and how do we know it worked?" Performance: "where's the first bottleneck as load 10×es?" If a design doc can't answer one of these in a sentence, that pillar is the gap.

## How an expert thinks through this

*"We're building a document-processing API: users upload PDFs (1–50MB), we OCR + extract, results queryable later. ~50k documents/day, bursty (9am spike), processing takes 30–120s per doc."*

Data first: PDFs land in S3 via presigned upload URLs — never proxy 50MB uploads through compute; that's paying Lambda/Fargate by the second to be a pipe. Results are structured extractions with query patterns unclear this early → Aurora Postgres, not DynamoDB; the product will want ad-hoc filters within months and JSONB covers the semi-structured parts. Reject DynamoDB explicitly: access patterns unknown, and 50k writes/day is nothing — scale isn't the constraint here, flexibility is. Reject OpenSearch for now: search is a "later" requirement and Postgres full-text will carry it surprisingly far.

Processing: 30–120s per doc — under Lambda's 15-minute cap, so Lambda is *possible*. Check the resource shape: OCR is CPU-heavy; if CPU-only and under 10GB memory, Lambda triggered from SQS is the cheapest bursty option and rides the 9am spike with zero tuning. If a GPU is required, Lambda is out → ECS on GPU EC2 with queue-depth autoscaling, and cold capacity vs queue latency becomes the design conversation. Decision: Lambda container image (OCR deps are multi-GB, zip won't fit) unless the GPU requirement is proven by benchmark, not assumed.

API layer: spiky, stateless, sub-second handlers → Lambda + API Gateway fits. Counter-consideration: the team already ships one container; a single small Fargate service behind an ALB is operationally simpler than lambda-per-route sprawl. Either is defensible — pick by team shape, and say so in the doc rather than pretending one is objectively right. Reject EKS outright: no K8s requirement, no K8s team; it would be a second job.

Networking: Lambda in-VPC to reach Aurora → add the S3 gateway endpoint immediately (free; otherwise NAT processes 50k × 25MB ≈ 1.25TB/day at $0.045/GB ≈ $56/day of pure waste). Put RDS Proxy in front of Aurora, because burst concurrency at 9am will exhaust Postgres connections — that connection exhaustion is the outage this design would otherwise ship with; the proxy is not optional.

Resilience: Aurora multi-AZ, SQS regionally durable, S3 regionally durable — the AZ-loss story is complete. Multi-region? An hour of downtime delays document processing; the queue absorbs the backlog; nobody dies. No. Cross-region snapshot copies and rebuildable IaC, yes. Write that reasoning down so nobody "upgrades" it later without new facts.

## Bill-shock and outage priors — what it usually is

When asked "why did the AWS bill jump?", check in this order (frequency-ranked from real reviews):
1. NAT gateway data processing (new data pipeline, ECR pulls, or an SDK talking to S3 from private subnets).
2. Cross-AZ transfer from a chatty new service, replicated queue/stream, or an LB spraying across zones.
3. CloudWatch Logs ingestion (new DEBUG logger, never-expire retention) or a runaway metrics cardinality bill.
4. Forgotten environments and unattached storage — snapshots, volumes, idle NAT/ALBs, stopped-but-billing resources.
5. On-demand databases someone sized "to be safe," and expired savings plans nobody renewed.
Only after these: per-service pricing subtleties (DynamoDB on-demand vs provisioned, S3 request fees, KMS).

When asked "why did it fall over?", the priors are:
1. Connection exhaustion (serverless burst × relational DB) or thread-pool exhaustion upstream of a slowed dependency.
2. Retry storms amplifying a partial failure into a full one (no jitter, no budget, timeouts inverted).
3. A zonal resource everyone believed was regional (single NAT, one-AZ targets, single-node cache).
4. Quota/limit hits during scale-out: account Lambda concurrency, EC2 vCPU quotas, ENI/IP exhaustion in small subnets.
5. Expired things: certificates, presigned URLs, IAM session assumptions inside long jobs.
Genuine AWS-side regional failure sits far below all of these — design and debug accordingly.

## Failure modes & pitfalls

- **Lambda → RDS without RDS Proxy.** Each concurrent Lambda opens its own connection; a 500-concurrent burst exhausts `max_connections` and takes the DB down for everyone. Proxy, or cap reserved concurrency deliberately.
- **NAT gateway processing on ECR/S3 pulls.** Fargate tasks in private subnets pulling multi-GB images through NAT on every scale-out. Fix: S3 gateway endpoint (ECR layers ride S3) + ECR interface endpoints.
- **Security group `0.0.0.0/0` on a database port** "temporarily for debugging." Any DB/cache SG with a CIDR ingress instead of an SG reference is a finding.
- **gp2 burst-credit exhaustion:** small busy gp2 volume runs out of IOPS credits and the DB mysteriously crawls at 3pm daily. Migrate to gp3 (independent IOPS/throughput, cheaper) — as of 2026 there is no reason to create new gp2.
- **DynamoDB hot partition:** `tenant_id` as partition key with one tenant at 80% of traffic → throttling far below provisioned capacity. Shard the hot key (`tenant_id#0..N`) or rethink the model.
- **S3 lifecycle to Glacier on small churning objects** — transition request fees plus 90–180-day minimum-storage charges exceed the savings. Compute break-even before adding the rule.
- **Multi-AZ RDS mistaken for read scaling.** The Multi-AZ standby (non-cluster) is not readable; read replicas are a separate feature. Aurora blurs this (replicas serve both roles) — know which engine you're on before promising read capacity.
- **CloudWatch Logs at $0.50/GB ingestion** quietly becoming a top-5 line item because a debug logger ships request bodies, and retention defaults to *never expire*. Set retention everywhere; sample verbose logs.
- **`iam:PassRole` with `Resource: "*"`** — the classic escalation: whoever can pass an admin role to a service they control *is* admin. Always scope PassRole to specific role ARNs.
- **Zonal resources treated as regional:** ALB targets all in one AZ; a single NAT gateway shared by all AZs (AZ outage severs egress for the *surviving* AZs — one NAT per AZ for real HA); single-node ElastiCache. Walk the diagram labeling each component zonal/regional.
- **The retry-storm outage:** downstream slows → Lambda hits API Gateway's 29s cap → clients retry → account-level concurrency exhausts → *unrelated* functions throttle (concurrency is shared account-wide). Reserve concurrency for critical functions; set timeouts shorter than your caller's.
- **VPC Lambda IP exhaustion:** functions in a /28-subnet VPC config run out of ENI IPs at scale-out. Give Lambda subnets generous CIDRs; they're free.
- **KMS request costs and S3:** encrypting millions of objects with a CMK without **S3 Bucket Keys** enabled multiplies KMS request charges ~100×. One checkbox.
- **Presigned URL expiry vs role session:** presigned S3 URLs die when the signing role's session expires, regardless of the URL's stated expiry — sign with longer-lived credentials for long-fuse links.
- **SQS visibility timeout shorter than processing time** → messages re-delivered to a second worker mid-processing → "mysterious duplicates" that idempotency alone shouldn't have to absorb. Visibility timeout ≥ 6× the p99 processing time when using batch-and-extend patterns, or extend heartbeat-style.
- **ALB idle timeout (default 60s) vs backend keep-alive:** if the target's keep-alive is shorter than the ALB's, the ALB reuses a connection the backend just closed → intermittent 502s that "make no sense." Backend keep-alive must exceed the ALB idle timeout.
- **RDS storage autoscaling ratchets up but never down** — a one-off import permanently inflates the storage bill. Snapshot-restore into a right-sized instance to reclaim.
- **Cost allocation tags not activated:** tags exist on resources but were never activated in the billing console, so six months of Cost Explorer grouping is empty. Activate cost-allocation tags the week the tagging standard ships.

## Worked micro-examples

**Cost shape of one compute decision.** API doing 2M requests/day, 150ms avg @ 512MB:
- Lambda: 2M × 30 × $0.20/1M ≈ $12 (requests) + 60M × 0.15s × 0.5GB × ~$0.0000167/GB-s ≈ $75 (compute) ≈ **~$90/mo**, scaling to zero at night.
- Fargate equivalent (2 × 0.5 vCPU/1GB always-on) ≈ **~$55/mo** + ~$20 ALB — cheaper *if* permanently right-sized, but absorbs a 10× spike only with autoscaling you now own.
- At 20M req/day, Lambda ≈ $900 while Fargate roughly 3×es — the crossover is real; compute it per workload rather than assuming either religion.

**Least-privilege with conditions instead of ARN sprawl:**
```json
{
  "Effect": "Allow",
  "Action": ["s3:GetObject", "s3:PutObject"],
  "Resource": "arn:aws:s3:::acme-docs-prod/*",
  "Condition": {
    "StringEquals": { "aws:PrincipalTag/team": "ingest" },
    "StringLike":  { "s3:prefix": "incoming/*" }
  }
}
```
The condition block is what keeps this policy correct when the bucket grows new prefixes for other teams — least privilege that survives change beats least privilege that was only accurate on day one.

## Verification / self-check

Before presenting an AWS design, confirm:
1. Every component labeled zonal/regional; the AZ-loss story written down, including NAT and load balancer placement.
2. Predicted top-5 bill lines, with data transfer estimated explicitly (NAT processing, cross-AZ, egress).
3. No workload uses static access keys; every role's policy names specific resources or binding conditions.
4. Connection math done for anything serverless talking to anything relational.
5. Retry/timeout chain checked: every timeout shorter than its caller's, retries bounded and jittered.
6. Service names verified current — if a service's status wasn't confirmed this year (App Runner-class quiet deprecations), check before recommending.
Stopping rule: when the design survives "an AZ dies at peak," "traffic 10×es," and "the bill is read line-by-line by a skeptical CFO," further architecture is speculation — ship it and let metrics adjudicate the rest.
