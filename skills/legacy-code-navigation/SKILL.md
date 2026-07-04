---
name: legacy-code-navigation
description: Load when working in a large or unfamiliar codebase — locating where behavior lives, understanding undocumented code before changing it, recovering the intent behind odd code via git history, detecting dead code, untangling dependencies, or making safe changes without full understanding of the system.
---

# Legacy Code Navigation

## Core mental model

- **You are building a mental model, not reading a book.** The goal is the minimum map that makes your specific change safe — typically under 2% of the codebase. Reading breadth-first "to understand the system" is procrastination with a virtuous feel; every reading step should answer a named question, and when the question is answered, stop reading.
- **Trace from an observable behavior, never from the directory tree.** Directory names lie (`utils/` contains business logic; `core/` is dead). A log line, an error message, a URL route, or a UI string is ground truth you can grep for — that's your entry point, always.
- **The code is the what; version control is the why.** Odd code has a reason, and the reason is in the commit that introduced it, and in the ticket/incident that commit references. Never delete or "clean up" something weird before running the archaeology — weirdness is load-bearing until proven otherwise. This is Chesterton's fence in executable form.
- **Data flow beats control flow for understanding; control flow beats data flow for debugging.** To learn what a system *does*, follow one datum (an order, a request payload) through its transformations and storage. To find why something *broke*, follow execution backwards from the failure point. Pick one deliberately; switching mid-trace without noticing is how you get lost.
- **In legacy code, the tests you add are your understanding, externalized.** A characterization test ("the system currently returns X for input Y" — asserted exactly as observed, even if X looks wrong) is worth more than an hour of reading, because it stays true after you stop looking, and it screams when your change breaks an assumption you didn't know you were making.

## Decision frameworks

### First-hour orientation in a brand-new repo
Do these in order; stop when your actual task's question takes over:
1. **Make it run** (or make its tests run) before reading anything deep — `README`, `Makefile`/`justfile`, `docker-compose.yml`, CI config (`.github/workflows/`) are the executable truth about how the thing builds and what its real entry points are. CI config never rots the way READMEs do.
2. **List the entry points:** route registrations, `main`/`manage.py`/`cmd/` dirs, cron schedules, queue consumer bindings, exported public API. This is the complete set of ways the system is caused to do anything.
3. **Churn map:** `git log --format='' --name-only --since='6 months' | sort | uniq -c | sort -rn | head -30` — the 30 hottest files are the living heart of the codebase and where your change most likely belongs.
4. **Read one vertical slice end to end** (one request from route to DB and back), not ten horizontal layers. One full slice teaches the house style — error handling, transaction boundaries, layering — which transfers to every other slice.
5. Skim the test directory names before the source: test names are the closest thing legacy code has to a requirements doc.

### Entry-point-first reading order (for "where does behavior X live?")
1. **Grep the most distinctive user-visible string:** error message, log line, button label, header name. Strip the variable parts first — for `"Failed to process order 8812"` grep `"Failed to process order"`. Not found? The string is built dynamically or lives in an i18n table: grep the i18n key, or grep distinctive *fragments* (`"Failed to process"`).
2. **Walk up from the hit** (IDE call hierarchy, or `grep -rn "function_name"`) until you reach a recognizable entry point: route table, CLI arg parser, cron/queue consumer registration, event-handler map, `main`.
3. **Walk down from that entry point** along only the branch your scenario takes. Note — don't read — everything else. Keep a written trace log: the question at top, files:lines visited, one-line finding each.
4. **Confirm with runtime evidence before trusting the static trace:** one added log line, a breakpoint, or a deliberate `raise Exception("AM I HERE")` in a dev environment, then trigger the behavior. Legacy systems typically contain 2+ code paths that *look* like they handle your case; only one runs. Static reading alone routinely picks the dead one. Budget: runtime confirmation within the first 10 minutes.

