# Edit: docs/pypto-runtime-design/appendix-a-glossary.md

- **Driven by proposal(s):** A4-P5, A4-P9, A4-R3-P1, A7-P8 (plus §7 catalog: A1-P3, A1-P6, A1-P8, A1-P9, A3-P7, A3-P8, A3-P12, A5-P1, A5-P2, A5-P3, A5-P5, A5-P8, A5-P11, A5-P13, A6-P2, A6-P9, A6-P11, A7-P4, A7-P6, A8-P2, A8-P4, A8-P5, A8-P7, A8-P8, A9-P2, A9-P5, A10-P3, A10-P7)
- **ADR link (if any):** ADR-014 (runtime::composition), ADR-015 (Invariant I-DIST-1), ADR-016 (TaskState), ADR-017 (closed-enum policy), ADR-018 (PeerHealthState), ADR-020 (coordinator_generation)
- **Timestamp (UTC):** 2026-04-19
- **Edit kind:** additive (new continuation section) + one in-place modification of the `Task State` entry

## Location

- Path: `docs/pypto-runtime-design/appendix-a-glossary.md`
- Section / heading:
  - In-place edit: `Task State` row in the main alphabetical table (original line 88)
  - Additive edit: new `## Additional v1 Canonical Terms (Architecture Review 2026-04-18)` section appended after the existing table (after the former line 97 `Worker State` row)
- Line range before edit: 1–97 (97-line file); after edit: 1–151

## Before

```88:88:docs/pypto-runtime-design/appendix-a-glossary.md
| **Task State** | Lifecycle phase (FREE → SUBMITTED → ... → RETIRED). | [02 §2.4.4](02-logical-view/07-task-model.md#244-task-state-summary), [04 §4.3](04-process-view.md#43-task-state-machine) |
```

(Prior file ended at line 97 with the `Worker State` row; no further content below.)

## After

Task State row updated in place:

```markdown
| **Task State** | Canonical lifecycle: `FREE → SUBMITTED → PENDING → DEP_READY → RESOURCE_READY → DISPATCHED → EXECUTING → COMPLETING → COMPLETED → RETIRED`, plus terminal `ERROR` (and optional `CANCELLED` if A3-P1b adopted). See [`modules/core.md §2.3`](modules/core.md) and ADR-016. [UPDATED: A4-P9: enumerate canonical lifecycle + ADR-016 link] | [02 §2.4.4](02-logical-view/07-task-model.md#244-task-state-summary), [04 §4.3](04-process-view.md#43-task-state-machine), [08 ADR-016](08-design-decisions.md) |
```

New continuation section appended after the existing `Worker State` row:

```markdown
## Additional v1 Canonical Terms (Architecture Review 2026-04-18)

The following terms were added during the architecture-review-enhancement run `2026-04-18-171357` and collected from §7 of the run's `final-proposal.md`. …

[UPDATED: A4-P5, A4-R3-P1, A7-P8, A1-P3, A1-P6, A1-P8, A1-P9, A3-P7, A3-P8, A3-P12, A5-P1, A5-P2, A5-P3, A5-P5, A5-P8, A5-P11, A5-P13, A6-P2, A6-P9, A6-P11, A7-P4, A7-P6, A8-P2, A8-P4, A8-P5, A8-P7, A8-P8, A9-P2, A9-P5, A10-P3, A10-P7: added v1 canonical terms]

| Term | Definition | Source proposal |
|------|------------|-----------------|
| `admission_pressure_policy` | … | A5-P8 |
| `AdmissionStatus` | … | A9-P5 |
| …                       | …  | …     |
| `WorkerState` | Worker FSM canonical type — states `{READY, BUSY, DRAINING, RETIRED, FAILED, UNAVAILABLE}` with `UNAVAILABLE` carrying `{Permanent, Quarantine(duration)}` (A5-P9 DRY-fold); cross-references A5-P11 / ADR-018 unified peer-health FSM. | A4-R3-P1 |
```

Already-present, no-op:

- `Simulation Mode` — already exists (line 79); §7 `SimulationMode` entry merely re-states the open-enum scope already captured by that row; no new row added.
- `Worker State` — already exists (line 97); §7 introduces the type name `WorkerState` with different canonical states. The old `Worker State` concept row is amended with a pointer to the new `WorkerState` type row below; no duplicate entry introduced.

## Rationale

- **A4-P9** demands the `Task State` row enumerate the canonical lifecycle and link to ADR-016 and `modules/core.md §2.3`. The in-place edit satisfies D7 (naming consistency) and V5 (diagram ↔ glossary alignment) without churning surrounding rows.
- **A4-P5** requires pointer entries for `EventLoopDeploymentConfig`, `SourceCollectionConfig` (folded), and `IEventCollectionPolicy` (retired) after the `EventHandlingConfig` entry. Because a continuation section is used (instead of inline insertion at line 28) the alphabetical sort of the original table is preserved in full, honoring the "preserve alphabetical sort where possible" rule.
- **A4-R3-P1** adds the `WorkerState` type-name row with the A5-P9 DRY-folded FSM states and cross-reference to A5-P11 (ADR-018).
- **A7-P8** establishes a single canonical `ScopeHandle` definition row, matching the module-side consolidation.
- The §7 catalog terms land in one batch marker so the diff stays ≤ 400 lines and each row still cites its driving proposal(s) in the third column.

## Verification steps

1. Confirm every term listed in §7 of `reviews/2026-04-18-171357/final/final-proposal.md` (32 rows) either appears in the new section, is already present in the main table (`Simulation Mode`, `Worker State`, `Task State`), or is explicitly noted as already-present above.
2. Confirm the `Task State` row cross-references ADR-016 and `modules/core.md §2.3`, satisfying A4-P9.
3. Scan the main alphabetical table (lines 1–97 pre-edit) to confirm no existing row was reordered or renamed by this edit.
4. Cross-check that `Worker State` still carries its original concept definition and now includes a forward pointer to the new `WorkerState` type row.
5. Re-run markdown table lint (if available) on the file; table separators and column counts (3 cols for the main table, 3 cols for the continuation table) are valid.
