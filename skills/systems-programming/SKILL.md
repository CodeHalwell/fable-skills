---
name: systems-programming
description: Load when writing or reviewing C/C++/Rust or other low-level code, reasoning about memory (stack/heap/mmap/page faults), syscall or I/O performance (epoll, io_uring, read vs mmap), undefined behavior, signal handlers, or when debugging with strace, perf, valgrind, or sanitizers.
---

# Systems Programming

## Core mental model

- **Virtual memory is a lazy promise.** `malloc`/`mmap` hand out address space, not RAM. Physical pages materialize on first touch via page faults — a *minor* fault (page already in page cache or zero-fill; ~a microsecond of kernel bookkeeping) vs a *major* fault (disk I/O; ~100µs–ms; catastrophic in a hot loop). Consequences: "allocated 8GB instantly" means nothing happened yet; the first pass over a big buffer is slow (fault-in) and later passes fast; RSS ≠ VSZ; and with Linux overcommit, the OOM killer — not `malloc` returning NULL — is how you usually die.
- **The compiler optimizes against a contract, not your mental model of the hardware.** Undefined behavior isn't "the value is garbage" — it's "the optimizer may assume this never happens" and deletes code accordingly: signed-overflow checks written as `if (x + 1 < x)` get compiled out, NULL checks after a dereference get removed, infinite loops without side effects vanish. Reason from the language contract; verify with sanitizers, never with "it worked when I ran it".
- **Syscalls cost roughly hundreds of nanoseconds to a microsecond each — trivial alone, dominant in a loop.** One `write(fd, buf, 1)` per byte turns a 100MB/s task into a syscall benchmark. The entire design space of buffered stdio, `readv`/`writev`, epoll batching, and io_uring is one idea: amortize kernel crossings. When something is mysteriously slow, run `strace -c` first — count the calls before optimizing the code between them.
- **Ownership/lifetime reasoning is the portable discipline; Rust's borrow checker is its mechanization.** For every allocation and every pointer, be able to answer: who frees this, and what guarantees the pointer doesn't outlive the object or get invalidated by a mutation? Apply the borrow-checker rules mentally in C/C++: one mutable reference XOR many readers; no reference outlives its owner; anything that can grow or rehash a container invalidates pointers into it. Most C/C++ memory bugs are exactly the programs the borrow checker would have rejected.
- **Know which of three budgets you're spending before touching code.** On-CPU (perf), off-CPU/blocked (strace -T, offcputime, iowait), or memory-bound (perf stat: IPC < 1 with high cache-miss rates). Each category has disjoint fixes — inlining helps none of your I/O waits; a better disk helps none of your cache misses. Guessing the category wrong wastes the whole optimization effort.

## Decision frameworks

### Stack vs heap

- **Stack:** fixed-size, scope-bounded, small. Allocation is free (a `sub rsp`), memory is cache-hot, no fragmentation, no allocator contention. Default for anything under a few KB with a statically known size.
- **Heap:** dynamic size, outlives its creating scope, large objects. Costs allocator locking/bookkeeping and colder cache.
- Hard rules:
  - No recursion with large per-frame arrays: thread stacks default to ~8MB for the main thread and often 2MB or less for pthreads — a 64KB buffer × 100 frames overflows a worker thread that "worked on main".
  - Never return a pointer/reference to a stack local — UB the moment the frame pops, and it *will* pass tests because the memory hasn't been reused yet.
  - No `alloca`/VLAs with attacker-influenced sizes: an unchecked size is a stack-overflow security bug, not a crash.
- Middle path for hot loops: reuse one heap buffer across iterations, or use an arena/bump allocator — allocator *traffic*, not allocation size, is what shows up in profiles.

### read() vs mmap for file I/O

| Situation | Use | Because |
|---|---|---|
| Sequential single pass (parsing, streaming, checksumming) | `read()` with a 64KB–1MB buffer | Kernel readahead makes it near-optimal; no page-table churn; errors arrive as `-1/errno`, not signals |
| Random access into a large file, re-read regions, shared across processes | `mmap` | No syscall per access; the page cache is your buffer; `MAP_SHARED` dedupes memory across processes |
| File may be truncated or modified by another process while mapped | `read()` | Touching a mapped page past the truncation point raises SIGBUS — nearly impossible to handle cleanly |
| Many small files | `read()` (or io_uring) | Per-file mmap/munmap plus TLB shootdowns dwarf the read cost |

Also: mmap silently converts "I/O wait" into "page fault inside your parsing loop" — profiles then blame the parser for what is actually the disk. Check `majflt` before believing such a profile.

### Event loop / async I/O model

- **epoll mental model:** register fds once (`epoll_ctl`), then `epoll_wait` returns *readiness*, not data — you still call `read()` afterward.
  - Level-triggered (default): keeps reporting readiness while data remains. Forgiving; use it unless you've measured a reason not to.
  - Edge-triggered (`EPOLLET`): reports only *transitions*. You must loop `read()`/`accept()` until `EAGAIN` on every event, on nonblocking fds, or you hang forever with data sitting in the socket buffer — the classic ET bug.
  - Readiness is a hint, not a promise: wakeups can race with other consumers, so a "ready" fd can still return `EAGAIN`. Handle it unconditionally.
  - epoll does not help with regular files — they're always "ready" and reads still block on disk. File-I/O concurrency needs threads or io_uring.
