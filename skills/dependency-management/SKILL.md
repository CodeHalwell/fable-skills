---
name: dependency-management
description: Load when adding, evaluating, upgrading, pinning, vendoring, or auditing third-party dependencies — package.json/pyproject/Cargo.toml changes, lockfile conflicts, Renovate/Dependabot setup, npm/PyPI supply-chain concerns, "should we use library X", CVE-driven upgrades, or monorepo internal versioning. Covers dependency risk economics, semver reality, install-script attacks, provenance, and upgrade cadence.
---

# Dependency Management

## Core mental model

- **Adding a dependency is a hiring decision, not a download.** You are hiring the maintainer onto your team: their release discipline becomes your upgrade treadmill, their security posture becomes your attack surface, their burnout becomes your fork. The install is free; the subscription is not. Price every `npm install foo` as: (hours to write the 20% of the feature you actually need) vs. (years of upgrades × breakage rate + audit cost + the tail risk of compromise or abandonment). For small utilities the buy-side almost never wins.
- **Your dependency is the transitive closure, not the name in the manifest.** A one-line `require` that pulls 80 transitive packages is 80 hiring decisions made blind, 80 maintainer accounts that can be phished. Judge weight by `npm ls --all | wc -l` / `cargo tree | wc -l`, not by the package's own size. leftpad (2016) broke thousands of builds not because anyone chose it but because it rode in transitively — the lesson is not "npm bad," it's *you own your closure*.
- **Semver is a social contract, and it is routinely broken — in both directions.** Empirically, minor/patch releases ship breaking changes (accidentally or "it was documented as internal"), and major releases are often trivial. Treat the version number as the maintainer's *claim*, your test suite as the *verification*. Never design a process that is safe only if semver is honored.
- **Compromise windows are hours; your exposure window is your choice.** The 2025–2026 npm attacks (chalk/debug, Shai-Hulud, AntV/TanStack) had malicious versions live for 2 hours to ~2 days before takedown. Consumers who install versions <24h old absorb nearly all of that risk; a 3–14 day cooldown on new versions eliminates most of it at almost zero cost. This is the single highest-leverage supply-chain control that exists as of 2026.
- **Trust attaches to people and pipelines, not artifacts.** xz-utils (2024) was a multi-year social-engineering attack on a burned-out maintainer — every artifact was "authentic." In May 2026 the Mini Shai-Hulud campaign published malicious npm packages with *cryptographically valid SLSA Build L3 provenance* by stealing the OIDC token from a compromised CI runner. Provenance tells you *which pipeline built it*, never *whether that pipeline was clean*. Signatures verify identity, not intent.

## First move by situation

| Situation | First move | Why |
|---|---|---|
| PR adds a new dependency | Run the 15-minute evaluation below before reading the feature code | The dep outlives the feature; it's the bigger review surface |
| CVE advisory fired | Check reachability first: do you call the vulnerable function with attacker-influenced input? | Most advisories are unreachable in your usage; triage beats panic-upgrading everything |
| Build broke, no code change | Diff the lockfile against last green; check whether CI runs an unfrozen install | "Nothing changed" almost always means a dependency changed |
| npm/PyPI compromise in the news | Grep your *lockfile* (not manifest) for the affected names+versions, then check CI secret exposure during the window | Transitive closure is what you run; manifests lie by omission |
| Package deprecated/unmaintained | Check if it's *finished* vs. *abandoned* (does it wrap a moving target?) before migrating | Migrating off stable, done code is negative-value churn |
| Two teams want different majors of X | Escalate now; never ship the split | Dual-major installs create singleton split-brain bugs that surface far from the cause |
| Lockfile merge conflict | Take one side wholesale, re-run the resolver | Hand-merged lockfiles produce graphs no resolver ever validated |

## Decision framework: should this dependency exist at all?

Ask in this order; each answer gates the next:

