# Applied Changes — `docs/pypto-runtime-design/03-development-view.md`

Run folder: `reviews/2026-04-18-171357/`.
Proposals applied (from `final/final-proposal.md`): **A2-P8**, **A7-P1**.

Line-count delta: 316 → 322 (+6).

Each change is a tightly-scoped `> [UPDATED: <id>: <reason>]` callout; no section rewrites.

---

## Change 1 — A2-P8 (record intentional closures; cross-ref ADR-014)

**Anchor:** §3.1 Module Structure — immediately after the per-module responsibilities table (end of the "Module Responsibilities" block, just before `---` / §3.1.1).

**Rationale:** Record the frozen `runtime::composition` sub-namespace (ADR-014, co-owned with A7-P6) as an intentional closure in this view and cross-link the canonical deviations list.

### Diff

```diff
 | `bindings/` | Expose runtime to Python via nanobind. Handle GIL, type conversion, error translation. | Python `simpler.Runtime`, `simpler.Task`, `simpler.DistributedRuntime` |

+> [UPDATED: A2-P8: Cross-ref ADR-014 freeze of `runtime::composition` sub-namespace and record intentional closures.]
+> The `runtime/` module hosts a closed sub-namespace `runtime::composition` (`MachineLevelRegistry`, `MachineLevelDescriptor`, `DeploymentConfig`, `deployment_parser`). Compose-order and sub-namespace boundary are **frozen for v1** per **ADR-014 (ADR-A7-R3)** with documented promotion triggers (cross-consumer list ≥ 2 or deployment-schema churn > N/quarter). Registry being frozen at init, synchronous-only policies, and the closed `DepMode` enum are recorded as intentional closures in [`10-known-deviations.md`](10-known-deviations.md) (cross-linked from `09-open-questions.md` Q5/Q11/Q12).
+
 ---

 ## 3.1.1 Module Decomposition Summary
```

---

## Change 2 — A7-P1 (break `scheduler/` ↔ `distributed/` cycle; restate layered DAG)

**Anchor:** §3.2 Dependency Graph — dependency-rules table row for `distributed/`; and a restating callout immediately above the D6 "Invariant" statement.

**Rationale:** Normative cycle removal (ADR-008 reinforcement). `scheduler/` MUST NOT `#include` any header from `distributed/`; remote notifications surface as `SchedulerEvent`s authored by `distributed/` and consumed via `core/` types.

### Diff (dependency-rules table)

```diff
-| `distributed/` | `core/`, `transport/` | `scheduler/` (for cross-node coordination), `runtime/` |
+| `distributed/` | `core/`, `transport/` | `runtime/` |
```

### Diff (layered-DAG callout)

```diff
+> [UPDATED: A7-P1: Restate strictly descending layered DAG and record cycle removal.]
+> **Layered DAG invariant (D6, ADR-008):** Module edges descend strictly `bindings > runtime > scheduler > distributed > transport > hal > core`; `profiling/` and `error/` are cross-cutting leaves. The previously documented `scheduler/ → distributed/` back-edge is removed: `scheduler/` MUST NOT `#include` any header from `distributed/`. Remote notifications surface as `SchedulerEvent`s produced by `distributed/` and consumed by `scheduler/` via `core/` types only. See `A7-P5` (distributed linkage) and `modules/distributed.md §1.3`.
+
 **Invariant (Rule D6):** No circular dependencies exist. The graph is verified at build time.
```

---

## Cross-references updated

- ADR-008 enforcement, ADR-014 (ADR-A7-R3) promotion triggers, `10-known-deviations.md`, `09-open-questions.md` Q5/Q11/Q12.
- Companion edits in `04-process-view.md §4.6.3` (A7-P5) and `modules/distributed.md §1.3` (A7-P1 / A7-P5 primary owners).

## Not applied here (owner lives elsewhere)

The deviation-entry text for A2-P8 is authored in `10-known-deviations.md` per its primary-owner diff. This file only cross-references the canonical location.
