---
name: systems-programming
description: Load when writing or reviewing C/C++/Rust or other low-level code, reasoning about memory (stack/heap/mmap/page faults), syscall or I/O performance (epoll, io_uring, read vs mmap), undefined behavior, signal handlers, or when debugging with strace, perf, valgrind, or sanitizers.
---

# Systems Programming

## Core mental model

- **Virtual memory is a lazy promise.** `malloc`/`mmap` hand out address space, not RAM. Physical pages materialize on first touch via page faults — a *minor* fault (page exists or is zero-fill; ~microsecond kernel bookkeeping) vs a *major* fault (disk I/O; ~ms on flash, catastrophic in a hot loop). Consequences: "allocated 8GB instantly" means nothing happened yet; the first pass over a big buffer is slow (fault-in) and every later pass fast; RSS ≠ VSZ; overcommit means the OOM killer, not `malloc` returning NULL, is how you usually die on Linux.
- **The compiler optimizes against a contract, not your mental model of the hardware.** Undefined behavior isn't "the value is garbage" — it's "the optimizer may assume this never happens" and deletes code accordingly: signed-overflow checks written as `if (x + 1 < x)` get compiled out, NULL checks after a dereference get removed, infinite loops without side effects vanish. Reason from the language contract; verify with sanitizers, not with "it worked when I ran it".
- **Syscalls cost ~hundreds of nanoseconds to ~a microsecond each — trivial alone, dominant in a loop.** One `write(fd, buf, 1)` per byte turns a 100MB/s task into a syscall benchmark. The whole design space of buffered stdio, `readv`/`writev`, epoll, and io_uring is one idea: amortize kernel crossings by batching. When something is mysteriously slow, `strace -c` first — count the calls before optimizing the code between them.
- **Ownership/lifetime reasoning is the portable discipline; Rust's borrow checker is just its mechanization.** For every allocation and every pointer, be able to answer: who frees this, and what guarantees the pointer doesn't outlive or alias-mutate it? Apply the borrow-checker rules mentally in C/C++: one mutable reference XOR many readers; no reference outlives its owner; anything that grows a container invalidates pointers into it. Most C/C++ memory bugs are exactly the programs the borrow checker would have rejected.
- **Know where your time actually goes before touching code.** On-CPU (perf), off-CPU/blocked (strace, `/proc/<pid>/stack`, offcputime), or memory-bound (perf counters: IPC < 1 with high cache misses). Each has disjoint fixes; guessing the category wrong wastes the whole optimization effort.

## Decision frameworks

