---
name: infrastructure-as-code
description: Load when writing or reviewing Terraform/OpenTofu (or Pulumi/CDK), designing module or state architecture, debugging state drift or plan surprises, choosing between IaC tools, handling secrets in IaC, or setting up environments and policy checks for infrastructure changes.
---

# Infrastructure as Code

## Core mental model

- **State is the crown jewel.** Terraform/OpenTofu is a three-way diff engine: configuration (desired) vs state (last-known) vs reality (refreshed). Every mystery — orphaned resources, "already exists" errors, plans that want to recreate everything — is one of those three legs disagreeing. Lose the state file and the tool forgets it owns anything; corrupt it and the tool will confidently do the wrong thing. Remote state with locking (S3 with native locking, or equivalent) is non-negotiable from day one; state file access is admin access (state contains resource attributes, historically including secrets — treat the bucket like a credentials store).
- **Never hand-edit what IaC manages.** A console change creates drift: the next plan either silently reverts the "fix" (re-breaking prod at an arbitrary future time, in someone else's unrelated apply) or errors. The discipline is cultural, not technical: emergency console changes are allowed, but the incident isn't closed until the change is codified or reverted. Detect drift by running plans on a schedule, not only on PRs.
- **The plan is the code review.** HCL diffs lie: a one-line change can mean in-place update, or destroy-and-recreate of a database. Review the *plan output*, not the diff. The three symbols that matter: `~` update in-place (usually fine), `-/+` **replace** (destroy then create — the dangerous one; the plan names the attribute that forced it), `-` destroy. Any replace or destroy of a stateful resource requires explicit human intent.
- **Blast radius is a design input.** One giant state = one lock queue, one plan that takes 20 minutes, one bad apply that can touch everything, one state corruption that loses everything. Split state along ownership and change-frequency lines; wire the pieces with remote-state reads or data sources.
- **Ordering comes from the graph, not the file.** Resources apply in dependency-graph order derived from references. If you need `depends_on`, first ask why no attribute reference exists — hidden dependencies (IAM propagation, eventual consistency) are legitimate; using `depends_on` to paper over a missing reference is a smell.

## Tool landscape (as of 2026)

- **Terraform vs OpenTofu**: genuinely diverged tools now, not a re-badge. Terraform (BSL license — restricts building competing products; ~1.14.x as of mid-2026) shipped ephemeral resources (1.10) and write-only arguments (1.11). OpenTofu (MPL-2.0, CNCF, ~1.11–1.12) has state encryption (1.7 — note: enabling it makes state unreadable by Terraform, a one-way door), early variable evaluation in backends/modules (1.8), provider `for_each` (1.9), OCI registry support (1.10), and its own ephemeral/write-only support (1.11). State remained binary-compatible around the 1.5.x fork point, but HCL-level divergence is growing — don't assume a config written for one runs on the other. Choosing: BSL exposure or wanting state encryption → OpenTofu; deep HCP/Sentinel investment → Terraform; either is production-grade, so the real answer is usually "whichever your org already standardized on — don't run both."
- **Pulumi/CDK** (programmatic): choose when the *logic* is genuinely programmatic — loops with complex conditionals, sharing types with app code, teams that refuse HCL. Costs: plan/preview is less universally reviewable, testing culture required (you now have real code paths), CDK adds CloudFormation's slowness and rollback model underneath. Don't pick Pulumi because HCL `for_each` felt awkward once; do pick it when you're generating infrastructure from application metadata.
- HCL remains the ecosystem default; provider coverage, hiring, and examples all favor it.

## State and environment architecture — the reasoning chain

Questions, in order:

1. **Who changes this and how often?** Network/VPC (platform team, monthly) doesn't belong in the same state as app services (app teams, daily). Split by owner × cadence. A rough shape that scales: `network` / `data` (stateful, guarded) / `platform` (cluster, shared services) / one state per app team or service group.
2. **What's the blast radius of a bad apply here?** Databases and stateful stores get their own state with `prevent_destroy` and the smallest possible surrounding config.
3. **How do environments differ?** Prefer **directory-per-environment** (`envs/prod/`, `envs/staging/`), each a thin root module calling shared versioned modules with different variables and its own backend. Workspaces share one backend and one code version — fine for ephemeral short-lived copies (PR previews), dangerous as prod/staging separation because `terraform workspace show` is the only thing between you and applying prod with staging vars, and you can't pin prod to an older module while staging tests a new one. Terragrunt or stacks tooling helps when directory count explodes; adopt when the pain is real, not preemptively.
4. **How do split states communicate?** `terraform_remote_state` data sources or plain data-source lookups by name/tag. Prefer the latter across team boundaries — remote-state reads couple consumers to the producer's *state layout* and require access to the producer's entire (secret-bearing) state; a data source by tag couples only to reality.

## Module design

- **Thin modules, versioned contracts.** A good module wraps a coherent unit (a service's runtime footprint; an opinionated VPC) with a small variable surface encoding *your org's* opinions. A god-module with 80 variables that "does everything" is a config file wearing module syntax — every consumer change risks every consumer.
- Version modules with git tags or a registry and pin consumers (`?ref=v2.3.0`). An unpinned module source means someone else's merge changes your prod plan.
- Composition over nesting: roots compose modules; modules nesting modules more than ~2 deep makes plans unreadable and variables tunnel through layers.
- Don't wrap a single resource in a module unless it adds real policy (naming, tags, mandatory encryption). One-resource pass-through modules are indirection tax.
- Outputs are the contract too: removing an output is a breaking change for unknown remote-state readers — version accordingly.

## Import vs recreate

When infrastructure exists but state doesn't know it (console-created legacy, state surgery, adoption): **import if it's stateful or referenced** (databases, DNS zones, IAM consumed by others), **recreate if it's cheap and fungible** (a security group nobody references, a lambda redeployable in seconds). Use `import` blocks (declarative, plannable, reviewable in the PR) over one-shot `terraform import` CLI where available. After import, iterate until `plan` is *empty* — a post-import plan showing changes means your config doesn't match reality yet, and applying would mutate a live resource you just adopted. For state surgery on refactors, `moved` blocks > `state mv` (reviewable, replayable); `removed` blocks / `state rm` to disown without destroying.

## Secrets in IaC (as of 2026)

Layered defense, best first:
1. **Keep secrets out of the workflow entirely**: IAM roles/OIDC instead of keys, cloud-managed passwords (e.g. RDS `manage_master_user_password`), certs from ACM — resources that never expose a secret to Terraform can't leak it.
2. **Ephemeral values / write-only arguments** (Terraform 1.10–1.11, OpenTofu 1.11): read a secret at apply time and hand it to a resource without it ever entering state or plan artifacts. This is the current-best pattern for "Terraform must pass a secret."
3. **Reference-don't-inline**: resource fields that accept a secret-manager ARN/path keep the secret in Vault/ASM/GSM; Terraform only handles the pointer.
4. `sensitive = true` is **display masking only** — the value still sits in state in plaintext. Never mistake it for protection.
5. Accept-and-contain: when a secret unavoidably lands in state (many older providers), the mitigation is state-store access control + encryption (OpenTofu state encryption, or KMS-encrypted backend) + audit.
Never: secrets in `.tfvars` committed to git, in HCL literals, or in plan files uploaded as CI artifacts (plans contain values too — guard plan artifacts like state).

## Plan-review discipline & policy-as-code

- CI posts the plan on the PR; humans approve the *plan*; apply runs the *saved plan artifact* (`terraform apply tfplan`), not a fresh plan — otherwise the thing applied isn't the thing reviewed.
- Automate destructive-change detection: parse `terraform show -json tfplan` and require elevated approval when any resource has `"actions": ["delete"]` or `["delete","create"]` — humans skim 400-line plans and miss the one replace.
- `lifecycle { prevent_destroy = true }` on every database, state bucket, DNS zone, KMS key. It fails the plan rather than allowing a quiet replace. (OpenTofu 1.12 allows dynamic/conditional `prevent_destroy`; stock Terraform requires a literal.)
- Policy-as-code (OPA/conftest on the JSON plan, or Sentinel on HCP): encode the rules you'd otherwise repeat in review — tags required, no public buckets, instance-type allowlists, "no delete on resources tagged critical". Start warn-only, promote to deny after tuning; a policy that blocks legitimate work gets bypassed culturally and then protects nothing.

## Escape hatches — the taxonomy, in descending order of preference

1. Native resource (always look again — providers grow fast).
2. Community/partner provider for the API.
3. Generic API resource (e.g. a REST/`http`-based provider) — declarative even if crude.
4. `null_resource`/`terraform_data` + `local-exec` — **last resort**: invisible to plan, no drift detection, runs where the plan runs (CI runner deps), and triggers/re-run semantics are a bug farm. Every `local-exec` needs a comment saying what native gap it fills and an issue link tracking its removal. Two or more `local-exec`s orchestrating each other means this piece doesn't belong in Terraform — move it to a real workflow (CI job, operator, script) that Terraform merely triggers or that runs after apply.

## How an expert thinks through it: "plan wants to destroy and recreate the prod RDS instance"

Never apply; read *why*. The plan marks `-/+` and names the culprit: `identifier` forces replacement — someone renamed the instance to match a new convention. Options: (a) apply — destroys prod data; obviously no, but say why in the PR so the author learns the `-/+` semantics. (b) Revert the rename — safe, but the convention change had a reason. (c) Check whether this is rename-in-place-able: for RDS, `identifier` change is actually an in-place rename via the API, but the provider models it as replacement? Verify in provider docs/changelog rather than assume — if the provider genuinely forces replacement, (d) do it as a managed migration: snapshot, create new alongside, cut over, then remove the old — as explicit separate changes, not one hidden inside a plan. Also fix the process gap: why did no `prevent_destroy` exist on this instance, and why did CI not flag a delete action on a `data`-tier resource? Add both. Considered and rejected: `terraform state rm` + re-import under the new name — actually the *right* trick when only the resource *address* changed (use `moved` blocks for that), but here the *cloud-side identifier* changed, so state surgery can't help. The lesson encoded: replaces are diagnosed from the plan's "forces replacement" attribute, and stateful resources get both a lifecycle guard and a policy check, because humans will eventually skim.

## Verification / self-check

- After any apply: run `plan` again — it must be empty. A non-empty second plan means perpetual drift (a provider default fighting your config, a value normalized by the API); fix it now or every future plan carries noise that trains reviewers to ignore plans.
- After import or refactor: empty plan is the definition of done.
- Grep state for secrets before declaring a workflow secure: `terraform show -json | grep -i` the credential names you fear. If present, that's your real security boundary — act accordingly.
- For a module: can you write its README contract (inputs, outputs, invariants) in 15 lines? If not, it's doing too much.
- Destructive-change drill: does your pipeline actually stop a plan containing a delete on a guarded resource? Test with a scratch resource, not by faith.
- Stopping rule: infrastructure code is done when plans are empty, state contains no unguarded secrets, every stateful resource has `prevent_destroy`, and a new team member can find which state owns a resource in under a minute. Refactoring module trees beyond that is aesthetics.
