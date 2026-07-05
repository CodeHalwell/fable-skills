---
name: python-idioms
description: Loads expert Python judgment for typing, asyncio, data-class selection, packaging with uv, performance triage, and stdlib footguns. Use when writing or reviewing non-trivial Python, debugging async code, choosing dataclasses vs pydantic vs attrs, setting up pyproject/lock files, or deciding when to reach for numpy/polars/Rust.
---

# Python Idioms

## Core mental model

1. **The type system is advisory, and it lies in known places.** Annotations are checked by external tools (pyright/mypy); the runtime enforces nothing. Known lies: `list[int]` is unchecked at runtime; a `TypedDict` value is a plain dict that may have extra or missing keys from any untyped producer; `cast()` is a no-op; `list[Dog]` is not `list[Animal]` (mutable containers are invariant) even though duck typing would accept it. Discipline: type-check the interior, *validate* the boundary — same shape as TypeScript's rule.
2. **asyncio is cooperative scheduling on one thread.** The loop cannot preempt; one blocking call anywhere freezes every task. Your protections are discipline (await only non-blocking things) and escape hatches (`asyncio.to_thread`, executors) — nothing else.
3. **Import time is execution time.** Module top level runs code, once, cached in `sys.modules`. Side effects there (opening connections, reading env, registering singletons) create import-order bugs, circular-import crashes, untestable modules, and slow CLIs. Top level defines; functions do.
4. **Context managers and generators are design tools, not sugar.** `with` is the idiom for any guaranteed pairing (acquire/release, patch/restore, begin/commit-or-rollback); `@contextmanager` turns a 10-line try/finally class into 4 lines. Generators are lazy pipelines — reach for them when data exceeds memory or when you want composable stages without materializing intermediates.
5. **Performance triage order: algorithm → batching → vectorization → native code.** A pure-Python loop over millions of items is ~100x slower than the numpy/polars equivalent. The expert's first question is never "how do I speed up this loop" but "why is there a per-item Python loop at all."
6. **Prefer the boring idiom.** Python rewards the obvious: comprehensions over map/filter chains, `pathlib` over `os.path`, early returns over nesting, exceptions over error codes (EAFP), plain functions over classes with one method.

## Current state (verified July 2026)

- **Python 3.14 is current stable**; 3.13 and 3.12 remain widely deployed — check the project's floor before using new syntax.
- **Free-threaded Python is officially supported as of 3.14 (PEP 779)** — no longer experimental. Single-threaded overhead dropped to roughly 5–10% (from ~40% in 3.13). It is still a *separate build* (`python3.14t`); the default build keeps the GIL. Consequences:
  - Do not claim a user's threads run in parallel unless they deliberately run the `t` build.
  - C extensions must declare free-threading support; loading one that doesn't re-enables the GIL. Check the ecosystem status of your specific wheels before promising speedups.
  - `threading` code that was "accidentally correct under the GIL" (unsynchronized counters, dict races) becomes actually racy — free-threading exposes latent bugs, it doesn't create them.
- 3.14 also ships: an experimental JIT, template strings (PEP 750 t-strings), and **deferred annotation evaluation by default** (PEP 649/749) — `from __future__ import annotations` is now legacy in 3.14-only code.
- **Packaging in 2026: uv won.** `uv init` / `uv add` / `uv sync` / `uv run` with `pyproject.toml` + `uv.lock` is the default recommendation. PEP 751 standardized `pylock.toml` as the interoperable lock format; uv keeps `uv.lock` for projects (richer) and interoperates via `uv export -o pylock.toml` and `uv pip` support. Poetry and pip-tools are legacy-team choices, not new-project choices. Lint/format: ruff replaced black+isort+flake8 in most stacks; pyright and mypy split the type-checking market (pyright is the common default with uv-era tooling).
- Typing floor for new code: `list[int]`, `X | None`, `Self`, PEP 695 generics (`class Stack[T]: ...`, `def first[T](xs: list[T]) -> T:`, 3.12+). `typing.List`, `Optional[X]`, and manual `TypeVar` declarations are legacy style.

## Decision frameworks with reasoning chains

**dataclass vs pydantic vs attrs.** The first question decides most cases: *does untrusted data cross this constructor?*
1. Yes (API bodies, config files, queue messages, LLM output) → **pydantic v2**. Or **msgspec** when serialization throughput is the measured bottleneck — it is several times faster, with a smaller feature set.
2. No — your own code constructs it from already-validated parts → **`@dataclass(slots=True, frozen=True)`** for value objects; add `kw_only=True` beyond ~3 fields. Pydantic here is pure overhead: validation cost on every construction plus metaclass complexity in tracebacks.
3. **attrs** earns its slot only for gaps in dataclasses: per-field validators without buying all of pydantic, or fine control (`__init_subclass__`-free machinery, field transformers).
4. What changes the answer: if the "internal" type later gets serialized to clients, don't migrate the domain object to pydantic — write one explicit converter at the boundary. Domain objects staying framework-free is worth one mapping function.

