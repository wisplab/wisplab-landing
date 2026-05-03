# Per-call Python sandboxes in WebAssembly: where Wisp fits

*Draft — 2026-05-03*

---

## TL;DR

Wisp is an open-source Python sandbox runtime that follows the same
integration model as E2B, Modal Sandbox, Daytona, Blaxel, Vercel
Sandbox, and Cloudflare Workers — agent frameworks plug it in as their
"tool execution backend" and ask it to run model-generated Python
safely.

The substrate is different: WASI Python under Wasmtime instead of a
Firecracker microVM, gVisor container, or Docker container. That choice
buys two things the others structurally cannot reach today:

1. **0.78 ms per-call cold start** at the executor (vs 25 ms warm
   resume / 90–150 ms cold for the others).
2. **Per-call fresh sandbox as the default**, not just an option. At a
   sub-millisecond startup cost, "new sandbox per tool call" is
   economically the same as "reuse one sandbox" — so we can default to
   fresh and recover *correctness* properties (no state leakage between
   tool calls, byte-identical replay) for free.

This post explains where Wisp belongs in the existing landscape — it's a
**complement, not a replacement** — and walks through how we got the
numbers.

---

## The agent tool-execution problem

A code agent (Claude Code, Aider, OpenCode, Cursor, OpenInterpreter,
LangChain agents, OpenAI Agents SDK, …) is a loop:

```
LLM ─→ "execute this Python" ─→ tool backend runs it ─→ result back to LLM ─→ ...
```

The agent's job is the loop. The **tool backend's job** is running the
model's generated code:

- Safely (model output is untrusted)
- Quickly (often dozens of tool calls per user turn)
- With the right capabilities (filesystem? shell? network? GPU?)

These are different concerns. The agent framework focuses on routing,
memory, and LLM orchestration; the tool backend focuses on isolation
and speed. Most agent frameworks today either implement the tool
backend ad hoc (subprocess + chmod, or maybe Docker) or call out to a
hosted sandbox service.

Wisp aims at the second slot.

---

## The current landscape (2026-05)

Eight serious players ship "sandbox-as-a-service for AI agents" today.
The April 2026 OpenAI Agents SDK release named seven of them as
built-in integrations (Blaxel, Cloudflare, Daytona, E2B, Modal,
Runloop, Vercel) — that's the canonical adoption layer.

Each made a different bet on substrate, lifecycle, and capability
model. **None of them is wrong.** They're optimized for different
agent shapes.

| Provider | Substrate | Per-call cold start | Lifecycle | Capability model | Sweet spot |
|---|---|---|---|---|---|
| **E2B** | Firecracker microVM + Jupyter inside | ~150 ms | Persistent (5-min idle timeout) | Broad: full Python + pip + shell + network inside VM | Data analysts, long notebook-style sessions |
| **Modal Sandbox** | Custom Rust container + gVisor | sub-second | Per-invocation | Broad: full container | ML workloads, batch jobs |
| **Daytona** | Docker container (+ optional Kata) | ~90 ms warm | Persistent dev environment | Broad: full container, kernel-shared | Dev environments, CI |
| **Blaxel** | Firecracker + perpetual env | ~25 ms warm resume | Persistent w/ aggressive idle shutdown | Broad: full VM | Intermittent agent traffic |
| **Vercel Sandbox** | Firecracker | sub-second | Per-invocation, 5-hour cap | Broad: full VM | One-off code execution at the edge |
| **Cloudflare Workers** | V8 isolate + Pyodide | sub-100 ms | Per-invocation | Narrow: V8-sandboxed JS-first; Python via Pyodide | Edge HTTP workloads |
| **AWS Lambda Sandbox** | Firecracker | 800–1500 ms (Python cold) | Per-invocation | Broad: full container | Cron, infrequent triggers |
| **Wisp** *(this work)* | wasmtime + WASI CPython | **0.78 ms** | Per-call fresh (default) | Capability-bridge (explicit allowlist) | High-frequency tool-call agents, RL rollouts |

A few honest notes:

- **Cold start numbers are not apples-to-apples.** E2B and Blaxel
  measure VM creation over their cloud RPC; Wisp measures the executor
  primitive in-process. End-to-end with HTTP, Wisp is closer to ~3 ms
  cold and the gap to the others narrows. Still substantial — see the
  decomposition section below.
- **"Sweet spot" isn't a knock on the others.** A persistent Jupyter
  inside a microVM is genuinely the right shape for "open a sandbox,
  hand the agent pandas, let it explore for 10 minutes". A per-call
  fresh wasm Instance would feel awkward there.
