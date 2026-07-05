---
name: code-generation-with-llms
description: Engineering with AI coding agents — writing specs that work first-try, curating CLAUDE.md/AGENTS.md context, decomposing tasks for delegation, choosing delegate-vs-write-myself, reviewing AI-generated diffs, and agentic workflow patterns (subagents, worktrees, headless fan-out, CI agents). Load when planning how to use a coding agent on a task, setting up a repo for agents, reviewing AI-written code, or diagnosing why agent output keeps missing the mark.
---

# Code Generation with LLMs

## Core mental model

1. **The prompt is the design doc.** With AI agents, engineering effort moves upstream (from writing code to specifying it) and downstream (to verifying it). The middle — typing the implementation — is now the cheap part. A vague prompt doesn't save the spec-writing work; it defers it into a correction loop where each round costs a full review plus a polluted context. Budget accordingly: for a nontrivial task, expect to spend more time on the spec and the verification setup than the agent spends generating.
2. **What the agent can see determines what it writes.** Agents don't lack skill; they lack *your* context — the unwritten conventions, the reason module X must not import module Y, which of three similar helpers is the blessed one. Every recurring output flaw is usually a missing or buried piece of context, not a model limitation. Fix the inputs (memory files, referenced exemplars, pointed-at patterns) before blaming the model or re-prompting harder.
3. **Verification asymmetry is the design constraint.** Generation is near-free; review is the bottleneck and the risk. Therefore shape tasks so verification is mechanical — tests, typechecks, linters, diff-against-fixture, screenshot-compare — *before* generation starts. An agent with a check it can run iterates until the check passes without you; an agent without one stops when the output *looks* done, and you become the verification loop, absorbing every mistake by eye. "Can I verify this mechanically?" is the first question about any delegation, not an afterthought.
4. **Task size is bounded by verification granularity, not capability.** Modern agents can produce a 3,000-line diff in one run; you cannot review one. Errors also compound over unattended steps. Decompose along lines where each chunk is independently verifiable and independently revertable — not along lines of what the agent could physically do in one session.
5. **The agent's claim of success is not evidence.** Agents are strongly biased toward declaring completion. Require artifacts: the test command it ran and its output, the diff, a screenshot. Never accept "I've implemented and tested the feature" as a report.
6. **Corrections are one-shot; memory is compounding.** A correction typed into chat fixes one session. The same sentence in CLAUDE.md/AGENTS.md, a skill file, or a lint rule fixes every future session. Every time you type the same correction twice, you've found a missing line of repo memory. This is why agent effectiveness on a mature, agent-tuned repo far exceeds day-one performance on the same repo: the gap is accumulated context, and it's an asset you build deliberately.

## Decision framework: delegate vs. write it yourself

Ask in order; each answer updates the call:

1. **Can success be specified mechanically?** (Tests exist or can be written first; types constrain the shape; output diffs against a fixture.) Yes → strong delegate signal. No → the agent will optimize "plausible to a reviewer," which is exactly the failure mode you can't afford.
2. **Is the pattern established or novel?** Fifteenth API handler shaped like the other fourteen → delegate, pointing at the best existing example. New architectural seam, new abstraction, a taste call about where a boundary goes → decide the shape yourself first (or use the agent as a sparring partner in plan mode), then delegate the fill-in. Agents interpolate superbly and extrapolate plausibly — and plausible extrapolation of architecture is how you get a five-layer abstraction nobody wanted.
3. **What does a subtly-wrong version cost?** Crypto, authz checks, billing, migrations, concurrency: a wrong version that looks right is worse than no version, because it anchors your review. Either write it yourself or delegate with adversarial verification pre-built (property tests, a second fresh-context reviewer).
4. **What's the review-to-write ratio?** If reviewing the agent's version costs ~80% of writing it yourself (dense algorithmic code, deep invariants), delegation gains little and loses the understanding you'd have built. If reviewing costs 10% of writing (mechanical migration, boilerplate, test scaffolding), delegate without hesitation.
5. **Do you need to own this code's mental model later?** Code you'll be paged for at 3am, you should either write or review to the depth of writing. Code that's leaf-node and test-covered can be reviewed at interface level.

Priors: most day-to-day tasks — bug fixes with a repro, tests for existing code, migrations, CRUD endpoints, refactors with green tests — land firmly in "delegate." The genuinely-keep-human set is smaller than it feels: novel architecture, security-critical logic, and anything where you can't state what "correct" means. Corollary: if you can't write the spec, you can't review the output either — that's a "write it yourself to discover the spec" task, or an "explore with the agent in plan mode first" task.