**Stack vs heap:**
- Stack: fixed-size, scope-bounded, small (KBs). Free to allocate (a `sub rsp`), cache-hot, no fragmentation. Default for anything under ~a few KB with static size.
- Heap: dynamic size, outlives scope, large. Costs allocator locks/bookkeeping and cache misses.
- Never: recursion with per-frame big arrays (thread stacks default ~8MB main / often 2MB or less for pthreads — a 64KB buffer × 100 frames overflows); returning pointers to stack locals (UB the moment the frame pops, and it *will* "work" in tests because the memory isn't reused yet); `alloca`/VLAs with attacker-influenced sizes (stack overflow → security bug).
- Middle path for hot loops: reuse one heap buffer across iterations (or an arena) instead of per-iteration malloc/free — allocator traffic, not allocation size, is what shows up in profiles.

**read() vs mmap for file I/O:**

| Situation | Use | Because |
|---|---|---|
| Sequential single pass (parsing, streaming, checksum) | `read()` with 64KB–1MB buffer | Readahead makes it near-optimal; no page-table churn; predictable error handling (`read` returns -1; mmap'd I/O errors arrive as SIGBUS) |
| Random access into a large file, re-read regions, or shared across processes | `mmap` | No syscall per access; page cache is your buffer; shared mappings dedupe memory |
| File may be truncated/modified by others while mapped | `read()` | Accessing a mapped page past a truncation point = SIGBUS — nearly impossible to handle cleanly |
| Many small files | `read()` (or io_uring) | mmap/munmap + TLB shootdowns per file dwarf the read cost |

mmap also silently turns "I/O wait" into "page fault inside your parsing code" — profiles look like the parser is slow when it's the disk.

**Event loop / async I/O model:**
- Mental model of `epoll`: you register fds once (`epoll_ctl`), then `epoll_wait` returns *readiness*, not data — you still `read()` after. Level-triggered (default): keeps reporting while data remains — forgiving. Edge-triggered (`EPOLLET`): reports only on *transitions* — you must loop `read()` until `EAGAIN` on every event or you hang forever with data sitting in the buffer (the classic ET bug); requires nonblocking fds.
- Readiness is a hint, not a guarantee: a wakeup can race with another consumer, so a "ready" fd can still return `EAGAIN` — always handle it.
- `epoll` doesn't help with regular files (always "ready"; reads still block on disk). For file I/O concurrency you need threads or io_uring.
- Mental model of `io_uring`: two shared ring buffers — you enqueue *operations* (not registrations) into the submission queue, kernel executes async, results appear in the completion queue. It's completion-based (like Windows IOCP), covers files as well as sockets, and can batch many ops per syscall (or zero syscalls with SQPOLL). Choose it for high-IOPS file/storage workloads and syscall-bound servers; plain epoll remains fine for ordinary network servers and has decades fewer sharp edges.
- Blocking threads (one per connection) are the right answer up to a few thousand connections — don't pay async complexity for 50 clients.

**Which tool answers which question:**

| Question | Tool | Invocation / read-out |
|---|---|---|
| What syscalls, how many, which fail? | strace | `strace -f -c ./prog` for counts; `-e trace=network -T` for per-call latency. 10–100× slowdown — never on hot prod; use `perf trace` there |
| Where is CPU time going? | perf | `perf record -g --call-graph dwarf ./prog; perf report`. Needs `-fno-omit-frame-pointer` or DWARF for real stacks; flamegraph it |
| Why is it slow while CPU is idle? | off-CPU analysis | strace `-T` on the blocking calls; bcc `offcputime`; check run-queue vs iowait in `vmstat` |
| Memory corruption (use-after-free, OOB)? | ASan first, valgrind second | `-fsanitize=address` at build: ~2× slowdown; valgrind memcheck: no rebuild needed but 20–50×. ASan catches stack/global OOB that valgrind misses |
| Leaks? | LSan (in ASan) or `valgrind --leak-check=full` | "Still reachable" ≠ leak; "definitely lost" is the actionable line |
| Data races? | TSan | `-fsanitize=thread`, incompatible with ASan in the same build — separate CI jobs |
| UB (overflow, misalignment, bad shifts)? | UBSan | `-fsanitize=undefined` — cheap enough to keep on in all test builds |
| Cache/branch behavior? | perf stat | `perf stat -d ./prog`; IPC < 1 with high LLC-miss% = memory-bound → fix data layout, not instruction count |

## Failure modes & pitfalls

- **UB classes the compiler actively exploits** — treat each as "this code path gets deleted": signed integer overflow (`x+1 < x` check compiled away — use `__builtin_add_overflow` or unsigned); dereference-then-NULL-check (check removed because the deref "proved" non-null); out-of-bounds and use-after-free (ASan territory); strict aliasing (`*(float*)&i` — use `memcpy`, it compiles to the same register move); data races (unsynchronized non-atomic access lets the compiler cache values in registers forever — your `while (!done_flag)` spins eternally without `atomic`); uninitialized reads; shift ≥ bit-width; `memcpy` with overlapping ranges (use `memmove`).
- **"It works in debug, breaks in release."** Almost always UB whose exploitation is optimization-dependent, or an uninitialized variable that debug's zeroed stack hid. Don't bisect optimization flags — run UBSan/ASan on the debug build; the bug is present there too, just latent.
- **Benchmarking the page cache / the allocator instead of your code.** First file-read benchmark run measures disk, subsequent ones measure memory (drop caches with `echo 3 > /proc/sys/vm/drop_caches` or compare both deliberately). First pass over fresh mallocs measures page faults. Compare `minflt`/`majflt` in `/usr/bin/time -v` between runs.
- **Signal handler calls something non-async-signal-safe.** `printf`, `malloc`, and anything locking are forbidden in handlers: if the signal lands while the main thread holds the allocator/stdio lock, the handler deadlocks or corrupts the heap — rare, unreproducible, prod-only. The safe list is roughly: `write`, `_exit`, `sig_atomic_t` flag sets, `signalfd`-style designs. Correct pattern: handler sets a `volatile sig_atomic_t` flag or `write()`s one byte to a self-pipe/`eventfd`; the event loop does the real work. Also: check `EINTR` on every blocking syscall (or install with `SA_RESTART` and know which calls it doesn't cover — `select`/`poll`/`epoll_wait` still return `EINTR`).
- **Iterator/pointer invalidation.** `std::vector::push_back` inside a loop holding a reference/iterator into the same vector — realloc moves the buffer, the reference dangles. Same bug wearing other clothes: `char *p = s.c_str()` then growing `s`; keeping a pointer into a rehashed hash map. Borrow-checker framing: you held a reference while mutating the owner.
- **Use-after-move / double-free from unclear ownership.** In C: two structs both storing a pointer, both "cleaning up". Fix by *documenting* ownership at every API boundary ("callee takes ownership", "borrowed, valid until X") — the comment IS the type system in C. In C++: `unique_ptr` by default; raw pointers mean "borrowed, never freed by holder".
- **Off-by-default thread-stack assumptions.** pthread stacks are commonly 2–8MB; a deep-recursion or big-frame function that works on the main thread overflows in a worker → SIGSEGV at an address just below the stack — the tell is the fault address adjacent to the stack region in the core dump.
- **Confusing RSS growth with a leak.** glibc malloc rarely returns freed memory to the OS (arena fragmentation); RSS plateaus high without a leak. Confirm with LSan/valgrind before "fixing". Conversely, real leaks in long-lived daemons: watch *slope*, not level.
- **Retrying `write()` wrong.** `write` may write *fewer* bytes than asked (short write) without error — code that doesn't loop drops data on sockets and pipes. Same for `read`. Every raw `write(fd, buf, n)` needs a loop or `writev`-with-tracking.
- **fork() in a threaded program.** Child inherits only the calling thread but ALL lock states — if another thread held the malloc lock at fork, the child deadlocks on its first allocation. Only async-signal-safe calls between `fork` and `exec` in threaded processes; prefer `posix_spawn`.
- **False sharing.** Two atomics/counters on the same 64-byte cache line, updated by different threads → line ping-pongs between cores; perf shows huge time on an innocuous increment. Fix: `alignas(64)` per-thread counters, aggregate on read.

## Worked micro-examples

**1. UB exploitation, concretely:**

```c
int overflows(int x)              /* intent: detect INT_MAX */
{
    return x + 1 < x;             /* signed overflow is UB          */
}
```
At `-O2`, GCC/Clang compile this to `return 0;` — the optimizer assumes `x+1` never overflows, so `x+1 < x` is always false, so your guard is gone and the caller proceeds into the overflow it "checked" for. Correct: `return x == INT_MAX;` or `int r; return __builtin_add_overflow(x, 1, &r);`. Verify: `clang -O2 -S` shows `xor eax,eax; ret`, and UBSan flags the original at runtime.

**2. Syscall-count diagnosis with strace.** A log-processing tool does 4MB/s on an NVMe drive. `strace -c ./tool file.log` shows:

```
% time     seconds  usecs/call     calls    errors syscall
 96.1    11.90        1          11534336           read
```
11.5M `read` calls for a 11.5MB file → someone is calling `read(fd, buf, 1)` (an unbuffered `fgetc`-style loop over a raw fd). Fix: wrap in a 256KB user-space buffer (or use stdio, which does this for you) → ~44 reads total. Throughput goes from syscall-bound (~1µs each → ~1MB/s per byte-loop) to disk-bound. The lesson generalizes: when `usecs/call ≈ 1` and `calls` is astronomical, the fix is batching, not a faster disk.

**3. Ownership reasoning preventing a C bug (borrow-checker mental model):**

```c
const char *name = get_user(db, id)->name;   /* borrow into db's cache   */
refresh_cache(db);                            /* mutates owner: may free/realloc entries */
printf("%s\n", name);                         /* use-after-free           */
```
Borrow-checker framing: `name` borrows from `db`; `refresh_cache(db)` needs exclusive (mutable) access to `db`; a live borrow across a mutation is exactly what rustc rejects (`cannot borrow db as mutable because it is also borrowed as immutable`). The C fix is the same thing rustc would force: end the borrow first — `strdup` the name (take ownership) or reorder so the borrow's lifetime doesn't cross the mutation. Apply this check mechanically whenever a pointer derived from a structure crosses a call that can mutate that structure.

## Verification / self-check

- Any C/C++ claim of correctness: has it run under `-fsanitize=address,undefined` (and TSan separately if threaded)? If not, say so — "compiles and passes tests" does not exclude UB.
- Any performance claim: name the evidence category — syscall counts (strace -c), on-CPU profile (perf), or hardware counters (perf stat) — and check the fix moved *that* number. "It feels faster" is not a measurement; neither is one warm-cache run.
- For every pointer in reviewed code: who owns it, when does it die, and does any mutation of the owner cross its live range?
- For every blocking syscall: is a short read/write, `EAGAIN`, and `EINTR` handled?
- For every signal handler: is every call in it on the async-signal-safe list?
- For any "the compiler/hardware wouldn't do that" argument: check the standard's contract, not intuition — then confirm with `-O2 -S` or godbolt-style disassembly if the stakes justify it.
