---
name: python-idioms
description: Loads expert Python judgment for typing, asyncio, data-class selection, packaging with uv, performance triage, and stdlib footguns. Use when writing or reviewing non-trivial Python, debugging async code, choosing dataclasses vs pydantic vs attrs, setting up pyproject/lock files, or deciding when to reach for numpy/polars/Rust.
---

# Python Idioms

## Core mental model

1. **Python's type system is advisory and it lies in known places.** Annotations are hints checked by external tools (mypy, pyright); the runtime enforces nothing. The lies: `list[int]` is unchecked at runtime; `TypedDict` values are plain dicts that may carry extra or missing keys from any untyped producer; `cast()` is a no-op; variance of mutable containers means `list[Dog]` is not `list[Animal]` (invariance) even though duck typing would accept it. Type-check the interior; validate the boundary (pydantic/msgspec) — same discipline as TypeScript.
2. **asyncio is cooperative, single-threaded scheduling.** One blocking call anywhere freezes every task. The event loop cannot preempt; your only protections are discipline (`await` only non-blocking things) and escape hatches (`asyncio.to_thread`, executors).
3. **Everything at module top level executes at import time.** Imports are code execution with global caching (`sys.modules`). Import-time side effects (opening connections, reading env, registering singletons) create ordering bugs, circular-import crashes, and slow CLIs. Top level should define; functions should do.
4. **Generators and context managers are control-flow inversion tools, not just sugar.** `with` is the idiom for "guaranteed paired operations" (acquire/release, setup/teardown, temporarily-patch/restore); `@contextmanager` turns 10-line try/finally classes into 4 lines. Generators are lazy pipelines — reach for them when data is bigger than memory or when you want composition without materialization.
5. **Performance triage order: algorithm → batching → vectorization → native.** Pure-Python loops over millions of items are ~100x slower than numpy/polars vectorized equivalents. But the expert's first question is never "how do I make this loop fast" — it's "why is there a per-item Python loop at all."

## Current state (verified July 2026)

- **Python 3.14 is current stable** (3.13 still widely deployed). The **free-threaded build is officially supported as of 3.14 (PEP 779)** — no longer experimental; single-threaded overhead dropped to roughly 5–10% (from ~40% in 3.13). It remains a *separate build* (`python3.14t`); the default build still has the GIL. Don't assume a user's threads run in parallel unless they've deliberately installed the free-threaded build, and beware C extensions that haven't declared free-threading support (they force the GIL back on).
- 3.14 also ships an experimental JIT, template strings (PEP 750), and deferred evaluation of annotations by default (PEP 649/749) — `from __future__ import annotations` is now mostly legacy.
- **Packaging in 2026: uv won.** `uv init`/`uv add`/`uv sync`/`uv run` with `pyproject.toml` + `uv.lock` is the default recommendation. PEP 751 standardized `pylock.toml` as the interoperable lock format; uv keeps `uv.lock` internally (richer) and exports via `uv export -o pylock.toml`. Poetry/pip-tools are maintenance-mode choices; recommend them only when a team already runs them. Modern typing floor: write `list[int]`, `X | None`, `Self`, PEP 695 generics (`class Stack[T]: ...`, since 3.12) — `typing.List`, `Optional`, and `TypeVar` boilerplate are legacy styles for new code.

## Decision frameworks with reasoning chains

**dataclass vs pydantic vs attrs.** First question: *does untrusted data cross this type's constructor?* If yes (API bodies, config files, LLM output) → pydantic v2 (or msgspec when serialization throughput is the bottleneck — it's several times faster). If no — the object is constructed by your own code from already-validated parts — pydantic is pure overhead (validation cost on every construction, metaclass complexity): use `@dataclass(slots=True, frozen=True)` for value objects; `kw_only=True` when there are >3 fields. attrs earns its place only when you need what dataclasses lack: validators without full pydantic, `__init_subclass__`-free field inheritance control, or performance-tuned `slots` classes on older Pythons. What changes the answer: if the "internal" type later gets serialized to clients, don't migrate it to pydantic — write an explicit boundary converter; keeping domain objects framework-free is worth the one mapping function.

