# Edit: docs/pypto-runtime-design/02-logical-view/07-task-model.md

- **Driven by proposal(s):** A1-P4, A3-P1, A3-P9, A3-P11, A4-P2, A5-P4, A9-P4
- **ADR link(s):** ADR-022 (Task hot/cold split); ADR-023 (canonical Task FSM)
- **Timestamp (UTC):** 2026-04-19
- **Edit kind:** modification (surgical callouts + one anchor fix + one `TaskDescriptor` field addition)

## Location

- Path: `docs/pypto-runtime-design/02-logical-view/07-task-model.md`
- Sections touched: §2.4.A (Submission model), §2.4.1 (Task structure), §2.4.4 (Task state summary), §2.4.5 (Deferred completion), §2.4.E (SPMD sibling tagging)
- Line range before edit: 27, 45, 117, 185, 201, 217 (insertion points; file grew to 289 lines)

## Before

- Broken anchor `#21_3_1_a-...` on Outstanding Submission Window cross-ref.
- `SubmissionDescriptor::Kind` treated as a struct field.
- Task struct layout un-specified re: hot/cold split; `TaskDescriptor` had no `idempotent` flag.
- `FAILED` appeared as a peer terminal state to `ERROR` / `CANCELLED`.
- Deferred-completion paragraph lacked an assumption marker.
- SPMD siblings had no cancellation-tag specification.

## After

### A4-P2 — anchor fix (§2.4.A)

- **Before:** `#21_3_1_a-outstanding-submission-window`
- **After:** `#2131a-outstanding-submission-window` (markdown-standard slug). Inline note: `[UPDATED: A4-P2: anchor slug fix]`.

### A9-P4 — drop `SubmissionDescriptor::Kind` (§2.4.A Submission Kinds table)

- **After (callout below the Kinds table):**

> [UPDATED: A9-P4: Kind is documentary only] `Kind ∈ {SINGLE, GROUP, SPMD}` is a **documentary classification** in this table. It is **not** a `SubmissionDescriptor` struct field; `SubmissionDescriptor::Kind` and `SubmissionDescriptor::kind` are removed. SPMD = `spmd.has_value()`; SINGLE = `tasks.size()==1 && !spmd.has_value()`; GROUP = `tasks.size()>=1 && !spmd.has_value()`.

### A1-P4 / A5-P4 — Task hot/cold split + `idempotent` flag (§2.4.1)

- **After (callout above §2.4.1):**

> [UPDATED: A1-P4/A5-P4: Task hot/cold split + idempotent bit] The Task struct is partitioned into a cache-line-aligned hot 64-byte tier (state, generation, pending_deps, dispatch_target, flags) and a cold tail (successor list, arg vector, diagnostics). Successor list is the target of the A3-P4 `DEP_FAILED` walk. Full layout in `modules/core.md §8`. A new `bool idempotent = true;` is added to `TaskDescriptor`; the runtime may retry a `FAILED` Task only when `idempotent && cause ∈ retry_catalog` (see `12-dependency-model.md` A5-P4 callout).

### A3-P1 / A3-P11 — canonical Task FSM + drain semantics (§2.4.4)

- **After (callout below the state-summary table):**

> [UPDATED: A3-P1: canonical terminal TaskState] Canonical FSM: `ADMITTED → READY → EXECUTING → COMPLETING → {COMPLETED | ERROR | CANCELLED}`. `FAILED` is a **Worker-side spelling** of the Scheduler-side `ERROR` and is retained only as a Worker-trace label. Cross-view spellings are normalized in `appendix-a-glossary.md`.
>
> [UPDATED: A3-P11: drain semantics] Drain rejection of in-flight `submit()` returns `AdmissionStatus::REJECT(Drain)`; the flag is sticky (A3-P12). Tasks already past admission run to retirement; `drain()` does not cancel in-flight Tasks.

### A3-P11 — `[ASSUMPTION]` marker (§2.4.5 Deferred Completion)

- **Before:** "Deferred Completion. The runtime currently tracks deferred completion …"
- **After:** "**[ASSUMPTION] Deferred Completion.** The runtime currently tracks deferred completion … See `09-open-questions.md Q8 Deferred Completion Tracking` for the open decision."

### A3-P9 — SPMD sibling tagging (§2.4.E)

- **After (callout):**

> [UPDATED: A3-P9: SPMD sibling cancellation tag] SPMD sibling sub-tasks are tagged at admission with `(parent_submission_id, spmd_index)`. A `DEP_FAILED` on any sibling triggers a sibling-scan by `parent_submission_id` and issues `CANCEL_REQUEST` to all still-pending siblings (A3-P4 aggregation edge).

## Rationale

07-task-model.md is where the Task / Submission contract is most visible to frontends. The edits (a) remove an unused discriminator (A9-P4), (b) tighten FSM terminology across views (A3-P1), (c) bind the retry contract to a single opt-in bit (A5-P4), (d) connect the cold-tail layout to the failure-propagation walk (A1-P4 ↔ A3-P4), and (e) fix a broken anchor (A4-P2).

## Verification steps

1. `grep -n "\[UPDATED: A" docs/pypto-runtime-design/02-logical-view/07-task-model.md` returns ≥ 6 hits.
2. `grep -n "#2131a-" docs/pypto-runtime-design/02-logical-view/07-task-model.md` — anchor fix applied.
3. `FAILED` must not appear in the canonical FSM diagram of `modules/core.md`.
4. `TaskDescriptor` in `modules/core.md` should also gain the `idempotent` field (cross-view sync).

## Line-count delta

- Before: 250 lines (pre-callouts)
- After: 289 lines (+39 lines)
