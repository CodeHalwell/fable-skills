---
name: ci-cd-pipelines
description: Load when designing, writing, or debugging CI/CD pipelines — GitHub Actions workflows, build/test/deploy stages, caching, flaky builds, pipeline security (secrets, untrusted PRs, action pinning), deployment gating, monorepo CI, or self-hosted runner decisions.
---

# CI/CD Pipelines

## Core mental model

- **Two SLOs: time-to-signal and trustworthiness.** Fix trust before speed — a faster suite nobody believes is faster noise. Fail fast, spend late; parallelize everything without a true data dependency.
- **CI is remote code execution as a service.** Every security question reduces to: whose code runs, with whose privileges? Untrusted (fork PR) code must never meet secrets.
- **Artifacts flow forward; nothing is rebuilt.** Promote the digest through environments; rebuilding per environment means you tested one binary and shipped another.
- For a slow pipeline, in order: stop running unneeded work → parallelize → cache → shard → only then bigger hardware (measure first; I/O-bound suites on 16-core runners is a money fire).

## Caching

Key on lockfile hash + `runner.os` (and arch — newly relevant with ARM runners), restore-keys on a prefix; prefer setup-actions' built-in `cache:`. Docker: `build-push-action` with `cache-from/to` paired with a cache-ordered Dockerfile or it caches nothing. **Cache poisoning is a real post-exploitation channel:** PR branches can't write main's cache, but a compromised trusted-branch job can seed caches consumed by later privileged jobs — release/signing jobs build from scratch or from caches only trusted branches can write; when in doubt on the release path, skip the cache.

## GitHub Actions specifics

- Reusable workflows = job-level units for org-standard pipelines (centrally versioned, pinned by ref); composite actions = step-level dedupe inside the caller's job. Called workflows don't see caller secrets (`secrets: inherit`) and caller `env:` does not propagate — the "works inline, breaks when extracted" pair.
- **OIDC everywhere; a long-lived cloud key in repo secrets is a finding.** Trust policies scope `sub` to repo *and* ref/environment (never `repo:org/*`), and use the **immutable `repository_id` claim** so trust survives renames and can't be re-claimed by a recreated same-name repo.
- Every deploy workflow gets `concurrency: { group: deploy-<env>, cancel-in-progress: false }` — `true` copied from the PR snippet kills half-finished production deploys. PR groups: `cancel-in-progress: true` to stop burning runners.
- `timeout-minutes` on every job (default is 6 hours; one hang holds a runner, a concurrency slot, and the merge queue hostage).

## Security posture

- **`pull_request_target` + checkout of PR head is the canonical hole** — fork code runs with base-repo secrets. Split: unprivileged `pull_request` produces artifacts; privileged `workflow_run` consumes them *as data, never executing them* (and never interpolating them into shell).
- Script injection: never interpolate `${{ github.event.* }}` into `run:`; pass through `env:` and quote.
- **Pin third-party actions to full commit SHAs** — the 2025 tj-actions/changed-files compromise (CVE-2025-30066, ~23k repos) worked by moving version tags to a malicious commit. GitHub's immutable-actions publishing reduces this class, but SHA-pinning remains the portable rule; let Renovate bump pins.
- Top-level `permissions: contents: read`; secrets scoped to **environments** with required reviewers — and set environment *branch* restrictions too, or anyone with push access to any branch can author a workflow targeting the environment; without both, the approval gate is decoration.
- Attestation is cheap: `actions/attest-build-provenance` at build, `gh attestation verify` (or admission policy) at deploy — closes "where did this artifact come from" in ~30 minutes of work.

## Flaky pipeline economics

