---
name: git-mastery
description: Load when doing nontrivial git work — rewriting/cleaning history (interactive rebase, fixup, splitting commits), recovering "lost" work (botched rebase, bad reset, force-push), choosing rebase vs merge, running git bisect, setting up worktrees, taming a monorepo (sparse-checkout, partial clone, maintenance), deciding submodule vs subtree, or splitting a big branch into stacked PRs.
---

# Git Mastery

## Core mental model

- **Git is an immutable content-addressable database with mutable pointers on top.** Objects (blob = file contents, tree = directory listing, commit = tree + parents + message) are keyed by their hash and never change. Everything scary — rebase, amend, reset, filter — only ever *creates new objects and moves pointers*. Nothing rewrites; "rewriting history" means "writing a parallel history and pointing the branch label at it."
- **A branch is a 41-byte file containing a commit hash.** `git branch foo` costs nothing and risks nothing. Once you internalize this, you stop fearing operations: before anything risky, `git branch backup` gives you a free, perfect undo point. Detached HEAD is not an error state; it's just HEAD pointing at a commit instead of a branch name.
- **Committed work is almost never lost.** Every commit you ever made — including "destroyed" ones — sits in the object store for ~90 days (30 for unreachable), findable via `git reflog`. Even *uncommitted-but-staged* content survives as dangling blobs (`git fsck --lost-found`). The only truly unrecoverable states: never-added working-tree changes wiped by `checkout -- .`/`reset --hard`, and stash entries after `stash drop` + expiry. Calibrate your caution accordingly: commit early and often, squash later — commits are the safety net, not the deliverable.
- **A commit is a snapshot, not a diff.** Diffs are computed on demand between snapshots. This is why rebase/cherry-pick can "replay" commits onto new bases (git re-derives the patch), why identical trees dedupe for free, and why `git log -p` of a merge looks empty by default (a merge has two parents; there is no single diff).
- **The history you publish is a communication artifact, not a lab notebook.** Local history is scratch space — rewrite freely. Published history on shared branches is an API other people's work depends on — never rewrite it. Every judgment call about rebase, squash, and force-push reduces to: *who has built on these commits?*

## Decision framework: rebase vs merge

Ask, in order:

