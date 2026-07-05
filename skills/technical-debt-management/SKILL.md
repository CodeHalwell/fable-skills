---
name: technical-debt-management
description: Load when deciding whether/how to pay down technical debt — prioritizing refactoring vs features, evaluating a rewrite vs incremental migration, building a debt registry or business case for debt work, interpreting code-quality/DORA metrics as debt evidence, or managing debt accumulation from AI-generated code. Also when a user says "the codebase is a mess", "should we rewrite", or "how do I get time for refactoring".
---

# Technical Debt Management

Compact sheet. The standard expert corpus — interest = churn × friction, hotspot analysis (2–5% of files absorb most commits; pull git churn before believing any debt narrative), Fowler quadrants with process responses, payback arithmetic with a <2-quarter bar and a remaining-lifetime clause, strangler-fig-over-rewrite with the seam as first project, rewrite veto checklist, migration prerequisites (tests/seam/end-condition + freeze), risk-and-velocity framing to product, boy-scout limits, SonarQube remediation totals as relative signal only, characterization-tests-not-coverage-mandates, delete-before-refactor, bus-factor hotspots where documentation doesn't work — is assumed known and appears only as anchors. Expanded sections carry what baseline answers miss.

## Anchors (apply without re-derivation)

- Portfolio, not pile: most debt is *never repaid, deliberately*; interest of untouched code is zero however ugly. Rank by churn × complexity × roadmap, never by offense taken; distrust any figure denominated in "total days to fix everything."
- Quadrant-check before prescribing (decision or accident? reasonable at the time? **is the generator still running?**). Reckless-quadrant debt regrows after cleanup unless pressure/skills are fixed — cleaning it is mowing a fertilized lawn.
- Payback = principal ÷ (touches/yr × friction/touch). Fund under ~2 quarters on the roadmap's path; decline long-payback items *in writing* and use the decline in negotiation.
- Rewrite: all six vetoes (platform-is-the-debt, stable domain, **written strangler costing**, frozen scope with executive sign-off, old-behavior experts on the team, reversible cutover). No strangler costing = the proposal is emotional; ask for the seam analysis. Cutover criteria = measured behaviors customers exercise, never the feature list.
- Migrations: owner + end condition + day-one ratchet (CI forbids *new* uses of the old pattern) or don't start; a stalled migration is a permanent extra dialect. Each strangler slice deletes the old path in the same PR — no deletion, no progress.
- Boy-scout only inside the function you were editing, separate commits; structural change rides its own PR series.
- Product framing: risk and lead time attached to named features with a payback date; zero craftsmanship language; concede cold-but-ugly code publicly — declining to fix things is what makes the rest credible.
- Test debt is measured by fear ("which change would we not dare make?") and defect escapes, not coverage; characterization tests pin the behaviors the upcoming refactor could break, nothing more.
- Bus-factor: `git shortlog -sn -- path` on every hotspot; remedy is routing actual work through a second person; docs decay and transfer no judgment.

## Instruments baseline answers don't reach for (the delta)

- **The friction log** — the highest-ROI debt instrument per unit effort, and absent from standard playbooks (which jump to churn mining and DORA). Append-only file; one line whenever debt costs anyone >30 min (`2026-06-03 bkim 1.5h orders/ suite flaked 3x`); <30 seconds to write, no triage or blame in the log; monthly someone aggregates by path (`sort | awk`) and the top themes become registry candidates *with empirical dev-day interest attached*. It captures what churn mining can't (flakes, env breakage, re-derived knowledge), it's egalitarian by design — the junior's daily 40-minute flake outranks the staff engineer's pet grievance, and "no friction entries" is evidence *against* funding, whoever is asking — and its numbers are what convert PM conversations from adjectives to arithmetic.
- **Registry entries need a trigger and a forced disposition.** Schema: what/where, quadrant, interest (evidence), principal, ratchet, trigger ("repay before feature X touches this"), disposition. Quarterly review; two reviews untouched → explicit "ACCEPTED permanently" close or escalate. A registry that only grows is a graveyard that destroys the practice's credibility.
- **Dependency/platform debt is calendar-debt, not churn-debt.** Churn-based prioritization prices the untouched EOL framework at zero — until the CVE lands and it's an emergency at the worst time. Manage it as scheduled maintenance (EOL tracking, automated PRs, deadlines) *outside* the hotspot process; never let either budget cannibalize the other. This is the standard method's blind spot: hotspot analysis structurally cannot see step-function debt.

