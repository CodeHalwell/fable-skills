---
name: cloud-architecture-azure
description: Load when designing, reviewing, or debugging architectures on Microsoft Azure — choosing compute (Functions/Container Apps/AKS/App Service), Entra ID and managed identities, storage/database selection (Blob, Cosmos DB, Azure SQL, PostgreSQL), VNets and private endpoints, Bicep vs Terraform, landing zones, or Azure OpenAI/Foundry integration. Also load for AWS-to-Azure translation questions.
---

# Azure Architecture Judgment

## Core mental model

- **Reason from Entra ID outward** (not from the VNet, as on AWS). Every workload holds a managed identity; a connection string with a password is the first thing to question in review.
- **The resource hierarchy is the control plane.** Management groups → subscriptions → resource groups; subscriptions are the real isolation/billing boundary and are meant to be numerous. On Azure, policy and landing zones *assume* this tree exists — deciding it is part of the architecture, not an afterthought.
- **Azure renames things; verify names before writing them down.** AD → Entra ID; Azure AI Studio → AI Foundry → (2026, progressively) *Microsoft Foundry*; classic Consumption → Flex Consumption; PostgreSQL Single Server → retired. Any Azure service name recalled from memory more than a year old is stale until checked — dead names break IaC and credibility.
- **Tier cliffs are the Azure-specific bill trap:** many services price in chunky SKUs where one checkbox feature (VNet integration, zone redundancy, sessions) forces the next tier for the whole resource. Budget by naming the tier-forcing feature. The other silent killers (egress, log ingestion, idle premium SKUs) carry over from AWS.

## Compute selection

1. Event-glue (queue/timer/webhook) → **Functions on Flex Consumption** (the 2026 serverless default; classic Consumption is legacy and cannot do VNet integration). Premium only for features Flex lacks in your region — check, don't assume.
2. Containerized services, Dapr/KEDA, scale-to-zero → **Container Apps** (managed K8s underneath without exposing it).
3. Need the actual Kubernetes API (CRDs/operators/mesh/GPU control) → **AKS — prefer AKS Automatic for new clusters** unless you need the knobs it removes. Choosing AKS is hiring yourself as a cluster operator; say so in the review.
4. Plain web app, .NET/Java team, deployment slots wanted → **App Service**. Long/heavy batch → ACA Jobs / Azure Batch, not stretched function timeouts.
5. Consumption billing wins below roughly 30–40% sustained utilization; above that, dedicated capacity + reservations. Compute the duty cycle; don't vibe it.

## Identity: the no-secrets architecture

Managed identity per workload + `DefaultAzureCredential` + data-plane RBAC roles on target resources; Key Vault only for third-party secrets you can't eliminate (via Key Vault references); `allowSharedKeyAccess: false` / `disableLocalAuth` enforced by Azure Policy; workload identity federation (OIDC) for CI — no service-principal client secrets with 2-year expiries. The design goal: app config contains **zero keys, passwords, or SAS tokens**. (Note: control-plane Contributor is not data-plane access — grant the data role explicitly, and expect a few minutes of role-assignment propagation lag; add retry to grant-then-use scripts.)

## Storage and databases

- **Blob tiers** Hot/Cool/Cold/Archive with 30/90-day minimums and hours-long Archive rehydration — same early-tiering trap as S3; model object age first.
- **Cosmos DB:** buy for global multi-region writes, guaranteed single-digit-ms latency, or elastic RU billing. Session consistency is the right user-facing default; Strong doubles read RU and forbids multi-region writes — push back on "Strong to be safe" with the RU bill. Watch normalized RU consumption for 429s, not averages. RU envelope: 1KB point read ≈ 1 RU, 1KB write ≈ 5–6 RU; if the math lands 10× over budget, fix the partition key or query shape, not the throughput dial.
- **Azure SQL vs PostgreSQL Flexible Server** is team gravity, not technology: SQL Server skills/licenses (Hybrid Benefit) → Azure SQL (vCore model, not DTU, for anything serious); open-source/extensions/portability → **Flexible Server** — the only Postgres that should appear in a 2026 design (Single Server is retired; Citus lives on as Cosmos DB for PostgreSQL).
- Queues: Service Bus (enterprise messaging: sessions, dedup, topics) / Storage Queues (cheap-basic) / Event Grid (reactive discrete events) / Event Hubs (streaming firehose, Kafka-protocol endpoint). Pick by delivery semantics.

## Networking

