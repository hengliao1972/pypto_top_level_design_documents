# 3. Development View

This view describes how the Simpler runtime is organized in source code — modules, packages, build units, and their dependencies.

---

## 3.1 Module Structure

The runtime decomposes into ten modules, each with a single clear responsibility:

```
simpler/
├── hal/                    # Hardware Abstraction Layer
│   ├── include/            # Public interfaces: IDevice, IMemory, IExecutionEngine,
│   │                       #   IRegisterBank, INetwork, IPlatformFactory
│   ├── src/                # Internal utilities
│   ├── a2a3/               # a2a3 platform implementation
│   │   ├── onboard/        # Real hardware (CANN runtime)
│   │   └── sim/            # Software simulation
│   ├── a5/                 # a5 platform implementation
│   │   ├── onboard/
│   │   └── sim/
│   └── tests/
│
├── core/                   # Core abstractions
│   ├── include/            # Task struct, TaskHandle, TaskKey, TaskState,
│   │                       #   Worker struct, WorkerState,
│   │                       #   ISchedulerLayer, FunctionRegistry, TaskSlotPool,
│   │                       #   StateMachine<TaskState>, StateMachine<WorkerState>,
│   │                       #   parent-child tracking
│   ├── src/
│   └── tests/
│
├── scheduler/              # Per-level scheduler implementations
│   ├── include/            # ITaskSchedulePolicy, IWorkerSelectionPolicy,
│   │                       #   IResourceAllocationPolicy, TaskManager,
│   │                       #   WorkerManager, ResourceManager
│   ├── src/
│   │   ├── task_manager.cpp           # Task state machine driver, dependency DAG
│   │   ├── worker_manager.cpp         # Worker state machine driver, pool management
│   │   ├── resource_manager.cpp       # Resource coordination (IMemoryManager + WorkerManager)
│   │   ├── policies/                  # Default and custom policy implementations
│   │   │   ├── fifo_task_policy.cpp
│   │   │   ├── priority_task_policy.cpp
│   │   │   ├── round_robin_worker_policy.cpp
│   │   │   ├── locality_worker_policy.cpp
│   │   │   └── greedy_resource_policy.cpp
│   │   └── levels/                    # Per-level ISchedulerLayer implementations
│   │       ├── aicore_dispatcher.cpp      # L0: AICore register-poll dispatch
│   │       ├── aicpu_scheduler.cpp        # L1: AICPU scheduler
│   │       ├── device_orchestrator.cpp    # L2: Device-level orchestration
│   │       ├── host_scheduler.cpp         # L3: Host-level scheduling
│   │       └── distributed_scheduler.cpp  # L4+: Distributed scheduling
│   └── tests/
│
├── memory/                 # Memory management
│   ├── include/            # TensorMap, address translation, IMemoryManager impls,
│   │                       #   distributed address space, remote memory registration
│   ├── src/
│   │   ├── ring_buffer_memory_manager.cpp
│   │   ├── pool_memory_manager.cpp
│   │   ├── rdma_registered_memory_manager.cpp
│   │   └── scratchpad_memory_manager.cpp
│   └── tests/
│
├── transport/              # Communication transports
│   ├── include/
│   ├── src/
│   │   ├── intra_node/     # Ring buffer protocol, shared memory mailbox,
│   │   │                   #   register-based signaling
│   │   └── inter_node/     # Message framing, serialization, reliable delivery
│   └── tests/
│
├── distributed/            # Distributed scheduling
│   ├── include/            # Distributed scheduling protocol, message types,
│   │                       #   remote task proxy, data movement coordinator,
│   │                       #   IPartitioner, node membership
│   ├── src/
│   └── tests/
│
├── profiling/              # Profiling, tracing, and logging
│   ├── include/            # Profiling levels, trace event model, collectors,
│   │                       #   logging framework, distributed trace correlation
│   ├── src/
│   └── tests/
│
├── error/                  # Error handling
│   ├── include/            # Error taxonomy, error codes, ErrorContext,
│   │                       #   propagation paths, Python exception mapping
│   ├── src/
│   └── tests/
│
├── runtime/                # Runtime composition and lifecycle
│   ├── include/            # Runtime class, DistributedRuntime, layer assembly,
│   │                       #   MachineLevelRegistry, DeploymentConfig
│   ├── src/
│   └── tests/
│
└── bindings/               # Python bindings
    ├── include/            # Nanobind module structure, exposed classes
    ├── src/
    └── tests/
```

