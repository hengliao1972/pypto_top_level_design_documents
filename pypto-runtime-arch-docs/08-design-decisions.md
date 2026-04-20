# 8. Design Decisions

This section documents key architectural decisions as Architecture Decision Records (ADRs). Each ADR follows the template from `.cursor/guides/architecture-design-guide/05-output-templates.md`.

---

## ADR-001: Recursive Hierarchical Execution Model

### Status

Accepted

### Context

The current Simpler runtime has a hardcoded 3-level architecture (Host → Device/AICPU → AICore) with ad-hoc scheduling at each level. Adding distributed scheduling or new hardware levels requires invasive changes across multiple files. The runtime needs to support an extensible hierarchy from single-chip to multi-cluster deployments.

### Decision

Adopt a **recursive hierarchical execution model** where the runtime is structured as a stack of Execution Layers. Each Layer has a Scheduler, Workers, and Memory Scope. A Worker at layer N may submit child tasks to the layer-N Scheduler, which dispatches them to layer-(N−1) Workers. This pattern is uniform at every level.

### Alternatives Considered

| Alternative | Pros | Cons | Why Rejected |
|-------------|------|------|--------------|
| Fixed 3-level architecture (current) | Simple, well-understood | Cannot extend without invasive changes; distributed is bolted on | Does not satisfy extensibility requirements |
| Flat task graph (single scheduler) | Simple scheduling logic | Cannot model hierarchical hardware; poor data locality | Poor fit for hierarchical NPU architecture |
| Microkernel with message-passing only | Maximum decoupling | High overhead for intra-chip communication; poor latency | Too expensive for nanosecond-scale dispatch |

### Consequences

**Positive:**
- Adding new hardware levels requires only implementing six interfaces and registering — no existing code modified (OCP, Rule E4).
- The same scheduling logic works from single-chip to multi-cluster.
- Hierarchical task trees naturally model orchestration patterns.

**Negative:**
- Higher conceptual complexity — developers must understand the recursive layer model.
- Potential for deep task trees that complicate debugging.

---

## ADR-002: Machine Level Registry with Pluggable Factories

### Status

Accepted

### Context

Different deployments require different numbers of hierarchy levels and different implementations at each level. Hardcoding topology in the framework couples runtime logic to specific hardware configurations.

### Decision

Introduce a **Machine Level Registry** where each level is a named, registered type with six pluggable component factories (Scheduler, Worker, Memory Manager, Memory Ops, Vertical Channel, Horizontal Channel). The topology is configuration, not code.

### Alternatives Considered

| Alternative | Pros | Cons | Why Rejected |
|-------------|------|------|--------------|
| Enum-based level selection (compile-time) | Fast dispatch; simple | Adding levels requires code changes; cannot mix configurations | Violates OCP |
| Dynamic plugin loading (dlopen) | Maximum flexibility | Complex lifecycle; hard to debug; ABI compatibility issues | Over-engineered for current needs |
| Template-based level composition | Zero runtime overhead | Requires compile-time knowledge of all levels; combinatorial explosion | Not practical for deployment-time configuration |

### Consequences

**Positive:**
- Deployment configuration fully describes topology — no code changes for new hardware.
- Level elision enables the same binary to run single-node or multi-cluster.
- Clean separation between framework and platform-specific implementations.

**Negative:**
- Virtual dispatch overhead at factory creation time (acceptable — creation is not a hot path).
- Registry must be frozen before use, preventing hot-reconfiguration.

---

## ADR-003: Uniform ISchedulerLayer Interface

### Status

Accepted

### Context

The current runtime has three different scheduling implementations (`DistScheduler`, `PTO2Runtime`, `AicpuExecutor`) with incompatible APIs. Code that interacts with scheduling must know which level it's talking to.

### Decision

Define a single **ISchedulerLayer** interface implemented by all levels. The interface includes: `submit()`, `submit_group()`, `submit_spmd()`, `notify_dep_satisfied()`, `notify_child_complete()`, `scope_begin/end()`, `drain()`, `shutdown()`. Component wiring is done via setters for Memory Manager, Memory Ops, and Channels.

### Alternatives Considered

| Alternative | Pros | Cons | Why Rejected |
|-------------|------|------|--------------|
| Per-level custom APIs (current) | Each API tailored to its level | No code reuse; cross-level logic must know all APIs | Violates DIP (Rule D2); high coupling |
| Minimal base + rich per-level extensions | Base provides commonality; extensions provide specificity | Consumers must still know specific types for advanced features | Partial solution; still couples consumers |

### Consequences

**Positive:**
- Runtime composition code (layer assembly) is generic — works with any ISchedulerLayer.
- Testing: mock scheduler implementations for unit testing at any level.
- Consistent behavior contracts across all levels.

**Negative:**
- Some interface methods may be no-ops at certain levels (e.g., `submit_spmd` at Core level). Mitigated by documenting which methods are meaningful per level.

---

## ADR-004: Separate Control and Data Paths

### Status

Accepted

### Context

Intra-chip communication and inter-node communication have very different performance characteristics. Mixing control messages (task submissions, completions) with bulk data transfers on the same path causes head-of-line blocking and prevents independent optimization.

### Decision

Separate **control path** (Vertical Channel + Horizontal Channel) from **data path** (IMemoryOps). Channels carry typed control messages; Memory Ops handle bulk data transfers. Each can be optimized independently (e.g., low-latency signaling channel for control, high-bandwidth DMA/RDMA for data).

### Alternatives Considered

| Alternative | Pros | Cons | Why Rejected |
|-------------|------|------|--------------|
| Unified channel (control + data on same path) | Simpler interface — one abstraction per boundary | Head-of-line blocking: large data transfers delay control messages; cannot independently optimize latency vs. throughput | Unacceptable latency impact on nanosecond-scale control paths (Rule X3) |
| Three separate paths (control, small data, bulk data) | Maximum optimization per path | Three interfaces per boundary; excessive complexity for marginal gain over two-path | Violates KISS (Rule G2); two-path provides sufficient separation |

### Consequences

**Positive:**
- Control messages are not blocked by large data transfers.
- Each path can use the optimal transport for its workload.
- Clean API separation makes each easier to implement and test.

**Negative:**
- Two separate interfaces to implement per level boundary (channel + memory ops).
- Coordination between control and data paths requires careful ordering (data must be transferred before consumer task is notified).

---

## ADR-005: Distributed-First Design

### Status

Accepted

### Context

The current runtime treats distributed execution as a separate, bolted-on concern. The scheduling logic for multi-device (within a node) and multi-node are fundamentally different code paths, leading to duplication and inconsistency.

### Decision

Treat distributed scheduling as a **first-class layer** in the hierarchy, using the same `ISchedulerLayer` interface and the same Vertical/Horizontal Channel abstractions. Multi-node is just another level with Workers that represent nodes, Memory that is RDMA-registered, and Channels that use network transport.

### Alternatives Considered

| Alternative | Pros | Cons | Why Rejected |
|-------------|------|------|--------------|
| Distributed as a separate subsystem (current approach) | Independent evolution of distributed and local code | Duplicated scheduling logic; inconsistent task model between local and distributed; separate testing | Maintenance burden; violates DRY; inconsistent contracts |
| Distributed as an external orchestrator (e.g., Kubernetes operator) | Leverage existing orchestration tooling; separation of concerns | Runtime has no control over scheduling granularity; cannot co-optimize with intra-node scheduling; high latency for fine-grained tasks | Too coarse-grained for sub-millisecond task dispatch |
| Distributed-first as a layer (chosen) | Uniform interfaces; same task model everywhere; testable with local transport | Distributed concerns (serialization, failure) add complexity to channel implementations | Accepted — complexity is contained in distributed-level implementations |

### Consequences

**Positive:**
- No special-case code for distributed mode — it's just another level.
- Same task model, state machine, and dependency tracking work across all levels.
- Testing can simulate distributed mode with shared-memory transport.

**Negative:**
- Distributed concerns (network failure, partial failure, message serialization) add complexity to the Channel and Memory Ops implementations at distributed levels.
- Causal consistency model requires careful implementation of `REMOTE_DEP_NOTIFY`.

---

## ADR-006: Tensor Lifecycle with Implicit Scope and Explicit Free

### Status

Accepted

### Context

Tensor buffer management must balance two goals: (1) safety — buffers must not be reclaimed while in use, and (2) efficiency — buffers should be reclaimed as early as possible to reduce memory pressure. The current runtime uses scope-based lifetime with no way to free buffers early.

### Decision

Use a **two-condition reclamation rule**: a tensor buffer is reclaimable when (1) its scope token has been applied (via scope exit OR explicit `free()`), AND (2) all declared consumers have completed. Explicit `free()` applies the scope token immediately, enabling earlier reclamation without bypassing safety.

### Alternatives Considered