- **io_uring mental model:** two shared ring buffers. You enqueue *operations* (not fd registrations) into the submission queue; the kernel executes them asynchronously; results appear in the completion queue. It is completion-based (like Windows IOCP), covers files as well as sockets, and batches many ops per syscall (or zero syscalls with SQPOLL). Choose it for high-IOPS storage workloads and syscall-bound servers; plain epoll remains fine for ordinary network servers and has years fewer sharp edges (and fewer seccomp/container restrictions).
- **Threads-per-connection is correct up to a few thousand connections.** Don't pay the async-complexity tax for 50 clients; a blocking thread is the easiest correct concurrency there is.

### Which tool answers which question

| Question | Tool | Invocation / how to read it |
|---|---|---|
| What syscalls, how many, which fail? | strace | `strace -f -c ./prog` for the count table; `-e trace=network -T` for per-call latency. 10–100× slowdown — never attach to hot prod; use `perf trace` there |
| Where is CPU time going? | perf | `perf record -g --call-graph dwarf ./prog; perf report`. Needs frame pointers (`-fno-omit-frame-pointer`) or DWARF for honest stacks; render a flamegraph for wide views |
| Why slow while CPU is idle? | off-CPU analysis | `strace -T` on the blocking calls; bcc `offcputime`; `vmstat` to split run-queue vs iowait |
| Memory corruption (UAF, OOB)? | ASan first, valgrind second | `-fsanitize=address` at build, ~2× slowdown; valgrind memcheck needs no rebuild but runs 20–50×. ASan catches stack/global overflows that valgrind misses |
| Leaks? | LSan (bundled with ASan) or `valgrind --leak-check=full` | "Still reachable" is usually not a leak; "definitely lost" is the actionable line |
| Data races? | TSan | `-fsanitize=thread`; incompatible with ASan in one build — separate CI jobs |
| UB (overflow, misalignment, bad shifts)? | UBSan | `-fsanitize=undefined`; cheap enough to leave on in every test build |
| Cache/branch behavior? | perf stat | `perf stat -d ./prog`; IPC < 1 with high LLC-miss% = memory-bound → fix data layout (SoA, smaller nodes, fewer pointers), not instruction count |

## Failure modes & pitfalls

- **UB classes the compiler actively exploits** — treat each as "this code path may be deleted":
  - Signed integer overflow: `if (x + 1 < x)` compiles to `if (false)`. Use `__builtin_add_overflow`, or do the arithmetic unsigned.
  - Dereference-then-NULL-check: the check is removed because the dereference "proved" the pointer non-null.
  - Out-of-bounds access and use-after-free: anything at all, including "works" — ASan territory.
  - Strict aliasing: `*(float*)&i` — use `memcpy` (compiles to the identical register move) or C++20 `bit_cast`.
  - Data races: unsynchronized non-atomic access lets the compiler cache the value in a register forever — your `while (!done_flag)` spins eternally. Use `std::atomic`/`_Atomic`, not `volatile` (volatile orders nothing between threads).
  - Uninitialized reads; shifts ≥ bit width; `memcpy` on overlapping ranges (use `memmove`); calling into UB via "harmless" macros.