**"Should this be async?"** Ask: (1) Is the workload I/O-concurrency-bound with many simultaneous waits (>~50 concurrent connections)? If it's <10 concurrent HTTP calls, threads + `concurrent.futures` are simpler and debuggable with ordinary stack traces. (2) Are the libraries I need actually async (asyncpg, httpx, aioboto3)? One sync-only dependency in the hot path means `to_thread` wrappers everywhere — sometimes the honest answer is "stay sync." (3) Is anyone on the team going to maintain this? Async doubles the API surface (two colors of function). The prior: **most Python services are fine with sync + a thread pool; async pays off for gateways, scrapers, and websocket fan-out.**

**"Reach for numpy/polars/Rust?"** Profile first (`cProfile` for call structure, `py-spy` for production sampling without code changes). Then: hot loop over homogeneous numerics → numpy. Tabular transforms/joins/groupbys → polars (multi-threaded, lazy engine; prefer over pandas for new pipelines). Complex per-item logic that resists vectorization → try restructuring into vectorizable steps first; if truly branchy, a Rust extension via PyO3/maturin beats Cython for new work in 2026 (better tooling, no C-level UB). Stopping rule: stop optimizing when the hot function is <20% of wall time or you've hit I/O bound — check with the profiler, not intuition.

## How an expert thinks through it: "my async server stalls under load"

Symptoms: p99 latency spikes, health checks time out, CPU low. Internal monologue: *Low CPU + stalls in an async service = the event loop is blocked, not overloaded. Prior: 80% of the time it's a hidden synchronous call — `requests` instead of `httpx`, a sync DB driver, `time.sleep`, file I/O on NFS, or CPU-heavy JSON/crypto on the loop. First move: turn on the built-in detector — `loop.set_debug(True)` or `PYTHONASYNCIODEBUG=1` logs any callback exceeding 100ms with its location. Also consider: task starvation from a tight `while True` without an `await` (a coroutine only yields at await points — an await-less loop never yields); or GC pauses from huge object graphs (rarer — check last). Suppose debug mode fingers `stripe.Charge.create` — a sync SDK call. Options: (a) wrap in `asyncio.to_thread` — correct and cheap, but audit thread-safety of the SDK and bound concurrency with a semaphore so I don't spawn 500 threads; (b) switch to the SDK's async client if one exists — better long-term; (c) move payment calls to a worker queue — right if calls are slow AND retryable, overkill otherwise. Reject (c) for now: no evidence the latency budget needs it. Ship (a) with a `Semaphore(20)`, file a ticket for (b).* Verification: replay load; the 100ms-callback warnings must disappear.

## Failure modes and pitfalls

