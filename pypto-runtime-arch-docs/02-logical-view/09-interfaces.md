# 2.6 Key Interface Definitions

> Part of the [Logical View](../02-logical-view.md). This module collects the principal interfaces that bind the other module components together: the uniform Scheduler contract, the Distributed Scheduler specialization, the three pluggable scheduling policy interfaces, and the event/execution interfaces supporting the event-driven model and deployment flexibility.

## 2.6.1 ISchedulerLayer

The uniform contract every Layer's Scheduler implements — the key abstraction enabling recursive hierarchical execution:

```cpp
class ISchedulerLayer {
public:
    virtual ~ISchedulerLayer() = default;

    // Layer identity
    virtual LayerId layer_id() const = 0;
    virtual const std::string& level_name() const = 0;

    // Component wiring (set during assembly)
    virtual void set_memory_manager(IMemoryManager* mgr) = 0;
    virtual void set_memory_ops(IMemoryOps* ops) = 0;
    virtual void set_vertical_channel_to_child(IVerticalChannel* ch) = 0;
    virtual void set_vertical_channel_to_parent(IVerticalChannel* ch) = 0;
    virtual void set_horizontal_channel(IHorizontalChannel* ch) = 0;

    // Submission lifecycle — all submissions go through a single entry point.
    // The returned SubmissionHandle identifies the whole Submission; per-task
    // handles can be retrieved via submission_tasks(handle). See §2.4.A.
    // [UPDATED: A3-P2] Normative return type is std::expected<SubmissionHandle,
    // ErrorContext>; see the callout below. A throwing overload wraps this.
    virtual std::expected<SubmissionHandle, ErrorContext>
        submit(const SubmissionDescriptor& desc) = 0;

    // [UPDATED: A9-P1] Overloads submit(TaskDescriptor), submit_group(...),
    // submit_spmd(...) are REMOVED from the interface. Ergonomic call sites use
    // the free-function builders build_single / build_group / build_spmd
    // (runtime/submission_builders.h).

    // Resolve the per-task handles for an admitted Submission.
    virtual std::span<const TaskHandle> submission_tasks(SubmissionHandle s) const = 0;

    // Dependency management
    virtual void notify_dep_satisfied(TaskHandle task, TaskHandle completed_dep) = 0;

    // Child task completion (called by layer below)
    virtual void notify_child_complete(TaskHandle parent, TaskHandle child) = 0;

    // Scope management
    virtual ScopeHandle scope_begin() = 0;
    virtual void scope_end(ScopeHandle scope) = 0;

    // Lifecycle
    virtual void init(const LayerConfig& config) = 0;
    virtual void drain() = 0;
    virtual void shutdown() = 0;

    // Status
    virtual bool is_idle() const = 0;
    virtual LayerStats get_stats() const = 0;
};
```

