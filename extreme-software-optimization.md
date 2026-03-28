---
description: "Profile-driven performance optimization with behavior proofs. Use when: optimize, slow, bottleneck, hotspot, profile, p95, latency, throughput, or algorithmic improvements."
allowed-tools:
  - Bash
  - Read
  - Grep
  - Glob
  - Edit
  - Write
  - Agent
---

# /extreme-software-optimization — Extreme Software Optimization

**Iron Law: NEVER COMMIT AN OPTIMIZATION THAT CANNOT PROVE ISOMORPHIC BEHAVIOR UNDER ALL INPUTS.**

An optimization that silently changes output under edge cases is a bug, not a speedup. The three-lock gate — benchmark + oracle + tests — must all pass before any change lands.

**argument-hint:** `[target — module path, function name, or 'all']` (default: profile entire project)

---

## Phase 1: Load Context

1. Read `CLAUDE.md` and `README.md` for project-specific profiling tools, benchmark commands, and test runners.
2. Identify the language(s) and runtimes in scope (Rust, Go, Python, TypeScript, etc.).
3. Check if golden artifacts exist at `.golden/` or `testdata/` — if not, invoke `/golden-artifacts` first to establish the reference corpus before any mutations.
4. Note existing benchmark infrastructure (Criterion, `testing.B`, `pytest-benchmark`, Vitest bench, hyperfine, etc.).

---

## Phase 2: Profile — Measure First, Optimize Second

**Never skip this phase. Optimizing unmeasured code is guesswork.**

### 2a. Establish Baseline Benchmarks

Run the existing benchmark suite and capture output:

```bash
# Rust
cargo bench --bench '*' 2>&1 | tee /tmp/bench-before.txt

# Go
go test ./... -bench=. -benchmem -count=3 2>&1 | tee /tmp/bench-before.txt

# Python
python -m pytest --benchmark-only --benchmark-json=/tmp/bench-before.json

# Node/TypeScript
npx vitest bench --reporter=json > /tmp/bench-before.json
```

If no benchmarks exist: write micro-benchmarks for the top 5 candidate functions before proceeding.

### 2b. CPU Profile

Identify hotspots with sampling profilers:

```bash
# Rust: flamegraph
cargo flamegraph --bin <binary> -- <args>

# Go: pprof
go tool pprof -http=:6060 cpu.prof

# Python: py-spy
py-spy record -o profile.svg -- python script.py

# Node: --prof
node --prof app.js && node --prof-process isolate-*.log
```

### 2c. Memory Profile

Look for allocation-heavy paths that create GC pressure:

```bash
# Rust: heaptrack or DHAT
cargo run --features dhat-heap

# Go: pprof mem
go test -memprofile mem.prof && go tool pprof mem.prof

# Python: memray
python -m memray run script.py && python -m memray flamegraph memray-*.bin
```

### 2d. Rank Hotspots

Produce a ranked table of hotspots, sorted by cumulative CPU time:

| Rank | Function | File | % CPU | Alloc/call | p95 Latency |
|------|----------|------|-------|------------|-------------|
| 1 | `foo()` | `src/foo.rs:42` | 38% | 3.2 KB | 340ms |
| 2 | `bar()` | `pkg/bar.go:17` | 21% | 0 | 12ms |

**Only optimize rank 1–3 per pass.** The Pareto distribution is real: the top hotspot is always worth more than the next five combined.

---

## Phase 3: Optimization Candidates — Apply Every Applicable Trick

For each hotspot, generate a list of candidates. Apply these lenses in order (highest expected gain first):

### Tier 1: Algorithmic Complexity (10x–1000x)

- Replace O(n²) nested loops with hash maps / sets → O(n)
- Replace linear search with binary search on sorted data → O(log n)
- Replace repeated recomputation with memoization / DP table
- Replace full traversal with early-exit guards (filter before map)
- Use monotonic stack/queue for sliding-window problems
- Recognize and apply standard IOI patterns: prefix sums, segment trees, topological sort, union-find

### Tier 2: Allocation Elimination (2x–20x)

- Reuse buffers across calls instead of allocating per-call
- Swap `Vec<u8>` / `[]byte` / `list` for stack-allocated arrays at small sizes
- Use object pools for frequently created/destroyed structs
- Replace `format!()` / `sprintf` / `str.format()` in hot paths with direct writes
- Move semantics: consume inputs instead of cloning (Rust `into_*` methods, Go value receivers)
- Zero-copy slices into existing buffers instead of new allocations

### Tier 3: Data Layout & Cache Locality (2x–10x)

- Struct-of-arrays vs array-of-structs for tight inner loops
- Pad hot fields to cache-line boundaries; move cold fields to separate structs
- Replace pointer-chasing linked structures with flat arrays + indices
- Sort data to maximize cache prefetch hits

### Tier 4: Compute Reduction (1.5x–5x)

- Strength reduction: replace expensive ops with cheaper equivalents (`/2` → `>>1`, `%power_of_2` → `& mask`)
- Hoist loop-invariant computations out of inner loops
- Precompute lookup tables for repeated expensive calculations
- Lazy evaluation: defer expensive operations until results are actually needed
- Short-circuit boolean evaluation order (cheapest checks first)

### Tier 5: Parallelism & I/O (variable)