- **Capability model is a security choice, not a feature gap.** E2B's
  "broad inside the VM" is fine because the VM boundary is strong.
  Wisp's "narrow + capability-bridge" is appropriate when the same
  daemon hosts code from many distrustful tenants.

---

## Where Wisp fits — and where it doesn't

| Use case | Wisp | Use one of the others |
|---|---|---|
| Agent makes 50+ short tool calls per turn (file ops, JSON parse, simple compute) | ✓ default per-call freshness, no state pollution between calls | persistent-VM models pay state-leakage cost |
| Tree-search RL rollout (MCTS, GRPO) | ✓ COW fork at sub-ms per branch (Spike B2 below) | Linux fork or Firecracker snapshot is 100–1000× slower |
| Multi-tenant SaaS hosting untrusted agent code | ✓ capability-bridge gives explicit allowlists per tenant | broad VM access works but auditing is harder |
| Data-analyst agent with `pandas` and a CSV that should persist | E2B/Daytona model is a better fit (state survives across cells) | Wisp can do session mode, but it's bolted on |
| Agent needing arbitrary `pip install` mid-session | E2B (full apt+pip inside VM) | Wisp's WASI Python doesn't have dlopen for native wheels |
| Browser automation (Playwright, etc.) | E2B's chrome-in-sandbox | not yet in scope |

The honest position: **for most agent frameworks today, Wisp is a
plug-in for the tool-execution layer**, sitting alongside (not
replacing) whichever sandbox provider they already use. We are best
when the agent loop is high-frequency and stateless-ish.

---

## The substrate, briefly

WebAssembly's linear memory is a single contiguous byte array per
Instance. That's what makes per-call fresh cheap:

- **Snapshot a Python interpreter mid-execution** = `memcpy` (or
  `mmap MAP_FIXED`) of the byte array
- **Reset to a known-good state** = re-mmap the snapshot file at the
  same address — kernel page-table operation, no actual data motion
- **Spawn N children sharing the same base** = mmap COW (copy-on-write
  on first write)

Native processes structurally can't do this — heap is scattered across
mmap regions, shared libs, stack; you can't dump or restore at byte
granularity without ptrace or full uVM serialization. Firecracker has
a snapshot facility, but it operates at VM granularity and takes
100–500 ms to restore.

The wasm Instance abstraction collapses snapshot/restore into one
syscall. That's where the substrate gap comes from.

---

## The cold-start descent (our spikes, with numbers)

We ran four spikes, each replacing the bottleneck of the previous one.
All measurements are on Apple Silicon M-series, 8 cores, Wasmtime
crate 27, in-process.

### Spike A — pooled in-process, fresh interpreter

```
| Subprocess wasmtime CLI    | ~400 ms |
| Pooled in-process, init    |  39 ms  |  ← Spike A
```

Pre-built engine, pre-compiled module, instantiate fresh, run
`wisp_init` per call. The 39 ms is dominated by
`Py_InitializeFromConfig` plus stdlib imports — most of CPython's
startup work. Already 10× faster than the subprocess baseline.

### Spike A2 — memory-snapshot restore

`wisp_init` is deterministic. Run it once, capture the linear memory,
restore the snapshot via `Memory::data_mut().copy_from_slice()`
instead of re-running init.

```
| memcpy snapshot/restore    |  1.68 ms  |  ← Spike A2
```

23× speedup. The memcpy was the dominant cost — bandwidth-bound,
doesn't parallelize.

### Spike A2.1 — mmap COW snapshot

Replace the per-call memcpy with a custom Wasmtime `MemoryCreator` that
mmaps the snapshot file `MAP_PRIVATE`. Per call, after instantiate,
re-mmap `MAP_PRIVATE | MAP_FIXED` to undo Wasmtime's data-segment init
writes:

```rust
unsafe impl MemoryCreator for CowMemoryCreator {
    fn new_memory(&self, ...) -> Result<Box<dyn LinearMemory>> {
        // 1. Reserve virtual region with PROT_NONE
        // 2. mmap MAP_PRIVATE of snapshot file at offset 0
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
| mmap COW snapshot          |  0.78 ms  |  ← Spike A2.1
```

The per-call snapshot reset dropped from a 1.07 ms memcpy to a 0.030 ms
syscall. The remaining 0.75 ms decomposes:

```
| Instantiate + data init    |   0.45 ms  ← wasmtime overhead
| mmap MAP_FIXED reset       |   0.03 ms  ← was memcpy 1.07 ms
| Grow memory                |   0.002 ms
| Alloc + write code         |   0.01 ms
| wisp_eval                  |   0.20 ms  ← page faults lazy
| Total                      |   0.78 ms
```

