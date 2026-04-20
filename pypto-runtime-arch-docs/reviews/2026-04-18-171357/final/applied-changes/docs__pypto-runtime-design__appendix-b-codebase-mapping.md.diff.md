# Edit: docs/pypto-runtime-design/appendix-b-codebase-mapping.md

- **Driven by proposal(s):** A2-P4, A4-P8, A4-R3-P1, A7-P4
- **ADR link (if any):** ADR-015 (Invariant I-DIST-1), ADR-016 (TaskState canonical spelling)
- **Timestamp (UTC):** 2026-04-19
- **Edit kind:** modification (Migration cross-link, TaskState rows update) + additive (WorkerState row, distributed-payload row)

## Location

- Path: `docs/pypto-runtime-design/appendix-b-codebase-mapping.md`
- Sections / headings touched:
  - Top intro block (A2-P4 cross-link to Migration & Transition Plan)
  - "Communication Mapping" table — new `Distributed payload structs` row appended (A7-P4)
  - "Task Subsystem Mapping" table — `TaskState (Host)` and `TaskState (Device)` rows updated in place (A4-P8)
  - "Worker Mapping" table — new `WorkerState` row appended (A4-R3-P1)
- Line range before edit: 1–82 (82-line file); after edit: 1–86

## Before

```3:5:docs/pypto-runtime-design/appendix-b-codebase-mapping.md
This table maps current Simpler runtime components to the formal terms and modules defined in the redesigned architecture. It serves as a refactoring roadmap.

---
```

```42:43:docs/pypto-runtime-design/appendix-b-codebase-mapping.md
| **TaskState (Host)** | `TaskState` enum (6 states: FREE, PENDING, READY, RUNNING, COMPLETED, CONSUMED) | `dist_types.h` | `core/task.h` (10 states) |
| **TaskState (Device)** | `PTO2TaskState` enum (5 states: PENDING, READY, RUNNING, COMPLETED, CONSUMED) | `pto_runtime2_types.h` | `core/task.h` (10 states) |
```

```60:61:docs/pypto-runtime-design/appendix-b-codebase-mapping.md
| **Worker (`"Host"`)** | `DistChipProcess` / `ChipWorker` | `src/common/distributed/`, `src/common/worker/` | `scheduler/host_scheduler` |
| **Worker (`"Chip"`)** | AICore via register dispatch | `platform_regs.h`, `aicpu_executor.cpp` | `scheduler/aicpu_scheduler` |
```

## After

Intro cross-link (A2-P4):

```markdown
> [UPDATED: A2-P4: cross-link to Migration & Transition Plan] For the phased retirement schedule of the current-implementation classes listed below (feature-flag, canary, rollback gate), see [`03-development-view.md §3.5 Migration & Transition Plan`](03-development-view.md#35-migration--transition-plan).
```

TaskState rows updated in place (A4-P8):

```markdown
| **TaskState (Host)** | `TaskState` enum (6 states: FREE, PENDING, READY, RUNNING, COMPLETED, CONSUMED) | `dist_types.h` | `core/task.h` (10 lifecycle states + `ERROR`; `CANCELLED` if A3-P1b adopted; see [`modules/core.md §2.3`](modules/core.md) and ADR-016) [UPDATED: A4-P8: sync TaskState count + new states] |
| **TaskState (Device)** | `PTO2TaskState` enum (5 states: PENDING, READY, RUNNING, COMPLETED, CONSUMED) | `pto_runtime2_types.h` | `core/task.h` (10 lifecycle states + `ERROR`; `CANCELLED` if A3-P1b adopted; see [`modules/core.md §2.3`](modules/core.md) and ADR-016) [UPDATED: A4-P8: sync TaskState count + new states] |
```

WorkerState row appended to Worker Mapping (A4-R3-P1):

```markdown
| **`WorkerState`** | Ad-hoc flags across `DistWorker` / `ChipWorker` state (no unified enum); recovery states implicit in `fault_handler` paths | `src/common/worker/`, `src/common/distributed/` | `core/worker.h` — canonical FSM `{READY, BUSY, DRAINING, RETIRED, FAILED, UNAVAILABLE{Permanent, Quarantine(duration)}}`; see [`02 §2.1.4.1`](02-logical-view/03-worker.md#2141-worker-state-machine), A5-P9 DRY-fold, A5-P11 unified peer-health FSM (ADR-018) [UPDATED: A4-R3-P1: WorkerState mapping] |
```

Distributed-payload row appended to Communication Mapping (A7-P4):

```markdown
| **Distributed payload structs** (`RemoteSubmitPayload`, `HeartbeatPayload`, `HandshakePayload`, …) | Inlined in distributed/transport glue; no clean ownership split | `src/common/distributed/` | `distributed/include/distributed/protocol_payloads.h` (owner = `distributed/`); `transport/messages.h` keeps only `MessageHeader`, framing, `MessageType` tags. Invariant I-DIST-1 (IWYU-CI) forbids `transport/` from including `distributed/` headers. [UPDATED: A7-P4: move distributed payloads out of `transport/`; ADR-015] |
```

## Rationale

- **A2-P4** requires Appendix-B to cross-link the new §3.5 Migration & Transition Plan so readers of the mapping can trace current-class retirement phases. The single-line intro pointer satisfies the cross-link requirement with the minimum possible churn.
- **A4-P8** mandates syncing the TaskState cell text to the new canonical 10-state lifecycle plus `ERROR` and (conditionally) `CANCELLED`, cross-referencing `modules/core.md §2.3` and ADR-016. Both Host and Device rows receive identical updates.
- **A4-R3-P1** demands a new `WorkerState` row next to the TaskState-mapping rows; placed in the Worker Mapping sub-table so the current-to-canonical FSM mapping is co-located with the rest of the worker rows. Cites A5-P9 DRY-fold and A5-P11 / ADR-018.
- **A7-P4** requires Appendix-B's ownership column to reflect the move of distributed payload structs out of `transport/` into `distributed/include/distributed/protocol_payloads.h`. Added as a new row in the Communication Mapping table, citing Invariant I-DIST-1 and ADR-015.

## Verification steps

1. Confirm the intro `> [UPDATED: A2-P4 …]` link resolves to an existing heading in `03-development-view.md` once A2-P4 lands there.
2. Confirm the two updated TaskState rows both carry the `[UPDATED: A4-P8 …]` marker and reference ADR-016 + `modules/core.md §2.3`.
3. Confirm the new `WorkerState` row sits within the "Worker Mapping" sub-table (rows stay aligned under the same 4-column header) and cites A5-P9 / A5-P11 / ADR-018.
4. Confirm the new distributed-payload row is under "Communication Mapping" and explicitly names Invariant I-DIST-1 (A7-P4) + ADR-015.
5. Spot-check that no other rows were reordered or dropped and all column separators remain correct.