**Key behavioral contracts:**
1. `submit()` accepts a **Submission** ([§2.4.A](07-task-model.md#24a-submission-model)) and returns a valid `SubmissionHandle` once the Submission has been atomically admitted. If the **Outstanding Submission Window** ([§2.1.3.1.A](02-scheduler.md#2131a-submission-admission--outstanding-window)) is saturated, the call may block or return a back-pressure status per `IResourceAllocationPolicy` ([§2.6.3](#263-schedule-policy-interfaces)).
2. A Submission's admission is atomic: either every Task slot and every `intra_edges` entry is allocated, or the Submission is rejected as a whole.
3. `notify_child_complete()` is called by the child Layer when a child Task completes, decrementing the parent's pending-child counter.
4. `drain()` blocks until all submitted Tasks (and descendants) reach RETIRED and all Submissions have emitted `SUBMISSION_RETIRED`.
5. Memory Manager and Memory Ops are wired from registered factories.
6. Vertical and Horizontal Channels are wired from registered factories.

**Internal delegation:** Each `ISchedulerLayer` implementation delegates to its three internal sub-components ([§2.1.3](02-scheduler.md)): `submit()` → TaskManager (admission, intra-group edge installation, dep-mode resolution), dispatch → WorkerManager (via ResourceManager), completion notifications → TaskManager → WorkerManager. The external interface is unchanged (ADR-003); the decomposition is an internal concern of each implementation.

> [UPDATED: A3-P2: normative `std::expected<SubmissionHandle, ErrorContext>` return shape]
> The normative signature is
> ```cpp
> std::expected<SubmissionHandle, ErrorContext> submit(const SubmissionDescriptor& desc);
> ```
> `AdmissionStatus::WAIT` blocks (never returns via the error path); `AdmissionStatus::REJECT(...)` returns `std::unexpected(ErrorContext{AdmissionRejected, ...})`. A convenience throwing overload is provided for call sites that prefer exceptions; the `std::expected`-based signature is the contract tested against. See the inline ADR reference in `08-design-decisions.md` (covered by ADR-017).

> [UPDATED: A9-P1: collapse `submit()` overloads]
> The overloads `submit(TaskDescriptor)`, `submit_group(...)`, and `submit_spmd(...)` are **removed** from `ISchedulerLayer`. Only `submit(const SubmissionDescriptor&)` remains. Ergonomic call sites use free-function builders `build_single`, `build_group`, `build_spmd` in `runtime/`. This proposal is absorbed into A7-P2's role-split (ISP) and lands atomically with it.

> [UPDATED: A3-P12: `drain()` + `submit()` concurrency + distributed semantics]
> Once `drain()` is entered, `submit()` returns `AdmissionStatus::REJECT(Drain)` with sub-code `DrainInProgress`; the drain flag is **sticky** (does not clear on completion). Multiple concurrent `drain()` calls are idempotent; `shutdown()` after `drain()` is safe. `DrainInProgress` is a Core-domain `ErrorCode`. Cross-Pod drain semantics: the drain flag propagates via `REMOTE_DRAIN` messages with the same sticky guarantee per peer.

## 2.6.1.A Submission Types

Supporting data types for `ISchedulerLayer.submit(...)` ([§2.4.A](07-task-model.md#24a-submission-model), [§2.4.C](07-task-model.md#24c-group-submission-semantics)):

```cpp
enum class DepMode : uint8_t {
    BARRIER,   // wait for all currently outstanding Submissions to complete
    DATA,      // install producer->consumer edges for tensors this Submission reads
    NONE,      // no external edges; caller-asserted independence
};

struct IntraGroupEdge {
    uint32_t producer_index;   // index into SubmissionDescriptor::tasks
    uint32_t consumer_index;   // index into SubmissionDescriptor::tasks
};

struct WorkspaceRequest {
    size_t          total_bytes;     // sum of aligned non-boundary tensor sizes
    size_t          alignment = 0;   // 0 => region.natural_alignment
    RegionId        region = RegionId::Default;
    // Per-tensor sub-range layout is computed by the MemoryManager; see §2.1.6.A.
    std::vector<SubRangeSpec> subranges; // (task_index, arg_index, size, alignment)
};

struct SubmissionDescriptor {
    // [UPDATED: A2-P1] uint16_t schema_version — cross-module contract bump.
    // Multi-node wire scope only; see appendix-c-compatibility.md.
    uint16_t                        schema_version = 1;

    // [UPDATED: A9-P4] `enum Kind` and `kind` field REMOVED.
    // SPMD = spmd.has_value(); Single = tasks.size()==1 && !spmd.has_value();
    // Group  = tasks.size()>=1 && !spmd.has_value().
    DepMode                         dep_mode = DepMode::DATA;

    // Tasks contained in this Submission. Exactly 1 for SINGLE; N for GROUP/SPMD.
    std::vector<TaskDescriptor>     tasks;

    // Optional pre-built intra-Submission DAG edges (must be acyclic).
    std::vector<IntraGroupEdge>     intra_edges;

    // Boundary classification. Either supplied by the caller (preferred) or
    // derived by the Scheduler from task args + intra_edges.
    std::vector<uint32_t>           boundary_in_tasks;
    std::vector<uint32_t>           boundary_out_tasks;

    // Optional Group Workspace request for non-boundary tensors (§2.4.D).
    std::optional<WorkspaceRequest> workspace_request;

    // SPMDTaskDescriptor subsumes `tasks` + `intra_edges` when present.
    std::optional<SPMDTaskDescriptor> spmd;

    // [UPDATED: A3-P7] FE_VALIDATED bit lets release builds skip re-validation
    // of preconditions already enforced by the frontend dependency analyzer.
    uint32_t                        flags = 0;  // bitfield; FE_VALIDATED = 0x1
};

struct SubmissionHandle {
    uint32_t layer_id;
    uint32_t slot_index;
    uint32_t generation;
};
```

**Lifetime.** A `SubmissionHandle` is valid from successful return of `submit(...)` until the Submission emits `SUBMISSION_RETIRED` ([§2.1.3.5](02-scheduler.md#2135-event-driven-execution-model)). After retirement, the slot may be recycled; subsequent use of the handle is detected via the `generation` counter.

**Task handle retrieval.** Per-task handles (for dependency callbacks, cancellation, or logging) are returned by `submission_tasks(SubmissionHandle)` in the order tasks appear in `SubmissionDescriptor::tasks`.

> [UPDATED: A2-P1: schema_version on multi-node wire contracts]
> A `uint16_t schema_version` is added to cross-module contracts crossing a node/process boundary: `MessageHeader`, `TaskDescriptor`, `FunctionDescriptor`, and stats — plus persistent artifacts (trace header). Non-wire in-process types are covered by the A2-P5 BC checklist rather than blanket versioning. Validation runs at bindings/handshake boundaries only; v1 readers ignore unknown additive fields. See `appendix-c-compatibility.md` for the full BC policy and `modules/transport.md §2.3` for `MessageHeader`.

> [UPDATED: A3-P7: SubmissionDescriptor precondition catalog (A)]
> Preconditions enforced on the admission path for `SubmissionDescriptor`:
> - `tasks.size() ≥ 1` → sub-code `EmptyTasks`
> - `intra_edges[i].producer ≠ consumer` → sub-code `SelfLoopEdge`
> - All edge / boundary indices within `[0, tasks.size())` → `EdgeIndexOOB`, `BoundaryIndexOOB`
> - Workspace subrange bounds valid → `WorkspaceSubrangeOOB`, `WorkspaceSizeMismatch`
> - SPMD-specific: `SpmdIndexOOB`, `SpmdSizeMismatch`, `SpmdBlockDimZero`
> - Capacity exhaustion sub-codes routed from A1-P8: `TaskSlotExhausted`, `ReadyQueueExhausted`
>
> Release builds skip this catalog when `SubmissionDescriptor.flags & FE_VALIDATED` is set (fast path). Debug builds always run. All sub-codes route to `AdmissionStatus::REJECT(Validation)` or `AdmissionStatus::REJECT(Exhaustion)` respectively (see A9-P5).

## 2.6.2 Distributed Scheduler

The `ISchedulerLayer` implementation for distributed Machine Levels partitions and routes Tasks across sibling instances:

1. Receives submissions from user or Distributed Orchestration Function.
2. Applies a **Partition Strategy** (`IPartitioner`) to assign Tasks to target instances.
3. Sends `REMOTE_SUBMIT` messages via the Horizontal Channel.
4. Tracks cross-instance dependencies via `REMOTE_DEP_NOTIFY`.
5. Aggregates completion via `REMOTE_COMPLETE`.

**Partition Strategies:**

| Strategy | Description |
|----------|-------------|
| `STATIC` | User-specified Node assignment |
| `ROUND_ROBIN` | Rotate across available Nodes |
| `DATA_LOCALITY` | Assign to Node holding input data |
| `WORK_STEALING` | Idle Nodes steal from overloaded Nodes |

**Consistency Model:** Causal consistency per task chain — if Task A completes before Task B is submitted and B depends on A, then B observes A's outputs. Cross-chain ordering is not guaranteed.

## 2.6.3 Schedule Policy Interfaces

Three strategy interfaces enable customizable scheduling behavior at each sub-component's decision points. All are optional — default implementations are provided. Policies are injected at Layer construction time via the policy factory fields in `MachineLevelDescriptor` ([§2.2.1](05-machine-level-registry.md#221-machine-level)) and remain fixed for the Layer's lifetime.

### ITaskSchedulePolicy

Controls task ordering and preemption within the TaskManager.

```cpp
class ITaskSchedulePolicy {
public:
    virtual ~ITaskSchedulePolicy() = default;

    // Order ready tasks for evaluation. Called when multiple tasks are DEP_READY.
    // Returns a permutation of the input task handles reflecting desired evaluation order.
    virtual void rank_ready_tasks(TaskHandle ready[], size_t count) = 0;

    // Decide whether a newly ready task should preempt a currently dispatched task.
    // Returns true if preemption should occur.
    virtual bool should_preempt(const TaskDescriptor& new_task,
                                const TaskDescriptor& current_task) = 0;

    // Observation hook: called on every task state transition.
    // Implementations MUST NOT block — this is on the critical path.
    virtual void on_task_state_change(TaskHandle task, TaskState old_state,
                                      TaskState new_state) = 0;
};
```

**Default implementation (`FifoTaskSchedulePolicy`):** FIFO ordering with an **earliest-Submission bias** — among ready tasks, those belonging to the Submission with the lowest `submission_id` are evaluated first (this is what lets the Outstanding Submission Window ([§2.1.3.1.A](02-scheduler.md#2131a-submission-admission--outstanding-window)) make forward progress by preferring the earliest outstanding Submission to complete first). Ties are broken by ready-queue arrival order. No preemption; empty observation hook.

### IWorkerSelectionPolicy

Controls which Worker(s) are assigned to a dispatched Task, supporting both single-worker and multi-worker (group-constrained) selection.

```cpp
struct WorkerGroupAvailability {
    uint32_t group_id;
    std::string group_type;
    // Per-type idle counts within this group
    struct TypeCount { uint32_t worker_type_id; uint32_t idle_count; };
    std::vector<TypeCount> available_by_type;
};

struct WorkerAllocation {
    enum Status { ALLOCATED, NO_SUITABLE_WORKER };
    Status status;
    std::vector<uint32_t> worker_ids;  // allocated Worker IDs (1 for single-worker, N for multi-worker)
    uint32_t group_id;                 // 0 if no group constraint applied
};

class IWorkerSelectionPolicy {
public:
    virtual ~IWorkerSelectionPolicy() = default;

    // Select worker(s) for the given task based on its TaskExecType.
    // For tasks with requires_worker_group=true, returns workers from the same group.
    // For single-worker tasks, returns a single worker ID.
    virtual WorkerAllocation select_workers(
        const TaskDescriptor& task,
        const TaskExecType& exec_type,
        const std::vector<WorkerGroupAvailability>& group_availability,
        const WorkerState all_workers[],
        size_t worker_count) = 0;

    // Observation hook: called on every worker state transition.
    virtual void on_worker_state_change(uint32_t worker_id,
                                        WorkerState old_state,
                                        WorkerState new_state) = 0;
};
```

**Default implementation (`RoundRobinWorkerSelectionPolicy`):** For single-worker Tasks, round-robin across idle Workers of the matching type. For multi-worker Tasks with group affinity, selects the first Worker Group that can satisfy all `required_slots`.

### IResourceAllocationPolicy

Controls resource allocation decisions in the ResourceManager.

```cpp
class IResourceAllocationPolicy {
public:
    virtual ~IResourceAllocationPolicy() = default;

    // Submission-level admission gate. Called once per SubmissionDescriptor
    // before any per-task slot is allocated (§2.1.3.1.A). Returning ADMIT
    // proceeds with atomic admission; WAIT queues the submission on the
    // admission queue until a SUBMISSION_RETIRED event fires; REJECT returns
    // a back-pressure error to the caller.
    // [UPDATED: A9-P5] AdmissionDecision is aliased to AdmissionStatus (see below).
    // New code should use AdmissionStatus; AdmissionDecision kept as a compat typedef.
    enum class AdmissionDecision : uint8_t { ADMIT, WAIT, REJECT };
    virtual AdmissionDecision should_admit(
        const SubmissionDescriptor& submission,
        const ResourceSnapshot&     available) = 0;

    // Decide how to allocate resources (memory + worker) for a DEP_READY task.
    // Returns an allocation descriptor or a status indicating the task should wait.
    virtual ResourceAllocationResult allocate_resources(
        const TaskDescriptor& task,
        const ResourceSnapshot& available) = 0;

    // Decide whether to reject new submissions due to resource pressure.
    // Called on each submit() when resources are constrained.
    virtual bool should_backpressure(const ResourceSnapshot& available) = 0;

    // Observation hook: called when resource state changes
    // (allocation, release, pool resize).
    virtual void on_resource_change(const ResourceSnapshot& snapshot) = 0;
};
```

**Default implementation (`GreedyResourceAllocationPolicy`):**
- `should_admit`: returns `ADMIT` if `outstanding_submissions < max_outstanding_submissions` **and** any requested workspace bytes fit in the data region; returns `WAIT` when the window is full; returns `REJECT` only when the Submission's workspace or slot requirement exceeds the Layer's configured capacity.
- `allocate_resources`: allocates immediately if resources are available; returns "wait" otherwise. Back-pressure when task slot pool utilization exceeds 90%.

**Supporting types:**

```cpp
struct ResourceSnapshot {
    size_t   available_memory;
    size_t   total_memory;
    uint32_t free_task_slots;
    uint32_t total_task_slots;

    // Submission-window accounting (§2.1.3.1.A).
    uint32_t outstanding_submissions;      // currently in-flight Submissions
    uint32_t max_outstanding_submissions;  // per-Layer cap (LevelParams)
    uint32_t admission_queue_depth;        // Submissions waiting for a window slot
    size_t   outstanding_workspace_bytes;  // sum of allocated group workspaces

    // Per-type worker counts
    struct WorkerTypeSnapshot {
        uint32_t worker_type_id;
        uint32_t idle_count;
        uint32_t total_count;
    };
    std::vector<WorkerTypeSnapshot> workers_by_type;
    // Per-group availability (§2.1.4.2)
    std::vector<WorkerGroupAvailability> group_availability;
};

struct ResourceAllocationResult {
    enum Status { ALLOCATED, WAIT, REJECTED };
    Status     status;
    BufferRef  buffer;                       // allocated buffer (if ALLOCATED)
    std::vector<uint32_t> worker_ids;        // allocated Worker(s) (if ALLOCATED)
    uint32_t   group_id;                     // WorkerGroup used (0 if no group constraint)
};
```

> [UPDATED: A9-P5: unified `AdmissionStatus` enum]
> A single enum unifies admission outcomes across `IResourceAllocationPolicy::should_admit` and `ResourceAllocationResult::status`:
> ```cpp
> enum class AdmissionStatus : uint8_t {
>     ADMIT,
>     WAIT,
>     REJECT_Exhaustion,   // REJECT(Exhaustion) — slot/workspace/queue overflow
>     REJECT_Validation,   // REJECT(Validation) — precondition/schema violation
>     REJECT_Drain,        // REJECT(Drain)      — drain()/shutdown() in progress
> };
> ```
> (Conceptually `REJECT(Exhaustion|Validation|Drain)`; flattened for C++ enum storage.) Scenario text in `06-scenario-view.md §6.2.3` uses `AdmissionStatus::REJECT(Exhaustion)` rather than the former `ResourceExhausted`. The prior `AdmissionDecision` is a typedef-compat alias.

## 2.6.4 Event and Execution Interfaces

These interfaces support the event-driven execution model ([§2.1.3.5](02-scheduler.md#2135-event-driven-execution-model)) and deployment flexibility ([§2.1.3.6](02-scheduler.md#2136-schedulerworker-deployment-flexibility)).

### IEventSource

An event source delivers `SchedulerEvent`s to the Scheduler's event loop. Each source operates in one of two delivery modes:

```cpp
class IEventSource {
public:
    virtual ~IEventSource() = default;

    enum DeliveryMode { QUEUE, POLL };
    virtual DeliveryMode delivery_mode() const = 0;

    // POLL mode: check for a pending event. Returns true if an event was retrieved.
    virtual bool poll(SchedulerEvent* out_event) = 0;

    // QUEUE mode: access the underlying event queue.
    virtual IEventQueue* queue() = 0;
};
```

Concrete implementations are registered per Machine Level. Examples:
- `RegisterPollEventSource` (Core level): polls AICore COND registers for completion flags.
- `QueueEventSource` (Host/Device level): wraps an `IEventQueue` fed by Worker threads or Channel handlers.
- `ChannelEventSource`: adapter that translates Vertical/Horizontal Channel messages into `SchedulerEvent`s.

### IEventQueue

A bounded, thread-safe queue for event delivery between producers (Workers, Channels) and the Scheduler's event loop consumer:

```cpp
class IEventQueue {
public:
    virtual ~IEventQueue() = default;

    // Producer side (may be called from any thread)
    virtual bool try_enqueue(const SchedulerEvent& event) = 0;

    // Consumer side (called only from the Scheduler event loop)
    virtual bool try_dequeue(SchedulerEvent* out_event) = 0;

    // Status
    virtual size_t size() const = 0;
    virtual size_t capacity() const = 0;
};
```

Default implementations:
- `SPSCRingEventQueue`: single-producer single-consumer lock-free ring buffer. Used when a single Worker or Channel feeds the Scheduler.
- `MPSCRingEventQueue`: multi-producer single-consumer lock-free ring buffer. Used when multiple Workers report completions concurrently.

Queue capacity is configurable via `LevelParams.event_queue_capacity`.

### EventHandlingConfig

Per-event-type configuration controlling inline vs. deferred handling:

```cpp
struct EventHandlingConfig {
    struct Entry {
        SchedulerEvent::Type event_type;
        enum Mode { INLINE, DEFERRED } mode;
    };
    std::vector<Entry> entries;   // per-event-type mode overrides
    Mode default_mode = INLINE;   // fallback for unconfigured event types
};
```

Configured per Machine Level via `LevelParams.event_handling_config`.

> [UPDATED: A9-P7: fold `SourceCollectionConfig` into `EventHandlingConfig`]
> The former standalone `SourceCollectionConfig` struct is **merged into `EventHandlingConfig`** (source list, per-source weight, max events per cycle). The combined struct is cold-start only and is validated in a single pass at `deployment_parser` init. The retained `EventHandlingConfig` shape is:
>
> ```cpp
> struct EventHandlingConfig {
>     // Inline vs deferred (existing).
>     struct Entry { SchedulerEvent::Type event_type; enum Mode { INLINE, DEFERRED } mode; };
>     std::vector<Entry>  entries;
>     Mode                default_mode = INLINE;
>
>     // [FOLDED FROM SourceCollectionConfig]
>     enum Strategy { FIXED_PRIORITY, ROUND_ROBIN, WEIGHTED_ROUND_ROBIN, CUSTOM };
>     struct SourceEntry {
>         uint32_t source_id;
>         uint32_t max_events_per_cycle = 0;   // 0 = unlimited
>         uint32_t weight               = 1;   // WEIGHTED_ROUND_ROBIN only
>     };
>     Strategy                  strategy             = ROUND_ROBIN;
>     std::vector<SourceEntry>  sources;                 // FIXED_PRIORITY order
>     uint32_t                  cycle_event_budget   = 0;
>     bool                      empty_source_backoff = true;
> };
> ```
>
> The legacy `SourceCollectionConfig` definition below is retained for a single release as a compat typedef alias, then removed per the `appendix-c-compatibility.md` deprecation procedure.

### SourceCollectionConfig (retired; compat alias only)

Declarative configuration consumed by `IEventCollectionPolicy` to govern how Stage A of the event loop ([§2.1.3.5](02-scheduler.md#scheduler-event-loop)) collects events from registered `IEventSource`s:

```cpp
// [RETIRED: A9-P7] Fields folded into EventHandlingConfig. Kept as compat typedef
// for one release per BC policy.
using SourceCollectionConfig = EventHandlingConfig;
```

Configured per Machine Level via `LevelParams.event_handling_config` (unified entry point).

### IEventCollectionPolicy

Pluggable policy that drives Stage A of the event loop. Given the current `SourceCollectionConfig` and observed drain statistics, it returns the next `(source, max_events)` pair to drain in the current cycle; returning `DONE` ends the cycle.

```cpp
class IEventCollectionPolicy {
public:
    virtual ~IEventCollectionPolicy() = default;

    struct NextStep {
        enum Kind { DRAIN, DONE } kind;
        IEventSource* source = nullptr;   // valid iff kind == DRAIN
        uint32_t      max_events = 0;     // 0 means unbounded for this step
    };

    virtual void init(const SourceCollectionConfig& cfg,
                      const std::vector<IEventSource*>& registered_sources) = 0;

    // Called repeatedly within one collection cycle until it returns DONE.
    virtual NextStep next() = 0;

    // Feedback after Stage A drained `drained` events from `source` in this step.
    virtual void report(IEventSource* source, uint32_t drained) = 0;

    // Called once when the cycle ends (used for fairness bookkeeping / backoff).
    virtual void on_cycle_end() = 0;
};
```

Default implementations (one per `Strategy`): `FixedPriorityCollectionPolicy`, `RoundRobinCollectionPolicy`, `WeightedRoundRobinCollectionPolicy`. A `CUSTOM` policy can be supplied via the factory interface.

### EventLoopDeploymentConfig

Per-Layer configuration that binds the three event-loop stages ([§2.1.3.5](02-scheduler.md#scheduler-event-loop)) to physical execution units. The stages are always logically present; this config only controls where they run and how they hand off work.

```cpp
struct EventLoopDeploymentConfig {
    enum Mode {
        SINGLE_THREADED,   // A -> B -> C on one thread (queues degenerate to direct calls)
        SPLIT_COLLECTION,  // A on its own thread; B+C share the Scheduler thread
        SPLIT_DEFERRED,    // A+B on the Scheduler thread; C on its own thread
        FULLY_SPLIT        // A, B, C each on their own execution unit
    };

    Mode     mode = SINGLE_THREADED;

    // Optional affinity hints (ignored if platform does not support pinning).
    int      collection_cpu_affinity = -1;   // Stage A
    int      classify_cpu_affinity   = -1;   // Stage B (Scheduler thread)
    int      deferred_cpu_affinity   = -1;   // Stage C

    // Internal queue sizing and type selection.
    size_t   inline_dispatch_queue_capacity = 1024; // A -> B
    size_t   deferred_queue_capacity        = 1024; // B -> C
    bool     inline_dispatch_is_mpsc        = false; // true when A fans in multiple pollers
};
```

Configured per Machine Level via `LevelParams.event_loop_deployment`. The runtime validates that `EventHandlingConfig`, `SourceCollectionConfig`, and `EventLoopDeploymentConfig` are mutually consistent (e.g., `SPLIT_COLLECTION` requires a queue type that supports cross-thread handoff).

### IExecutionPolicy

Governs how a physical execution unit allocates time between Scheduler and Worker roles in Interleaved deployment mode ([§2.1.3.6](02-scheduler.md#2136-schedulerworker-deployment-flexibility)):

```cpp
class IExecutionPolicy {
public:
    virtual ~IExecutionPolicy() = default;

    enum Action { CONTINUE_SCHEDULING, EXECUTE_TASK, YIELD };

    // Called after each event-loop iteration to decide the next action.
    virtual Action next_action(const SchedulerStats& stats,
                               const TaskHandle* ready_task) = 0;
};
```

Default implementations:
- `DedicatedExecutionPolicy`: always returns `CONTINUE_SCHEDULING`. Workers are separate execution units.
- `InterleavedExecutionPolicy`: executes one ready task after each event-loop drain, then returns to scheduling.
- `BatchedExecutionPolicy`: accumulates N ready tasks, dispatches and executes all inline, then returns to scheduling.

Configured per Machine Level via `execution_policy_factory` in `MachineLevelDescriptor` ([§2.2.1](05-machine-level-registry.md#221-machine-level)).

## 2.6.5 Role Interfaces (A7-P2)

> [UPDATED: A7-P2: split `ISchedulerLayer` into role interfaces (absorbs A9-P1)]

`ISchedulerLayer` is refactored into a set of **role interfaces** each satisfying ISP (Interface Segregation Principle). Consumers include only the narrow header they need; implementers inherit from the aggregator. The role split is a **contract** — all role interfaces land atomically in a single PR, ordered after (or co-committing with) A3-P2's `std::expected` return-shape change.

```cpp
// core/i_scheduler_wiring.h — set during Layer assembly
class ISchedulerWiring {
public:
    virtual ~ISchedulerWiring() = default;
    virtual void set_memory_manager(memory::IMemoryManager* mgr) = 0;
    virtual void set_memory_ops(memory::IMemoryOps* ops) = 0;
    virtual void set_vertical_channel_to_child(transport::IVerticalChannel* ch) = 0;
    virtual void set_vertical_channel_to_parent(transport::IVerticalChannel* ch) = 0;
    virtual void set_horizontal_channel(transport::IHorizontalChannel* ch) = 0;
};

// core/i_scheduler_submit.h — admission entry point (A9-P1 absorbed — only one submit)
class ISchedulerSubmit {
public:
    virtual ~ISchedulerSubmit() = default;
    virtual std::expected<SubmissionHandle, ErrorContext>
        submit(const SubmissionDescriptor& desc) = 0;
    virtual std::span<const TaskHandle> submission_tasks(SubmissionHandle s) const = 0;
};

// core/i_scheduler_completion.h — upward/child notifications
class ISchedulerCompletion {
public:
    virtual ~ISchedulerCompletion() = default;
    virtual void notify_dep_satisfied(TaskHandle task, TaskHandle completed_dep) = 0;
    virtual void notify_child_complete(TaskHandle parent, TaskHandle child) = 0;
};

// core/i_scheduler_lifecycle.h — init / drain / shutdown / status
class ISchedulerLifecycle {
public:
    virtual ~ISchedulerLifecycle() = default;
    virtual void init(const LayerConfig& config) = 0;
    virtual void drain() = 0;            // sticky; see A3-P12
    virtual void shutdown() = 0;
    virtual bool is_idle() const = 0;
    virtual LayerStats get_stats() const = 0;
};

// core/i_scheduler_scope.h — scope begin/end
class ISchedulerScope {
public:
    virtual ~ISchedulerScope() = default;
    virtual ScopeHandle scope_begin() = 0;
    virtual void        scope_end(ScopeHandle scope) = 0;
};

// Aggregator for implementers / compatibility consumers.
class ISchedulerLayer : public ISchedulerWiring,
                       public ISchedulerSubmit,
                       public ISchedulerCompletion,
                       public ISchedulerLifecycle,
                       public ISchedulerScope {
public:
    virtual LayerId             layer_id()    const = 0;
    virtual const std::string&  level_name()  const = 0;
};
```

**Per-interface contracts.**
- `ISchedulerWiring`: called exactly once during Layer assembly; setters are not thread-safe and must be invoked before `init()`. After `init()` returns, pointers are immutable for the Layer's lifetime.
- `ISchedulerSubmit`: `submit()` admits atomically (A3-P2 `std::expected`). `submission_tasks()` is safe to call from any thread until the returned handle's Submission is retired.
- `ISchedulerCompletion`: `notify_*` methods are called from the child Layer or from internal completion handlers only; they decrement parent pending-child counters and MUST NOT allocate on the hot path.
- `ISchedulerLifecycle`: `drain()` is sticky (A3-P12) and idempotent; `shutdown()` is only valid after `drain()` has completed.
- `ISchedulerScope`: `scope_begin()` returns a stable `ScopeHandle` usable across threads; `scope_end()` is idempotent per handle.

**Implementation-side vtable layout** is unchanged relative to the single aggregated interface because the role interfaces share the same base implementation object — consumers include narrow headers purely for compile-time dependency reduction.

## 2.6.6 Event-Loop Test Seam (A9-P2)

> [UPDATED: A9-P2: single `IEventLoopDriver` test-only seam]

The sole interface retained from the collapsed policy-pluggability set is a **test-only event-loop driver**. It is gated on `LayerConfig.enable_test_driver` and does not link in release binaries.

```cpp
// core/i_event_loop_driver.h — test-only (gated on enable_test_driver)
class IEventLoopDriver {
public:
    virtual ~IEventLoopDriver() = default;

    // Advance the event loop by at most `max_events` events. Returns the number
    // processed. Used with RecordedEventSource to deterministically replay traces.
    virtual uint32_t step(uint32_t max_events) = 0;

    // Attach a RecordedEventSource for deterministic replay.
    virtual void attach_source(IEventSource* src) = 0;
};

// Recorded event source for deterministic replay (A8-P2).
class RecordedEventSource : public IEventSource {
    // Implementation replays a serialized EventTrace; see modules/scheduler.md §5.2.
};
```

A serialized `EventTrace` replayed through `RecordedEventSource` produces **bit-identical state transitions** across deployments — this is the normative definition of determinism for the scheduler under test. Release binaries omit both `IEventLoopDriver` and `RecordedEventSource`.

## 2.6.7 Distributed Protocol Handler (A2-P6)

> [UPDATED: A2-P6: v1 free function `handle_peer_protocol_message`; v2 abstract class deferred]

v1 ships a **free function** as the sole distributed-protocol entry point, registered in a static `O(1)` table keyed by `MessageType`:

```cpp
// distributed/protocol.hpp (v1)
namespace pypto::distributed {

// Free-function handler keyed by MessageType. Top-N builtin short-circuit +
// indexed load + indirect branch; no virtual dispatch in v1.
using DistributedMessageHandler = void (*)(NodeId peer,
                                           std::span<const std::byte> payload,
                                           const ProtocolContext& ctx);

void register_handler(MessageType type, DistributedMessageHandler h);
void handle_peer_protocol_message(NodeId peer,
                                  MessageType type,
                                  std::span<const std::byte> payload,
                                  const ProtocolContext& ctx);

} // namespace pypto::distributed
```

The speculative abstract class `IDistributedProtocolHandler` is **deferred**: it is declared as a free function only until a second backend exists, at which point ADR records the adoption trigger. CRTP / `final` devirtualization covers the single v1 implementation. The header-independence invariant **I-DIST-1** (ADR-015) enforces that `distributed/` headers are non-includable from `transport/`; see `07-cross-cutting-concerns.md` for the IWYU-CI lint and `modules/distributed.md §3.2` for the registry.

## 2.6.8 Async-Policy Extension Seam (A2-P7)

> [UPDATED: A2-P7: Q-record only — no v1 interface]

No v1 interface is shipped for async-submit / async-policy. The extension axes are captured in Q15 (`09-open-questions.md`): (1) async-policy (Q11), (2) transport-capability semantics (RDMA `rkey` vs TCP TLS binding), and (3) async-submit return path. Any future interface landing here MUST be co-voted with A3/A5/A9 and recorded as a Phase-5 re-vote trigger.