**"Should this be async?"** Ask in order:
1. Is the workload I/O-bound with *many* concurrent waits (>~50 simultaneous connections: gateways, scrapers, websocket fan-out)? Under ~10 concurrent calls, threads + `concurrent.futures` are simpler and give ordinary stack traces.
2. Are the libraries actually async (asyncpg, httpx, aioboto3)? One sync-only dependency in the hot path means `to_thread` wrappers everywhere — sometimes the honest answer is "stay sync."
3. Will the team maintain two colors of function? Async doubles the API surface.
Prior: **most Python services are fine sync with a thread pool**; async pays off at high fan-out, not as a default.

**"Reach for numpy/polars/Rust?"** Profile first — `cProfile`/`snakeviz` for structure, `py-spy` for sampling a live process with zero code changes. Then:
- Hot loop over homogeneous numerics → numpy.
- Tabular transforms/joins/groupbys → **polars** for new pipelines (multi-threaded, lazy engine); pandas where the ecosystem (plotting, legacy) demands it.
- Branchy per-item logic → first try restructuring into vectorizable steps (masks, `np.where`, groupby); genuinely irreducible → a **Rust extension via PyO3 + maturin** (the 2026 default over Cython for new native code: better tooling, no C-level UB).
- Stopping rule: stop when the hot function is <20% of wall time or the profile shows I/O — measured, not intuited.

## How an expert thinks through it: "my async server stalls under load"

Symptoms: p99 spikes, health checks time out, CPU is *low*.

Internal monologue: *Low CPU + stalls in async = the event loop is blocked, not overloaded. Prior: ~80% of these are a hidden synchronous call — `requests` instead of `httpx`, a sync DB driver, `time.sleep`, file I/O, or CPU-heavy JSON/crypto on the loop. Cheapest evidence first: enable the built-in detector — `PYTHONASYNCIODEBUG=1` or `loop.set_debug(True)` logs every callback exceeding 100ms with its location; also `loop.slow_callback_duration = 0.05` to tighten it. Competing hypotheses to keep alive: a tight `while True` without an await (a coroutine yields ONLY at await points — an await-less loop never yields, and `await asyncio.sleep(0)` is the explicit yield); GC pauses from huge object graphs (rarer; check last). Debug log fingers `stripe.Charge.create` — sync SDK. Options: (a) wrap in `asyncio.to_thread` — correct, cheap; must bound concurrency with a semaphore so a burst doesn't spawn 500 threads, and confirm the SDK is thread-safe; (b) the SDK's async client if one exists — better long-term; (c) offload payments to a worker queue — right only if calls are slow AND retryable; rejected for now, no evidence the latency budget needs that machinery. Ship (a) with `Semaphore(20)`; ticket for (b).*

Verification: replay load; the 100ms-callback warnings must disappear and p99 must follow.

## Failure modes and pitfalls