### Minimal-reading model building
Cheapest-information-first order when you need "how does this subsystem work" rather than one behavior:
- **Read the data model before the code.** Twenty table definitions (or the ORM models, or the protobuf schemas) tell you more than two hundred classes: entities, relationships, state machines (`status` columns and their values), and soft-delete/versioning conventions all live there.
- **Read the config surface next:** env vars, flag definitions, settings files. Every conditional behavior worth knowing about usually has a knob, and the knob list is short.
- **Read test *names* before test bodies, and bodies before implementation.** `test_refund_rejected_after_30_days` is a requirement statement; the suite's names are the closest thing to a spec that stays true.
- **Read each module's public interface (exports, `__init__.py`, header) and skip the internals** until a specific question forces you in.
- **Draw the state machine for the core entity** (order status, job lifecycle) from the enum + the writes to it. Most legacy business logic is guards on state transitions; the diagram makes 40 scattered `if status ==` checks legible.
- Timebox: if 30 minutes of this hasn't produced a workable model, stop and go behavior-first (trace one real request) — some systems are only legible dynamically.

### Runtime observation toolkit (when reading stalls, watch instead)
- Debugger breakpoint or targeted log line at the suspected fork in the road — one run answers what an hour of reading guesses at.
- HTTP edges: mitmproxy / `curl -v` replays; see the *actual* requests, not the ones the code appears to make.
- DB truth: enable query logging (or `EXPLAIN`-log slow queries) and trigger the behavior — the SQL stream is the system's honest diary.
- Syscall level: `strace -f -e trace=network,file` when you suspect the process touches something no code path admits to (config files, DNS, sockets).
- Live process: `py-spy dump` (Python) / `jstack` (JVM) for "what is it doing *right now*" on a wedged or slow process, no restart needed.

### Version-control archaeology toolkit
| Question | Command |
|---|---|
| When did this string/flag/constant appear or disappear? | `git log -S "exact_string" --oneline -- path/` — pickaxe finds commits that changed the *count* of occurrences |
| How did this pattern-shaped thing evolve? | `git log -G "regex"` — matches diff content; catches moves and edits that `-S` misses |
| Who wrote this line, in what commit? | `git blame -w -M -C file` (`-w` ignores whitespace, `-M`/`-C` trace moves and copies), then `git show <sha>` for the full commit and message |
| Blame lands on a formatting/refactor commit? | Re-blame before it: `git blame <sha>^ -- file`, repeat down the chain. Or use `git log -L 42,60:path/file.py` to see every change to that range in order. Check for `.git-blame-ignore-revs` and use `--ignore-revs-file` |
| Whole history of one function? | `git log -L :funcname:path/file.py` — function-scoped log |
| What changed *with* it? | `git show <sha> --stat` — co-changed files reveal hidden coupling (the "always edit these two together" pairs) |
| The why-chain | Commit message → PR number → PR discussion → linked ticket/incident. The real reason is usually 2 hops in. Commit says just "fix"? Check adjacent commits by the same author the same day |
| Where is the *live* code? | `git log --oneline --since="1 year" -- path/ | wc -l` per directory — churn maps activity. High churn = where changes happen (yours probably belongs there); zero churn = either stable bedrock or abandoned; treat with caution either way |

### Safe-change protocol (changing what you don't fully understand)
1. **Characterize first:** pin current behavior around the change site with tests asserting what it *does*, not what docs claim. If it's untestable, that's step 0: break the blocking dependency with the smallest seam — extract-and-override, pass the collaborator in as a parameter, or wrap the static/global call. Do not refactor beyond what the seam requires.
2. **Prefer additive shapes:** a new function called from one new call site beats editing a 400-line function that 30 places call. Sprout method/class: new logic in new code, one call inserted into the legacy mass.
3. **Scope discipline:** drive-by cleanups (renames, formatting, "fixing" adjacent oddities) go in separate commits or not at all. They turn a reviewable 10-line diff into a 500-line risk, and they poison `git blame` for the next archaeologist. One behavioral change per commit.
4. **Blast-radius check before merging:** enumerate every caller of what you changed, *including string-based references* — reflection, `getattr`, template files, YAML/config referencing dotted paths, SQL kept in strings, task names. IDE find-usages misses all of these; whole-repo `grep -rn "name"` across every file type is the floor, not the ceiling.
5. **Ship behind a reversible mechanism when uncertainty remains:** flag, config default, or parallel-run comparison. And verify the rollback story rather than assuming it — in legacy systems "we can always roll back" is often false (data migrations, cache poisoning, downstream consumers of changed output).

