---
name: cloud-architecture-azure
description: Load when designing, reviewing, or debugging architectures on Microsoft Azure — choosing compute (Functions/Container Apps/AKS/App Service), Entra ID and managed identities, storage/database selection (Blob, Cosmos DB, Azure SQL, PostgreSQL), VNets and private endpoints, Bicep vs Terraform, landing zones, or Azure OpenAI/Foundry integration. Also load for AWS-to-Azure translation questions.
---

# Azure Architecture Judgment

## Core mental model

- **Identity is the center of gravity.** On AWS you reason from the VPC outward; on Azure you reason from Entra ID outward. Every service authenticates via Entra; every workload should hold a managed identity; secrets are a design smell, not a default. If an Azure design contains a connection string with a password, ask why before anything else.
- **The resource hierarchy is the control plane.** Management groups → subscriptions → resource groups → resources, with RBAC and Azure Policy inherited downward. Subscriptions are the real isolation and billing boundary (analogous to AWS accounts, but cheap and expected to be numerous). Architecture on Azure includes deciding this tree; on AWS teams often skip the equivalent and get away with it — on Azure, policy and landing zones assume the tree exists.
- **Azure renames things; verify names before writing them down.** AD → Entra ID; Azure AI Studio → Azure AI Foundry → (as of 2026, progressively) *Microsoft Foundry*; classic Functions Consumption → Flex Consumption. Treat any Azure service name from memory as stale until checked — recommendations citing dead names destroy credibility and break IaC.
- **PaaS-first is the culturally correct default.** Azure's center of mass is App Service/Container Apps/managed databases with deep Entra integration, not raw VMs. Fighting this (building everything on VMs + VM-based networking) means fighting the platform's IAM, patching, and diagnostics story simultaneously.
- **The bill's silent killers are the same as AWS** (egress, logging, idle premium SKUs) **plus one Azure special:** per-SKU tiering. Many services price by tier (Basic/Standard/Premium) where a checkbox feature forces the next tier for the whole resource. Know which feature is forcing your tier.

## Compute selection — the decision chain

Ask in order:

1. **Is it event-glue or an application?** Single-purpose event handlers (queue → transform → store, timers, webhooks) → **Azure Functions**. Anything with routes, sessions, or a deployable "service" identity → containers or App Service.
2. **Functions: which plan?** As of 2026 the **Flex Consumption plan is the recommended serverless default** (classic Consumption is legacy for new apps): scale-to-zero billing plus VNet integration, per-instance memory choice, optional always-ready instances for cold-start-sensitive paths. Choose Dedicated/App Service plan hosting only when Functions must ride existing reserved App Service capacity; choose Premium when you need features Flex lacks in your region — check current regional availability rather than assuming.
3. **Container or code?** Team ships containers, needs sidecars/Dapr, polyglot microservices, KEDA event-scaling → **Azure Container Apps (ACA)**. ACA is managed Kubernetes-underneath without exposing Kubernetes — the right default for containerized services.
4. **Do you need the actual Kubernetes API?** Custom operators/CRDs, GPU scheduling with fine control, service mesh requirements, existing K8s estate → **AKS**. As of 2026, prefer **AKS Automatic** for new clusters unless you need the knobs it removes (it manages node pools, upgrades, and security defaults). Choosing AKS is hiring yourself as a cluster operator; make that explicit in the review.
5. **Is it a plain web app the team wants to babysit least?** .NET/Java/Node web app, no container pipeline, want deployment slots and easy custom domains → **App Service** still earns its keep. It's boring in the good way.
6. **Consumption vs dedicated reasoning** (applies to Functions and ACA alike): consumption billing wins below roughly 30–40% sustained utilization; above that, dedicated capacity (App Service plan, ACA dedicated/workload profiles, AKS with reservations) is cheaper. Compute the duty cycle; don't vibe it.

