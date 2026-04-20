# Edit: docs/pypto-runtime-design/02-logical-view/10-platform.md

- **Driven by proposal(s):** A9-P8
- **ADR link(s):** ADR-016 (closed v1 policy enums)
- **Timestamp (UTC):** 2026-04-19
- **Edit kind:** modification (single callout appended to §2.8.1)

## Location

- Path: `docs/pypto-runtime-design/02-logical-view/10-platform.md`
- Sections touched: §2.8.1 (Simulation Modes, end)
- Line range before edit: 41 (insertion point; file grew to 44 lines)

## Before

Platform and Variant were described via parameter tables but no statement was made about the polymorphism / discovery contract: a reader could reasonably infer that new platforms might be added via an `IPlatform` / `IVariant` interface.

## After

### A9-P8 — platform abstraction simplification (§2.8.1)

- **After (callout):**

> [UPDATED: A9-P8: platform abstraction simplification — closed enums + registry] `Platform`, `Variant`, and `SimulationMode` are closed enums; build targets `{a2a3, a2a3sim, a5, a5sim}` are fixed; new platforms land as a Machine Level Registry table entry + build-target bump (see `appendix-c-compatibility.md`). No `IPlatform` / `IVariant` polymorphic interface in v1. Selection is `O(1)` via the Registry's static table. Promotion trigger (Q14): a third platform with non-tabular parameters.

## Rationale

The two-platform v1 matrix (a2a3 / a5) does not justify a polymorphic platform interface; a registry table plus a closed enum gives the same reader ergonomics with fewer moving parts and no virtual dispatch in `MachineLevelDescriptor` selection. The promotion trigger is captured as Q14 so the decision is reversible.

## Verification steps

1. `grep -n "\[UPDATED: A9-P8" docs/pypto-runtime-design/02-logical-view/10-platform.md` returns 1 hit.
2. `09-open-questions.md` should contain Q14 (third-platform promotion trigger).
3. `05-machine-level-registry.md` must not add an `IPlatform` interface row.

## Line-count delta

- Before: 42 lines
- After: 44 lines (+2 lines)
