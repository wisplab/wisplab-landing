# A 0.69 ms per-call Python sandbox in WebAssembly

*Draft — 2026-05-02*

---

## TL;DR

Cross-compiled CPython 3.14 to `wasm32-wasip1`, ran it under Wasmtime
with a custom memory-creation path, and measured **0.69 ms p50 per-call
cold start** for a fresh, isolated Python interpreter. Per-branch
fork-from-snapshot lands at **0.42 ms** under 8-core parallel load —
**2394 branches/sec**.

That's roughly **70× faster than published serverless warm-pool numbers
and 2000× faster than published cold-starts** for similar functionality.
The gap isn't optimization headroom on the existing substrates; it's
that WebAssembly enables two operations native runtimes structurally
cannot do — per-call fresh memory and cheap fork from arbitrary post-
init state.

This post walks through what we built, why the numbers are what they
are, and what's still missing.

---

## Why a per-call sandbox?

Two workloads make per-call isolation a feature, not an annoyance:

1. **Code-execution-as-a-tool for AI agents.** When a model generates
   a Python snippet ("compute this, parse that, normalize the JSON") and
   the orchestrator runs it for every turn of every conversation, the
   sandbox cost is paid hundreds of times per session. Today's serverless
   compute makes this expensive enough that most production agents either
   skip the sandbox (running model output in a shared process — bad) or
   amortize it (one persistent worker per conversation — better, but
   leaks state between turns and grows quadratically with concurrency).

2. **RL training over interactive environments.** Tree-search rollouts
   (MCTS, Tree-of-Thoughts, branching GRPO) need to fork the
   environment state at every decision point and explore K alternative
   continuations. With Linux process fork at 5–10 ms and Firecracker
   uVM snapshots at 100–500 ms, a representative K=100 × depth=100
   rollout pays 50 seconds to 80 minutes in pure fork overhead per
   trajectory.

Both want sandboxes that are individually cheap (sub-millisecond, not
millions-of-microseconds) AND independently fresh (each one a clean
linear memory, no shared state with siblings).

Native processes can be one or the other but not both. Pooled-worker
designs are cheap but stateful. Microvm/container per call is fresh
but slow. The product space we're filling didn't have a substrate that
checked both boxes.

## Why WebAssembly

WebAssembly's linear memory is a single `mmap`-able byte array per
instance. That makes two things trivial that aren't on a Linux process:

- **Snapshot a Python interpreter mid-execution.** After
  `Py_InitializeFromConfig` plus your common imports
  (`json, re, math, hashlib, sqlite3, ...`), the entire runtime state
  fits in ~10 MB of linear memory. Capture once, restore per call.
- **Reset that snapshot back to its original byte-exact state in one
  syscall.** With the right kernel-side trick, the cost approaches the
  cost of a single page-table operation — no actual data motion.

We don't have to invent a new sandbox primitive. The WebAssembly memory
model already gives us one. The work is wiring CPython through it.

## Path D: build the WASI Python distribution from scratch

We considered three other paths first and rejected them:

| Path | What it is | Why it didn't fit |
|---|---|---|
| A: subprocess Wasmtime CLI per call | Spawn `wasmtime python.wasm script.py` | ~400 ms per call. Bound by Wasmtime startup, not Python |
| B: Pyodide on WASI | Reuse the Pyodide ecosystem | Pyodide targets Emscripten with a JS host. Not WASI-compatible |
| C: A pooled in-process Python | Reuse one Python instance per worker | Defeats per-call freshness — worker carries state |
| **D**: **Cross-compile CPython 3.14 to WASI from scratch** | **What we did** | **Hardest, but the only one with the right substrate** |

