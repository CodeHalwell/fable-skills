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

## Repository layout — the shape that survives growth

```
infra/
  modules/                    # shared, versioned via tags (or a separate modules repo)
    service-runtime/          # opinionated: ECS/k8s service + alarms + dashboard
    postgres/                 # opinionated: encrypted, backed-up, prevent_destroy baked in
  envs/
    prod/
      network/                # one state each — split by owner × cadence × blast radius
        backend.tf  main.tf   # main.tf is THIN: module calls + env-specific variables only
      data/
      platform/
      services/
    staging/                  # same structure, different variables — diffable against prod
```

Rules that make this work: roots contain no resource logic (if a root grows raw resources, that's a module trying to be born); `envs/prod` and `envs/staging` should diff cleanly (structural drift between envs is how "worked in staging" dies); every directory maps to exactly one state file, and the mapping is guessable from the path.

## Validation and testing — what's worth the effort

In increasing cost, stop where returns flatten:
1. **`terraform fmt -check` + `terraform validate`** in CI — free, catches syntax and internal consistency.
2. **`tflint`** with provider rulesets — catches invalid instance types, deprecated arguments, unused declarations that `validate` misses.
3. **Policy on the plan** (OPA/conftest/Sentinel) — org rules, the highest-value layer (see below).
4. **`terraform test`** (native, 1.6+) for *modules with logic*: variable validation branches, conditionals, for_each shaping. Run with mocked providers or plan-only assertions in CI; reserve real-apply tests for the few modules where a regression is catastrophic and cheap to exercise in a sandbox account.
5. Full ephemeral-environment integration tests (Terratest-style) — expensive to keep green; justified for a platform team's core modules, waste for leaf configs. Most leaf-config confidence should come from plan review + staging applies, not test suites.

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

## Debugging a surprising plan — the reasoning chain

When a plan shows changes nobody expects, walk the three-way diff deliberately:
1. **Which leg moved?** Config (check `git log -p` on the directory *and* on module refs — an unpinned module or bumped provider changes plans with zero local diff), state (did someone run surgery? check backend versioning/audit), or reality (drift: someone touched the console).
2. **For a `~` update**: is the new value yours (config change) or the API's (normalization — e.g., a policy JSON reordered, case-folded ARN)? Perpetual normalization diffs get fixed by matching the canonical form in config, not by `ignore_changes`.
3. **For `-/+` replace**: the plan prints `# forces replacement` next to the exact attribute. That attribute is the whole investigation — who changed it, and is replacement acceptable for this resource class?
4. **For unexplained `known after apply` cascades**: usually one upstream computed attribute changed (or a data source moved to apply-time), fanning out. Find the root resource; ignore the fan-out.
5. **For "already exists" on create**: reality has it, state doesn't — import or delete the stray, never blind-apply with a rename to dodge the collision (now you have two).
Prior: in mature configs, the most common causes of surprise plans are, in order: provider version bumps, unpinned modules, console drift, and API normalization. Genuine Terraform bugs are last — exhaust the boring causes first.

## Escape hatches — the taxonomy, in descending order of preference

1. Native resource (always look again — providers grow fast).
2. Community/partner provider for the API.
3. Generic API resource (e.g. a REST/`http`-based provider) — declarative even if crude.
4. `null_resource`/`terraform_data` + `local-exec` — **last resort**: invisible to plan, no drift detection, runs where the plan runs (CI runner deps), and triggers/re-run semantics are a bug farm. Every `local-exec` needs a comment saying what native gap it fills and an issue link tracking its removal. Two or more `local-exec`s orchestrating each other means this piece doesn't belong in Terraform — move it to a real workflow (CI job, operator, script) that Terraform merely triggers or that runs after apply.

## Failure modes & pitfalls

- **`count` where `for_each` belongs.** Resources created with `count` are addressed by index; removing the first element of the input list shifts every index, and the plan destroys-and-recreates *everything after it*. Use `for_each` keyed on stable identifiers for any collection that can change membership. The tell in review: a plan destroying resources whose config "didn't change."
- **Removing a resource block removes its `prevent_destroy` with it.** Lifecycle guards live in config: delete the block and the next plan happily destroys the resource — the guard can't protect against its own removal. Backstop with policy-as-code on the JSON plan (deny deletes on protected tags), which survives config deletion.
- **Unpinned providers / uncommitted lockfile.** No `required_providers` version constraint + no committed `.terraform.lock.hcl` = teammates and CI resolve different provider versions and get different plans from identical config. Commit the lockfile; run `terraform providers lock -platform=linux_amd64 -platform=darwin_arm64` so both CI and laptops verify hashes.
- **Applying a fresh plan after approving an old one.** `plan` on PR, human approves, CI later runs `apply` (no saved plan): whatever changed in between — new commits, drift, provider release — is applied sight-unseen. Apply the saved artifact; and treat a plan older than ~a day as stale (re-plan and re-review; the world drifted under it).
- **`sensitive = true` mistaken for encryption.** It masks CLI output only. The value is plaintext in state and in plan files. Protection = state-store access control/encryption + ephemeral/write-only patterns, not the flag.
- **Refactoring addresses without `moved` blocks.** Renaming a resource or moving it into a module changes its address; Terraform sees "old one deleted, new one added" → destroy/create. Every refactor PR that touches addresses needs `moved` blocks (or documented `state mv`), and its plan must show *zero* create/destroy.
- **`ignore_changes` as drift concealer.** Legitimate for fields mutated by external systems (autoscaler-managed `desired_count`). Illegitimate as a way to stop plan noise you don't understand — you've now made Terraform blind to that field forever, and the config is a lie. Diagnose the perpetual diff instead (usually API normalization or a provider default).
- **Wrong-workspace apply.** Workspaces share directory and backend; the only guard is remembering `terraform workspace select`. If prod and staging are workspaces, an automation bug or tired human applies across the boundary. Directory-per-env makes the mistake structurally harder; if you keep workspaces, hard-fail in CI when `terraform.workspace` doesn't match the pipeline's target var.
- **Data source vs bootstrap ordering.** A config whose data source looks up a resource that doesn't exist yet fails at *plan* time — circular on first bootstrap. Split bootstrap state, or accept explicit two-phase applies; don't hack it with `depends_on` on data sources (that forces read-at-apply and perpetual "known after apply" diffs).
- **State surgery under a live lock, or `-lock=false` habits.** Running `state` commands while CI holds the lock, or routinely passing `-lock=false` because "the lock was stuck", is how state corruption happens. Stuck locks have a real cause (killed run); use `force-unlock` with the lock ID after confirming the holder is dead, and never disable locking in automation.
- **Secrets in plan artifacts.** Teams lock down state, then upload `tfplan` (which embeds values, including sensitive ones) as a world-readable CI artifact. Plan files inherit state's classification.
- **`-target` as a habit.** `terraform apply -target=...` skips the rest of the graph — dependencies don't update, and repeated targeting leaves state permanently inconsistent with config. It's a break-glass tool for recovering from a wedged state, and every use should end with a full clean plan. A runbook that says "always apply with -target" describes a state-splitting problem being solved with a footgun.
- **State in git or on a laptop.** Local `terraform.tfstate` committed to the repo: no locking (concurrent applies corrupt), secrets in git history forever, merge conflicts in JSON nobody can safely resolve. This is the first thing to fix in any inherited config — migrate with `terraform init -migrate-state` before touching anything else.
- **`local-exec` with unpinned triggers.** A `null_resource`/`terraform_data` provisioner with no `triggers` runs exactly once ever (never re-runs on input changes); with `triggers = { always = timestamp() }` it runs every apply and makes every plan dirty. Both are usually wrong — trigger on the hash of actual inputs, and re-read the escape-hatch taxonomy before keeping it at all.

## Worked micro-examples

**Destructive-change gate in CI (the check humans skip):**

```bash
terraform plan -out=tfplan
terraform show -json tfplan | jq -e '
  [.resource_changes[]
   | select(.change.actions | index("delete"))] | length == 0
' >/dev/null || { echo "::error::plan contains destroys — needs elevated approval"; exit 1; }
```

**Safe refactor, adopt, and guard — the three blocks that replace CLI surgery:**

```hcl
moved {                                   # rename without destroy/create
  from = aws_s3_bucket.assets
  to   = module.assets.aws_s3_bucket.this
}

import {                                  # adopt console-created resource, reviewably
  to = aws_db_instance.legacy
  id = "legacy-prod-db"
}

resource "aws_db_instance" "legacy" {
  # ... config matching reality; iterate until `plan` is empty ...
  lifecycle { prevent_destroy = true }
}
```

**Backend that treats state as a crown jewel (AWS shape):** S3 bucket with versioning + SSE-KMS + tight bucket policy, native S3 state locking (lockfile-based, current replacement for the DynamoDB-table pattern), and separate keys per state: `key = "network/prod/terraform.tfstate"`. Bucket versioning is your state-corruption undo button — verify it's on before you need it.

## How an expert thinks through it: "plan wants to destroy and recreate the prod RDS instance"

Never apply; read *why*. The plan marks `-/+` and names the culprit: `identifier` forces replacement — someone renamed the instance to match a new convention. Options: (a) apply — destroys prod data; obviously no, but say why in the PR so the author learns the `-/+` semantics. (b) Revert the rename — safe, but the convention change had a reason. (c) Check whether this is rename-in-place-able: for RDS, `identifier` change is actually an in-place rename via the API, but the provider models it as replacement? Verify in provider docs/changelog rather than assume — if the provider genuinely forces replacement, (d) do it as a managed migration: snapshot, create new alongside, cut over, then remove the old — as explicit separate changes, not one hidden inside a plan. Also fix the process gap: why did no `prevent_destroy` exist on this instance, and why did CI not flag a delete action on a `data`-tier resource? Add both. Considered and rejected: `terraform state rm` + re-import under the new name — actually the *right* trick when only the resource *address* changed (use `moved` blocks for that), but here the *cloud-side identifier* changed, so state surgery can't help. The lesson encoded: replaces are diagnosed from the plan's "forces replacement" attribute, and stateful resources get both a lifecycle guard and a policy check, because humans will eventually skim.

## Verification / self-check

- After any apply: run `plan` again — it must be empty. A non-empty second plan means perpetual drift (a provider default fighting your config, a value normalized by the API); fix it now or every future plan carries noise that trains reviewers to ignore plans.
- After import or refactor: empty plan is the definition of done.
- Grep state for secrets before declaring a workflow secure: `terraform show -json | grep -i` the credential names you fear. If present, that's your real security boundary — act accordingly.
- For a module: can you write its README contract (inputs, outputs, invariants) in 15 lines? If not, it's doing too much.
- Destructive-change drill: does your pipeline actually stop a plan containing a delete on a guarded resource? Test with a scratch resource, not by faith.
- Stopping rule: infrastructure code is done when plans are empty, state contains no unguarded secrets, every stateful resource has `prevent_destroy`, and a new team member can find which state owns a resource in under a minute. Refactoring module trees beyond that is aesthetics.
