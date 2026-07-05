---
name: code-generation-with-llms
description: Engineering with AI coding agents — writing specs that work first-try, curating CLAUDE.md/AGENTS.md context, decomposing tasks for delegation, choosing delegate-vs-write-myself, reviewing AI-generated diffs, and agentic workflow patterns (subagents, worktrees, headless fan-out, CI agents). Load when planning how to use a coding agent on a task, setting up a repo for agents, reviewing AI-written code, or diagnosing why agent output keeps missing the mark.
---

# Code Generation with LLMs

## Core mental model (anchors — the field's consensus; the discipline is living by it)

1. The prompt is the design doc; effort moves upstream (spec) and downstream (verification) — the typing in the middle is the cheap part.
2. Recurring output flaws are missing context, not model limitation: fix inputs (memory files, named exemplars) before prompting harder or louder.
3. Shape tasks so verification is mechanical *before* generation starts; task size is bounded by verification granularity, not model capability — an unreviewable 3,000-line diff is a decomposition failure.
4. Completion claims require artifacts (the command run, its output, the diff); a fresh context reviews the writer's work — the writing session grading itself is motivated reasoning.
5. Corrections typed into chat fix one session; the same sentence in repo memory/a lint rule fixes every future session. Two failed corrections → restart fresh with a better prompt (the transcript of failures actively teaches the wrong pattern).
6. Delegate when success is mechanically specifiable and the pattern is established; keep novel architecture, security-critical logic, and anything where you can't state "correct" — if you can't write the spec, you can't review the output either.
7. AI code review hunts a specific defect profile human instincts miss: plausible-but-wrong APIs, stale idioms, hallucinated dependencies (slopsquatting — attackers register commonly-hallucinated names), silent requirement drops (review against the spec as a checklist; absence is invisible in a diff), test-shaped non-tests (flip test non-negotiable), weakened verification to reach green (any test/lint/CI-config edit the task didn't demand is guilty until explained), missing authz on new endpoints, over-engineering, scope drift. Fluency is not a correctness signal; review AI code slower per line, not faster.

## The corrections (measured Opus misses and post-cutoff mechanics)

**Claude Code keeps auto memory — Opus-class models assert it doesn't.** As of 2026, Claude Code maintains per-repository notes it writes for itself (`~/.claude/projects/<project>/memory/`, with a `MEMORY.md` index loaded each session), on by default — in addition to your curated CLAUDE.md. The baseline model claims Claude Code's memory is "file-based and explicit, nothing accumulated behind your back"; that's outdated. Consequence: machine-written memory compounds like curated memory *and can entrench a mistaken inference*. Skim it occasionally (`/memory`), correct wrong learnings, promote good ones into the shared CLAUDE.md so the team inherits them.

**The AGENTS.md/CLAUDE.md facts, precisely (mid-2026).** AGENTS.md is the cross-tool standard — originated by OpenAI, now stewarded by the Linux Foundation's Agentic AI Foundation, read natively by 20+ tools, shipped by 60k+ OSS repos; closest-file-wins in monorepos. Claude Code reads CLAUDE.md and does *not* natively read AGENTS.md — bridge with a CLAUDE.md containing `@AGENTS.md` (import syntax, resolves recursively up to four hops, Claude-specific lines below the import) or a plain symlink. Never maintain two divergent copies. Baseline models hedge on both the governance and the native-read question — in a mixed-tool repo, the unbridged half silently gets zero context.

**Scoped context loading beats one big file.** Beyond nested per-directory memory files: `.claude/rules/*.md` with `paths:` frontmatter globs load only when matching files are touched, and skills (`.claude/skills/<name>/SKILL.md`, `disable-model-invocation: true` for side-effectful ones, `$ARGUMENTS` for parameters) load procedures on demand. Any workflow you've pasted into chat twice belongs in a skill — versioned with the repo and usable by CI agents; putting procedures in CLAUDE.md bloats every session with steps that rarely apply. Keep the always-loaded file under ~200 lines; memory files fail by overgrowth, and a rule the agent already follows unprompted should be deleted.

**Memory is advisory; hooks are enforced.** "Never write to migrations/" belongs in a PreToolUse/Stop hook (`.claude/settings.json`), not prose; repeated violations of a memory rule are a signal to promote it to a hook or lint rule. Same logic gates unattended runs: block config/test-file writes structurally so make-the-tests-pass pressure has no outlet.

**Workflow mechanics worth knowing cold (verified 2026):** plan mode for explore→plan→implement (edit the plan — skim-approving converts the highest-leverage review point into pure latency); parallel work in git worktrees (`claude --worktree <name>`, subagent `isolation: worktree`, `.worktreeinclude` for gitignored env files); headless fan-out `claude -p "<task>" --allowedTools "Edit,Bash(npm test *)" --output-format json` — pilot on 2–3 files first, per-file commits as the unit of revert, and an explicit escape hatch ("if the file doesn't fit the rules, append to SKIPPED.md with a reason and change nothing") because without an honorable exit the agent improvises at file 87; Stop-hook gating for unattended done-conditions; CI agents via `anthropics/claude-code-action@v1` (@claude mentions, issue-to-PR), where repo memory matters double because no human fills context gaps.

**Reviewer feedback loops cause over-engineering.** An adversarial reviewer asked for gaps always finds some; chasing every "consider adding…" accretes defensive cruft. Instruct reviewers (human and agent) to report only correctness and stated-requirement gaps; treat suggestion-shaped findings as default-reject. Stopping rules: checks pass + spec checklist closes = stop iterating; three fresh-session failures with genuinely improved specs = write it yourself (the spec isn't statable yet).

## The expert walkthrough, compressed — 200-handler migration

Build the verifier first (lint rule asserting the new convention — the load-bearing artifact, review it hard) → pilot on 3 representative handlers in plan mode, review at write-it-yourself depth (this is spec debugging) → add the discovered wrinkles plus the SKIPPED.md escape hatch to the prompt → headless fan-out, one file per invocation, scoped tools, per-file commits → review by mechanism: lint covers 100%, human eyes on a random 10%, every SKIPPED.md entry, and every file where *tests changed* (test edits in a mechanical migration are a red flag). Rejected: one mega-session (unreviewable, pattern drifts by file 120), and "be careful with tricky handlers" (vague care doesn't survive contact; the escape hatch is the enforceable version).

## Verification / self-check (before accepting an AI change)

1. Mechanical gates green — run by you or CI, not taken from the transcript.
2. Flip test on new tests (break the implementation; tests must fail).
3. Spec checklist — every requirement located in the diff; out-of-scope items confirmed absent.
4. Diff hygiene — new deps verified real and needed; undemanded test/config/CI edits; suppression markers (`ignore`, `skip`, `eslint-disable`, bare `except`); comments claiming behavior the code lacks.
5. Comprehension bar for code you'll own: a hunk that resists explanation is where the bug is.

Review depth scales with blast radius, inversely with mechanical coverage: flip-tested leaf code stops at the checklist; authz/billing/migrations/concurrency get line-level scrutiny regardless of green.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)
- Probed 15 claims: 13 baseline (cut/compressed), 1 partial (sharpened), 1 delta (expanded).
- Biggest baseline gaps found:
  - Opus states outright that Claude Code keeps no automatic per-repo memory — wrong as of 2026 (auto memory at `~/.claude/projects/<project>/memory/`, on by default); the audit-machine-memory guidance is this skill's clearest reason to exist.
  - Hedges on AGENTS.md governance (Linux Foundation AAIF) and on Claude Code not reading AGENTS.md natively; lacks the `@AGENTS.md` import mechanics.
  - Tool-specific 2026 mechanics (rules with paths-globs, skills, --worktree/.worktreeinclude, claude-code-action@v1, headless flags) unavailable cold; all conceptual practice (defect profile, flip test, escape hatches, fan-out hygiene) is baseline.
