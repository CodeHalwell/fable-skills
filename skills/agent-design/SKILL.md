---
name: agent-design
description: Architecting agentic LLM systems — tool design, context management over long horizons, single-agent vs multi-agent vs code-orchestrated workflows, error recovery and stopping criteria, verification after mutations, human-in-the-loop placement, and agent evaluation. Load when designing or debugging an agent, an agent's tool set, a multi-step LLM workflow, or an agent that loops/stalls/claims false success.
---

# Agent Design

## Core mental model

1. **Tools are the highest-leverage surface — most "agent" problems are tool problems.** The model reasons over exactly what tool names, descriptions, parameters, and *return values* tell it. Before touching the system prompt, orchestration, or model choice, audit the tools: are there few of them, orthogonal, with descriptions that say when to use (and when NOT to use) each, and do their outputs give the model what its *next* decision needs? A mediocre model with excellent tools outperforms an excellent model with vague, overlapping tools.
2. **An agent is a loop that must terminate.** Every agent needs explicit stopping criteria — max steps, budget, timeout, "done" condition the agent must *demonstrate* — designed before the happy path. The default failure mode of an unbounded loop isn't crashing; it's burning $40 re-listing the same directory.
3. **Prefer the least agency that solves the task.** If the step sequence is known ahead of time, write it as code that calls the model (a workflow), not a model that decides steps (an agent). Agents buy flexibility at the price of reliability, cost, latency, and debuggability — spend that price only where the path genuinely can't be predetermined. "Agentic" is a cost, not a feature.
4. **Context is the agent's working memory and it degrades over long horizons.** Every step appends tool results; the window fills with stale observations, and models increasingly lose the thread ("goal drift") as the transcript grows. Long-horizon agents need deliberate memory architecture — compaction, external notes, files — not just a big window. The window being *big enough* doesn't mean the middle of it still steers behavior.
5. **The agent's claim of success is not evidence of success.** Verify mutating actions with a separate read/check (tests pass? row exists? file diff matches intent?), and evaluate agents on *task outcomes*, not on whether their steps looked reasonable. Models are eager to declare completion; premature-completion is a top-3 failure mode and the reason "did it actually work" checks must be structural, not vibes.

## Decision frameworks

**Architecture selection (in increasing order of agency — stop at the first row that fits):**
| Task shape | Build | Reasoning |
|---|---|---|
| Fixed, known steps (extract → transform → validate → write) | Code-orchestrated workflow: code controls flow, model fills specific steps | Deterministic control flow, unit-testable stages, per-stage retries. Most "we need an agent" requests land here |
| Known steps + branching on model judgment | Workflow with model-as-router at branch points | Keep flow in code; the model makes bounded choices from enumerated options |
| Unknown/variable path, tools needed, single coherent goal (debug this failure, research this question, do this refactor) | Single agent, tool loop, tight stopping criteria | The genuine agent case: path discovered by doing |
| Task decomposes into *independent, parallelizable* subtasks with clean interfaces (research N companies, review M files) | One orchestrator (or plain code) spawning parallel subagents; results merged | Parallelism and context isolation are real wins here — each subagent gets a fresh window |
| "Multiple agents debating/collaborating in free-form conversation" | Almost never | Multi-agent chat compounds error rates, multiplies cost, and mostly relocates bugs into inter-agent misunderstanding. Use only with a structural reason: context isolation, true parallelism, or genuinely different tool/permission sets per role |

**Tool design rules:**
- Fewer, task-shaped tools over many API-shaped ones. Wrap `search_flights(origin, dest, date)` rather than exposing 12 raw REST endpoints; collapse `get_user`, `get_user_by_email`, `lookup_account` into one tool with clear parameters. Overlapping tools force a choice the model has no basis to make — every such choice is a new error source.
- Description = when to use + when not to + what it returns + a concrete example of good arguments. The description is a prompt; treat it with prompt-engineering care. Most tool-selection errors trace to descriptions a new team member also couldn't act on.
- Return values are prompts too. Return what the next decision needs: on error, return *actionable* text ("file not found; sibling files: [a.py, b.py]") not a stack trace; on success, return confirmation + salient state, not 40KB of raw JSON (truncate/summarize, offer a drill-down tool). An agent that "ignores errors" is usually an agent whose errors are unreadable.
- Make dangerous operations structurally hard to misuse: separate `delete_row(id)` from `delete_all(confirm_phrase=...)`; require IDs obtained from a prior read; make destructive tools return a preview + require a second confirming call for large blast radii.
- Idempotency & typed args: enum/constrain parameters so malformed calls fail at validation (with a corrective message the model can act on) rather than half-executing.