The `wisp_eval` cost rose slightly compared to A2 because pages are
faulted in lazily during execution. Net win: ~2.4× over memcpy, and
unlike memcpy this approach doesn't burn memory bandwidth — meaning it
parallelizes better.

---

## The other primitive: cheap fork from a snapshot

Spawn K children from a single post-init snapshot, each running a
divergent Python expression.

| K | memcpy backend (br/s) | mmap COW backend (br/s) |
|---|---|---|
| 1 | 367 | 1163 |
| 8 | 697 | 1702 |
| 64 | 926 | 2231 |
| 256 | 1025 | **2363** |
| 1024 | 799 | **2394** |

Peak parallel throughput on 8 cores: **2394 branches/sec** with the
COW backend, 2.3× more than memcpy. Per-branch latency at K=64
parallel: p50 3.42 ms, p99 6.17 ms.

Comparison to native runtimes for a tree-search workload (K=100 ×
depth=100 = 10 000 forks per trajectory):

| Substrate | Per-trajectory branching cost |
|---|---|
| Linux process fork (Python interpreter) | 50–100 s |
| Firecracker uVM snapshot | 16–83 min |
| Wisp WASM memcpy snapshot | ~10 s |
| **Wisp WASM mmap COW snapshot** | **~4.2 s** |

Useful for tree-search RL: every WASM child gets a *truly fresh*
sandbox — new linear memory, no shared state with siblings or parent
beyond the snapshot contents.

---

## End-to-end through the daemon

The 0.78 ms number is the executor primitive. End-to-end through the
HTTP daemon (`wisp-runtime`):

```
$ for i in {1..10}; do
    curl -s -X POST http://127.0.0.1:9000/v1/eval \
      -H "Content-Type: application/json" \
      -d '{"code":"print(2+2)"}' | jq .elapsed_us
  done
14225
1511
1356
1203
1189
1130
1284
1189
1120
1255
```

**Steady state ~1.1–1.3 ms** including HTTP framing, JSON
serialization, and the channel hop from tokio to a worker thread. The
first call is warm-up of the JIT cache for the first Instance.

This is the cold start an agent framework will measure when it
integrates the Wisp client SDK.

---

## What WebAssembly does NOT solve

Honest disclaimers, since these come up:

- **Wasm Python is not faster Python.** CPython compiled to WASM runs
  10–30% slower than native CPython. We sell *cheap isolation*, not
  faster execution.
- **No `pip install` of native wheels.** WASI doesn't have `dlopen`,
  so packages with C extensions need to be built into the runtime
  ahead of time. (We have a build pipeline for that — see `scripts/`
  in the repo.)
- **No threads, no `multiprocessing`, no `subprocess`.** WASI Preview
  1 doesn't expose any of these. Threading-using libraries fail at
  import. This is generally fine for a sandbox (you don't *want* it
  spawning threads), but it does block some libraries.
- **No real sockets in-sandbox.** WASI Preview 1 has no `getsockname`.
  Outbound HTTPS goes through a host capability bridge instead — same
  pattern Cloudflare Workers, Deno, and other capability runtimes use.
- **`_ssl` and `_ctypes` modules are missing.** Architectural, not
  fixable from our side — would need WASI Preview 2 for `_ssl` (and
  `_ctypes` is blocked at the WebAssembly layer regardless).

For most agent tool calls (parse this JSON, transform that data,
validate this schema, do this hash), none of these matter. For some
agent workloads (HTTP scraping, browser control, full notebook
exploration), one of the persistent-VM substrates is the right choice.

---

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

# And the daemon + SDK:
cd ../.. && cargo run --release -p wisp-runtime &
pip install -e sdks/python
python -c "import wisp; print(wisp.Client().eval('print(2+2)').stdout)"
```

Each `bench/*/FINDINGS.md` documents the methodology.

---

## What's next

1. **Daemon-side capabilities** — `shell`, `file_read`, `file_write`,
   `web_fetch`, `grep`. Configured via allowlist. Without these the
   sandbox is too narrow for real tool-call workloads.
2. **Sandbox-side `wisp` Python module** — friendly wrapper around the
   capability bridge, so user code does `wisp.shell("ls")` instead of
   raw `_wisp.call_host(...)`.
3. **Reference integration with one open-source agent framework** —
   probably smolagents or OpenCode. Not asking them to merge a PR;
   just a working example so anyone can copy the pattern.
4. **M1 build pipeline → numpy** — let the sandbox actually run real
   data science.
5. **Session API** — for users who want E2B-like persistent semantics
   when that's the right fit.

If you operate an agent framework or platform and the pattern above
fits something you're hitting, we'd love to hear from you.

— Wisp