- **VNets are regional; subnets are not AZ-bound** (unlike AWS — zone redundancy is a property of the resource, not the subnet). Delegated subnets get consumed by PaaS injection; size address space generously.
- **Private endpoints are the default PaaS posture, and the #1 failure is DNS:** endpoint + `privatelink.*` zone + zone-link to **every consumer VNet** ship as one unit (peering does not propagate DNS; custom DNS must forward to 168.63.129.16). Check DNS resolution before anything else when "it's down."
- Outbound-heavy App Service/Functions exhaust SNAT ports (~128/instance) and it looks like the remote API's fault — fix structurally with connection reuse + VNet integration + NAT Gateway.
- **Azure does not charge for cross-AZ traffic within a region** — don't import AWS zone-locality contortions; make everything zone-redundant by default. Internet egress and NAT per-GB still apply.
- L4 → Load Balancer; regional L7/WAF → Application Gateway; global HTTP entry → Front Door — and lock origins to your Front Door instance (Private Link origin, or `AzureFrontDoor.Backend` tag *plus* `X-Azure-FDID` check — the tag alone admits every Front Door tenant) or the WAF is decorative. Don't stack Front Door + App Gateway without a stated reason.

## AWS-to-Azure translation (where the mapping lies)

| AWS | Azure | The catch |
|---|---|---|
| Account | Subscription | Cheaper/more numerous; management groups do the org layer |
| IAM roles + policies | Entra ID + RBAC | Data-plane roles are separate grants per resource |
| Lambda | Functions (Flex Consumption) | Plan choice changes capabilities, not just price |
| ECS/Fargate | Container Apps | KEDA/Dapr baked in; less raw control |
| EKS | AKS (Automatic) | More managed than EKS defaults |
| SQS / SNS / EventBridge | Storage Queues or Service Bus / SB topics / Event Grid | Service Bus covers queue+topic roles |
| Kinesis | Event Hubs | Kafka-protocol endpoint is often the real reason |
| DynamoDB | Cosmos DB | RU billing + 5 consistency levels — easier to overspend |
| CloudFormation | ARM/Bicep | No state file; drift semantics differ from Terraform |
| CloudWatch | Azure Monitor + Log Analytics + App Insights | Three brands, one ingestion bill |
| VPC endpoints / PrivateLink | Private endpoints + Private DNS zones | The DNS half is mandatory and is the usual failure |

Cost levers with no AWS equivalent: **Azure Hybrid Benefit** (often the largest single discount — ask "do they own licenses?" first) and **Dev/Test subscription pricing** for non-prod (a discount class AWS lacks; forgetting it is a standing bill leak).

## IaC and landing zones (as of 2026 — the fast-moving part)

- **Bicep vs Terraform:** Azure-only → Bicep (day-zero ARM coverage, no state file); multi-cloud → Terraform/OpenTofu with the **AzAPI** provider covering `azurerm` gaps. One source of truth per scope.
- **Landing zones:** Microsoft has consolidated on **Azure Verified Modules (AVM)**. The Bicep AVM Platform Landing Zone went **GA in early 2026**; the legacy **ALZ-Bicep repo is archived**; the Terraform **`caf-enterprise-scale` module is in extended support** — migrate to the AVM-based Terraform accelerator. Generated landing-zone IaC should start from the AVM accelerator, not blog posts referencing the old repos (models trained before 2026 still recommend them).
- **Azure Policy is the enforcement layer** — deny public network access, require managed identity, auto-deploy diagnostics — assigned at management-group level. Prefer `Deny`/`DeployIfNotExists` over wiki pages.

## Observability defaults

One Log Analytics workspace per environment (not per app); diagnostics routed by policy, not clicks. App Insights workspace-based with connection strings (instrumentation keys are legacy); **sampling on and daily caps set before the first bill**; Basic-vs-Analytics table plans per table. Alert early on platform throttling signals: Cosmos 429 rate, SNAT port usage, App Service plan CPU/memory (the plan, not the app, saturates).

## Azure OpenAI / Foundry (fast-moving — verified mid-2026)

- The brand is **Azure AI Foundry, progressively becoming "Microsoft Foundry"** — expect docs and SDKs under the new name. Azure OpenAI Service still exists, but new work targets a Foundry resource (`Microsoft.CognitiveServices/accounts`, kind `AIServices`): OpenAI models plus other providers behind one endpoint/credential. Upgrading an existing Azure OpenAI resource is opt-in and preserves endpoint/keys/fine-tunes.
- Review-surviving pattern: public network access disabled + private endpoint; managed identity + `Cognitive Services OpenAI User`; PTU reservations for steady volume, pay-as-you-go for spiky (the consumption-vs-dedicated math, applied to tokens).
- Capacity is quota-per-region-per-model; production needs a stated fallback (second region/model version) because model retirements run on Microsoft's clock.

## How an expert thinks through this

*".NET SaaS team moving a monolith + 4 background jobs; needs Postgres-equivalent, on-prem AD users, 'no secrets in config'."*