1. **Has anyone else based work on these commits?** (Pushed to a branch teammates pull? Someone's PR targets it?) If yes → never rebase them. This single question settles 80% of cases. Force-pushing a shared branch converts your cleanup into everyone else's conflict-resolution session.
2. **Is this a private feature branch being freshened against main?** → rebase (`git pull --rebase`, or `git rebase main`). Merge commits from main *into* a feature branch ("back-merges") create a rope-ladder history that makes `log`, `bisect`, and revert harder, and they bury your actual changes.
3. **Is this a feature branch landing into main?** → your platform's squash-merge or rebase-merge if the branch is one logical change; a true merge commit (`--no-ff`) if the branch is a curated sequence of commits worth preserving (then the merge commit marks the feature boundary and `git revert -m 1 <merge>` can back the whole feature out atomically).
4. **Release/long-lived branches** (release-1.x receiving hotfixes) → merge only, and prefer cherry-pick with `-x` (records the source hash) for moving individual fixes across release lines. Rebasing a release branch destroys the audit trail that release branches exist to provide.
5. **Force-push discipline:** on your own branches, always `git push --force-with-lease` (refuses if the remote moved since your last fetch — i.e., if a teammate or CI pushed). Bare `--force` is only correct when you have *just* fetched and consciously intend to discard the remote state. As of 2026 there's still no reason to ever type bare `--force` in a script.

What changes the answer: a "private" branch that CI or a deploy system tracks by SHA is effectively shared. A "shared" branch with exactly one other consumer whom you can ping is effectively private.

## History surgery: the toolkit

- **Interactive rebase verbs, in order of how often experts actually use them:** `fixup`/`squash` (merge into previous), `reword` (message only — don't `edit` just to fix a typo), `drop`, reorder-by-moving-lines, `edit` (stop to change content), `break` (stop *between* commits to test). `fixup -C <sha>` keeps the fixup's message instead. Rarely-known: `exec make test` lines between picks turn a rebase into a per-commit CI run — `git rebase -i --exec 'make test' main` verifies every commit in the stack builds, which is what makes the stack bisectable later.
- **Splitting a commit** (the operation people avoid because they've never drilled it): `git rebase -i`, mark the commit `edit`; at the stop, `git reset HEAD^` (un-commits, keeps working tree), then build the pieces with `git add -p` + `git commit` repeatedly, then `git rebase --continue`. For splitting the *most recent* commit, skip the rebase: `git reset HEAD^` and recommit in pieces.
- **rerere (reuse recorded resolution):** `git config rerere.enabled true` today, on every machine. Git records each conflict hunk's resolution and silently replays it the next time the identical conflict appears — which is exactly what happens when you rebase a long-lived branch repeatedly, carry a patch across releases, or restack a PR stack. Without it you re-resolve the same conflicts weekly and each re-resolution is a fresh chance to botch one. Check what it did with `git rerere diff`; if it recorded a *wrong* resolution, `git rerere forget <path>` before re-resolving.
- **Removing a secret or huge blob from history:** `git filter-repo` (the tool the git project itself points to; `filter-branch` is deprecated, slow, and footgun-laden), e.g. `git filter-repo --invert-paths --path secrets.env` or `--strip-blobs-bigger-than 10M`. It rewrites every descendant hash — this is a coordinated event (everyone re-clones, all open PRs die), not a quiet fix. And rotate the secret regardless: history rewriting doesn't un-leak anything already fetched by forks, mirrors, or scrapers.
- **Server-side / worktree-less plumbing** worth knowing exists: `git merge-tree` performs *real* merges (rename detection, recursive bases) with no worktree or index — since Git 2.38 this is how platforms do server-side merges, and it's the fast way to answer "will these two branches conflict?" without touching your checkout. `git replay` (Git 2.44+) is an in-memory rebase that works in bare repos and can replay multiple branches at once.

## Worktrees: parallel work without stashing

- `git worktree add ../repo-hotfix hotfix-branch` gives a second working directory sharing the same object store and refs — near-instant, no duplicate storage (vs. a second clone which duplicates objects and has *separate* refs you must push/pull between).
- Reach for a worktree, not `git stash`, whenever a context switch will outlive your short-term memory: reviewing a PR while mid-feature, running a long test suite on one branch while editing another, keeping a `git bisect` session isolated so your editor and dev server never see the checkout churn, or running multiple coding agents on the same repo in parallel. Stash is for 5-minute interruptions; stashes older than a day are where changes go to be forgotten.
- Mechanics that bite: one branch per worktree at a time (checkout of an already-checked-out branch fails — use a second branch or detached HEAD); remove with `git worktree remove <path>` and clean stale entries with `git worktree prune`; `git worktree list` when you've lost track. Long-lived worktrees pin their commits against `gc`.

## Decision framework: splitting work into reviewable units

- The unit of review is *one reversible decision*: reviewers should be able to say yes/no to each PR independently. Ask: "if PR 3 is rejected, do PRs 1–2 still stand alone?" If not, your split is cosmetic, not structural.
- Standard decomposition order that almost always works: (1) pure refactor/rename with zero behavior change, (2) new code paths behind a flag or unused, (3) the wiring/flip, (4) cleanup of the old path. Mechanical changes (formatting, codemod) always ride alone — a 4,000-line rename PR takes 5 minutes to review *if nothing else is in it*.
- Commit messages: subject ≤ ~50 chars, imperative ("Add X", not "Added"/"Adds"), body explains *why* and what alternatives were rejected — the diff already shows *what*. The body is where you talk to the person running `git blame` in three years. Reference issues; note deliberate weirdness ("intentionally O(n²): n ≤ 8").
- **Stacked-PR tooling, as of mid-2026:** the landscape has consolidated. [Graphite](https://graphite.dev) is the dominant commercial CLI+web option (auto-restack, merge queue). GitHub itself shipped **native stacked PRs** via the `gh-stack` CLI extension in **private preview (April 2026, waitlist at gh.io/stacksbeta)** — `gh stack sync` cascades rebases across the stack, the PR UI gets a stack map, and branch protection evaluates against the final target; expect this to become the default answer once it GAs. Jujutsu (`jj`) is the credible git-replacement track (change-based identity; `jj-spr` bridges it to GitHub). Still alive and useful: `ghstack` (Meta), `spr`/`stack-pr` (Modular), `git-machete`. `git-branchless` remains alpha; momentum moved to jj. **With zero extra tooling:** plain git handles small stacks fine since 2.38 — branch-per-layer plus `git rebase main --update-refs` (set `rebase.updateRefs=true`) moves the whole stack in one rebase, then force-push each branch with `--force-with-lease`.

## Decision framework: submodules vs subtree vs neither

1. First ask: **can this just be a package?** A registry dependency (npm/PyPI/crate) with versioning beats both. Vendoring via git is the fallback, not the default.
2. **Default to avoiding submodules.** Their failure modes are structural: detached-HEAD checkouts by default, clones silently missing content without `--recurse-submodules`, the superproject pins a SHA so every dependency bump is a commit *in two repos*, and `git status` lies to teammates who forgot `git submodule update`. Every team has a "submodules broke my checkout" week.
3. **Submodules are right when:** the subproject is *independently developed and versioned*, you need an exact-SHA pin for compliance/reproducibility, the history is huge and you don't want it in your objects, or you must not copy the code in (license, size). Mitigate with `git config submodule.recurse true` and `git clone --recurse-submodules` in docs/CI.
4. **`git subtree` is right when:** you want the code *in* your repo (one clone, one commit, greppable, no extra workflow for teammates) and only occasionally sync upstream. Cost: `subtree pull/push` are clunky and merge history gets noisy. For "vendor it and rarely update," subtree — or honestly a plain copy plus a `VENDORED_FROM` note — wins.
5. Tiebreaker: **who touches it?** If everyone on the team must build against it daily → subtree/vendor (zero per-person setup). If only a release engineer updates it quarterly → submodule is tolerable.

## How an expert thinks through this: "the rebase ate my commits"

Teammate: "I rebased, resolved conflicts for an hour, then it looked wrong so I `rebase --abort`ed — no wait, I `reset --hard`ed something — and now two days of commits are gone from my branch."

Internal monologue:

1. *First, stop the bleeding.* Tell them: no more writing commands, especially no `gc`, no more `reset --hard`. Everything committed still exists; we're doing archaeology, not surgery.
2. *Where would the commits be?* They were committed → they're objects in `.git/objects` → reachable from *some* reflog entry. `git reflog` (HEAD's log) first. **Rejected:** searching `git log --all` — the commits are unreachable from any current ref, so `--all` won't show them. **Rejected:** `git fsck --lost-found` as the first move — it works but returns an unordered pile of hashes; the reflog gives me *named, timestamped, ordered* history of where HEAD pointed. fsck is the fallback if reflogs were nuked.
3. `git reflog` shows: `reset: moving to origin/feature`, before that `rebase (finish)`, before that a run of `rebase (pick)` entries, and before *that* — `commit: add retry logic`. There. The pre-rebase tip. Also check the branch's own reflog, `git reflog show feature` — branch reflogs survive even when HEAD's log is cluttered by the rebase's own churn.
4. *Which state do they actually want?* Two candidates: the pre-rebase original (`feature@{1.day.ago}` or the hash before `rebase (start)`) and the post-conflict-resolution rebased version (the hash at `rebase (finish)`). The hour of conflict resolution is real work — prefer the *finished rebase* tip if it exists. Inspect both: `git log --oneline <hash>` and `git diff <hash1> <hash2>`.
5. *Restore without new risk:* `git branch rescue <finish-hash>` — create a pointer, don't move anything yet. Diff `rescue` against `origin/feature` to confirm contents, then `git checkout feature && git reset --hard rescue`. **Rejected:** cherry-picking the lost commits one by one onto the current branch — more steps, more conflict surface, and it loses committer dates and any merge structure; moving a pointer to an existing good state is strictly better when the good state exists as a commit.
6. *Why did the rebase "look wrong" in the first place?* Don't skip this — usually a conflict resolved backwards, or an upstream that itself had been force-pushed. Check `git range-diff origin/main feature@{1} rescue` to see exactly what the rebase changed per-commit. If the same conflicts will recur next rebase, enable `git config rerere.enabled true` now so this hour of resolution gets recorded and replayed automatically next time.

Stopping rule: recovery is done when `git diff rescue expected-state` is empty and the teammate confirms the working tree builds — not when "the log looks about right."

## Failure modes & pitfalls

- **`git pull` into a rebase-in-progress or with default merge config on a rebased branch.** Symptom: "Your branch and origin/X have diverged" after you rebased, then `git pull` creates a merge of your branch with *its own pre-rebase self*, resurrecting every commit you just cleaned. After rebasing an already-pushed branch, the only correct next command is `git push --force-with-lease`, never `git pull`. Set `pull.rebase=true` and `pull.ff=only` mindsets: if pull wants to merge, stop and look.
- **Resolving a rebase conflict with the sides swapped.** During `rebase`, "ours" is the branch you're rebasing *onto* (upstream) and "theirs" is *your own commit* — the reverse of merge. People run `git checkout --ours file` intending to keep their work and silently delete it. Verify with `git log -1 REBASE_HEAD` (the commit being replayed) before choosing sides.
- **`--force` instead of `--force-with-lease`, or `--force-with-lease` after a `git fetch` you didn't inspect.** The lease compares against your remote-tracking ref; if you fetch and then force-push without looking, the lease is trivially satisfied and you can still stomp a teammate's push. Fetch, *read* `git log HEAD..origin/branch`, then push.
- **Rebasing/squashing commits that a stacked branch is based on, without `--update-refs`.** The child branches still point at the old commits; the next rebase of each child replays already-landed changes and conflicts with themselves. Fix the workflow: `git config rebase.updateRefs true` (Git 2.38+) so interactive rebase moves every branch label in the stack. If you're already in the broken state, repair with `git rebase --onto new-parent old-parent child`.
- **`git rebase --onto` with the operands reversed.** Signature is `--onto <newbase> <upstream> [<branch>]`: commits in `<upstream>..<branch>` get replayed onto `<newbase>`. Getting `<newbase>`/`<upstream>` backwards transplants the wrong range somewhere surprising. Sanity-check first: `git log --oneline upstream..branch` must list exactly the commits you intend to move.
- **fixup workflow done manually.** Editing an earlier commit by making a change, running interactive rebase, and hand-reordering is error-prone. Use the machinery: `git commit --fixup=<sha>` (or `--fixup=amend:<sha>` to also edit the message, `--squash=<sha>`), then `git rebase -i --autosquash main`. Set `rebase.autoSquash=true`. For "fix whichever commit last touched these lines," `git absorb` automates target selection.
- **Believing a stash or reflog protects you across `git stash drop` / branch deletion forever.** Reflog entries expire (default 90 days reachable / 30 unreachable, then `gc` collects). Deleted-branch commits are findable only via HEAD's reflog or fsck (a deleted branch's own reflog is deleted with it). Recover *promptly*; for a just-dropped stash, `git fsck --unreachable | grep commit` and inspect candidates.
- **Running bisect with a test script that returns the wrong exit codes.** `git bisect run` treats exit 0 = good, 1–124/126/127 = bad, **125 = skip this commit** (unbuildable). A script that exits 2 on "can't build" marks buildless commits *bad* and bisect converges on an innocent build-break commit. Also: if the failure is flaky, loop the test inside the script until statistical confidence, or bisect converges on noise.
- **Bisecting when good/bad have inverted semantics** (e.g., hunting when a *fix* appeared). Don't mentally invert good/bad — you'll slip. Use `git bisect start --term-new=fixed --term-old=broken`.
- **Treating worktrees as copies.** `git worktree add ../proj-review pr-branch` shares one object store and one config across worktrees — that's the point (cheap, instant, no re-clone) — but a branch can be checked out in only one worktree at a time, and `git worktree remove` (not `rm -rf`) is required or you leak administrative entries (`git worktree prune` cleans up after a manual delete). Hooks and `.git/info/exclude` are shared; per-worktree config needs `extensions.worktreeConfig`.
- **Cloning a monorepo with `--depth=1` when you meant partial clone.** Shallow clones break `log`, `blame`, `bisect`, and make later fetches expensive and semantics weird (`--unshallow` refetches everything). Partial clone (`--filter=blob:none`) keeps *full commit history* and lazily fetches file contents on demand — as of 2026 it's mature and the right default for large repos; shallow is only for throwaway CI checkouts that will never fetch again.
- **Writing sparse-checkout patterns in non-cone mode by habit.** Cone mode (directory-based) is the default since Git 2.37 and is the only mode with acceptable performance at scale (non-cone gitignore-style matching is quadratic-ish and effectively deprecated). Use `git sparse-checkout set dir1 dir2` and add `--sparse-index` (or `index.sparse=true`) so `status`/`add` scale with your cone, not the repo.
- **Letting `git gc --auto` ambush interactive work in a big repo.** Enable scheduled background maintenance instead: `git maintenance start` (Git 2.30+) registers hourly `prefetch`, `commit-graph`, incremental repack. Or run `scalar register` (in core git since 2.38) which sets partial-clone-friendly defaults, maintenance, and FS monitor in one shot. Symptom you need this: first `git status` of the day takes 10+ seconds.
- **Client-side hooks as a security or policy boundary.** Hooks don't clone with the repo (by design — arbitrary code execution), so `pre-commit` linting is a convenience, not enforcement; anyone can `--no-verify`. Real policy lives server-side (`pre-receive`/`update` hooks, or platform branch protection). Distribute client hooks via `core.hooksPath` pointed at an in-repo directory or a manager like `pre-commit` — never by asking people to copy files into `.git/hooks`.
- **Amending or rebasing a commit that's already part of an open PR others reviewed, without `range-diff`.** Reviewers can't see what changed between force-pushes. Post `git range-diff origin/main old-tip new-tip` output (or rely on the platform's force-push compare) so re-review is a diff-of-diffs, not a full re-read.
- **Reverting a merge commit, then expecting to re-merge the branch later.** `git revert -m 1 <merge>` reverts the *content* but the merge itself stays in history — a later re-merge of the same branch brings in *nothing* (git sees those commits as already merged). To re-land the feature you must revert the revert, or rebase the branch into new commits first. Decide at revert time whether the feature will return, and leave a note in the revert message.
- **Cherry-picking across release branches without `-x`.** Six months later nobody can tell which fixes made it into release-1.x. `git cherry-pick -x` appends `(cherry picked from commit <sha>)`, making cross-branch audit greppable. Corollary: check whether a fix is present with `git branch --contains <sha>` (exact) and `git cherry -v release-1.x main` (patch-equivalence, catches cherry-picks with different hashes).
- **Letting a formatting/codemod commit destroy `git blame`.** After a repo-wide reformat, every line blames to the reformat. Record such commits in `.git-blame-ignore-revs` and set `git config blame.ignoreRevsFile .git-blame-ignore-revs` (GitHub's blame view reads the same file). Do this in the reformat PR itself, not after the complaints.
- **`git stash pop` into a conflict.** On conflict, pop applies the changes *and keeps the stash*, leaving a confusing half-state; people then re-pop and double-apply. Prefer `git stash apply`, verify, then `git stash drop`. Better: stop using stash for anything nontrivial (see worktrees) — and never carry >2 stashes; `stash@{3}` from last sprint is unlabeled, context-free, and half of it already landed some other way.
- **A hook that "doesn't run."** Checklist in order: is the file executable (`chmod +x`)? Named exactly right with no extension (`pre-commit`, not `pre-commit.sh`)? Is `core.hooksPath` set (then `.git/hooks/` is ignored entirely)? Did the invoking command bypass hooks (`--no-verify`, some GUI clients, `git commit --amend` still runs pre-commit but rebase's re-commits don't)? Hooks get a sanitized environment — a hook that works in your shell but not from the IDE is missing PATH entries.
- **Expecting GitHub to accept a SHA-256 repo.** As of 2026, Git 2.51+ makes SHA-256 the default object format for *new local* repos and Git 3.0 (targeted late 2026) doubles down, but GitHub still doesn't host SHA-256 repositories and there's no in-place migration. For anything pushed to GitHub, stay SHA-1 (`git init --object-format=sha1` if your git defaults to sha256); check your hosting before adopting reftable/`sha256` defaults.

## Worked micro-examples

**1. Fully automated bisect of a performance regression (flaky-safe):**

```bash
cat > /tmp/bisect-test.sh <<'EOF'
#!/bin/bash
make -j8 build || exit 125          # can't build => skip, NOT bad
for i in 1 2 3; do                  # 3 runs to beat noise
  t=$(./bench --json | jq .p50_ms)
  (( $(echo "$t > 180" | bc) )) && exit 1   # regression threshold
done
exit 0
EOF
chmod +x /tmp/bisect-test.sh
git bisect start HEAD v2.14.0
git bisect run /tmp/bisect-test.sh   # walks ~log2(N) commits unattended
git bisect log > /tmp/bisect.log     # keep the transcript, then:
git bisect reset
```

**2. Turn one messy 40-commit branch into a 3-PR stack (plain git, 2.38+):**

```bash
git config rebase.updateRefs true
git rebase -i main                       # squash/fixup/reorder into 3 clean commits
git branch step1 HEAD~2
git branch step2 HEAD~1
git branch step3 HEAD
git push -u origin step1 step2 step3     # open PR: step1->main, step2->step1, step3->step2
# after review feedback on step1:
git checkout step3
git commit --fixup=$(git rev-parse step1) # or: git absorb
git rebase -i --autosquash main           # --update-refs moves step1/step2 labels too
git push --force-with-lease origin step1 step2 step3
```

(With Graphite this is `gt create`/`gt restack`/`gt submit`; with GitHub's gh-stack preview, `gh stack sync` replaces the rebase+push dance.)

**3. Monorepo onboarding recipe (partial clone + cone sparse-checkout + maintenance):**

```bash
git clone --filter=blob:none --no-checkout https://host/bigrepo.git
cd bigrepo
git sparse-checkout set --sparse-index services/payments libs/common
git checkout main
git maintenance start        # background prefetch, commit-graph, incremental repack
# later, widen scope on demand:
git sparse-checkout add services/billing
```

**4. Recovery recipes (memorize the shapes, not the hashes):**

```bash
git reflog                                  # every place HEAD has been
git reflog show mybranch                    # every place the branch pointed
git checkout -b rescue HEAD@{5}             # resurrect a pre-disaster state
git branch rescue $(git rev-parse ORIG_HEAD) # ORIG_HEAD = tip before last rebase/merge/reset
git fsck --unreachable | grep commit        # last resort: orphaned commits (dropped stash)
git range-diff main old-tip new-tip         # audit exactly what a rewrite changed
```

## Verification & self-check

- **Before any history rewrite:** `git branch backup-$(date +%s)` (free) and confirm `git status` is clean. After the rewrite: `git range-diff main backup-* HEAD` — every hunk shown must be a change you *intended*; unexplained hunks mean a conflict was resolved wrong.
- **Before any force-push:** `git fetch` then `git log --oneline HEAD..origin/$(git branch --show-current)` — must be empty or contain only commits you're knowingly discarding. Then `--force-with-lease`, never bare.
- **Content-preservation check after squash/reorder-only rebases:** `git diff backup HEAD` must be empty — reordering and squashing change history, not the final tree. Any diff means you dropped or mangled a change.
- **After a "recovery":** the rescued state must build/test, and `git diff rescue <what-user-expected>` must be explainable line by line. Recovery of the wrong reflog entry looks identical to success until someone notices missing work next week.
- **Before recommending version-gated features** (`--update-refs` 2.38, `git replay` 2.44, `merge-tree` real merges 2.38, `maintenance` 2.30, cone-default 2.37): check `git --version` in the target environment; CI images and old LTS distros routinely ship git several years stale.
- **Stopping rules:** history polishing is done when each commit builds independently and the sequence tells the review story — further squashing is procrastination. Bisect is done when the culprit commit *explains the symptom* (if it plausibly can't, believe the bisect but ask what it perturbs — or suspect a flaky test poisoned the run; check `git bisect log` for skips). Monorepo tuning is done when `git status` is sub-second and clone time is acceptable — chasing further wins past that is waste.