### Dead code detection — evidence hierarchy
Static "no references found" is the *weakest* evidence, not proof. In descending strength:
1. **Production telemetry:** add a counter/log to the suspect path, wait a full business cycle — month-end and year-end jobs exist — confirm zero hits. Tombstone pattern: log-and-wait beats delete-and-pray.
2. **Entry-point audit:** unreachable from any route table, cron schedule, queue binding, CLI registration, or exported symbol — verified by grep, not only by import analysis.
3. **Dynamic-reference sweep:** grep the name *as a string* everywhere — templates, YAML, DB rows storing class paths, reflection sites. Convention-based frameworks (Django signals, Rails callbacks, Celery task names, Spring beans) make import-graph tools blind.
4. **Static tools last:** `vulture` (Python), `knip`/`ts-prune` (TypeScript), compiler dead-code warnings — excellent for *generating candidates*, never sufficient for deletion.
Delete in a dedicated commit whose message carries the evidence ("dead since 2023 per pickaxe; zero telemetry hits over 35 days"), so the revert is trivial and the next archaeologist understands.

### Dependency untangling (extracting or modifying a tangled piece)
- **Map the true edges first:** what does the target import, and what imports it? `pydeps`/`import-linter` (Python), `dependency-cruiser` (JS), or plain grep. The tangle is usually 2–3 specific edges, not "everything touches everything."
- **Break edges cheapest-first:**
  1. Move a constant or type to a neutral module both sides can import.
  2. Invert one edge: define an interface where it's consumed; have the low-level side implement it.
  3. Replace a direct call with a callback/handler passed in at construction.
  Adopting a DI framework is never step one.
- **Global mutable state is the usual root tangle** (module-level singletons, thread-locals, a `settings` object imported everywhere). Don't eliminate it globally; *parameterize your slice* — accept the value as an argument defaulting to the global, so your code becomes testable while the rest of the world stays untouched:
  ```python
  def price(order, tax_table=None):
      tax_table = tax_table if tax_table is not None else global_tax.TABLE
  ```
- **Extract along the seam where data is already serialized** (a dict passed between phases, a DB row, a queue message) — that boundary is proven narrow. Extracting along an object graph full of live references is surgery on conjoined twins.

## Failure modes & pitfalls

