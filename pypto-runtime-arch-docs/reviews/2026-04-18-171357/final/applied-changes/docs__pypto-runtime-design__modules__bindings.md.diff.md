# Edit: docs/pypto-runtime-design/modules/bindings.md

- **Driven by proposal(s):** A1-P11, A3-P7, A3-P10, A5-P4, A6-P8, A6-P10, A6-P11, A7-P9, A8-P4, A8-P5, A8-P10 (+ cross-refs A2-P9, A4-P6, A8-P2)
- **ADR link(s):** ADR-011-R2, ADR-015, ADR-016, ADR-017 (cross-references; bindings behaviour tracks them)
- **Timestamp (UTC):** 2026-04-19
- **Edit kind:** additive (consolidated `## 13. Review Amendments (R3)` section appended before Document status footer)

## Location

- Path: `docs/pypto-runtime-design/modules/bindings.md`
- Section / heading: new `## 13. Review Amendments (R3)`; references §2.1, §2.2, §2.4, §4.3, §5.2, §5.5, §9.2, §9.5, §10
- Line range before edit: 417–418 (footer only)

## Before

```417:418:docs/pypto-runtime-design/modules/bindings.md
**Document status:** Draft — ready for review.
```

## After

A new `## 13. Review Amendments (R3)` heading is inserted before the footer, with one `[UPDATED: <id>: ...]` quoted callout per assigned proposal and three cross-reference callouts (A2-P9 Python-side, A4-P6 Related ADRs, A8-P2 RecordedEventSource). Per-proposal edits follow.

## Per-proposal edits

### A1-P11 — Per-arg Python↔C budget + per-arg validation

> [UPDATED: A1-P11: per-arg marshaling budgets] Scalar ≤20 ns; DLPack contiguous tensor ≤200 ns; BufferRef passthrough ≤30 ns; total ≤500 ns for ≤8 args; above 8 args adds `0.2 μs + 0.05 μs × extra_args`. Inline structural validation runs in a single pass < 30 ns. SPMD shape sub-codes surface here.

- **Rationale:** X9 / S3.

### A3-P7 — Submission preconditions at Python↔C boundary

> [UPDATED: A3-P7: submission preconditions + FE_VALIDATED] Sub-codes `EmptyTasks / SelfLoopEdge / EdgeIndexOOB / BoundaryIndexOOB / WorkspaceSubrangeOOB / WorkspaceSizeMismatch`; SPMD `SpmdIndexOOB, SpmdSizeMismatch, SpmdBlockDimZero`; routed `TaskSlotExhausted / ReadyQueueExhausted`. `FE_VALIDATED` flag skips re-validation in release.

- **Rationale:** G3 / S3.

### A3-P10 — Python exception mapping completeness

> [UPDATED: A3-P10: full Domain → simpler.*Error mapping] One row per `Domain`; scheduler-domain errors raise `simpler.SchedulerError` (not `simpler.RuntimeError`).

- **Rationale:** G1 / D7.

### A5-P4 — `idempotent: bool` on Python Task

> [UPDATED: A5-P4: simpler.Task(..., idempotent=True)] Scheduler MAY retry only when `idempotent == True`. Caller retry of a rejected submission is independent of the flag.

- **Rationale:** DS4.

### A6-P8 — DLPack byte-cap + capsule ownership

> [UPDATED: A6-P8: DeploymentConfig.max_import_bytes (1 GiB)] Stride × shape overflow guarded by A1-P11 compare stack + this cap. Rejects negative / mismatched-device / overflowing tensors with `ValueError`.

- **Rationale:** S3.

### A6-P10 — Capability-scoped log / trace sinks

> [UPDATED: A6-P10: add_sink(callable, *, severity_min, domains, logical_system_id)] Dispatch filters events by capability. Caller-scoped default; unscoped admin sinks require `diagnostic_admin_token`.

- **Rationale:** S2.

### A6-P11 — Python-side `register_factory` audit

> [UPDATED: A6-P11: Python register_factory audit] `simpler.runtime.register_factory(descriptor, *, registration_token)`. Missing/wrong → `simpler.AuthorizationError`; post-freeze → `simpler.RegistrationClosedError`. Audit event emitted.

- **Rationale:** S2 / S4.

### A7-P9 — Python `MemoryError` class dedup

> [UPDATED: A7-P9: rename simpler.errors.MemoryError] → `simpler.errors.SimplerMemoryAccessError(SimplerError)`. Keep `simpler.errors.SimplerOutOfMemory(builtins.MemoryError)` distinct.

- **Rationale:** D7.

### A8-P4 — `simpler.Runtime.dump_state() -> dict`

> [UPDATED: A8-P4: dump_state Python binding] Returns the structured JSON as a Python dict; scope = caller's `logical_system_id`; unscoped requires `diagnostic_admin_token`.

- **Rationale:** O4 / DfD.

### A8-P5 — `simpler.Runtime.on_alert(callback)`

> [UPDATED: A8-P5: on_alert + AlertRule Python binding] Python-visible `AlertRule` including `logical_system_id`.

- **Rationale:** O5.

### A8-P10 — Structured KV log bridge

> [UPDATED: A8-P10: add_log_sink Python KV shape] `{severity, category, kv_dict, correlation_id}`.

- **Rationale:** O2.

### Cross-ref A2-P9 — Python-side trace schema version

> [UPDATED: A2-P9: Python trace sink version guard] Python trace sinks receive `{magic, trace_schema_version, platform, mode}` once at stream start; mismatched major → `ProtocolVersionMismatch`.

### Cross-ref A8-P2 — `RecordedEventSource` Python seam

> [UPDATED: A8-P2-ref: simpler.testing.event_loop_driver(runtime)] Available when `enable_test_driver=True`; drives `EventLoopRunner::step()` for deterministic replays.

### Cross-ref A4-P6 — Related ADRs footer

> [UPDATED: A4-P6-ref: Related ADRs footer] Bindings-side behaviour tracks ADR-011-R2, ADR-015, ADR-016, ADR-017 (SimulationMode gating, I-DIST-1 header independence, canonical TaskState spelling, closed-enum rationale).

## Verification steps

1. `rg -n "\[UPDATED: " docs/pypto-runtime-design/modules/bindings.md` prints 14 entries in §13.
2. `rg -n "SimplerMemoryAccessError|SimplerOutOfMemory" docs/pypto-runtime-design/modules/bindings.md` surfaces A7-P9.
3. `rg -n "register_factory" docs/pypto-runtime-design/modules/bindings.md docs/pypto-runtime-design/modules/runtime.md` shows consistent surface with A6-P11.
4. Cross-view: `07-cross-cutting-concerns.md §7.2.4` (A6-P10 sink filtering) and `modules/runtime.md §13` (A8-P4 / A8-P5) corroborate.
