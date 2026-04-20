# Edit: docs/pypto-runtime-design/02-logical-view/03-worker.md

- **Driven by proposal(s):** A1-P9, A5-P8, A5-P9
- **ADR link(s):** ADR-020 (Worker UNAVAILABLE variant fold)
- **Timestamp (UTC):** 2026-04-19
- **Edit kind:** modification (surgical callout insertions)

## Location

- Path: `docs/pypto-runtime-design/02-logical-view/03-worker.md`
- Sections touched: §2.2.2 (Worker state machine), §2.2.4 (WorkerGroup)
- Line range before edit: 77, 131 (insertion points; file grew to 164 lines)

## Before

- Worker state section listed `IDLE / EXECUTING / UNAVAILABLE / QUARANTINED` as four peer states.
- WorkerGroup section noted per-Worker availability without specifying the encoding or partial-group degradation policy.

## After

Two callout blocks, adjacent to the relevant paragraphs.

## Per-proposal edits

### A5-P9 — UNAVAILABLE Worker state variants (§2.2.2)

- **Before:** `QUARANTINED` was a peer state of `UNAVAILABLE`.
- **After (callout):**

> [UPDATED: A5-P9: fold QUARANTINED into UNAVAILABLE + variant] `QUARANTINED` is dissolved into `UNAVAILABLE + variant ∈ {Permanent, Quarantine(duration_ms)}`. `Quarantine(duration)` auto-transitions back to `IDLE` when the duration expires and `WorkerHealthCheckPolicy.verify(worker)` passes; `Permanent` requires operator intervention (`ADMIN_RESET`). Config knobs: `worker_quarantine_min_ms` (default 200 ms), `worker_quarantine_max_ms` (default 5 s), `worker_quarantine_backoff` (default exponential, base 2).

- **Rationale:** V3 / R5 — state machine cardinality reduction without losing expressiveness; all recovery-timing knobs named and defaulted.

### A1-P9 / A5-P8 — WorkerGroup availability bitmap + degradation policy (§2.2.4)

- **Before:** Group availability was described as "per-Worker availability"; partial-group response was informal.
- **After (callout):**

> [UPDATED: A1-P9: WorkerGroup availability bitmap] For up to 64 WorkerGroups per Layer, per-group readiness is encoded as a single `uint64_t available_mask` alongside the group table; scan uses `__builtin_ctzll(mask & eligible_mask)` — O(1) next-available. On the 108-core Ascend a5 platform this also serves as the ACK bitmap for Worker completion. For > 64 groups, a segmented bitmap (array-of-uint64) is used; degradation to O(⌈N/64⌉) is still cache-line friendly.
>
> [UPDATED: A5-P8: partial-group degradation policy] `partial_group_policy ∈ {FAIL_SUBMISSION, SHRINK_AND_RETRY, DEGRADED_CONTINUE}`; rule bindings in `modules/scheduler.md §13` cross-ref A5-P8 table.

- **Rationale:** X2 / R4 — cache-line-sized resource mask + first-class degradation.

## Rationale

These callouts keep Worker state and group-availability narrative intact and collapse the previous ambiguity over `QUARANTINED` vs `UNAVAILABLE` and informal degradation notes. The bitmap encoding is the same ACK structure that A1-P9 specified in `modules/scheduler.md`, so `03-worker.md` now points to a single source of truth.

## Verification steps

1. `grep -n "\[UPDATED: A" docs/pypto-runtime-design/02-logical-view/03-worker.md` should return 2 hits.
2. Confirm the FSM references in `07-task-model.md §2.4.4` (A3-P1) do not re-introduce `QUARANTINED`.
3. Confirm `partial_group_policy` value list matches `modules/scheduler.md` A5-P8 rule table.

## Line-count delta

- Before: 150 lines (pre-callouts)
- After: 164 lines (+14 lines)
