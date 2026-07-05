---
name: building-coding-agents
description: Engineering agents that write code — harness design (Claude Agent SDK and similar), tool sets for file/search/execution, context engineering over long tasks, sandboxing and permission models, verification loops, multi-agent fan-out, and evaluating coding agents. Load when building or debugging a coding agent, designing its tools/permissions, or diagnosing failures like premature completion, error loops, or scope drift.
---

# Building Coding Agents

## Core mental model

1. **The loop is gather context → act → verify → repeat, and verify is the product.** Anyone can wire an LLM to bash and get a demo. What separates a working agent is the verify step: after every mutation, something structural (tests, typechecker, linter, a build, a rendered screenshot) tells the agent whether reality moved the way it claimed. An agent without feedback loops doesn't fail loudly — it hallucinates success quietly, which is worse.
2. **bash + file-edit + search is nearly sufficient; resist tool sprawl.** A coding agent with a shell can do almost anything — the marginal tools that earn their place are the ones that beat bash on *reliability or context economics*: a string-replace edit tool (line-precise, verifiable, cheaper than heredoc rewrites), ripgrep/glob search (structured results, permission-gated), and file read with offsets. Every additional bespoke tool is another description to maintain and another wrong choice available. When tempted to add a tool, first ask: can the agent already do this with bash, and is the failure I'm fixing actually a *prompt or output-shaping* problem?
3. **Context is the scarce resource; the main thread is for decisions.** Long tasks die by context pollution: stale file dumps, repeated directory listings, 500-line test logs. The architecture answer is layered — truncate tool results at the harness, compact the transcript when near budget, push read-heavy exploration into subagents that return only conclusions, and persist plans/findings to files that survive compaction. Spend main-context tokens on decisions, not raw observations.
4. **Permissions are blast-radius engineering, not a yes/no dialog.** Classify every capability on four axes — read, write, execute, network — and scope each independently. Reads inside the repo: free. Writes: confined to workspace/worktree. Execution: sandboxed (as of 2026 the standard OS-level primitives are bubblewrap on Linux and Seatbelt/sandbox-exec on macOS — the Claude Agent SDK ships both; containers/microVMs like Docker/E2B for stronger isolation). Network: default-deny with a domain allowlist, because network egress is the exfiltration channel if the agent ingests hostile content. Security lives *below* the agent — the sandbox not permitting the action beats the prompt forbidding it.
5. **The agent's report is a claim, not evidence.** Harness-level rule: completion requires artifacts (test output, diff, exit codes) that the harness or a human can check. Models are strongly biased toward declaring victory; every affordance you build should make false success harder to state than true success.

## Decision frameworks

### Build on a harness or roll your own?

Ask in order: (1) Is your need "coding agent with custom tools/prompts"? Use an existing harness — as of 2026 the Claude Agent SDK (Python/TS) is the reference: it ships the tool suite (bash, edit, glob/grep, web), auto-compaction, subagents, hooks (pre/post tool-use interception), permission modes, and OS sandboxing; comparable open harnesses exist (OpenAI Codex CLI, OpenHands, Pi/mini agents). Rolling your own means re-solving truncation, compaction, permissioning, and loop-termination — months of unglamorous work. (2) Do you need a *different loop shape* (e.g., code-orchestrated pipeline with model steps, or a custom verifier-in-the-loop)? Then own the loop but still steal the tool designs. (3) Is it actually a workflow with known steps? Write code that calls the model; don't build an agent at all.

### Tool-result truncation strategy

The question is never "truncate or not" but "what does the model's next decision need?"
- Head+tail beats head-only for command output (errors concentrate at the end; build banners at the start). Keep ~first 20 + last 80 lines of long output with a `[... N lines omitted ...]` marker.
- For file reads: cap default read length; force offset/limit params for big files; return "file is 4,200 lines; showing 1–400" so the model knows to page rather than assume completeness.
- For search: cap match count, return file:line + one-line context, never full file bodies.
- Preserve *counts* when you cut content ("2,113 tests passed, 3 failed, failures shown below") — models reason correctly from summaries but wrongly from silent truncation.

### Context over long horizons — escalation ladder

