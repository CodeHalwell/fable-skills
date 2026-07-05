---
name: technical-debt-management
description: Load when deciding whether/how to pay down technical debt — prioritizing refactoring vs features, evaluating a rewrite vs incremental migration, building a debt registry or business case for debt work, interpreting code-quality/DORA metrics as debt evidence, or managing debt accumulation from AI-generated code. Also when a user says "the codebase is a mess", "should we rewrite", or "how do I get time for refactoring".
---

# Technical Debt Management

## Core mental model

- **Debt is a portfolio, not a pile.** Each item has a principal (cost to fix), an interest rate (ongoing drag), and a term (how long you'll hold it). Portfolio management means most debt is *never repaid* — deliberately. The skill is choosing which 10% to pay, not scheduling all of it. A backlog labeled "tech debt" sorted by how offended engineers are is not a portfolio; it's a complaint box.
- **Interest = change frequency × friction per change.** Debt in code nobody touches has an interest rate of zero — it is free, forever, no matter how ugly. Debt in a file changed weekly compounds. This single principle resolves most prioritization arguments: pull the churn data before believing anyone's debt narrative, including your own.
- **The type of debt dictates the response.** "Debt" covers a rushed hack, a decade of missing tests, a wrong architecture, and a framework the world abandoned. These need different treatments (revert, incremental hardening, strangler migration, platform migration). Diagnose before prescribing — Fowler's quadrants below are the diagnostic.
- **Invisible debt cannot be negotiated.** Product can't trade off against a vibe. Debt earns capacity only when it appears as evidence: lead-time deltas on tickets touching the bad module, incident counts, onboarding time, a friction log. Your job is to convert engineer pain into legible risk and velocity numbers.
- **Debt is a loan against understanding, and some loans are good.** Shipping with a known shortcut to hit a market window is often correct. The failure is not borrowing — it's borrowing without recording the loan, or paying interest indefinitely on a loan you could retire in a week.
- **Stop the borrowing before repaying the principal.** Refactoring a module while the pattern that produced the mess is still the path of least resistance just resets the clock. Every paydown needs a ratchet: a lint rule, a template, a CI check, or an ownership change that makes the old pattern harder than the new one.

## Taxonomy: name the quadrant, change the response

Fowler's two axes — deliberate/inadvertent × prudent/reckless — matter because each quadrant implies a different fix and a different *process* fix:

| Quadrant | Sounds like | Correct response | Process response |
|---|---|---|---|
| Deliberate–prudent | "Ship now, we know the cost, ticket filed" | Honor the plan: repay on the recorded schedule | None — this is healthy engineering |
| Deliberate–reckless | "No time for design, just ship it" (chronically) | Triage the damage like inadvertent debt | Fix the incentive/pressure system, not the code first |
| Inadvertent–reckless | "What's a transaction boundary?" | Fix the code AND close the skill gap (review, pairing) | Training, review standards — else it regenerates |
| Inadvertent–prudent | "Now we know how we *should* have built it" | Normal cost of learning; refactor if hot, else leave | None — punishing this poisons the culture |

Reasoning chain when you meet debt: (1) *Was this a decision or an accident?* Check commit messages, ADRs, ask the author. (2) *Was it reasonable given what they knew?* If yes, drop the blame frame entirely — it changes what you say to the team and to product. (3) *Is the generator still running?* Deliberate-reckless and inadvertent-reckless debt regrows after cleanup unless you fix pressure or skills. Cleaning code produced by a broken process is mowing a lawn you keep fertilizing.

## Prioritization: interest-rate reasoning and hotspots

The expert's first question is never "what's the worst code?" — it's "**where does bad code intersect with current change?**" That intersection is a hotspot (Tornhill's method: churn × complexity), and hotspots are always a tiny fraction of the codebase — typically low single-digit percent of files absorb most commits.

Reasoning chain, in order:

1. **Pull churn from version control** (12 months, or since the last big reorg). This is behavioral ground truth; static analysis is opinion.
2. **Cross with a complexity/health signal** on the top-churn files only. Deep indentation, function length, or a tool score (CodeScene's Code Health, or plain `wc -l` as a first cut). Complexity without churn = ignore. Churn without complexity = fine, that's just an active file.
3. **Cross with the roadmap.** A hotspot on next quarter's critical path outranks a hotter file in a feature area going into maintenance. Interest is *future* change frequency; git history is only its best predictor.
4. **Estimate interest in dev-days/year**, roughly: (touches/year) × (extra friction per touch, from friction log or PR cycle-time comparison). Compare to principal (refactor estimate). Payback under ~2 quarters: strong case. Payback beyond the code's expected lifetime: leave it, however ugly.
5. **What would change my mind:** a roadmap shift (frozen module about to become hot — its debt just repriced from 0% to prime), an incident cluster (stability risk trumps velocity math), or a compliance/security finding (not really debt — a liability with a deadline).

Prior to state out loud: **most of what engineers call "the debt problem" is 2–5 files.** Whole-codebase quality scores hide this; they average the frozen 95% into the number. Distrust any debt figure denominated in "total days to fix everything" (SonarQube-style remediation totals) — it counts free debt at full price.

### First-move table by debt class

Different debt classes have different interest mechanics, so the first move differs:

| Debt class | Interest mechanism | First move | Default treatment |
|---|---|---|---|
| Code-level (duplication, complexity, naming) | Slows every edit in the file | Churn × complexity ranking | Boy-scout if small; scoped refactor if hotspot |
| Architectural (wrong boundaries, god module) | Every feature pays a coordination/coupling tax | Map change-coupling (files that always change together across module lines) | Strangler behind a seam; never drive-by |
| Test debt | Fear-driven slowdown + defect escapes; interest spikes exactly when you refactor | Ask "what change are we afraid to make?" | Characterization tests on hotspots only, not blanket coverage |
| Dependency/platform debt | Near-zero daily, but step-function spikes (CVE, EOL, forced migration) | Inventory versions vs EOL/advisory dates | Scheduled maintenance with deadlines — calendar-driven, not churn-driven |
| Knowledge debt (bus factor) | Single owner of a hot module = latency + risk | Cross-reference hotspots with author concentration (`git shortlog -sn -- path`) | Pairing/review rotation on that module; docs are the weakest fix |
| Product/domain debt (features nobody uses, wrong model) | Every change navigates dead concepts | Usage instrumentation | Delete before refactoring — the cheapest refactor is `rm` |

The last row is the most-missed lever: before pricing a refactor of anything, ask **"can we delete it instead?"** Usage data kills more debt per dev-day than any refactoring technique.

## Making debt visible

- **Debt registry:** a lightweight, reviewed list — not Jira dust. Each entry: what/where, quadrant, interest estimate (who it slows, how often), principal estimate, ratchet needed, trigger condition ("repay before feature X touches this"). Review quarterly; entries untouched for two reviews get an explicit decision: *accept permanently* (document and close) or escalate. A registry that only grows is a graveyard and destroys the practice's credibility.
- **Friction log:** engineers append one line whenever debt costs them >30 min ("2h: test suite for orders/ flaky, reran 3x"). After a month you have the empirical interest rates — usually surprising, and far more persuasive to product than adjectives.
- **DORA-style metrics as evidence, not as the debt measure:** lead time and change-failure rate *localized to the debt area* vs the rest of the codebase is the argument ("changes under `billing/` take 3.1x longer and fail 2x more often"). Global DORA numbers are too coarse to indict a module. As of 2026, DORA's framing is that AI-era throughput gains amplify whatever system quality already exists — instability shows up downstream of weak controls (see AI section).
- **Never present debt without a denominator.** "300 code smells" means nothing. "The 4 files where 60% of this quarter's changes land score in the bottom decile of health" is a decision-ready sentence.

## Paying it down: boy-scout rule and its limits

Boy-scouting (leave code slightly better per visit) is the right tool **only** for debt whose fix fits inside the current change without expanding the review contract: renames, dead-code deletion, extracting a function, adding a test for the thing you're touching. Keep it in separate commits; if it doubles the diff, split the PR.

It structurally **cannot** retire: wrong module boundaries, schema problems, cross-cutting patterns, framework migrations. Those need design, sequencing, and migration windows — attempting them drive-by yields the worst state in software: **two ways of doing everything, forever.** A half-migrated pattern costs more than either endpoint because every reader must learn both and every change must decide.

Rule for structural debt: **no migration starts without an owner, an end condition, and a ratchet** (CI check forbidding *new* uses of the old pattern — enforce the ratchet on day one, migrate the stock over time). If you can't name all three, don't start; a started-and-stalled migration is negative progress. Budget dedicated capacity — a fixed weekly slice or a scoped project — and aim it at registry hotspots, not at whatever annoys whoever has slack time.

## Rewrite economics

**Default: strangler fig.** Route traffic through a seam, replace behind it slice by slice, delete the old slice each time, keep both deployable throughout. Every increment ships value and de-risks the next. If no seam exists, *creating the seam is the first project* — an anti-corruption layer or API boundary — not a reason to big-bang.

**The second-system trap:** a rewrite must hit feature parity with a moving target while the old system keeps accreting behavior nobody documented, staffed by people who romanticize what they'll fix "this time" and pack in every deferred idea. Expected timeline slip is the norm, not the exception, and mid-rewrite you're paying two mortgages: full maintenance on the old system plus full build cost on the new, with zero shipped value until cutover.

A true rewrite is justified only when **all** of these hold — treat it as a checklist, misses are vetoes:

1. The platform/runtime itself is the debt (EOL, unhirable, blocks a hard requirement) — no amount of refactoring escapes it.
2. The domain is now well-understood and stable — you're not rewriting to *discover* requirements.
3. A strangler approach was seriously costed and is genuinely worse (usually: no seam can be created at acceptable cost).
4. Scope can be frozen: the old system goes feature-frozen except for legal/security fixes, with executive sign-off in writing.
5. People who understand the old system's behavior (not its code — its *behavior*, the weird edge cases customers depend on) are on the rewrite team.
6. There's a cutover plan with reversibility, and the business survives the parity window.

If someone proposes a rewrite and can't produce the strangler costing (item 3), the proposal is emotional, not economic. Sympathize, then ask for the seam analysis.

## Negotiating debt work with product

- **Never argue cleanliness, craftsmanship, or "doing it right."** These are hobby-words to a PM — they price at zero. Translate into the two currencies product already trades in: **risk** (incident probability, blast radius, bus factor, compliance exposure) and **velocity** (lead time, forecast confidence, cost of delay on named roadmap items).
- **Attach debt to features, not to virtue:** "Feature X routes through `billing/`; at current friction it lands ~3 weeks later and with elevated incident risk. Two weeks of paydown first makes X faster *and* Y and Z after it." This converts paydown from a tax into an investment with a named beneficiary.
- **Bring the localized evidence** (friction log, per-module lead time) and a small ask with a payback date. Small, measured, delivered-on-time debt projects build the credit rating that funds bigger ones. A vague "we need a quality sprint" spends credibility and buys nothing durable.
- **Concede genuinely free debt.** Publicly deprioritizing ugly-but-frozen code is the move that makes product trust your prioritization of the rest. An engineer who wants to fix everything is arguing aesthetics; one who declines to fix cold code is arguing economics.
- **Pre-handle the standard objections**, because they will come in this order:
  - *"Can't we do it after the release?"* — Sometimes yes; say yes when true (that's the deliberate-prudent quadrant: record it, schedule it). Say no with a number when false: "after" means the feature is built *on top of* the debt, raising the principal — give the revised estimate for fixing it post-release vs pre.
  - *"Why wasn't this raised earlier?"* — Don't get defensive; the honest answer is usually "the interest rate just changed" (the roadmap moved into this module). Debt priorities are supposed to change when plans change.
  - *"How do I know this won't grow into a rewrite?"* — Show the scoped end condition and the pause-safe sequencing (each slice ships alone). This is exactly the fear the strangler structure exists to answer; if you can't answer it, your plan is under-specified.
  - *"Can we just be more careful instead?"* — "Careful" is not a mechanism. Point at the generator analysis: the debt regrows because a pressure or tooling gradient favors it; the ratchet changes the gradient, willpower doesn't.

## Debt in the AI-codegen era (as of 2026)

Verified state of the evidence, mid-2026:

- **GitClear's 2026 "Maintainability Gap" research** (623M code changes, 2023–2026): code-block duplication up 81% since 2023 and at record levels; within-commit copy/paste rose from ~9.4% (2022) to ~15.7% (H1 2026) while moved/refactored code collapsed from ~21% to ~3.8%; changes touching code >12 months old fell ~74% (1.7% → 0.46%); error-masking constructs (broad catches that swallow failures) up 47%; short-term churn up 15%. Direction: AI assistance correlates with more paste, less refactoring, less legacy maintenance.
- **DORA 2025 (State of AI-assisted Software Development, ~5,000 respondents):** AI adoption now associates with *higher* throughput (a reversal from 2024) but *still with lower delivery stability*. Framing: AI is an amplifier — it magnifies existing team strengths and dysfunctions; the gains land where version control practice, small batches, and platform quality are already strong.

Operational consequences — what to actually do differently:

- **Treat duplication as the dominant AI-era debt class.** Generation makes copying cheaper than abstracting, so entropy's path of least resistance changed. Duplication is peak-interest debt precisely when requirements change (N divergent copies to find and fix). Ratchet it: duplicate-block detection in CI (e.g., `jscpd`) with a no-new-duplication gate, and prompt/review norms that ask "does this already exist?" before "does this work?"
- **Review for design, not just correctness.** AI code is unusually likely to be locally plausible and globally wrong — reimplementing an existing helper, ignoring the codebase's idioms, catching-and-logging where the system's error contract says propagate. Reviewer question order: right layer? existing abstraction? error contract honored? *then* is it correct.
- **Watch the maintenance ratio.** The GitClear signal that old code increasingly goes untouched means debt is *silently aging into the frozen zone while still hot* — teams add beside instead of changing within. If your churn analysis shows new files piling up around an old core that nobody edits but everything calls, that core's effective interest is being paid as duplication around it.
- **Provisioning rule:** velocity gains from codegen are real but partly borrowed. Reinvest a slice of the gain (the same tools that generate code make tests, migrations, and refactors cheaper too — use them for paydown) rather than banking 100% as feature throughput. Teams that bank it all are running the deliberate-reckless quadrant at machine speed.

## How an expert thinks through this

You inherit a 7-year-old order-management service. The team says "it's all debt, we should rewrite"; the PM says velocity halved this year. Internal monologue:

*First: distrust the narrative, pull data.* `git log` churn for 12 months, cross with file size/indent depth. Result: 380 files; 62% of commits touch 6 files; two of those — `OrderProcessor` (4.1k lines, deeply nested) and `PricingRules` — are where every delayed ticket died. The other "horrifying" code people mention in standup is churn-cold. *So "it's all debt" is false. It's mostly two files' debt, experienced daily by everyone.*

*Consider the rewrite.* Checklist: runtime is fine (supported LTS), domain is stable — but item 3 fails immediately: nobody costed a strangler, and `OrderProcessor` has an obvious seam (order-state transitions already flow through one dispatch point). Also item 5 fails: the two engineers who know why the weird partial-refund path exists are the ones who'd be pinned maintaining the frozen system. *Rejected — and rejected out loud with the checklist, so it stays rejected instead of resurfacing every quarter.*

*Consider "20% time for quality."* Rejected: unaimed capacity historically goes to pet peeves — someone will beautify the cold reporting module because it offends them. Untargeted paydown of zero-interest debt at real cost. *Capacity must be aimed at named hotspots.*

*Consider "add tests everywhere first."* Rejected as stated: blanket coverage on 380 files spends most effort on frozen code. Corrected: characterization tests on the two hotspots only — they're a prerequisite to refactoring safely, not a virtue standalone.

*Quadrant check on the hotspots.* Commit history shows `PricingRules` grew via copy-pasted rule blocks under quarter-end pressure for years — deliberate-reckless with a still-running generator. *So the plan must include a ratchet (new pricing rules go through a rule-table interface, lint forbids extending the old switch) or it regrows.*

*Now the PM conversation.* Not "the code is bad": "Tickets touching these 2 files run 3× lead time — here's the Jira query. Q3's discount-engine work lives exactly there. Proposal: 3 weeks — characterization tests, extract the rule engine behind a seam, ratchet on. Discount engine then lands faster than currently forecast, and we deprioritize everything else on the debt list, including things engineers complain about." The concession buys the credibility; the named-feature payback buys the three weeks.

*Stopping rule:* refactor until the discount-engine work is unblocked and the ratchet holds — not until the file is beautiful. Remaining ugliness in the hotspot reprices next quarter with fresh churn data.

### Second scenario: the AI-velocity trade

Six months after rolling out coding agents, feature throughput is up ~40% and leadership wants to bank all of it. You're asked whether that's safe. Internal monologue:

*What does the evidence say generically?* As of 2026: throughput gains are real (DORA 2025 reversed its 2024 finding), but stability degrades where controls are weak, and the industry-wide code signature is more duplication, less refactoring, less old-code maintenance (GitClear). Generic evidence justifies checking, not concluding — *measure this repo.*

*Cheap local measurements, one afternoon:* duplicate-block trend (`jscpd` on HEAD vs 6 months ago), churn concentration (is new code piling up in new files beside an untouched core?), revert/hotfix rate trend, and the friction log. Suppose results: duplication up 2.2×, hotfix rate creeping, and the old `auth/` core has near-zero edits but rising numbers of new wrappers around it.

*Reject "pause AI usage"* — throughput gain is real value, and the tool isn't the quadrant; the missing controls are. This is deliberate-reckless *by the organization* if it continues after this analysis, which is exactly the framing that gets leadership's attention: the loan is now being taken knowingly.

*Reject "add a quality gate on everything"* — blanket gates on a 40%-faster pipeline create queue pressure that teams will route around; gates must be few and targeted (duplication ratchet, error-contract lint) or they get exception-processed to death.

*Proposal:* bank 30 of the 40 points; reinvest 10 as agent-driven paydown aimed at the wrapper accretion around `auth/` (the same agents that generated the wrappers can execute the consolidation cheaply, with human design review). Re-measure duplication and hotfix rate in a quarter; the reinvestment slice floats on those two numbers. *The deliverable is a control system, not a one-time cleanup.*

## Failure modes & pitfalls

- **Prioritizing by static-analysis severity instead of churn × pain.** SonarQube-style "1,240 days of remediation" totals price frozen debt at full value and say nothing about interest. Result: sprints spent fixing cold code while the hotspot burns. Correction: behavioral data (version control) first; static analysis only to rank *within* the hot set.
- **Refactoring cold code because it's offensive.** The strongest emotional pull in debt work and almost always negative-ROI: real principal paid on a 0% loan, plus regression risk in code with no tests and no recent human context. Correction: ugliness is not interest; check churn and roadmap before touching anything.
- **Boy-scouting structural debt.** A feature PR that "also" moves files between modules or renames a core concept halfway. Now review can't separate feature risk from refactor risk, `git blame` is polluted, and if the feature reverts, the refactor half-reverts with it. Correction: structural change is its own PR series with its own owner and ratchet; boy-scouting stays inside the function you were already editing.
- **Starting a migration you can't finish.** The codebase with three ORMs, two HTTP clients, and a "new" service pattern from 2023 at 40% adoption. Every abandoned migration adds a permanent dialect. Correction: no start without owner + end condition + day-one ratchet on new uses; if capacity disappears mid-flight, an explicit decision — finish narrow or roll back — beats drifting.
- **The registry graveyard.** Debt items filed to Jira, never re-reviewed, until "tech debt" label = noise and filing feels pointless. Correction: quarterly review with forced disposition; "accepted permanently" is a legitimate, honest closure that keeps the live list short and believable.
- **Framing debt work as cleanliness to product.** "We need to clean up the code" reliably prices at zero and teaches product that engineering asks for vacations from roadmap. Correction: risk and lead-time framing, attached to named features, with a payback date — every time, even when it feels obvious internally.
- **The annual "debt sprint."** One cathartic week of unfocused cleanup, then eleven months of accumulation — the process equivalent of crash dieting. It also signals that debt is separate from normal work. Correction: continuous small capacity aimed at the registry, plus scoped projects for structural items.
- **Treating inadvertent-prudent debt as negligence.** Retroactively blaming past authors for not knowing what the domain later taught everyone. Poisons culture, and misdirects the fix toward "be more careful" when the code simply needs updating to current understanding. Correction: quadrant-check before assigning any process remedy.
- **Paying principal, leaving the generator running.** Beautiful refactor of the config system; six months later it's regrown, because quarter-end pressure (or the AI assistant's path of least resistance) still rewards the old pattern. Correction: every paydown ships with its ratchet; if you can't articulate the ratchet, you haven't found the cause of the debt.
- **Rewrite by attrition denial.** Teams that reject the big-bang rewrite but also never create the seam, re-litigating the rewrite argument every quarter for years while the strangler option quietly expires (the people who understand the old behavior leave). Correction: "no rewrite" must be paired with funding the seam; the do-nothing option has an expiry date, name it.
- **Feature-parity as rewrite acceptance criteria.** Parity includes ten years of features nobody uses; chasing it is how rewrites go 2–3× over. Correction: cutover criteria are the measured behaviors current customers exercise (instrument the old system to find out — usage data, not the feature list).
- **Accepting AI-generated duplication because review only checked correctness.** Three near-identical validation blocks land in a week, each locally fine; the divergence bug arrives at the first requirements change and gets fixed in two of three copies. Correction (as of 2026 this is the fastest-growing debt intake): duplication gates in CI, and review order = layer/abstraction/error-contract before line-level correctness.
- **Measuring debt paydown by activity instead of friction.** "We closed 40 debt tickets" while lead time in the hot module is unchanged — you paid the wrong principal. Correction: the success metric is the interest (lead time, failure rate, friction-log volume in the target area), stated before the work starts.
- **Test debt managed as a coverage percentage.** Mandating "80% coverage" on a legacy module produces assertion-light tests written to satisfy the gate — they freeze current behavior *including its bugs*, add CI minutes, and give false confidence for the refactor. Correction: measure test debt by fear ("which change would we not dare make?") and defect escapes; write characterization tests that pin the behaviors the upcoming refactor must preserve, not lines for the counter.
- **Premature DRY as debt "prevention."** Merging two similar-looking code paths that are only coincidentally alike creates a shared abstraction both callers must now negotiate — coupling debt, which is more expensive than the duplication was. Correction: rule of three, and check whether the copies change *for the same reason*; same-reason duplication is debt, different-reason similarity is a coincidence. (This cuts against the AI-duplication point above — the resolution is: gate *within-change* copy-paste, tolerate cross-domain similarity.)
- **Treating dependency upgrades as portfolio debt.** Churn-based prioritization says the untouched framework version is free — until the CVE or EOL notice arrives and it's an emergency at the worst time. Correction: dependency/platform debt is calendar-debt, not churn-debt; manage it as scheduled maintenance (automated PRs, EOL tracking) outside the hotspot process, and never let it crowd out the hotspot budget or vice versa.
- **Ignoring knowledge debt until the resignation.** A hotspot with one meaningful author is a compound liability: every fix queues on one person, and their departure converts code debt to archaeology. Correction: check author concentration on every hotspot (`git shortlog -sn --since="2 years ago" -- path`); the remedy is forced circulation (pairing, review rotation, assigning the next feature there to someone else) — documentation alone decays and doesn't transfer the judgment.
- **Letting the debt agenda be set by seniority instead of evidence.** The staff engineer's pet architectural grievance gets funded; the junior's daily 40-minute test flake doesn't get reported. Correction: the friction log is egalitarian by design — fund what it says, and treat "no friction entries" as evidence against funding, whoever is asking.

## Worked micro-examples

**1. Hotspot analysis from raw git, no tooling** (CodeScene et al. do this better; this is the zero-dependency version):

```bash
# Commits per file, last 12 months (churn)
git log --since="12 months ago" --numstat --format= -- src/ \
  | awk 'NF==3 {n[$3]++} END {for (f in n) print n[f], f}' | sort -rn | head -15

# Complexity proxy for the top candidates: mean indentation depth
awk '{ match($0, /^[ \t]*/); d += RLENGTH; n++ } END { printf "%.1f\n", d/n }' src/billing/order_processor.py
```

Rank by churn, then flag files that are also large/deep. Expect a power law — if the top 5 files don't clearly separate from the rest, widen the window or aggregate by directory.

**2. Interest-vs-principal calculation** (the arithmetic behind "is this worth fixing"):

`PricingRules`: 47 touching commits/year; friction log and PR data say each touch costs ~0.5 extra dev-day (re-learning the switch, extra review round, flaky tests) → **interest ≈ 23 dev-days/year**. Refactor estimate (characterization tests + extract rule table + ratchet): **~15 dev-days principal** → payback ≈ 8 months, and the module is on Q3's critical path. Fund it. Compare `LegacyReportRenderer`: 2 touches/year × 1 day friction = 2 dev-days/year interest against ~20 days principal → 10-year payback on code likely retired in 3. Decline it, in writing, and use the decline in the product negotiation.

**3. Debt registry entry** (the whole schema — if an entry needs more than this, it's a design doc):

```markdown
| id | what/where | quadrant | interest (evidence) | principal | ratchet | trigger | disposition |
| D-31 | Pricing switch, src/billing/pricing_rules.py | delib-reckless | ~23 dd/yr (friction log 03-06/2026; PR lead time 3.1x repo median) | ~15 dd | lint: no new cases in legacy switch; new rules via RuleTable | before Q3 discount engine | FUNDED 2026-07 |
| D-14 | Report renderer string templating | inadv-prudent | ~2 dd/yr | ~20 dd | n/a | revisit if reporting re-enters roadmap | ACCEPTED permanently 2026-04 |
```

**4. Friction log — format and use** (the highest-ROI debt instrument per unit effort):

```text
# FRICTION.md — append-only, one line per event, no debate at write time
2026-06-03  bkim    1.5h  orders/ test suite flaked 3x before green (retry roulette)
2026-06-03  asousa  0.5h  re-derived PricingRules tier precedence AGAIN; no source of truth
2026-06-05  jlee    2.0h  local env broke after pulling; billing/ needs undocumented seed data
2026-06-09  bkim    1.0h  reviewer round-trip because OrderProcessor change touched 4 unrelated concerns
```

Rules that make it work: entries take <30 seconds to write (or nobody writes them); no fixing, no blaming, no triaging in the log itself; monthly, someone aggregates by path and theme — `sort` + `awk` is enough — and the top 2 themes become the quarter's registry candidates with dev-day interest figures attached. The log's authority comes from being boring, cheap, and written at the moment of pain rather than reconstructed in a planning meeting.

**5. Strangler-fig sequencing** (for the `OrderProcessor` scenario above — the shape generalizes):

```text
Step 0  Seam audit: all order-state transitions already pass through dispatch();
        if they didn't, step 0 becomes "route them through one" (weeks, ships alone).
Step 1  Characterization tests at the seam: record real inputs/outputs at dispatch()
        (goldens from prod traffic samples), not unit tests of internals.
Step 2  Ratchet ON before any migration: CI fails if a new transition type is
        added to the legacy switch. New behavior must use the new handler API.
Step 3  Migrate ONE transition (pick the simplest, not the most valuable) end-to-end,
        behind a flag; run old and new in shadow mode, diff outputs for a week.
Step 4  Cut over that transition; DELETE the legacy branch for it in the same PR.
        No deletion, no progress — parallel paths kept "just in case" are the
        two-dialects failure mode with extra steps.
Step 5  Repeat by risk-ascending order. Publish a burn-down (transitions remaining);
        a visible counter is what keeps a multi-quarter migration funded.
```

Each step ships independently and the project can pause safely after any step 4 — that pause-safety is the economic argument that beats a rewrite, and it's worth stating explicitly in the product negotiation.

## Verification / self-check

Before presenting a debt plan, confirm:

1. **Every prioritized item has behavioral evidence** — churn data, friction-log entries, or localized lead-time/failure numbers. If any item's justification is an adjective ("messy", "horrible"), pull the data or cut the item.
2. **Every item is quadrant-labeled** and reckless-quadrant items include a process/ratchet component, not just a code fix.
3. **The plan declines something visibly** — named ugly-but-cold debt explicitly accepted. A plan that fixes everything prioritizes nothing.
4. **Migrations have owner + end condition + day-one ratchet**; rewrites have the full 6-item checklist with a written strangler costing.
5. **The product framing contains zero craftsmanship language** — only risk, lead time, and named roadmap beneficiaries, with a payback estimate you'd bet on.
6. **Success metrics are interest metrics** (friction in the target area, before/after), declared before work starts.
7. **Stopping rule stated:** paydown ends when the triggering feature is unblocked and the ratchet holds — not when the code is beautiful. If you can't say what "done" unblocks, the item isn't ready to fund.

Time-sensitive claims (GitClear duplication/refactoring trends, DORA throughput-vs-stability findings) are accurate as of mid-2026; re-verify before citing specific figures later than that.
