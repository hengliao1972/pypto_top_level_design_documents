# Applied Changes — `docs/pypto-runtime-design/06-scenario-view.md`

Run folder: `reviews/2026-04-18-171357/`.
Proposals applied (from `final/final-proposal.md`): **A3-P3**, **A3-P4**, **A3-P5**, **A3-P10**, **A4-R3-P2**, **A9-P5**.

Line-count delta: 154 → 165 (+11).

Each change is a tightly-scoped `> [UPDATED: <id>: <reason>]` callout or inline `**[UPDATED: <id>]**` marker; no section rewrites. New admission-path scenarios §6.2.5–§6.2.7 are owned by the A3-P3 author and are cross-referenced here only.

---

## Change 1 — A3-P4 and A3-P5 (SPMD aggregation edge + spmd_index retention)

**Anchor:** §6.1.3 SPMD Kernel Launch — two new rows inserted between step 6 and step 7 of the scenario table.

### Diff

```diff
 | 6 | Chip Scheduler tracks group completion: all 24 sub-tasks COMPLETED |
+| 6a | **[UPDATED: A3-P4]** On **any** sub-task failure, the SPMD aggregation edge fires: the parent Orchestration Task is notified with a `DEP_FAILED(producer_key, ErrorCode)` event and transitions `COMPLETING → ERROR`. Surviving siblings that are still `EXECUTING` run to completion but their outputs are discarded per A3-P5 sibling-cancellation policy. Siblings that had not yet dispatched are cancelled before dispatch. |
+| 6b | **[UPDATED: A3-P5]** Cancelled / discarded siblings contribute a `CANCELLED` entry to `ErrorContext.remote_chain` that **retains the sibling's `spmd_index`** so the parent (and user) can identify which SPMD shard was cancelled. The `spmd_index` is preserved for the full 24-way SPMD edge case. |
 | 7 | SPMD group complete → parent Orchestration Task's pending child counter decremented |
```

---

## Change 2 — A4-R3-P2 in §6.2.1 (Task canonical spelling)

**Anchor:** §6.2.1 "Failure: AICore Hang" — step 7 of the scenario table.

### Diff

```diff
 | 5 | Device Scheduler: parent Orchestration Task transitions COMPLETING → ERROR |
 | 6 | Error propagated to Host Scheduler via Vertical Channel |
-| 7 | Host Scheduler: Task COMPLETED(error); `notify_parent_complete` with error context |
+| 7 | **[UPDATED: A4-R3-P2]** Host Scheduler: parent Task transitions `COMPLETING → ERROR` (canonical spelling per ADR-016; was "COMPLETED(error)"); `notify_parent_complete` with error context |
```

Note: steps 3 (Worker `FAILED`) and 5 (Task `COMPLETING → ERROR`) already matched the canonical spelling and were left unchanged.

---

## Change 3 — A4-R3-P2 in §6.2.2 (ABORT_ALL spelling)

**Anchor:** §6.2.2 "Failure: Remote Node Crash" — step 6a.

### Diff

```diff
 | 5 | **Policy evaluation:** Check `DistributedConfig.failure_policy` |
-| 6a | **If ABORT_ALL:** All in-flight tasks on Node₀ are cancelled; parent Task → ERROR; Python gets `simpler.DistributedError` |
+| 6a | **[UPDATED: A4-R3-P2]** **If ABORT_ALL:** All in-flight tasks on Node₀ are cancelled; parent Task transitions `COMPLETING → ERROR` (canonical spelling per ADR-016); Python gets `simpler.DistributedError` |
```

Note: step 3 ("DistributedScheduler marks Node₁ as FAILED") already matched the canonical Worker `FAILED` spelling and was left unchanged. Step 6b/6c were checked and are consistent with ADR-016.

---

## Change 4 — A3-P3, A3-P10, A9-P5 in §6.2.3 (Task Slot Pool Exhaustion)

**Anchor 1:** §6.2.3 preamble — cross-reference callout to new §6.2.5–§6.2.7 (authored by A3-P3 primary-owner diff).

**Anchor 2:** §6.2.3 steps 2, 4, 5 — inline spelling fixes for A9-P5 (admission enum unification) and A3-P10 (Scheduler-domain exception mapping).

### Diff

```diff
 ### 6.2.3 Failure: Task Slot Pool Exhaustion

 **Context:** High-throughput submission; Chip-level task slot pool is full.

+> [UPDATED: A3-P3: Cross-references to new admission-path failure scenarios §6.2.5–§6.2.7.]
+> This scenario captures the **resource-exhaustion** axis of admission failure. Three additional admission-path failure scenarios are added by A3-P3 and sit as siblings in §6.2:
+>
+> - **§6.2.5** Admission Rollback on Cyclic `intra_edges` → `AdmissionStatus::REJECT(Validation::CyclicDependency)` (see A3-P7, A3-P8).
+> - **§6.2.6** Admission Rejection on Workspace Exhaustion → `AdmissionStatus::REJECT(Exhaustion::WorkspaceExhausted)`.
+> - **§6.2.7** `NONE` Mode Misuse (debug-only assertion, A3-P15) → `AdmissionStatus::REJECT(Validation::NoneModeHiddenDependency)`.
+>
+> All three cross-reference the admission storm behavior described in this §6.2.3.
+
 | Step | What Happens |
 |------|-------------|
 | 1 | Orchestration Function calls `scheduler.submit(child_task)` |
-| 2 | `IMemoryManager.alloc_task_slot()` fails → `ResourceExhausted` error |
+| 2 | **[UPDATED: A9-P5]** `IMemoryManager.alloc_task_slot()` fails → admission returns `AdmissionStatus::REJECT(Exhaustion)` (was `ResourceExhausted`; enums unified per A9-P5). `SubmissionDescriptor.flags` sets the `TaskSlotExhausted` sub-code (A1-P8, routed via A3-P7). |
 | 3 | `on_submit` handler detects failure; returns error status to Orchestration Function |
-| 4 | Orchestration Function MAY: (a) wait and retry (back-pressure), (b) propagate error upward |
-| 5 | If propagated: parent Task → ERROR → `simpler.RuntimeError("task slot pool exhausted at Chip level")` |
+| 4 | Orchestration Function MAY: (a) wait and retry (back-pressure), (b) propagate error upward (retry of a rejected submission by the caller is independent of the `idempotent` flag per A5-P4). |
+| 5 | **[UPDATED: A3-P10]** If propagated: parent Task transitions `COMPLETING → ERROR` → `simpler.SchedulerError("task slot pool exhausted at Chip level")` (Scheduler-domain errors raise `simpler.SchedulerError`, not generic `simpler.RuntimeError`, per full Domain→Python mapping). |
```

---

## Cross-references updated

- ADR-016 (Task canonical spelling: `COMPLETING → ERROR` for Task, `FAILED` for Worker).
- `modules/bindings.md §2.2` (A3-P10 Python exception mapping primary owner).
- `02-logical-view/09-interfaces.md` (A9-P5 `AdmissionStatus` unified enum primary owner).
- `02-logical-view/02-scheduler.md` (A3-P4 / A3-P5 primary owner for DEP_FAILED event).

## Not applied here (owner lives elsewhere)

New subsection bodies for **§6.2.5 / §6.2.6 / §6.2.7** (A3-P3 primary content) are authored by the scheduler-scenario owner in a subsequent slice; this root-view diff only installs the forward cross-references.
