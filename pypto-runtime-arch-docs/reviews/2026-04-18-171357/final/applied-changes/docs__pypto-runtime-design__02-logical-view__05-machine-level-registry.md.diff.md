# Edit: docs/pypto-runtime-design/02-logical-view/05-machine-level-registry.md

- **Driven by proposal(s):** A2-P2, A9-P2, A9-P3, A9-P7
- **ADR link(s):** ADR-016 (closed v1 policy enums); ADR-021 (extension_map opaque payload)
- **Timestamp (UTC):** 2026-04-19
- **Edit kind:** modification (surgical callout insertions + table row update)

## Location

- Path: `docs/pypto-runtime-design/02-logical-view/05-machine-level-registry.md`
- Sections touched: §2.2.1 (Machine Level table), §2.2.3 (IHorizontalChannel)
- Line range before edit: 21 (table row), 149 (post-interface)

## Before

- `LevelParams` row listed `source_collection_config` as a separate field; `DeploymentMode` / `PolicyKind` were described as open factories.
- `IHorizontalChannel` interface exposed `barrier()` and `all_reduce()` as peer methods.
- `MachineLevelDescriptor` had no extension payload mechanism.

## After

### A9-P7 / A2-P2 / A9-P2 — `LevelParams` table row

- **Before row:** `level_params | Level-specific parameters (…, source_collection_config, event_handling_config, deployment_factory, policy_factory, …)`
- **After row:** `level_params | Level-specific parameters (…, event_handling_config — **with SourceCollectionConfig folded in (A9-P7)**; `LevelOverrides` remains closed for v1 (v2 schema-registered in Q6); deployment and policy pluggability collapse to closed enums `DeploymentMode` / `PolicyKind` (A9-P2).`

### A9-P3 / A2-P2 — IHorizontalChannel callout

- **After (callout):**

> [UPDATED: A9-P3: remove speculative collectives from IHorizontalChannel] `barrier()` and `all_reduce()` are removed from `IHorizontalChannel`. Collective semantics belong in the Orchestration level; the horizontal channel remains a point-to-point + broadcast transport.
>
> [UPDATED: A2-P2: MachineLevelDescriptor.extension_map] `MachineLevelDescriptor` exposes an **opaque** `std::unordered_map<std::string, std::any> extension_map` for forward-compatible per-level payloads. v1 variants are closed and discoverable at compile time; `extension_map` is the only open axis and is schema-validated at deployment-parser init (unknown keys → WARN, never REJECT).

## Per-proposal edits

### A9-P7 — fold SourceCollectionConfig

Row updated above; the unified `EventHandlingConfig` shape is documented in `09-interfaces.md §2.6.4`.

### A2-P2 — open extension_map + closed v1 variants

Covered by the IHorizontalChannel callout (second paragraph).

### A9-P2 — closed `DeploymentMode`/`PolicyKind` enums

Table row note; closed enums are also stated in `02-scheduler.md §2.1.3.5` end-of-section callout and in `09-interfaces.md §2.6.6`.

### A9-P3 — drop speculative collective ops

IHorizontalChannel callout (first paragraph). `IHorizontalChannel` is now a narrow P2P+broadcast interface.

## Rationale

A major theme of the review is surface minimization: closed enums + extension_map replace speculative polymorphism with a single, versioned, opt-in forward-compat axis. Removing `barrier()` / `all_reduce()` from the horizontal channel avoids encoding collectives into the transport seam and keeps a future orchestration-level interface free to choose its own semantics.

## Verification steps

1. `grep -n "\[UPDATED: A" docs/pypto-runtime-design/02-logical-view/05-machine-level-registry.md` returns 2 hits.
2. `09-interfaces.md §2.6.4` (EventHandlingConfig) should show the folded SourceCollectionConfig shape.
3. `IHorizontalChannel` in `09-interfaces.md` (or `modules/transport.md`) must not declare `barrier()` / `all_reduce()`.

## Line-count delta

- Before: 163 lines (pre-callouts)
- After: 174 lines (+11 lines)
