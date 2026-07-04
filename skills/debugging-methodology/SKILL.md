---
name: debugging-methodology
description: Load when diagnosing a bug, test failure, crash, regression, flaky behavior, or "works on my machine" report — any time the task is finding WHY software misbehaves rather than writing new features. Covers reproduction discipline, bisection, hypothesis ranking, stack traces/core dumps, and Heisenbug tactics.
---

# Debugging Methodology

## Core mental model

- **Debugging is search-space reduction, not inspiration.** Every action you take should cut the space of possible causes, ideally in half. If an experiment can't rule anything out regardless of outcome, don't run it. "Let me try changing X and see" is only legitimate when both outcomes are informative.
- **Reproduce before you theorize.** A bug you cannot reproduce is a bug you cannot verify a fix for. Until you have a reproduction, your job is not "find the bug," it is "find the trigger." These require different tactics (log mining, environment diffing, load simulation) than cause-finding does.
- **The bug is almost never where the symptom is.** The stack trace shows where the invariant was *detected*, not where it was *violated*. A NullPointerException site tells you a null arrived; the question is who manufactured it, possibly seconds and several async hops earlier. Work backwards along the dataflow, not outward from the crash line.
- **Trust the machine, distrust your model.** When observation contradicts your mental model, the model is wrong — including "impossible" observations. The three classic escapes ("the compiler is broken," "it's a hardware fault," "cosmic ray") are correct maybe 1 time in 10,000. First assume: you're running stale code, the wrong binary, the wrong environment, or your print statement isn't where you think it is.
- **One change per experiment.** If you change two things and the bug disappears, you've learned less than one bit and you may have masked the bug rather than fixed it. Revert everything that wasn't the fix.

## Decision framework: what to do first

| Situation | First move | Why |
|---|---|---|
| Regression, was working recently | `git bisect` immediately, before reading any code | Bisection is O(log n) and requires zero understanding; code-reading is O(n) and requires correct understanding |
| Fails in prod, not locally | Diff the environments (versions, env vars, data volume, TZ, locale, concurrency level) before touching code | The code is identical; the delta is by definition environmental |
| Fails for one input/user/record | Minimize the failing input (delta debugging: repeatedly delete half, keep failing half) | A 5-line repro is 100x faster to reason about than a 5MB one |
| Intermittent / flaky | First make it *more* frequent (loop it 1000x, add load, shrink timeouts, run under `stress`/`-race`), THEN debug | You cannot bisect or verify fixes against a 1-in-500 failure |
| Fails only at scale/under load | Suspect: timeouts, pool exhaustion, O(n²) blowup, race windows widening | These are the only things load changes |
| Crash with core dump / stack trace | Read the trace fully (all frames, all threads) before forming any hypothesis | Cheap, complete evidence beats speculation |
| No idea at all | Cut the system in half: verify the data is correct at the midpoint of the pipeline | Binary search over *space* when you can't search over *time* |

## Hypothesis ranking (spend effort proportional to prior probability)

Check in this order; each level is roughly 3–10x more likely than the next:

1. **Your own most recent change** — even if it "can't be related." The last-change prior is the strongest single prior in debugging.
2. **Your code, elsewhere** — a violated assumption between your modules.
3. **Your configuration/usage of a dependency** — wrong flag, misread API contract (e.g., assuming `dict.items()` order pre-3.7 semantics, mutating a list while iterating, timezone-naive datetime compared to aware).
4. **A recently-upgraded dependency** — check the lockfile diff.
5. **A bug in a mature library/framework** — possible, but demand a minimal repro against the library alone before believing it.
6. **OS/compiler/hardware** — only after a minimal repro survives on a second machine.

**When the last-change prior misleads:** (a) latent bugs exposed by an unrelated change — your change altered timing, memory layout, or iteration order, and the real bug is old (uninitialized memory, race, hash-order dependence); (b) time bombs — nothing changed in the repo but a cert expired, a date rolled over, a disk filled, an external API changed; if `git log` is quiet for days before the breakage, look at the calendar and the environment, not the diff; (c) coordinated systems — the breaking change landed in *another* repo/service. Rule: if bisect lands on a commit that plausibly can't cause the symptom, believe the bisect anyway and ask "what does this commit perturb?" (timing, ordering, layout) rather than re-running it hoping for a different answer.

## Bisection tactics beyond `git bisect`