1. **"What fraction of this library do I need?"** If <20% and the needed part is <200 lines, default to writing it. A date-formatting call does not justify hiring moment's successor. Exceptions where you always buy regardless of fraction: crypto, timezone databases, HTML/URL/email parsing, sanitization, anything with a spec longer than your codebase. Rule: *never hand-roll code whose bugs are security bugs or standards-compliance bugs.*
2. **"Is it alive?"** Evidence, cheapest first: last release date and cadence (steady beats bursty); open-issue response time on *bug* issues (ignore feature requests); bus factor — how many people have publish rights and commit regularly (one person = one phishing email from compromise; this is how chalk/debug fell in Sep 2025); is it a company-backed project whose company still needs it? A finished, stable library (chunked releases stopped because it's *done*) is fine for leaf utilities but disqualifying for anything touching an evolving platform (browser APIs, Python versions, TLS).
3. **"What does its history predict?"** Read the CHANGELOG for the last 2 majors: how often did minors break people (check issues titled "X broke after upgrade")? How long were deprecation windows? A library that shipped 3 majors in 2 years will ship 3 more — budget that treadmill or don't hire.
4. **"What's the transitive bill?"** Run the dry-run install and count new packages and install scripts. 5 new transitive deps is normal; 150 means you're importing a framework wearing a utility's name. Check whether any new dep requires install scripts (`pnpm licenses`/`pnpm approve-builds` will surface them).
5. **"What's the exit cost?"** Estimate the diff to remove it later. Wrap wide-API deps (HTTP clients, ORMs, logging) behind your own thin module so exit cost stays constant; never wrap narrow pure functions (waste of indirection).

What changes the answer: a dep you'd reject for a product you maintain for 10 years is fine for a prototype — but write the decision down, because prototypes ship. And a *dev-only* dependency is cheaper on the security axis only if it never runs in CI with credentials; Shai-Hulud specifically harvested CI secrets, so `devDependencies` running in a token-bearing pipeline are production attack surface.

## Decision framework: pinning and lockfiles

The question is never "pin or not" — it's *who consumes this manifest*:

- **Applications (deployables, services, CLIs you ship):** exact-pin nothing by hand, range in the manifest, **lockfile is law**. The lockfile is the pin; ranges in the manifest just describe upgrade intent. Commit the lockfile, install with the frozen mode in CI — `npm ci`, `pnpm install --frozen-lockfile`, `uv sync --locked`, `pip install -r` a compiled file — never the bare install command, which will happily rewrite resolution. A CI job that runs `npm install` is not reproducible and silently absorbs whatever was published last night.
- **Libraries (things others `require`):** stay permissive in the manifest (`^1.2.0`, `>=1.2,<2`) because your pins multiply across every consumer — exact-pinning in a library causes diamond conflicts and forces duplicate installs. But still *commit your lockfile* for your own dev/CI reproducibility; it doesn't ship to consumers (npm ignores dep lockfiles; Rust guidance now says commit Cargo.lock even for libraries). Then add a CI job that installs with *lowest* satisfying versions (`cargo +nightly -Z minimal-versions`, or a scheduled unlocked build) so your declared ranges are actually tested, not aspirational.
- **Python as of 2026:** `uv.lock` is the de-facto app lockfile; PEP 751 `pylock.toml` (approved 2025) is the interchange standard — uv exports it (`uv export --format pylock.toml`), pip can install from it (experimental). Don't hand-maintain `requirements.txt` anymore; compile it.
- **Never** resolve a lockfile merge conflict by hand-editing hashes. Take either side's file wholesale, then re-run the resolver (`pnpm install --lockfile-only`, `uv lock`) so the tool regenerates a consistent graph.

## Decision framework: upgrade cadence

Small-and-often strictly dominates big-bang. Reasoning: upgrade pain is superlinear in the size of the version jump (changelogs compound, migration guides assume you did the previous one, blame-space for regressions grows), while the per-upgrade fixed cost is small once automated. A team that upgrades weekly does 50 trivial upgrades a year; a team that upgrades annually does one archaeology project with the same total delta and none of the isolation.

- Run **Renovate** (preferred: grouping, `packageRules`, lockfile maintenance, monorepo awareness) or **Dependabot** (fine for simple repos, zero setup on GitHub). Configure both of these or don't bother: (1) **cooldown** — Renovate `minimumReleaseAge` / Dependabot `cooldown` block (GA since July 2025); (2) **grouping** so 30 patch bumps arrive as one PR your CI validates as a unit.
- Auto-merge patch/minor for deps with good test coverage *through your own test suite*, never on trust. Majors get a human and the migration guide.
- Security advisories bypass the cooldown (Dependabot does this automatically) — a known CVE in the old version outweighs the unknown risk of the new one.
- Ordering rule for a stale codebase: upgrade the *platform* (runtime, framework) last; get every leaf dep current first so the framework upgrade isn't fighting stale peers.

## Supply-chain defenses that actually work (state of 2026)

Ranked by leverage; implement top-down:

1. **Cooldown on new versions** (see above). Nearly every 2025–2026 npm compromise window closed within days.
2. **Disable install scripts.** pnpm v10 blocks dependency lifecycle scripts by default (allowlist via `onlyBuiltDependencies` / `pnpm approve-builds`); npm v12 flips the same default (as of 2026); Bun has `trustedDependencies`. Yarn Berry and pip-installing-wheels don't run arbitrary install code either (sdists do — prefer wheels, use `--only-binary :all:` where feasible). Historically most npm malware detonated in `postinstall`; this single setting defangs it. Keep the allowlist under 10 entries and treat every addition as a security review.
3. **Frozen installs everywhere automated** (CI, Docker, prod). An unfrozen install in CI is an RCE-by-publish primitive.
4. **Malicious-package scanning, not just CVE scanning.** These are different products: `osv-scanner` (v2, Google) checks lockfiles against known vulns *and* the OpenSSF malicious-packages list; Socket.dev does behavioral analysis (install scripts, network/fs access, obfuscation, typosquat similarity) that catches never-before-seen malware. CVE scanners alone would have flagged none of the Shai-Hulud packages on day zero.
5. **Scope your credentials like you'll be phished, because a maintainer you depend on will be.** npm classic tokens were permanently revoked (late 2025); granular tokens now max 90 days with 2FA. If you publish packages: use Trusted Publishing (OIDC, GA on npm since July 2025; PyPI supports GitHub/GitLab/Google Cloud/ActiveState) so there is no long-lived token to steal, and enable provenance. If you consume packages: treat provenance as *one* signal (it proves repo↔artifact linkage) but remember Mini Shai-Hulud shipped valid attestations — provenance ≠ clean.
6. **Typosquat hygiene.** Registry-side defenses block only near-exact confusables; the working consumer defenses are: copy-paste install commands from the project's own README/docs (never type from memory, never from an LLM answer without checking — slopsquatting, registering names LLMs hallucinate, is a real 2025+ vector), prefer scoped packages (`@org/x` — the scope is ownership-verified), and let Socket/osv-scanner diff review new names in PRs.

## The canonical incidents, distilled to operating principles

Each famous incident encodes one durable rule; cite the rule, not the war story:

- **leftpad (npm, 2016)** → *availability is part of the dependency contract.* An 11-line package's unpublish broke builds worldwide. Principles: your build must survive the registry deleting anything (proxy/cache with retention); trivially-writable code should be written, not hired; know your transitive closure before it teaches you itself.
- **event-stream (npm, 2018)** → *ownership transfer is a supply-chain event.* A tired maintainer handed the keys to a volunteer who backdoored it for one target app. Principle: "maintained" by *whom* matters as much as "maintained" — new-publisher-on-old-package is a signal your tooling (Socket flags this) should surface.
- **xz-utils (2024)** → *the attacker's budget can exceed your diligence.* Years of legitimate contribution, pressure-campaign sock puppets, then a backdoor hidden in build scripts and binary test files — caught only by luck (a Postgres engineer profiling ssh latency). Principles: maintainer burnout is a security vulnerability, fund/adopt what you depend on; build steps and test fixtures are code review surface; no evaluation checklist catches a determined nation-state, so defense-in-depth (least-privilege runtime, egress control) must assume some dependency is hostile.
- **chalk/debug + Shai-Hulud (2025)** → *maintainer identity is the perimeter, and it's phishable.* One phished maintainer → 2.6B weekly downloads' worth of packages trojaned; the worm then auto-propagated via harvested npm tokens in CI. Principles: cooldowns convert "hours until takedown" into a non-event; CI secrets are the crown jewels — short-lived OIDC over long-lived tokens; assume any single-maintainer dep will eventually be compromised for a window.
- **Mini Shai-Hulud (2026)** → *cryptographic trust chains are only as good as their weakest runtime.* Valid SLSA provenance on malware, signed with OIDC tokens lifted from compromised runners. Principle: verification tech shifts *where* you must look (from artifact to pipeline), it never removes the need to look.

## Vendoring judgment

Vendor (copy the source into your tree) when: the dep is small, stable, and load-bearing (a 300-line algorithm); upstream is dead but the code is fine; you need patches upstream won't take; or your build must work air-gapped. Do not vendor things that receive security patches (TLS, parsers, image decoders) — you are silently opting out of the patch stream, which is how vendored copies of zlib/xml libs stay exploitable for a decade. If you must, record the exact upstream commit in a header and subscribe to upstream advisories. The middle path most people forget: a *fork you track* (dependency pinned to your fork's URL) keeps the patch stream visible while giving you control. And an org-level registry proxy (Artifactory/Verdaccio/`--offline` caches) gives you leftpad-immunity — unpublished upstream ≠ broken build — without vendoring anything.

