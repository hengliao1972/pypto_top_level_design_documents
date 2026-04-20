# Edit: docs/pypto-runtime-design/02-logical-view/02-scheduler.md

- **Driven by proposal(s):** A1-P1, A1-P2, A1-P5, A1-P8, A1-P14, A3-P4, A3-P7, A3-P8, A3-P15, A5-P6, A5-P8, A9-P2, A10-P1, A10-P7, A10-P10
- **ADR link(s):** ADR-019 (admission shard default), ADR-018 (watchdog normative elision), ADR-017 (`std::expected` admission return)
- **Timestamp (UTC):** 2026-04-19
- **Edit kind:** modification (surgical callout insertions; no section rewrites)

## Location

- Path: `docs/pypto-runtime-design/02-logical-view/02-scheduler.md`
- Sections touched: §2.1.3.1.A (Submission admission), §2.1.3.1.B (Dependency resolution), §2.1.3.1.C (Concurrency), §2.1.3.2 (Completion contract), §2.1.3.5 (Event-driven execution)
- Line range before edit: 70–364 (insertion points; file grew to 435 lines)

## Before

The sections listed above contained admission/DAG/event-loop narrative but lacked:
- admission queue sizing and diagnostic sub-codes,
- precondition/validation catalog cross-refs,
- `producer_index` capacity/placement/rehash bounds,
- shard-count `LevelParam` default and deployment threshold,
- SPMD aggregation edge / successor walk for `DEP_FAILED`,
- deadman-timer write-elision cross-ref,
- `DeploymentMode`/`PolicyKind` closed-enum statement and `IEventLoopDriver` test-seam cross-ref,
- SPMD fan-out and event-loop stage latency budgets.

## After

Seven focused `> [UPDATED: <id>: <reason>]` callout blocks were inserted adjacent to the relevant paragraphs. The callouts are adjacent to the text they qualify; no prior narrative was removed.

## Per-proposal edits

### A1-P8 — admission queue sizing + diagnostic sub-codes (§2.1.3.1.A)

- **Before:** Admission rejection path listed only the high-level contract.
- **After (callout):**

> [UPDATED: A1-P8: admission queue sizing + diagnostic sub-codes] The `Admission queue` is a bounded SPSC ring of `SubmissionDescriptor*` with capacity `max_admission_backlog` (default 64). Overflow returns `AdmissionStatus::REJECT(Exhaustion)` with sub-code `WouldBlock`. The `ReadyQueue` is an intrusive min-heap over Task slot indices sized `max_outstanding_submissions × expected_tasks_per_submission × 2`, placed in the CONTROL region with `isolate_cache_line=true`. Sub-codes `TaskSlotExhausted` / `ReadyQueueExhausted` route via A3-P7.

- **Rationale:** X2 / X4 — capacity-bounded, no hot-path allocations.

### A3-P7 — admission-path preconditions + SPMD sub-codes (§2.1.3.1.A)

- **After (callout):**

> [UPDATED: A3-P7: admission-path preconditions + SPMD sub-codes] Before step 1, TaskManager validates `SubmissionDescriptor` preconditions (`EmptyTasks / SelfLoopEdge / EdgeIndexOOB / BoundaryIndexOOB / WorkspaceSubrangeOOB / WorkspaceSizeMismatch`). SPMD additionally validates `SpmdIndexOOB / SpmdSizeMismatch / SpmdBlockDimZero`. Catalog is the SSoT for admission error mapping and is shared with `09-interfaces.md §2.6.1.A`.

- **Rationale:** R4 / V5 — precondition catalog mirrored at public API.

### A3-P8 — cyclic `intra_edges` detection + FE_VALIDATED fast-path (§2.1.3.1.A)

- **After (callout):**

> [UPDATED: A3-P8: cyclic intra_edges detection] Step 2 performs white/grey/black DFS; release builds skip when `SubmissionDescriptor.flags & FE_VALIDATED` is set. Debug always runs. `AdmissionStatus::REJECT(Validation)` sub-code `CyclicDependency`.

- **Rationale:** X2 — amortized O(|tasks|+|edges|) with fast-path elision.

### A3-P15 — NONE-mode hidden-dependency debug cross-check (§2.1.3.1.A)

- **After (callout):**

> [UPDATED: A3-P15: NONE-mode hidden-dep cross-check] Debug builds dry-run `DATA`-style lookup; hit logs `WARN NoneModeHiddenDependency` but is not enforced. Release skips. See `07-cross-cutting-concerns.md §7.2`.

- **Rationale:** R6 / O5 — diagnostic without release penalty.

### A5-P8 — admission-pressure degradation policy (§2.1.3.1.A)

- **After (callout):**

> [UPDATED: A5-P8: admission-pressure degradation policy] `admission_pressure_policy ∈ {REJECT, COALESCE, DEFER}`; `max_deferred` bounds DEFER queue. Each value tied to an A8-P5 AlertRule; rule bindings in `modules/scheduler.md`.

