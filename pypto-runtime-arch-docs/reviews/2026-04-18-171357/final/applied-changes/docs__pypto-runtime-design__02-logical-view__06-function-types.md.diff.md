# Edit: docs/pypto-runtime-design/02-logical-view/06-function-types.md

- **Driven by proposal(s):** A3-P9
- **ADR link(s):** —
- **Timestamp (UTC):** 2026-04-19
- **Edit kind:** modification (single callout insertion)

## Location

- Path: `docs/pypto-runtime-design/02-logical-view/06-function-types.md`
- Sections touched: §2.3.1 (AICore Function)
- Line range before edit: 23 (insertion point; file grew to 59 lines)

## Before

The AICore Function section did not specify where `spmd_index` and `spmd_size` arrive at kernel invocation, and referenced `SubmissionDescriptor::Kind` as the SPMD discriminator.

## After

### A3-P9 — `spmd_index` / `spmd_size` delivery (§2.3.1)

- **After (callout):**

> [UPDATED: A3-P9: spmd_index/spmd_size delivery via TaskArgs.scalar_args] For SPMD sub-tasks, the runtime injects `spmd_index` and `spmd_size` at **reserved** `TaskArgs.scalar_args` indices 0 and 1 respectively (no separate `Kind` discriminator). SPMD-ness is derivable from `SubmissionDescriptor.spmd.has_value()` (A9-P4: `enum Kind` is removed). Frontend-authored AICore Functions read these two scalars via the same ABI as any other scalar argument; the reservation is contractual and validated by `A3-P7` admission precondition `SpmdScalarReservation`.

- **Rationale:** V2 / V3 — removes an enum discriminator and surfaces SPMD inputs through the existing, stable scalar-args ABI.

## Rationale

Injecting `spmd_index` / `spmd_size` into reserved scalar slots eliminates a bespoke SPMD-only kernel ABI and allows the same kernel to be invoked in single, group, and SPMD modes without conditional compilation. It also closes the loop with A9-P4 (drop `Kind`) by removing the other place where `Kind` was user-visible.

## Verification steps

1. `grep -n "\[UPDATED: A3-P9" docs/pypto-runtime-design/02-logical-view/06-function-types.md` returns 1 hit.
2. Confirm `07-task-model.md` A9-P4 callout says `Kind` is documentary, not a struct field.
3. Confirm `09-interfaces.md §2.6.1.A` admission precondition catalog lists `SpmdScalarReservation` (or equivalent SPMD scalar-args validation).

## Line-count delta

- Before: 45 lines
- After: 59 lines (+14 lines)