- **Time:** `git bisect run ./repro.sh` with an exit-code script fully automates it. Make the script return 125 for "can't test this commit" (build broken) so bisect skips it. Beware: flaky repros poison bisect — loop the repro inside the script until confidence is high (e.g., 20 clean runs to call it "good").
- **Space:** binary-search the pipeline. Serialize intermediate state, verify it at the midpoint, recurse into the bad half. For a bad rendering: is the model correct going *into* the renderer? For a bad API response: is the DB row correct?
- **Config:** diff the working vs. broken config, then binary-search the diff — apply half the differing keys at a time. Same for feature flags, env vars, and dependency-lockfile diffs.
- **Data:** delta-debug the failing input. Concretely: split the input in half; if either half still fails, recurse on it; if neither does, remove smaller chunks. Tools: `creduce`/`cvise` for code inputs, `ddmin` logic by hand for data.
- **Code:** comment out / stub out half the suspect code path. Crude but decisive when you can't run a debugger (e.g., prod-only, embedded).

## Instrumentation vs. reading code

- **Read code when:** the state space is small, the bug is logic (wrong branch, wrong formula, off-by-one), or reproduction is expensive (prod-only, hardware-in-loop). Reading finds "the code says X but should say Y" bugs fast.
- **Instrument when:** behavior depends on runtime state you can't infer (concurrency, external data, accumulated state), or when your code-reading has produced two failed hypotheses in a row — that means your model is wrong and you need ground truth, not more theory.
- Instrument at **boundaries** (function entry/exit, queue put/get, request/response), logging *values*, not just "got here." `print("here")` tells you control flow; `print(f"user_id={uid!r} balance={bal!r} tz={tz!r}")` tells you the violated invariant. Always use `!r` / equivalent — the difference between `''`, `None`, and `'None'` is often the whole bug.
- Prefer a debugger over prints when you'd need >3 print-iterate cycles: one `breakpoint()` session with the ability to inspect anything beats recompiling five times. Prefer prints when the bug is timing-sensitive or spans many iterations (conditional breakpoints on iteration 10,000 excepted).
- In prod: structured logs + one correlation ID per request. Never debug prod by adding prints in a loop that runs per-item at high QPS — sample (`if random.random() < 0.001`) or gate on the failing entity ID.

## Reading stack traces and core dumps

- Read the **bottom-most frame in *your* code**, not the top frame (usually library internals) and not only the exception message. In Python, read the *last* traceback in a chained `The above exception was the direct cause of...` block last — the *first* one is the root cause.
- In concurrent crashes, dump **all threads** (`jstack`, `py-spy dump`, gdb `thread apply all bt`). The crashing thread is often the victim; another thread holding a lock or corrupting shared state is the culprit. Two threads each waiting on the other's lock = your deadlock, right there.
- Core dumps: `gdb ./binary core`, then `bt full` for locals per frame. If the stack is garbage (addresses like `0x0000000000000000` or repeated single frame), suspect stack smashing or a call through a corrupted function pointer — the trace is unreliable; switch to ASan/valgrind on a repro instead.
- An exception message that names an *internal* detail ("KeyError: 'user_id'") is a dataflow question: search for who builds that dict, not who reads it.

## Differential debugging

When you have a working case and a broken case, stop theorizing and diff:
- Two environments: `diff <(env | sort)` on both; `pip freeze`/`npm ls` diff; kernel/libc versions last.
- Two inputs: minimize both, then diff the minimal pair — the distinguishing byte is the trigger.
- Two executions: log both with identical instrumentation, diff the logs (`diff <(sort a.log) <(sort b.log)` if ordering is nondeterministic). The first divergence point is upstream of the bug's manifestation and downstream of its cause — bisect between them.
- Two versions: bisect (see above).

## Heisenbug tactics (bug disappears under observation)

- Disappears under debugger/prints → timing-dependent: it's a **race**. Adding I/O serialized your threads. Confirm with a race detector (`go test -race`, TSan `-fsanitize=thread`, Java's `vmlens`/JFR) instead of prints.
- Disappears in debug build → **optimization-revealed UB** (C/C++: uninitialized read, strict aliasing, signed overflow) or an uninitialized variable that debug builds zero-fill. Run ASan/MSan/UBSan on the *release* flags.
- Disappears after restart, returns after hours → **accumulated state**: leak, unbounded cache, counter overflow, fragmentation. Graph memory/fd counts over time; don't hunt for a logic bug.
- Appears only in CI → environment: fewer cores (different interleavings), no TTY, different TZ/locale (`LC_ALL=C`), smaller /tmp, different clock resolution. Reproduce with `taskset -c 0` (1 CPU) and `TZ=UTC` locally.
- Never "fix" a Heisenbug with a sleep. A sleep that makes it pass converts a frequent failure into a rare one — strictly worse. Find the missing happens-before edge (lock, join, channel, await).

## Failure modes and pitfalls

