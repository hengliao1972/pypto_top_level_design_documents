# Edit: docs/pypto-runtime-design/modules/scheduler.md

- **Driven by proposal(s):** A1-P1, A1-P2, A1-P8, A1-P14, A5-P6, A5-P8, A5-P9, A5-P12, A6-P13, A8-P2, A8-P3, A8-P12, A10-P1 (+ cross-ref to A5-P11 in `distributed.md §3.5`)
- **ADR link(s):** ADR-018 (unified peer-health FSM); ADR-019 (admission shard default)
- **Timestamp (UTC):** 2026-04-19
- **Edit kind:** additive (consolidated `## 13. Review Amendments (R3)` section appended before Document status footer)

## Location

- Path: `docs/pypto-runtime-design/modules/scheduler.md`
- Section / heading: new `## 13. Review Amendments (R3)`; references §2.2, §2.4, §2.5, §2.6, §4.1, §4.2, §4.3, §5.1, §5.2, §5.3, §6, §8
- Line range before edit: 563–564 (footer only)

## Before

```563:564:docs/pypto-runtime-design/modules/scheduler.md
**Document status:** Draft — ready for review.
```

## After

A new `## 13. Review Amendments (R3)` heading is inserted before the footer, with one `[UPDATED: <id>: ...]` quoted callout per assigned proposal plus one cross-reference to `distributed.md §3.5`. Per-proposal edits follow.

## Per-proposal edits

### A1-P1 — Pre-size `producer_index`; forbid rehash

- **Before:** §4.2 listed `producer_index` without capacity contract.
- **After (callout in §13):**

> [UPDATED: A1-P1: pre-size producer_index; no rehash] Init capacity `max_outstanding_submissions × expected_outputs_per_submission × 2`, CONTROL region, `isolate_cache_line`. Overflow uses bounded open-addressed probing (L<0.5, probe ≤2); probe exhaustion → `ErrorCode::ResourceExhausted` and back-pressure via `IResourceAllocationPolicy.should_admit`. `producer_index_capacity` exposed in §8.

- **Rationale:** X2 / X4 / P5 — removes hot-path allocation.

### A1-P2 — Bound DATA-mode lock hold; shard count as LevelParam

- **Before:** §6 concurrency block lacked time bounds / shard default.
- **After (callout in §13):**

> [UPDATED: A1-P2: DATA-mode lock hold bounds] Shared hold ≤ `B×A` probes (≤500 ns); exclusive hold ≤ `B×A` + `|S.OUT∪S.INOUT|`; default `producer_index_shards=max(1, scheduler_thread_count)`; p99 reader ≤ 3 μs / writer ≤ 10 μs per shard.

- **Rationale:** X3 / P5 — pins concurrency SLOs.

### A1-P8 — Pre-size Outstanding/Ready; CONTROL placement

- **Before:** §4.1 / §4.3 did not bind capacities or regions.
- **After (callout in §13):**

> [UPDATED: A1-P8: pre-size Outstanding/Ready] ReadyQueue as intrusive min-heap over slot indices, CONTROL region, isolate_cache_line; Admission queue bounded SPSC of `SubmissionDescriptor*`, default 64, overflow→`WouldBlock`; OutstandingWindow fixed array indexed `submission_id % max_outstanding_submissions`. Sub-codes `TaskSlotExhausted` / `ReadyQueueExhausted` routed via A3-P7.

- **Rationale:** X2 / X4 — capacity-bounded, no hot-path allocations.

### A1-P14 — `producer_index` CONTROL placement + shard default

- **Before:** Placement/shard default implicit.
- **After (callout in §13):**

> [UPDATED: A1-P14: CONTROL region + shard default] `producer_index` in CONTROL_AREA (Chip/Device) or CONTROL_HEAP (Host); default shards `max(1, scheduler_thread_count)`; rehash disabled.

- **Rationale:** X4 — admission reads fit in CONTROL.

### A5-P6 — Scheduler-thread watchdog / deadman timer (normative elision)

- **Before:** §2.5 event-loop lacked stall detection.
- **After (callout in §13):**

> [UPDATED: A5-P6: watchdog + write-elision] Per-cycle deadman timestamp; monitor at `scheduler_watchdog_ms` (default 250 ms) raises `SchedulerStalled`. Store elided into per-cycle TSC capture; may skip ≤ K=16 cycles on burst (≤1.6 ms ≪ 250 ms budget).

- **Rationale:** R5 / O5 — detectable liveness without hot-path cost.

### A5-P8 — Degradation specs + alert rules

- **Before:** Admission pressure and partial-group responses were informal.
- **After (callout in §13):**

> [UPDATED: A5-P8: admission-pressure + partial-group policies] `admission_pressure_policy ∈ {REJECT, COALESCE, DEFER}` with `max_deferred`; `partial_group_policy ∈ {FAIL_SUBMISSION, SHRINK_AND_RETRY, DEGRADED_CONTINUE}`; externalized rules file; each value tied to an A8-P5 AlertRule.

