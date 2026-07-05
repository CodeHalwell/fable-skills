---
name: python-idioms
description: Loads expert Python judgment for typing, asyncio, data-class selection, packaging with uv, performance triage, and stdlib footguns. Use when writing or reviewing non-trivial Python, debugging async code, choosing dataclasses vs pydantic vs attrs, setting up pyproject/lock files, or deciding when to reach for numpy/polars/Rust.
---

# Python Idioms

## Core mental model (anchors — details are recoverable cold)

1. Type annotations are advisory; validate at boundaries (pydantic/msgspec), type-check the interior. `TypedDict`/`cast` enforce nothing at runtime; mutable containers are invariant.
2. asyncio is cooperative on one thread; one blocking call freezes every task. Import time is execution time — top level defines, functions do.
3. Performance triage: algorithm → batching → vectorization → native code. Ask "why is there a per-item Python loop" before "how do I speed up this loop."

## Current state (verified July 2026)

- Python 3.14 current stable; free-threading officially supported (PEP 779), separate `t` build, ~5–10% single-threaded overhead, C extensions must ship `cp314t` wheels or the GIL silently re-enables. 3.14 also: deferred annotations by default (PEP 649/749 — `from __future__ import annotations` is legacy in 3.14-only code), t-strings (PEP 750), experimental JIT.
- Packaging: **uv won** — `uv init/add/sync/run`, `pyproject.toml` + `uv.lock`; PEP 751 `pylock.toml` for interop (`uv export -o pylock.toml`). PEP 735 `[dependency-groups]` for dev deps. ruff for lint+format; pyright (or mypy) for types. PEP 723 inline metadata + `uv run script.py` for single-file scripts.
- Typing floor: PEP 695 generics (3.12+), `X | None`, `Self`. `typing.List`/`Optional`/manual `TypeVar` are legacy style.

Note: Opus-class baselines already know most of the above — the load-bearing correction is only the *free-threading fine print*: don't promise thread parallelism unless the user deliberately runs the `t` build, and warn that GIL-era "accidentally correct" unsynchronized code becomes actually racy there (free-threading exposes latent bugs, it doesn't create them).

## Judgment calls where the reflex answer differs

- **"Should this be async?" — the common threshold is too high.** The reflex says "async above ~1000 concurrent connections." Calibrate lower and structural: async pays off from ~50 simultaneous *waits* (gateways, scrapers, websocket fan-out) — but only if the libraries are actually async; one sync-only dependency in the hot path means `to_thread` wrappers everywhere, and the honest answer is often "stay sync." Under ~10 concurrent calls, threads + `concurrent.futures` win on stack traces alone. Prior: most Python services are fine sync with a thread pool; async is a fan-out tool, not a default.
- **dataclass vs pydantic vs attrs vs msgspec** — one question decides: does untrusted data cross this constructor? Yes → pydantic v2 (msgspec when serialization throughput is the *measured* bottleneck). No → `@dataclass(slots=True, frozen=True)`; `kw_only=True` beyond ~3 fields. attrs only for per-field validators without buying pydantic. The architecture rule that survives review: **pydantic at the edge, dataclass inside, one explicit converter between** — do not migrate domain objects to pydantic when they later get serialized; write the one mapping function. One parse per datum per entry; re-validation inside the boundary is a signal you don't trust your own types.
- **`asyncio.TaskGroup` is the default; a bare `create_task` needs a written reason** (loop holds only a weak ref; owner-less tasks vanish and swallow exceptions). `gather` only for best-effort fan-out where you inspect every result.
- **Rust via PyO3 + maturin is the 2026 default over Cython** for new native code; polars over pandas for new pipelines. Stopping rule: stop optimizing when the hot function is <20% of wall time or the profile shows I/O.

## Diagnosis priors (compressed — the tools, not the walkthroughs)

- Async stalls with low CPU → blocked loop. `PYTHONASYNCIODEBUG=1` logs callbacks >100ms; tighten with `loop.slow_callback_duration = 0.05`. Remember the non-obvious culprit: an await-less `while True` never yields — `await asyncio.sleep(0)` is the explicit yield. Fix with `asyncio.to_thread` + a `Semaphore` bound so a burst doesn't spawn 500 threads.
- Slow CLI startup → `python -X importtime -c "import mycli"` (visualize with tuna); function-local imports for heavy optional deps are perfectly idiomatic. Target <~200ms for a CLI. Also grep top levels for `os.environ[...]`/file reads — import-order correctness bugs, not just latency.
- Profile with `py-spy` for live processes (zero code changes), `cProfile`/snakeviz for structure.