- Parallelize independent work with `rayon` / `goroutines` / `asyncio` / `Promise.all`
- Batch I/O operations; replace N sequential reads with one vectored read
- Pipeline: overlap compute with I/O using channels/streams
- Use `mmap` for large read-only files instead of buffered reads

### Tier 6: Compiler / Runtime Hints (10%–3x)

- Add `#[inline]` to hot functions called across crate boundaries (Rust)
- Use `__attribute__((hot))` / `likely()` / `unlikely()` (C/C++)
- Annotate with `@functools.lru_cache` or `@cache` for pure functions (Python)
- Use `Uint8Array` / `ArrayBuffer` instead of JS objects in hot paths
- Enable profile-guided optimization (PGO) for release builds

---

## Phase 4: The Autoresearch Loop

**Run experiments. Reject more than you keep. The benchmark is the judge, not intuition.**

For each candidate optimization:

1. **Create a branch or working copy** — never mutate the baseline in-place.
2. **Apply the change** — one optimization at a time, not batched.
3. **Run the three-lock gate:**
   - **Lock 1 — Benchmark:** Run the benchmark suite. Must show measurable improvement (≥5% for micro, ≥2% for end-to-end). If flat or regressed: **auto-revert**.
   - **Lock 2 — Oracle:** Compare outputs against golden artifacts (`.golden/` or `testdata/`). Any byte-level divergence = **auto-revert**.
   - **Lock 3 — Tests:** Run the full test suite. Any new failures = **auto-revert**.
4. **Only if all three locks pass:** keep the change and continue to the next candidate.
5. **Log every experiment** — keep a running table even for reverted changes.

### Experiment Log (maintain this throughout)

| # | Change | Lock 1 (bench) | Lock 2 (oracle) | Lock 3 (tests) | Decision |
|---|--------|---------------|-----------------|-----------------|----------|
| 1 | Buffer reuse in `encode()` | +9.4x | PASS | PASS | **KEPT** |
| 2 | SIMD path in `hash()` | +0.3x (noise) | PASS | PASS | reverted |
| 3 | DP table in `search()` | +3.1x | FAIL (edge case) | PASS | reverted |

---

## Phase 5: Isomorphism Proof

Before reporting results, produce a formal isomorphism proof for every **kept** change:

### Required Evidence

1. **Ordering/byte-equivalence:** Encode/decode round-trips produce identical output. State this explicitly.
2. **Hash stability:** If hashing was changed, prove the hash function is semantically equivalent (streaming vs batched BLAKE3, etc.).
3. **Golden output parity:** State the exact count of golden tests passed.
4. **Pre-existing failures:** Run tests on the original `main` branch and confirm any failures existed before your changes (not caused by them).
5. **Edge-case coverage:** Identify edge cases that benchmarks don't cover (empty input, max-size input, adversarial input). Run these specifically.
6. **Fuzz confirmation** (if available): Run fuzz targets for 60 seconds and report zero crashes.

### Isomorphism Proof Template

```
Isomorphism Proof
─────────────────
- Output equivalence:    Yes — identical byte sequences for all tested inputs
- Hash stability:        Yes — streaming components produce same hash as batched
- Golden output parity:  1,380 / 1,380 tests pass across all modified modules
- Pre-existing failures: 8 failures in fcp-raptorq confirmed pre-existing on main
- Zero regressions:      All modified crates pass linting with zero new warnings
- Fuzz:                  60s fuzz run — 0 crashes, 0 output divergences
```

---

## Phase 6: Report

### Changes Made

| File | Function | Optimization | Mechanism |
|------|----------|--------------|-----------|
| `src/encode.rs` | `SymbolRecord::encode_to()` | Zero-copy buffer writes | Eliminates per-symbol Vec allocation |
| `pkg/hash.go` | `SchemaId.Hash()` | Streaming hash | Feeds components directly; avoids format+concat |

### Benchmark Results

| Benchmark | Before | After | Speedup |
|-----------|--------|-------|---------|
| frame encode 10x1024 | 4.5µs | 478ns | **9.4x** |
| frame encode 64x256 | 22µs | 2.1µs | **10.5x** |
| schema hash | 481ns | 134ns | **3.6x** |
| serialize_large | 35µs | 16µs | **2.2x** |

### Isomorphism Proof

_(Paste proof from Phase 5)_

### Experiment Summary

- Experiments run: N
- Kept: K
- Auto-reverted: N-K (benchmark failed: X, oracle failed: Y, tests failed: Z)

### Status

End with one of:

- **STATUS: OPTIMIZED** — Changes made, all locks passed, isomorphism proven.
- **STATUS: BASELINE_ONLY** — No wins found this pass; profiling data captured for next pass.
- **STATUS: BLOCKED** — Golden artifacts missing or test suite too weak to verify isomorphism safely. Run `/golden-artifacts` first.

---

## Repeat Passes

This skill is designed to be run repeatedly. Each pass finds the next layer of wins:

- **Pass 1:** Obvious algorithmic improvements (O(n²) → O(n), allocation elimination)
- **Pass 2:** Data layout, cache locality, compiler hints
- **Pass 3:** Parallelism, batching, pipeline overlaps
- **Pass N:** Increasingly subtle micro-optimizations

The three-lock gate prevents any pass from introducing silent regressions. Run until `STATUS: BASELINE_ONLY` — no further wins detected.