## Monorepo internal versioning

- Internal packages should reference each other by **workspace protocol** (`"foo": "workspace:*"` in pnpm/Yarn, path deps in Cargo, `uv` workspace sources), never by registry version — otherwise you build against yesterday's publish of your own code.
- Choose fixed vs. independent versioning by *consumer*: if external users install your packages à la carte, use independent versions with Changesets (each PR declares its bump; the tool computes the cascade). If packages only make sense together (an SDK family), fix-version them — one number, zero matrix-compatibility questions.
- Never let two internal packages depend on different majors of a shared third-party dep "temporarily." In hoisted node_modules layouts this creates split-brain singletons (two React copies = hook errors; two instances of anything with module-level state = unexplainable bugs). Enforce with a single version catalog (pnpm `catalog:`, or syncpack) at the root.

## How an expert thinks through this

Scenario: a PR adds `fast-csv-parse` (hypothetical) to a payments service to parse a 50MB nightly settlement file.

*First question — do we need to hire?* CSV parsing is spec-shaped (quoting, embedded newlines, BOM, encodings) — hand-rolling is the classic trap where your bugs are correctness bugs against a spec. So buying is right in principle. Reject "just `line.split(',')`" immediately: it fails on quoted commas, and settlement data will contain them.

*Second — is this the right hire?* Check the registry page: last publish 3 weeks ago — good — but version history shows 0.x with 40 releases in a year. 0.x means the maintainer explicitly reserves the right to break every release; on a payments service that's a weekly treadmill. Check maintainers: one account, no 2FA badge, no provenance. Check `npm ls` dry-run: 23 transitive deps including a `postinstall` that builds a native addon "for speed." A native build step in a CSV parser is a red flag twice over: it's the malware detonation point *and* an ops burden (musl/alpine builds break). Check downloads: 4k/week. Low usage means few eyes; the herd-immunity argument for popular packages (someone else finds the malware in hours) doesn't apply.