| Signal | Points to | Because |
|---|---|---|
| Event glue: queue/timer/webhook handlers | Functions (Flex Consumption) | Scale-to-zero + bindings do the plumbing |
| Containerized microservices, Dapr/KEDA, scale-to-zero | Container Apps | Managed K8s underneath, none of the K8s ops |
| Needs the Kubernetes API itself | AKS (Automatic for new builds) | Nothing else exposes CRDs/operators/mesh |
| Plain web app, .NET/Java team, slots wanted | App Service | Lowest-touch PaaS with mature deploy story |
| Long/heavy batch jobs | ACA Jobs / Azure Batch | Function timeouts are the wrong fight |
| GPU with driver control, licensed software | VMs / VM Scale Sets | The managed layers subtract needed control |

## Identity: the no-secrets architecture

- **Every workload gets a managed identity** (system-assigned for 1:1 lifecycle, user-assigned when shared across resources or needed pre-deployment). Code uses `DefaultAzureCredential` (Azure SDKs), which resolves managed identity in Azure and developer credentials locally — the same code, zero secrets, both places.
- **Grant data-plane RBAC roles to the identity** on the target resource: `Storage Blob Data Contributor` on the storage account, `Key Vault Secrets User` on the vault, Azure SQL Entra authentication with the identity as a database user. The design goal: `az resource list` of your app's config contains **no keys, no passwords, no SAS tokens**.
- Key Vault is for the secrets you can't eliminate (third-party API keys) — accessed via managed identity, referenced from App Service/Functions with Key Vault references rather than copied into app settings.
- Disable key-based auth where possible (`allowSharedKeyAccess: false` on storage accounts, `disableLocalAuth` on Service Bus/Cosmos) — enforce via Azure Policy so the posture is structural, not aspirational.
- For CI/CD, use **workload identity federation** (GitHub Actions OIDC → Entra app) — no service-principal client secrets with 2-year expiries silently breaking pipelines.

## Storage and databases

- **Blob tiers: Hot, Cool, Cold, Archive.** Cool (30-day minimum) for backups and infrequent reads; Cold (90-day minimum) sits between Cool and Archive; Archive is offline — rehydration takes hours, so anything with a "restore quickly" requirement can't live there. Same trap as S3: early-tiered churning data costs more than Hot via early-deletion and read penalties — model object age first. Lifecycle management rules are set per storage account; use them.
- **Cosmos DB — buy it for the guarantees, not the fashion.** Justified by: global multi-region writes, guaranteed single-digit-ms latency at scale, or truly elastic per-request (serverless/autoscale RU) needs. Its five consistency levels are the applied part: **Session** (default) is right for user-facing apps — a user reads their own writes, cross-user staleness tolerable. **Strong** costs you: it constrains multi-region topology and doubles read RU cost vs eventual in practice — take it only for invariants (balances, inventory). **Bounded staleness** is the honest compromise for global reads with a defined lag. If someone picks Strong "to be safe," push back with the RU bill and region constraints. RU under-provisioning shows up as 429s — watch normalized RU consumption, not averages.
- **Azure SQL vs PostgreSQL Flexible Server:** not a technology beauty contest — it's team gravity. .NET shop, existing SQL Server skills/licenses (Azure Hybrid Benefit), need managed HA with minimal thought → Azure SQL Database. Open-source alignment, extensions (PostGIS, pgvector), exit-portability → **PostgreSQL Flexible Server** (the Single Server product is retired; Flexible is the only Postgres option that should appear in a 2026 design). Both do zone-redundant HA; both do Entra auth — require it.
- Queues: Service Bus for enterprise messaging (sessions, ordering, dedup, DLQ, topics); Storage Queues only for cheap-and-basic; Event Grid for reactive pub/sub of discrete events; Event Hubs for streaming/telemetry firehose. Choosing Event Hubs for job queues or Service Bus for telemetry are both category errors.

## Networking

