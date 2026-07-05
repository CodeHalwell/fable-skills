---
name: building-coding-agents
description: Engineering agents that write code — harness design (Claude Agent SDK and similar), tool sets for file/search/execution, context engineering over long tasks, sandboxing and permission models, verification loops, multi-agent fan-out, and evaluating coding agents. Load when building or debugging a coding agent, designing its tools/permissions, or diagnosing failures like premature completion, error loops, or scope drift.
---

# Building Coding Agents

Frontier models already know the loop (gather → act → verify), the minimal tool set, truncation, compaction, sandbox axes, and the classic failure modes. This sheet keeps the anchors, the harness-side specifics, and the calibration a cold model gets wrong.

## Anchors (compressed — the parts to hold, not re-derive)

1. **Verify is the product.** After every mutation something structural (typecheck, targeted test, build, screenshot) tells the agent the truth; an agent without feedback hallucinates success quietly. Verifier quality is the ceiling on agent quality — a mediocre model with a tight loop beats a frontier model flying blind.
2. **bash + string-replace edit + read + grep/glob is nearly sufficient.** A new tool earns its place only by beating bash on reliability or context economics (structured/truncated output, a distinct permission gate, high frequency × error rate).
3. **Truncation: head+tail, always** (~first 20 + last 80 lines, `[... N omitted ...]` marker), and **preserve counts** ("2,113 passed, 3 failed") — models reason correctly from summaries but wrongly from silent truncation.
4. **Context ladder:** per-result truncation → compaction at >30–50 steps (summary must carry decisions *with reasons* and dead ends *with evidence*; test by cold-resuming — if the resumed agent retries a ruled-out fix, the template is broken) → plan file (`TODO.md`) → subagent fan-out for read-heavy phases → file-based memory across sessions.
5. **Permissions = blast-radius grid** (read/write/execute/network scoped independently): reads in repo free; writes workspace/worktree-only; execution inside an OS sandbox — **bubblewrap on Linux, Seatbelt/sandbox-exec on macOS are what the Claude Agent SDK ships**; containers/microVMs (Docker, E2B, Firecracker/gVisor) for untrusted-origin code; network default-deny with domain allowlist. Secrets never enter the sandbox — the harness performs privileged actions (push, PR) *outside*, from artifacts the sandbox produced. Once the agent reads untrusted content, treat its subsequent tool calls as potentially attacker-directed.
6. **Error math:** 5%/step ≈ 40% failure over 10 dependent steps. Strongest model in the decision loop, cheap models in bounded subagents — the smarter planner takes fewer steps and is often net cheaper.
7. **Multi-agent only when subtasks are independent, interface-clean, individually verifiable.** Test: if you can't write each subtask's acceptance test (or the merge step) before spawning, it isn't decomposed enough. Shared mutable state → worktrees per agent or don't parallelize.
8. **Completion is a harness-verified state transition, not a model sentence.** Re-run the verification from the harness, parse exit codes, diff-guard test files, and catch "0 tests ran" reported as success.
9. Ship a checked-in `CLAUDE.md`/`AGENTS.md`: exact build/test/lint commands, project map, conventions, landmines. Cheapest performance win available; keep it short — a stale one is worse than none.

## Build on a harness or roll your own?

Need = "coding agent with custom tools/prompts" → use an existing harness; as of 2026 the **Claude Agent SDK** (Python/TS) is the reference: tool suite, auto-compaction, subagents, **hooks (pre/post tool-use interception)**, permission modes, OS sandboxing. Rolling your own re-solves truncation/compaction/permissioning/termination — months of unglamorous work. Different loop shape (code-orchestrated pipeline, verifier-in-the-loop) → own the loop, steal the tool designs. Known steps → it's a workflow; don't build an agent.

## Failure modes (the big four + defenses — structural, never prompt-only)