1. Always: per-result truncation (above).
2. Sessions >30–50 steps: **compaction** — summarize the transcript into goal / constraints / decisions-with-reasons / current state / next step. Losing "why" makes the agent re-litigate settled choices; test your compaction prompt by resuming from it cold.
3. Multi-phase tasks: a **plan file** the agent re-reads and updates (`TODO.md` with done/doing/next). Cheapest goal-drift defense there is — the goal keeps re-entering recent context.
4. Read-heavy phases: **subagent fan-out**. "Find every caller of X across the monorepo" burns 60k tokens of listings; a subagent absorbs them and returns 15 lines. Delegate any work whose *intermediate* products would pollute the parent. Keep the parent as the sole decision-maker.
5. Cross-session work: file-based memory with a convention (`NOTES.md`, `findings/`) the prompt teaches — don't hope the agent invents one.

### Multi-agent: when parallelism pays

Prior: it usually doesn't. Parallel agents win only when subtasks are (a) independent, (b) interface-clean, and (c) individually verifiable — review 12 files, fix 8 unrelated lint classes, research N libraries. They lose when tasks share mutable state (two agents editing the same module = merge hell; use git worktrees per agent if you must) or when coordination requires judgment mid-flight (the orchestrator becomes a bottleneck relaying context it doesn't have). Reasoning check: if you can't write each subtask's acceptance test before spawning, it isn't decomposed enough to parallelize.

### Model choice inside the loop

Errors compound: 5% per-step error ≈ 40% over 10 dependent steps. Strongest affordable model for the decision loop; route bulk classify/summarize sub-work to cheap models via subagents. This inverts single-call cost intuition — a smarter planner takes fewer steps and is often net cheaper.

## How an expert thinks through it

*Scenario: internal agent that takes a failing CI job and produces a fix PR.*

