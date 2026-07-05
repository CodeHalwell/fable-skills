---
name: cli-tool-design
description: Load when designing or reviewing a command-line tool — argument/subcommand structure, flags vs positionals, exit codes, stdout/stderr separation, --json and color/TTY output handling, config precedence, error message wording, destructive-action safety, help text, choosing a CLI framework (clap, click/typer, cobra, commander/oclif/citty), or picking a distribution strategy per language.
---

# CLI Tool Design

Compact sheet. Standard expertise — stdout=data/stderr=everything-else, grep's 0/1/2 model, shell-reserved codes and mod-256 wrapping, prompts to stderr gated on TTY, `allow_abbrev=False`, optional-value flag ambiguity + `require_equals`, SIGPIPE handling in Python/Rust, dry-run-shares-the-real-code-path, config precedence, secrets-never-in-argv, `-v`=verbose/`-V`=version, bare-tool→stderr/exit-2 vs `--help`→stdout/0, OSC 8 as TTY-gated progressive enhancement — is assumed known and appears only as anchors. Expanded sections carry the spec subtleties and 2026 version facts baseline models get wrong.

## Anchors (apply without re-derivation)

- Two audiences per output decision (human at TTY, program in pipeline); `isatty()` per stream is the switch; auto-switch decoration only, never data content — format changes go through an explicit `--json`/`--format`.
- Exit contract: 0 success (including "nothing to do" unless the tool is a search/check — decide grep-style 0/1/2 before v1); 2 usage; 3–63 yours, documented; stay out of 126/127/128+n; map internal errors explicitly (exit(-1)→255, exit(256)→0 = silent false success).
- Argument shaping: one verb → no subcommands; might-grow-verbs → subcommand from day one (retrofitting breaks every invocation); the object of the verb is positional (max two, same type), everything else flags; no optional middle positionals; `--` before child-command args.
- Prompts: stdin AND stderr TTYs, else exit 2 with "use --yes"; prompt text to stderr (Python `input()` prompts to stdout — wrong); piped stdin is data, not a keyboard.
- Destructive ladder: dry-run → TTY confirm default-No → typed resource name → `--yes` for scripts; split `--force` meanings (`--yes`/`--overwrite`/`--ignore-lock`); batch-destructive verbs are dry-run-by-default.
- Config: flags > env > project > user > system > defaults; every key gets `MYTOOL_UPPER_SNAKE` env + a flag that can set any value incl. back to default (`--no-X` inverses); XDG paths incl. on macOS for dev tools; print effective config with provenance.
- Errors: what failed (quote the offending value) / why / what to do next; rewrite dependency errors at your boundary; usage errors get one usage line + `Try --help`, not the 80-line dump.
- `--json` schema, exit codes, flag names, and human output that scripts scrape are all API: additive-only, version or never rename. Ship `--version` greppable, shell completions, examples near the top of help.
- Buffering: stdout block-buffers when piped — flush after meaningful writes or "the log line before the crash" vanishes from CI capture.

## Spec subtleties baseline models get wrong

- **`NO_COLOR` disables only when set to a *non-empty* value** (the spec was amended: presence-and-non-empty, so `NO_COLOR=` empty does NOT disable). The common wrong beliefs are truthiness (`NO_COLOR=0` still disables — correct behavior, value is non-empty) and bare-presence-including-empty (wrong). Full precedence, implemented once as a per-stream function: non-empty `NO_COLOR` → off; else `CLICOLOR_FORCE`/`FORCE_COLOR` non-empty-and-not-`0` → on even piped (treat `0` as off for both — the Node ecosystem does, whatever force-color.org implies); else `CLICOLOR=0` or `TERM=dumb` → off; else isatty per stream.
- Progress/animation keys off *stderr* TTY specifically (a spinner in CI logs is 10k lines of `\r` garbage); color decisions are per-stream, decided once at startup.

## Framework/distribution state (verified July 2026 — recheck before citing)

| Language | Default | 2026 state baseline models miss |
|---|---|---|
| Rust | clap 4.x (4.6.1) | derive API is the standard style; `lexopt`/`pico-args` only for compile-time/size constraints |
| Python | click 8.4.x or typer 0.26.x | typer (still 0.x, FastAPI org) **vendors click since 0.26** — pin accordingly if you also import click directly; click requires ≥3.10 |
| Go | cobra v1.10.x | urfave/cli v3 stable as the lighter alternative |
| Node | **commander v15 (May 2026): ESM-only, Node ≥22.12 floor** — that floor becomes your users' floor | oclif for plugin CLIs; citty for zero-dep |

Distribution: Rust/Go static binaries + Homebrew + cargo-dist/GoReleaser is the gold standard and a legitimate language-choice reason. Python CLIs: lead the README with `uv tool install`/`pipx` or absorb dependency-conflict bug reports; PyInstaller/shiv trade startup and platform matrices. Node: declare `engines`, bundle deps. CI must smoke-test the shipped artifact, not the dev checkout.

## Design walkthrough discipline (the part that isn't in any spec)

For each command, write down: which parameters are the verb's object (positional) vs modifiers (flags); whether two positionals could be flipped by a tired human at 2 a.m. (different types → flags); where the secret would leak (argv/ps/history → env/file/stdin instead); what a script will scrape within a week (add `--json` *now*, sizes as integer bytes in JSON, pretty units only in the human table — the git-porcelain lesson); and the stopping rule — v1 surface is a permanent contract, so everything cut is a v2 option and everything shipped is maintenance forever. "Nothing to prune" exits 0; a `verify` command uses 0/1/2. Decide per command, out loud, in the design doc.

## Verification gauntlet

1. Pipe: `tool cmd > out` pure data; `| head -1` no traceback/panic; `2>/dev/null` kills progress/prompts, keeps data.
2. Exit: `echo $?` for success / failure / usage / found-nothing — four deliberate documented values.
3. CI: stdin from `/dev/null`, no TTY — nothing hangs, colors, or animates; refusals are instructive.
4. Env: `NO_COLOR=1`, `NO_COLOR=` (empty — must NOT disable), `CLICOLOR_FORCE=1 | cat`, `TERM=dumb`.
5. Convention: `-h`/`--help`/`--version` everywhere; near-universal short letters; no secret in argv.
6. Contract: list everything you couldn't rename tomorrow — is all of it documented as a promise?

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)
- Probed 14 claims: 11 baseline (compressed to anchors), 2 partial (NO_COLOR empty-string spec detail — baseline asserted presence-including-empty disables, which is wrong under the amended spec; non-TTY-refusal exit code), 1 delta (2026 framework state: commander v15 ESM-only/Node≥22.12, typer vendoring click since 0.26).
- Opus cold reproduced: exit-code contract incl. mod-256, stream discipline, prompt matrix, dry-run structure, allow_abbrev, SIGPIPE in both Python and Rust, clap `require_equals`/`default_missing_value`, config precedence, OSC 8 gating, an exhaustive review checklist.
- Biggest gaps: the NO_COLOR amendment and current package-version floors — recheck both before citing.
