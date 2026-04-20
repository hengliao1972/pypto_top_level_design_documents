# Appendix B — Current Codebase Mapping

This table maps current Simpler runtime components to the formal terms and modules defined in the redesigned architecture. It serves as a refactoring roadmap.

> [UPDATED: A2-P4: cross-link to Migration & Transition Plan] For the phased retirement schedule of the current-implementation classes listed below (feature-flag, canary, rollback gate), see [`03-development-view.md §3.5 Migration & Transition Plan`](03-development-view.md#35-migration--transition-plan).

> [UPDATED: 2026-04-19-P11.3: clean-room bring-up landed] The **Target Module** column of the tables below refers to planned modules. The `pypto-runtime/` clean-room implementation has now landed all ten target modules through Phase P10 (`error/`, `hal/`, `core/`, `profiling/`, `memory/`, `transport/`, `scheduler/`, `distributed/` skeleton, `runtime/`, `bindings/`). For per-module evidence, public-surface summaries, and residual deferrals see the Phase-1 through Phase-10 checkpoints in `pypto-runtime/docs/plans/2026-04-19-pypto-runtime-master.md`. Items listed as "No equivalent" in the **Current Implementation** column (e.g. `ISchedulerLayer`, `IHorizontalChannel`, Machine Level Registry, Function Cache) are now concrete in the clean-room tree; the retirement of the listed **Current Implementation** in `src/` remains out of scope of this bring-up (Phase-1 Q7 locked the bring-up to clean-room).

---

## Component Mapping

| Formal Term | Current Implementation | Current Location | Target Module |
|-------------|----------------------|-----------------|---------------|
| **Machine Level `"Host"`** | `DistWorker` + `DistScheduler` | `src/common/distributed/` | `scheduler/host_scheduler`, `runtime/` |
| **Machine Level `"Device"`** | `AicpuExecutor` orchestration path | `src/a2a3/runtime/.../aicpu/` | `scheduler/device_orchestrator` |
| **Machine Level `"Chip"`** | `AicpuExecutor` scheduler path | `src/a2a3/runtime/.../aicpu/` | `scheduler/aicpu_scheduler` |
| **Machine Level `"Core"`** | `aicore_execute()` register poll loop | `src/a2a3/runtime/.../aicore/` | `scheduler/aicore_dispatcher` |
| **Machine Level Registry** | No equivalent — topology hardcoded | Spread across `platform_config.h`, `aicpu_executor.cpp`, `dist_scheduler.h` | `runtime/machine_level_registry` |
| **ISchedulerLayer** | No unified interface | Ad-hoc per-level APIs | `core/i_scheduler_layer.h` |

## Memory Subsystem Mapping

| Formal Term | Current Implementation | Current Location | Target Module |
|-------------|----------------------|-----------------|---------------|
| **IMemoryManager** | `PTO2TensorMap` (ring buffer), `DistTensorMap` (pool) | `pto_runtime2_types.h`, `dist_tensormap.h` | `memory/` |
| **IMemoryOps** | Hardcoded DMA calls, `memcpy`, core load/store | Spread across runtime and platform code | `memory/` |
| **Memory Scope (`"Host"`)** | Host DRAM, `MemoryAllocator` | `memory_allocator.h` | `memory/pool_memory_manager` |
| **Memory Scope (`"Device"`)** | HBM via `DeviceRunner` | `device_runner.h` | `memory/ring_buffer_memory_manager` |

## Communication Mapping

| Formal Term | Current Implementation | Current Location | Target Module |
|-------------|----------------------|-----------------|---------------|
| **IVerticalChannel** | Ad-hoc register writes + ring buffer + DMA | Spread across platform code | `transport/` |
| **IHorizontalChannel** | No equivalent (TPUSH/TPOP in ISA only) | — | `transport/` (to be created) |
| **Vertical Channel (Device→Chip)** | `DistRing`, PTO2 task window (Ring Buffer) | `dist_ring.h`, `pto_runtime2_types.h` | `transport/intra_node/ring_buffer` |
| **Vertical Channel (Chip→Core)** | `platform_regs.h` register functions (Register Bank) | `src/a2a3/platform/include/` | `transport/intra_node/register_bank` |
| **Distributed payload structs** (`RemoteSubmitPayload`, `HeartbeatPayload`, `HandshakePayload`, …) | Inlined in distributed/transport glue; no clean ownership split | `src/common/distributed/` | `distributed/include/distributed/protocol_payloads.h` (owner = `distributed/`); `transport/messages.h` keeps only `MessageHeader`, framing, `MessageType` tags. Invariant I-DIST-1 (IWYU-CI) forbids `transport/` from including `distributed/` headers. [UPDATED: A7-P4: move distributed payloads out of `transport/`; ADR-015] |

## Task Subsystem Mapping

| Formal Term | Current Implementation | Current Location | Target Module |
|-------------|----------------------|-----------------|---------------|
| **Task (Host)** | `DistTaskSlotState` | `dist_types.h` | `core/task.h` |
| **Task (Device)** | `PTO2TaskSlotState` | `pto_runtime2_types.h` | `core/task.h` |
| **TaskState (Host)** | `TaskState` enum (6 states: FREE, PENDING, READY, RUNNING, COMPLETED, CONSUMED) | `dist_types.h` | `core/task.h` (10 lifecycle states + `ERROR`; `CANCELLED` if A3-P1b adopted; see [`modules/core.md §2.3`](modules/core.md) and ADR-016) [UPDATED: A4-P8: sync TaskState count + new states] |
| **TaskState (Device)** | `PTO2TaskState` enum (5 states: PENDING, READY, RUNNING, COMPLETED, CONSUMED) | `pto_runtime2_types.h` | `core/task.h` (10 lifecycle states + `ERROR`; `CANCELLED` if A3-P1b adopted; see [`modules/core.md §2.3`](modules/core.md) and ADR-016) [UPDATED: A4-P8: sync TaskState count + new states] |
| **Tensor Lifecycle (`free()`)** | `task_freed` flag, `ref_count`/`fanout_count` protocol | `pto_runtime2_types.h` (device) | `memory/`, `core/` |
| **Tensor Lifecycle (scope)** | Scope-exit token via `fanout_count` initial +1 | `pto_runtime2_types.h`, `dist_types.h` | `memory/`, `core/` |

## Function Subsystem Mapping

| Formal Term | Current Implementation | Current Location | Target Module |
|-------------|----------------------|-----------------|---------------|
| **Function** | `Callable<>` template | `callable.h` | `core/function.h` |
| **AICore Function** | `CoreCallable` — `binary_data()`, `binary_size_`, `resolved_addr_` | `callable.h` | `core/function.h` |
| **Orchestration Function** | `ChipCallable` + device `.so` via `dlopen` | `callable.h`, `aicpu_executor.cpp` | `core/function.h` |
| **Function Cache** | No equivalent — re-loaded each init | — | `core/function_cache` (to be created) |

## Worker Mapping

| Formal Term | Current Implementation | Current Location | Target Module |
|-------------|----------------------|-----------------|---------------|
| **Worker (`"Host"`)** | `DistChipProcess` / `ChipWorker` | `src/common/distributed/`, `src/common/worker/` | `scheduler/host_scheduler` |
| **Worker (`"Chip"`)** | AICore via register dispatch | `platform_regs.h`, `aicpu_executor.cpp` | `scheduler/aicpu_scheduler` |
| **`WorkerState`** | Ad-hoc flags across `DistWorker` / `ChipWorker` state (no unified enum); recovery states implicit in `fault_handler` paths | `src/common/worker/`, `src/common/distributed/` | `core/worker.h` — canonical FSM `{READY, BUSY, DRAINING, RETIRED, FAILED, UNAVAILABLE{Permanent, Quarantine(duration)}}`; see [`02 §2.1.4.1`](02-logical-view/03-worker.md#2141-worker-state-machine), A5-P9 DRY-fold, A5-P11 unified peer-health FSM (ADR-018) [UPDATED: A4-R3-P1: WorkerState mapping] |

## Platform and API Mapping

| Formal Term | Current Implementation | Current Location | Target Module |
|-------------|----------------------|-----------------|---------------|
| **Platform** | `platform_config.h` constants | `src/{a2a3,a5}/platform/include/` | `hal/` |
| **Platform Variant** | onboard vs sim directory split | `src/{a2a3,a5}/platform/{onboard,sim}/` | `hal/{a2a3,a5}/{onboard,sim}/` |
| **C API boundary** | `pto_runtime_c_api.h` | `src/common/worker/` | `runtime/`, `bindings/` |
| **Profiling** | `PerformanceCollector` (host/aicpu/aicore variants) | `platform/include/{host,aicpu,aicore}/` | `profiling/` |

## Features Without Current Equivalent

| Formal Term | Notes | Target Module |
|-------------|-------|---------------|
| **Deferred Completion** | Planned via `complete_in_future` — see `runtime_async.md` | `core/`, `scheduler/` |
| **Valid Shape** | `ContinuousTensor` lacks `valid_shapes` — see `tensor_valid_shape.md` | `core/` |
| **SPMD Task** | `call_spmd` / `CALL_TASK` broadcast exist but not formalized | `core/`, `scheduler/` |
| **Distributed Data Access Model** | `LinquOrchestratorState` handle-based tracking in `linqu_runtime_design.md` | `distributed/`, `memory/` |
| **External Data Services** | No equivalent — see `linqu_data_system.md` | Integration points in `memory/`, `runtime/` |
| **Multi-Tenancy** | No equivalent — see `linqu_runtime_design.md` §4.3 | `runtime/` |
| **`pl.Level` (DSL aliases)** | `pl.Level.*` enum in `machine_hierarchy_and_function_hierarchy.md` | `core/`, `runtime/` |
