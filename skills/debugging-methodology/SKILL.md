---
name: debugging-methodology
description: Load when diagnosing a bug, test failure, crash, regression, flaky behavior, or "works on my machine" report — any time the task is finding WHY software misbehaves rather than writing new features. Covers reproduction discipline, bisection, hypothesis ranking, stack traces/core dumps, and Heisenbug tactics.
---

# Debugging Methodology — expert correction sheet

The standard playbook (reproduce first; `git bisect run` with exit-125 skip; amplify flaky failures before bisecting; TSan for Heisenbugs; ASan/UBSan built with release flags for release-only bugs; heap-snapshot diffing for slow degradation; fix at the boundary, not the crash site; flip-test the fix) is reliable baseline knowledge — apply it without re-deriving it here. This sheet keeps only the corrections and compressed rules that go beyond that baseline.

## Corrections that beat the reflex

1. **Bisect lands on an "impossible" commit (rename, comment, config-key change): believe it, and switch questions.** The reflex is to suspect a corrupted bisect, or to lead with the half-applied-rename theory (a stale consumer silently reading the old key and getting a default). Check that with one grep — but the higher-yield question is "what does this commit *perturb*?": dict/iteration order, memory layout, timing, worker startup order. Usually the commit is the *trigger* and the bug is a latent race or order-dependence that predates it. Confirm by running the **parent** commit under a race detector or at high rep count — if the latent bug fires there too, fix the race and keep the rename. Do not re-run the bisect hoping for a different answer.

2. **`git log` quiet for days before the breakage → stop reading diffs, read the calendar.** Time bombs break unchanged code: cert expiry, date rollover (DST, month/year boundary, leap), disk filling, an external API changing contract, base-image `:latest` drift. When "what changed?" answers "nothing in this repo," the delta is wall-clock time or the environment. Check expiries, disk, and upstream deploys before touching code. Corollary for coordinated systems: the breaking change may have landed in *another* repo/service — bisect over deploy timestamps, not this repo's commits.

3. **After two failed hypotheses from code-reading, the next hypothesis is not the move — instrumentation is.** Two consecutive misses means your model of the runtime state is wrong; more theory compounds the error. Get ground truth at the boundaries, logging *values* with `!r`/equivalent — the difference between `''`, `None`, and `'None'` is often the whole bug, and `print("here")` wastes the cycle. Then explicitly write down 3 alternative hypotheses and the cheapest experiment that discriminates among them.

## Compressed decision rules

- Every experiment must cut the hypothesis space *whichever way it comes out*; if both outcomes leave you equally ignorant, don't run it. One change per experiment; revert everything that wasn't the fix.
- No repro yet → your job is trigger-finding (log mining, env diffing, load simulation), not cause-finding. Different tactics.
- Regression → bisect before reading any code. Prod-only → diff environments (versions, env vars, data volume, TZ, locale, concurrency) before code. One failing input → delta-minimize it (`creduce`/`cvise`, ddmin by hand). No idea at all → verify state at the pipeline midpoint and recurse into the bad half.
- Bisection generalizes beyond time: binary-search the config diff (apply half the differing keys), the failing input, or the code path (stub half) when a debugger isn't available.
- Priors, each level ~3–10x less likely: your last change ≫ your other code ≫ your usage of a dependency ≫ recently-upgraded dependency ≫ mature-library bug ≫ OS/compiler/hardware (~1 in 10⁴ — demand a minimal repro surviving on a second machine first).
- Prints don't appear / an obviously-fatal edit doesn't crash → you are editing code that isn't running (stale artifact, shadowed module, wrong process/branch/venv). Prove execution with a deliberate crash before doing anything else.
- Never fix a Heisenbug with a sleep: it converts a frequent failure into a rare one — strictly worse. Find the missing happens-before edge (lock, join, channel, await).
- Two executions, one good one bad → identical instrumentation on both, diff the logs (sort first if ordering is nondeterministic). The first divergence is downstream of the cause and upstream of the symptom — bisect between them.
- Garbage stack trace (null addresses, one repeated frame) → stack smashing or a call through a corrupted function pointer; the trace itself is unreliable — switch to ASan on a repro instead of reading it harder.
- Boundary/off-by-one suspicion → don't reason about the range, enumerate n=0,1,2 explicitly in a scratch test. Boundary bugs are precisely the ones strong engineers reason about instead of enumerating.
- Grep for the error string finds nothing ≠ "not my code": the string may be built by formatting/concatenation, live in a dependency, or be localized. Grep distinctive substrings and the exception class.
- "Clearing the cache fixed it" is not a diagnosis — clearing also restarts the process and forces a rebuild. Isolate: does clearing *only* the cache fix it?
- Prod instrumentation: one correlation ID per request; never per-item prints at high QPS — sample (`if random.random() < 0.001`) or gate on the failing entity ID.
- Flaky test: capture the failure (seed logging, `rr`, loop-until-fail with full logs) *before* any change — each blind rerun destroys evidence. Distrust the reporter's repro recipe: users report the last thing they did, not the trigger.

## Verification before declaring the bug fixed

- Flip test both directions: fix on → repro passes; fix reverted → repro fails again. If you can't turn the bug back ON, you haven't found the cause — "can't repro anymore" after assorted changes is not a fix.
- The root cause must explain *every* observed detail — the timing, the specific values, why it started when it started, why only some users. Unexplained residue means a second bug or a wrong theory.
- Flaky: N clean runs where chance passage < 1% at the prior failure rate (30% flaky → ≥13; run 50).
- Bugs come in families: grep for the same pattern (same author, same copy-paste, same misunderstood API), and audit sibling data through the same boundary — one string-typed datetime from a TEXT column means every datetime through that mapper is suspect.
- Regression test added and confirmed failing on pre-fix code *before* removing instrumentation.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)
- Probed 12 claims: 10 baseline (cut/compressed), 2 partial (sharpened), 0 delta.
- Opus cold already produces the full standard playbook: flaky-aware `git bisect run` with 0.7^K rep math and exit-code conventions (125/≥128), ASan+UBSan on release flags, TSan for Heisenbugs, heap-snapshot diffing, boundary-not-crash-site fixes, flip-test + statistical verification. All of it was cut or compressed to one-liners.
- Biggest gaps: (1) on an "impossible" bisect result, Opus leads with the half-applied-rename theory and bisect-corruption checks, burying the trigger-vs-latent-bug perturbation question (timing/ordering/layout) — now correction #1; (2) no time-bomb heuristic (quiet git log → check calendar/expiries/other repos, not diffs) — now correction #2.
- Restructured from a full survey into a correction sheet per ≥80%-baseline rule; worked examples removed (Opus reproduced both cold).