- **Trusting names and comments over evidence.** `validate_order()` also mutates inventory; the comment says "temporary hack, remove after Q2" — of 2019. Names describe the author's intent at write time, not current behavior. Trust only: (a) what you traced at runtime, (b) what tests assert, (c) what git history documents.
- **Fixing the bug at the symptom site.** You find where the crash happens, add the null check, ship. But the null was manufactured three layers up by a swallowed exception, and your patch just entombed the real bug and moved the blast to a subtler place. Correction: trace data flow *backwards* to where the bad value was created; fix there, or document explicitly why you're patching downstream.
- **The confident wrong entry point.** The codebase has `OrderProcessor`, `OrderService`, `OrderManager`, and `LegacyOrderHandler`; you pick the best-named one and spend an hour in code that hasn't run since 2021. Correction: runtime confirmation inside 10 minutes, before deep reading; check the deploy/entry wiring — the class that runs is the one referenced in the route/task/DI registration, not the one with the best name.
- **Refactor-then-change.** "This is unreadable; I'll clean it up first, then make my change." Without characterization tests, the cleanup is the riskiest change you could possibly make, because behavior lives in the weirdness: the redundant re-fetch inside the loop is masking a stale-cache bug; the `sleep(0.1)` is a race-condition bandage; the duplicated branch differs by one character on purpose. Correction: characterize → make the change → *then* refactor if still worthwhile, as its own reviewed diff.
- **Deleting the fence before asking why it's there.** A duplicate-looking null check, an `except Exception: pass`, an off-by-one that "must be a bug." Run `git log -S` first — five minutes of pickaxe regularly surfaces "added after incident INC-4432," saving you from re-causing INC-4432. If archaeology finds nothing, treat it as untested behavior: remove behind a flag or with telemetry, never blind.
- **Grep-blindness to dynamic dispatch.** You changed a method signature; the IDE said 3 callers; production disagrees, because the callers include a Celery task invoked by string name, a `getattr(obj, action + "_handler")`, and a YAML pipeline definition. Correction: whole-repo string grep for the bare name across *all* file types, plus fragments if the name is built by concatenation.
- **Reading breadth-first until context evaporates.** Twenty files deep, you no longer remember the question. Correction: keep the written trace log, and enforce a stack discipline — any excursion that hasn't touched the original question in 3 hops gets popped. Write the map as you go (a 15-line "how a request flows" note); that note is the deliverable that makes hour two faster than hour one.
- **Assuming the tests describe intended behavior.** Legacy suites contain tests that assert bugs (written from observed behavior), tests disabled with `@skip("flaky")` hiding real races, and mocks that drifted from the real collaborator years ago. A failing legacy test after your change means "behavior changed," not automatically "you're wrong" — decide which, using `git log -L` on the test itself.
- **Modernizing the stack as a side quest.** Upgrading the framework "while you're in there" mixes an unbounded-risk change into your bounded one; when production breaks, you can't tell which change did it. Version upgrades in legacy systems are their own project with their own parallel-run plan.
- **Believing the architecture diagram.** The wiki diagram is from 2020; three services have been added, one merged, and the "deprecated" queue carries 40% of traffic. Diagrams are hypotheses. Verify against deploy configs, live route tables, and traffic metrics before routing your change through the picture.
- **Single-repo tunnel vision.** The behavior you're hunting doesn't exist in this repo at all — it's in a sidecar, an nginx rewrite rule, a database trigger, a feature-flag service, or the *other* consumer of the same queue. If a thorough grep finds no plausible source for an observed behavior, widen the system boundary before doubting the observation.
- **Clone divergence.** You find the bug, fix it, and it persists — because the function was copy-pasted into four places years ago and you fixed the one that doesn't run (or only one of three that do). After locating any bug in legacy code, grep for a distinctive line of its *body*, not its name, to find the siblings.
- **Environment drift.** It works locally and fails in prod because prod has different config, feature flags, data volume, or a proxy in front. When behavior differs by environment, diff the *configuration surface* first (env vars, flag states, infra config) — it's a smaller search space than the code and the culprit more often.
- **Treating data as code's junior partner.** In old systems the database contains states the current code can no longer produce — orphaned rows, retired enum values, formats from three migrations ago. Code-only reasoning says "this branch is impossible"; the data says otherwise. Before removing a defensive branch or tightening a parser, query production for the "impossible" values: `SELECT status, COUNT(*) FROM orders GROUP BY 1` regularly ends arguments.
- **Asking no one.** An hour of archaeology can be thirty seconds of asking the person `git blame` names — if they're still around, a short, specific question ("this 30s threshold in the replayer — peak-load thing?") with your evidence attached gets context no tool has: the constraint that was political, the migration that was abandoned halfway. Do the archaeology first so the question is sharp; then actually ask.

## Worked micro-examples