### Module Responsibilities

| Module | Responsibility | Key Interfaces Owned |
|--------|---------------|---------------------|
| `hal/` | Abstract platform-specific hardware behind uniform interfaces. Lowest module — everything above depends only on HAL interfaces. | `IDevice`, `IMemory`, `IExecutionEngine`, `IRegisterBank`, `INetwork`, `IPlatformFactory` |
| `core/` | Define the fundamental data structures and contracts: Task, Worker, state machines, function registry, scheduler layer interface. | `ISchedulerLayer`, `Task`, `TaskKey`, `Worker`, `WorkerState`, `FunctionRegistry`, `StateMachine<TaskState>`, `StateMachine<WorkerState>` |
| `scheduler/` | Implement `ISchedulerLayer` per Machine Level via TaskManager, WorkerManager, and ResourceManager sub-components. Provide schedule policy interfaces and default implementations. `TaskManager` also owns the `producer_index` lifecycle (insert on admission, evict on `SUBMISSION_RETIRED`) that realizes the runtime half of the [Dependency Model §2.10](02-logical-view/12-dependency-model.md). No separate dependency-analyzer module exists at this level — frontend dep analysis (per [`tensor-dependency.md`](../../tensor-dependency.md)) lives above `bindings/`. | `TaskManager`, `WorkerManager`, `ResourceManager`, `ITaskSchedulePolicy`, `IWorkerSelectionPolicy`, `IResourceAllocationPolicy`, per-level scheduler implementations |
| `memory/` | Manage memory allocation, address translation, and tensor maps per level. | `IMemoryManager` implementations, `IMemoryOps` implementations, `TensorMap` |
| `transport/` | Implement `IVerticalChannel` and `IHorizontalChannel` for both intra-node and inter-node communication. | `IChannel` (unified), ring buffer, register bank, DMA control, network message transports |
| `distributed/` | Orchestrate cross-node scheduling: message protocol, remote task proxies, partition strategies, data movement coordination. | `IPartitioner`, distributed message types, remote task proxy |
| `profiling/` | Collect, correlate, and export profiling, tracing, and logging data across all layers and nodes. | Profiling levels, trace event model, collectors, logging framework |
| `error/` | Define error taxonomy, error codes, propagation paths, and Python exception mappings. | `ErrorContext`, error code system, exception mapping |
| `runtime/` | Compose all modules into a runnable runtime. Wire layers, manage lifecycle, handle deployment configuration. | `Runtime`, `DistributedRuntime`, `MachineLevelRegistry`, `DeploymentConfig` |
| `bindings/` | Expose runtime to Python via nanobind. Handle GIL, type conversion, error translation. | Python `simpler.Runtime`, `simpler.Task`, `simpler.DistributedRuntime` |

> [UPDATED: A2-P8: Cross-ref ADR-014 freeze of `runtime::composition` sub-namespace and record intentional closures.]
> The `runtime/` module hosts a closed sub-namespace `runtime::composition` (`MachineLevelRegistry`, `MachineLevelDescriptor`, `DeploymentConfig`, `deployment_parser`). Compose-order and sub-namespace boundary are **frozen for v1** per **ADR-014 (ADR-A7-R3)** with documented promotion triggers (cross-consumer list ≥ 2 or deployment-schema churn > N/quarter). Registry being frozen at init, synchronous-only policies, and the closed `DepMode` enum are recorded as intentional closures in [`10-known-deviations.md`](10-known-deviations.md) (cross-linked from `09-open-questions.md` Q5/Q11/Q12).

---

## 3.1.1 Module Decomposition Summary

### Module Inventory