- **VNets are regional; subnets are not AZ-bound** (unlike AWS — zone redundancy is a property of the resource, not the subnet). Delegated subnets are consumed by PaaS injection (ACA, Flexible Server) — plan address space generously; VNets are hard to renumber.
- **Private endpoints are the default posture for PaaS in serious environments:** a NIC in your VNet with a private IP for the storage account/SQL/Key Vault, paired with a Private DNS zone (`privatelink.*`) so the public FQDN resolves privately. The #1 private-endpoint failure is DNS: the endpoint exists but clients still resolve the public IP because the Private DNS zone isn't linked to the client's VNet. Check DNS before checking anything else.
- Service endpoints are the older, cheaper mechanism (traffic stays on backbone but the resource keeps a public IP with firewall rules) — acceptable middle ground; know which one you're using.
- **Hub-spoke** is the canonical enterprise topology: shared services (firewall, DNS, ExpressRoute/VPN gateways) in a hub VNet, workloads in peered spokes, spoke-to-spoke via the hub firewall. Don't build it for a 3-service startup; do build it the moment there's a connectivity-to-on-prem or central-egress-inspection requirement, because retrofitting is painful. NSGs ≈ security groups (stateful); Azure Firewall/NVA handles centralized egress control.
- Azure does not charge for cross-AZ traffic within a region (unlike AWS's $0.01/GB) — but internet egress (~$0.087/GB tier-1) and NAT Gateway per-GB processing still apply; the egress-trap reasoning from AWS carries over.
- **Load-balancing tier selection:** Azure Load Balancer for L4 within a region; **Application Gateway** (with WAF) for regional L7/path routing into VNets; **Azure Front Door** for global HTTP entry — CDN, anycast, WAF, and origin failover in one. The common miswire is Front Door pointed at public origins with no lock: restrict origins to accept only your Front Door instance (private link origins or the `X-Azure-FDID` check), or the WAF is decorative. Don't stack Front Door + App Gateway without a stated reason (usually: global entry + per-VNet TLS/WAF policy); each hop is money and latency.

## AWS-to-Azure translation (where the mapping is honest, and where it lies)

| AWS | Azure | The catch |
|---|---|---|
| Account | Subscription | Subscriptions are cheaper/more numerous; management groups do the org layer |
| IAM roles + policies | Entra ID + RBAC role assignments | Azure RBAC is coarser; data-plane roles are separate grants per resource |
| Lambda | Functions (Flex Consumption) | Bindings replace much SDK glue; plan choice changes capabilities, not just price |
| ECS/Fargate | Container Apps | ACA bakes in KEDA/Dapr; less raw control than ECS task definitions |
| EKS | AKS | AKS Automatic ≈ managed-er than EKS defaults |
| SQS / SNS / EventBridge | Storage Queues or Service Bus / Service Bus topics / Event Grid | Service Bus covers both queue+topic roles; Event Grid is the reactive bus |
| Kinesis | Event Hubs | Kafka-protocol compatible endpoint is often the real reason to pick it |
| DynamoDB | Cosmos DB | Cosmos bills RUs, offers 5 consistency levels — richer and easier to overspend |
| CloudFormation | ARM/Bicep | Bicep has no state file; drift semantics differ from Terraform entirely |
| CloudWatch | Azure Monitor + Log Analytics + App Insights | Three brands, one ingestion bill |
| VPC endpoints / PrivateLink | Private endpoints + Private DNS zones | The DNS half is mandatory on Azure and the usual failure point |

## Cost model differences from AWS

- **Cross-AZ traffic is free on Azure** — architectures penalized on AWS (zone-spanning chatty services, zone-redundant Kafka) are cheaper here; don't import AWS zone-locality contortions without checking.
- **Licensing is the big Azure-only lever:** Azure Hybrid Benefit (bring Windows Server/SQL licenses) can dwarf reservations for Microsoft-stack estates; always ask "do they own licenses?" before optimizing anything else.
- **Tier cliffs instead of smooth per-use:** many services (App Service plans, App Gateway, Service Bus Premium, API Management) price in chunky SKU tiers where one feature (VNet integration, zone redundancy, sessions) forces the next tier for the whole resource. On AWS the analogous costs are more often per-use. Budget by naming the tier-forcing feature.
- **Commitments:** Azure reservations + the Azure savings plan for compute mirror AWS RIs/Savings Plans; same layering logic (reserve the floor, savings plan the flexible middle, on-demand the rest). Dev/Test subscription pricing is an extra discount class AWS lacks — use it for non-prod.
- **Egress and NAT per-GB are the same silent killers as AWS**; Log Analytics ingestion plays the CloudWatch role and reaches top-5 line-item status just as reliably.

## IaC and landing zones

- **Bicep vs Terraform:** Azure-only estate → Bicep (day-zero resource coverage via ARM, no state file to manage, first-class in Microsoft's own accelerators). Multi-cloud or heavy third-party providers → Terraform/OpenTofu, using **AzAPI** provider to cover gaps where `azurerm` lags new services. Don't mix both for the same resources; pick one source of truth per scope.
- **Landing zones, as of 2026:** Microsoft has consolidated on **Azure Verified Modules (AVM)** — the Bicep AVM Platform Landing Zone went GA in early 2026, the legacy ALZ-Bicep repo is archived, and the Terraform `caf-enterprise-scale` module is in extended support (migrate to the AVM-based Terraform accelerator). If you're generating landing-zone IaC, start from the AVM accelerator, not from blog posts referencing the old repos.
- **Azure Policy is the enforcement layer**: deny public network access, require managed identity, restrict regions/SKUs, auto-deploy diagnostics — assigned at management-group level so new subscriptions inherit the guardrails. Prefer `Deny`/`DeployIfNotExists` policies over wiki pages.

## Observability defaults

- One Log Analytics workspace per environment (not per app) so cross-resource queries work; route diagnostics there via policy (`DeployIfNotExists`), not per-resource clicks.
- Application Insights (workspace-based; connection strings, not legacy instrumentation keys) for app telemetry — but turn **sampling on** and cap daily ingestion before the first bill, not after. Set table-plan (Basic vs Analytics) per table: chatty container logs don't need full-query pricing.
- Alert on the platform's own throttling signals early: Cosmos 429 rate, SNAT port usage, App Service CPU/memory percent per plan (the plan, not the app, is the unit that saturates).

## Azure OpenAI / Foundry (fast-moving — verified mid-2026)

- The platform brand is **Azure AI Foundry, progressively rebranded "Microsoft Foundry"**; Azure OpenAI Service still exists and still receives new models, but new work should target a Foundry resource (`Microsoft.CognitiveServices/accounts`, kind `AIServices`), which fronts OpenAI models *plus* other providers (Llama, Mistral, etc.) behind one endpoint and credential. Upgrading an existing Azure OpenAI resource to Foundry is opt-in and preserves endpoint/keys/fine-tunes.
- Integration pattern that survives review: Foundry resource with **public network access disabled + private endpoint**, callers authenticate with **managed identity + `Cognitive Services OpenAI User` RBAC role** (no API keys in app settings), throughput via pay-as-you-go for spiky workloads or **provisioned throughput (PTU)** reservations for steady high volume — the consumption-vs-dedicated reasoning again, applied to tokens.
- Capacity is quota-per-region-per-model; production designs need a stated fallback (second region, second model version) because model-version retirements are announced on Microsoft's clock, not yours.

## How an expert thinks through this

*"SaaS team (mostly .NET) moving a monolith + 4 background jobs to Azure; needs Postgres-equivalent, on-prem AD users, and 'no secrets in config' from the security team."*

Identity first, because everything hangs off it: Entra ID already synced from on-prem (verify Entra Connect exists). Every compute unit gets a managed identity; the security requirement is therefore satisfiable structurally, not by vault-sprinkling.

Compute: monolith is a web app the team wants low-touch → App Service (deployment slots for their release process) or ACA if they're containerizing anyway. They're not — App Service, Premium v3 with zone redundancy. Reject AKS instantly: no K8s skills, no K8s requirements — it would be a second job. Background jobs: queue-driven, minutes-long, bursty → Functions on Flex Consumption (VNet integration is required to reach the private database — Flex has it; classic Consumption wouldn't, which is exactly why classic is the wrong default). One job runs 40 minutes nightly → that's beyond a sane function; make it an ACA Job or WebJob — don't bend the function timeout.

Data: they said Postgres-equivalent, but they're a SQL Server shop with licenses → challenge the requirement. If it's genuinely Postgres (existing schema, PostGIS), PostgreSQL Flexible Server zone-redundant with Entra auth. Connection path: private endpoint + Private DNS zone linked to *both* the App Service integration VNet and the Functions VNet — write the DNS linkage down; it's the thing that will break.

Secrets audit: DB = Entra auth via managed identity; Storage = RBAC, shared-key disabled; the one third-party API key goes in Key Vault behind a Key Vault reference. Config now contains zero secrets — requirement met by architecture. IaC: Azure-only shop → Bicep, with policies denying public network access on data services assigned at the subscription's management group.

## Bill-shock and outage priors — what it usually is on Azure

Bill jumped? Check in order: (1) Log Analytics/Application Insights ingestion (new AKS/ACA diagnostics, verbose sampling off); (2) idle premium SKUs — App Gateway v2 instances, API Management Premium, Service Bus Premium bought for one feature; (3) forgotten non-prod running 24/7 without Dev/Test pricing; (4) egress/NAT per-GB from a new integration; (5) unattached managed disks and orphaned public IPs. Only then per-service subtleties (Cosmos RU over-provisioning, storage transaction fees).

Fell over? Check in order: (1) DNS — private endpoint resolving to public IP or a missing zone link (the Azure-specific #1); (2) RBAC/identity — role not assigned, propagation lag, wrong identity picked by `DefaultAzureCredential`; (3) SNAT port exhaustion on outbound-heavy App Service/Functions without NAT Gateway — connection failures under load that look like the remote service's fault; (4) throttling — Cosmos 429s, ARM API limits during parallel deploys; (5) subscription/regional quota hits during scale-out. Genuine platform outage sits below all of these.

## Failure modes & pitfalls

- **Private endpoint created, Private DNS zone not linked** to the caller's VNet → client resolves public IP → firewall denies → "it's down" tickets. Always: endpoint + zone + zone-VNet-link as one unit (AVM modules do this for you).
- **`DefaultAzureCredential` works locally, fails deployed** (or vice versa): locally it used the developer's `az login`; in Azure the managed identity lacks the RBAC role, or with multiple user-assigned identities the client ID wasn't specified. Check `AZURE_CLIENT_ID` and role assignments before touching code.
- **RBAC role assignment lag:** data-plane role assignments can take a few minutes to propagate; deployment scripts that grant-then-immediately-use fail intermittently. Add retry, don't re-architect.
- **Classic Consumption chosen for a function needing VNet access** — it can't. Flex Consumption or Premium. (As of 2026 classic Consumption is legacy; new apps shouldn't use it at all.)
- **Cosmos DB partition key chosen as `/type` or a date** → hot partition, 429 throttling at low overall RU. Partition key must distribute writes *and* match dominant query filter; changing it later means container migration.
- **Azure SQL DTU-model tiers picked by habit** — vCore model with Entra auth and Hybrid Benefit is almost always the right call for production as of 2026; DTU obscures what you're buying.
- **Storage account with `allowBlobPublicAccess` default-enabled on old accounts** and SAS tokens in URLs living in logs. Policy-deny public access; prefer user-delegation SAS (Entra-signed, short-lived) when SAS is unavoidable.
- **Log Analytics ingestion becoming a top-3 cost line** because AKS/ACA diagnostic settings ship every container stdout at per-GB ingestion rates. Set table-level retention, use Basic Logs tier for chatty low-value tables, sample.
- **Subscription-as-junk-drawer:** everything in one subscription, RBAC at resource level, policy unenforceable. Split prod/non-prod subscriptions early under a management group; it's nearly free and un-retrofittable later without pain.
- **Renamed-service drift in generated code:** referencing retired PostgreSQL Single Server, classic Application Insights instrumentation keys (use connection strings), or "Azure Active Directory" endpoints in new designs. Verify names against current Microsoft Learn docs whenever the recalled name is more than a year old.
- **SNAT exhaustion misdiagnosed for weeks:** outbound calls from App Service/Functions intermittently failing under load; teams blame the remote API. Fix structurally: VNet integration + NAT Gateway (dedicated SNAT ports), connection reuse in the HTTP client.
- **Front Door origin left publicly reachable** — attackers bypass the WAF by hitting the origin directly. Lock origins to Front Door (private link or FDID header validation at the origin).
- **Zone redundancy assumed rather than configured:** many Azure services are zonal-by-default or need a ZRS/zone-redundant SKU explicitly (storage LRS vs ZRS, App Service zone redundancy needs the right plan tier and instance count ≥ 2). Audit each stateful resource's actual redundancy setting.
- **Bicep `existing` + role assignment GUID collisions** on re-deploy across scopes — name role assignments with `guid()` seeded by *all* distinguishing inputs (scope, principal, role) or redeploys fail with conflicts.

## Worked micro-example — no-secrets blob access (Bicep + .NET)

```bicep
resource storage 'Microsoft.Storage/storageAccounts@2024-01-01' = {
  name: 'stappdata${uniqueString(resourceGroup().id)}'
  location: location
  sku: { name: 'Standard_ZRS' }
  kind: 'StorageV2'
  properties: { allowSharedKeyAccess: false, publicNetworkAccess: 'Disabled' }
}
// Grant the app's managed identity data-plane access
resource blobRole 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  scope: storage
  name: guid(storage.id, app.id, 'blob-contributor')
  properties: {
    principalId: app.identity.principalId
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions',
      'ba92f5b4-2d11-453d-a403-e96b0029c9fe') // Storage Blob Data Contributor
    principalType: 'ServicePrincipal'
  }
}
```

```csharp
// Same code locally (az login) and in Azure (managed identity) — no connection string.
var client = new BlobServiceClient(
    new Uri("https://stappdata.blob.core.windows.net"),
    new DefaultAzureCredential());
```

**Cosmos RU envelope (the arithmetic that prevents 429 surprises):** a 1KB point read ≈ 1 RU; a 1KB write ≈ 5–6 RU (more with indexed properties); queries cost by data scanned within the partition. So 2,000 reads/s + 300 writes/s of 1KB items ≈ 2,000 + ~1,700 ≈ 3,700 RU/s *average* — provision autoscale with a max comfortably above the observed peak-to-average ratio, and remember cross-partition queries multiply the read cost by partitions touched. If the RU math lands 10× above budget, the fix is the partition key or the query shape, not a bigger throughput dial.

## Verification / self-check

1. Zero secrets in app settings/IaC — every data-plane access is a managed identity + RBAC role you can name.
2. Every private endpoint has its DNS zone linked to every consumer VNet — trace one resolution end-to-end.
3. Service names checked against current Microsoft Learn docs this session if anything was recalled from memory (Azure renames things).
4. Consumption-vs-dedicated arithmetic done for each compute choice; tier-forcing features identified.
5. The subscription/management-group tree and policy assignments are stated, not implied.
Stopping rule: when identity, DNS, and the resource tree are explicit and the design deploys from clean IaC into an empty subscription, stop architecting — remaining doubts are answered by deploying, not by more diagrams.