- **Premature completion** → finish action requires evidence arguments the harness re-checks.
- **Deceptive green** (weakened asserts, `@skip`, hardcoded expecteds, editing the test) → diff-gate test files, held-out tests, mutation checks on suspicious passes. The most damaging failure because it passes every naive check.
- **Error-loop spirals** → root causes in observed order: truncated error output (fix first), missing capability, ambiguous goal; harness loop-detection (hash tool+args, interrupt after N) as backstop, not cure.
- **Scope drift** → plan file with non-goals, diff-size guardrail per task class; review the diff, not the narrative.
- Also: permission fatigue (a gate humans rubber-stamp is worse than no gate plus a hard sandbox — auto-allow reads/workspace writes, prompt only on real escalations); sandbox theater (writes to `~/.bashrc`/`.git/hooks` escape on next shell — close writes+network+secrets together).

## Benchmark calibration (where cold models are wrong, as of mid-2026)

The stale reflex is "frontier ≈ 70–80% on SWE-bench Verified." Current: **frontier models score ~90–95% on Verified — the top of the range is saturating**; **SWE-bench Pro sits ~80% at the frontier** and **Terminal-Bench** (terminal/infra tasks, tbench.ai) retains real headroom. SWE-bench is a *family* (original, Verified, Pro, Multilingual, Live) and cross-variant comparisons are meaningless. Public scores measure model+scaffold; your harness changes yours — evaluate end-to-end on your own task distribution with held-out verification, and keep every past failure as a regression task.

## How an expert thinks it through (compressed): CI-failure → fix-PR agent

Start from verification, not the prompt: terminal condition = failing test passes + affected suite green, checked by the harness parsing exit codes. Add a `run_tests` tool not because bash can't run pytest but to return structured, truncated results and log invocations for the eval set. CI logs are huge → first step is a subagent that reads the raw log and returns failing IDs + tracebacks + suspected files. No `gh` token inside the sandbox — one prompt injection in a CI log away from pushing to main; the harness creates the PR outside after checks pass. After 3 test runs with no progress, interrupt: a written failure report is a *successful* product outcome. Ship rule: ≥60% correct green PRs, 0% false-green, on 20 historical failures; iterate on failure transcripts, not prompt vibes.

## Worked micro-example — harness-side output shaping + external completion gate

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
    permissionMode: "acceptEdits",          // free workspace writes; escalations still prompt
    hooks: {
      PostToolUse: [{ hooks: [async (input) => {
        if (input.tool_name === "Bash") {   // raw logs never hit the context window
          input.tool_response.content = truncateOutput(String(input.tool_response.content));
        }
        return { continue: true };
      }]}],
    },
  },
})) { /* ... */ }

// completion gate lives OUTSIDE the model:
const verified = await run("pytest tests/affected -q");
if (verified.exitCode !== 0) reject("agent claimed done; verification failed");
```

Two load-bearing ideas: output shaping lives in a hook (never the model's own discipline), and "done" is an exit code the harness observes.

Compaction template sections that earn their bytes: task + explicit non-goals; constraints; **decisions with reasons**; **dead ends with evidence** ("timeout bump masked it locally, still flaked in CI — do not retry"); current state; next step.

## Verification and self-check

- Regression-run the same 10–20 tasks per harness change; compare end-to-end success/cost/steps, not transcript aesthetics.
- Audit five random transcripts per iteration for the big four; read the diffs, not the agent's summaries of them.
- Red-team the sandbox per design change: write outside workspace, reach a non-allowlisted domain, read a secret, persist across sessions — all four must fail.
- Ship (behind review) when held-out verification passes on most tasks and false-success ≈ 0; past that, invest in the verifier and task distribution, not prompt polish.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)
- Probed 12 claims: 10 baseline (cut/compressed), 1 partial, 1 delta (expanded).
- Delta: benchmark calibration — Opus says frontier ≈70–80% on SWE-bench Verified; reality mid-2026 is ~90–95% (saturating), Pro ~80%, Terminal-Bench with headroom.
- Opus cold nails: minimal tool set + tool-earning test, head+tail truncation with counts, context ladder, compaction resume-test, deceptive-green defenses, sandbox primitives (Landlock/seccomp/bubblewrap/Seatbelt), 0.95^10 math, error-loop root-cause ordering, AGENTS.md content. All compressed to anchors; SDK-specific hook/permission-mode shapes kept as reference.