| Module | Responsibility | Key Interfaces Owned | Design Status |
|--------|---------------|---------------------|---------------|
| `hal/` | Abstract platform-specific hardware behind uniform interfaces | `IDevice`, `IMemory`, `IExecutionEngine`, `IRegisterBank`, `INetwork`, `IPlatformFactory` | Draft — [modules/hal.md](modules/hal.md) |
| `core/` | Define fundamental data structures and contracts | `ISchedulerLayer`, `Task`, `TaskKey`, `Worker`, `WorkerState`, `FunctionRegistry`, `StateMachine<TaskState>`, `StateMachine<WorkerState>` | Draft — [modules/core.md](modules/core.md) |
| `scheduler/` | Implement `ISchedulerLayer` via TaskManager, WorkerManager, ResourceManager; provide policy interfaces and defaults | `TaskManager`, `WorkerManager`, `ResourceManager`, `ITaskSchedulePolicy`, `IWorkerSelectionPolicy`, `IResourceAllocationPolicy` | Draft — [modules/scheduler.md](modules/scheduler.md) |
| `memory/` | Manage memory allocation, address translation, and tensor maps | `IMemoryManager` impls, `IMemoryOps` impls, `TensorMap` | Draft — [modules/memory.md](modules/memory.md) |
| `transport/` | Implement Vertical and Horizontal Channels | `IVerticalChannel` impls, `IHorizontalChannel` impls | Draft — [modules/transport.md](modules/transport.md) |
| `distributed/` | Orchestrate cross-node scheduling | `IPartitioner`, distributed message types, remote task proxy | Draft — [modules/distributed.md](modules/distributed.md) |
| `profiling/` | Collect, correlate, and export profiling/tracing/logging data | Profiling levels, trace event model, collectors | Draft — [modules/profiling.md](modules/profiling.md) |
| `error/` | Define error taxonomy, codes, propagation, exception mapping | `ErrorContext`, error code system | Draft — [modules/error.md](modules/error.md) |
| `runtime/` | Compose modules into a runnable runtime; manage lifecycle | `Runtime`, `DistributedRuntime`, `MachineLevelRegistry` | Draft — [modules/runtime.md](modules/runtime.md) |
| `bindings/` | Expose runtime to Python via nanobind | `simpler.Runtime`, `simpler.Task`, `simpler.DistributedRuntime` | Draft — [modules/bindings.md](modules/bindings.md) |

### Module Design Priority

Modules should be designed in dependency order — leaf modules first:

| Priority | Module | Rationale |
|----------|--------|-----------|
| 1 | `error/` | Foundation — no dependencies; used by all other modules |
| 2 | `hal/` | Hardware abstraction boundary; all platform-specific code depends on this |
| 3 | `core/` | Core data structures (`Task`, `ISchedulerLayer`) used by all higher modules |
| 4 | `profiling/` | Cross-cutting; depends only on `hal/` |
| 5 | `memory/` | Memory management; depends on `hal/`, `core/` |
| 6 | `transport/` | Communication channels; depends on `hal/`, `core/` |
| 7 | `scheduler/` | Scheduler implementations; depends on `core/`, `memory/`, `transport/` |
| 8 | `distributed/` | Distributed scheduling; depends on `core/`, `transport/` |
| 9 | `runtime/` | Composition layer; depends on all above |
| 10 | `bindings/` | Top-level consumer; depends on `runtime/` |

### Module Detailed Design Documents

| Module | Document | Status |
|--------|----------|--------|
| `hal/` | [modules/hal.md](modules/hal.md) | Draft |
| `core/` | [modules/core.md](modules/core.md) | Draft |
| `scheduler/` | [modules/scheduler.md](modules/scheduler.md) | Draft |
| `memory/` | [modules/memory.md](modules/memory.md) | Draft |
| `transport/` | [modules/transport.md](modules/transport.md) | Draft |
| `distributed/` | [modules/distributed.md](modules/distributed.md) | Draft |
| `profiling/` | [modules/profiling.md](modules/profiling.md) | Draft |
| `error/` | [modules/error.md](modules/error.md) | Draft |
| `runtime/` | [modules/runtime.md](modules/runtime.md) | Draft |
| `bindings/` | [modules/bindings.md](modules/bindings.md) | Draft |

---

## 3.2 Dependency Graph

The module dependency graph is a strict DAG. Dependencies flow upward from `hal/` (lowest) to `bindings/` (highest):

