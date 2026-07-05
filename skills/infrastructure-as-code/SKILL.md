---
name: infrastructure-as-code
description: Load when writing or reviewing Terraform/OpenTofu (or Pulumi/CDK), designing module or state architecture, debugging state drift or plan surprises, choosing between IaC tools, handling secrets in IaC, or setting up environments and policy checks for infrastructure changes.
---

# Infrastructure as Code

## Core mental model

- **State is the crown jewel** — a three-way diff (config vs state vs reality); every mystery is one leg disagreeing. Remote state with locking from day one; state access is admin access.
- **Review the plan, not the HCL diff.** `~` update, `-/+` replace (the plan names the forcing attribute — that attribute is the whole investigation), `-` destroy. Any replace/destroy of a stateful resource requires explicit human intent.
- **Blast radius is a design input**: split state by owner × change cadence (`network`/`data`/`platform`/per-team services), stateful stores in their own smallest-possible state. Prefer plain data-source lookups by tag over `terraform_remote_state` across team boundaries — remote-state reads couple you to the producer's layout and grant access to their secret-bearing state.
- Emergency console changes are allowed; the incident isn't closed until codified or reverted. Run scheduled drift plans, not just PR plans.

## Tool landscape (as of 2026 — the fast-moving part)

- **Terraform (BSL, ~1.14.x) and OpenTofu (MPL-2.0/CNCF, ~1.11–1.12) are genuinely diverged tools now.** Terraform: ephemeral resources (1.10), write-only arguments (1.11), Stacks (HCP-coupled). OpenTofu: **state encryption (1.7 — enabling it is a one-way door; encrypted state is unreadable by Terraform)**, early variable eval in backends (1.8), provider `for_each` (1.9), OCI registry support (1.10), its own ephemeral/write-only (1.11), **dynamic/conditional `prevent_destroy` (1.12 — stock Terraform still requires a literal)**. Models trained pre-2026 lag OpenTofu by 2+ minor versions and miss the one-way encryption door. Choosing: BSL exposure or state encryption → OpenTofu; HCP/Sentinel investment → Terraform; otherwise whichever the org standardized on — never both on the same resources.
- **S3-backend locking is native lockfile-based now** (`use_lockfile = true`, TF 1.10 / Tofu 1.9); the DynamoDB table is legacy for new setups. Bucket versioning is the state-corruption undo button — verify before needing it.
- Pulumi/CDK: pick when the logic is genuinely programmatic (generating infra from application metadata), not because HCL `for_each` felt awkward once; you inherit a testing-culture requirement, and CDK adds CloudFormation's slowness and rollback model underneath.

## Environments, modules, layout

Directory-per-environment (thin roots calling versioned modules, own backends); workspaces only for ephemeral copies — as prod/staging separation, `workspace show` is the only guard against applying prod with staging vars, and you can't pin prod to an older module. `envs/prod` and `envs/staging` should diff cleanly; every directory maps to exactly one state, guessable from the path; roots contain no resource logic. Modules: small opinionated variable surfaces (a god-module with 80 variables is a config file wearing module syntax); pin by tag (`?ref=v2.3.0` — an unpinned source means someone else's merge changes your prod plan); outputs are contract (removal breaks unknown remote-state readers); ≤2 nesting levels; no one-resource pass-through modules unless they add policy.

## Validation ladder

fmt/validate → tflint (invalid instance types, deprecated args) → **policy on the JSON plan (the highest-value layer)** → `terraform test` for modules with real logic → Terratest-style ephemeral integration only for a platform team's core modules. Leaf-config confidence comes from plan review + staging applies, not test suites. Policy starts warn-only, promotes to deny after tuning — a policy that blocks legitimate work gets culturally bypassed and then protects nothing.

## Import, refactor, secrets

- Import blocks (declarative, plannable) over CLI import; iterate until **plan is empty — that is the definition of done** (a non-empty post-import plan would mutate the live resource you just adopted). `moved` blocks > `state mv`; `removed` blocks to disown without destroying. Import stateful/referenced things; recreate cheap fungible ones.
- Secrets, best first: (1) never let Terraform touch them (IAM roles/OIDC, `manage_master_user_password`, ACM); (2) **ephemeral values + write-only arguments** (TF 1.10–1.11 / Tofu 1.11) — e.g. `password_wo` + `password_wo_version` fed from an ephemeral secrets-manager read, landing in neither state nor plan; (3) reference-don't-inline (pass the ARN/path); (4) `sensitive = true` is display masking only — plaintext in state, plan files, and `output -json`; (5) when a secret unavoidably lands in state: access control + encryption + audit. **Plan artifacts embed values, including sensitive ones — classify tfplan like state**; teams lock the bucket then upload the plan as a world-readable CI artifact.

## Plan-review discipline