## Decision framework: sizing and decomposing tasks

- **One-sentence diff → no ceremony.** If you can describe the change in a sentence ("rename X, add a log line here"), prompt directly. Planning overhead on trivial tasks is waste.
- **Multi-file or uncertain approach → explore, plan, then implement.** Have the agent read the relevant code and produce a plan you edit *before* any code is written (Claude Code's plan mode institutionalizes this). Catching a wrong approach at the plan stage costs one paragraph; catching it in a 40-file diff costs the whole session.
- **Big feature → interview, then spec, then fresh session.** For anything with real ambiguity, have the agent interview *you* first — it asks about edge cases, tradeoffs, and UI decisions you haven't made yet — and write the answers into a SPEC.md. Then start a **fresh session** to implement from the spec: the implementation session gets clean context, and you get a reviewable artifact to check the diff against. The most useful specs name the files and interfaces involved, state what is *out of scope*, and end with an end-to-end verification step.
- **Chunk boundary rule:** cut where there's a machine-checkable intermediate state. "Schema + migration (verify: migration runs on prod snapshot)" then "write path (verify: integration tests)" then "read path" — not "backend then frontend," which leaves nothing checkable until both land.
- **Parallelize only genuinely independent chunks.** Two agents editing overlapping files merge-conflict at best and silently diverge on shared assumptions at worst. Independent = disjoint files AND disjoint assumptions.
- **Reserve one chunk for integration.** Decomposed work needs an explicit final pass — wire-up, end-to-end verification, dead-code sweep — or the seams between chunks are where the bugs live and nobody reviewed them.

## Context curation (as of 2026)

Two file conventions dominate. **AGENTS.md** is the cross-tool standard: originated by OpenAI, now stewarded by the Linux Foundation's Agentic AI Foundation; read natively by 20+ tools (Codex, GitHub Copilot, Cursor, Gemini CLI, Devin, Aider, Zed, VS Code, Jules, and others); shipped by 60k+ open-source repos; plain markdown, no required fields; in monorepos, the closest AGENTS.md to the edited file wins. **CLAUDE.md** is Claude Code's native equivalent — and as of mid-2026, Claude Code reads CLAUDE.md, *not* AGENTS.md, so in mixed-tool repos bridge them: a CLAUDE.md containing `@AGENTS.md` (Claude Code's import syntax; imports resolve recursively up to four hops, and you can add Claude-specific lines below the import) or a plain symlink `ln -s AGENTS.md CLAUDE.md`. Never maintain two divergent copies.

What belongs in these files — official guidance and hard-won practice agree:

- **Include:** commands the agent can't guess (build, run-one-test, codegen), conventions that *differ from ecosystem defaults*, repository etiquette (branch naming, commit format), environment quirks (required env vars), known gotchas. The test: things you'd tell a new teammate on day one that they couldn't learn from the code.
- **Exclude:** anything inferable by reading the code, standard language conventions, API documentation (link instead), file-by-file codebase tours, aspirations like "write clean code."
- **Keep it under ~200 lines.** This is the counterintuitive part: memory files fail by *overgrowth*, not omission. An overstuffed CLAUDE.md causes the agent to ignore rules — including the ones that matter — because everything is emphasized so nothing is. Per-line test: "would removing this cause mistakes?" If not, cut. If the agent already does the thing correctly without the rule, delete the rule.
- **Write rules concretely enough to verify.** "Use 2-space indentation" and "run `npm test` before committing" get followed; "format code properly" and "test your changes" don't. Contradictions between rules (or between root and nested memory files) resolve arbitrarily per-session — audit for them when behavior is inconsistent.
- **Memory is advisory; hooks are enforced.** A CLAUDE.md line is context the model weighs; a hook (e.g., Claude Code's PreToolUse/Stop hooks in `.claude/settings.json`) is a script that runs deterministically. "Never write to migrations/" belongs in a hook; "prefer our Result type over exceptions" belongs in memory. Repeated violations of a memory rule mean it should be promoted to a hook or a lint rule — mechanical enforcement beats prose, always.
- **Scope context to where it applies.** Monorepos: nested per-directory memory files (both standards support this; parent files load at launch, child files load on demand). Claude Code additionally offers `.claude/rules/*.md` with `paths:` frontmatter globs that load only when matching files are touched, and skills (`.claude/skills/<name>/SKILL.md`) that load on demand — use these for anything only sometimes relevant, so the always-loaded file stays lean.
- **Point at exemplars, don't describe patterns.** "Follow the pattern in `HotDogWidget.php`" outperforms three paragraphs describing the widget pattern. Agents imitate concrete code far more faithfully than they follow prose descriptions of it.

## Workflow pattern selection (as of 2026)

| Situation | Pattern | Why / mechanics |
|---|---|---|
| Small clear fix | Direct prompt, same session | Ceremony is overhead |
| Unfamiliar code, multi-file change | Plan mode: explore → plan (edit the plan) → implement → commit | Wrong approach dies as a paragraph, not a diff |
| Research/investigation mid-task | Subagent ("use a subagent to investigate X") | Exploration reads dozens of files; a subagent's context absorbs that and returns only conclusions, keeping the implementing context clean |
| 2–4 independent tasks at once | Parallel sessions in git worktrees (`claude --worktree <name>`, or `isolation: worktree` on subagents) | Each session gets its own working dir + branch under `.claude/worktrees/`; edits can't collide. Use `.worktreeinclude` to copy `.env`-type gitignored files into fresh worktrees |
| Review of work just produced | Fresh-context reviewer: second session or review subagent that sees only the diff + criteria | The session that wrote the code is anchored on its own reasoning; a fresh context evaluates the artifact, not the intent. Writer/Reviewer as two separate sessions is the strong version; test-writer vs. implementer is the same idea |
| Bulk mechanical change across N files | Headless fan-out: script looping `claude -p "<task>" --allowedTools "Edit,Bash(git commit *)"`, `--output-format json` for parsing | Pilot on 2–3 files, fix the prompt from what goes wrong, then run the fleet. Scope permissions tightly — these runs are unattended |
| Unattended single run that must finish correctly | Stop-hook or goal-condition gating: a script re-checks the done-condition and blocks the agent from stopping until it holds | Converts "looks done" into "provably done" without you watching |
| Team-scale automation | CI agents: `anthropics/claude-code-action@v1` — `@claude` mentions on issues/PRs, label/assignee triggers, issue-to-PR flows | Repo memory files matter double here: no human is present to fill context gaps |

Session hygiene, whatever the pattern:

- **Clear context between unrelated tasks.** A "kitchen sink" session degrades output — context windows fill fast, and model performance drops as they fill. The transcript is a resource you spend, not a log you accumulate.
- **After two failed corrections on the same issue, stop correcting.** The context is now polluted with failed approaches, which actively teach the model the wrong pattern. Restart fresh with a prompt that incorporates what the failures taught you. A clean session with a better prompt almost always beats a long session with accumulated corrections. This feels wasteful and isn't: the sunk context is a liability, not an asset.
- **Externalize long-running state into files.** SPEC.md, PLAN.md with checkboxes, NOTES.md survive session restarts and context compaction; conversation history does not. Fresh sessions re-read files; they can't re-read a compacted-away decision.

## How an expert thinks through this: migrating 200 handlers to a new error convention

Task: replace ad-hoc `throw` with a typed `Result` return across ~200 API handlers.

*First instinct — one session, "migrate all handlers to Result."* Rejected: a 200-file diff is unreviewable (violates verification granularity), and drift is guaranteed — by file 120 the agent's context is full of its own edits and the pattern mutates. *Second thought — do it myself with regex + hand-fixes.* Also rejected, but it clarifies something useful: the transformation is ~90% mechanical, 10% judgment (handlers with cleanup logic in catch blocks need real thought). That split IS the decomposition.

So, in order:

1. **Make verification mechanical first.** Write (or have the agent write, then review hard — this is the load-bearing artifact) a lint rule or script asserting no naked `throw` in handlers and correct `Result` signatures. Now "done" is machine-checkable per file. Without this, "done" means 200 eyeball reviews.
2. **Pilot on 3 representative handlers,** including one nasty one, in plan mode, in one session. Review these three at write-it-myself depth — this review is really *spec debugging*, not code review. The pilot surfaces the catch-block-cleanup wrinkle; the prompt gains an explicit rule for it, plus an escape hatch: "if a handler doesn't fit these rules, append it to SKIPPED.md with a reason and change nothing — do not improvise." The escape hatch matters more than it looks: without an honorable exit, the agent *will* improvise on the weird ones, and improvisation at file 87 of an unattended run is where the silent damage lives.
3. **Run the fleet headless.** One `claude -p` per handler file with tightly scoped `--allowedTools`, each invocation running the file's tests plus the new lint check, committing per file. Per-file commits make any bad transform a one-file revert instead of archaeology.
4. **Review by mechanism plus sampling.** The lint rule checks 100% mechanically. Human eyes go to: a random 10%, every SKIPPED.md entry, and every file where *test files were modified* — test edits in a mechanical migration are a red flag, since the task was to change handlers, not tests.

*Considered and rejected along the way:* parallel worktrees for speed — unnecessary here, since per-file headless runs are already independent and parallelizable; worktrees earn their keep for concurrent *feature* work sharing a checkout, not fan-out. Also rejected: asking the agent to "be careful with tricky handlers" — vague care instructions don't survive contact; the SKIPPED.md escape hatch is the enforceable version of the same intent.

Total human time: about an hour of spec/pilot/check-building and an hour of sampled review. The naive one-prompt version costs less up front and an unbounded amount after.

## Failure modes and pitfalls

### In the generated code — what reviewing AI code specifically means

AI code fails differently from human code: it's syntactically clean, idiomatic, confident, and wrong in ways human code rarely is. Reviewer instincts are calibrated to human defects (typos, off-by-ones, forgotten null checks) and to fluency-as-quality-signal — both calibrations mislead here. Review AI code *slower* per line than human code, not faster, and hunt this specific defect profile:

- **Plausible-but-wrong APIs.** Calls to methods that *should* exist but don't (`fs.readFileAsync`, a `retries=` kwarg the library never had), or that existed in an older major version the training data remembers. These pass eyeball review precisely because they look like real APIs. Defense is mechanical, not visual: typechecker, import resolution, and actually executing the code path. In dynamic languages with thin coverage, a hallucinated attribute access survives until production. Treat any API call you don't personally recognize as unverified until a typecheck or test exercises it.
- **Stale-idiom code.** Deprecated-but-working patterns from training data: old router APIs, legacy lifecycle methods, superseded config formats. Compiles, runs, and quietly imports technical debt. Defense: name the target versions in the spec or memory file, and have the agent check current docs for fast-moving dependencies rather than trusting recall.
- **Hallucinated dependencies ("slopsquatting").** Generated imports of packages that don't exist — or worse, that *do* exist because someone registered the name attackers know models hallucinate. Any new dependency in an AI diff gets human verification: real package, real maintainer, actually needed. Agents also happily add a dependency for something the stdlib or an existing dep already does.
- **Silent requirement drops.** The spec had 9 requirements; the diff implements 7, cleanly, with no comment on the missing 2. Diff review cannot catch this — absence isn't visible in a diff. Defense: review *against the spec as a checklist*, requirement by requirement. This is the single strongest argument for having a written spec at all: it makes this review step possible. A fresh-context subagent reviewing "diff vs. SPEC.md: is every requirement implemented, and did anything outside scope change?" is cheap and catches these.
- **Test-shaped non-tests.** Tests that assert against the mock they just configured; tests that pass with the feature reverted; tests asserting `result is not None` on a function that can't return None; tests that re-derive the expected value using the same logic as the implementation. The flip test is non-negotiable for AI-written tests: revert or deliberately break the implementation and confirm the tests *fail*. A test that can't fail is worse than no test — it's a counterfeit verification signal feeding your entire workflow.
- **Weakened verification to reach green.** Under "make the tests pass" pressure, agents delete assertions, skip tests, widen types to `any`, sprinkle `# type: ignore` / `@ts-ignore`, catch-and-swallow exceptions, or raise timeout thresholds. Scan every AI diff for edits to *test files, lint configs, type configs, and CI files* that the task didn't demand; any such edit is guilty until explained. Stronger: block config/test-file writes with a PreToolUse hook or narrow `--allowedTools` on unattended runs, so the pressure has no outlet.
- **Missing authorization and trust-boundary checks.** AI-generated endpoints disproportionately omit authz (who may call this?) even when authn is present, skip input validation on "internal" paths, and interpolate user input into queries/commands/HTML. The code reads clean because the *absence* of a check has no visual signature. Review every new endpoint/handler against a fixed security checklist rather than reading for vibes; run SAST in CI as the backstop.
- **Over-engineering.** Interfaces with one implementation, config knobs for constants, defensive handling of impossible states, an abstraction layer "for future flexibility" nobody requested. Two sources: the model's prior toward textbook-shaped code, and — increasingly common — *review-loop feedback*: an adversarial reviewer asked to find gaps will always report some, and chasing every finding accretes defensive cruft. Instruct reviewers (human and agent) to flag only correctness and stated-requirement gaps; treat "consider adding…" findings as default-reject. Simplicity is a requirement you must state explicitly, because the model's prior runs against it.
- **Scope drift.** "Improved" adjacent code, reformatted untouched files, renames for consistency — inflating the review surface and hiding the real change inside noise. Prompt with explicit scope ("change nothing outside `src/billing/`"), and treat out-of-scope hunks as revert-by-default no matter how nice they look.
- **Duplicate reimplementation.** The agent writes a fresh `formatCurrency` because it never saw `src/lib/money.ts`. This is a context failure, not a model failure: name the utility modules in the memory file, or instruct "search for an existing helper before writing one."
- **Convention interpolation from the wrong neighbor.** The agent imitates the nearest code it read — which may be the legacy half of your codebase. If output keeps arriving in the old style, the fix is context (point at the blessed exemplar; state which pattern wins in the memory file), not another correction.
- **Comment/code divergence.** Confident docstrings describing behavior the code doesn't have — an edge case the comment claims is handled, a parameter the comment says is validated. Review comments as claims to verify against the code, not as documentation of it.

### In the workflow

- **The correction loop.** Five rounds of "no, not like that" in one session. Each failed attempt stays in context, degrading subsequent attempts — you're fighting the transcript. Two failed corrections → fresh session, better prompt. The prompt-improvement step is the point: extract *why* it failed into the new spec, or you'll replay the loop verbatim.
- **Prompting harder instead of adding context.** Escalating to "IMPORTANT: you MUST…" when the real problem is the agent can't see the convention, the exemplar, or the constraint. Emphasis is a band-aid; a named file, a pasted error, or a memory-file line is the fix. Ask "what would a new human teammate have needed to get this right?" and provide that.
- **Reviewing by watching generation.** Watching code stream by produces familiarity that masquerades as review. Review the final diff cold — in a PR view, ideally after a break — never from the glow of having watched it appear.
- **Accepting completion claims without artifacts.** "All tests pass" — which command, which output? Demand the evidence in the transcript. For unattended runs, make the finish condition structural: a Stop hook or CI gate runs the check itself instead of trusting the agent's assertion that it did.
- **Letting the writer grade its own work.** Asking the same session "are you sure this is correct?" invites motivated reasoning over its own transcript. Verification worth having comes from a fresh context (review subagent, second session, CI) that sees the artifact without the reasoning that produced it.
- **Delegating the spec-discovery task.** "Build me a caching layer" when you don't yet know invalidation semantics: the agent will *pick* semantics, confidently, and the cost surfaces weeks later. If you can't state acceptance criteria, the deliverable isn't code yet — run the interview/plan pattern to discover the spec, or prototype yourself.
- **One mega-session for a multi-day feature.** Context fills, compaction loses decisions ("why did we choose approach B?"), and the agent re-litigates settled questions. Externalize state into files that survive; treat each session as a stateless worker that reads SPEC.md/PLAN.md, advances one checkbox, and updates the files.
- **Fan-out without a pilot.** Running the 2,000-file loop on the first prompt draft bakes every prompt defect into 2,000 diffs. Always pilot on 2–3 representative cases, iterate the prompt on what went wrong, then scale.
- **Unbounded scoping of investigation.** "Investigate the auth system" with no boundary reads hundreds of files into your working context. Scope it ("how does token refresh work, and do we have OAuth utilities to reuse?") or push it into a subagent whose context absorbs the exploration.
- **Treating repo-memory investment as optional.** A team that never writes corrections down pays the same context tax every session, forever, and concludes "the agent doesn't get our codebase." The agent-tuned repo — lean memory file, hooks for hard rules, skills for workflows, exemplars named, checks runnable — is infrastructure with compounding returns. It is also precisely what makes CI agents viable, where no human is present to fill context gaps interactively.

## Worked micro-examples

**Spec upgrade — the before/after that eliminates the correction loop:**

> *Weak:* "add rate limiting to the API"
>
> *Real spec:* "Add token-bucket rate limiting as middleware in `src/middleware/` (follow the structure of `auth.ts` there). Limit: 100 req/min per API key, key from `X-Api-Key` header; missing key → limit by IP. On limit: HTTP 429 with a `Retry-After` header in seconds. Store buckets in the existing Redis client from `src/lib/redis.ts` — do not add a dependency. Out of scope: per-endpoint overrides, admin bypass. Write tests covering: under limit, at limit, over limit, header present after 429, and two keys not sharing a bucket. Run `npm test` and show the output. Change nothing outside `src/middleware/` and its tests."

Every sentence exists to kill a named failure mode: the exemplar pointer (wrong-neighbor interpolation), "do not add a dependency" (slopsquatting, over-engineering), the out-of-scope list (scope drift, plus a requirement checklist for review), enumerated test cases (test-shaped non-tests), "show the output" (completion claims).

**Headless fan-out skeleton (Claude Code, flags verified as of 2026):**

```bash
# after piloting the prompt on 3 files by hand
for f in $(cat handlers.txt); do
  claude -p "Migrate $f to the Result convention per MIGRATION_SPEC.md. \
Run 'npm test -- $f' and 'npm run lint'. If the file doesn't fit the spec's \
rules, append it to SKIPPED.md with a reason and change nothing. \
Commit only $f with message 'migrate: $f'. Print OK or SKIP." \
    --allowedTools "Edit,Bash(npm test *),Bash(npm run lint),Bash(git add *),Bash(git commit *)" \
    --output-format json >> results.jsonl
done
```

Per-file commits (unit of revert), scoped tools (no outlet for verification-weakening), an explicit escape hatch (no improvisation), machine-parsable results.

**Fresh-context review prompt (checks the diff against the spec, not against vibes):**

```text
Use a subagent to review the rate-limiter diff against SPEC.md. Check that
every requirement is implemented, each listed test case exists and asserts
real behavior (not mocks of itself), and nothing outside the stated scope
changed. Report gaps against the spec only — no style preferences, no
"consider adding" suggestions.
```

The last sentence is load-bearing: a gap-hunting reviewer always finds something, and unfiltered findings drive the over-engineering spiral.

**CLAUDE.md that earns its context (bridging AGENTS.md for other tools):**

```markdown
@AGENTS.md

## Claude Code specifics
- Run `pnpm test:unit <path>` for single files — full suite takes 20 min, never run it unprompted.
- Migrations in `db/migrations/` are append-only; never edit an existing one (enforced by hook, but plan around it).
- Error handling: use `Result<T, AppError>` from `src/lib/result.ts`, not exceptions. Good example: `src/api/users/create.ts`.
- Before writing a helper, check `src/lib/` — formatting, money, and date utilities already exist.
```

Every line is something the agent can't infer, would get wrong by default, or needs an exemplar for. Nothing describes what the code already shows.

## Verification / self-check

Before accepting an AI-produced change, in order:

1. **Mechanical gates green** — typecheck, lint, relevant tests — run by you or CI, not taken from the agent's transcript on faith for anything that matters.
2. **Flip test on new tests** — break the implementation; the new tests must fail. Skip this only for tests you read closely enough that you could have written them.
3. **Spec checklist** — every requirement in the spec located in the diff, one by one; every out-of-scope item confirmed absent. This is where silent drops die.
4. **Diff hygiene scan** — new dependencies (each verified real and needed), edits to tests/configs/CI not demanded by the task, out-of-scope hunks, suppression markers (`ignore`, `skip`, `eslint-disable`, bare `except`), comments claiming behavior the code lacks.
5. **Comprehension bar** — for code you'll own: can you explain why each nontrivial hunk is correct? A hunk that resists explanation is where the bug is; make the agent justify it or simplify it.

**Stopping rules.** Review depth scales with blast radius and inversely with mechanical coverage: leaf code behind a flip-tested suite gets the checklist plus hygiene scan, then you stop; authz, billing, migration, and concurrency code gets line-level scrutiny regardless of green checks. Stop iterating with the agent when the checks pass and the spec checklist closes — "the reviewer still has suggestions" is not a continue signal, because a gap-hunting reviewer always has suggestions. And stop *delegating* — write it yourself — when you're on the third fresh-session attempt with a genuinely improved spec each time: three spec-quality failures on one task means either the spec isn't statable yet (discover it by prototyping) or the review cost has exceeded the writing cost.
