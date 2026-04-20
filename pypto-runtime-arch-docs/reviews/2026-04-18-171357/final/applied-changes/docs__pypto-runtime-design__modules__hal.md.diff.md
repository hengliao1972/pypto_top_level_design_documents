# Edit: docs/pypto-runtime-design/modules/hal.md

- **Driven by proposal(s):** A4-P1, A1-P3, A1-P13, A5-P12, A7-P3, A8-P1, A8-P6, A8-P7, A8-P11
- **ADR link(s):** ADR-011 (SimulationMode casing); ADR-015 (I-DIST-1 / header lint)
- **Timestamp (UTC):** 2026-04-19
- **Edit kind:** additive (consolidated `## 13. Review Amendments (R3)` section appended before Document status footer)

## Location

- Path: `docs/pypto-runtime-design/modules/hal.md`
- Section / heading: new `## 13. Review Amendments (R3)`; references §2.4, §2.6, §2.7, §2.8 (new), §7, §8, §9
- Line range before edit: 452–453 (footer only)

## Before

(Sole previous line at tail was the document status footer; no §13 existed.)

```452:453:docs/pypto-runtime-design/modules/hal.md
**Document status:** Draft — ready for review.
```

## After

A new `## 13. Review Amendments (R3)` heading is inserted before the footer, containing one `[UPDATED: <id>: ...]` quoted callout per assigned proposal. See the module file for the verbatim text; per-proposal edits below.

## Per-proposal edits

### A4-P1 — Canonicalize HAL enum member casing

- **Before:** `Variant { Onboard, Sim }` and `SimulationMode { Performance, Functional, Replay }` (§2.7 lines 163–164; §5 config table lines 394–395).
- **After (callout in §13):**

> [UPDATED: A4-P1: canonical HAL enum casing] Rewrite `Variant { Onboard, Sim }` → `{ ONBOARD, SIM }`; `SimulationMode { Performance, Functional, Replay }` → `{ PERFORMANCE, FUNCTIONAL, REPLAY }` to match every other view and ADR-011. Config column examples are updated accordingly; rename is contained in `modules/hal.md` + ADR-011.

- **Rationale:** D7/V5 consistency with ADR-011-R2 and sibling view docs.

### A1-P3 — Function Cache LRU + capacity + HEARTBEAT presence bloom

- **Before:** (new content — no before)
- **After (callout in §13):**

> [UPDATED: A1-P3: Function Cache LRU bound + HEARTBEAT presence bloom] Function Cache bounded by `function_cache_bytes` (default 64 MiB) with LRU; evictions recorded in `RuntimeStats`; peer cache presence published in `HeartbeatPayload.function_bloom[4]` (`uint64_t`); coordinator Bloom-checks before deciding to inline a binary in `REMOTE_SUBMIT`.

- **Rationale:** P2 — caps HAL-resident cache and feeds A1-P6 REMOTE_BINARY_PUSH gating.

### A1-P13 — Bound `args_blob` copy cost (1 KiB ring-slot fast path)

- **Before:** `KernelDispatch` contract silently assumed bounded args size.
- **After (callout in §13):**

> [UPDATED: A1-P13: 1 KiB ring-slot fast path] `args_size ≤ 1 KiB` fast path copies through a pre-registered ring slot (Host→Chip DMA ≥ 4 GB/s → ≤ 250 ns). For `args_size > 1 KiB`, scheduler stages args in a data-plane `BufferRef` and passes a handle; §4.8.1 budget extends by `0.5 μs/KiB` above threshold. Staging failure → `ErrorCode::ResourceExhausted` (A5-P12).

- **Rationale:** P6/X9 — puts a concrete latency ceiling on the HAL dispatch cost model.

### A5-P12 — Name `ErrorCode::ResourceExhausted` for BufferRef staging alloc

- **Before:** Slow-path failure mode was unnamed.
- **After (callout in §13):**

> [UPDATED: A5-P12: ErrorCode::ResourceExhausted on staging alloc] Staging `BufferRef` allocation failure on A1-P13's `>1 KiB` slow path surfaces `ErrorCode::ResourceExhausted`. Mirrored by one row in `scheduler.md §5`.