| Alternative | Pros | Cons | Why Rejected |
|-------------|------|------|--------------|
| Scope-only lifetime (current) | Simple | Buffers held until scope exit even if no longer needed | Memory waste in long scopes |
| Reference counting only | Early reclamation | Cannot protect against undeclared future consumers within a scope | Safety violation |
| Manual free only | Maximum control | Error-prone; use-after-free risk | Unsafe for generated code |

### Consequences

**Positive:**
- Complementary mechanisms: `free()` reduces lifetime within a scope; ring stack eliminates cross-scope blocking.
- Safe: fanout counter prevents premature reclamation even with explicit `free()`.
- DSL-visible: `pl.free(tensor)` gives users control over memory pressure.

**Negative:**
- Two-condition rule is more complex to understand and debug than either mechanism alone.
- Deferred completion adds another dimension (hardware completion must also be tracked).

---

## ADR-007: Function Caching via Content Hash

### Status

Accepted

### Context

In the current runtime, functions are re-loaded on each device initialization. For distributed deployments, transferring function binaries to every node on every invocation wastes bandwidth.

### Decision

Cache registered Functions per Machine Level Instance, keyed by **content hash** of the binary. Subsequent tasks referencing the same hash reuse the cached function. For distributed levels, binaries are sent to remote nodes only if their cache lacks the hash.

### Consequences

**Positive:**
- "Register once, invoke many" reduces binary transfer overhead.
- Content-hash keying enables cache sharing across different function names with identical code.
- Lazy loading (request-on-miss) reduces startup latency for large function sets.

**Negative:**
- Hash collisions (extremely unlikely with cryptographic hashes) could cause silent correctness bugs. Mitigated by using SHA-256 or equivalent.
- Cache management (eviction, invalidation) adds operational complexity.

---

## ADR-008: Scheduler Internal Decomposition with Strategy-Pattern Policy Hooks

### Status

Accepted

### Context

The `ISchedulerLayer` implementations are monolithic — each per-level scheduler handles task lifecycle, worker dispatch, resource allocation, and dependency tracking in a single class. This has three problems:

1. **No separation of concerns within the scheduler.** Task state management, worker lifecycle, and resource coordination are interleaved, making each implementation hard to understand and test in isolation (violates SRP, Rule D3).
2. **No formal Worker state machine.** Workers are treated as opaque execution slots. There is no structured lifecycle tracking, making failure detection and recovery ad-hoc per level.
3. **No customizable scheduling policies.** Scheduling behavior (task ordering, worker selection, resource allocation strategy) is hardcoded per level. Adapting scheduling behavior for different workloads or deployment scenarios requires modifying scheduler implementations.

### Decision

Decompose each `ISchedulerLayer` implementation into three internal sub-components — **TaskManager**, **WorkerManager**, and **ResourceManager** — and introduce three **strategy interfaces** (`ITaskSchedulePolicy`, `IWorkerSelectionPolicy`, `IResourceAllocationPolicy`) as hooks at scheduling decision points.

- **TaskManager** owns the task state machine, task slot pool, dependency DAG, and scope management.
- **WorkerManager** owns a new formal Worker state machine, the worker pool, and worker availability tracking.
- **ResourceManager** is a coordinator that queries `IMemoryManager` (from the `memory/` module) and WorkerManager to decide when a task has sufficient resources.
- **Policy interfaces** are C++ abstract classes injected at construction via factory registration in `MachineLevelDescriptor`. Default implementations (FIFO, round-robin, greedy) are provided.

The external `ISchedulerLayer` interface (ADR-003) is **preserved unchanged**. This is an internal decomposition.

### Alternatives Considered

| Alternative | Pros | Cons | Why Rejected |
|-------------|------|------|--------------|
| Monolithic scheduler with configuration flags | Simple; no new abstractions | Scheduling behavior still hardcoded; flag combinatorics grow unmanageable; SRP violation persists | Does not address core concerns |
| Event-driven hook chain (emit events, handlers intercept) | Maximum flexibility; supports chaining | Complex dispatch logic; ordering issues between hooks; harder to reason about correctness | Over-engineered; chain ordering is error-prone on the critical path |
| Full module split (task/, worker/, resource/ as separate modules) | Maximum decoupling | Too many module boundaries; increases cross-module wiring complexity; per-level implementations still need tight coordination | Excessive granularity; coordination overhead outweighs decoupling benefit |

### Consequences

**Positive:**
- Each sub-component has a single responsibility and can be tested in isolation (SRP, Rule D3).
- Worker lifecycle is formally tracked — enables structured failure detection and recovery.
- Scheduling behavior is customizable per Machine Level without modifying scheduler implementations (OCP, Rule E4).
- Default policies ensure backward compatibility — no change for existing configurations.
- Policy interfaces are a natural extension point for future optimizations (priority scheduling, data-locality-aware dispatch, adaptive back-pressure).

**Negative:**
- Internal complexity increases — three collaborating objects per scheduler instead of one.
- Policy interface calls add a virtual dispatch on the hot path (mitigated: policy methods are designed to be lightweight; observation hooks must not block).
- Three additional factory fields in `MachineLevelDescriptor` (optional, nullable — minimal registration burden).

### Related