- **Rationale:** R4 — back-pressure is a first-class decision.

### A5-P9 — Worker `UNAVAILABLE + {Permanent, Quarantine}` variants

- **Before:** A separate `QUARANTINED` state was tentatively proposed.
- **After (callout in §13):**

> [UPDATED: A5-P9: UNAVAILABLE + Quarantine variants] `RECOVERING → UNAVAILABLE(Quarantine(worker_quarantine_ms))`; probe success → `IDLE`; probe failure / `max_quarantine_cycles` → `UNAVAILABLE(Permanent)`. Config `worker_quarantine_ms`, `worker_probe_retries`.

- **Rationale:** R3 / R4 DRY — avoids a duplicate terminal state.

### A5-P12 — `ErrorCode::ResourceExhausted` row

- **Before:** Slow-path staging alloc failure unnamed here.
- **After (callout in §13):**

> [UPDATED: A5-P12: ResourceExhausted row] Staging `BufferRef` alloc failure on the `>1 KiB` slow path surfaces `ErrorCode::ResourceExhausted`. Mirrored in `hal.md §13`.

- **Rationale:** G3 — closes R2 Con 1 residual.

### A6-P13 — Per-tenant submit rate-limit

- **Before:** Admission had no tenant-quota gate.
- **After (callout in §13):**

> [UPDATED: A6-P13: per-tenant submit rate-limit] `DeploymentConfig.per_tenant_submit_qps: uint32 | None`; leaky-bucket relaxed counter; on breach → `AdmissionDecision::REJECTED` with `TenantQuotaExceeded`; partitioned per tenant.

- **Rationale:** S1 — multi-tenant isolation at admission.

### A8-P2 — Single `IEventLoopDriver` test seam

- **Before:** Event-loop was not test-drivable without bespoke hooks.
- **After (callout in §13):**

> [UPDATED: A8-P2: IEventLoopDriver + step() + RecordedEventSource] Gated on `enable_test_driver`; `EventLoopRunner::step(max_events)` controls draining; `RecordedEventSource` replays traces deterministically. Release binary unaffected.

- **Rationale:** X5 — DfT seam without release cost.

### A8-P3 — Stats + histograms on scheduler

- **Before:** Stats structs undeclared at this layer.
- **After (callout in §13):**

> [UPDATED: A8-P3: stats + histograms] Enumerate `LayerStats`, `TaskManagerStats`, `WorkerStats`. `LatencyHistogram` with pre-allocated log-scale buckets; < 5 ns branchless insert; seqlock snapshot; bucket array pinned to CONTROL (see `profiling.md §13`).

- **Rationale:** O3 / O5.

### A8-P12 — Stable PhaseIds for Submission lifecycle

- **Before:** PhaseIds were ad-hoc tags at callsites.
- **After (callout in §13):**

> [UPDATED: A8-P12: PhaseId constants] `SUBMISSION_ADMIT, SUBMISSION_RETIRE, DATA_DEP_SCAN, BARRIER_ATTACH, NONE_ATTACH, WORKSPACE_ALLOC, ADMISSION_QUEUE_DRAIN, WORKER_DISPATCH, WORKER_COMPLETE`. Per-phase durations sum to admission latency within 1%.

- **Rationale:** O1.

### A10-P1 — Default `producer_index` sharding policy

- **Before:** Sharding policy not specified; default implicit.
- **After (callout in §13):**

> [UPDATED: A10-P1: default sharding policy] Shard count 8 Host / 4 Device / 1 Chip and below; RW lock per shard keyed by `hash(BufferRef) mod N`; default `shards=1` preserves today's fast path; deployment cue triggers opt-in at `concurrent_submitters × cluster_nodes ≥ 64` (ADR-019).

- **Rationale:** P4 / DS6.

### Cross-ref — A5-P11 peer-health FSM source-of-truth

- **Before:** Scheduler implicitly shadowed breaker/heartbeat state.
- **After (callout in §13):**

> [UPDATED: A5-P11-ref: PeerHealthState source-of-truth in distributed.md §3.5] Scheduler branches on peer health consult the unified FSM in `distributed.md §3.5`; no scheduler-local shadow state.

- **Rationale:** R3 / R5 + DRY — one authority across breaker + heartbeat + auth.

## Verification steps

1. `rg -n "\[UPDATED: " docs/pypto-runtime-design/modules/scheduler.md` prints 14 entries in §13 (13 proposals + cross-ref).
2. `rg -n "producer_index_capacity|producer_index_shards" docs/pypto-runtime-design/modules/scheduler.md` confirms §8 Configuration aligns with A1-P1 / A1-P14.
3. `rg -n "ResourceExhausted" docs/pypto-runtime-design/modules/hal.md docs/pypto-runtime-design/modules/scheduler.md` appears in both (A5-P12 row mirror).
4. Cross-view: `distributed.md §3.5` (new) provides the canonical `PeerHealthState`.