- **Rationale:** R4 — back-pressure is first-class.

### A1-P1 / A1-P14 / A10-P10 — `producer_index` sizing + CONTROL placement (§2.1.3.1.B)

- **After (callout):**

> [UPDATED: A1-P1/A1-P14: producer_index pre-sized in CONTROL] Init capacity `max_outstanding_submissions × expected_outputs_per_submission × 2`; CONTROL region; `isolate_cache_line=true`. Rehash forbidden on hot path; overflow uses bounded open-addressed probe (L<0.5, probe ≤2) → `ResourceExhausted` and `IResourceAllocationPolicy.should_admit` back-pressure. `producer_index_capacity` exposed in `modules/scheduler.md §8`. [A10-P10] P4 ceiling: worst-case admission lookup fits within the CONTROL-region cache-line budget.

- **Rationale:** X2 / X4 / P5.

### A1-P2 / A10-P1 / A10-P7 — shard count `LevelParam` + bounded lock hold (§2.1.3.1.C)

- **After (callout):**

> [UPDATED: A1-P2/A10-P1/A10-P7: producer_index_shards LevelParam] `producer_index_shards` (alias `admission_shards`) is a `LevelParams` knob; default `max(1, scheduler_thread_count)` (=1 for Chip). Deployment cue: flip to per-shard mode when `concurrent_submitters × cluster_nodes ≥ 64`. Bounded shared hold ≤ `B×A` probes (≤500 ns); exclusive hold ≤ `B×A + |S.OUT∪S.INOUT|`; p99 reader ≤ 3 μs / writer ≤ 10 μs per shard.

- **Rationale:** X3 / P5.

### A3-P4 — SPMD aggregation edge + successor-walk `DEP_FAILED` (§2.1.3.2)

- **After (callout):**

> [UPDATED: A3-P4: DEP_FAILED propagation via successor walk + SPMD aggregation] On Task `ERROR`, TaskManager walks the successor list from the Task's cold tail, emitting `DEP_FAILED` on each successor. For SPMD submissions, an aggregation edge joins every sibling to the parent completion; any sub-task `ERROR` triggers SPMD-wide `DEP_FAILED` to all still-pending siblings (cancellable via `(parent_submission_id, spmd_index)` tag).

- **Rationale:** R2 / O5 — single-source failure propagation, cancellation-safe.

### A5-P6 — scheduler-thread watchdog / deadman timer (§2.1.3.5 prologue)

- **After (callout):**

> [UPDATED: A5-P6: scheduler-thread watchdog] Per-cycle monotonic timestamp; `scheduler_watchdog_ms` (default 250 ms) raises `SchedulerStalled` alert. Store may be elided into per-cycle TSC capture; elision bounded to ≤ 16 cycles (≤ 1.6 ms ≪ budget). See `modules/scheduler.md §5.2`.

- **Rationale:** R5 / O5 — detectable liveness without hot-path cost.

### A9-P2 / A1-P5 — closed enums + `IEventLoopDriver` test seam + latency budgets (end of §2.1.3.5)

- **After (callout):**

> [UPDATED: A9-P2/A1-P5: closed policy enums + event-loop latency budgets] `DeploymentMode` and `PolicyKind` are closed enums; `IEventLoopDriver` is the single test-only seam (gated on `LayerConfig.enable_test_driver`), omitted from release binaries. See `09-interfaces.md §2.6.6`. Latency budgets: SPMD fan-out admission ≤ p99 50 μs for 256 siblings; event-loop Collect → Classify → Drain stages ≤ p99 5 / 2 / 10 μs under `max_outstanding_submissions` saturation.

- **Rationale:** X1 / V2 — surface minimization + measurable SLOs.

## Rationale

The scheduler file absorbed the largest share of structural upgrades (hot-path sizing, admission taxonomy, failure propagation, watchdog). Each callout keeps the existing narrative intact while adding normative qualifiers adjacent to the paragraph they modify. This satisfies the surgical-callout rule from the review governance and keeps diff review localized.

## Verification steps

1. `grep -n "\[UPDATED: A" docs/pypto-runtime-design/02-logical-view/02-scheduler.md | wc -l` should return ≥ 9 hits.
2. Cross-reference `09-interfaces.md §2.6.1.A` (precondition catalog) and `modules/scheduler.md §13` (A1/A5/A10 rule bindings) to confirm the catalog and rule IDs match 1-to-1.
3. Ensure the `DeploymentMode` / `PolicyKind` note here agrees with `05-machine-level-registry.md` A2-P2 callout (closed v1 enums + extension_map).

## Line-count delta

- Before: 365 lines (baseline, pre-callouts)
- After: 435 lines (+70 lines, all within 9 `> [UPDATED: ...]` callouts)