*Consider the alternative hires.* Node has `csv-parse` (part of the long-lived `csv` project): decade of history, multiple maintainers, pure JS, no install scripts, millions of weekly downloads. It's likely slower. *Does speed matter?* 50MB nightly batch — even 10MB/s finishes in 5 seconds, once a day. Performance is a non-requirement being used to justify risk. Reject `fast-csv-parse`.

*Consider not hiring at all, seriously this time:* the file comes from one known counterparty — fixed schema, no quotes? Tempting, but counterparties change formats without notice, and the failure mode (silently mis-parsed money) is maximal. Spec-shaped + money = buy the boring one. 

*Now the terms of employment:* add `csv-parse` with a caret range, lockfile commits the exact version, Renovate `minimumReleaseAge: 7 days` already covers it, no install scripts to approve. Wrap it? No — the call surface is one function in one module; a wrapper adds indirection with no exit-cost reduction. Write down the rejected candidate and why, in the PR description, so the next person doesn't "upgrade" to the fast one.

Stopping rule: the evaluation above took ~15 minutes. Spend that for any new prod dependency; spend an hour+ only for deps that touch auth, money, crypto, or parsing untrusted input. Below that, process cost exceeds risk.

## Failure modes & pitfalls

- **Running `npm install` (unfrozen) in CI/Docker.** Every build re-resolves ranges and will absorb a package published minutes ago — this is exactly the window the chalk/debug attackers monetized (live 2 hours). Use `npm ci` / `--frozen-lockfile` / `uv sync --locked`. Symptom that you're exposed: CI passes/fails differently than local with no code change.
- **`Dockerfile` copies `package.json` but not the lockfile** (or `.dockerignore` excludes it), so the image build resolves fresh even though the repo is disciplined. Check: build must fail if the lockfile is absent — frozen installs do; bare installs silently proceed.
- **Treating the lockfile diff as noise in code review.** A 3,000-line lockfile diff from a one-dep bump means the resolver moved *other* things — that's where a malicious transitive version rides in. Minimum bar: scan for *new package names* and *changed registry URLs/integrity hashes for unchanged versions* (the latter is a tampering signal). Tools: `pnpm why`, Socket's PR diff, `osv-scanner --lockfile`.
- **Believing `^1.2.3` protects you from breakage.** Caret ranges assume semver honesty; minors break people constantly. The protection is lockfile + CI on upgrade PRs, not the range operator. Conversely, don't "fix" this by exact-pinning a *library's* manifest — you'll cause duplicate-install diamonds in every consumer (two copies of your peer dep = two module-level singletons = the classic "Invalid hook call" / instanceof-fails bug class).
- **Confusing `dependencies` and `devDependencies` risk profiles.** "It's only a devDependency" is false comfort: dev deps run on developer laptops (with SSH keys, cloud creds) and in CI (with publish tokens) — Shai-Hulud's worm propagated precisely by harvesting those. The axis that matters is *where does its code execute*, not which manifest key it's under.
- **Ignoring pnpm's blocked-build warning.** pnpm v10 prints "Ignored build scripts: …" and people add the package to `onlyBuiltDependencies` reflexively to make the warning go away — reintroducing the exact attack vector the default closed. Approve only packages that genuinely need compilation (native addons: esbuild, sharp, better-sqlite3), and read the script first (`cat node_modules/<pkg>/package.json | jq .scripts`).
- **Trusting provenance/attestations as a malware verdict.** "It has npm provenance / SLSA attestation, it's safe" — May 2026 Mini Shai-Hulud packages carried *valid* SLSA Build L3 provenance signed via stolen CI OIDC tokens. Provenance authenticates the build pipeline; it says nothing about whether the source or pipeline was compromised. Use it to detect *impersonation* (artifact not built from the claimed repo), not maliciousness.
- **Upgrading everything the moment it's released ("freshest = safest").** Without a cooldown you are the canary. Freshest-is-safest is only true for *security patches to known CVEs*; for everything else, day-0 adoption maximizes supply-chain exposure. Set `minimumReleaseAge`/`cooldown` and let advisories bypass it.
- **Deferring majors until forced.** Skipping v2 and v3 to jump v1→v4 means migration guides no longer apply (each assumes the previous major), deprecation shims that would have warned you are already deleted, and community answers are gone. If you can't afford majors now, you're choosing to afford something worse later. Budget: every prod dep gets its major within ~2 quarters of release or gets an explicit "we are freezing this, exit plan is X" note.
- **Resolving a security advisory by adding an `overrides`/`resolutions` force-pin and moving on.** Forcing a transitive version the parent never tested can break the parent silently (it satisfied the advisory scanner, not the parent's compatibility). Overrides are a tourniquet: file/track the upstream bump, remove the override when the parent releases. Audit your overrides block quarterly — every stale entry is an untested version combination.
- **Removing a dep from the manifest without checking phantom usage.** In hoisted node_modules, code can `require` packages it never declared (they were hoisted by a sibling). Removal then breaks *other* code at runtime, not install time. pnpm's strict layout prevents this class; with npm/Yarn hoisting, grep for imports before removing, or use `depcheck`/`knip`.
- **Vendoring a security-relevant lib and forgetting it exists.** The vendored copy silently exits the advisory pipeline — scanners key off manifests and lockfiles, not copied source. If you vendor, add the upstream to your watch list explicitly and record the commit SHA in the vendored tree.
- **Monorepo internal deps by registry version.** `"@ourorg/utils": "^2.1.0"` inside the same monorepo builds against the *published* package, so local changes to utils don't take effect until publish — the "my fix works locally in utils but not in the app" mystery. Use `workspace:*` (it's rewritten to a real version at publish time).
- **Adopting a package because an LLM (or a tutorial) named it, without checking the registry.** Hallucinated package names get registered by attackers (slopsquatting). Before installing anything you didn't already know: open the registry page, check downloads, repo link that actually contains the code, and publish date — a 3-week-old package with a famous-sounding name is a trap.
- **`npm audit fix --force` as a reflex.** The `--force` variant installs semver-*major* bumps of direct deps to satisfy advisories — it will silently move you across breaking majors with zero migration work done. Run plain `npm audit fix`, triage the remainder by reachability, and take majors deliberately. Related: chasing a zero-advisory dashboard by force-pinning transitive versions trades a *known, possibly unreachable* vuln for *unknown, untested* version combinations.
- **Misreading caret semantics on 0.x.** `^0.2.3` means `>=0.2.3 <0.3.0` — for 0.x, caret only floats the *patch* level, and 0.x minors are declared-breaking. Teams "on the latest 0.x" via caret are actually frozen at a minor and surprised months later. If you must depend on 0.x software in prod, treat every update as a major: read the diff.
- **Depending on a git branch instead of a tag/SHA.** `"foo": "github:org/foo#main"` is a moving target with no immutability, no cooldown, no registry-side malware scanning, and it breaks the moment the branch is force-pushed or deleted. Pin git deps to a full commit SHA, and treat them as vendoring-with-extra-steps: you now own watching that repo.
- **Misusing `peerDependencies` direction.** A plugin must declare its host (`react`, `eslint`) as a peer, not a regular dep — declaring it regular bundles a second host copy into every consumer (the dual-React bug again, self-inflicted). Symmetric error: apps "fixing" peer-warnings with `--legacy-peer-deps` permanently, which disables the resolver's only compatibility check; fix the actual conflict or use an explicit override with a tracking issue.
- **The kitchen-sink internal `common`/`utils` package.** Every internal consumer inherits every transitive dep of everything in the grab-bag, and any change forces a rebuild/republish cascade across the monorepo. Split by dependency weight: pure-function utils (zero deps) separate from the package that drags in AWS SDKs.