2% flaky × 50 PRs/day ≈ a false red daily → "re-run fixes it" culture → real failures get re-run too, and the signal is gone (that's the expensive part, not compute). Policy: auto-detect (fail→pass on same-SHA retry), quarantine out of the required path *visibly* (ticket, owner, deadline), track quarantine size as team health. Blanket `retry: 3` is signal destruction dressed as pragmatism — a 1-in-3 race condition ships silently. Root causes are boringly consistent: shared state, real network, time dependence, port collisions, missing awaits.

## Deployment gating and monorepo CI

- Environments with protection rules; promotion by digest; post-deploy verification *in the pipeline* (a job ending at `kubectl apply` has requested, not deployed). Prefer machine gates (canary metrics, error budgets) over rubber-stamp approvals that launder responsibility.
- Path filters first; graduate to affected-graph tools (Nx/Turborepo/Bazel) when the hand-maintained "if lib changed, test these 14 apps" map appears — that map is a dependency graph, badly.
- **The required-check/skip trap has two distinct failure modes, and models conflate them:** a *job-level* `if:`-skipped required check reports "skipped," which branch protection **treats as passing** — path-filtered pipelines quietly merge untested code; a *workflow-level* `on.paths` filter means the check is **never reported** and the PR hangs on "Expected." The fix for both: one fan-in job as the *only* required check, `if: always()`, explicitly failing on `contains(needs.*.result, 'failure') || contains(needs.*.result, 'cancelled')` (skipped upstreams are acceptable — that's the path filter working).
- Merge queues solve "two green PRs break combined" by testing each against the queue head; adopt when merge rate makes rebase-and-rerun a daily tax (~10–20+ merges/day on the branch), not below.

## Self-hosted runners

Justified by private-network access, GPUs, extreme scale, compliance — and you now own patching, autoscaling, and isolation. Persistent runners accumulate state between jobs; **self-hosted on a public repo = fork PRs execute on your infra — the whole vulnerability, not a nuance.** If you must: ephemeral runners (actions-runner-controller), never public repos, isolated cloud permissions. If the reason is "hosted is slow," price larger hosted runners against an SRE first.

## Failure modes & pitfalls (checklist)

- **`GITHUB_TOKEN` pushes don't trigger workflows** (recursion protection) — the "bot PR shows no CI" mystery; use a GitHub App token or explicit `workflow_dispatch`.
- Every `if:` without a status function carries an implicit `success()` — `if: steps.x.outcome == 'failure'` alone never runs; cleanup/notify needs `always()`/`failure()` spelled out.
- Matrix: `fail-fast: true` cancels siblings and people chase the wrong leg (set false for diagnostic matrices); matrix job outputs collapse to whichever leg wrote last — per-leg results go through artifacts.
- Shallow checkout (depth 1, no tags) silently breaks `git describe`/changelogs/affected-diffs — `fetch-depth: 0` or `fetch-tags: true`.
- Monorepo `hashFiles('package-lock.json')` at root when the lockfile lives in `apps/web/` — paths are repo-root-relative.
- Job outputs must be declared in the `outputs:` map; step outputs aren't automatically job outputs.
- Artifacts: bounded retention (release artifacts belong in a registry), scoped per run (cross-workflow passing needs explicit download by run ID).
- `continue-on-error` as flake management makes failure invisible on a green build — canary matrix legs only.

## Worked micro-example: a hardened deploy job

```yaml
deploy-prod:
  needs: [build]
  runs-on: ubuntu-latest
  timeout-minutes: 20
  environment: production            # required reviewers + env-scoped secrets + branch restriction
  concurrency: { group: deploy-prod, cancel-in-progress: false }
  permissions: { id-token: write, contents: read }
  steps:
    - uses: actions/checkout@08c6903cd8c0fde910a37f88322edcfb5dd907a8   # SHA-pinned (comment the tag)
    - uses: aws-actions/configure-aws-credentials@a159d7bb5354cf786f855f2f5d1d8d768d9a08d1
      with:
        role-to-assume: arn:aws:iam::123456789012:role/gha-prod-deploy  # trust scoped repo+environment+repository_id
        aws-region: us-east-1
    - env:
        IMAGE: ${{ needs.build.outputs.image-digest }}   # promote what was built, never rebuild
      run: |
        gh attestation verify oci://"$IMAGE" --owner my-org
        ./scripts/deploy.sh "$IMAGE"
    - run: ./scripts/smoke.sh https://api.example.com    # deploy isn't done at 'apply'
```

## How an expert thinks through it: "CI takes 30 minutes and everyone re-runs it"

Trust first: 30 days of run data shows 8% same-SHA flips — two e2e specs sharing a seeded user plus one real third-party API call; quarantine all three today with tickets. Then latency: the DAG is fully serialized lint→build→unit→e2e; parallelize (wall time = e2e's 13m), shard e2e 4-way (~7m), lockfile-keyed caches (3m installs → 20s) → ~8 minutes. Rejected: 16-core runners (measured: I/O-bound), test-impact analysis for e2e (sharding was an hour of work for most of the win — cheap-and-dumb before smart), deleting e2e (the problem was isolation, not existence). Stopping rule: p50 feedback <10 min, same-SHA reproducibility >99%.

## Priors an expert carries

- A red main is an incident: revert in minutes or fix-forward within the hour.
- "Random" CI failure priors: test isolation > real network > runner contention > cache staleness > actual infrastructure — in that order.
- Workflow security review = the same four greps every time: `pull_request_target` + head checkout, `${{ github.event.* }}` in `run:`, tag-pinned actions, over-broad permissions/stray long-lived keys. Run them before reading anything else.

## Verification / self-check

- Same-SHA re-run twice: identical result, or find the nondeterminism first.
- The four security greps come back clean.
- Trace one artifact prod → commit → run → attestation; if deploy rebuilds instead of promotes, fix that first.
- Kill a dependency mid-run: does the pipeline fail clearly or hang/mislead?

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)

- Probed 14 claims: 13 baseline (cut/compressed), 1 partial (sharpened), 0 delta.
- Opus cold nails: pwn-request mechanics + the workflow_run split, tj-actions/SHA-pinning (with CVE number), GITHUB_TOKEN no-trigger, implicit `success()`, cache-poisoning scoping and release-path rule, OIDC `repository_id` hardening, deploy concurrency, attestation flow, the four audit greps, matrix-output collapse, shallow-checkout, and flake economics incl. why blanket retries are wrong.
- Sharpened: the required-check/skip semantics — Opus says a skipped required check makes the PR "hang waiting"; job-level `if:` skips are actually **treated as passing** by branch protection (silent untested merges), while only workflow-level path filters hang. The two-failure-mode distinction is this file's sharpest retained correction.