```
                        bindings/
                           │
                        runtime/
                      ┌────┼────────────────────┐
                      │    │                    │
                 scheduler/ │              distributed/
                   │    │   │                │    │
                   │    │   │                │    │
                   │  memory/  transport/────┘    │
                   │    │         │               │
                   └────┼─────────┘               │
                        │                         │
                      core/ ──────────────────────┘
                        │
                      hal/
                        │
                   profiling/ ◄─── (cross-cutting: used by all modules above hal/)
                   error/     ◄─── (cross-cutting: used by all modules above hal/)
```

**Dependency rules:**

| Module | Depends On | Depended On By |
|--------|-----------|----------------|
| `hal/` | (none) | `core/`, `memory/`, `transport/`, `scheduler/` |
| `core/` | `hal/` | `scheduler/`, `memory/`, `transport/`, `distributed/`, `runtime/` |
| `transport/` | `hal/`, `core/` | `scheduler/`, `distributed/`, `runtime/` |
| `memory/` | `hal/`, `core/` | `scheduler/`, `distributed/`, `runtime/` |
| `scheduler/` | `core/`, `memory/`, `transport/` | `runtime/` |
| `distributed/` | `core/`, `transport/` | `runtime/` |
| `profiling/` | `hal/` (minimal) | All modules (cross-cutting) |
| `error/` | (none) | All modules (cross-cutting) |
| `runtime/` | `hal/`, `core/`, `scheduler/`, `memory/`, `transport/`, `distributed/`, `profiling/`, `error/` | `bindings/` |
| `bindings/` | `runtime/` | (none — top-level consumer) |

> [UPDATED: A7-P1: Restate strictly descending layered DAG and record cycle removal.]
> **Layered DAG invariant (D6, ADR-008):** Module edges descend strictly `bindings > runtime > scheduler > distributed > transport > hal > core`; `profiling/` and `error/` are cross-cutting leaves. The previously documented `scheduler/ → distributed/` back-edge is removed: `scheduler/` MUST NOT `#include` any header from `distributed/`. Remote notifications surface as `SchedulerEvent`s produced by `distributed/` and consumed by `scheduler/` via `core/` types only. See `A7-P5` (distributed linkage) and `modules/distributed.md §1.3`.

**Invariant (Rule D6):** No circular dependencies exist. The graph is verified at build time.

---

## 3.3 Build Targets

### 3.3.1 Build Target Matrix

| Module | Library Type | Host-Only | Cross-Compile | Optional |
|--------|-------------|-----------|---------------|----------|
| `hal/` | Compiled (per platform) | No | Yes (onboard) | No |
| `core/` | Compiled | Yes | No | No |
| `scheduler/` | Compiled | Partial (host + device schedulers) | Yes (device schedulers) | No |
| `memory/` | Compiled | Partial | Yes (device memory managers) | No |
| `transport/` | Compiled | Partial | Yes (intra-node device transports) | Network backends optional |
| `distributed/` | Compiled | Yes | No | Yes (only for multi-node) |
| `profiling/` | Header-only + compiled | Both | Yes (device-side collectors) | No |
| `error/` | Header-only + compiled | Both | Yes (device-side error codes) | No |
| `runtime/` | Compiled | Yes | No | No |
| `bindings/` | Compiled (nanobind module) | Yes | No | No |

### 3.3.2 Platform Build Matrix

| Target | Platform | Variant | Modules Compiled |
|--------|----------|---------|-----------------|
| `a2a3` | a2a3 | ONBOARD | All (cross-compile for device modules) |
| `a2a3sim` | a2a3 | SIM | All (host-compiled, simulation providers) |
| `a5` | a5 | ONBOARD | All (cross-compile for device modules) |
| `a5sim` | a5 | SIM | All (host-compiled, simulation providers) |

### 3.3.3 CMake Structure

