---
name: systems-programming
description: Load when writing or reviewing C/C++/Rust or other low-level code, reasoning about memory (stack/heap/mmap/page faults), syscall or I/O performance (epoll, io_uring, read vs mmap), undefined behavior, signal handlers, or when debugging with strace, perf, valgrind, or sanitizers.
---

# Systems Programming

Compact checklist. The standard expert material — UB exploitation (`x+1<x` folds to false), overcommit/page-fault mechanics and OOM-kill, read-vs-mmap tradeoffs incl. SIGBUS-on-truncate and the fault-hides-I/O profiling artifact, epoll ET drain-until-EAGAIN, syscall-cost batching, async-signal-safety and the self-pipe/signalfd pattern, fork-in-threads deadlock, `volatile`-is-not-atomic, RSS retention vs leaks, false sharing, sleep-"fixed" races — is assumed known and appears only as anchors.

## Anchors (apply without re-derivation)

- UB is license to delete: signed overflow checks, dereference-then-NULL-check, `while (!plain_bool)` — use `__builtin_add_overflow`, check before the op, `std::atomic`. "Works in debug, breaks in release" = run ASan/UBSan on the *debug* build; the bug is there, latent.
- Triage before optimizing: `time ./prog` — real≫user+sys = blocked (strace -T, offcputime); sys≫user = syscall-bound (`strace -c`); user-dominant = CPU or memory-bound (perf stat: IPC < ~1 + high LLC misses = fix data layout, not instruction count).
- Syscalls ≈ 100ns–1µs each (worse with mitigations): astronomical `calls` with `usecs/call ≈ 1` in `strace -c` means batching, not faster hardware.
- epoll: readiness not data; ET requires nonblocking fds + drain loops; "ready" can still EAGAIN; regular files are always ready (`epoll_ctl` gives EPERM) — files need io_uring or threads.
- Signal handler: set `sig_atomic_t` flag or write one byte to a self-pipe/eventfd; nothing else. `SA_RESTART` does not cover `select`/`poll`/`epoll_wait` — every blocking syscall still needs an EINTR loop.
- Every raw `read`/`write` needs a short-count loop; `read()==0` is EOF, distinct from -1; store results in `ssize_t` (a size_t hides the -1); `O_CLOEXEC` everywhere.
- Between `fork` and `exec` in a threaded process: async-signal-safe calls only; prefer `posix_spawn`.
- RSS high after free = usually allocator retention/fragmentation, not a leak — confirm with LSan/`malloc_trim(0)`; for daemons watch the *slope*, not the level.
- Sanitizer matrix: ASan (+LSan) for memory, UBSan always-on in test builds, TSan in a separate CI job (incompatible with ASan). "Compiles and passes tests" does not exclude UB.

## Corrections and calibrations (where reflexes go wrong)

- **Thread-stack sizes are a platform trap, not a constant.** glibc pthreads inherit `RLIMIT_STACK` (typically 8MB) so code "works" on Linux — then dies on **musl/Alpine (128KB default)** or **macOS non-main threads (512KB)**. Deep recursion or large frames that pass CI on glibc are the classic container-port SIGSEGV; the tell in a core dump is a faulting address just below a thread's stack region. Set `pthread_attr_setstacksize` explicitly for anything recursive, or move buffers to the heap.
- **Ownership reasoning is the borrow checker run by hand — apply it to "read-only-looking" calls.** The killer C/C++ pattern is a pointer derived from a structure crossing a call that can mutate that structure *lazily* (cache refresh, rehash, realloc, `c_str()` then grow). Mechanical check for every pointer in review: who owns it, when does it die, does any mutation of the owner cross its live range — including calls that merely *look* const.
- **Benchmark hygiene is fault accounting, not repetition.** First run over a file measures the disk; second measures RAM. First pass over fresh allocations measures page faults, not your algorithm. Report `minflt`/`majflt` from `/usr/bin/time -v` alongside timings; a "20% speedup" that coincides with a fault-count drop is a caching artifact, not your optimization.
- **Cache-line padding: pad to 128 bytes when it matters.** The reflex `alignas(64)` can still false-share on CPUs with adjacent-line prefetchers (Intel fetches line pairs); per-thread hot slots want 128-byte separation. Confirm contention with `perf c2c` (HITM) before and after — padding "fixes" that don't move HITM counts were placebo.

## Verification / self-check

- Any correctness claim: has it run under `-fsanitize=address,undefined` (TSan separately if threaded)? If not, say so.
- Any performance claim: name the evidence category (syscall counts / on-CPU profile / hardware counters) and confirm the fix moved *that* number; report fault counts with timings.
- Every pointer: owner, lifetime, mutations crossing its live range. Every blocking syscall: short counts, EAGAIN, EINTR. Every handler: async-signal-safe list only.
- "The compiler wouldn't do that": check the standard's contract, then `-O2 -S` when stakes justify.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)
- Probed 14 claims: 13 baseline (compressed to anchors), 1 partial (thread-stack defaults — baseline knew musl 128KB; sharpened into the glibc/musl/macOS porting trap and fixed this skill's own imprecise "often 2MB" claim), 0 delta.
- Opus cold reproduced: UB folding with correct alternatives, overcommit/fault costs/OOM mechanics, read-vs-mmap incl. SIGBUS and the profiling artifact, ET drain pattern + EPERM on files + io_uring, full triage methodology, signal-safety, fork hazards, RSS retention, false sharing incl. perf c2c, exhaustive raw-I/O bug list.
- Kept expanded: platform stack-size trap; borrow-checker-by-hand for lazily-mutating calls; fault-accounting benchmark hygiene; 128-byte padding vs adjacent-line prefetch.