## AI-codegen era (as of mid-2026 — the numbers baseline models lack)

- **GitClear 2026 "Maintainability Gap" (623M changes, 2023–2026):** code-block duplication up **81%** since 2023; within-commit copy/paste ~9.4% (2022) → **~15.7%** (H1 2026); moved/refactored code collapsed ~21% → **~3.8%**; changes touching code >12 months old down **~74%** (1.7% → 0.46%); error-masking constructs up **47%**. Baseline models cite 2024-era directional findings; these are the current magnitudes.
- **DORA 2025 (~5,000 respondents): AI now associates with *higher* throughput — a reversal of 2024 — but still lower delivery stability.** Framing: AI amplifies existing strengths and dysfunctions; gains land where version control, small batches, and platform quality are already strong. (Baseline models still quote the 2024 "throughput down 1.5%" result.)
- Operational consequences: duplication is the dominant AI-era debt class (peak interest exactly when requirements change N copies) — ratchet it with `jscpd`-style no-new-duplication CI gates; review order = right layer? existing abstraction? error contract? *then* correctness, because AI code is locally plausible and globally wrong; watch the **maintenance ratio** — new wrappers piling up around an untouched-but-hot core means its interest is being paid as duplication around it; reinvest a fixed slice of the throughput gain as agent-driven paydown (banking 100% is deliberate-reckless at machine speed — name it that to leadership, and make the reinvestment slice float on measured duplication/hotfix trends).
- Resolution of the DRY tension: **gate within-change copy-paste, tolerate cross-domain similarity.** Rule of three still applies — same-reason duplication is debt; different-reason similarity merged prematurely becomes coupling debt, which costs more than the duplication did.

## Verification / self-check

1. Every prioritized item has behavioral evidence (churn, friction-log lines, localized lead-time/failure deltas); adjectives get cut or measured.
2. Every item quadrant-labeled; reckless items ship with a process/ratchet component.
3. The plan visibly declines something (named cold-ugly code accepted); a plan that fixes everything prioritizes nothing.
4. Migrations have owner + end condition + day-one ratchet; rewrites have the 6-item checklist with written strangler costing.
5. Product framing: risk, lead time, named beneficiaries, payback date — no craftsmanship words.
6. Success metric is the interest (friction, lead time in the target area), declared before work starts; paydown stops when the triggering feature is unblocked and the ratchet holds — "the file is now beautiful" is past the stopping point.

Time-sensitive claims (GitClear 2026, DORA 2025) accurate as of mid-2026; re-verify before citing later.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)
- Probed 13 claims: 11 baseline (compressed to anchors), 1 partial (rewrite vetoes — sharpened the written-strangler-costing veto), 1 delta (2025/2026 AI-codegen evidence: baseline cites DORA 2024 throughput-drop and directional GitClear; current state is throughput-reversal-with-instability and the 2026 magnitudes).
- Opus cold reproduced: hotspot method incl. power-law expectation and tooling, payback arithmetic with thresholds, quadrants + process responses, strangler prerequisites and two-and-a-half-systems failure, PM negotiation incl. all three objections, coverage-mandate rebuttal with mutation testing, SonarQube treatment, deletion candidates, bus-factor remedies.
- Biggest gaps kept expanded: the friction log (absent from baseline playbooks), calendar-debt vs churn-debt blind spot, forced registry dispositions, 2026 figures.
