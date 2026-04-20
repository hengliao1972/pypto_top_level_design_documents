# Edit: docs/pypto-runtime-design/modules/profiling.md

- **Driven by proposal(s):** A1-P7, A1-P10, A2-P9, A8-P3, A8-P5, A8-P6, A8-P8, A8-P9, A8-P10, A8-P12
- **ADR link(s):** —
- **Timestamp (UTC):** 2026-04-19
- **Edit kind:** additive (consolidated `## 13. Review Amendments (R3)` section appended before Document status footer)

## Location

- Path: `docs/pypto-runtime-design/modules/profiling.md`
- Section / heading: new `## 13. Review Amendments (R3)`; references §2.1, §2.2, §2.3, §2.5, §3.1, §3.4, §4.1, §4.2, §6, §7, §8, §9.2
- Line range before edit: 465–466 (footer only)

## Before

```465:466:docs/pypto-runtime-design/modules/profiling.md
**Document status:** Draft — ready for review.
```

## After

A new `## 13. Review Amendments (R3)` heading is inserted before the footer, with one `[UPDATED: <id>: ...]` quoted callout per assigned proposal. Per-proposal edits follow.

## Per-proposal edits

### A1-P7 — Per-thread local seq + offline merge

- **Before:** §4.2 `SequenceAllocator::next` was a global atomic; every event contended it.
- **After (callout in §13):**

> [UPDATED: A1-P7: per-thread local_seq + offline merge] Each `TlsRingBuffer` owns `uint64_t local_seq`; global atomic removed. Sink merges streams by `(timestamp_ns, thread_id, local_seq)`; `thread_id` is the deterministic tiebreak.

- **Rationale:** X3 / P1 — removes hot-path atomic.

### A1-P10 — Workload-level profiling-overhead CI gate

- **Before:** No CI gate for profiling overhead.
- **After (callout in §13):**

> [UPDATED: A1-P10: profiling overhead CI gate] `bench_profiling_overhead.cpp` drives 10 s × 1 M task submissions at L0/L1/L2 on sim; CI asserts `L1 ≤ 1.01× L0` and `L2 ≤ 1.05× L0`. Regression blocks merge.

- **Rationale:** P1 — enforces the overhead SLO.

### A2-P9 — Versioned trace schema header

- **Before:** Trace files had no schema-version header.
- **After (callout in §13):**

> [UPDATED: A2-P9: one-off trace-file header] `{magic, trace_schema_version: uint16_t, platform, mode}`. REPLAY engine validates on init; mismatched major → `ProtocolVersionMismatch`. Header-only (not per-event).

- **Rationale:** E6.

### A8-P3 — Stats structs + `LatencyHistogram` pinned to CONTROL

- **Before:** Histogram placement unspecified.
- **After (callout in §13):**

> [UPDATED: A8-P3: LatencyHistogram pinned to CONTROL] Pre-allocated log-scale buckets; branchless insert (< 5 ns); seqlock snapshot. Bucket array pinned to CONTROL (cache-line-resident).

- **Rationale:** O3 / O5.

### A8-P5 — External alert-rules file + `AlertRule.logical_system_id`

- **Before:** Alert rules were inline; no tenant scoping.
- **After (callout in §13):**

> [UPDATED: A8-P5: AlertRule schema incl. logical_system_id] `{metric, op, threshold, window_ms, severity, logical_system_id}`. Prom/OTEL sinks opt-in deviation; rules externalized to parameterize A5-P8 policies.

- **Rationale:** O5 / E3 / X8.

### A8-P6 — Trace time-alignment contract (`IClockSync`)

- **Before:** Skew bound and merge rule unspecified.
- **After (callout in §13):**

> [UPDATED: A8-P6: time-alignment + tie-breaker] `skew_max_ns ≤ 100 µs`; merge by `(sequence, correlation_id, happens_before)`; tie-breaker `min(node_id)` in youngest all-online epoch; correlation-id chain dominates timestamp ties.

- **Rationale:** O1.

### A8-P8 — AICore in-core trace ring + DMA uploader

- **Before:** L2 opt-in emit protocol was ad-hoc.
- **After (callout in §13):**

> [UPDATED: A8-P8: AICore event ring + DMA uploader] Fixed-size ring; Chip-level DMA upload at `sink_flush_interval_ms`; capability bit `AICORE_TRACE_RING_SUPPORTED`; default L1 compile-strips L2 emits; L2 write < 50 ns amortized.

- **Rationale:** O3 — bounded cost for in-core traces.

### A8-P9 — Profiling drop/degraded as first-class alerts

- **Before:** Drop/degrade counters not exposed.
- **After (callout in §13):**

> [UPDATED: A8-P9: dropped + degraded counters + alerts] `TraceEvents_dropped`, `TraceSink_degraded_ms`, `LogEvents_dropped`. Alert rules: drop > 0 over 1 min → Warning; degraded > 5 s → Warning.

- **Rationale:** O3 / O5.

### A8-P10 — Structured KV logging

- **Before:** Logger emitted plain strings only.
- **After (callout in §13):**

> [UPDATED: A8-P10: Logger::log_kv] `log_kv(Severity, category, {kv...}, correlation_id)`; string-message shim retained; JSON-line sink default.

- **Rationale:** O2.

### A8-P12 — Stable PhaseIds for Submission lifecycle

- **Before:** PhaseIds ad-hoc tags at callsites.
- **After (callout in §13):**

> [UPDATED: A8-P12: canonical PhaseId constants] `SUBMISSION_ADMIT, SUBMISSION_RETIRE, DATA_DEP_SCAN, BARRIER_ATTACH, NONE_ATTACH, WORKSPACE_ALLOC, ADMISSION_QUEUE_DRAIN, WORKER_DISPATCH, WORKER_COMPLETE`. Per-phase durations sum to admission latency within 1%.

- **Rationale:** O1.

## Verification steps

1. `rg -n "\[UPDATED: " docs/pypto-runtime-design/modules/profiling.md` prints 10 entries in §13.
2. `rg -n "bench_profiling_overhead|L1 ≤ 1.01" docs/pypto-runtime-design/modules/profiling.md` surfaces A1-P10.
3. `rg -n "AICORE_TRACE_RING_SUPPORTED" docs/pypto-runtime-design/modules/profiling.md` surfaces A8-P8.
4. Cross-view: `scheduler.md §13` (A8-P12 PhaseIds) and `core.md §13` (A8-P3 histogram) agree.