```
CMakeLists.txt                    # Top-level: project, options, platform detection
├── hal/CMakeLists.txt            # HAL interfaces + platform implementations
├── core/CMakeLists.txt           # Core abstractions (Task, FSM, ISchedulerLayer)
├── scheduler/CMakeLists.txt      # Scheduler implementations
├── memory/CMakeLists.txt         # Memory management implementations
├── transport/CMakeLists.txt      # Transport implementations
├── distributed/CMakeLists.txt    # Distributed scheduling (optional component)
├── profiling/CMakeLists.txt      # Profiling/tracing/logging
├── error/CMakeLists.txt          # Error handling
├── runtime/CMakeLists.txt        # Runtime composition
├── bindings/CMakeLists.txt       # Python bindings (nanobind)
├── cmake/
│   ├── toolchain-a2a3.cmake      # Cross-compilation toolchain for a2a3
│   ├── toolchain-a5.cmake        # Cross-compilation toolchain for a5
│   └── platform.cmake            # Platform detection and configuration
└── tests/CMakeLists.txt          # Test infrastructure
```

Each platform provides a `platform.cmake` defining toolchains, flags, and HAL implementation sources. The runtime build discovers and includes the platform plugin.

### 3.3.4 Network Backend Build

Network backends are optional build components selected at configure time or runtime:

| Backend | Library | Condition |
|---------|---------|-----------|
| RDMA (ibverbs/HCCL) | `transport_rdma` | `-DENABLE_RDMA=ON` |
| TCP sockets | `transport_tcp` | Always available |
| Shared-memory loopback | `transport_shm` | Always available (default for simulation) |

### 3.3.5 Python Package Build

The Python package is built via scikit-build-core or CMake-based wheel:
- nanobind module compilation.
- Platform-specific wheel variants.
- Optional distributed extra: `pip install simpler[distributed]`.

### 3.3.6 Test Infrastructure

| Test Level | Scope | Infrastructure |
|-----------|-------|----------------|
| Unit tests | Per module | CTest, each module's `tests/` directory |
| Integration tests | Per platform | Platform-specific test fixtures |
| System tests | End-to-end | Full runtime with real or simulated hardware |
| Distributed tests | Multi-node | Multi-process test harness simulating multi-node |

---

## 3.4 Interface Headers / API Surface

Each module exposes its public interface through its `include/` directory. Internal implementation details are in `src/` and not accessible to other modules.

| Module | Public Header(s) | Key Exports |
|--------|------------------|-------------|
| `hal/` | `hal/i_device.h`, `hal/i_memory.h`, `hal/i_execution_engine.h`, `hal/i_register_bank.h`, `hal/i_network.h`, `hal/i_platform_factory.h` | Platform-agnostic hardware interfaces |
| `core/` | `core/task.h`, `core/worker.h`, `core/function.h`, `core/i_scheduler_layer.h`, `core/state_machine.h`, `core/task_slot_pool.h` | Core data structures (Task, Worker, states) and scheduler contract |
| `scheduler/` | `scheduler/task_manager.h`, `scheduler/worker_manager.h`, `scheduler/resource_manager.h`, `scheduler/i_task_schedule_policy.h`, `scheduler/i_worker_selection_policy.h`, `scheduler/i_resource_allocation_policy.h`, `scheduler/levels/*.h` | Scheduler sub-components, policy interfaces, per-level implementations |
| `memory/` | `memory/i_memory_manager.h`, `memory/i_memory_ops.h`, `memory/tensor_map.h` | Memory management interfaces |
| `transport/` | `transport/i_vertical_channel.h`, `transport/i_horizontal_channel.h` | Channel interfaces and implementations |
| `distributed/` | `distributed/i_partitioner.h`, `distributed/distributed_protocol.h` | Distributed scheduling protocol |
| `runtime/` | `runtime/runtime.h`, `runtime/machine_level_registry.h`, `runtime/deployment_config.h` | Runtime composition and configuration |
| `bindings/` | (Python module, not C++ headers) | `simpler.Runtime`, `simpler.Task`, etc. |

### Inter-Module Contracts

For each module boundary:

- **Thread safety**: Documented per interface method.
- **Ownership**: Clear ownership transfer semantics (move, shared_ptr, raw pointer with lifetime guarantee).
- **Lifetime**: Components created by factories are owned by the Layer instance that requested them.
- **Distributed contracts**: Message ordering is per-channel FIFO. Delivery is guaranteed or reported as failure. Timeout handling is mandatory on all remote operations (Rule R1).