## Worked micro-examples

**1. Renovate config encoding the whole strategy (app repo):**

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["config:recommended"],
  "minimumReleaseAge": "7 days",
  "lockFileMaintenance": { "enabled": true, "schedule": ["before 6am on monday"] },
  "packageRules": [
    { "matchUpdateTypes": ["patch", "minor"], "groupName": "non-major", "automerge": true },
    { "matchUpdateTypes": ["major"], "automerge": false, "dependencyDashboardApproval": true },
    { "matchDepTypes": ["devDependencies"], "matchUpdateTypes": ["patch", "minor"],
      "groupName": "dev non-major", "automerge": true }
  ],
  "vulnerabilityAlerts": { "minimumReleaseAge": "0 days" }
}
```
Cooldown for everything, advisories exempt, non-majors grouped and automerged *through CI*, majors need a human. Dependabot equivalent uses the `cooldown:` block (GA July 2025) with `default-days: 7`.

**2. pnpm supply-chain settings (pnpm v10+, `pnpm-workspace.yaml`):**

```yaml
minimumReleaseAge: 4320        # minutes = 3 days; registry-side cooldown at resolve time
onlyBuiltDependencies:          # install-script allowlist — keep this list tiny
  - esbuild
  - sharp
```
Scripts blocked by default (v10 behavior); the two entries are real native-build packages. Every addition to this list is a security decision, not a warning-silencing chore.

**3. Python app discipline with uv (as of 2026):**

```toml
# pyproject.toml — ranges express intent; uv.lock is the law
[project]
dependencies = ["httpx>=0.27,<1", "pydantic>=2.7,<3"]

