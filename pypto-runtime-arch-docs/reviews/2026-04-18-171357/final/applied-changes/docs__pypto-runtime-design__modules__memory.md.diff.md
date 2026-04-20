# Edit: docs/pypto-runtime-design/modules/memory.md

- **Driven by proposal(s):** A5-P7, A6-P5, A8-P3 (+ A3-P2 cross-ref)
- **ADR link(s):** —
- **Timestamp (UTC):** 2026-04-19
- **Edit kind:** additive (consolidated `## 13. Review Amendments (R3)` section appended before Document status footer)

## Location

- Path: `docs/pypto-runtime-design/modules/memory.md`
- Section / heading: new `## 13. Review Amendments (R3)`; references §2.1, §2.2, §2.4, §4.5
- Line range before edit: 527–528 (footer only)

## Before

```527:528:docs/pypto-runtime-design/modules/memory.md
**Document status:** Draft — ready for review.
```

## After

A new `## 13. Review Amendments (R3)` heading is inserted before the footer, with one `[UPDATED: <id>: ...]` quoted callout per assigned proposal and one cross-reference to `02-logical-view/09-interfaces.md`. Per-proposal edits follow.

## Per-proposal edits

### A5-P7 — `Timeout` on `IMemoryOps` async

- **Before:** `wait(AsyncHandle) → Status`, `poll(AsyncHandle) → bool` (§2.2 table, lines 102–103).
- **After (callout in §13):**

> [UPDATED: A5-P7: Timeout on IMemoryOps async] `wait(AsyncHandle, Timeout) → Status`; `poll(AsyncHandle, Timeout = Timeout::zero()) → bool`. Returns `TransportTimeout` on deadline. RDMA backend dedup-window + A5-P10 idempotency cover retry side-effects.

- **Rationale:** R1 — bounded wait on every async memory op.

### A6-P5 — Scoped, revocable RDMA `rkey`

- **Before:** `rkey` lifetime was not bound to Submission retirement.
- **After (callout in §13):**

> [UPDATED: A6-P5: scoped revocable rkey] `register_peer_read(BufferRef, NodeId, Scope) → RkeyToken` + `deregister_peer_read(RkeyToken)`. `distributed_scheduler` deregisters on `SUBMISSION_RETIRED`. Replayed `REMOTE_DATA_READY` after retirement fails with `RkeyRevoked`.

- **Rationale:** S2 / S3 — prevents long-lived credentials and replay.

### A8-P3 — `MemoryStats` schema

- **Before:** Memory stats surface was undeclared.
- **After (callout in §13):**

> [UPDATED: A8-P3: MemoryStats schema] Enumerate `MemoryStats` with concrete fields + units (arena peak bytes, ring occupancy, RDMA outstanding MRs, registration churn). Latency histogram shares the A8-P3 `LatencyHistogram` primitive; bucket arrays pinned to CONTROL (see `profiling.md §13`).

- **Rationale:** O3 / O5.

### A3-P2 — Cross-ref to normative `std::expected` submit signature

- **Before:** Error handoff from memory → submit was implicit.
- **After (callout in §13):**

> [UPDATED: A3-P2-ref: expected<SubmissionHandle, ErrorContext>] Memory-side errors that bubble through `submit()` are packaged as `ErrorContext` and returned as `unexpected(ErrorContext{...})`; authoritative in `02-logical-view/09-interfaces.md`.

- **Rationale:** LSP / D7 — keeps error propagation consistent across modules.

## Verification steps

1. `rg -n "\[UPDATED: " docs/pypto-runtime-design/modules/memory.md` prints 4 entries in §13.
2. `rg -n "register_peer_read|deregister_peer_read" docs/pypto-runtime-design/modules/memory.md` surfaces A6-P5.
3. `rg -n "TransportTimeout" docs/pypto-runtime-design/modules/memory.md` surfaces A5-P7.
4. Cross-view: `distributed.md §5.3` still references `REMOTE_DATA_READY`; replayed-after-retirement path is now rejected with `RkeyRevoked`.