- **Rationale:** G3 — closes R2 Con 1 residual.

### A7-P3 — Invert `core/` ↔ `hal/` for handle types

- **Before:** `DeviceAddress`, `NodeId` typedefs lived in `hal/` headers, forcing `core::TaskHandle` to transitively include HAL.
- **After (callout in §13):**

> [UPDATED: A7-P3: handle-type inversion] Move `using DeviceAddress = std::uint64_t;` and `using NodeId = std::uint64_t;` from `hal/` to `core/types.h`. HAL re-exports from its own `types.h` (which `#include <core/types.h>`). `core::TaskHandle` compiles without `hal/`.

- **Rationale:** D2 — removes upstream dependency inversion.

### A8-P1 — Injectable `IClock`

- **Before:** Clock usage inlined `clock_gettime` / `CLOCK_MONOTONIC_RAW` call sites scattered across modules.
- **After (callout in §13):**

> [UPDATED: A8-P1: injectable IClock seam] New §2.8 `IClock { now_ns(), monotonic_raw_ns(), steady_tick() }` with `SystemClock`, `FakeClock`, `PerformanceSimClock` realizations. Threaded via `ProfilingConfig`, `worker_timeout_ms`, heartbeat, Timeouts. Release bin identical via link-time specialization.

- **Rationale:** X5 — makes time a test seam without release cost.

### A8-P6 — Distributed trace time-alignment contract (`IClockSync`)

- **Before:** No declared skew bound or merge rule.
- **After (callout in §13):**

> [UPDATED: A8-P6: IClockSync::offset_ns() + merge rule] PTP/NTP dependency declared; `skew_max_ns ≤ 100 µs`. Merge: primary sort by `(sequence, correlation_id, happens_before)` with skew-windowed reorder; tie-breaker `min(node_id)` in the youngest all-online epoch. Correlation-id chain dominates timestamp ties.

- **Rationale:** O1 — defines deterministic cross-node trace ordering.

### A8-P7 — `IFaultInjector` sim-only seam

- **Before:** Chaos / fault-injection was ad-hoc in sim.
- **After (callout in §13):**

> [UPDATED: A8-P7: IFaultInjector seam] Add `schedule_fault(FaultSpec)` compiled only under sim. Covers DMA-timeout, register-read-fault, RDMA-loss, heartbeat-miss, AICore-hang, slot-pool-exhausted. Hooks into `hal/sim`, `transport/`, `distributed/` dedup. Drives A5-P5 chaos matrix. Never linked in onboard release.

- **Rationale:** X5 / DfT — unified sim-only injection surface.

### A8-P11 — HAL contract test suite (sim + onboard) + header-independence lint

- **Before:** Contract tests ran in sim only; no IWYU-CI rule enforcing I-DIST-1.
- **After (callout in §13):**

> [UPDATED: A8-P11: HAL contract tests + IWYU-CI for I-DIST-1] Single `.cpp` suite runs against both `a2a3sim` and `a2a3` onboard CI. IWYU-CI rule enforces `distributed/` headers non-includable from `transport/` (I-DIST-1 per A2-P6 / A7-P4) and verifies HAL handle types sourced from `core/types.h` per A7-P3. Required CI check.

- **Rationale:** X5 / DfT + D2 — structural lint locks the dependency inversion in place.

## Verification steps

1. `rg -n "\[UPDATED: " docs/pypto-runtime-design/modules/hal.md` prints one line per proposal id in §13.
2. `rg -n "SimulationMode \{" docs/pypto-runtime-design/modules/hal.md` surfaces only the uppercase form inside the §13 callout; legacy §2.7 wording is superseded by the marker.
3. `rg -n "## 13\. Review Amendments \(R3\)" docs/pypto-runtime-design/modules/hal.md` returns exactly one hit.
4. Cross-view consistency: `scheduler.md §13` (A5-P12 row) and `distributed.md §13` (A8-P7 reference) corroborate.
