---
name: legacy-code-navigation
description: Load when working in a large or unfamiliar codebase — locating where behavior lives, understanding undocumented code before changing it, recovering the intent behind odd code via git history, detecting dead code, untangling dependencies, or making safe changes without full understanding of the system.
---

# Legacy Code Navigation

## Core mental model

- **You are building a mental model, not reading a book.** The goal is the minimum map that makes your specific change safe — typically <2% of the codebase. Reading breadth-first "to understand the system" is procrastination with a virtuous feel; every reading step should answer a named question.
- **Trace from an observable behavior, never from the directory tree.** Directory names lie (`utils/` contains business logic, `core/` is dead). A log line, an error message, a URL route, or a UI string is ground truth you can grep for; that's your entry point, always.
- **The code is the what; version control is the why.** Odd code has a reason, and the reason is in the commit that introduced it (and the incident/ticket that commit references). Never delete or "clean up" something weird before running the archaeology — weirdness is load-bearing until proven otherwise (Chesterton's fence, executable form).
- **Data flow beats control flow for understanding; control flow beats data flow for debugging.** To learn what a system *does*, follow a datum (an order, a request payload) through its transformations and storage. To find why something *broke*, follow execution from the failure point backwards. Pick deliberately; mixing them mid-trace is how you get lost.
- **In legacy code, tests-you-add are your understanding, externalized.** A characterization test ("the system currently does X for input Y" — assert whatever it actually returns, even if it looks wrong) is worth more than an hour of reading, because it keeps being true after you stop looking.

## Decision frameworks

### Entry-point-first reading order (for "where does behavior X live?")
1. Grep for the most distinctive user-visible string: error message, log line, button label, header name. Strip variable parts first — `"Failed to process order %s"` means grep `"Failed to process order"`. If not found, the string may be built dynamically or come from an i18n table: grep the i18n key, or grep for distinctive *fragments*.
2. From the hit, walk *up* (find callers: IDE call hierarchy, or `grep -rn "function_name"`) until you reach a recognizable entry point: route table, CLI arg parser, cron/queue consumer registration, event handler map.
3. Now walk *down* from that entry point along only the branch your scenario takes. Note — don't read — everything else.
4. Confirm with runtime evidence before trusting the static trace: add one log line / breakpoint / `raise Exception("AM I HERE")` and trigger the behavior. In legacy systems there are typically 2+ code paths that look like they handle your case; only one actually runs. Static reading alone routinely picks the dead one.

### Version-control archaeology toolkit
| Question | Command |
|---|---|
| When did this line/flag/constant appear or disappear? | `git log -S "the_exact_string" --oneline -- path/` (pickaxe: finds commits changing the *count* of occurrences) |
| How has this regex-shaped thing evolved? | `git log -G "pattern"` (matches diff content, catches moves that -S misses) |
| Who wrote this and in what commit? | `git blame -w -M -C file` (`-w` ignores whitespace, `-C` traces through copies) — then `git show <sha>` for the full commit + message |
| Blame shows a useless refactor/format commit? | Re-blame before it: `git blame <sha>^ -- file`, repeat. Or `git log -L 42,60:path/file.py` to see every change to that line range in order. Check for a `.git-blame-ignore-revs` file: `git blame --ignore-revs-file` |
| What was this function's whole history? | `git log -L :funcname:path/file.py` (function-scoped log; works when git can find the function heading) |
| What else changed with it? | `git show <sha> --stat` — co-changed files reveal hidden coupling (the "always edit these two together" pairs) |
| Why-chain | commit message → PR number → PR discussion → linked ticket/incident. The real reason is usually 2 hops in. A commit saying "fix" with no context: check adjacent commits by the same author the same day |

- Frequency archaeology: `git log --oneline --since="1 year" -- path/ | wc -l` per directory tells you where the *live* code is. High-churn files are where changes happen (and where yours probably belongs); zero-churn subtrees are candidates for "nobody understands this anymore" caution or dead code.

### Safe-change protocol (change without full understanding)
1. **Characterize first:** pin current behavior around the change site with tests that assert what it *does* (not what docs say). If untestable, that's step 0: break the hard dependency with the smallest seam — extract-and-override, inject the collaborator, or wrap the static call — the minimal set of legacy seam techniques; don't refactor beyond what the seam requires.
2. **Prefer additive shapes:** new function called from one new call site > editing a 400-line function 30 places call. Sprout method/class: put new logic in new code, insert one call into the legacy mass.
3. **Scope discipline:** in legacy code, drive-by cleanups (renames, formatting, "fixing" adjacent oddities) go in separate commits or not at all — they turn a reviewable 10-line diff into a 500-line risk, and they invalidate `git blame` for the next archaeologist. One behavioral change per commit.
4. **Blast-radius check before merging:** grep every caller of what you changed, including string-based references (reflection, `getattr`, template files, config/YAML referencing dotted paths, SQL in strings). IDE find-usages misses all of these; `grep -rn "name"` across the whole repo (including non-code files) is the floor.
5. **Ship behind a reversible mechanism** when uncertainty remains: flag, config default, or parallel-run comparison. In legacy systems "we can always roll back" is often false (data migrations, cache poisoning) — verify the rollback story, don't assume it.

### Dead code detection — evidence hierarchy
Static "no references" is the *weakest* evidence. In order of strength:
1. **Production telemetry:** add a counter/log to the suspect path, wait a full business cycle (month-end, year-end jobs exist!), confirm zero hits. Tombstone pattern: log-and-wait beats delete-and-pray.
2. **Route/entry audit:** unreachable from any route table, cron, queue binding, CLI command, or exported symbol — checked by grep, not just by import analysis.
3. **Dynamic-reference sweep:** grep for the name as a *string* everywhere (templates, YAML, DB rows storing class paths, reflection). Frameworks with convention-based dispatch (Django signals, Rails callbacks, Spring beans by name) make import-graph tools blind.
4. **Static tools last:** `vulture` (Python), `knip`/`ts-prune` (TS), compiler dead-code warnings — great for generating candidates, never sufficient for deletion.
Delete in a dedicated commit titled with the evidence ("dead since 2023 per pickaxe; zero hits over 35 days of telemetry"), so reverting is trivial and the next archaeologist understands.

### Dependency untangling (when you must extract/modify a tangled piece)
- Map the true dependency direction first: what does the target import, and what imports it? `pydeps`/`import-linter` (Python), `depcruise` (JS), or just grep. The tangle is usually 2–3 specific edges, not "everything."
- Break edges cheapest-first: (1) move a constant/type to a neutral module, (2) invert an edge by introducing an interface the low-level side implements, (3) replace a direct call with a callback/event passed in at construction. Full DI framework adoption is never step one.
- Global mutable state (module-level singletons, thread-locals, `settings` imported everywhere) is the usual root tangle. Don't try to eliminate it globally; *parameterize your slice* — accept the value as an argument, defaulting to the global, so your code is testable while the rest of the world is unchanged: `def price(order, tax_table=None): tax_table = tax_table or global_tax.TABLE`.

## Failure modes & pitfalls

- **Trusting names and comments over evidence.** `validate_order()` also mutates inventory; the comment says "temporary hack, remove after Q2" — of 2019. In legacy code, names describe the author's intent at write time, not current behavior. Correction: trust only (a) what you traced at runtime, (b) what tests assert, (c) what git history documents.
- **Fixing the bug at the symptom site.** You find the null check that would prevent the crash and add it — but the null was manufactured 3 layers up by a swallowed exception, and your patch entombs the real bug. Correction: trace data flow *backwards* to where the bad value was created; fix there or explicitly document why you're patching downstream.
- **The confident wrong entry point.** Codebase has `OrderProcessor`, `OrderService`, `OrderManager`, and `LegacyOrderHandler`; you pick the well-named one and spend an hour in code that hasn't run since 2021. Correction: runtime confirmation (log line, breakpoint) within the first 10 minutes, before deep reading. Also check deploy config — the class that runs is the one referenced in the entry-point wiring, not the best-named one.
- **Refactor-then-change.** "This is unreadable; I'll clean it first, then make my change." Without characterization tests, the cleanup is the highest-risk change you could make — behavior lives in the weirdness (that re-fetch inside the loop is masking a stale-cache bug; that `sleep(0.1)` is a race-condition bandage). Correction: characterize → change → *then* refactor if still worth it, as a separate reviewed diff.
- **Deleting the fence before asking why it's there.** Duplicate-looking null check, a `except Exception: pass`, an off-by-one that "must be a bug." Run `git log -S` first — five minutes of pickaxe regularly surfaces "added after incident INC-4432" and saves you re-causing INC-4432. If archaeology finds nothing, treat it as untested behavior: remove behind a flag or with telemetry, not blind.
- **Grep-blindness to dynamic dispatch.** You changed a method signature; IDE said 3 callers; production says otherwise, because callers include a Celery task name in a string, a `getattr(obj, action + "_handler")`, and a YAML pipeline definition. Correction: whole-repo string grep for the bare name (all file types), plus grep for the name with its underscored fragments if it's built by concatenation.
- **Reading breadth-first until context evaporates.** Twenty files deep, you no longer remember the question. Correction: keep a written trace log — question at top, files/lines visited with one-line findings — and a hard rule: any excursion that hasn't touched the question in 3 hops gets popped off the stack. Write the map as you go (a 15-line "how a request flows" note); it's the deliverable that makes hour two faster than hour one.
- **Assuming the tests describe intended behavior.** Legacy test suites contain tests that assert bugs (written from observed behavior), tests disabled with `@skip("flaky")` hiding real races, and mocks that no longer match the real collaborator. Correction: a failing legacy test means "behavior changed," not "your change is wrong" — decide which, using history (`git log -L` on the test).
- **Modernizing the stack as a side quest.** Upgrading the framework/lib version "while you're in there" mixes an unbounded-risk change with your bounded one. Version upgrades in legacy code are their own project with their own parallel-run plan.

## Worked micro-example: "orders sometimes get double-shipped — find why"

1. Entry point via string: `grep -rn "shipment created"` → hit in `fulfillment/tasks.py:ship_order()` log line. Confirm with runtime evidence: recent prod logs show the message twice for order 88712, 40s apart. Ground truth: `ship_order` executes twice.
2. Walk up: who invokes `ship_order`? Grep shows a Celery task registration and one enqueue site in `checkout/complete.py`. But grep for the *string* `"ship_order"` also finds `ops/replay_stuck_orders.py` — a cron that re-enqueues orders stuck in `PAID` for >30s. Second dispatcher found; IDE call-hierarchy alone would have missed it (task invoked by name).
3. Why does a normal order look "stuck"? Trace the datum: order status. `ship_order` sets `SHIPPED` at the *end*; the task takes 45s (carrier API). Window: 30s cron threshold < 45s task duration → replay fires during the first run. Race confirmed by the 40s gap in the logs.
4. Why is the threshold 30s? Archaeology: `git log -S "STUCK_THRESHOLD"` → commit 2 years ago, message: "reduce to 30s after INC-2201 (orders stuck at peak)". So the fence is load-bearing: don't just raise the threshold — peak-time stuck orders were a real incident.
5. Safe change: make `ship_order` idempotent instead — atomic compare-and-set `UPDATE orders SET status='SHIPPING' WHERE id=%s AND status='PAID'` gate at the top (`rowcount == 0` → another worker owns it, return). Characterization test first: enqueue same order twice, assert exactly one carrier call (mock at the HTTP seam). Threshold untouched; INC-2201 protection intact; diff is ~15 lines.

The expert signature: string-grep found the second dispatcher, data-flow tracing found the race window, pickaxe stopped a "fix" that would have re-caused an old incident, and the final change was additive and idempotency-shaped rather than a tuning tweak.

## Self-check before presenting conclusions about legacy code

- Did I confirm the code I analyzed actually *runs* for this scenario (log/breakpoint/telemetry), or is my whole analysis static? Static-only conclusions about legacy systems are drafts.
- For any "this code is wrong/dead/redundant" claim: did I run the archaeology (`git log -S/-G`, blame chain to the ticket)? Can I state why it was written?
- Did I grep the whole repo for string-based references (configs, templates, reflection) before claiming I've found all callers?
- Is my change additive/seam-based with characterization tests pinned around it, and is the diff free of drive-by edits?
- Can I state the blast radius (every caller, every consumer of changed data) and the rollback story in two sentences each? If not, the change isn't ready.