The build uses CPython's official `Tools/wasm/wasi` driver with
[wasi-sdk 32](https://github.com/WebAssembly/wasi-sdk). Plus our own
M0.5 stdlib pass to get useful modules built in:

| Module | Status | How |
|---|---|---|
| `zlib` | ✓ | Cross-compile zlib-1.3.1 with wasi-sdk |
| `sqlite3` | ✓ | Cross-compile sqlite-amalgamation-3530000, link static |
| `_hashlib` | ✓ | Cross-compile OpenSSL 3.4 libcrypto only (no-sock / no-stdio for WASI) |
| `_ssl` | — | Skipped: WASI Preview 1 has no `getsockname` / sockets layer |
| `_ctypes` | — | Skipped: WASI has no executable-memory closures |

The result is `python-reactor.wasm`: a 34 MB WebAssembly module that
exposes a small ABI:

```c
int32_t wisp_init(void);                    // initialize Python runtime + pre-imports
int32_t wisp_eval(int32_t ptr, int32_t len); // run a Python source slice
int32_t wisp_alloc(int32_t size);           // host writes source into linear memory here
void    wisp_free(int32_t ptr);
```

Built in WASI Reactor mode — `_initialize` runs once per instance, then
`wisp_init` brings up CPython, then `wisp_eval` runs source. The host
can interleave `wisp_alloc` / `wisp_eval` / linear-memory snapshot
operations between calls.

## The cold-start descent

We ran four spikes, each replacing the bottleneck of the previous one.

### Spike A — pooled in-process, fresh interpreter

```
| Subprocess wasmtime CLI    | ~400 ms |
| Pooled in-process, init    |  39 ms  |  ← Spike A
```

Pre-built engine, pre-compiled module, instantiate fresh, run
`wisp_init` per call. The 39 ms is dominated by `Py_InitializeFromConfig`
plus stdlib imports — those constitute most of CPython's startup work.

This was already 10× faster than the subprocess baseline and good enough
for most workloads. But cheap relative to native ≠ cheap on its own
terms. Could we cut the init cost itself?

### Spike A2 — memory-snapshot restore

The trick: `wisp_init` is deterministic. Run it once, capture the
linear memory, then for every subsequent call instantiate a fresh
instance and restore the snapshot via `Memory::data_mut().copy_from_slice()`
instead of re-running `wisp_init`.

```
| memcpy snapshot/restore    |  1.68 ms  |  ← Spike A2
```

23× speedup over Spike A. Now the per-call cost is ~1 ms of memcpy
(10 MB at native memory bandwidth) plus ~0.6 ms of instantiate +
eval. The memcpy was the dominant cost: bandwidth-bound, doesn't
parallelize.

### Spike A2.1 — mmap COW snapshot

Replace the per-call memcpy with a custom Wasmtime `MemoryCreator` that
mmaps the snapshot file `MAP_PRIVATE`. Then per call, after instantiate,
re-mmap `MAP_PRIVATE | MAP_FIXED` to undo Wasmtime's data-segment init
writes:

```rust
unsafe impl MemoryCreator for CowMemoryCreator {
    fn new_memory(&self, ...) -> Result<Box<dyn LinearMemory>> {
        // 1. Reserve virtual region with PROT_NONE
        // 2. mmap MAP_PRIVATE of snapshot file at offset 0
        //    (reads share kernel page cache; writes COW)
        // 3. mmap MAP_PRIVATE | MAP_ANON for trailing zero pages
    }
}

// per call:
let r = libc::mmap(
    base, snapshot.len(),
    PROT_READ | PROT_WRITE,
    MAP_PRIVATE | MAP_FIXED,
    snap_fd, 0,
);
```

Result:

```
| mmap COW snapshot          |  0.69 ms  |  ← Spike A2.1
```

The per-call snapshot reset dropped from a 1.07 ms memcpy to a 0.030 ms
syscall. The remaining 0.66 ms decomposes:

```
| Instantiate + data init    |   0.45 ms  ← wasmtime overhead
| mmap MAP_FIXED reset       |   0.03 ms  ← was memcpy 1.07 ms
| Grow memory                |   0.002 ms
| Alloc + write code         |   0.01 ms
| wisp_eval                  |   0.20 ms  ← page faults lazy
| Total                      |   0.69 ms
```

The `wisp_eval` cost rose slightly compared to A2 because pages are now
faulted in lazily during execution. But this is overall a ~2.4× win,
and unlike memcpy it doesn't burn memory bandwidth — meaning it should
parallelize better.

## The other primitive: cheap fork from a snapshot

The branching workload tells the same story from a different angle:
spawn K children from a single post-init snapshot, each running a
divergent Python expression.

| K | memcpy backend (br/s) | mmap COW backend (br/s) |
|---|---|---|
|   1 |  367 |  1163 |
|   8 |  697 |  1702 |
|  64 |  926 |  2231 |
| 256 | 1025 | **2363** |
|1024 |  799 | **2394** |

Peak parallel throughput on 8 cores: **2394 branches/sec** with the COW
backend, 2.3× more than memcpy. Per-branch latency at K=64 parallel:
p50 3.42 ms, p99 6.17 ms.

Comparison to native runtimes for a tree-search workload (K=100 ×
depth=100 = 10 000 forks per trajectory):

| Substrate | Per-trajectory branching cost |
|---|---|
| Linux process fork | 50–100 s |
| Firecracker uVM snapshot | 16–83 min |
| Wisp WASM memcpy snapshot | ~10 s |
| **Wisp WASM mmap COW snapshot** | **~4.2 s** |

100–1000× faster than Firecracker. 10–25× faster than raw Linux fork.
And every WASM child is a *truly fresh* sandbox — new linear memory,
no shared state with siblings or parent beyond the snapshot contents.

## Where the architecture wins, and where it doesn't

The wins:

- **Per-call freshness comes for free.** Each `Instance` gets its own
  linear memory; there is no shared-process problem to begin with.
- **Snapshot/restore is cheap because linear memory is just an `mmap`.**
  Native processes can't snapshot at byte granularity without ptrace
  or full uVM serialization.
- **Bytecode-determinism enables COW.** Wasm execution is deterministic
  given the same memory + inputs. We can share read-only pages across
  thousands of children safely.

The honest limits we measured:

1. **8-core scaling is 2.0×, not 8×.** With memcpy out of the way, the
   bottleneck moves to Wasmtime's `instantiate` (~0.45 ms per call) and
   kernel page-fault contention. Future work: pooling-allocator + COW
   together (Wasmtime today doesn't compose them); pre-faulting via
   `MAP_POPULATE`; cross-thread memory-image sharing.
2. **`_ssl` and `_ctypes` are missing.** WASI Preview 1 has no usable
   socket layer (`getsockname` is gated to wasip2+) and no executable-
   memory primitives (libffi closures need writable+executable pages).
   `_ssl` is mostly cosmetic anyway since the sandbox can't open
   sockets; `_ctypes` matters for the long tail of native-extension
   wheels.
3. **No third-party C extensions yet.** `numpy` and friends require a
   build pipeline that takes a setup.py + sdist and produces a WASI
   wasm object. That's the next milestone (M1).
4. **Wasmtime's data-segment init runs on every instantiate.** It
   consumes 0.45 ms of our 0.69 ms total. A "skip data init when host
   memory matches" feature would drop us toward 0.25 ms total.

## Reproduce

Everything in this post is in
[github.com/wisplab/wisp](https://github.com/wisplab/wisp), MIT-licensed.

```bash
git clone https://github.com/wisplab/wisp
cd wisp/runtime/cpython-wasi && ./build.sh        # CPython 3.14 → wasm
./build-sqlite.sh && ./build-openssl.sh           # M0.5 stdlib deps
./wisp_entry/build.sh                              # python-reactor.wasm

cd ../../bench/python-wasi-cow && cargo run --release            # Spike A2.1
cd ../python-wasi-cow-branching && cargo run --release            # Spike B2
```

Each `bench/*/FINDINGS.md` documents the measurement methodology and
hardware. Apple Silicon M-series, 8 cores, Wasmtime 27 in-process.

## What's next

Two tracks:

1. **M1: cross-build pipeline for arbitrary Python packages.** Wraps
   wasi-sdk + CPython headers as a build env so `wisp build numpy`
   produces a WASI-loadable wheel-like artifact. Gateway to numpy →
   pandas → scikit-learn.
2. **Spike B3: COW + pooling allocator.** Today the Wasmtime pooling
   allocator (which would help instantiate cost) is incompatible with
   `with_host_memory` (which we need for COW). Resolving this should
   close the gap toward 8-core linear scaling.

This is open-source platform work, no commercial layer. If you build
on top of it, we'd love to hear what you used it for.

— Wisp