## Pitfalls checklist (one-liners; expanded only where cold answers miss)

Baseline (kept for completeness): mutable default args; late-binding closures (`lambda i=i:`); `except BaseException`/bare `except` swallowing `CancelledError` (re-raise if intercepted); TypedDict from `json.loads` KeyErrors (validate with `pydantic.TypeAdapter`); `lru_cache` on methods immortalizes `self` (use `cached_property`); generator `finally` runs at GC time — resource `with` lives *inside* the generator or wrap consumption in `closing()`; `json.loads(s, parse_float=Decimal)` at financial boundaries; `datetime.now(timezone.utc)` never `utcnow()`; `removesuffix` not `strip("suffix")`; `os.path.join("/a", "/b") == "/b"`; argv-list subprocess with `check=True`; `is` only for `None`/sentinels.

Expanded — commonly missed even by strong baselines:
- **`json.dumps` emits `NaN`/`Infinity` by default — invalid JSON that a downstream parser rejects.** Set `allow_nan=False` at boundaries.
- **Shadowing stdlib module names** (`types.py`, `email.py`, `queue.py`, `test.py` in package root) → baffling `partially initialized module` errors. Check filenames first when imports misbehave.
- **`@overload` implementations are only loosely checked** against the declared overloads — a wrong body passes the type checker; test each overload path at runtime.
- **`@runtime_checkable` Protocol `isinstance` checks names only, not signatures** — duck-typing hint, not a contract; don't build dispatch on it.
- **`yield` inside `with` in an async generator** + early consumer exit can run cleanup in a *different task* at shutdown ("Future attached to a different loop"). Prefer `contextlib.aclosing` or `async with` in the consumer.
- **Exception-swallowing `__exit__`**: returning truthy (or `@contextmanager` catching around `yield` without re-raising) silently suppresses exceptions — suppress only named types.
- **pytz in new code** — `pytz.localize` misuse is the classic LMT-offset bug; `zoneinfo` always.
- Stdlib gems that prevent reinvention: `itertools.pairwise`/`batched` (3.12+), `contextlib.ExitStack` (dynamic number of CMs), `tomllib`, `shutil.which`, `bisect`/`heapq`, `dataclasses.replace`.

## Micro-example: the boundary pattern (the shape reviewers should enforce)

```python
class SignupRequest(BaseModel):          # EDGE: untrusted JSON crosses here
    email: EmailStr
    referrer: str | None = None

@dataclass(slots=True, frozen=True)      # INTERIOR: constructed from validated parts
class Account:
    email: str
    tier: str

def create_account(req: SignupRequest) -> Account:   # the one conversion point
    return Account(email=req.email, tier="free" if req.referrer is None else "trial")
```

## Verification and stopping rule

1. Run the code mentally at import time — anything executing that shouldn't?
2. Async: every task has an owner; every `except` audited for swallowed cancellation; name the blocking-call audit you did.
3. Performance claims demand a profile; version-gate advice (PEP 695 → 3.12+, TaskGroup/`except*` → 3.11+, t-strings/deferred annotations → 3.14, free-threading → the `t` build specifically).

Done when boundaries validate, tasks are owned, blocking calls are off the loop. Adding pydantic to internal call chains "for safety" is waste.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)

- Probed 14 claims: 12 baseline (cut/compressed), 2 partial (sharpened), 0 hard delta.
- Opus cold already knows: 3.14 free-threading status + 5–10% overhead, uv/PEP 751, TaskGroup/gather semantics, importtime/asyncio-debug workflow, parse_float=Decimal, lru_cache-on-method. This skill was ≥80% baseline and was restructured to a correction sheet.
- Remaining value: async-threshold calibration (~50 waits, not ~1000; "most services fine sync"), pydantic-edge/dataclass-interior architecture rule, and the rarer pitfalls (json NaN, stdlib-name shadowing, overload/protocol loose checks, async-generator cleanup task).