**Context management across long horizons:**
| Mechanism | Use when | Detail |
|---|---|---|
| Truncate/summarize old tool results in place | Always, by default | A 5-step-old directory listing earns a one-line summary; keep the most recent results verbatim |
| Rolling compaction (summarize transcript into a structured brief when near budget) | Sessions beyond ~30–50 steps | The compaction summary must preserve: goal, constraints, decisions made + why, current state, next step. Losing "why" causes the agent to re-litigate settled decisions |
| Scratchpad / plan file the agent updates | Multi-phase tasks | An explicit, re-read plan ("done: 1,2; now: 3") is the single cheapest goal-drift defense — the goal keeps re-entering recent context |
| File-based memory (notes the agent writes/reads via tools) | Very long horizons, cross-session work | Files survive compaction and restarts; teach the agent a convention (e.g., `NOTES.md`, `findings/`) rather than hoping it invents one |
| Subagent isolation | Bulky exploratory work (read 30 files, big searches) | The subagent's window absorbs the bulk; only its conclusion returns to the parent. Rule: delegate work whose *intermediate* products would pollute the parent's context |

**Human-in-the-loop placement — position gates by irreversibility × blast radius, not by step count:**
- Auto-proceed: reversible + low blast radius (read, search, draft, branch-local edits).
- Confirm before execution: irreversible or external-facing (send email, merge, deploy, payment, delete) — show the *exact* action content, not a paraphrase.
- Batch review: high-volume low-stakes actions; sample-audit instead of gating each one.
- Escalate: agent uncertain, repeated failures, or action outside granted scope.
Two failure directions, both fatal: gate everything → humans rubber-stamp without reading (alarm fatigue defeats the gate); gate nothing → one bad loop mass-deletes. Also enforce scope *below* the agent (API-token permissions, sandbox, allowlists) — the agent asking permission is UX; the credential not having permission is security.

