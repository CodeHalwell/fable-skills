---
name: legacy-code-navigation
description: Load when working in a large or unfamiliar codebase — locating where behavior lives, understanding undocumented code before changing it, recovering the intent behind odd code via git history, detecting dead code, untangling dependencies, or making safe changes without full understanding of the system.
---

# Legacy Code Navigation

Baseline competence assumed: grep-from-a-user-visible-string, walk up to the entry point, confirm at runtime early, `git log -S/-G/-L` + blame-chain archaeology, Chesterton's fence, characterization tests, seams-by-parameter, tombstone-then-delete for dead code, idempotency-not-timing for races. This sheet covers the corrections and anchors on top of that.

## Corrections to the default instincts

- **Read one vertical slice end to end, not ten horizontal layers.** The reflex when orienting is breadth-first structure reading (top-level dirs, then each layer). It fails because layers teach vocabulary, not behavior. One request traced route→service→DB→response teaches the house style — error handling, transaction boundaries, layering — which transfers to every other slice. Budget: the minimum map for a safe change is typically under 2% of the codebase; every reading step must answer a named question, and when it's answered, stop.
- **Read the data model before the code, when learning a subsystem.** The reflex order is tests → public interface → implementation. Better: twenty table definitions (or ORM models/protobufs) beat two hundred classes — entities, relationships, state machines (`status` columns and their values), soft-delete/versioning conventions all live there. Then the **config surface** (env vars, flags, settings): every conditional behavior worth knowing has a knob, and the knob list is short. Then **draw the state machine for the core entity** from the enum plus the writes to it — most legacy business logic is guards on state transitions, and the diagram makes 40 scattered `if status ==` checks legible. Timebox 30 minutes; if no workable model, go behavior-first (trace one real request) — some systems are only legible dynamically.
- **CI config is the entry-point document that never rots.** READMEs and wiki diagrams drift; `.github/workflows/`, `Makefile`, `docker-compose.yml`, cron schedules, and queue-consumer bindings are executable truth about how the thing builds and every way it is caused to do anything.
- **Choose data flow vs control flow deliberately.** To learn what a system *does*, follow one datum (an order, a payload) through its transformations. To find why something *broke*, follow execution backwards from the failure point. Switching mid-trace without noticing is how traces get lost — keep a written trace log (question at top, file:line per hop, one-line finding), and pop any excursion that hasn't touched the original question in 3 hops.
- **Fix where the bad value is created, not where it crashes.** The reflex is to add the null check at the crash site. The null was manufactured three layers up by a swallowed exception; the patch entombs the real bug and moves the blast somewhere subtler. Trace data flow backwards to the origin; fix there, or document explicitly why you're patching downstream.
- **Runtime confirmation within the first 10 minutes.** Legacy systems typically contain 2+ code paths that *look* like they handle your case (`OrderProcessor`, `OrderService`, `OrderManager`, `LegacyOrderHandler`); only one runs, and static reading routinely picks the dead one. The class that runs is the one referenced in the route/task/DI registration, not the one with the best name. One added log line or deliberate `raise Exception("AM I HERE")`, then trigger the behavior.

## Anchors and less-known moves

### Orientation
- Churn map: `git log --format='' --name-only --since='6 months' | sort | uniq -c | sort -rn | head -30` — the hottest files are where your change most likely belongs. Zero churn = stable bedrock *or* abandoned; suspicious either way.
- Grep strings with variable parts stripped: for `"Failed to process order 8812"` grep `"Failed to process order"`; not found → i18n key or dynamic assembly → grep fragments.

### Archaeology extras (beyond -S/-G/-L/blame basics)
- `git show <sha> --stat` — co-changed files reveal hidden coupling: the "always edit these two together" pairs.
- Commit message is just "fix"? Check adjacent commits by the same author the same day; the context commit is usually nearby. The real why is typically 2 hops in: commit → PR discussion → linked ticket/incident.
- Blame lands on a formatting commit: re-blame before it with `git blame <sha>^ -- file` and repeat down the chain (works even without `.git-blame-ignore-revs`).

### Dead code — the tier everyone skips
After import analysis, whole-repo string sweep (all file types — templates, crontabs, YAML, DB rows storing dotted paths), and telemetry over a full business cycle (month-end and quarter-end jobs exist), there is a fourth question static and runtime tools both miss: **is the *output* consumed?** A function can run monthly from a `crontab.tpl` dispatch-by-name and write to `s3://reports-bucket/legacy/` where a finance ETL reads it — alive and load-bearing, invisible to every reference-based tool. Check object timestamps/access logs on whatever the candidate writes. Delete in a dedicated commit whose message carries the evidence tier reached.