- **Fixing the symptom at the detection site.** Adding `if x is None: return` at the crash line hides the invariant violation; the null still comes from somewhere and will surface elsewhere. Fix where the invariant is first broken, or at minimum fail loudly there.
- **Editing code that isn't running.** Symptoms: your prints don't appear, an obviously-fatal edit doesn't crash. Check: right branch? build actually ran? right virtualenv/container? cached `.pyc`/dist bundle? two copies of the module on `sys.path`? *Prove* your code runs by adding a deliberate crash first.
- **Declaring victory when the bug stops reproducing.** "Can't repro anymore" after random changes is not a fix. Re-apply the revert: if you can't turn the bug back ON by undoing your fix, you haven't found the cause.
- **Debugging a flaky test by rerunning until green.** Each rerun destroys the evidence. Capture the failure first: run with seed logging, record/replay, or loop-until-fail with full logs (`while ./test; do :; done`).
- **Trusting the reproduction "recipe" from the reporter.** Users report the last thing they did, not the triggering condition. Reproduce it yourself before investing in their theory of cause.
- **Anchoring on the first plausible hypothesis.** After two failed fix attempts on the same theory, the theory is wrong. Explicitly write down 3 alternative hypotheses and the cheapest experiment that discriminates between them.
- **Grepping for the error string in your repo and finding nothing → conclusion "not my code."** The string may be built with formatting/concatenation, come from a dependency, or be localized. Grep for distinctive *substrings* and for the error *class*.
- **Blaming caching without evidence.** "Clear the cache" fixes things by side effect (it also restarts, resets state, forces rebuild). Isolate: does clearing *only* the cache fix it?
- **Off-by-one blindness in reasoning about ranges.** When the bug involves boundaries, don't reason — enumerate. Write out n=0, n=1, n=2 explicitly on paper/in a scratch test. Most "logic" bugs a strong engineer ships are boundary bugs they reasoned about instead of enumerating.

## Worked micro-example: automated bisect of a flaky regression

Nightly job started failing ~30% of runs sometime in the last 2 weeks (~120 commits). Naive bisect would misclassify commits 70% of the time. Fix: amplify, then bisect.

```bash
cat > /tmp/repro.sh <<'EOF'
#!/bin/bash
make -j build || exit 125          # 125 = "skip this commit" (broken build)
for i in $(seq 1 20); do           # 20 runs: P(all pass | 30% flaky) ≈ 0.7^20 ≈ 0.08%
  timeout 60 ./job --seed "$i" || exit 1
done
exit 0
EOF
chmod +x /tmp/repro.sh
git bisect start HEAD 'HEAD@{2 weeks ago}'
git bisect run /tmp/repro.sh
```

Bisect lands on a commit that only *renames a config key*. Don't reject it — ask what it perturbs: the rename changed dict insertion order, which changed worker startup order, which widened a pre-existing race between worker 0 and the scheduler. The rename is the *trigger*; the race is the *bug*. Confirm by running the parent commit 200x under `-race` — the race detector fires there too. Fix the race; the rename stays.

## Worked micro-example: backwards dataflow from a trace

```
TypeError: unsupported operand type(s) for -: 'str' and 'datetime.timedelta'
  File "billing/invoice.py", line 88, in due_date
    return self.issued_at - GRACE
```

Don't patch line 88 with `parse(self.issued_at)`. Trace the dataflow: who sets `issued_at`? Constructor takes it verbatim → callers: one path passes `datetime.utcnow()`, another passes `row["issued_at"]` from a DB driver that returns ISO strings for TEXT columns. Root cause: the column is TEXT, not TIMESTAMP, so *every* string-path datetime in the codebase is suspect, not just this one. Correct fix: convert at the boundary (the DB row → domain object mapping), add a type assertion in the constructor (`assert isinstance(issued_at, datetime)`), and audit the other fields on the same row. The crash was one symptom of a boundary-typing bug with many symptoms.

## Verification before declaring the bug fixed

1. **Flip test:** with the fix, the repro passes; revert the fix, the repro fails again. Both directions, or you haven't established causation.
2. **Explain the whole symptom:** your root cause must account for *every* observed detail — the timing, the specific values, why it started when it started, why only some users. Unexplained residue means a second bug or a wrong theory.
3. **For flaky bugs:** N clean runs where N makes chance passage < 1% given the prior failure rate (30% flaky → ~13 runs minimum; run 50).
4. **Check the blast radius:** grep for the same pattern elsewhere — bugs come in families (same author, same copy-paste, same misunderstood API).
5. **Add the regression test before cleaning up** the instrumentation, and confirm the test fails on the pre-fix code.
