---
name: cli-tool-design
description: Load when designing or reviewing a command-line tool — argument/subcommand structure, flags vs positionals, exit codes, stdout/stderr separation, --json and color/TTY output handling, config precedence, error message wording, destructive-action safety, help text, choosing a CLI framework (clap, click/typer, cobra, commander/oclif/citty), or picking a distribution strategy per language.
---

# CLI Tool Design

## Core mental model

- **You are designing two interfaces at once: one for humans at a TTY, one for programs in a pipeline.** Every output decision (format, color, progress, prompts) must be made twice, and `isatty()` is the switch between them. Tools people love get both right; tools people curse pick one and force it on the other audience.
- **stdout is your return value; stderr is your logging.** stdout carries data — the thing another program would consume. Everything else (progress, warnings, hints, prompts, timing) goes to stderr. This single rule is what makes `yourtool | jq`, `yourtool > file`, and `yourtool | xargs` work. Violate it once and every downstream consumer must work around you forever.
- **Exit codes, stdout schema, flag names, and error formats are API surface.** Scripts will be written against all of them within a week of release. Renaming a flag or changing a default is a breaking change with the same weight as breaking a REST endpoint. Design them like you'd design a public API: deliberately, and stable-by-default.
- **The first 30 seconds decide adoption.** A user who types `tool`, `tool -h`, or `tool --help` and gets a usage line, a one-sentence description, and two copy-pasteable examples will stay. Errors are part of this: every error message is a micro-help-page for exactly the situation the user is in.
- **Guessable beats documented.** Follow POSIX/GNU conventions and the patterns of git/docker/kubectl not because they're optimal but because a million users' muscle memory is your free UX budget. Innovate in what your tool does, not in how flags parse. clig.dev (actively maintained, as of 2026) is the reference codification of this consensus — human-first, composable, conventional.
- **Safety is asymmetric.** An unnecessary confirmation costs one keystroke; a missing one costs someone's production database. Destructive actions default to safe (dry-run/confirm) and require explicit escalation (`--force`, typed resource names).

## Decision framework: shaping the argument surface

Ask in this order:

1. **Does the tool do one thing or several?** One verb (like `grep`, `curl`) → no subcommands, ever; don't build `mytool run` when `mytool` is the only action. Multiple verbs or multiple nouns → subcommands (`tool <noun> <verb>` like `git remote add`, or `tool <verb>` like `docker pull`). If you *might* grow verbs later, start with a subcommand even for one action — retrofitting subcommands onto a flat CLI breaks every existing invocation.
2. **For each parameter: is it the object of the verb, or a modifier of it?** The one or two things the command *acts on* are positional (`rm FILE`, `cp SRC DST`). Everything else is a flag. Reasoning chain: could a user guess the meaning from `tool cmd a b c` in shell history? Two positionals of the same type (`cp src dst`) is the maximum before order becomes a memory test — three or more, or two of different types, convert to flags. clig.dev is explicit here: prefer flags to args; the extra typing buys legibility.
3. **Is the parameter required?** Required flags are a smell but legal (`--env` on a deploy command is better forced than defaulted to prod). Never make a *positional* optional in the middle (`tool [maybe] definitely` is unparseable by humans).
4. **Short or long form?** Every flag gets a `--long-form`; only the 3–6 most-used get single-letter shorts. Honor near-universal letters: `-v` verbose, `-q` quiet, `-f` force, `-n` dry-run/count, `-o` output, `-h` help. If `-v` must mean version, you've left convention — prefer `-V`/`--version` (the git/clap convention) and keep `-v` for verbose.
5. **Boolean or valued?** Never design a flag with an *optional* value (`--color [WHEN]`): `--color auto` is ambiguous with `--color` followed by a positional `auto`. Either require the value (`--color=auto`, with `=` accepted) or split into paired booleans (`--color/--no-color`). Provide `--no-X` inverses for booleans whose default might be true, or config-file users can never override back to false from the command line.
6. **Where do flags go relative to positionals?** GNU-style parsers permute (`tool file --flag` works); strict POSIX stops at the first positional. If your tool *executes another command* (`tool run -- cmd args...`), stop parsing at `--` and say so in help — otherwise your flags collide with the child's.

## stdout/stderr and exit codes as API