### Safe change
- Blast radius floor: whole-repo grep for the bare name as a *string* across all file types (reflection, `getattr(...)` concatenation, Celery task names, YAML pipelines, SQL in strings, templates). If the name is built by concatenation, grep fragments.
- Verify the rollback story instead of assuming it — in legacy systems "we can always roll back" is often false: data migrations already applied, caches poisoned, downstream consumers already ingesting the changed output format.
- Extract along the seam where data is already serialized (a dict between phases, a DB row, a queue message) — that boundary is proven narrow. Extracting along an object graph of live references is surgery on conjoined twins. And break tangles cheapest-first: move a shared constant/type out → invert one edge with a consumer-defined interface → callback passed at construction. A DI framework is never step one.

## Pitfalls Opus-class intuition misses

- **Clone divergence.** You fix the bug and it persists — the function was copy-pasted into four places years ago and you fixed the one that doesn't run. After locating any legacy bug, grep for a distinctive line of its *body*, not its name, to find the siblings.
- **The database contains states the code can no longer produce.** Orphaned rows, retired enum values, formats from three migrations ago. Code-only reasoning says "this branch is impossible"; the data says otherwise. Before tightening a parser or deleting a defensive branch: `SELECT status, COUNT(*) FROM orders GROUP BY 1` against production regularly ends the argument.
- **Environment drift diagnosis order.** Works locally, fails in prod → diff the *configuration surface* first (env vars, flag states, proxy/infra config), not the code — smaller search space, more often the culprit.
- **Legacy test suites assert bugs.** Tests written from observed behavior encode the bug as the spec; `@skip("flaky")` hides real races; mocks drift from real collaborators for years. A failing legacy test after your change means "behavior changed," not automatically "you're wrong" — decide which with `git log -L` on the test itself.
- **The weirdness is the behavior.** Cleanup-before-change without characterization tests is the riskiest change available: the redundant re-fetch in the loop masks a stale-cache bug, the `sleep(0.1)` bandages a race, the duplicated branch differs by one character on purpose. Order: characterize → change → refactor as its own reviewed diff. Pickaxe first — five minutes of `git log -S` regularly surfaces "added after INC-4432," saving you from re-causing INC-4432.
- **The second dispatcher.** When something runs twice or at odd times, the IDE call hierarchy shows one enqueue site; a whole-repo *string* grep for the task name finds the ops replay script / cron that also dispatches it by name. Duplicate execution in legacy systems usually means two dispatchers or at-least-once redelivery — fix with an atomic claim (`UPDATE ... SET status='SHIPPING' WHERE id=%s AND status='PAID'`, bail if 0 rows), and pickaxe the existing threshold before "tuning" it: it was probably set after an incident you'd be re-causing.
- **Single-repo tunnel vision includes the database.** Behavior findable nowhere despite thorough grep: beyond proxies/gateways/other services, check DB triggers and the feature-flag service — the two sources people forget are queryable at all.

## Self-check before presenting conclusions

- Static-only analysis is a draft — label it as such unless a log/breakpoint/telemetry confirmed the traced path actually runs.
- Every "wrong/dead/redundant" claim: archaeology done, and I can state why it was written? If not, it's a guess.
- Blast radius (all callers including string-based; all consumers of changed output) and rollback story, each stateable in two sentences.
- Production data checked for "impossible" states before tightening validation or removing defensive branches.
- Observed vs inferred kept distinct in the writeup; leave a 15-line "how a request flows" trace note so the next hour one is shorter than mine.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)
- Probed 12 claims: 8 baseline (cut/compressed), 4 partial (sharpened), 0 delta (expanded).
- Opus cold already nails: string-grep entry, early runtime confirmation, full pickaxe/blame/-L toolkit, dead-code evidence hierarchy + tombstone protocol, Chesterton's fence workflow, IDE-blind caller categories, seam-by-parameter, idempotency-over-timing fixes, ask-a-human etiquette. All compressed to a one-line "baseline assumed" preamble.
- Biggest gaps found: reads horizontal layers instead of one vertical slice, and orders subsystem reading tests/interface-first instead of data-model → config → state-machine.
- Dead-code hierarchy misses the output-consumption tier (artifact written to S3/reports consumed by an external ETL).
- Unprobed retained pitfalls (historically weak spots): clone divergence, impossible-in-code DB states, symptom-site fixes, second dispatcher via string grep.