- **Fire-and-forget task garbage collection.** `asyncio.create_task(coro())` without keeping a reference: the loop holds only a *weak* reference, so the task can be GC'd mid-flight and silently vanish. Correct: `t = create_task(...); tasks.add(t); t.add_done_callback(tasks.discard)` — or better, **`asyncio.TaskGroup`** (3.11+), which owns its children, cancels siblings on failure, and raises `ExceptionGroup` (handle with `except*`). TaskGroup is the default; a bare `create_task` needs a written reason.
- **Swallowed cancellation.** `CancelledError` subclasses `BaseException`, so `except Exception` correctly lets it fly. But bare `except:` or `except BaseException:` around an `await` eats cancellation — timeouts then hang forever and shutdown never completes. If you intercept it for cleanup, re-raise.
- **`asyncio.gather` semantics.** Default `gather` raises on the first failure while the *other tasks keep running unsupervised*; `return_exceptions=True` converts failures into return values that are easy to never check. Use TaskGroup for "all or nothing"; reserve `gather(..., return_exceptions=True)` for "collect every outcome and inspect each."
- **Blocking calls camouflaged as innocent:** `time.sleep`, `requests.*`, `open().read()` on network filesystems, `subprocess.run`, `hashlib` over large buffers, `json.dumps` of huge objects, ORM lazy-loads. In async code each needs `asyncio.to_thread` / `run_in_executor`, or an async-native replacement.
- **Mutable default arguments.** `def f(x, acc=[])` shares one list across all calls. Use a `None` sentinel; in dataclasses, `field(default_factory=list)` (dataclasses at least raise on bare mutable defaults).
- **Late-binding closures in loops.** `[lambda: i for i in range(3)]` — every lambda returns 2. Fix: `lambda i=i: i` or `functools.partial`.
- **TypedDict trust misplacement.** `def f(d: MovieDict)` then `d["year"]` KeyErrors in production because the dict came from `json.loads`. TypedDict is a static claim, not a validator. At boundaries: `pydantic.TypeAdapter(MovieDict).validate_python(raw)`.
- **`@overload` bodies aren't verified against the overloads** — checkers validate call sites against the signatures but only loosely check the implementation; a wrong body passes. Test each overload path at runtime.
- **`@runtime_checkable` Protocol checks names, not signatures.** `isinstance(x, SupportsClose)` passes for any object with a `close` attribute regardless of its signature. Runtime protocols are duck-typing hints, not contracts; don't build dispatch on them.
- **Import-time side effects and circulars.** `from app.db import engine` where `db.py` builds the engine at import: now test collection needs a database. Fix: lazy init (`@functools.cache`-wrapped `get_engine()`) or app-factory pattern. Circular imports needed only for annotations: `if TYPE_CHECKING:` import — clean now that annotations are deferred by default on 3.14.
- **stdlib gems experts actually use:** `itertools.pairwise` / `batched` (3.12+), `functools.cache` / `cached_property`, `collections.Counter` / `defaultdict` / `deque`, `pathlib` everywhere, `shutil.which`, `textwrap.dedent`, `tomllib` (3.11+, read-only TOML), `zoneinfo` (never pytz in new code — `pytz.localize` misuse is the classic LMT-offset bug), `dataclasses.replace`, `contextlib.ExitStack` for a dynamic number of context managers, `contextlib.suppress` for intentional ignoring, `bisect`/`heapq` before writing a search or priority queue.
- **stdlib footguns:** `datetime.utcnow()` — deprecated and returns a *naive* datetime; use `datetime.now(timezone.utc)`. `json.dumps` happily emits `NaN` (invalid JSON — set `allow_nan=False` at boundaries). `str.strip("suffix")` strips a character *set*; use `removesuffix`/`removeprefix`. `os.path.join("/a", "/b")` → `/b`. `copy.copy` shares nested interiors. `subprocess.run` — always list argv, `check=True`, never `shell=True` with interpolated strings. `re` module: catastrophic backtracking on adversarial input — prefer `re2`-style patterns or bound input length.
- **`is` vs `==` on small ints/strings** — works in tests by interning accident, fails in production. `is` is for `None`, `True`, `False`, and sentinels only.

## Worked micro-examples

**Structured concurrency with bounded fan-out (the 2026-default shape):**
```python
import asyncio, httpx

async def fetch_all(urls: list[str]) -> dict[str, str]:
    results: dict[str, str] = {}
    sem = asyncio.Semaphore(20)                     # explicit concurrency bound
    async with httpx.AsyncClient(timeout=10.0) as client:
        async with asyncio.TaskGroup() as tg:       # owns children: no GC'd tasks, no orphans
            async def one(u: str) -> None:
                async with sem:
                    r = await client.get(u)
                    r.raise_for_status()
                    results[u] = r.text
            for u in urls:
                tg.create_task(one(u))
    return results   # reaching this line proves every task completed or the group raised
```

**A context manager as a design tool (guaranteed pairing, 4 lines):**
```python
from contextlib import contextmanager
import time, logging

@contextmanager
def timed(label: str):
    t0 = time.perf_counter()
    try:
        yield
    finally:
        logging.info("%s took %.1fms", label, (time.perf_counter() - t0) * 1e3)

with timed("reindex"):
    reindex_all()
```

**uv project skeleton (pyproject.toml essentials):**
```toml
[project]
name = "svc"
requires-python = ">=3.12"
dependencies = ["httpx>=0.28", "pydantic>=2.7"]

[dependency-groups]            # PEP 735 dev-dependency groups, uv-native
dev = ["pytest>=8", "ruff>=0.5", "pyright>=1.1"]
```
Workflow: `uv add httpx` (updates pyproject + `uv.lock`), `uv sync` (reproduce the env), `uv run pytest` (never hand-activate venvs in scripts/CI), `uv export -o pylock.toml` when another tool needs the PEP 751 standard format. Single-file scripts: PEP 723 inline metadata + `uv run script.py`.

## Verification and stopping rule

Before presenting Python advice or code:
1. Run it mentally at *import time* — does anything execute that shouldn't?
2. For async code: trace every task to an owner (TaskGroup or stored reference + done-callback), audit every `except` for swallowed `CancelledError`, and name the blocking-call audit you did.
3. For typing claims: remember which guarantees are static-only — would this code survive an untyped caller handing it garbage? If the data is external, where exactly is the validation line?
4. For performance claims: demand a profile. Never assert a bottleneck that wasn't measured; never recommend numpy/Rust for code that runs once a day in 3 seconds.
5. Version-gate the advice: PEP 695 syntax needs 3.12+; TaskGroup/`except*` need 3.11+; t-strings and default-deferred annotations need 3.14; free-threading claims need the `t` build specifically.

Stopping rule: done when boundaries validate, tasks are owned, blocking calls are off the loop, and the profiler attributes remaining time to I/O or genuine compute. Micro-optimizing Python bytecode beyond that point, or adding pydantic to internal call chains "for safety," is waste.