CI posts the plan; humans approve the plan; **apply runs the saved plan artifact** — a fresh apply after approval ships whatever changed in between, sight-unseen (and `apply tfplan` refusing a stale plan is the safety working, not an obstacle). Automate destructive-change detection on `terraform show -json` (require elevated approval on any `delete` action — humans skim 400-line plans). `prevent_destroy` on every database/state bucket/DNS zone/KMS key — **but the guard lives in config: deleting the resource block deletes the guard with it, and the next plan happily destroys.** Backstop with plan-JSON policy (deny deletes on protected tags) and cloud-native deletion protection (`deletion_protection`, termination protection), which survive config deletion.

## Debugging a surprising plan

Which leg moved? Config (check module refs and provider bumps — zero local diff needed), state (surgery?), reality (console drift). Perpetual `~` normalization diffs: match the API's canonical form, don't `ignore_changes`. `-/+`: read the `# forces replacement` attribute. `known after apply` cascades: find the one upstream computed root; ignore the fan-out. "Already exists" on create: import or delete the stray — never rename-to-dodge (now there are two). Priors in mature configs: provider bumps > unpinned modules > console drift > API normalization > actual Terraform bugs, in that order.

## Escape hatches

Native resource → community provider → generic API/REST resource → `local-exec` **last** (invisible to plan, no drift detection, CI-runner deps; no `triggers` = runs once ever, `timestamp()` triggers = dirty every plan — hash the actual inputs). Two `local-exec`s orchestrating each other means this piece doesn't belong in Terraform. Every one carries a comment naming the native gap and an issue link for removal.

## Failure modes & pitfalls (checklist)

- `count` for collections with changeable membership → index-shift destroys everything after the removed element; the tell is a plan destroying resources whose config "didn't change." `for_each` on stable keys.
- Unpinned providers / uncommitted lockfile → different plans from identical config; `providers lock -platform=` for both CI and laptop platforms.
- `ignore_changes` as drift concealer (legitimate only for externally-mutated fields like autoscaler `desired_count`).
- `depends_on` on a data source forces read-at-apply → perpetual `known after apply`; if the object is managed in the same config, reference the resource attribute instead. Bootstrap circularity = split bootstrap state or explicit two-phase, not data-source hacks.
- Wrong-workspace applies; `-lock=false` habits and state surgery under a live lock (stuck locks: `force-unlock` with the lock ID after confirming the holder is dead); `-target` as a habit is a state-splitting problem being solved with a footgun — every use ends with a full clean plan.
- State in git/on laptops: the first thing to fix in any inherited config (`init -migrate-state` before touching anything else).

## Worked micro-example — the destructive-change gate

```bash
terraform plan -out=tfplan
terraform show -json tfplan | jq -e '
  [.resource_changes[] | select(.change.actions | index("delete"))] | length == 0
' >/dev/null || { echo "::error::plan contains destroys — needs elevated approval"; exit 1; }
```

## How an expert thinks through it: "plan wants to destroy/recreate prod RDS"

Read why before anything: `-/+` names `identifier` as forcing replacement — a naming-convention rename. Options: revert (safe, default); if only the *Terraform address* changed, `moved` blocks (state surgery can't help when the *cloud-side identifier* changed — know which case you're in); if the cloud identifier genuinely must change, check whether the provider models an API-supported in-place rename as replacement (verify in provider docs; RDS `ModifyDBInstance` can rename), else snapshot → create-alongside → cutover as explicit separate changes. Then fix the process gap: why no `prevent_destroy`, why no CI delete-gate on a `data`-tier resource — add both, because humans will eventually skim.

## Verification / self-check

- After any apply/import/refactor: a second plan must be **empty** — perpetual diffs train reviewers to ignore plans.
- Grep rendered state for the credential names you fear; if present, that's your real security boundary.
- Destructive-change drill: prove the pipeline stops a delete on a guarded resource with a scratch resource, not faith.
- A module README contract fits in 15 lines, or it's doing too much; a new team member finds which state owns a resource in under a minute.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)

- Probed 14 claims: 13 baseline (cut/compressed), 1 partial (sharpened), 0 delta.
- Opus cold nails: native S3 lockfile locking, ephemeral/write-only secret flow with exact versions, `sensitive=true` scope, count/for_each shift, prevent_destroy's self-deletion gap + backstops, moved/import blocks, workspace anti-pattern, saved-plan-artifact discipline, plan-file classification, `-target` judgment, and the RDS-rename walkthrough (including the ModifyDBInstance nuance).
- Sharpened: OpenTofu currency — Opus lags at "1.9–1.10," misses OCI registry support (1.10), dynamic `prevent_destroy` (1.12), and the **one-way door of state encryption** (Terraform can't read encrypted state); that irreversibility is the delta worth keeping loud.