**Stopping criteria — implement all four, in the harness (not the prompt):**
1. Hard step/budget/time caps (in code; the model can't be trusted to count its own steps).
2. Progress detection: same tool + same/equivalent args N times, or no state change in K steps → interrupt with "you appear stuck; state what you learned and change approach or stop."
3. Success condition requiring evidence: the finish action takes proof arguments (test output, diff, fetched confirmation), and the harness checks them where possible.
4. Failure exit as a first-class outcome: "report inability + what was tried + best partial result" must be an explicitly allowed, prompted-for ending — otherwise the agent fabricates success rather than admit failure, because the transcript offered it no honorable exit.

## Failure modes and pitfalls

- **Tool-call loops.** Agent alternates between two searches, or retries a failing call verbatim. Root causes, in observed order: tool errors that don't say what to do differently; missing tool for the actual need (agent substitutes the nearest one repeatedly); goal ambiguity. Fix the return values first. Harness-level loop detection (identical call hashing) is the backstop, not the fix.
- **Goal drift.** Twenty steps in, a research agent is summarizing an interesting-but-irrelevant tangent. Corrections: plan file re-read each phase; original task restated in the compaction brief; orchestrator-level relevance check on subagent returns. Detect in evals by scoring final output against the *original* instruction, never against what the agent redefined the task to be.
- **Premature completion claims.** "I've fixed the bug and all tests pass" — no test was run. Correction: make completion a tool call with required evidence fields; run the verification *in the harness* (actually execute the tests; diff the file; query the row). Never forward an agent's self-reported success to a user or a dependent system unverified.
- **Verification theater.** Agent writes code, then "verifies" by re-reading the code and declaring it correct. Reading is not running. The verify step must exercise a *different channel* than the mutation: run tests after edits, GET after POST, `ls` after write, screenshot after UI change. Same principle as double-entry bookkeeping.
- **Silent tool failure treated as success.** Tool returns `{"status": "error"}` inside a 200-shaped payload; agent barrels on and later actions corrupt state. Make failures loud in the returned text ("ERROR: ... . Do not proceed as if this succeeded.") and test agent behavior under injected tool failures — fault injection is the agent equivalent of chaos testing and almost nobody does it.
- **Over-tooling.** 40 tools in context: selection accuracy drops, prompt cost balloons, near-duplicate tools split the model's confidence. Corrections: consolidate overlapping tools; group rarely-used ones behind a two-step pattern (a `list_capabilities`/router tool) or scope toolsets per phase. If two tools' descriptions could answer the same request, merge them or sharpen the boundary sentence in each.
- **Compaction that loses decisions.** After summarization the agent redoes work or reverses a settled choice ("actually, let's use library X" — which it rejected pre-compaction for a concrete reason). The compaction prompt must explicitly demand decisions-with-reasons and things-tried-that-failed; test compaction by comparing agent behavior with and without it on the same task.
- **Multi-agent as a debugging multiplier.** A wrong answer now requires tracing which agent misunderstood which other agent — and inter-agent messages are lossy paraphrases. If you must go multi-agent: structured (schema'd) inter-agent messages, single writer per resource, orchestrator owns global state; never have two agents mutate the same artifact concurrently.
- **Evaluating step imitation instead of outcomes.** Scoring "did the agent follow the expected trajectory" penalizes valid alternate paths and rewards cargo-cult step sequences. Evaluate: task success (checkable end state), cost/steps to success, and safety violations en route. Keep a suite of end-to-end tasks with programmatic success checks; run it on every prompt/tool/model change — agents regress from tiny changes (one reworded tool description) more than any other LLM system.
- **Sandbox-prod parity gap.** Agent developed against a mock that returns clean data; prod tool returns paginated, rate-limited, occasionally-empty responses. The agent's error handling was never exercised. Mocks must reproduce failure shapes, not just success shapes.
- **Unbounded retry cost.** Retry-on-failure at three nested layers (tool wrapper, agent loop, orchestrator) multiplies: 3×3×3 = 27 attempts of an expensive subagent. Budget retries globally, propagate a cost context downward, and cap subagent spend explicitly.

## Worked micro-examples

**1. Tool description, weak → strong:**
```json
// WEAK — the model must guess semantics, units, limits, and failure behavior
{"name": "search", "description": "Searches the database",
 "parameters": {"q": {"type": "string"}}}

// STRONG
{"name": "search_orders",
 "description": "Full-text search over customer orders. Use for finding orders by product name, customer email, or order notes. NOT for aggregate stats (use get_order_stats) and NOT for looking up a known order ID (use get_order). Returns at most 20 matches, newest first; if 20 are returned, results were truncated — narrow the query. Example: query='refund dyson v11', status='open'.",
 "parameters": {
   "query":  {"type": "string", "description": "Keywords, not natural-language questions. 2-6 terms work best."},
   "status": {"type": "string", "enum": ["open", "closed", "any"], "description": "Default 'any'."}}}
```
Load-bearing pieces: when NOT to use (routes traffic between sibling tools), truncation semantics (prevents "there are only 20 orders" false conclusions), argument shape guidance with an example, and an enum that makes an invalid status a validation error instead of a silent mismatch.

**2. Agent loop skeleton with the four stopping criteria in the harness:**
```python
def run_agent(task, tools, max_steps=25, budget_usd=2.00):
    state = AgentState(task=task)
    for step in range(max_steps):                                  # (1) hard cap
        if state.cost > budget_usd:
            return state.fail("budget exceeded", partial=state.best_result())
        action = llm_decide(state.context(), tools)                # returns tool call or finish
        if action.is_finish():
            ok, detail = verify(action.evidence, task)             # (3) evidence checked in code
            if ok: return state.succeed(action)
            state.observe(f"Completion rejected: {detail}. Continue or report inability.")
            continue
        if state.repeats(action, n=3):                             # (2) progress detection
            state.observe("Stuck: 3 identical calls. Summarize findings; change approach or finish with report_failure.")
            continue
        result = execute(action)                                   # errors -> actionable text, not raise
        state.observe(truncate_or_summarize(result))               # context hygiene every step
    return state.fail("step limit", partial=state.best_result())   # (4) failure is a real outcome
```
Note `verify()` runs in the harness with real checks (execute tests, diff files, re-fetch the record) — the agent supplies evidence; the code judges it. `report_failure` exists as a tool so the honest exit is always available.

**3. Verification-after-mutation pattern (the read-back rule):**
```text
Agent task: "update the customer's plan to 'pro' in Stripe and our DB."
Wrong: call stripe.update → call db.update → finish("done").
Right: stripe.update → stripe.GET subscription (assert plan == "pro")
       → db.update → db.SELECT (assert plan == "pro" AND updated_at fresh)
       → finish(evidence={stripe_sub_id, plan_readback, db_row}).
```
Rule of thumb: every mutating tool in the toolset should have a cheap corresponding read tool, and the agent's instructions (plus the finish-evidence schema) should require the read-back. If you can't name the read-back for a mutation, that mutation shouldn't be agent-invocable.

## Verification / self-check

- Tool audit: could a new engineer, given only the tool names/descriptions, pick the right tool for 10 sample requests? Any two tools they'd confuse must be merged or disambiguated with "NOT for" sentences.
- Termination audit: identify the code line enforcing each of — step cap, budget cap, repeat detection, evidence-checked completion, honorable failure exit. "It's in the prompt" fails this audit.
- Fault injection: rerun the eval suite with each tool forced to fail/return-empty once — does the agent notice, adapt, or falsely succeed?
- Context audit at step 30 of a long trace: is the original goal (or the plan file) inside the most recent few thousand tokens? Are stale bulky tool results summarized?
- Mutation audit: list every mutating tool → its read-back verification → whether the harness or only the agent performs it. Close any gap on irreversible actions.
- Eval discipline: task-outcome suite with programmatic success checks exists, runs on every tool/prompt/model change, and scores against the original instruction — plus cost-to-success, so a "fix" that doubles steps is visible.
- Least-agency check: for each agentic component, articulate why a coded workflow could not do it. No articulation → demote it to a workflow.