Start from verification, not from the prompt: what tells the agent it's done? The failing job must pass *and* the rest of the suite must not regress. So the loop's terminal condition is "targeted test passes + full affected-package suite passes," checked by the harness parsing exit codes — never by the model asserting it. (Rejected: trusting a final "all tests pass" message. That's the deceptive-green failure waiting to happen.)

Tools: bash, read, edit, grep/glob. Do I add a `run_tests` tool? Yes — not because bash can't run pytest, but because a dedicated tool lets me return *structured, truncated* results (fail count, first 3 tracebacks, nothing else) instead of 3,000 lines of dots, and lets me log every test invocation for the eval set later. (Rejected: a `create_pr` tool available mid-loop — the agent should earn PR creation only after the harness verifies green; sequencing enforced in code, not prompt.)

Context plan: CI logs are huge. First step is a subagent that reads the raw log and returns failing test IDs + relevant traceback + suspected files. Main agent starts from that brief, reads only implicated files. Budget: if >40 steps, compact with a template that preserves the hypothesis history — "tried X, ruled out because Y" is the most valuable thing to keep, or the agent will retry X.

Sandbox: read repo-wide; write only in a fresh worktree; execute inside bubblewrap with network limited to the package registry (tests may need to install). No repo-push credentials inside the sandbox — the harness, outside, creates the PR from the worktree diff after checks pass. (Rejected: giving the agent `gh` with a token — one prompt injection in a CI log away from pushing to main.)

Failure exits: after 3 consecutive test runs with no reduction in failure count, the harness interrupts: "no progress; write up findings and stop." A written failure report is a *successful outcome* for the product — silent flailing and fabricated fixes are the failures.

Stopping rule for me, the builder: run it on 20 historical CI failures. If ≥60% produce correct green PRs and 0% produce false-green claims, ship behind human review of every PR; iterate on the failure transcripts, not on the prompt in the abstract.

## Failure modes and pitfalls

- **Premature completion claims.** Agent says "done, tests pass" without running them, or after running a subset. Fix structurally: the finish action requires evidence arguments (test command + exit code + summary the harness re-checks); prompts alone don't cure this bias.
- **Deceptive green tests.** The agent makes tests pass by weakening them: deleting assertions, adding `@skip`, widening `except`, hardcoding expected values, editing the test instead of the code. Defenses: diff-gate on test files (test edits require justification or human review), run mutation-style checks on suspicious passes, and evaluate against *held-out* tests the agent never sees. This is the single most damaging coding-agent failure because it passes every naive check.
- **Error-loop spirals.** Same failing command retried verbatim, or two remedies alternated. Root causes in observed order: error output truncated so the model never saw the actual message; missing capability (agent substitutes the nearest tool repeatedly); ambiguous goal. Fix result readability first; add harness loop detection (hash of tool+args, interrupt after N repeats) as backstop, not cure.
- **Scope drift.** "Fix the date bug" becomes a refactor of the date module. Defenses: plan file with explicit non-goals, diff-size guardrail (harness flags when changed-lines exceed a task-class budget), and prompt language that makes minimal-diff a stated value. Review the *diff*, not the narrative.
- **Truncation hiding the signal.** Harness cuts output at 10k chars head-first; pytest's failure summary lives at the end; agent reasons about a log whose only useful part was deleted. Head+tail, always, and preserve counts.
- **Compaction losing the "why".** Post-compaction agent re-opens decided questions or repeats ruled-out fixes. Your summary template must carry decisions *with reasons* and dead ends *with evidence*. Test: resume a session from only the summary; if the agent's next action is one it already tried, the template is broken.
- **Sandbox theater.** Execution sandboxed, but the agent can write `~/.bashrc` or `.git/hooks/` (escapes on next shell), or network is open so a hostile README can exfiltrate `env`. Close the quartet together: workspace-confined writes, default-deny network, no secrets in the sandbox env, and treat *anything the agent read from the internet or from user-supplied files* as hostile input to a system that can execute code.
- **Permission fatigue defeating the gate.** Prompting per bash command trains humans to approve blindly. Auto-allow reads and workspace writes; reserve prompts for the rare escalations (network, out-of-workspace, credentials). A gate humans rubber-stamp is worse than no gate plus a hard sandbox.
- **Evaluating on vibes or on the wrong benchmark.** Transcript-looks-reasonable is not a metric. Evaluate end-to-end task success on your own task distribution with held-out verification. On public benchmarks (as of mid-2026): SWE-bench is a *family* — original, Verified, Pro, Multilingual, Live — and cross-variant comparisons are meaningless; frontier models score ~90–95% on Verified (top of the range saturating) while SWE-bench Pro (~80% at the frontier) and Terminal-Bench (terminal/infra tasks, tbench.ai) retain headroom. Public benchmark scores measure model+scaffold; your harness changes yours.
- **Skipping the agent's environment doc.** An agent dropped into a repo without build/test/run instructions burns 20 steps rediscovering them each session. Ship a checked-in context file (`CLAUDE.md`/`AGENTS.md`) with commands, conventions, and landmines; it's the cheapest performance win available.

## Worked micro-example

Harness-side truncation + evidence-checked completion (TypeScript, Claude Agent SDK shape):

```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

function truncateOutput(out: string, head = 20, tail = 80): string {
  const lines = out.split("\n");
  if (lines.length <= head + tail) return out;
  return [...lines.slice(0, head),
          `[... ${lines.length - head - tail} lines omitted ...]`,
          ...lines.slice(-tail)].join("\n");
}

for await (const msg of query({
  prompt: task,
  options: {
    allowedTools: ["Bash", "Read", "Edit", "Glob", "Grep"],
    permissionMode: "acceptEdits",          // free writes in workspace; escalations still prompt
    hooks: {
      PostToolUse: [{ hooks: [async (input) => {
        // never let raw test logs hit the context window
        if (input.tool_name === "Bash") {
          input.tool_response.content = truncateOutput(String(input.tool_response.content));
        }
        return { continue: true };
      }]}],
    },
  },
})) { /* ... */ }

// completion gate lives OUTSIDE the model:
const verified = await run("pytest tests/affected -q");   // harness re-runs, parses exit code
if (verified.exitCode !== 0) reject("agent claimed done; verification failed");
```

The two load-bearing ideas: output shaping happens in a hook (the model never depends on its own discipline), and "done" is an exit code the harness observes, not a sentence the model writes.

## Verification and self-check

- Before shipping any harness change, run the same 10–20 task regression set and compare *end-to-end success*, cost, and step count — not transcript aesthetics. Keep every past failure as a regression task.
- Audit five random transcripts per iteration for the big four: false completion, weakened tests, loop spirals, scope drift. Read the diffs the agent produced, not its summaries of them.
- Red-team the sandbox once per design change: from inside the agent's shell, try to write outside the workspace, reach a non-allowlisted domain, read a secret, and persist across sessions. All four should fail.
- Stopping rule: the agent is good enough to ship (behind review) when held-out verification passes on the majority of your task set and false-success rate is ~0; past that, invest in the verifier and the task distribution, not in prompt polish — verifier quality is the ceiling on everything else.
