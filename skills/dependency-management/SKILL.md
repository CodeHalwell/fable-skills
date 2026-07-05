---
name: dependency-management
description: Load when adding, evaluating, upgrading, pinning, vendoring, or auditing third-party dependencies — package.json/pyproject/Cargo.toml changes, lockfile conflicts, Renovate/Dependabot setup, npm/PyPI supply-chain concerns, "should we use library X", CVE-driven upgrades, or monorepo internal versioning. Covers dependency risk economics, semver reality, install-script attacks, provenance, and upgrade cadence.
---

# Dependency Management

Compact sheet. The standard expert corpus — cooldown/`minimumReleaseAge` as the #1 control (and its exact Renovate/Dependabot/pnpm/uv config), lockfile-is-law with frozen CI installs, library-ranges vs app-locks plus minimal-versions CI, PEP 751 `pylock.toml` with uv as de-facto standard, pnpm v10 blocked install scripts, devDependencies-run-where-the-tokens-live, `^0.x` caret semantics, take-one-side-and-rerun lockfile merges, reachability-first CVE triage, `npm audit fix --force` dangers, the 15-minute new-dep evaluation, dual-React split-brain + `workspace:*` + catalogs, Trusted Publishing/OIDC, vendoring vs registry-proxy leftpad-immunity, overnight-break-means-a-dependency-moved — is assumed known and appears only as anchors. Expanded sections carry the post-2025 facts and judgments baseline models miss.

## Anchors (apply without re-derivation)

- A dependency is a hiring decision priced on the transitive closure, not the manifest line. Never hand-roll code whose bugs are security or spec-compliance bugs (crypto, tz, URL/HTML/email parsing); default to writing anything <200 lines of which you need <20%.
- Apps: ranges in manifest, lockfile is law, frozen installs everywhere automated (`npm ci`, `--frozen-lockfile`, `uv sync --locked`); a Dockerfile that copies `package.json` without the lockfile silently un-freezes the image build. Libraries: permissive ranges, commit the lockfile anyway (dev/CI only), test the declared *floor* (minimal-versions job) or the range is aspirational.
- Cooldown 3–7 days on everything, advisories bypass it (Renovate `minimumReleaseAge` + `vulnerabilityAlerts: 0 days`; Dependabot `cooldown` block; pnpm `minimumReleaseAge`; uv `exclude-newer`). Freshest-is-safest is true only for known-CVE patches; otherwise day-0 adoption makes you the canary.
- Install scripts: pnpm v10 blocks by default — keep `onlyBuiltDependencies` under ~10 genuinely-native entries and read the script before approving; don't silence the warning reflexively. Bun `trustedDependencies`. Prefer wheels / `--only-binary :all:` on Python.
- Malicious-package scanning ≠ CVE scanning: osv-scanner (vulns + OpenSSF malicious list) plus Socket-style behavioral analysis; CVE scanners flag zero worm packages on day zero.
- Lockfile diffs are review surface: scan for new package names and changed integrity hashes on *unchanged* versions (tampering signal).
- Upgrade cadence: small-and-often dominates (pain superlinear in jump size); group non-majors, automerge through your own tests; majors within ~2 quarters or a written freeze + exit plan; platform last, leaves first.
- Overrides/resolutions are tourniquets: untested version combinations; track and remove when the parent releases; audit the block quarterly.
- Phantom deps (hoisted `require` of undeclared packages) break on removal — pnpm strict layout prevents; else `knip`/`depcheck` before deleting.
- Git-branch deps are vendoring-with-extra-steps: pin to a full SHA or don't.
- peerDependencies: plugins declare their host as peer; apps never "fix" peer conflicts with `--legacy-peer-deps` permanently.

## Post-2025 facts and the judgments they change (the delta)

- **Provenance is not a malware verdict — proven in production, May 2026.** The Mini Shai-Hulud campaign published malicious npm packages carrying *cryptographically valid SLSA Build L3 provenance*, signed via OIDC tokens stolen from compromised CI runners. Baseline reasoning gets the theory right ("provenance proves the pipeline, not intent") but doesn't know it's no longer hypothetical. Operational meaning: use provenance to detect impersonation (artifact ≠ claimed repo), never as a safety signal; a "requires attestation" policy stops token-theft republish, not a compromised runner.
- **npm v12 ships install-scripts-disabled as the default** (2026), matching pnpm v10 — if your org pinned old npm to "keep postinstall working," that grace period is ending; build the allowlist now rather than during the forced upgrade.
- **Slopsquatting is a live vector:** attackers register package names LLMs hallucinate. Never install a name sourced from an LLM answer or tutorial without opening the registry page: downloads, repo link that actually contains the code, publish date. A 3-week-old package with a famous-sounding name is a trap. (Same discipline covers classic typosquats: copy install commands from the project's own docs; prefer ownership-verified scopes.)
- **The incident → principle mapping, compressed** (cite the rule, not the war story): leftpad = your build must survive the registry deleting anything (proxy/cache); event-stream = new-publisher-on-old-package is a signal your tooling should surface; xz = build scripts and test fixtures are review surface, burnout is a vulnerability, and defense-in-depth must assume some dependency is hostile; chalk/debug + Shai-Hulud = maintainer identity is the phishable perimeter and CI secrets are the crown jewels (short-lived OIDC over stored tokens); Mini Shai-Hulud = every verification technology relocates the place you must look, it never removes it.
- **Dev-only is not lower-risk where it counts:** Shai-Hulud harvested CI and laptop credentials — the axis is *where the code executes*, not the manifest key. A devDependency in a token-bearing pipeline is production attack surface; the only legitimate discount is runtime-CVE reachability.

## Verification / self-check

1. Reproducibility flip test: wipe the env, frozen install, tests green — then grep the CI workflows to confirm they use the same frozen command (don't assume).
2. Closure diff on any change: new transitive names identified; none are weeks-old packages with no repo.
3. `osv-scanner --lockfile` clean; install-script allowlist unchanged or reviewed as a security decision.
4. Libraries: declared floors actually tested.
5. New prod dep: PR states what was rejected and why, and the wrap boundary (or why none). Can't write the two-sentence removal plan = don't understand the hire.

Stopping rule: frozen + cooldown-with-advisory-bypass + script allowlist + 15-minute evaluations = done for a normal product team; SBOM programs, mirrors, hermetic builds only pay when others depend on your output or compliance demands it.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)
- Probed 14 claims: 12 baseline (compressed to anchors), 1 partial, 2 delta (Mini Shai-Hulud valid-provenance malware, May 2026; npm v12 install-script default flip).
- Opus cold reproduced: cooldown as #1 control with exact four-tool config, Shai-Hulud/chalk-debug mechanics and windows, PEP 751/uv state, pnpm v10 defaults + Bun, library-vs-app pinning + minimal-versions CI, reachability triage, `audit fix --force`, dual-major split-brain + workspace protocol/catalogs, token revocation + Trusted Publishing, registry-proxy leftpad-immunity.
- Biggest gaps: 2026 incidents/defaults above, plus slopsquatting (absent from baseline answers) — these are what this skill is for.