- Data → stdout. Everything else → stderr. "Everything else" includes: progress bars, spinners, `Fetching...` logs, deprecation warnings, prompts, and the "Done! 3 files written" summary. Test: `tool ... > out.json` must yield valid JSON, and the user must still see progress.
- Exit code contract to publish in `--help` or man page:

| Code | Meaning | Notes |
|---|---|---|
| 0 | Success | Also for `--help`/`--version` when explicitly requested |
| 1 | Operation failed | Generic runtime failure |
| 2 | Usage error | Bad flags/args; print usage hint to stderr. (Matches bash builtins, grep's "trouble") |
| 3–63 | Yours to define | Document each; scripts will branch on them |
| 64–78 | sysexits.h range | Optional BSD convention (`EX_USAGE`=64, `EX_UNAVAILABLE`=69...); use it or the 3–63 space, not both |
| 126/127/128+n | Reserved by shells | Not-executable / not-found / killed-by-signal-n — never emit these yourself |

- **Distinguish "failed" from "found nothing."** grep's contract (0 = match, 1 = no match, 2 = error) is the model: if your tool searches/checks/diffs, "clean result of absence" deserves its own code so `if tool check; then` works in scripts. Decide this before v1 — flipping it later breaks every caller.
- Exit codes are one byte. Returning `-1` or an errno like `256` wraps mod 256 (256 → 0 → *success*). Map internal errors to your published codes explicitly.

## Output design: the isatty switchboard

- Decide per stream, at startup: `stdout_is_tty`, `stderr_is_tty`. Then:
  - **Color**: precedence (verified against no-color.org / force-color.org / the CLICOLOR spec, as of 2026): `NO_COLOR` set to any non-empty value → off, unconditionally, highest priority. Else `CLICOLOR_FORCE` non-empty-non-`0` or `FORCE_COLOR` non-empty → on even when piped. Else `CLICOLOR=0` or `TERM=dumb` → off. Else: on iff the stream is a TTY. Implement this once as a function (see micro-example) — most hand-rolled versions get the precedence wrong.
  - **Progress**: only when *stderr* is a TTY. When it isn't (CI logs), replace animated bars with occasional plain lines or nothing — a spinner in a CI log is 10,000 lines of `\r` garbage.
  - **Prompts**: only when *stdin and stderr* are both TTYs. Piped stdin means stdin is data, not a keyboard. Non-TTY + prompt needed → fail immediately with exit 2 and "use --yes to run non-interactively," never hang waiting for input CI will never send.
  - **Format**: human tables/summaries on a TTY are fine, but the moment output might be consumed by scripts, add `--json` (or `--format=json`). Emit newline-delimited JSON for streams of records, a single object otherwise. The `--json` schema is API: version it or only ever add fields. Do NOT auto-switch data *content* on isatty (ls-style column tricks are grandfathered; new tools that silently change fields when piped generate heisenbugs) — auto-switch decoration only, and make format explicit via the flag.
- Terminal hyperlinks (OSC 8) are broadly supported as of 2026 (iTerm2, Konsole, Alacritty, kitty, Windows Terminal; used by eza, bat, fd, gcc) and degrade gracefully to plain text — use them for URLs/file paths on TTYs, gate on the same color logic.
- Handle SIGPIPE: `tool | head -1` closes the pipe early. Python raises `BrokenPipeError` and prints a traceback on exit unless you catch it; Rust ignores SIGPIPE by default so writes return errors you must not `unwrap()`. Dying quietly on EPIPE is the correct behavior.

## Config precedence and environment

- Precedence, highest wins: **command-line flags > environment variables > project-local config > user config > system config > built-in defaults**. Users must be able to override any config-file setting for one invocation without editing files.
- Every config key gets a predictable env var: `MYTOOL_UPPER_SNAKE_KEY`. Every env var gets a flag. Generate all three from one schema or they will drift.
- Config file location: respect `$XDG_CONFIG_HOME` (default `~/.config/mytool/`) on Linux — and strongly consider using the same path on macOS rather than `~/Library/Application Support` (developers expect XDG for dev tools). Never scatter `~/.mytoolrc` for a new tool.
- **Secrets never travel in flags** — argv is visible in `ps` and shell history. Accept secrets via env var, file path (`--token-file`), or stdin prompt. Warn (stderr) if you detect a secret-shaped flag value.
- Print the *effective* config with provenance (`mytool config list` showing `registry = https://... (from ~/.config/mytool/config.toml)`); half of all support requests are "which config is it actually using."

## Error messages that teach

Every error answers three questions in order: **what failed, why, what to do next.**

```
error: cannot deploy to 'prod': branch 'fix/tmp' is not on main

  Deploys to prod must come from main (policy: PROD_BRANCH_ONLY).

  Try:  git switch main && mytool deploy prod
  Or:   mytool deploy staging   (no branch restriction)
```

- Name the offending *value* the user gave, quoted — `unknown format 'yamll'` beats `invalid format`. Add did-you-mean for close matches (every framework below has typo suggestions built in or one flag away).
- Rewrite dependency errors at your boundary. `ECONNREFUSED 127.0.0.1:5432` becomes `error: cannot reach the database at localhost:5432 — is it running? (start one with: mytool db up)`. Raw tracebacks/panics are for `--verbose` and bug reports, not default output.
- Usage errors (exit 2) print a one-line usage reminder plus `Try 'mytool --help'`, not the full help dump — dumping 80 lines of help after a typo buries the actual error.

## Destructive-action safety

- Classify commands: read-only, mutating-recoverable, destructive-irreversible. Only the third class gets friction.
- The ladder: `--dry-run` (print the plan, exit 0) → interactive confirm on TTY (`Delete 14 snapshots? [y/N]`, default No) → typed resource name for catastrophic scope (`type 'prod-db' to confirm`) → `--force`/`--yes` to skip confirmation for scripts. Non-TTY without `--yes` = refuse with exit 2, never assume yes.
- `--dry-run` must execute the *same* planning code as the real run, stopping just before the side effect. A dry-run built as a separate code path drifts and lies within three releases.
- `--force` means exactly one thing. If it currently means both "skip confirmation" and "overwrite existing" and "ignore lock," split it (`--yes`, `--overwrite`, `--ignore-lock`) before users learn the dangerous conflation.

## Help text craft

- `-h` = concise: usage line, one-sentence description, the flags people actually use, 2–3 real examples with realistic values. `--help` may be longer. Examples are the most-read section — put them near the top, make them copy-pasteable.
- Order flags by frequency of use, not alphabetically; group related flags under headings. An alphabetical dump of 40 flags is write-only documentation.
- Bare `tool` with no args: if the tool needs args, print concise help to *stderr* and exit 2 (it's a usage error). Explicit `tool --help` goes to *stdout* exit 0 — so `tool --help | less` works.
- Ship shell completions (all major frameworks generate bash/zsh/fish); completions are discoverability that help text can't provide.

## Framework landscape (verified July 2026)

| Language | Default pick | State as of 2026 | When to deviate |
|---|---|---|---|
| Rust | **clap 4.x** (4.6.1, Apr 2026) | Derive API (`#[derive(Parser)]`, `#[command(subcommand)]`, `#[command(flatten)]`) is the standard style; builder API remains for dynamic CLIs | Compile time/binary size matters → hand-roll over `lexopt`/`pico-args` |
| Python | **click** (8.4.2, Jun 2026; requires Python ≥3.10) or **typer** (0.26.8, Jun 2026) | typer is actively maintained under the FastAPI org and since 0.26.0 *vendors* click (8.3.1) rather than depending on it — pin accordingly if you also import click directly | Zero-dependency requirement → stdlib `argparse` (but set `allow_abbrev=False`, see pitfalls) |
| Go | **cobra** (v1.10.x, Dec 2025) | The kubectl/gh/docker lineage; noun-verb trees, completions, powerful but heavy | Lighter/declarative taste → `urfave/cli` v3 (stable, v3.6.x); trivial tool → stdlib `flag` |
| Node | **commander** (v15, May 2026 — ESM-only, requires Node ≥22.12) | Dominant for single-package CLIs | Plugin architecture / big product CLI → oclif (@oclif/core v4); zero-dep minimal → citty (0.2.x, UnJS) |

Choose the boring default for your language unless a listed deviation applies; framework choice is rarely where a CLI wins or loses.

## Distribution realities

- **Rust/Go**: the gold standard — one static binary per platform (Go: `CGO_ENABLED=0`; Rust: `x86_64-unknown-linux-musl` for portable Linux). Ship GitHub Releases + a Homebrew formula/tap + an install script; add cargo-dist/GoReleaser to automate. This is a legitimate reason to pick Rust/Go for a CLI even when you'd script the logic faster in Python.
- **Python**: the pain case. `pip install` into a random env breaks other tools' deps; the working answers are isolated-env installers — `uv tool install mytool` or `pipx install mytool` — and your README must say so, or users will pip-install into system Python and file bug reports about your dependencies. Single-file options (PyInstaller/shiv) trade startup time and platform matrices. Budget real time for packaging or don't ship Python CLIs to non-Python users.
- **Node**: `npm install -g` couples your tool to whatever Node the user has; declare `engines`, and remember commander 15's Node ≥22.12 floor is *your* floor too. `npx mytool` is fine for occasional use. Bundle dependencies (esbuild) so a `node_modules` tree isn't your install footprint.
- Whatever the language: `mytool --version` must print an exact, greppable version, and CI should build/install/smoke-test the actual artifact users get, not the dev checkout.

## How an expert thinks through it: designing `dbsnap`

Task: CLI for database snapshots — create, list, restore, delete old ones.

*Four verbs, one noun → subcommands, verb-style: `dbsnap create|list|restore|prune`. Considered `dbsnap snapshot create` (noun-verb, kubectl-style) — rejected: one noun means the noun is noise; noun-verb earns its keep only with ≥2 resource types. If volumes get snapshots later, I'll add `dbsnap volume ...` and keep the bare verbs as aliases for the db noun.*

*`restore` takes what? The snapshot ID — that's the object, so positional: `dbsnap restore snap-01H8...`. Target database? A modifier → `--target`. Considered `dbsnap restore SNAP TARGET`, two positionals — rejected: they're different types and reversible by a tired human; `restore prod-copy snap-01H8` failing at 2 a.m. because the order was flipped is a design bug, not a user error.*

*Connection config: flag `--db-url`? It'd carry a password into `ps` and shell history — rejected as the primary path. Precedence: `--db-url` exists for smoke tests, but docs lead with `DBSNAP_DB_URL` env var and the config file with a `password_file` key. Add the startup warning when the URL-with-password arrives via argv.*

*`list` output: human table on TTY. Scripts will want IDs — add `--json` now, not when asked, because otherwise someone ships `dbsnap list | awk '{print $1}'` to prod this week and my column order is frozen forever (the git-porcelain lesson: git had to bolt `--porcelain` modes on after scripts scraped human output). JSON schema: `{"id":..., "created_at":..., "size_bytes":...}` — sizes as bytes-integer in JSON, pretty "1.2 GiB" only in the human table. NDJSON? Only if listings can be huge; they can't here — single array object.*

*`prune`: destructive. Default behavior — considered "just delete per retention policy," rejected: first run at a new shop deleting 200 snapshots silently is a career event for somebody. Default prints the plan and a `run with --yes to apply` hint — effectively dry-run-by-default for the batch-destructive verb — plus explicit `--dry-run` for scripting the check. Exit codes: 0 = pruned (or plan shown), 1 = error; and give "nothing to prune" exit 0 not 1 — prune is not a search, absence isn't a distinct outcome scripts need. Contrast: a hypothetical `dbsnap verify` WOULD use grep-style 0/1/2.*

*`restore` over an existing database: catastrophic scope → TTY confirm shows snapshot age and target, requires typing the target DB name; `--yes` skips for automation; non-TTY without `--yes` exits 2 with the exact rerun command in the error. Considered a `--force` that both skips confirm and drops the existing DB — rejected, split: `--yes` (skip confirm) vs `--overwrite` (allowed to replace non-empty target).*

*Stopping rule: flags cover create/list/restore/prune with ~4 flags each, help fits one screen per subcommand, `--json` on list only (others print nothing to stdout but IDs). I am NOT adding `--format=yaml`, shell-completion-of-snapshot-IDs, or a TUI — v1 API surface is what I'll maintain forever; everything cut is a v2 option, everything shipped is a contract.*

## Failure modes and pitfalls

- **Progress/log lines on stdout.** `tool export > data.csv` yields a CSV with `Exporting... done` as line 1. Grep your codebase for bare `print(`/`println!(`/`console.log(` and audit each: data or diagnostics? In Node, `console.error` is the stderr twin; in Python pass `file=sys.stderr` (click: `click.echo(..., err=True)`).
- **Exit 0 on failure.** Top-level `except Exception: print(e)` without `sys.exit(1)`, or a Node CLI that logs an error inside a promise chain and lets the process exit naturally (0). Every error path must terminate through one function that maps error → published exit code. Test with `tool bad-input; echo $?` in CI.
- **Prompting when stdin isn't a TTY.** The confirm prompt reads stdin; in `cat ids.txt | tool delete` it consumes the first data line as the answer, or in CI it hangs forever. Gate every prompt on `sys.stdin.isatty()` (and stderr TTY), fail with exit 2 + `--yes` hint otherwise. Click's `click.confirm` raises an `Abort` on non-TTY — but only if you didn't pass `default=`.
- **Prompt text on stdout.** The `[y/N]` line must go to stderr, or it corrupts `tool > out` and is invisible in `tool | less`. Most hand-rolled `input("Continue?")` in Python gets this wrong (`input()`'s prompt goes to stdout).
- **argparse prefix abbreviation.** Python `argparse` accepts `--for` for `--force` by default (`allow_abbrev=True`). Users script the abbreviation; you add `--format` in v1.3; every script passing `--for` now dies ambiguous. Set `allow_abbrev=False` on day one. (click/typer don't abbreviate; clap's equivalent is opt-in.)
- **Optional-value flags eating positionals.** `--color` with optional WHEN: `tool --color file.txt` parses `file.txt` as WHEN. Require `=` for the valued form or use paired booleans. In clap, `num_args(0..=1)` + `require_equals(true)` is the sanctioned pattern; without `require_equals` you've shipped the ambiguity.
- **Changing human output that scripts already scrape, or "fixing" `--json` field names.** Both are breaking changes. Additive-only JSON; if you must break, add `--format=json2` or a top-level `"version"` key from day one. Grep GitHub for your tool's name + `awk`/`jq` before "improving" output.
- **`-v` collision.** `-v` prints version in your tool; users from git/kubectl expect verbose, run `tool -v delete ...` expecting logs, and get version-then-exit — or worse, your parser treats it as verbose and users expecting version get a mutation. Pick `-V` for version, `-v/-vv` for verbosity, and make `--version` always work.
- **Swallowed BrokenPipeError / SIGPIPE unwrap.** `tool list | head` ends with an ugly traceback (Python) or a panic on `.unwrap()` of a failed stdout write (Rust). Python: wrap main in a `BrokenPipeError` handler that closes stderr and exits 0 (per the Python docs' recipe, `devnull` dup onto stdout to avoid the shutdown flush error). Rust: handle write errors on stdout or restore default SIGPIPE behavior explicitly.
- **Buffering surprises.** stdout is line-buffered at a TTY, block-buffered when piped: your "log line before the crash" never appears in the CI capture, and stdout/stderr interleave out of order in logs. Flush after meaningful stdout writes; or in Python honor `PYTHONUNBUFFERED`; don't debug phantom "lost output" for a day (it's this).
- **Color decision half-implemented.** Supporting `NO_COLOR` but not checking isatty (colored garbage in pipes), or checking isatty but not `NO_COLOR`/`TERM=dumb`, or testing `NO_COLOR=1` specifically (the spec is *any non-empty value*; `NO_COLOR=0` also disables — presence, not truthiness, verified against the spec as of 2026). Use the switchboard function below; also note `FORCE_COLOR=0` is treated as "off" by much of the Node ecosystem even though force-color.org says presence forces — treat `0` as off for both FORCE_ and CLICOLOR_FORCE to match user expectation.
- **Config without override symmetry.** A boolean that's `true` in the config file with only a `--flag` to turn it *on* — no `--no-flag` — means file users can never disable per-invocation. Every config-file key needs a flag that can set it to *any* value, including back to default.
- **Dry-run as a separate code path.** `if dry_run: print(plan)` where `plan` is computed by different code than the executor uses. The fix is structural: one function computes the action list; dry-run prints it, real run executes it. If your dry-run has ever said "would delete 3" and the run deleted 4, this is why.
- **Version-check/telemetry phone-home on every invocation.** Adds 300ms latency, breaks air-gapped users, and violates trust when undisclosed. If you must: async check, cached daily, stderr notice only on TTY, honor an opt-out env var, and document it.
- **Exceeding one byte of exit code.** `sys.exit(len(errors))` with 300 errors → exit 44. `process.exit(-1)` → 255. Clamp and map.
- **Windows as an afterthought.** ANSI codes need enabling on older Windows consoles (Windows Terminal is fine; libraries handle it — raw escape writers don't), path handling breaks on `\`, and `~` doesn't expand. If you claim Windows support, CI on Windows; if you don't, say so in the README rather than half-working.

## Worked micro-examples

**1. The color switchboard (Python, stdlib only) — encode the verified precedence once:**

```python
import os, sys

def use_color(stream=sys.stdout) -> bool:
    env = os.environ
    if env.get("NO_COLOR"):                      # any non-empty value: off (highest priority)
        return False
    if env.get("CLICOLOR_FORCE", "0") not in ("", "0"):
        return True                              # force on, even piped
    if env.get("FORCE_COLOR") not in (None, "", "0"):
        return True
    if env.get("CLICOLOR") == "0" or env.get("TERM") == "dumb":
        return False
    return stream.isatty()

COLOR_OUT = use_color(sys.stdout)   # decide per stream, once, at startup
COLOR_ERR = use_color(sys.stderr)
```

**2. Subcommand skeleton with correct stream/exit discipline (typer 0.26.x, as of 2026):**

```python
import sys, typer
from typing import Annotated

app = typer.Typer(no_args_is_help=True)

@app.command()
def prune(
    keep: Annotated[int, typer.Option(help="Snapshots to retain")] = 10,
    yes: Annotated[bool, typer.Option("--yes", help="Skip confirmation")] = False,
):
    plan = compute_prune_plan(keep)              # SAME code path as execution
    if not plan:
        typer.echo("Nothing to prune.", err=True)   # diagnostics -> stderr
        raise typer.Exit(0)                          # absence is success here
    for snap in plan:
        typer.echo(snap.id)                          # data -> stdout
    if not yes:
        if not sys.stdin.isatty():
            typer.echo("error: refusing to prune without --yes when not interactive", err=True)
            raise typer.Exit(2)                      # usage error
        typer.confirm(f"Delete {len(plan)} snapshots?", abort=True)  # prompt -> stderr, Abort -> exit 1
    execute(plan)
```

**3. Exit-code contract as shipped in help text:**

```
EXIT STATUS
  0  success (including "nothing to do")
  1  operation failed
  2  usage error (bad flags or arguments)
  4  snapshot not found
  5  target database not empty (rerun with --overwrite)
```

Scripts can now write `if dbsnap restore "$id" --target x --yes; then` and branch on `4` vs `5`. That block is a compatibility promise — additions fine, renumbering never.

## Verification / self-check

Before presenting a CLI design or implementation, run this gauntlet:

1. **Pipe test:** `tool cmd > out && wc -l out` — is `out` pure data? `tool cmd | head -1` — no traceback/panic? `tool cmd 2>/dev/null` — did progress and prompts disappear but data survive?
2. **Exit test:** `tool cmd; echo $?` for the success, failure, usage-error, and "found nothing" cases — four distinct, documented, deliberate values (even if "found nothing" deliberately equals 0).
3. **CI test:** run with stdin from `/dev/null` and no TTY — every path either completes or fails fast with an instructive message; nothing hangs, nothing colors, nothing animates.
4. **Env test:** `NO_COLOR=1`, `CLICOLOR_FORCE=1 tool | cat`, `TERM=dumb` behave per the precedence above.
5. **Convention test:** `-h`, `--help`, `--version` all work; `--help` on every subcommand; help shows examples; flags follow the near-universal letters; nothing requires a secret in argv.
6. **Contract test:** could you rename any flag, change any default, or reorder any output column tomorrow? Everything you couldn't is API — is all of it listed somewhere as a promise?
7. **Stopping rule:** stop adding surface when every subcommand's `-h` fits one screen, every flag exists because a real workflow demanded it (not symmetry or "might be nice"), and the `--json` schema covers what scripts need. A CLI is done when there's nothing left to *remove* that a named user story would miss — extra flags are permanent maintenance, not generosity.