Identity first: Entra Connect already syncs on-prem; every compute unit gets a managed identity — the security requirement is satisfied structurally, not by vault-sprinkling. Compute: low-touch web app → App Service Premium v3 zone-redundant (reject AKS: no K8s skills or requirements — a second job). Jobs: bursty, queue-driven → Functions Flex Consumption — chosen specifically because reaching the private database requires VNet integration, which classic Consumption lacks. The 40-minute nightly job becomes an ACA Job, not a bent function timeout. Data: they *said* Postgres but they're a SQL shop with licenses — challenge it; if genuinely Postgres, Flexible Server zone-redundant with Entra auth, private endpoint + DNS zone linked to *both* consumer VNets (write the linkage down; it's what will break). Secrets audit ends at zero. IaC: Bicep + management-group policies denying public network access.

## Bill-shock and outage priors — the Azure ordering

Bill jumped? The reflex (trained on AWS) is to hunt new compute first — on Azure check, in order: (1) **Log Analytics/App Insights ingestion** (new AKS/ACA diagnostics, sampling off — reaches top-3 reliably); (2) **idle premium SKUs** bought for one feature (App Gateway v2, APIM Premium, Service Bus Premium); (3) non-prod running 24/7 without Dev/Test pricing; (4) egress/NAT per-GB; (5) unattached disks and orphaned public IPs; (6) expired reservations (rate change, no usage change). Only then per-service subtleties (Cosmos RU over-provisioning).

Fell over? (1) **DNS** — private endpoint resolving public (the Azure-specific #1); (2) RBAC — role missing, propagation lag, wrong identity picked by `DefaultAzureCredential` (set `AZURE_CLIENT_ID` with multiple user-assigned identities); (3) SNAT exhaustion; (4) throttling (Cosmos 429s, ARM API limits during parallel deploys); (5) subscription/regional quota. Genuine platform outage sits below all of these.

## Failure modes & pitfalls

- Private endpoint created, DNS zone not linked to the caller's VNet → resolves public → "it's down." Endpoint + zone + link = one unit (AVM modules do this).
- `DefaultAzureCredential` works locally, fails deployed (or vice versa): different principal per environment — check `AZURE_CLIENT_ID` and role assignments before touching code.
- Classic Consumption chosen for a function needing VNet access — it can't, ever. Flex or Premium.
- Cosmos partition key `/type` or a date → hot partition, 429s at low total RU; changing it later = container migration.
- Azure SQL DTU tiers picked by habit — vCore + Entra auth + Hybrid Benefit is the 2026 production answer.
- Old storage accounts with `allowBlobPublicAccess` enabled and long-lived SAS in URLs — policy-deny; prefer short-lived user-delegation SAS.
- Log Analytics ingestion as a top-3 line because AKS/ACA ships every container stdout at per-GB rates — Basic Logs tier, retention, sampling.
- Subscription-as-junk-drawer: split prod/non-prod subscriptions under a management group early; nearly free now, un-retrofittable later.
- Renamed-service drift in generated code: Single Server, instrumentation keys, "Azure Active Directory" endpoints — verify against current Learn docs.
- Zone redundancy assumed rather than configured: LRS vs ZRS, App Service zone redundancy needs the right tier and ≥2 instances — audit each stateful resource's actual setting.
- Bicep role-assignment name collisions on redeploy — seed `guid()` with *all* distinguishing inputs (scope, principal, role).

## Verification / self-check

1. Zero secrets in app settings/IaC — every data-plane access is an identity + a role you can name.
2. Every private endpoint's DNS zone linked to every consumer VNet — trace one resolution end-to-end.
3. Service names checked against current Learn docs if recalled from memory (Azure renames things).
4. Consumption-vs-dedicated arithmetic done; tier-forcing features named.
5. Subscription/management-group tree and policy assignments stated, not implied.
Stopping rule: when identity, DNS, and the resource tree are explicit and the design deploys from clean IaC into an empty subscription, stop architecting — remaining doubts are answered by deploying.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)

- Probed 14 claims: 11 baseline (cut/compressed), 3 partial (sharpened), 0 delta.
- Opus cold already nails: Flex Consumption default, AKS Automatic, PE-DNS diagnosis, SNAT exhaustion, Cosmos consistency economics, Front Door origin lock, Hybrid Benefit — those sections were compressed to anchors.
- Kept/sharpened the real gaps: AVM landing-zone status (ALZ-Bicep *archived*, Bicep AVM LZ GA early 2026, caf-enterprise-scale in *extended support* — Opus knows only "superseded"), the progressive "Microsoft Foundry" rebrand, and Azure bill-jump ordering (Opus checks new resources/Cosmos first; in practice Log Analytics ingestion, idle premium SKUs, and missing Dev/Test pricing lead).