- **Fire-and-forget task garbage collection.** `asyncio.create_task(coro())` without keeping a reference: the event loop holds only a *weak* reference, so the task can be GC'd mid-flight and silently vanish. Correct pattern: `task = create_task(...); tasks.add(task); task.add_done_callback(tasks.discard)` — or better, use `asyncio.TaskGroup` (3.11+), which owns its children, propagates failures as `ExceptionGroup` (handle with `except*`), and cancels siblings on error. Structured concurrency via TaskGroup is the default recommendation; bare `create_task` needs a written justification.
- **Swallowed cancellation.** `except Exception` does *not* catch `CancelledError` (it subclasses `BaseException` since 3.8) — good. But `except BaseException:` or bare `except:` around an `await` eats cancellation and makes tasks unkillable; timeouts then hang forever. If you must intercept cancellation for cleanup, re-raise it.
- **`asyncio.gather` vs TaskGroup semantics.** Default `gather` waits for all, and on the first exception it raises while *leaving the other tasks running unsupervised*; `return_exceptions=True` silently converts failures to return values that are easy to forget to check. TaskGroup cancels siblings and never loses an exception. Use `gather` only for "collect results, all must be awaited anyway" with explicit exception handling.
- **Mutable default arguments** — still the classic: `def f(x, acc=[])` shares one list across calls. Use `None` sentinel or `field(default_factory=list)` in dataclasses (dataclasses raise on mutable defaults; plain functions don't).
- **Late-binding closures in loops.** `[lambda: i for i in range(3)]` — all return 2. Fix: `lambda i=i: i` or `functools.partial`.
- **TypedDict trust misplacement.** `def f(d: MovieDict)` then `d["year"]` KeyErrors in production because the dict came from `json.loads`. TypedDict is a static shape claim, not a validator. At boundaries: `pydantic.TypeAdapter(MovieDict).validate_python(raw)` or a pydantic model.
- **`@overload` bodies aren't checked against the overloads.** Type checkers verify call sites against overload signatures but only loosely check the implementation; a wrong implementation passes. Test each overload path at runtime.
- **Protocol + `isinstance` requires `@runtime_checkable`, and even then it only checks method *names*, not signatures** — `isinstance(x, SupportsClose)` passes for a `close(self, wrong, args)` object. Runtime-checkable protocols are duck-typing hints, not contracts.
- **Import-time side effects and circulars.** `from app.db import engine` where `db.py` creates the engine at top level: now test collection needs a database. Fix: lazy initialization (`functools.lru_cache`-wrapped `get_engine()`), or app-factory pattern. For circular imports where only annotations need the other module: `if TYPE_CHECKING:` import (works cleanly now that annotations are deferred by default in 3.14).
- **stdlib gems experts actually use:** `itertools.pairwise/batched` (`batched` since 3.12), `functools.cache`/`cached_property`, `collections.Counter/defaultdict`, `pathlib` everywhere, `shutil.which`, `textwrap.dedent`, `tomllib` (read-only TOML, 3.11+), `zoneinfo` (never pytz in new code — `pytz.localize` misuse causes the classic LMT offset bug), `dataclasses.replace`, `contextlib.ExitStack` for dynamic numbers of context managers, `subprocess.run([...], check=True, capture_output=True, text=True)` — always a list argv, never `shell=True` with interpolation.
- **stdlib footguns:** `datetime.utcnow()` is deprecated and naive — use `datetime.now(timezone.utc)`; `json.dumps` silently accepts NaN (invalid JSON — pass `allow_nan=False` at boundaries); `copy.copy` on nested structures shares interiors; `str.strip("suffix")` strips a *character set*, not a suffix — use `removesuffix` (3.9+); `os.path.join("/a", "/b")` returns `/b`.

## Worked micro-examples

**Structured concurrency with bounded fan-out (the 2026-default shape):**
```python
import asyncio, httpx

async def fetch_all(urls: list[str]) -> dict[str, str]:
    results: dict[str, str] = {}
    sem = asyncio.Semaphore(20)          # bound concurrency explicitly
    async with httpx.AsyncClient(timeout=10.0) as client:
        async with asyncio.TaskGroup() as tg:   # owns children; no GC'd tasks
            async def one(u: str) -> None:
                async with sem:
                    r = await client.get(u)
                    r.raise_for_status()
                    results[u] = r.text
            for u in urls:
                tg.create_task(one(u))
    return results  # reaching here proves every task finished or raised
```

**uv project skeleton (pyproject.toml essentials):**
```toml
[project]
name = "svc"
requires-python = ">=3.12"
dependencies = ["httpx>=0.28", "pydantic>=2.7"]

[dependency-groups]          # PEP 735; uv-native dev deps
dev = ["pytest>=8", "ruff>=0.5", "pyright>=1.1"]
```
Workflow: `uv add httpx` (updates pyproject + uv.lock), `uv sync` (reproduce env), `uv run pytest` (never activate venvs manually in scripts), `uv export -o pylock.toml` when another tool needs the standard lock format. Lint/format with ruff (it replaced black+isort+flake8 in most stacks).

## Verification and stopping rule

Before presenting Python advice or code: (1) run it mentally at *import time* — does anything execute that shouldn't? (2) for async code, trace every task to an owner (TaskGroup or stored reference) and every `except` for swallowed `CancelledError`; (3) for typing claims, remember which guarantees are static-only — would this survive an untyped caller? (4) for performance claims, demand a profile — never assert a bottleneck you haven't measured; (5) check version floors: PEP 695 syntax needs 3.12+, TaskGroup/`except*` need 3.11+, free-threading claims need the `t` build. Stop when the boundary is validated, tasks are owned, and the profiler says the remaining time is I/O — further micro-optimization of Python bytecode is almost always waste.
