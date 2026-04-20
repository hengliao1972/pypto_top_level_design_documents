# 1. Introduction

## 1.1 Purpose

This document set describes the architecture of the redesigned **Simpler runtime** — the execution engine that schedules work, moves data, and manages hardware resources for the PTO compiler stack on Ascend NPU hardware.

The redesign transforms the current monolithic runtime into a cleanly layered, modular architecture. The target audience is runtime architects, implementation engineers, and AI agents tasked with generating code from these specifications.

## 1.2 Scope

### In Scope

- **Hardware Abstraction Layer (HAL)**: Decoupling runtime logic from platform specifics (a2a3, a5, simulation).
- **Recursive Hierarchical Execution Model**: A stack of scheduling layers where upper layers submit tasks to lower layers, with no fixed depth limit.
- **Distributed Scheduling**: Cross-node scheduling message exchange and data movement as a first-class concern.
- **Runtime Interface and Python Bindings**: Clean API surface for C++ and Python consumers.
- **Profiling, Tracing, and Logging**: Unified cross-cutting observability.
- **Error Handling**: Consistent error detection, propagation, and recovery across host, AICPU, and AICore.
- **Task and State Machine Model**: Formal lifecycle with pluggable hooks.
- **Abstract Machine Model**: Hierarchical schedulers, workers, and memories.
- **Build System**: Modular composition of platform × runtime × variant.

### Out of Scope

- PTO compiler front-end and IR design.
- Kernel (AICore Function) implementation details and ISA specification.
- External data services implementation (distributed shared memory, block storage, DFS, key-value DB) — only integration points are defined.
- Specific ML model architectures or operator implementations.

## 1.3 Requirements Summary

### Functional Requirements

| ID | Requirement |
|----|-------------|
| FR-1 | Support hierarchical task scheduling across an extensible N-layer architecture (not a fixed 3-level tree). |
| FR-2 | Support distributed multi-node execution with cross-node task submission and data movement. |
| FR-3 | Provide a uniform `ISchedulerLayer` contract across all layers (submit, schedule, dispatch, complete). |
| FR-4 | Support multiple function types: AICore (leaf compute), Orchestration (control flow), Host (Python/C++), Distributed Orchestration (cross-node). |
| FR-5 | Support SPMD task dispatch (same function, different data, parallel execution). |
| FR-6 | Provide tensor lifecycle management with implicit scope and explicit `free()`. |
| FR-7 | Support function caching via content hash ("register once, invoke many"). |
| FR-8 | Expose a stable C API and Python bindings for runtime control. |
| FR-9 | Support multi-tenancy via isolated Logical System namespaces. |
| FR-10 | Support a CPU-based **Simulation** facility that executes the same runtime logic over a specified Platform without real hardware, offering three Simulation Modes: (a) **Performance** — leaf (AICore) Functions are not actually computed; modeled execution time is recorded and a simulated performance trace is emitted; (b) **Functional** — AICore Functions are executed on host CPU cores and produce real compute results; (c) **Replay** — runtime execution is reconstructed from profiling and trace data captured on real hardware, yielding a fully detailed runtime timeline. All three modes share the same `ISchedulerLayer`, Task model, and HAL contracts as ONBOARD execution (NFR-3). |

### Non-Functional Requirements

| ID | Requirement |
|----|-------------|
| NFR-1 | **Performance**: Zero-overhead profiling when disabled (compile-time and runtime switches). Minimize data movement on critical paths. |
| NFR-2 | **Extensibility**: Adding a new hardware level requires only implementing six component interfaces and registering — no existing code modified (OCP). |
| NFR-3 | **Portability**: Same runtime logic runs on real hardware (ONBOARD) and software simulation (SIM) via HAL abstraction. |
| NFR-4 | **Modularity**: Clean module boundaries with DAG dependencies. Each module independently testable. |
| NFR-5 | **Backward compatibility**: Existing DSL constructs (`pl.incore`, `pl.auto_incore`, `FunctionType.*`) map to the unified grammar. |
| NFR-6 | **Reliability**: Partial failure handling in distributed mode. Fatal vs. recoverable error classification. |
| NFR-7 | **Observability**: Distributed trace correlation with global timestamps. Chrome Trace / Perfetto compatible event model. |

## 1.4 Key Architectural Principles

Three principles govern the redesign:

1. **Hierarchical Recursion**: The runtime is a stack of layers. Each layer has a scheduler, workers, and memory. A task executing at layer N can submit child tasks to the layer N scheduler, which dispatches them to layer N−1 workers. This pattern is uniform from distributed host down to AICore dispatch.

2. **Distributed-First**: Multi-node / multi-device scheduling is not an afterthought but a first-class layer in the hierarchy. Scheduling messages and data movement between nodes use the same transport abstractions as intra-device communication.

3. **Uniform Layer Interface**: Every layer implements the same `ISchedulerLayer` contract, differing only in what "worker" and "memory" mean at that level.

## 1.5 Assumptions

- [ASSUMPTION] The target platforms are Ascend NPU hardware (a2a3, a5 families) with CANN runtime support for onboard execution.
- [ASSUMPTION] Simulation mode uses host threads and host memory to model device behavior, with the same abstract interfaces.
- [ASSUMPTION] Python 3.8+ is the minimum supported Python version for bindings (nanobind).
- [ASSUMPTION] C++17 is the implementation language for all runtime modules.
- [ASSUMPTION] The PTO compiler generates hierarchy labels (`pl.Level` tags) on all outlined functions, which the runtime resolves at dispatch time.
- [ASSUMPTION] External data services (distributed shared memory, block storage, DFS, key-value DB) are deployed and managed independently of the runtime.

## 1.6 Relationship to Other Documents

- **[Architecture Design Guide](../../.cursor/guides/architecture-design-guide/00-index.md)**: The methodology this document follows.
- **[Architecture Redesign (source)](../architecture-redesign/)**: The original stage-based working documents from which this restructured version is derived.
- **[Known Deviations](10-known-deviations.md)**: Rule violations with justification (per [04-agent-rules.md](../../.cursor/guides/architecture-design-guide/04-agent-rules.md)).
