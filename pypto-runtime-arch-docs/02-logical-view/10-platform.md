# 2.8 Platform

> Part of the [Logical View](../02-logical-view.md). This module defines how the Abstract Machine is parameterized by a concrete hardware configuration (Platform) and an execution environment (Variant), together identifying a build target.

A **Platform** parameterizes the Abstract Machine with concrete hardware values:

| Parameter | a2a3 | a5 |
|-----------|------|----|
| `max_blockdim` | 24 | 36 |
| `cores_per_blockdim` | 3 | 3 |
| `max_aicpu_threads` | 4 | 7 |
| `max_cores` | 72 | 108 |
| `num_dies` | 1 | 2 |

A **Platform Variant** distinguishes the execution environment:

| Variant | Description |
|---------|-------------|
| `ONBOARD` | Real hardware (cross-compiled, CANN runtime) |
| `SIM` | Software simulation (host-compiled, threads + host memory) |

The combination `(Platform, Variant)` identifies a build target: `a2a3`, `a2a3sim`, `a5`, `a5sim`.

## 2.8.1 Simulation Modes

When the Variant is `SIM`, the runtime runs entirely on host CPU. To serve different verification and analysis goals, a `SIM` build is further parameterized by a **Simulation Mode**. The mode is selected at runtime initialization (it does not require a separate build target) and is scoped to the leaf-level (AICore) execution behavior; all higher Layers behave identically regardless of mode.

| Mode | Leaf Function Execution | Compute Result | Output | Input |
|------|-------------------------|----------------|--------|-------|
| `PERFORMANCE` | Not executed. The HAL's leaf `IExecutionEngine` consults a per-Function **timing model** and advances a simulated clock; no compute is performed. | None (outputs unchanged / undefined) | Chrome Trace / Perfetto-compatible simulated performance trace (§7.2) | Timing model per Function (latency, resource usage) |
| `FUNCTIONAL` | Executed on host CPU cores via a CPU backend of the leaf Function. | Bit-accurate with the mathematical specification of the AICore Function; may diverge from hardware in numerical precision unless the backend models it. | Real output tensors; optional wall-clock trace | CPU-compiled AICore Function bodies |
| `REPLAY` | Not executed. The leaf `IExecutionEngine` reads previously captured per-Task events from a **trace source** and advances state machines accordingly. | None (outputs taken from, or omitted by, the captured trace) | Reconstructed runtime trace identical in structure to a live `PERFORMANCE` trace, annotated with the captured timestamps and metadata | Profiling/trace artifact captured from an `ONBOARD` run (§7.2) |

**Design invariants:**

1. Simulation Mode is a property of the **leaf Machine Level implementation** only. The `ISchedulerLayer` contract, Task state machine, Memory Scope, Vertical/Horizontal Channels, and Worker model are identical to the `ONBOARD` variant. No runtime logic above the leaf `IExecutionEngine` distinguishes simulation from onboard execution.
2. Simulation Mode is orthogonal to the Platform (`a2a3`, `a5`, ...). The same Simulation Mode implementation works across Platforms; Platform-specific numbers (`max_blockdim`, `cores_per_blockdim`, ...) continue to parameterize the abstract machine.
3. `PERFORMANCE` and `REPLAY` modes produce traces in the same event format as live profiling output (§7.2.2) so that downstream analysis tooling is shared.
4. `FUNCTIONAL` mode may be combined with profiling to obtain an approximate timing trace for host execution, but such traces are explicitly labeled and are not a substitute for `PERFORMANCE` or `REPLAY` output.

**Configuration:** Simulation Mode is supplied via `LevelParams` for the leaf Machine Level; the Machine Level Registry resolves the appropriate `IExecutionEngine` factory from the registered leaf-level factories based on the selected mode. This follows the same pluggable-factory pattern used for other Machine Level components (ADR-002).

> [UPDATED: A9-P8: platform abstraction simplification — closed enums + registry]
> `Platform` and `Variant` are closed enums with no runtime discovery API in v1. The permissible build targets are the fixed matrix `{a2a3, a2a3sim, a5, a5sim}`; new platforms are added by committing a new entry to the Machine Level Registry table in `05-machine-level-registry.md` plus a build-target bump (appendix-c-compatibility.md). `SimulationMode` is likewise a closed enum `{PERFORMANCE, FUNCTIONAL, REPLAY}`. No `IPlatform` / `IVariant` polymorphic interface is introduced in v1. Platform-dependent behavior is encapsulated in the leaf `IExecutionEngine` factory selected by `MachineLevelDescriptor.variant_tag`; the selection is `O(1)` via a static table lookup in the Registry (no dynamic discovery). Adding a new platform or variant is therefore a **registry + build-target** change rather than a new interface. Trigger for promotion to a polymorphic interface: a third platform with non-tabular parameters; recorded as Q14 in `09-open-questions.md`.
