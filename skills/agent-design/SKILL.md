---
name: agent-design
description: Architecting agentic LLM systems — tool design, context management over long horizons, single-agent vs multi-agent vs code-orchestrated workflows, error recovery and stopping criteria, verification after mutations, human-in-the-loop placement, and agent evaluation. Load when designing or debugging an agent, an agent's tool set, a multi-step LLM workflow, or an agent that loops/stalls/claims false success.
---

# Agent Design

## Core mental model (anchors — standard doctrine; the value is applying it without exception)

1. Tools are the highest-leverage surface: few, orthogonal, task-shaped; descriptions say when NOT to use; *return values are prompts* — errors must say what to do differently, success returns what the next decision needs, not 40KB of JSON.
2. An agent is a loop that must terminate: step/budget caps, repeat detection, evidence-checked completion — all in the harness, never only in the prompt.
3. Least agency that solves the task: known steps → code-orchestrated workflow; model-as-router at branch points; a true agent only where the path is discovered by doing. Multi-agent free-form chat: almost never — structural reasons only (context isolation, true parallelism, distinct permissions).
4. Context degrades over long horizons: summarize stale tool results, compaction brief, plan file re-read each phase, subagents to absorb bulky exploration.
5. The agent's claim of success is not evidence: verify mutations through a *different channel* than the mutation (run tests, GET after POST, re-SELECT); score evals on task outcome against the *original* instruction, plus cost-to-success.
6. Decision loop gets the strongest model (5%/step error ≈ 40% over 10 dependent steps); bulk sub-work goes cheap. Opposite of single-call cheap-first intuition.
7. Prompt injection via tool results is an attack surface by construction; fencing helps, but the only robust line is capability scoping — a fully hijacked agent must still be unable to exfiltrate or destroy.

## The corrections (what the standard playbook omits)

**Design the honorable exit or the agent fabricates success.** Everyone builds success paths and step caps; almost nobody makes *failure a first-class outcome*. If the transcript offers no legitimate way to stop without succeeding, an agent that can't finish will claim it finished — premature-completion is a top-3 failure mode and it's largely *induced by the harness*, not by model dishonesty. Concretely: a `report_failure` tool (what was tried, best partial result) that is explicitly allowed in the prompt; a finish action with required evidence fields that the *harness* checks (execute the tests, diff the file, re-fetch the record) and rejects with "continue or report inability"; and on stuck-detection, an interrupt that names both options. The same principle produced SKIPPED.md escape hatches in bulk migrations: without an honorable exit, the model improvises.

**Plan-once-never-replan.** The agent writes a good 8-step plan; step 3's result invalidates it; steps 4–8 execute anyway because the plan sits in context looking authoritative. Prompt for explicit revision after surprising results ("after each step, state whether the plan still holds"); in workflows, put replan checkpoints in code. Reviewers check that a plan exists, not that it's ever re-derived.

**Every mutating tool needs a named read-back.** Sharper than "verify your work": for each mutation in the toolset, there must exist a cheap corresponding read tool, the finish-evidence schema should require the read-back result, and *if you can't name the read-back for a mutation, that mutation shouldn't be agent-invocable*. This turns verification from a behavioral hope into a toolset-design invariant you can audit.

**Retry budgets multiply across layers.** Retry-on-failure at tool wrapper × agent loop × orchestrator = 3×3×3 = 27 attempts of an expensive subagent, invisible until the bill. Budget retries globally and propagate a cost context downward; each layer reads the remaining budget instead of owning its own retry count.

**Compaction must carry decisions-with-reasons and dead-ends.** Generic summaries preserve goal and state; what gets lost is *why* rejected options were rejected and what was already tried — so the post-compaction agent re-litigates settled choices and re-walks dead-ends. Demand both explicitly in the compaction prompt, and test compaction by running the same task with and without it.

**Loop root causes, in observed order:** (1) tool errors that don't say what to do differently, (2) missing tool for the actual need (agent substitutes the nearest one repeatedly), (3) goal ambiguity. Fix return values first; identical-call hashing in the harness is the backstop, not the fix.

## Compressed checklist (one-liners; each earns its place)

- Dangerous ops structurally hard to misuse: preview + confirming second call for large blast radius; IDs must come from a prior read; enums so malformed calls fail validation with corrective text.
- Validate tool-arg *semantics* at the boundary (parse the datetime, check the ID exists) — tools are the agent's type system.
- Make tool failures loud in returned text ("ERROR: … Do not proceed as if this succeeded") and fault-inject in evals: force each tool to fail/return-empty once; watch for false success.
- Serialize mutating calls per resource; parallelism for reads or verified-disjoint writes only.
- HITL gates by irreversibility × blast radius: show the exact action, not a paraphrase; gate-everything → rubber-stamping, gate-nothing → mass delete; enforce scope *below* the agent (credentials, sandbox, egress allowlists).
- >~15–20 tools: consolidate overlaps, namespace, or scope toolsets per phase; two descriptions answering the same request = merge or add "NOT for" boundaries.
- Move enforcement from prompt scar tissue into the harness (the delete tool requires confirmation; no prompt rule needed); prune the prompt against the eval suite.
- Mocks must reproduce failure shapes (pagination, rate limits, empties), not just success shapes.
- Give the loop model room to think before non-trivial actions; the expensive failure is a wrong action, not a slow step.
- File-based memory (NOTES.md, plan file) survives compaction and restarts; teach the convention, don't hope the agent invents one.
- Eval suite of end-to-end tasks with programmatic success checks, rerun on every tool/prompt/model change — agents regress from one reworded tool description more than any other LLM system.

## Worked micro-example — the harness owns termination and honesty

```python
for step in range(max_steps):                          # hard cap in code
    if state.cost > budget: return state.fail("budget", partial=state.best())
    action = llm_decide(state.context(), tools)
    if action.is_finish():
        ok, detail = verify(action.evidence, task)     # harness runs tests/diffs/re-fetches
        if ok: return state.succeed(action)
        state.observe(f"Completion rejected: {detail}. Continue or report_failure.")
        continue
    if state.repeats(action, n=3):
        state.observe("Stuck: 3 identical calls. Change approach or finish with report_failure.")
        continue
    state.observe(truncate_or_summarize(execute(action)))
return state.fail("step limit", partial=state.best())  # failure is a real outcome
```
`report_failure` exists as a tool, so the honest exit is always available; `verify()` judges evidence in code — the agent supplies it, the harness rules.

## Verification / self-check

- Termination audit: point to the code line for step cap, budget cap, repeat detection, evidence-checked completion, honorable failure exit. "It's in the prompt" fails.
- Mutation audit: every mutating tool → its read-back → whether the harness (not just the agent) performs it; no gap on irreversible actions.
- Global retry budget traced through all layers; worst-case attempt count computed.
- Fault-injection run in the eval suite; compaction A/B tested for decision retention.
- Least-agency check per component: articulate why code couldn't do it, or demote to a workflow.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)
- Probed 15 claims: 13 baseline (cut/compressed), 2 partial (sharpened), 0 delta.
- Biggest baseline gaps found:
  - Opus verifies completion claims but misses that fabricated success is *induced by harness design* — no honorable failure exit (report_failure tool, evidence-rejected-continue path) in its answers.
  - States "verify with ground truth" but not the auditable invariant: every mutating tool pairs with a named read-back, else not agent-invocable.
  - Silent on plan-staleness (never-replan) and on retry-budget multiplication across nested layers.
