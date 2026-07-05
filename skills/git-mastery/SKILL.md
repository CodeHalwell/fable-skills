---
name: git-mastery
description: Load when doing nontrivial git work — rewriting/cleaning history (interactive rebase, fixup, splitting commits), recovering "lost" work (botched rebase, bad reset, force-push), choosing rebase vs merge, running git bisect, setting up worktrees, taming a monorepo (sparse-checkout, partial clone, maintenance), deciding submodule vs subtree, or splitting a big branch into stacked PRs.
---

# Git Mastery

Compact sheet. Standard expertise — reflog/fsck recovery, rebase ours/theirs inversion, bisect exit codes (125 = skip), `--update-refs` (2.38), `--force-with-lease` and its fetch loophole, `filter-repo` + rotate, pickaxe/`blame -w -C -C -C`/`merge-tree --write-tree`/`range-diff`, partial clone vs shallow, cone sparse-checkout, `maintenance start`, revert-of-merge semantics, rerere, `.git-blame-ignore-revs` — is assumed known and appears only as anchors. The expanded sections are 2026-state facts and judgment calls where the standard answer is incomplete.

## Anchors (apply without re-derivation)

- Everything committed survives ~30–90 days unreferenced; `git branch backup` before any rewrite is free. `ORIG_HEAD` = tip before the last rebase/merge/reset.
- Rebase vs merge reduces to: who has built on these commits? Private freshening → rebase; landing → squash or `--no-ff` per whether the sequence is curated; release branches → merge/cherry-pick `-x` only. A "private" branch that CI tracks by SHA is shared.
- `--force-with-lease` compares your remote-tracking ref — any fetch (IDE auto-fetch included) trivially satisfies it. Fetch, *read* `git log HEAD..origin/branch`, then push; add `--force-if-includes` (2.30+) for the extra reflog check.
- Rebase conflicts: "ours" = upstream, "theirs" = your commit (inverse of merge). Verify with `git log -1 REBASE_HEAD` before picking sides.
- `bisect run`: 0 good, 1–124 bad, **125 skip** — `make || exit 125`, or build failures convict an innocent commit; loop flaky tests to confidence; script must test the symptom, not "tests pass"; `--first-parent` (2.29+) for merge-heavy history; inverted semantics → `--term-new=fixed --term-old=broken`, never mental inversion.
- Fixups: `git commit --fixup=<sha>` + `rebase -i --autosquash`; `git absorb` picks targets. Splitting: mark `edit`, `git reset HEAD^`, recommit with `add -p`. `exec make test` lines make a stack bisectable.
- Stacks in plain git: `rebase.updateRefs=true` (2.38+) moves every branch label in one rebase; repair a broken stack with `rebase --onto new-parent old-parent child` (signature: `--onto <newbase> <upstream> [<branch>]` — sanity-check `git log upstream..branch` first).
- After rebasing a pushed branch, the only correct next command is `--force-with-lease` — `git pull` merges the branch with its own pre-rebase self and resurrects everything.
- Worktrees over stash for anything outliving short-term memory; one branch per worktree; `worktree remove`, not `rm -rf`. Stash: `apply`-verify-`drop`, never `pop` into a conflict.
- Monorepo: `--filter=blob:none` (full history, lazy blobs) over `--depth=1` (breaks log/blame/bisect); cone-mode sparse-checkout + `--sparse-index`; `git maintenance start` or `scalar register`.
- Submodule vs subtree: first ask "can it be a package?"; submodules only for independently-versioned, exact-SHA-pinned, rarely-touched deps; subtree/vendor when everyone builds against it daily.
- Client hooks are convenience, not policy (`--no-verify` exists); enforcement is server-side or branch protection; distribute via `core.hooksPath`.

## 2026-state facts (verify `git --version` and hosting before relying)

- **Stacked-PR tooling has consolidated:** Graphite is the dominant commercial CLI/web option. **GitHub shipped native stacked PRs via the `gh-stack` CLI extension in private preview (April 2026, waitlist gh.io/stacksbeta)** — `gh stack sync` cascades restacks, the PR UI gains a stack map, branch protection evaluates against the final target; expect it to become the default answer at GA. Jujutsu (`jj`) is the credible git-replacement track (change-based identity; `jj-spr` bridges to GitHub); `ghstack`/`spr`/`git-machete` alive; `git-branchless` stalled — momentum moved to jj. Baseline models report "GitHub has nothing native" — that's now stale.
- **SHA-256:** still NOT the default — Git 2.51's release notes only *plan* the flip ("Flipping the default hash function to SHA-256 at Git 3.0 boundary is planned", with reftable also becoming default at 3.0, targeted late 2026); current releases only prepare/test it behind a breaking-changes build gate. Don't add SHA-1 overrides preemptively. When 3.0 lands, note **GitHub still doesn't host SHA-256 repos** and there's no in-place migration or interop bridge — check hosting before adopting sha256/reftable.
- Version gates worth re-checking in CI images (routinely years stale): `--update-refs` 2.38, real `merge-tree` 2.38, `git replay` 2.44, `maintenance` 2.30, cone-default 2.37, `--force-if-includes` 2.30.

## Judgment call: botched-rebase recovery (where baseline advice diverges)

Both candidates exist in the reflog: the pre-rebase tip and the `rebase (finish)` tip. The reflexive answer is "restore the pre-rebase original — the conflict resolution is unverified." Don't discard an hour of resolution work reflexively: anchor *both* (`git branch rescue-orig <pre>`, `rescue-done <finish>`), then **audit the resolution instead of guessing** — `git range-diff origin/main rescue-orig rescue-done` shows exactly what the rebase changed per commit; unexplained hunks = a conflict resolved wrong. Prefer the finished tip when the range-diff is clean and it builds; fall back to the original and redo the rebase (with `rerere.enabled true`, so the hour of resolutions replays automatically) when it isn't. Restore by pointing the branch at an existing good commit — never cherry-pick the pieces back one by one. Also check the branch's own reflog (`git reflog show feature`) — HEAD's log is cluttered by the rebase's own churn; note a *deleted* branch's reflog dies with it (HEAD reflog or fsck only).

## Verification / self-check

- Before rewrite: clean status + backup branch. After: `range-diff` explains every hunk; squash/reorder-only rebases must show empty `git diff backup HEAD`.
- Before force-push: `git log HEAD..origin/<branch>` empty or knowingly discarded.
- After recovery: rescued state builds and `git diff rescue expected` is explainable line by line — the wrong reflog entry looks identical to success until next week.
- Stopping rules: polishing ends when each commit builds independently and the sequence tells the review story; bisect ends when the culprit *explains the symptom* (else suspect flaky poisoning — check `bisect log` for skips); monorepo tuning ends at sub-second `status`.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)
- Probed 14 claims: 11 baseline (compressed to anchors), 1 partial (recovery-candidate preference — sharpened into the range-diff-audit rule), 2 delta (2026 stacked-PR state incl. GitHub's native gh-stack preview; SHA-256 default in Git 2.51+/3.0 timeline vs GitHub hosting).
- Opus cold reproduced: reflog/fsck recovery mechanics, ours/theirs inversion, bisect 125 semantics, `--update-refs` with exact version, the force-with-lease fetch loophole incl. `--force-if-includes`, filter-repo caveats, all interrogation commands, partial-clone/cone/maintenance setup, revert-of-merge, rerere, blame-ignore-revs.
- Biggest baseline gaps: post-cutoff tooling/hosting facts (gh-stack, SHA-256 defaults) — this skill's main reason to exist; keep re-verifying them.
