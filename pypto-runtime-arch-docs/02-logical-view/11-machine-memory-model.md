# 2.9 Machine Model and Memory Model Abstraction

> Part of the [Logical View](../02-logical-view.md). This section formalizes contracts that span the entire Abstract Machine: what the **machine model** hides/exposes and the invariants it maintains, and what visibility, consistency, and ordering guarantees the **memory model** provides. Individual components are specified in §2.1–§2.5; this document defines the cross-cutting rules that tie them together.

## 2.9.1 Machine Model Abstraction

The Abstract Machine ([§2.1.1](01-domain-model.md#211-the-abstract-machine)) defines a platform-independent computational model. This subsection specifies what the abstraction hides, what it exposes, and what invariants it maintains.

### Abstraction Boundaries

The machine model establishes three abstraction boundaries:

| Boundary | What Is Hidden | What Is Exposed |
|----------|---------------|-----------------|
| **Platform boundary** | ISA details, register layouts, DMA engine specifics, chip topology (die count, core arrangement), CANN runtime API | `IPlatformFactory` → `IDevice`, `IMemory`, `IExecutionEngine`, `IRegisterBank` (HAL interfaces, [§3.1](../03-development-view.md)) |
| **Level boundary** | Transport mechanism (shared memory, DMA, RDMA, TCP), addressing scheme (pointers vs. handles), synchronization primitives (atomics, polling, interrupts) | Six per-level component interfaces: `ISchedulerLayer`, Workers, `IMemoryManager`, `IMemoryOps`, `IVerticalChannel`, `IHorizontalChannel` ([§2.2.1](05-machine-level-registry.md#221-machine-level)) |
| **Variant boundary** | Whether execution is on real hardware (ONBOARD) or simulated (SIM) | Identical interface behavior — same `ISchedulerLayer`, same `IMemoryOps`, same `IVerticalChannel` contracts regardless of variant |

### Abstraction Invariants

The following invariants hold regardless of platform, deployment size, or variant:

1. **Recursive Uniformity.** Every Layer in the stack has the same structural composition: `(Scheduler, Workers[], MemoryScope)`. A Worker at layer N interacts with the runtime identically whether N is 0 (Core) or 7 (Global). No layer has special-case logic that breaks the uniform pattern.

2. **Interface Completeness.** Each Machine Level is fully described by its six component factories ([§2.2.1](05-machine-level-registry.md#221-machine-level)). Given a valid set of factories, the framework can assemble a functioning Layer without additional per-level framework code.

3. **Topology Independence.** The framework contains zero hardcoded references to level names (`"Core"`, `"Chip"`, `"Host"`, etc.) or level counts. All topology knowledge resides in the Machine Level Registry configuration.

4. **Platform Transparency.** Code above the HAL boundary (schedulers, workers, memory managers) depends only on abstract interfaces. Switching from a2a3 to a5 requires changing factory bindings in the registry — no scheduler, memory manager, or communication logic changes.

5. **Variant Transparency.** Switching between ONBOARD and SIM requires changing the HAL implementation (real DMA vs. `memcpy`, real registers vs. host variables). All code above the HAL is identical between variants.

### Hardware-to-Abstract Mapping

The machine model maps physical hardware to abstract concepts:

| Physical Hardware Concept | Abstract Machine Concept | Parameterized By |
|--------------------------|-------------------------|------------------|
| AICore compute pipeline (Cube/Vector/Scalar) | Worker at leaf Machine Level | `worker_type_name`, `LevelParams` |
| Core-local SRAM (L1, L0A, L0B, L0C) | Memory Scope at leaf level | `ScratchpadMemoryManager` |
| AICPU thread | Worker at intermediate Machine Level | `worker_thread_count` |
| Device HBM | Memory Scope at Device level | `RingBufferMemoryManager` |
| Host DRAM | Memory Scope at Host level | `PoolMemoryManager` |
| Register Bank (DATA_MAIN_BASE, COND) | Vertical Channel (Chip → Core) | `IVerticalChannelFactory` |
| DMA engine | Memory Operations + Vertical Channel (Host ↔ Device) | `IMemoryOpsFactory`, `IVerticalChannelFactory` |
| RDMA NIC | Horizontal Channel + Memory Operations (Pod level) | `IHorizontalChannelFactory`, `IMemoryOpsFactory` |
| TPUSH/TPOP hardware flags | Horizontal Channel (Core ↔ Core within CoreGroup) | `IHorizontalChannelFactory` |
| PCIe link | Encapsulated in DMA-based `IVerticalChannel` | HAL `IMemory` |
| InfiniBand / RoCE link | Encapsulated in RDMA-based `IHorizontalChannel` | HAL `INetwork` |

The mapping is unidirectional: abstract concepts do not leak platform details upward. New hardware (e.g., a future accelerator with a different memory hierarchy) is supported by implementing the six component interfaces for new Machine Levels — no framework changes required.

## 2.9.2 Memory Model Abstraction

The memory model defines how data is addressed, how writes propagate across the Layer Stack, and what ordering guarantees the runtime provides. The model is **hierarchical** — each level has its own memory semantics, and the overall system behavior emerges from the composition of per-level rules.

### Hierarchical Address Space

The memory model is built on a hierarchy of Address Spaces aligned with the Layer Stack:

```
Global Address Space
├── Level N Address Space (root)
│   ├── Instance 0
│   │   └── Local addresses
│   └── Instance 1
│       └── Local addresses
├── Level N-1 Address Space
│   ├── Instance 0.0 (child of N.0)
│   │   └── Local addresses
│   └── Instance 0.1 (child of N.0)
│       └── Local addresses
│   ...
└── Level 0 Address Space (leaf)
    └── Instance 0.0.0...
        └── Local addresses
```

Each Memory Scope ([§2.1.5](04-memory.md#215-memory-scope)) defines a local Address Space. A **Global Address** `(level_path[], local_address)` uniquely identifies any location in the system. Address translation between levels is provided by `IMemoryOps.to_global()` / `from_global()`.

**Key property:** Local addresses are opaque across levels. A local address in the `"Core"` Memory Scope cannot be dereferenced by a Worker at the `"Host"` level. All cross-level access requires explicit data movement via `IMemoryOps`.

**Memory Regions are orthogonal to the Address Space.** A Memory Scope may be internally partitioned into multiple **Memory Regions** ([§2.1.5.1](04-memory.md#2151-memory-region)) — e.g. a chip scope into `CONTROL_SCRATCH` on on-chip SRAM plus `DATA_HBM` on device HBM. This partitioning refines only the *physical placement* and *cache characteristics* of allocations; it does **not** change any rule in this section. The visibility, consistency, and ordering guarantees defined below apply uniformly to every Region of a Scope; a Worker that can see the Scope can address every Region in it. Placement decisions (which Region to allocate from, whether to isolate a cache line) are described separately in [§2.1.8 Access-Efficiency Considerations](04-memory.md#218-access-efficiency-considerations) and do not affect the memory model.

### Memory Visibility Rules

Memory visibility is strictly hierarchical, with three access tiers:

| Access Tier | Scope | Mechanism | Latency |
|-------------|-------|-----------|---------|
| **Direct** | All Workers within the same Layer instance | Shared Memory Scope — no transfer needed | Register/SRAM access time |
| **Adjacent** | Workers in parent or child Layer (one hop) | `IMemoryOps.copy_to_child()` / `copy_from_child()` | Transport-dependent (DMA, on-chip copy, load/store) |
| **Remote** | Workers in non-adjacent Layers or sibling instances at distributed levels | Multi-hop through intermediate layers, or `IMemoryOps.copy_to_peer()` for same-level peers | Cumulative hop latency |

No Worker can access another Layer's Memory Scope without explicit data movement. This enforces data locality and makes all data flow explicit in the task DAG — there are no hidden side channels.

### Memory Consistency Model

The runtime provides different consistency guarantees depending on the communication boundary:

| Boundary | Consistency Model | Guarantee |
|----------|------------------|-----------|
| **Intra-Layer** (Workers within same Layer) | Sequential consistency within task DAG | If Task A produces data D and Task B depends on A, then B observes D. The Scheduler's dependency tracking enforces this — no reordering of dependent task execution. |
| **Vertical** (Parent ↔ Child Layer) | Task-completion consistency | Data written by a child Task is visible to the parent upon receiving `notify_child_complete`. Data staged by `copy_to_child()` is visible to the child upon Task dispatch. |
| **Horizontal intra-node** (Siblings on same physical node) | Hardware-dependent, abstracted by channel | Intra-chip: hardware coherence via shared memory + atomics. Cross-device: DMA completion guarantees visibility. The `IHorizontalChannel` and `IMemoryOps` implementations encapsulate the underlying coherence protocol. |
| **Horizontal inter-node** (Siblings on different physical nodes) | Causal consistency per task chain ([§2.6.2](09-interfaces.md#262-distributed-scheduler)) | If Task A completes before Task B is submitted and B depends on A, then B observes A's outputs. Cross-chain ordering is not guaranteed. Enforced by `REMOTE_DEP_NOTIFY`. |

### Ordering Rules

1. **Dependency ordering.** If Task B declares a dependency on Task A, then B is guaranteed to observe all of A's memory effects. This holds at every level and across levels.

2. **Scope ordering.** All tasks within a scope must complete before the scope-exit token is applied. A scope boundary acts as a memory fence for all tasks in that scope.

3. **Vertical ordering.** Data staged via `copy_to_child()` is visible before the child Task begins executing. Data produced by a child Task is visible to the parent after `notify_child_complete` is delivered.

4. **No implicit ordering.** Tasks without explicit dependencies or scope boundaries may execute in any order and may observe each other's memory effects in any order. The runtime does not provide total ordering across independent task chains.

### Memory Lifetime Model

The memory model defines when buffers may be reclaimed. Reclamation uses a two-condition rule (detailed in [§2.4.5](07-task-model.md#245-tensor-lifecycle)):

```
Buffer reclaimable ⟺ (scope_token_applied ∨ explicit_free_called) ∧ (ref_count == fanout_count)
```

**Safety guarantee:** No buffer is reclaimed while any declared consumer Task is still executing. This holds regardless of whether `free()` is called, whether the scope has exited, or whether the consumer is on a different level or a different node.

**Deferred completion:** For asynchronous hardware operations (DMA, RDMA), the `complete_in_future` mechanism ([§2.4.5](07-task-model.md#245-tensor-lifecycle)) ensures that buffers backing in-flight transfers are not reclaimed until hardware completion is confirmed.

### Memory Model Per Level Category

Different categories of Machine Levels operate under different underlying physical memory models. The abstract interfaces (`IMemoryManager`, `IMemoryOps`) hide these differences, presenting a uniform API regardless of the underlying hardware:

| Level Category | Physical Memory | Address Model | Coherence | Typical Levels |
|---------------|----------------|---------------|-----------|----------------|
| **On-chip** | SRAM (L1, L0A/B/C), shared memory | Direct pointers in shared address space | Hardware coherence — shared GM visible to all on-chip entities | `"Core"`, `"CoreGroup"`, `"Chip"` |
| **On-device** | HBM (Global Memory) | Device-local addresses; not host-accessible | DMA completion guarantees visibility; no hardware coherence with host | `"Device"` |
| **Host** | DRAM | Virtual addresses in host process | OS-provided coherence; standard CPU memory model | `"Host"` |
| **Distributed** | RDMA-registered buffers, remote memory | Opaque handles; explicit transfer required | Message-passing semantics; RDMA completion guarantees visibility | `"Pod"`, `"Supernode"`, `"Global"` |

**Simulation variant:** In SIM mode, all levels use host DRAM with `memcpy` for data movement. The abstract interfaces behave identically, but coherence is trivially provided by shared host-process memory. This allows testing of multi-level memory flow without hardware.

### Memory Model Diagram

```
┌──────────────────────────────────────────────────────────────────────┐
│ Level Category     │ Memory        │ Consistency       │ Coherence  │
├────────────────────┼───────────────┼───────────────────┼────────────┤
│ Distributed        │ RDMA buffers  │ Causal per chain  │ Message    │
│ (Pod, Supernode)   │ (opaque hdl)  │                   │ passing    │
├──────── Horizontal Channel (RDMA / TCP) ────────────────────────────┤
│ Host               │ Host DRAM     │ Task-DAG          │ OS/CPU     │
│                    │ (virt addr)   │ sequential        │ coherence  │
├──────── Vertical Channel (DMA + MMIO) ──────────────────────────────┤
│ On-device          │ HBM           │ Task-DAG          │ DMA        │
│ (Device)           │ (device addr) │ sequential        │ completion │
├──────── Vertical Channel (shared memory + polling) ─────────────────┤
│ On-chip            │ Shared SRAM   │ Task-DAG          │ Hardware   │
│ (Chip, CoreGroup)  │ (direct ptr)  │ sequential        │ shared GM  │
├──────── Vertical Channel (Register Bank ACK/FIN) ───────────────────┤
│ Leaf               │ Core SRAM     │ N/A (leaf)        │ Local      │
│ (Core)             │ (L1/L0)       │                   │            │
└──────────────────────────────────────────────────────────────────────┘
```

The layers above each Vertical Channel boundary cannot directly access the memory below it. All cross-boundary data flow is explicit, mediated by `IMemoryOps`, and ordered by the task dependency DAG.