- **"Works in debug, breaks in release."** Almost always UB whose exploitation is optimization-dependent, or an uninitialized variable that debug's zeroed stack hid. Don't bisect optimization flags to "fix" it — run UBSan/ASan on the debug build; the bug is present there too, just latent.
- **Benchmarking the page cache or the allocator instead of your code.** First file-read run measures the disk; subsequent runs measure RAM — either drop caches (`echo 3 > /proc/sys/vm/drop_caches`) or report both deliberately. First pass over fresh allocations measures page faults, not your algorithm. Compare `minflt`/`majflt` from `/usr/bin/time -v` across runs before trusting any number.
- **Signal handler calls something non-async-signal-safe.** `printf`, `malloc`, and anything that takes a lock are forbidden in handlers: if the signal lands while the main thread holds the allocator or stdio lock, the handler deadlocks or corrupts the heap — rare, unreproducible, prod-only. The safe pattern: the handler sets a `volatile sig_atomic_t` flag or `write()`s one byte to a self-pipe/`eventfd`; the event loop does the real work. Related: handle `EINTR` on every blocking syscall — or install handlers with `SA_RESTART` and know it doesn't cover `select`/`poll`/`epoll_wait`, which still return `EINTR`.
- **Iterator/pointer invalidation.** `push_back` inside a loop that holds a reference or iterator into the same vector — reallocation moves the buffer and the reference dangles. The same bug in other clothes: `const char *p = s.c_str()` then growing `s`; holding a pointer into a hash map across an insert that rehashes. Borrow-checker framing: you held a borrow across a mutation of the owner.
- **Use-after-move / double-free from unclear ownership.** In C: two structs both store the pointer, both "clean up". Fix by *documenting* ownership at every API boundary ("callee takes ownership" / "borrowed; valid until X") — in C, the comment is the type system. In C++: `unique_ptr` by default; a raw pointer means "borrowed, holder never frees"; `shared_ptr` is a last resort, not a default (cycles leak; refcount contention).
- **Thread-stack assumptions.** Deep recursion or big frames that work on the 8MB main stack overflow a 2MB worker stack → SIGSEGV whose faulting address sits just below a thread's stack region in the core dump — that adjacency is the tell.
- **Confusing RSS growth with a leak.** glibc malloc rarely returns freed memory to the OS (arena fragmentation), so RSS plateaus high with zero leaks. Confirm with LSan or valgrind before "fixing". Conversely, for real leaks in daemons, watch the *slope* of RSS over hours, not the level.
- **Short reads/writes unhandled.** `write` may legally write fewer bytes than requested with no error, especially on sockets and pipes; code that doesn't loop silently drops data. Same for `read`. Every raw `read`/`write` needs a loop (or `writev` with tracking) — this is the most common bug in hand-rolled I/O.
- **fork() in a threaded program.** The child inherits only the calling thread but ALL lock states: if another thread held the malloc lock at fork time, the child deadlocks on its first allocation. Between `fork` and `exec`, only async-signal-safe functions are safe in a threaded process; prefer `posix_spawn`.
- **False sharing.** Two counters/atomics on the same 64-byte cache line, written by different threads → the line ping-pongs between cores and perf shows massive time on an innocent-looking increment. Fix: `alignas(64)` per-thread slots, aggregate at read time.
- **Missing memory-order reasoning papered over with sleeps.** A test that stops failing when you add `usleep` has a data race or ordering bug, not a timing bug. Find it with TSan; fix it with proper synchronization (mutex, or acquire/release atomics if you can justify each ordering in a comment — if you can't write the comment, use `seq_cst` or a mutex).

## Worked micro-examples

### 1. UB exploitation, concretely

```c
int overflows(int x)              /* intent: detect INT_MAX */
{
    return x + 1 < x;             /* signed overflow is UB */
}
```

At `-O2`, GCC and Clang compile this to `return 0;` — the optimizer assumes `x+1` never overflows, so `x+1 < x` is always false, the guard is gone, and the caller proceeds into the very overflow it "checked" for. Correct versions: `return x == INT_MAX;` or `int r; return __builtin_add_overflow(x, 1, &r);`. Verify the claim yourself: `clang -O2 -S` emits `xor eax, eax; ret`, and UBSan flags the original at runtime with `signed integer overflow`.

### 2. Syscall-count diagnosis with strace

A log-processing tool manages 4MB/s on an NVMe drive. `strace -c ./tool file.log`:

```
% time     seconds  usecs/call     calls    errors syscall
 96.1    11.90        1          11534336           read
```

11.5M `read` calls for an 11.5MB file → something is calling `read(fd, buf, 1)` — an unbuffered byte-at-a-time loop over a raw fd. Fix: wrap the fd in a 256KB user-space buffer (or use stdio, which exists to do exactly this) → ~44 reads total; throughput goes from syscall-bound to disk-bound. The general lesson: when `usecs/call ≈ 1` and `calls` is astronomical, the fix is batching, not faster hardware — and `strace -c` found it in ten seconds without reading any source.

### 3. Ownership reasoning preventing a C bug (borrow checker as mental model)

```c
const char *name = get_user(db, id)->name;   /* borrow into db's cache */
refresh_cache(db);                            /* mutates owner: may free/realloc entries */
printf("%s\n", name);                         /* use-after-free */
```

Borrow-checker framing: `name` immutably borrows from `db`; `refresh_cache(db)` requires exclusive (mutable) access; a live borrow across an exclusive mutation is exactly what rustc rejects (`cannot borrow *db as mutable because it is also borrowed as immutable`). The C fix is whatever rustc would have forced: end the borrow first — `strdup(name)` to take ownership, or reorder so the borrow's live range doesn't cross the mutation. Apply this check mechanically whenever a pointer derived from a structure crosses any call that can mutate that structure — including "read-only-looking" calls that may lazily rebuild caches.

## Verification / self-check

- Any C/C++ correctness claim: has it run under `-fsanitize=address,undefined` (and TSan separately if threaded)? If not, say so explicitly — "compiles and passes tests" does not exclude UB, and UB does not reliably crash.
- Any performance claim: name the evidence category — syscall counts (`strace -c`), on-CPU profile (perf), hardware counters (`perf stat`) — and confirm the fix moved *that* number. One warm-cache run is not a measurement; report fault counts alongside timings.
- For every pointer in reviewed code: who owns it, when does it die, and does any mutation of the owner cross its live range?
- For every blocking syscall: are short reads/writes, `EAGAIN`, and `EINTR` handled?
- For every signal handler: is every function it calls on the async-signal-safe list?
- For any "the compiler/hardware wouldn't do that" argument: check the standard's contract, not intuition — then confirm with `-O2 -S` or a godbolt-style disassembly when the stakes justify it.