### 1. "Is this giant function safe to delete?" — dead-code walkthrough
Candidate: `export_legacy_report()` in `reports/exports.py`, 300 lines, last meaningful edit 4 years ago per `git log -- reports/exports.py`.
1. Import analysis: `grep -rn "export_legacy_report" --include='*.py'` → only the definition and one commented-out call. Weak evidence — keep going.
2. String sweep across *all* file types: `grep -rn "export_legacy_report" .` → a hit in `crontab.tpl`: `0 2 1 * * run_task export_legacy_report`. It runs monthly at 2am, dispatched by name. The import-graph verdict was wrong.
3. Is the output consumed? The function writes `s3://reports-bucket/legacy/`. Bucket access logs / object timestamps show objects written monthly and *read* by an IP belonging to the finance ETL. Not dead — load-bearing, invisible to every static tool.
4. Counterfactual world where step 3 showed no reads: still don't delete yet — add a tombstone log, wait one full business cycle (month-end *and* quarter-end), then delete cron entry and function in one commit whose message carries the evidence.
Lesson: each escalation (imports → strings → runtime/consumption evidence) overturned the previous verdict. Deletion decisions get made at the strongest evidence tier you can afford, never the weakest.

### 2. "Orders sometimes get double-shipped — find why"

1. **Entry via string.** `grep -rn "shipment created"` → one hit, a log line in `fulfillment/tasks.py:ship_order()`. Runtime ground truth: production logs show the message *twice* for order 88712, 40 seconds apart. So `ship_order` executes twice — fact, not theory.
2. **Walk up: who invokes it?** Call hierarchy shows a Celery task registration and one enqueue site in `checkout/complete.py`. But a whole-repo grep for the *string* `"ship_order"` also finds `ops/replay_stuck_orders.py` — a cron that re-enqueues any order stuck in `PAID` for more than 30s. Second dispatcher found; the IDE alone would have missed it (task invoked by name).
3. **Why does a normal order look "stuck"?** Trace the datum — order status. `ship_order` sets `SHIPPED` at the *end*; the task takes ~45s (slow carrier API). Window: 30s cron threshold < 45s task duration, so the replayer fires while the first run is still in flight. The 40s gap in the logs matches. Race explained.
4. **Why is the threshold 30s?** Archaeology: `git log -S "STUCK_THRESHOLD"` → a commit from two years ago: "reduce to 30s after INC-2201 (orders stuck at peak)." The fence is load-bearing — raising the threshold would re-expose INC-2201's failure mode at peak traffic.
5. **Safe change: make `ship_order` idempotent instead of tuning the timer.** Atomic claim at the top:
   ```python
   rows = db.execute(
       "UPDATE orders SET status='SHIPPING' WHERE id=%s AND status='PAID'", [oid])
   if rows == 0:
       return  # another worker owns this order
   ```
   Characterization test first: enqueue the same order twice, assert exactly one carrier call (mock at the HTTP seam). Threshold untouched; INC-2201 protection intact; the diff is ~15 lines, additive in shape.

The expert signature: string-grep found the second dispatcher, data-flow tracing found the race window, pickaxe stopped a "fix" that would have re-caused an old incident, and the final change was idempotency-shaped rather than a timing tweak (timing tweaks move races; they don't remove them).

## Self-check before presenting conclusions about legacy code

- Did I confirm the code I analyzed actually *runs* for this scenario (log, breakpoint, telemetry) — or is the whole analysis static? Static-only conclusions about legacy systems are drafts, and should be labeled as such.
- For every "this code is wrong / dead / redundant" claim: did I run the archaeology (`git log -S/-G`, blame chain to the ticket)? Can I state why it was written? If I can't, the claim is a guess.
- Did I grep the entire repo — all file types — for string-based references before claiming I've found all callers or that code is dead?
- Is the change additive/seam-based, with characterization tests pinned around the site, and is the diff free of drive-by edits?
- Can I state the blast radius (every caller, every consumer of changed data or output format) and the rollback story, each in two sentences? If not, the change isn't ready to ship.
- Did I check production data for states my code-level reasoning declared impossible, before tightening any validation or deleting any defensive branch?
- Does each conclusion distinguish what I *observed* (logs, runtime traces, data) from what I *inferred* (static reading)? Presenting inference as observation is how wrong mental models propagate to the next person.
- Did I leave the map better than I found it — the trace note, the ADR-style comment, the commit message with the why — so the next person's hour one is shorter than mine?