[tool.uv]
exclude-newer = "7 days"   # uv's cooldown: resolver ignores anything published more recently
```

```bash
uv lock                      # resolve; commit uv.lock
uv sync --locked             # CI/prod install: fails if lock is stale, never re-resolves
uv lock --upgrade-package httpx   # targeted upgrade, one PR, one review
uv export --format pylock.toml -o pylock.toml   # PEP 751 interchange for pip-only consumers
```

Same shape as the JS setup: cooldown at resolve time, frozen installs in automation, upgrades as reviewable single-package diffs. Publishing side: use PyPI Trusted Publishing via `pypa/gh-action-pypi-publish`, which generates PEP 740 attestations automatically — no API token to leak.

**4. The buy-vs-build arithmetic, explicit.** Feature: retry-with-jitter around HTTP calls. Library option: a retry package + 6 transitive deps, history of 2 majors in 3 years. Build option: ~40 lines with tests. Cost model: build = 3 hours once. Buy = 0 hours now + 2 major migrations × ~2h + weekly Renovate PR review amortized ~15 min/yr + nonzero compromise tail on a package with one maintainer. Build wins — *and* the code has zero upgrade treadmill forever. Flip the inputs (feature = TLS cert validation, build cost = weeks, bug cost = catastrophic): buy wins in one step. The framework is the same; only the numbers move.

## Verification / self-check

Before declaring dependency work done:

1. **Reproducibility flip test:** delete the env (`rm -rf node_modules` / fresh venv), install with the frozen command, run tests. Then confirm CI uses the *same frozen command* — grep the workflow files, don't assume.
2. **Closure diff:** for any added/upgraded dep, list *new* transitive names (`pnpm why`, lockfile diff) and confirm none are unknown-to-you packages published in the last few weeks with no repo.
3. **Script surface:** `osv-scanner --lockfile=<file>` clean, and the install-script allowlist unchanged (or its change reviewed as security-relevant).
4. **Range honesty (libraries only):** your declared minimum versions actually pass tests (minimal-versions build or scheduled unlocked CI), or you tighten the range to what you test.
5. **Exit note:** for any new prod dep, the PR states what was rejected and why, and where the wrap-boundary is (or why none). If you can't write the removal plan in two sentences, you don't understand what you just hired.

Stopping rule: when installs are frozen and reproducible, cooldown + advisory bypass are automated, install scripts are allowlisted, and every manifest change gets the 15-minute evaluation — stop. Further hardening (full SBOM programs, registry mirrors, hermetic builds) is warranted only when you ship software others depend on or operate under compliance mandates; for a normal product team it's diminishing returns past the list above.
