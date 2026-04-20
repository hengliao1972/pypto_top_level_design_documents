# Edit: docs/pypto-runtime-design/modules/core.md

- **Driven by proposal(s):** A1-P4, A3-P1, A6-P7, A7-P3, A7-P7, A7-P8, A8-P3
- **ADR link(s):** ADR-016 (TaskState canonical spelling); ADR-017 (Closed-enum-in-hot-path policy / Kind drop)
- **Timestamp (UTC):** 2026-04-19
- **Edit kind:** additive (consolidated `## 13. Review Amendments (R3)` section appended before Document status footer)

## Location

- Path: `docs/pypto-runtime-design/modules/core.md`
- Section / heading: new `## 13. Review Amendments (R3)`; references §2.3, §2.5, §2.6, §2.8, §3.3, §4.2, §5.1
- Line range before edit: 568–569 (footer only)

## Before

```568:569:docs/pypto-runtime-design/modules/core.md
**Document status:** Draft — ready for review.
```

## After

A new `## 13. Review Amendments (R3)` heading is inserted before the footer, with one `[UPDATED: <id>: ...]` quoted callout per assigned proposal. Per-proposal edits follow.

## Per-proposal edits

### A1-P4 — Enumerate `Task` hot/cold field split

- **Before:** `Task` struct / §4.2 did not distinguish hot vs cold tier nor justify AoS.
- **After (callout in §13):**

> [UPDATED: A1-P4: Task hot/cold field split] Hot 64-B tier: `state, fan_in_counter, submission_id, exec_type_id, worker_id, dispatch_ts_ns`. Cold tail: `function_id, args_blob_ptr, parent_task_key, dbg_name, trace_flags, error_ctx_ptr`. AoS per-slot; first cache line hot; `ErrorContext` written only on FAILED; SoA for `fan_in_counter` considered and rejected.

- **Rationale:** X4 — pins the hot/cold split so downstream perf bounds (A1-P1/P2) hold.

### A3-P1 — `ERROR` (and optional `CANCELLED`) TaskState + canonical spelling

- **Before:** TaskState enumerated `FREE..RETIRED` with no terminal error state.
- **After (callout in §13):**

> [UPDATED: A3-P1: add ERROR/CANCELLED TaskState] Add `ERROR` reachable from `DISPATCHED, EXECUTING, COMPLETING` with `on_fail(task, error)`. Optional `CANCELLED` with `on_cancel(task, reason)`. Canonical spelling: `COMPLETING → ERROR` (Task) and `FAILED` (Worker). See ADR-016.

- **Rationale:** LSP + D7 — Task FSM is lossy without a terminal failure state.

### A6-P7 — `FunctionDesc.Attestation`

- **Before:** Function registration had no attestation field.
- **After (callout in §13):**

> [UPDATED: A6-P7: FunctionDesc.Attestation] Add `Attestation { key_id; signature_over_hash; }` to `FunctionDesc`. `DeploymentConfig.allow_unsigned_functions` defaults false in multi-tenant, true in single-tenant / SIM. Unsigned + `allow_unsigned=false` → `FunctionNotAttested`. Gated by `trust_boundary.multi_tenant`.

- **Rationale:** S1 / S3 — binary provenance gate.

### A7-P3 — Handle types sourced from `core/types.h`

- **Before:** `DeviceAddress`, `NodeId` lived in `hal/` headers.
- **After (callout in §13):**

> [UPDATED: A7-P3: handle types sourced from core/types.h] `using DeviceAddress = std::uint64_t;` and `using NodeId = std::uint64_t;` declared in `core/types.h`. HAL re-exports from its own `types.h`. `core::TaskHandle` compiles without `hal/`.

- **Rationale:** D2 — removes the upstream dependency.

### A7-P7 — Forward-decl contract

- **Before:** §3.3 did not mandate forward decls on `ISchedulerLayer`.
- **After (callout in §13):**

> [UPDATED: A7-P7: forward-decl contract] `core/i_scheduler_layer.h` forward-declares `memory::IMemoryManager`, `memory::IMemoryOps`, `transport::IVerticalChannel`, `transport::IHorizontalChannel`; full defs only included by implementers. Logical DAG adds "interface reference only" edges.

- **Rationale:** D2 / D6 — keeps consumers from transitively pulling full transport/memory.

### A7-P8 — Consolidate `ScopeHandle` ownership

- **Before:** `ScopeHandle` referenced from both `core/` and `memory/` with ambiguous canonical site.
- **After (callout in §13):**

> [UPDATED: A7-P8: canonical ScopeHandle in core/] Declare `ScopeHandle` in `core/include/core/scope.h`. `memory/` consumes via `#include <core/scope.h>`. Glossary lists a single canonical definition.

- **Rationale:** D7 — single source of truth.

### A8-P3 — Stats structs + latency histograms

- **Before:** `LayerStats`, `TaskManagerStats`, `WorkerStats` not normatively enumerated here.
- **After (callout in §13):**

> [UPDATED: A8-P3: stats structs + latency histograms] Enumerate `LayerStats`, `TaskManagerStats`, `WorkerStats` with concrete fields + units. Add `LatencyHistogram` (pre-allocated log-scale buckets; branchless insert; seqlock snapshot). Bucket arrays pinned to CONTROL region (see `profiling.md §13`).

- **Rationale:** O3 / O5 — makes observability shape explicit on the core side.

## Verification steps

1. `rg -n "\[UPDATED: " docs/pypto-runtime-design/modules/core.md` prints one line per proposal id in §13.
2. `rg -n "COMPLETING → ERROR" docs/pypto-runtime-design/modules/core.md` returns a hit (canonical spelling) and cross-references ADR-016.
3. `rg -n "core/types.h" docs/pypto-runtime-design/modules/core.md docs/pypto-runtime-design/modules/hal.md` confirms both files agree on A7-P3.
4. Cross-view: `appendix-a-glossary.md` `Task State` entry (A4-P9) lists the canonical lifecycle.