- [ADR-003: Uniform ISchedulerLayer Interface](#adr-003-uniform-ischedulerlayer-interface) — external interface preserved
- [ADR-002: Machine Level Registry with Pluggable Factories](#adr-002-machine-level-registry-with-pluggable-factories) — policy factories follow the same registration pattern
- [02-logical-view §2.1.3](02-logical-view/02-scheduler.md#213-scheduler) — scheduler sub-component definitions
- [02-logical-view §2.6.3](02-logical-view/09-interfaces.md#263-schedule-policy-interfaces) — policy interface specifications
- [04-process-view.md §4.3.5](04-process-view.md#435-worker-state-machine) — Worker state machine

---

## ADR-009: Heterogeneous Worker Types with Worker Group Affinity

**Status:** Accepted  
**Date:** 2026-04-17  
**Deciders:** Architecture team  

### Context

The Ascend NPU contains **heterogeneous** compute cores — AIC (cube) and AIV (vector) — organized into **core wraps** (1 AIC + 2 AIV each). Tasks may require different combinations of core types: single AIV, single AIC, one AIC + one AIV, or one AIC + two AIV. Multi-core tasks must use cores from the **same** core wrap. Allocating individual cores within a wrap reduces the wrap's availability for larger tasks.

The architecture prior to this decision treated Workers as homogeneous within a Layer. The flat-pool model could not express:
1. Worker type distinctions (AIC vs. AIV)
2. Task requirements for specific worker types and quantities
3. Co-location constraints (same group) for multi-worker tasks
4. Resource exclusion effects (allocating one worker reduces group capacity)

This pattern is not unique to the Core level — similar heterogeneous grouping may appear at Device level (AICPU + AICore), Host level (different accelerator types), or future hardware.

### Decision

Introduce three general-purpose abstractions — **Worker Type** (`WorkerTypeDescriptor`), **Task Execution Type** (`TaskExecType`), and **Worker Group** (`WorkerGroup`) — as first-class entities in the architecture:

1. **`WorkerTypeDescriptor`**: Declares a class of Workers with a name, numeric ID, and capability bitmask. Registered per Machine Level.
2. **`TaskExecType`**: Declares which Worker Types a Task requires (`required_slots`: list of `{worker_type_id, count}`) and whether those Workers must come from the same Worker Group (`requires_worker_group`).
3. **`WorkerGroup`**: A named, non-overlapping partition of physical Workers. The WorkerManager tracks per-group availability and enforces the **Resource Exclusion Rule**: a `requires_worker_group` task can only be scheduled if all its `required_slots` can be satisfied by idle members of a single group.

Each Worker carries a `worker_type_id` and `group_id`. Each Task carries an `exec_type_id`. The `IWorkerSelectionPolicy` interface returns a `WorkerAllocation` (set of worker IDs + group ID) instead of a single worker index.

Homogeneous levels define one Worker Type, no groups, and one TaskExecType — preserving backward compatibility.

### Alternatives Considered

| Alternative | Pros | Cons | Why Rejected |
|-------------|------|------|--------------|
| **A: Flat homogeneous pool (status quo)** | Simple; no new abstractions | Cannot express type constraints, grouping, or resource exclusion; would require ad-hoc workarounds | Fundamentally cannot model the hardware |
| **B: Per-type worker pools without groups** | Supports type matching; simpler than groups | No co-location constraint; multi-worker tasks could get workers from different wraps; no resource exclusion tracking | Insufficient for core wrap topology |
| **C: Worker Types + Worker Groups with affinity (chosen)** | General; captures type matching, co-location, and exclusion; backward compatible | Additional complexity in WorkerManager and policy interfaces | Accepted — necessary complexity |
| **D: Hardware-specific core wrap abstraction** | Directly models the target hardware | Not generalizable; would need redesign for new hardware topologies | Violates portability principles (Rule D4) |

### Consequences

**Positive:**
- Accurately models the Ascend NPU core topology (AIC/AIV, core wraps) and resource exclusion effects.
- Abstractions are **general**: the same model can express AICPU+AICore grouping, multi-accelerator nodes, or future hardware topologies.
- Worker Groups are non-overlapping, simplifying exclusion tracking (no shared-membership conflicts).
- Backward compatible: homogeneous levels are a degenerate case (one type, no groups).
- Enables sophisticated scheduling policies (e.g., prefer groups with maximum remaining capacity, bin-packing across groups).

**Negative:**
- WorkerManager complexity increases: must maintain per-group availability index and enforce exclusion rules.
- `IWorkerSelectionPolicy` interface is more complex (`select_workers` returns a `WorkerAllocation` instead of a single index).
- Three new fields in `MachineLevelDescriptor` (`worker_types`, `task_exec_types`, `worker_groups`).
- Per-group availability tracking adds O(G) bookkeeping per Worker state transition (G = number of groups; 24 for a2a3 — negligible).

### Related

- [ADR-008: Scheduler Internal Decomposition with Strategy-Pattern Policy Hooks](#adr-008-scheduler-internal-decomposition-with-strategy-pattern-policy-hooks) — WorkerManager and policy interfaces extended by this ADR
- [ADR-002: Machine Level Registry with Pluggable Factories](#adr-002-machine-level-registry-with-pluggable-factories) — Worker topology registered via MachineLevelDescriptor
- [02-logical-view §2.1.4.2](02-logical-view/03-worker.md#2142-heterogeneous-worker-types-and-worker-groups) — Worker Type, Worker Group definitions
- [02-logical-view §2.4.8](02-logical-view/07-task-model.md#248-task-execution-types) — Task Execution Type definition
- [04-process-view.md §4.3.5](04-process-view.md#435-worker-state-machine) — Group-aware Worker State Machine

---

## ADR-010: Event-Driven Scheduler with Configurable Delivery and Deployment

**Status:** Accepted  
**Date:** 2026-04-17  
**Deciders:** Architecture team  

### Context

The Scheduler drives Task and Worker state machines through state transitions. Events that trigger these transitions originate from multiple sources:

- **Internal**: the Scheduler itself generates follow-on events (e.g., completing a task produces DEP_SATISFIED events for dependents).
- **External**: Workers report completion, parent-layer Workers submit tasks, Vertical/Horizontal Channels deliver messages, hardware raises completion flags.

The previous design described transition handlers (§4.3.2) but left ambiguous:
1. How external events reach the Scheduler (direct method call? queue? polling?).
2. Whether internal follow-on events are handled inline or deferred.
3. How the Scheduler's logical role maps to physical execution units — particularly on resource-constrained processors (AICPU) where dedicating separate threads to scheduling and execution is wasteful.

Different Machine Levels have fundamentally different event delivery characteristics: the Core level polls hardware registers, the Host level receives thread-signaled completions, and distributed levels receive network messages. A single fixed mechanism cannot serve all levels efficiently.

### Decision

1. **Event-driven event loop**: The Scheduler is structured as a loop that collects events from registered `IEventSource` instances, dispatches them to handlers, and repeats. All state transitions flow through this loop — no external thread directly calls into Scheduler internals.

2. **Configurable event delivery**: Each event source declares a delivery mode:
   - **Queue** (`IEventQueue`): producer enqueues; Scheduler dequeues. Used for thread-to-thread communication.
   - **Poll**: Scheduler calls `IEventSource.poll()` each iteration. Used for hardware register/flag-based sources.

3. **Configurable event handling**: Each event type is configured as **inline** (execute handler immediately) or **deferred** (queue for later batch processing) via `EventHandlingConfig` in `LevelParams`.

4. **Scheduler–Worker deployment flexibility**: An `IExecutionPolicy` determines whether a physical execution unit is dedicated to scheduling, dedicated to task execution, or interleaves both:
   - `DedicatedExecutionPolicy`: Scheduler and Workers on separate execution units (default).
   - `InterleavedExecutionPolicy`: single execution unit alternates between roles.
   - `BatchedExecutionPolicy`: accumulates tasks, executes a batch inline, returns to scheduling.

### Alternatives Considered

| Alternative | Pros | Cons | Why Rejected |
|-------------|------|------|--------------|
| **A: Direct method calls into Scheduler** | Simple; no queue overhead | Thread safety requires locks on every call; external threads can block Scheduler internals; no clear concurrency boundary | Violates single-threaded event loop principle; hard to reason about correctness |
| **B: Fixed queue-only delivery** | Uniform mechanism; simple implementation | Cannot efficiently handle hardware poll-based sources (adds unnecessary interrupt/signal overhead for register-polled cores); loses low-latency polling advantage | Inefficient for Core/Chip levels where polling is the natural hardware interface |
| **C: Fixed poll-only delivery** | Low latency for hardware sources | Inefficient for thread-based sources (busy-wait wastes CPU); doesn't scale to distributed levels with network I/O | Wastes resources at Host/Pod levels |
| **D: Event loop with configurable delivery and handling (chosen)** | Supports both queue and poll per source; inline vs. deferred per event type; deployment flexibility via IExecutionPolicy | More interfaces to implement; configuration complexity | Accepted — necessary flexibility |
| **E: Hardcoded per-level Scheduler implementations** | Each level optimized independently | Duplicated logic; violates DRY; no reuse of common event loop structure | Maintainability cost too high |

### Consequences

**Positive:**
- The Scheduler has a single, well-defined concurrency boundary: the event loop. No external thread directly mutates Scheduler state.
- Event delivery is matched to the hardware's natural signaling mechanism (poll for registers, queue for threads, queue for network I/O).
- Inline vs. deferred handling enables latency optimization on the critical path while allowing batching where beneficial.
- `IExecutionPolicy` enables efficient use of resource-constrained processors (AICPU) by allowing interleaved scheduling and execution without dedicated thread overhead.
- The same `ISchedulerLayer` implementation can operate in Dedicated, Interleaved, or Batched mode — only the registered policy differs.

**Negative:**
- Two new factory fields in `MachineLevelDescriptor` (`event_source_factories`, `execution_policy_factory`).
- `EventHandlingConfig` adds per-event-type configuration surface — potential for misconfiguration.
- Interleaved mode requires careful `IExecutionPolicy` tuning to avoid scheduling starvation (if too many tasks are executed inline) or execution starvation (if the event loop never yields to task execution).
- Poll-based sources consume CPU cycles during idle periods; must be paired with appropriate yield policy.

### Related

- [ADR-003: Uniform ISchedulerLayer Interface](#adr-003-uniform-ischedulerlayer-interface) — the external interface is unchanged; the event loop is an internal implementation concern
- [ADR-008: Scheduler Internal Decomposition with Strategy-Pattern Policy Hooks](#adr-008-scheduler-internal-decomposition-with-strategy-pattern-policy-hooks) — TaskManager, WorkerManager, ResourceManager are invoked by event handlers within the loop
- [02-logical-view §2.1.3.5](02-logical-view/02-scheduler.md#2135-event-driven-execution-model) — event model definition
- [02-logical-view §2.1.3.6](02-logical-view/02-scheduler.md#2136-schedulerworker-deployment-flexibility) — deployment flexibility
- [02-logical-view §2.6.4](02-logical-view/09-interfaces.md#264-event-and-execution-interfaces) — IEventSource, IEventQueue, IExecutionPolicy interfaces
- [04-process-view.md §4.1.4](04-process-view.md#414-scheduler-event-loop) — event loop runtime behavior

---

## ADR-011: Simulation Facility with Three Leaf-Execution Modes

**Status:** Accepted  
**Date:** 2026-04-17  
**Deciders:** Architecture team

### Context

The runtime must be usable for three development and analysis workflows that do not require real Ascend NPU hardware:

1. **Performance modeling** — predict the execution time of an application on a specific hardware Platform before the hardware is available, or compare designs, without running actual compute.
2. **Host-side functional verification** — run the full runtime end-to-end on developer laptops and CI hosts, producing real compute results for correctness testing of orchestration, scheduling, memory flow, and AICore Function bodies.
3. **Post-mortem reconstruction** — given a profiling/trace artifact captured on real hardware, replay the runtime execution offline to analyze scheduling decisions, dependency resolution, memory traffic, and timing in detail.

The existing `SIM` Platform Variant (§2.8) already builds the runtime against a software HAL using host threads and host memory, but it does not distinguish among these three workflows. Ad-hoc forks of the `SIM` build to serve each workflow would duplicate runtime logic, diverge over time, and undermine NFR-3 (Portability).

A further constraint (Rule D4 / OCP, ADR-002): adding simulation behaviors must not require modifying the scheduler, task model, memory, or channel modules. Only the leaf-level execution engine changes among the three workflows.

### Decision

Introduce a **Simulation Mode** sub-parameter of the `SIM` Platform Variant. The Mode selects which implementation of the leaf-level `IExecutionEngine` is instantiated; all other Layers, interfaces, and Machine Level components are unchanged.

Three Modes are defined:

1. **`PERFORMANCE`** — the leaf `IExecutionEngine` consults a per-Function **timing model**, records modeled start/duration events into the profiling collector (§7.2), and returns completion without performing compute. Output tensors are not produced. The emitted trace uses the same event format as live profiling.

2. **`FUNCTIONAL`** — the leaf `IExecutionEngine` invokes a **host CPU backend** of the AICore Function, producing real output tensors. Timing is real host wall-clock timing and is not representative of the target hardware; `FUNCTIONAL` traces, when collected, are explicitly labeled.

3. **`REPLAY`** — the leaf `IExecutionEngine` reads a previously captured trace artifact (produced by a live `ONBOARD` run) and advances Task / Worker state machines in replay order, reconstructing the runtime timeline without performing compute. The reconstructed trace has the same structure as a live `PERFORMANCE` trace.

Mode selection is configuration in `LevelParams` for the leaf Machine Level, resolved through the existing Machine Level Registry factory pattern (ADR-002). The `ISchedulerLayer`, Task state machine, Memory Scope, Vertical/Horizontal Channels, Worker model, and trace event schema are **preserved unchanged** (ADR-003, ADR-010, §7.2).

### Alternatives Considered

| Alternative | Pros | Cons | Why Rejected |
|-------------|------|------|--------------|
| **A: Three separate `SIM` build targets** (`a2a3sim_perf`, `a2a3sim_func`, `a2a3sim_replay`, ...) | Strong static separation; each binary minimal | Combinatorial build-target explosion (Platforms × Modes); duplicated leaf-engine wiring; replay vs. perf have nearly identical code paths | Violates OCP (Rule E4) and build system KISS (Rule G2) |
| **B: Single `SIM` build that internally branches per operation** | One binary, no configuration plumbing | Scatters Mode-specific branches across HAL and scheduler; hard to reason about correctness; violates SRP | Violates Rule D3 |
| **C: External replayer consuming trace files, separate from runtime** | Replayer stays simple | Replayer must re-implement Task state machine, scheduler behavior, dependency resolution — duplicating runtime logic and diverging over time | Duplicates runtime logic; violates DRY |
| **D: Mode as leaf-level `IExecutionEngine` factory selection (chosen)** | Reuses the entire runtime stack above the leaf; one build per `(Platform, Variant)`; Mode is configuration; no change to existing interfaces | Factory registration must enumerate per-Mode implementations; the per-Function timing model and per-Function CPU backend must be registered alongside the Function binary | Accepted — aligns with ADR-002, ADR-003 |
| **E: Inject simulation as an Orchestration-level wrapper** | Keeps leaf engine untouched | Cannot suppress compute without also bypassing the leaf Worker (timing traces become inaccurate); cannot drive state machines from replay input | Cannot meet the performance-accuracy or replay-fidelity goals |

### Consequences

**Positive:**

- The full runtime stack (scheduler, task model, memory, channels, profiling) is exercised in every Mode — simulation and onboard paths are the same code above the leaf engine, preventing drift.
- `PERFORMANCE` and `REPLAY` produce traces in the same format as live profiling, so trace analysis tooling is shared across simulation, replay, and onboard runs (§7.2.2).
- Adding Modes in the future (e.g., **`HYBRID`** — compute on CPU but time from a model) is an additive leaf-engine registration; no existing code changes (OCP, Rule E4).
- `FUNCTIONAL` Mode is a natural vehicle for unit and integration testing on CI hosts without NPU hardware — same runtime logic as production.
- `REPLAY` Mode converts trace artifacts into a debuggable, reproducible runtime execution, dramatically shortening post-mortem analysis loops.

**Negative:**

- Each AICore Function now has up to three companion artifacts: the device binary (for `ONBOARD`), a timing model (for `PERFORMANCE`), and a CPU backend (for `FUNCTIONAL`). Tooling must be extended to produce and register these; this burden is on the compiler / kernel-author tooling, not the runtime.
- `REPLAY` fidelity is bounded by the granularity and completeness of the captured trace. Events not captured cannot be reconstructed. This is an expected property and is documented alongside the profiling levels (§7.2.1); Levels 2–4 are suitable replay inputs.
- `FUNCTIONAL` traces may mislead if consumed as performance data; the Mode label on the trace and the documentation must make this unambiguous.

### Related

- [ADR-002: Machine Level Registry with Pluggable Factories](#adr-002-machine-level-registry-with-pluggable-factories) — Mode selection reuses the factory registration mechanism
- [ADR-003: Uniform ISchedulerLayer Interface](#adr-003-uniform-ischedulerlayer-interface) — simulation changes no scheduler interface
- [ADR-010: Event-Driven Scheduler with Configurable Delivery and Deployment](#adr-010-event-driven-scheduler-with-configurable-delivery-and-deployment) — replay drives the same event loop from recorded events
- [01-introduction.md §1.3 FR-10](01-introduction.md#13-requirements-summary) — simulation functional requirement
- [02-logical-view §2.8.1](02-logical-view/10-platform.md#281-simulation-modes) — Simulation Mode definition
- [07-cross-cutting-concerns.md §7.2](07-cross-cutting-concerns.md#72-observability) — trace event schema shared with simulation and replay

<!-- [UPDATED: A9-P6, A6-P12: new ADR added] -->

### Revision R2 (2026-04-18) — SimulationMode option (iii) + REPLAY deferral

**Status:** Accepted  
**Date:** 2026-04-18  
**Deciders:** Architecture team  
**Driven by:** A9-P6 (defer PERFORMANCE/REPLAY simulation), A6-P12 (signed/schema-validated REPLAY trace)

#### Context

Review round 3 (Aspects A5, A7, A8, A9) raised that implementing all three SimulationMode variants in v1 — as originally decided — exceeds the reviewed scope and creates risk of silent fallback when a REPLAY-only artifact is loaded against an unimplemented engine. Separately, A6-P12 required the REPLAY trace to be signed and schema-validated, which cannot be guaranteed if the REPLAY engine itself is still in flux.

#### Decision (amends original Decision)

Adopt **option (iii)** — v1 ships `FUNCTIONAL` leaf-engine only:

1. `SimulationMode` enum stays **open** (extension-point enum per ADR-017), but v1's Machine Level Registry **factory-registers `FUNCTIONAL` only**. `PERFORMANCE` and `REPLAY` scaffolding interfaces are declared in code but are **not** factory-registered.
2. Constructing a leaf Layer with `mode == PERFORMANCE` or `mode == REPLAY` in v1 rejects explicitly with `ErrorCode::SimulationModeNotImplemented` — **no silent fallback** to `FUNCTIONAL`.
3. The REPLAY **trace envelope + file format is frozen at this revision** (signed, schema-validated; see A6-P12 and A2-P9 for the header shape `{magic, trace_schema_version, platform, mode, capture_node_id, content_hash, detached_signature}`). Envelope compatibility is therefore a v1 promise even though no engine consumes it.
4. REPLAY-engine implementation is **deferred to v2**. Concrete promotion triggers are recorded as **Q17**.

#### Consequences (amendment)

**Positive:**

- v1 scope matches the reviewed surface (FUNCTIONAL is the only leaf-engine build actually hardened).
- Trace artifacts captured during v1 `ONBOARD` runs remain valid inputs to the v2 REPLAY engine because the envelope is frozen now.
- Explicit rejection of unregistered modes prevents the silent-miss failure mode A9-P6 flagged.

**Negative:**

- Tools that emit `PERFORMANCE`/`REPLAY` configurations will fail fast on v1 and must gate on `Runtime::capabilities()` (new capability probe reused from ADR-017's registry lookup).
- Any future divergence from the frozen envelope format requires a major-version bump of `trace_schema_version` and a coordinated engine/capture change.

---

## ADR-012: Submission Model with Dependency Modes, Outstanding Window, and Group Workspace

**Status:** Accepted  
**Date:** 2026-04-18  
**Deciders:** Architecture team

### Context

The original Task Model (§2.4) modeled every `submit()` call as a single Task with an explicitly enumerated `DepList`. As workloads have grown (fused operator graphs, SPMD fan-outs, multi-iteration streaming), three problems have become acute:

1. **Dependency construction cost** — at high submission rates, walking every prior Task to derive producer→consumer edges dominates the admission critical path. Some submissions are known by the caller to be independent; others require a simple barrier; yet others benefit from automatic producer discovery.
2. **Unbounded in-flight work** — without a back-pressure mechanism tied to admission, a fast producer can flood the Scheduler with tasks that the downstream pipeline cannot retire in time, spiking memory and dependency-structure cost.
3. **Group-shaped work is inefficient under a per-task API** — fused op blocks and SPMD kernels produce N tasks that share a known internal DAG. With a per-task API, the Scheduler re-derives external dependencies for every task in the group (instead of only the group's boundary-in tasks) and allocates per-tensor storage for every intermediate tensor (instead of a single arena).

### Decision

Introduce a **Submission** abstraction as the unit of `ISchedulerLayer.submit(...)`, with three orthogonal additions:

1. **Dependency Mode** (`DepMode`) per Submission: `BARRIER` (serialize after all outstanding Submissions), `DATA` (runtime-derived producer→consumer edges on boundary inputs), `NONE` (no external edges — caller-asserted).
2. **Outstanding Submission Window**: per-Layer limit on concurrently in-flight Submissions (Submission count, not task count); admission blocks once the window is full; the default `ITaskSchedulePolicy` biases ready-queue ordering by `submission_id` to favor earliest-first completion so the window drains in order.
3. **Group Submission + Group Workspace**: a Submission may contain a pre-built intra-group DAG; external dependency edges attach only to boundary-in tasks; non-boundary tensors are carved from a single Group Workspace arena that is allocated at admission and freed at retirement.

Concrete interface changes are specified in [§2.6.1.A](02-logical-view/09-interfaces.md#261a-submission-types), [§2.1.6.A](02-logical-view/04-memory.md#216a-group-workspace-memory), and [§2.1.3.1.A / §2.1.3.1.B](02-logical-view/02-scheduler.md#2131a-submission-admission--outstanding-window).

### Alternatives Considered

| Alternative | Pros | Cons | Why Rejected |
|-------------|------|------|--------------|
| **A: Always compute DATA-mode edges from all outstanding tasks** | Always correct; minimal API surface | Worst-case O(outstanding × boundary) on the hot path; no escape hatch for known-independent streams | Penalizes the workloads that would benefit most (iteration-level data loaders) |
| **B: Per-task outstanding-window** (cap the number of outstanding tasks rather than Submissions) | Simple accounting | Group Submissions with many tasks get throttled even when the pipeline has capacity at the Submission level; fan-out vs. per-task back-pressure become tangled | Hurts large Group Submissions disproportionately |
| **C: No group abstraction; let callers emit N `submit()` calls with explicit deps** | No new types | Re-derives external edges N times; fragments allocation; serializes admission on per-task lock | Misses the key optimization enabled by caller-known internal DAG |
| **D: Use workspace only as a caller-managed external API** (caller allocates + passes buffers) | Simplest memory-manager change | Pushes lifetime management to the caller; no opportunity for the runtime to free as a unit on retirement; cross-scope lifetime surfaces leak | Violates ADR-006 and puts ref-count bookkeeping back on the control plane |
| **E: Submission with DepMode + Outstanding Window + Group Workspace (chosen)** | Each addition addresses one of the three pressures independently; together they preserve uniform Task state machine (ADR-003) and uniform per-Layer Scheduler (ADR-008) | Introduces several new types (`SubmissionDescriptor`, `DepMode`, `IntraGroupEdge`, `WorkspaceHandle`) | Accepted — minimal surface, orthogonal concerns |

### Consequences

**Positive:**

- `DATA`-mode cost is bounded by `boundary_in_count × avg_arg_count × O(1)` producer-index lookups, independent of Submission size.
- `NONE` mode is a single-bit hot path for known-independent submission streams (data loaders, micro-batches).
- `BARRIER` mode is a first-class primitive without requiring a synthetic zero-arg Task.
- The three modes also have different **synchronization** obligations that directly follow from their external-edge semantics: `DATA` is the only mode that requires holding the per-Layer dep-metadata lock during admission (it concurrently reads/writes `producer_index` and fan-in counters that prior-task completion handlers are mutating); `BARRIER` needs only an atomic snapshot of the outstanding-Submission set and takes no dep-metadata lock; `NONE` takes no lock at all. Callers that can assert `NONE` or `BARRIER` therefore avoid admission-side lock contention entirely — a meaningful win in multi-threaded scheduler deployments and for inline-admission paths. Specified in [§2.1.3.1.B](02-logical-view/02-scheduler.md#2131b-dependency-resolution-per-submission) and [§2.10.4](02-logical-view/12-dependency-model.md#2104-dependency-maintenance).
- The Outstanding Submission Window gives a single, principled back-pressure point independent of per-task resource caps.
- Group Submissions turn `N` tensor allocations + `N` frees into `1` workspace `alloc_workspace` + `1` `free_workspace`, cutting memory-manager traffic on admission/retirement.
- SPMD Submissions become a trivial specialization of Group Submission — one window slot regardless of `spmd_count`.
- Uniform Task state machine (ADR-003) and per-Layer Scheduler decomposition (ADR-008) are unchanged; the Submission layer is additive.

**Negative:**

- New types (`SubmissionDescriptor`, `SubmissionHandle`, `DepMode`, `IntraGroupEdge`, `WorkspaceRequest`, `WorkspaceHandle`) widen the public API surface.
- `NONE` mode is caller-asserted; mis-use is a correctness bug the runtime will not detect.
- Boundary classification must be accurate. The runtime accepts caller-supplied masks and, when absent, derives classification from `intra_edges` plus tensor-argument inspection — wrong classification on the caller's part can either miss a real external edge (`DATA` mode, non-boundary task actually reading an external tensor) or waste a workspace slot (tensor marked non-boundary but read by a later Submission). Both failures are detectable in diagnostic builds but not on the hot path.
- `max_outstanding_submissions` becomes a per-Layer tuning parameter with real latency-vs-throughput trade-offs. Default values must be chosen per Layer (§`modules/scheduler.md` §8).

### Related

- [ADR-003: Uniform ISchedulerLayer Interface](#adr-003-uniform-ischedulerlayer-interface) — Submission API extends `ISchedulerLayer` additively; Task state machine unchanged.
- [ADR-006: Tensor Lifecycle with Implicit Scope and Explicit Free](#adr-006-tensor-lifecycle-with-implicit-scope-and-explicit-free) — Group Workspace is a bulk-lifetime complement, not a replacement.
- [ADR-008: Scheduler Internal Decomposition with Strategy-Pattern Policy Hooks](#adr-008-scheduler-internal-decomposition-with-strategy-pattern-policy-hooks) — admission window is owned by TaskManager; `IResourceAllocationPolicy.should_admit` is the new policy hook.
- [02-logical-view §2.4.A–§2.4.D](02-logical-view/07-task-model.md#24a-submission-model) — Submission Model.
- [02-logical-view §2.1.3.1.A / §2.1.3.1.B](02-logical-view/02-scheduler.md#2131a-submission-admission--outstanding-window) — window + dep-mode mechanics.
- [02-logical-view §2.1.6.A](02-logical-view/04-memory.md#216a-group-workspace-memory) — Group Workspace interface.
- [04-process-view §4.5.5 / §4.5.6 / §4.5.7](04-process-view.md#455-per-submission-dependency-modes) — runtime behavior.

---

## ADR-013: Two-Stage Dependency Analysis — Frontend Intra-Submission DAG + Runtime Cross-Submission RAW

### Status

Accepted

### Context

A separate design document, [`tensor-dependency.md`](../../tensor-dependency.md), proposes a full two-layer dependency analysis for the data-driven programming model: a **data-dependency layer** built on tensor-object identity (RAW) and a **resource-dependency layer** built on per-memref version lists (WAR, WAW, and memref sub-range overlap — e.g., for assemble-style ops). The document optionally unifies the two layers with a token-tensor mechanism so that all hazards appear as RAW edges in a single DAG.

The runtime already has dependency machinery: `SubmissionDescriptor` carries pre-built `intra_edges` ([§2.4.A](02-logical-view/07-task-model.md#24a-submission-model)), and `TaskManager` maintains a `BufferRef → producer` `producer_index` for cross-Submission edge construction in `DATA` mode ([§2.1.3.1.B](02-logical-view/02-scheduler.md#2131b-dependency-resolution-per-submission)). That mechanism, however, tracks only one producer per `BufferRef` — it is RAW-only and has no version list.

Three options were considered for absorbing the tensor-dependency.md design into the runtime architecture:

1. **Move the full two-layer analysis into the runtime's TaskManager.** TaskManager would maintain per-`BufferRef` version lists, detect WAR/WAW across Submissions, and handle memref sub-range overlap for assemble.
2. **Keep the runtime as-is and push all dep work to the frontend.** The frontend resolves every hazard (intra- and cross-Submission) before calling `submit(...)`; the runtime becomes a pure dispatcher with no dep state.
3. **Split placement.** The frontend owns intra-Submission hazard analysis (per tensor-dependency.md), producing `intra_edges` + boundary masks; the runtime owns cross-Submission RAW via its existing `producer_index`, with WAR/WAW and sub-range deferred.

### Decision

Adopt **option 3 (split placement)** for v1:

- The frontend (tracer/DSL layer above `bindings/`) is the normative owner of the tensor-dependency.md algorithm. Per Submission, it builds the full RAW / WAR / WAW / assemble-overlap analysis, then lowers the result into the runtime's first-class `intra_edges` + `boundary_in_tasks` / `boundary_out_tasks` fields of `SubmissionDescriptor`. The token mechanism from tensor-dependency.md §7, if used, is a frontend-internal realization of edges — tokens need not appear as runtime `BufferRef`s.
- The runtime's `TaskManager` continues to run `DATA`-mode resolution as RAW-only, via the single-valued `producer_index`. Insertion on admission, eviction on `SUBMISSION_RETIRED`, and generation-guarded lookups are formalized as the `producer_index` lifecycle ([§2.1.3.1.B](02-logical-view/02-scheduler.md#2131b-dependency-resolution-per-submission), [§2.10.4](02-logical-view/12-dependency-model.md#2104-dependency-maintenance)).
- The soundness of RAW-only cross-Submission resolution rests on the **non-aliasing intermediate-memref invariant** owed by `IMemoryManager` ([§2.1.6](02-logical-view/04-memory.md#216-memory-manager), [§2.10.5](02-logical-view/12-dependency-model.md#2105-invariants)): without memref overlap across live `BufferRef`s, there is no cross-Submission WAR/WAW for the runtime to miss.
- Cross-Submission WAR/WAW, runtime sub-range overlap analysis, and token-tensor realization in the runtime are deferred as open questions ([09-open-questions.md](09-open-questions.md)).

The normative dependency-model contract is collected in [§2.10 Dependency Model](02-logical-view/12-dependency-model.md).

### Alternatives Considered

| Alternative | Pros | Cons | Why Rejected |
|-------------|------|------|--------------|
| Full two-layer analysis in TaskManager | Single source of truth; frontend stays simple; handles dynamic aliasing the frontend cannot see at trace time | Bloats the runtime's hot path (per-argument version-list maintenance); duplicates compiler-style analysis that the frontend already needs for codegen; makes retirement GC non-trivial (version nodes may outlive any single Submission) | Too much mechanism for the current workload; no concrete trigger requires runtime-side WAR/WAW yet |
| Push all dep work to the frontend | Runtime becomes tiny; dependency analysis stays in one layer | Frontend would need to understand cross-Submission outstanding windows, admission ordering, and retirement events — concerns that are naturally the runtime's. Every caller (not just the trace/DSL) would have to reimplement this. | Breaks encapsulation; the `producer_index` is cheaper and simpler than exposing runtime state to every frontend |
| Token-based unification inside the runtime | Uniform RAW-edge model; compatible with existing `producer_index` | Duplicates mechanism — tokens become synthetic `BufferRef`s whose only purpose is edge construction, while `intra_edges` already carries the same information as first-class edges; extra memory traffic and debug noise | Runtime already has first-class edges; no need to smuggle them through tensors |

### Consequences

**Positive:**

- Runtime dependency surface stays small: one `producer_index` hash map, RAW-only edges, no version-list maintenance on the hot path.
- Frontend complexity is bounded to one Submission at a time (per tensor-dependency.md), with per-Submission GC: the frontend's analysis state is discarded after `submit(...)` returns.
- The handoff contract (`intra_edges`, boundary masks) is already a first-class part of `SubmissionDescriptor` — no new fields or interfaces required.
- Non-aliasing invariant is a single, auditable obligation on `IMemoryManager`, making the soundness argument self-contained.

**Negative:**

- Correctness of intra-Submission hazard detection (RAW / WAR / WAW / assemble overlap) is a frontend responsibility; a missing `intra_edges` entry is a frontend defect the runtime will not detect. Diagnostic-build assertions (e.g., conflicting `BufferRef` writes with no joining edge) are a future addition.
- Workloads that require cross-Submission memref reuse (dynamic aliasing not resolvable at trace time) cannot be expressed safely in v1 without either extending `DATA` mode or conservatively promoting to `BARRIER`. This trade-off is captured as an open question.
- Two documents now describe the end-to-end dependency pipeline: [`tensor-dependency.md`](../../tensor-dependency.md) (frontend algorithm) and [§2.10](02-logical-view/12-dependency-model.md) (runtime contract). They must be kept in sync via the terminology map and the §0 scope note in tensor-dependency.md.

### Related

- [ADR-006: Tensor Lifecycle with Implicit Scope and Explicit Free](#adr-006-tensor-lifecycle-with-implicit-scope-and-explicit-free) — `BufferRef` identity and the non-aliasing invariant.
- [ADR-012: Submission Model with Dependency Modes, Outstanding Window, and Group Workspace](#adr-012-submission-model-with-dependency-modes-outstanding-window-and-group-workspace) — the Submission carrier this ADR plugs into; `DATA` mode's RAW-only scope is formalized here.
- [02-logical-view §2.10](02-logical-view/12-dependency-model.md) — the normative dependency contract.
- [02-logical-view §2.1.6](02-logical-view/04-memory.md#216-memory-manager) — non-aliasing invariant on `IMemoryManager`.
- [04-process-view §4.5.8](04-process-view.md#458-dependency-construction-pipeline) — runtime construction/retirement flow.
- [`tensor-dependency.md`](../../tensor-dependency.md) — frontend dep-analysis specification (workspace root).
- [09-open-questions.md](09-open-questions.md) — deferred: runtime WAR/WAW, sub-range overlap on `TaskArgs`, token adoption.

---

<!-- [UPDATED: A7-P6: new ADR added] -->

## ADR-014: Freeze `runtime::composition` Sub-Namespace + Promotion Triggers

**Status:** Accepted  
**Date:** 2026-04-18  
**Deciders:** Architecture team  
**Driven by:** A7-P6 (extract MLR + deployment parser)

### Context

`MachineLevelRegistry`, `MachineLevelDescriptor`, `DeploymentConfig`, and the deployment-configuration parser are currently defined inside the top-level `runtime/` module. A7-P6 observed that these four types form a cohesive "composition" concern — they translate deployment configuration into the factory-wired Layer tree — and that keeping them alongside `Runtime` blurs SRP (Rule D3) and invites accidental `transport/`- or `distributed/`-side includes. Extracting them into a dedicated module (`composition/`) is attractive but premature: the composition surface has exactly one consumer (`runtime::Runtime`) and ships a stable deployment schema. Promoting too early would add a module-boundary without a second consumer to justify it.

### Decision

Freeze the four composition concerns inside a **sub-namespace** `runtime::composition` with its own `include/` directory (`runtime/include/runtime/composition/*.h`). Cross-includes from other runtime headers into `composition/` headers are **forbidden** and enforced by IWYU-CI. The sub-namespace is **not** promoted to a top-level `composition/` module in v1.

Promotion is triggered when **either** of the following holds:

1. The cross-consumer list grows to **≥ 2** modules that include `composition/` headers (today only `runtime/` consumes them). A legitimate second consumer — e.g., a standalone deployment-validator tool or a `distributed_scheduler` that wants to resolve peer Machine Levels independently — flips this trigger.
2. Deployment-schema churn exceeds **N changes / quarter** (baseline N = 4, revisited at promotion time). Frequent schema change indicates the composition surface is evolving independently of `Runtime` and should own its release cadence.

On promotion, `runtime::composition` is moved under `composition/` with its `runtime::composition` namespace preserved as an alias for one release to keep source-level compatibility.

### Consequences

**Positive:**

- Cohesion: all composition types live under one namespace + `include/` subtree, improving discoverability.
- IWYU-CI enforces the boundary today, so the cost of eventual promotion is mostly moving files and adding a CMake target.
- Explicit promotion triggers replace a standing "when should we split this?" debate.

**Negative:**

- Sub-namespace inside a top-level module is a less familiar pattern than per-module split; module-ownership diagrams must call out that `runtime::composition` is a frozen sub-namespace, not a separate module.
- Rewriting `#include "runtime/machine_level_registry.h"` to `#include "runtime/composition/machine_level_registry.h"` touches downstream callers.

### Related

- [ADR-002: Machine Level Registry with Pluggable Factories](#adr-002-machine-level-registry-with-pluggable-factories) — the registry is the central composition type.
- [03-development-view.md §3.1](03-development-view.md#31-module-structure) — module listing + dependency graph updated with the sub-namespace.
- [`modules/runtime.md`](modules/runtime.md) — module design notes.

---

<!-- [UPDATED: A7-P4, A2-P6: new ADR added] -->

## ADR-015: Distributed Header Registration Policy (Invariant I-DIST-1)

**Status:** Accepted  
**Date:** 2026-04-18  
**Deciders:** Architecture team  
**Driven by:** A7-P4 (move distributed payload structs to `distributed/`), A2-P6 (`IDistributedProtocolHandler` abstract boundary)

### Context

Before this decision, `transport/messages.h` defined both framing types (`MessageHeader`, `MessageType` tags) and distributed-protocol payload types (`RemoteSubmitPayload`, `HeartbeatPayload`, and siblings). Co-locating the two groups violates D3/D5/SRP: `transport/` should own a typed byte channel, and `distributed/` should own the meaning of each `MessageType`. The co-location also exposes `distributed/` types to any `transport/` consumer via header reachability, which breaks the A2-P6 goal of a clean `IDistributedProtocolHandler` boundary.

### Decision

Adopt **Invariant I-DIST-1**: no header under `distributed/include/` may be includable (directly or transitively) from any translation unit whose owning target is `transport/`. Concretely:

1. Remove all `*Payload` struct definitions from `transport/messages.h`. `transport/` retains only `MessageHeader`, framing, and `MessageType` tag enum.
2. Move payload struct definitions to `distributed/include/distributed/protocol_payloads.h` and related files.
3. `transport/` public API becomes `send(peer, MessageType, span<const std::byte>, Timeout)` — payloads are opaque byte spans at the transport boundary.
4. Enforce the non-inclusion rule via **IWYU-CI** (include-what-you-use lint, fail-on-violation) plus a Bazel/CMake **visibility rule** that denies `transport/` targets from depending on `distributed/`.
5. The v1 `DistributedMessageHandler` free function (per A2-P6) lives in `distributed/protocol.hpp` keyed by `MessageType`; transport hands it an opaque span plus the type tag.

### Consequences

**Positive:**

- Transport can be unit-tested with a synthetic-loopback harness that never links `distributed/`.
- Adding a new distributed message type requires **zero** changes to `transport/`.
- A2-P6's deferred `IDistributedProtocolHandler` abstract class can be introduced in v2 behind the same header boundary with no breaking change to transport.

**Negative:**

- The transport send path pays one extra indirection (span + type tag vs. typed struct); measured cost is < 1 ns on the control path (A2-P6 microbench placeholder, not a hot-path regression).
- Two translation units instead of one for the message layer — marginal build-time cost.

### Related

- [ADR-005: Distributed-First Design](#adr-005-distributed-first-design) — distributed is a first-class layer; I-DIST-1 protects that layering.
- [`modules/transport.md`](modules/transport.md) §2.3 — transport API.
- [`modules/distributed.md`](modules/distributed.md) — protocol-payload ownership.
- [Appendix B](appendix-b-codebase-mapping.md) — ownership column updated.

---

<!-- [UPDATED: A3-P1, A4-R3-P2: new ADR added] -->

## ADR-016: TaskState Canonical Spelling + Transitions

**Status:** Accepted  
**Date:** 2026-04-18  
**Deciders:** Architecture team  
**Driven by:** A3-P1 (add `ERROR` state to Task FSM), A4-R3-P2 (unify terminal-failed-Task spelling)

### Context

Three independent problems converged on the Task state machine:

1. The original FSM had no terminal failure state — a Task that failed during `DISPATCHED`, `EXECUTING`, or `COMPLETING` had nowhere to go (LSP / D7 violation observed during scenario-view review).
2. Cancellation had no formal transition; scenario documents referenced ad-hoc "CANCELLED" states.
3. `06-scenario-view.md` used three different spellings for terminal-failed Tasks (`COMPLETED(error)`, `ERROR`, `FAILED`), conflating Task and Worker failure and drifting from the process view.

### Decision

Canonicalize the TaskState FSM as follows:

1. **`ERROR`** is the canonical terminal failure state for Tasks. It is reachable from `DISPATCHED`, `EXECUTING`, and `COMPLETING` via a new handler `on_fail(task, error)`. The transition is `COMPLETING → ERROR` (Task).
2. **`CANCELLED`** is an **optional** state (`A3-P1b`, medium severity, conditional on cancellation becoming a v1 feature) reachable from `PENDING`, `DEP_READY`, `RESOURCE_READY`, `DISPATCHED`, and `EXECUTING` via `on_cancel(task, reason)`.
3. The **Worker** terminal failed state is spelled **`FAILED`** (not `ERROR`). This keeps Task-level and Worker-level failure distinguishable in traces and diagrams.
4. `06-scenario-view.md` lines 91, 93, 95, 108, 111 collapse the three prior spellings to `COMPLETING → ERROR` (Task) or `FAILED` (Worker), consistent with the above.

### Consequences

**Positive:**

- One canonical spelling across process view, scenario view, and glossary (D7 / V5 consistency restored).
- `ERROR` gives downstream consumers (retry, sibling cancellation, parent propagation) a first-class state to observe rather than a sentinel.
- Optional `CANCELLED` is pre-factored so cancellation can ship without re-opening the FSM.

**Negative:**

- Every handler table that previously treated failure as "sentinel on `COMPLETING`" must now enumerate `ERROR`.
- Appendix-B TaskState count and all FSM diagrams require regeneration (A4-P8 handles the doc side).

### Related

- [A3-P1](reviews/2026-04-18-171357/final/final-proposal.md) — original ERROR-state proposal.
- [A4-R3-P2](reviews/2026-04-18-171357/final/final-proposal.md) — scenario-view spelling unification.
- [04-process-view.md §4.3.1-.2](04-process-view.md#431-task-state-machine) — state diagram + handler table.
- [02-logical-view §2.4.4](02-logical-view/07-task-model.md#244-task-state-model) — TaskState definition.
- [Appendix B](appendix-b-codebase-mapping.md) — TaskState enumeration.

---

<!-- [UPDATED: A2-P3, A9-P4: new ADR added] -->

## ADR-017: Closed-Enum-in-Hot-Path Policy

**Status:** Accepted  
**Date:** 2026-04-18  
**Deciders:** Architecture team  
**Driven by:** A2-P3 (open extension-point enums; keep `DepMode` closed), A9-P4 (drop `SubmissionDescriptor::Kind`)

### Context

The design has two categories of enum:

- **Deployment-time enums** (`FailurePolicy`, `TransportBackend`, `NodeRole`, `SimulationMode`): read once per Layer creation; extensibility (OCP / Rule E4) dominates over dispatch cost.
- **Hot-path enums** (`DepMode`, historically `SubmissionDescriptor::Kind`): read per Submission on the admission critical path; virtual dispatch or registry lookup would be a measurable regression (Rule X3).

Before this ADR, it was unclear which enums were allowed to remain closed `enum class` values versus which must become string-keyed registry entries. Reviewers A2 and A9 raised the conflict: A2-P3 wants most enums open (string-keyed factories at `Runtime::init`), while A9-P4 and A1 insisted closed enums on the admission path.

### Decision

Codify a two-rule policy:

1. **Default: open.** Enums that name an extension point consumed at deployment time (`FailurePolicy`, `TransportBackend`, `NodeRole`, `SimulationMode`) become **string IDs resolved via a registry at `Runtime::init`**. The resolver returns a factory; the enum itself, where retained, is a bookkeeping tag only.
2. **Exception: closed.** Enums read on the admission or dispatch critical path may remain closed `enum class` values **iff** (a) the closed set is documented in the owning module, (b) a microbenchmark demonstrates that registry-keyed dispatch regresses the relevant p50/p99 admission budget by more than **10 %**, or (c) the author records the benchmark as "benchmark-or-veto" evidence in the ADR-018 tracking doc. `DepMode` meets (a) + (b); `SubmissionDescriptor::Kind` was dropped entirely (A9-P4) once the `spmd.has_value()` discriminator proved sufficient.

Any future proposal to promote a closed enum to open (or vice versa) must attach a benchmark that satisfies the rule above; the benchmark is the veto gate.

### Consequences

**Positive:**

- Extensibility (E4) is the default; perf-motivated closure is an explicit, evidenced exception.
- `DepMode` keeps its single-byte hot-path read; no registry lookup on admission.
- `SubmissionDescriptor::Kind` removal is recorded with its rationale rather than appearing as a silent deletion.

**Negative:**

- Reviewers must agree on microbenchmark methodology (documented under `modules/scheduler.md` microbench section).
- A closed enum that is later found to need extension requires a registry migration; the cost is bounded by the number of call sites, but it is not zero.

### Related

- [A2-P3](reviews/2026-04-18-171357/final/final-proposal.md) — open extension-point enums rule.
- [A9-P4](reviews/2026-04-18-171357/final/final-proposal.md) — `Kind` removal.
- [A2-P8](reviews/2026-04-18-171357/final/final-proposal.md) — `DepMode` recorded as a known deviation.
- [10-known-deviations.md](10-known-deviations.md) — closed-enum rule exceptions index.

---

<!-- [UPDATED: A5-P11: new ADR added] -->

## ADR-018: Single Unified Peer-Health FSM

**Status:** Accepted  
**Date:** 2026-04-18  
**Deciders:** Architecture team  
**Driven by:** A5-P11 (unified peer-health FSM)

### Context

Peer-health state was previously tracked by three independent mechanisms: the circuit breaker (A5-P2), the heartbeat subsystem (A10-P6), and the authentication layer (A6-P2). Under chaos / fault scenarios from A5-P5 — notably the "§6.2.2 replay: recovering network + credential churn" trace — the three views oscillated (breaker re-opening while heartbeat said `HEALTHY`, auth flipping `AUTH_REVOKED` while the breaker was `HALF_OPEN`) because no single store held the authoritative peer state. Reviewers A5, A10, A6 concurred that the correct fix is one FSM co-owned by all three producers.

### Decision

Introduce a **single unified peer-health FSM** with states:

- `HEALTHY`
- `SUSPECT` — heartbeat miss short of fast-threshold; breaker counting
- `OPEN` — breaker tripped
- `HALF_OPEN` — breaker probing
- `UNAVAILABLE(Quarantine)` — caller-initiated isolation (folded from A5-P9)
- `LOST` — heartbeat-timeout-declared unreachable
- `AUTH_REVOKED` — credential failure or mTLS pinning rejection

Ownership model:

- **Breaker** (A5-P2) drives `HEALTHY ↔ SUSPECT ↔ OPEN ↔ HALF_OPEN` transitions via accumulator evidence; `breaker_auth_fail_weight` (A5-P13) weights `AuthenticationFailed` events.
- **Heartbeat** (A10-P6) drives `HEALTHY ↔ SUSPECT ↔ LOST` transitions using `heartbeat_miss_threshold_fast` and the hysteresis rule.
- **Auth** (A6-P2) drives any state → `AUTH_REVOKED` transition (irreversible for the current credential generation).
- `UNAVAILABLE(Quarantine)` is set by the caller (e.g., drain / maintenance) and is orthogonal to the other producers.

State is stored once, per peer, behind a single RW lock. All three producers read and write through that lock; transition rules are a union of the three producers' rules with `AUTH_REVOKED` as a dominant sink.

### Consequences

**Positive:**

- No oscillation under recovering-network + credential-churn scenarios (A5-P5 6.2.2 regression case now passes as of R3).
- Diagnostic `dump_state()` (A8-P4) exposes one row per peer instead of three partial views.
- Prometheus / OTEL alert rules (A8-P5) bind to a single `PeerHealthState` metric.

**Negative:**

- Lock contention on peer-health writes is potentially higher than before; mitigated by per-peer lock (peer count is bounded; Q18 tracks the > 128-peer shard-cap concern).
- All three producers must handle the richer enum; unit tests must cover cross-producer transition races.

### Related

- [ADR-005: Distributed-First Design](#adr-005-distributed-first-design).
- [A5-P2](reviews/2026-04-18-171357/final/final-proposal.md) — circuit breaker.
- [A6-P2](reviews/2026-04-18-171357/final/final-proposal.md), ADR-020 — authentication.
- [A10-P6](reviews/2026-04-18-171357/final/final-proposal.md) — heartbeat.
- [A5-P13](reviews/2026-04-18-171357/final/final-proposal.md) — `breaker_auth_fail_weight`.
- [`modules/distributed.md`](modules/distributed.md) §3.5 — FSM definition.

---

<!-- [UPDATED: A10-P1, A10-P7: new ADR added] -->

## ADR-019: Admission Shard Default + Deployment Cue

**Status:** Accepted  
**Date:** 2026-04-18  
**Deciders:** Architecture team  
**Driven by:** A10-P1 (default `producer_index` sharding policy), A10-P7 (two-tier sharded TaskManager)

### Context

A10-P1 introduced a per-shard lock keyed by `hash(BufferRef) mod N` for the `producer_index`, and A10-P7 introduced a two-tier sharded TaskManager. Both are necessary for scalability beyond single-host deployments, but adding sharding by default penalizes the common single-submitter / single-node case (extra atomic + shard dispatch on every admission). Reviewers were split: A10 wanted `shards=8` default; A1 wanted `shards=1` to preserve the hot path. The resolution requires both a conservative default **and** a clear deployment cue that documents when to opt in.

### Decision

1. **Default shard count = 1** across all Machine Levels. This preserves today's single-threaded admission fast path and avoids any microbenchmark regression on single-submitter workloads.
2. **Opt-in recommended values:** shards = 8 at Host, shards = 4 at Device, shards = 1 at Chip and below. Per-Layer configuration in `LevelParams`.
3. **Deployment cue** (documented in `05-physical-view.md`): opt in to sharding when `concurrent_submitters × cluster_nodes ≥ 64`. Below that threshold, the per-shard overhead outweighs contention reduction.
4. The **outstanding-submission window** remains a **global atomic counter** across shards (A10-P7); sharding the counter would break the Layer-wide back-pressure invariant.
5. Cross-shard ordering is by global `submission_id`; earliest-first completion bias is preserved **per-shard**.

### Consequences

**Positive:**

- Single-submitter / single-node deployments (today's primary workload) pay zero sharding overhead.
- Large-cluster deployments have a documented, evidence-backed opt-in threshold rather than a "tune your shards" knob.
- The global outstanding-window atomic is a known cost paid once per admission, independent of shard count.

**Negative:**

- Deployment authors must read the cue; misconfiguration (e.g., `shards=8` on a small cluster) regresses admission latency.
- Two-tier bookkeeping (per-shard state + global counter) is harder to reason about than a flat scheduler.

### Related

- [A10-P1](reviews/2026-04-18-171357/final/final-proposal.md).
- [A10-P7](reviews/2026-04-18-171357/final/final-proposal.md).
- [A1-P14](reviews/2026-04-18-171357/final/final-proposal.md) — `producer_index` cap/layout.
- [05-physical-view.md](05-physical-view.md) — deployment cue table.
- [`modules/scheduler.md`](modules/scheduler.md) — sharded TaskManager notes.

---

<!-- [UPDATED: A6-P2: new ADR added] -->

## ADR-020: Coordinator Generation + Stale-Coordinator Reject

**Status:** Accepted  
**Date:** 2026-04-18  
**Deciders:** Architecture team  
**Driven by:** A6-P2 (concrete node authentication in HANDSHAKE)

### Context

`HandshakePayload` (A6-P2) establishes node identity via mTLS with cert pinning. Under coordinator failover (A5-P3), the old coordinator's credentials remain valid until TTL expiry, enabling a "demoted-coordinator replay" attack: a malicious or disconnected former coordinator replays a stored HANDSHAKE against peers and reinserts itself into the `ClusterView`. Cert pinning alone does not prevent this because the demoted coordinator's cert is still valid. Review A6-R3 required a generation-based reject rule.

### Decision

1. Add `uint64_t coordinator_generation` to `HandshakePayload` and to `ClusterView`. The coordinator generation is **monotonically bumped** by the surviving peers on any coordinator failover.
2. On receipt of a HANDSHAKE, a peer compares the inbound `coordinator_generation` against the locally known current generation. If the inbound value is **strictly less than** the local value, the handshake is rejected with `ErrorCode::StaleCoordinatorClaim` and the offending node transitions to `AUTH_REVOKED` in the unified peer-health FSM (ADR-018).
3. Equal generation is accepted (ordinary re-handshake); strictly greater generation triggers a coordinator-discovery path outside this ADR (covered by A10-P2 v2 decentralized coordinator ADR).
4. `cluster_view` gains a `verified: bool` flag (A6-P2) set only after a successful generation-checked handshake; downstream scheduler decisions predicate on `verified == true`.

### Consequences

**Positive:**

- Demoted-coordinator replay attacks fail at HANDSHAKE with a first-class error code.
- `coordinator_generation` is a single uint64 on the handshake path — no hot-path cost.
- The generation-bump protocol aligns with the v2 decentralized-coordinator ADR trigger without pre-committing a specific algorithm.

**Negative:**

- Surviving peers must agree on when to bump the generation (quorum rule); A5-P3's fail-fast coordinator path specifies the bump as part of the new-coordinator election.
- Tools that replay captured HANDSHAKE traces for testing must synthesize a valid generation per run.

### Related

- [A6-P2](reviews/2026-04-18-171357/final/final-proposal.md).
- [A5-P3](reviews/2026-04-18-171357/final/final-proposal.md) — v1 deterministic coordinator fail-fast.
- [ADR-018: Single Unified Peer-Health FSM](#adr-018-single-unified-peer-health-fsm) — `AUTH_REVOKED` sink.
- [`modules/transport.md`](modules/transport.md) §2.3 / §5.3 — HANDSHAKE payload.
- [07-cross-cutting-concerns.md §7.1.3](07-cross-cutting-concerns.md#713-authentication-and-authorization) — authentication model.
