# 2.1.3 Scheduler

> Part of the [Logical View](../02-logical-view.md). This module describes the per-Layer Scheduler: its internal decomposition into TaskManager, WorkerManager, and ResourceManager sub-components; the event-driven execution model that drives state machine transitions; and the deployment flexibility that lets the Scheduler share or own physical execution units.

A Scheduler is the per-Layer component responsible for task lifecycle, worker dispatch, resource coordination, and completion propagation. Every Scheduler implements the `ISchedulerLayer` interface ([§2.6.1](09-interfaces.md#261-ischedulerlayer)) and is structured as an **event loop** ([§2.1.3.5](#2135-event-driven-execution-model)) that retrieves and processes `SchedulerEvent`s to drive state machine transitions. Scheduler thread count is configurable via `LevelParams.scheduler_thread_count` (default 1). The mapping of the Scheduler's logical role to physical execution units is a deployment decision ([§2.1.3.6](#2136-schedulerworker-deployment-flexibility)).

Internally, each Scheduler is decomposed into three collaborating sub-components:

```
ISchedulerLayer (external interface — unchanged, see ADR-003)
├── TaskManager        ← task state machine, dependency DAG, slot pool, scopes
├── WorkerManager      ← worker state machine, worker pool, availability tracking
└── ResourceManager    ← resource coordination: queries IMemoryManager + WorkerManager
```

## 2.1.3.1 TaskManager

The **TaskManager** owns the task state machine (FREE through RETIRED), the task slot pool, the dependency DAG, parent-child relationship tracking, and scope management. It is the primary driver of task lifecycle within a Layer.

Responsibilities:
1. **Admit** Submissions ([§2.4.A](07-task-model.md#24a-submission-model)) under the **Outstanding Submission Window** ([§2.1.3.1.A](#2131a-submission-admission--outstanding-window)) and allocate task slots for every Task they contain atomically.
2. **Resolve** each Submission's `dep_mode` ([§2.1.3.1.B](#2131b-dependency-resolution-per-submission)) into concrete external dependency edges; **install** intra-Submission pre-built edges (`intra_edges`) for Group Submissions.
3. **Track** dependencies (fan-in counters) and readiness.
4. **Request** resource allocation from the ResourceManager when a Task becomes DEP_READY. For Group Submissions, request the **Group Workspace** ([§2.4.D](07-task-model.md#24d-group-workspace-memory)) at admission so that non-boundary tensor storage is reserved in one allocation.
5. **Dispatch** RESOURCE_READY Tasks to the WorkerManager.
6. **Process** completions: notify dependents, decrement parent pending-child counters, manage scope exits.
7. **Retire** Tasks: release resources, flush profiling, recycle slots.
8. **Retire Submissions**: when every Task belonging to a Submission reaches RETIRED, emit `SUBMISSION_RETIRED` ([§2.1.3.5](#2135-event-driven-execution-model)) to free the Submission's window slot and, for Group Submissions, free the Group Workspace.

The task slot pool and dependency DAG state are allocated from the Layer's **control-plane Memory Region** via `IMemoryManager::alloc_task_slot` with a `PlacementHint` targeting the control region; see [§2.1.8 Access-Efficiency Considerations](04-memory.md#218-access-efficiency-considerations). At the Chip level this region has `backing = ON_CHIP_SRAM`, which is what enables sub-μs Chip → Core dispatch. Per-Task atomic ref-counters and fan-in counters are allocated with `isolate_cache_line = true` so the memory engine does not serialize concurrent atomic updates on the same cache line.

At decision points, TaskManager invokes an `ITaskSchedulePolicy` ([§2.6.3](09-interfaces.md#263-schedule-policy-interfaces)):
- **Ready queue ordering**: which DEP_READY task to evaluate first. The default ordering prefers tasks belonging to earlier Submissions (lower `submission_id`) to enforce the **earliest-first completion bias** required by the Outstanding Submission Window ([§2.1.3.1.A](#2131a-submission-admission--outstanding-window)); custom policies may override this.
- **Preemption**: whether a higher-priority task should preempt.
- **Observation**: notification on every task state change (for monitoring/logging policies).

### 2.1.3.1.A Submission Admission & Outstanding Window

The TaskManager maintains a per-Layer **Outstanding Submission Window** that bounds how many Submissions are in-flight at once. The bound is a Submission count — **not** a task count — so a Group Submission occupies exactly one slot regardless of the number of tasks it contains.

**Definitions**
- **Outstanding**: a Submission is outstanding from admission until every one of its tasks reaches RETIRED (at which point the TaskManager emits `SUBMISSION_RETIRED`).
- **`max_outstanding_submissions`**: per-Layer threshold, configured in `LevelParams`. Default is deployment-specific (typical values 8–64 at Host level, 2–16 at Device/Chip levels).
- **Admission queue**: FIFO of Submissions that arrived while the window was full.

**Admission rule**

```
on submit(SubmissionDescriptor S):
    if outstanding_count < max_outstanding_submissions:
        admit(S)                                       // fast path
    else:
        enqueue(S, admission_queue)                    // blocked

on SUBMISSION_RETIRED(S_retired):
    outstanding_count -= 1
    while outstanding_count < max_outstanding_submissions
          and admission_queue not empty:
        admit(dequeue(admission_queue))
```

**`admit(S)`** is the atomic step:
1. Allocate Task slots for every `TaskDescriptor` in `S.tasks`.
2. Install all `intra_edges` between those slots.
3. Resolve `S.dep_mode` and attach external edges only to `S.boundary_in_tasks` ([§2.1.3.1.B](#2131b-dependency-resolution-per-submission)).
4. If `S.workspace_request` is set, request the group workspace from the ResourceManager ([§2.4.D](07-task-model.md#24d-group-workspace-memory)).
5. Assign `S.submission_id` (monotonic per-Layer) and insert `S` into the in-flight Submission registry.
6. Increment `outstanding_count`.

If any of steps 1–4 fails (slot pool full, workspace allocation rejected, cyclic `intra_edges`, etc.), the admission is rolled back as a whole: slots are released, partial edges discarded, workspace freed, and the Submission is re-queued or rejected per `IResourceAllocationPolicy.should_admit` ([§2.6.3](09-interfaces.md#263-schedule-policy-interfaces)).

> [UPDATED: A1-P8: admission queue sizing + diagnostic sub-codes]
> The `Admission queue` is a bounded SPSC ring of `SubmissionDescriptor*` with capacity `max_admission_backlog` (default 64). Overflow returns `AdmissionStatus::REJECT(Exhaustion)` (see [§2.6.1](09-interfaces.md#261-ischedulerlayer)) with sub-code `WouldBlock`. The `ReadyQueue` is an intrusive min-heap over Task slot indices sized `max_outstanding_submissions × expected_tasks_per_submission × 2`, placed in the CONTROL region with `isolate_cache_line=true`. Exhaustion of either structure surfaces diagnostic sub-codes `TaskSlotExhausted` / `ReadyQueueExhausted`, routed via the A3-P7 precondition catalog.

> [UPDATED: A3-P7: admission-path preconditions + SPMD sub-codes]
> Before step 1 of `admit(S)`, the TaskManager validates `SubmissionDescriptor` preconditions: `tasks.size() ≥ 1`; `intra_edges[i].producer ≠ consumer`; indices in `[0, tasks.size())`; boundary subsets valid; workspace subrange bounds. Violations surface as `AdmissionStatus::REJECT(Validation)` with sub-codes `EmptyTasks / SelfLoopEdge / EdgeIndexOOB / BoundaryIndexOOB / WorkspaceSubrangeOOB / WorkspaceSizeMismatch`. SPMD submissions additionally validate `SpmdIndexOOB`, `SpmdSizeMismatch`, `SpmdBlockDimZero`. The precondition catalog is the single source of truth for admission error mapping and is shared with `09-interfaces.md §2.6.1.A`.

> [UPDATED: A3-P8: cyclic `intra_edges` detection (debug / FE_VALIDATED)]
> Step 2 of `admit(S)` performs a white/grey/black DFS over `intra_edges` to reject cycles; cost O(|tasks| + |intra_edges|), ≤ 50 ns per edge amortized. Release builds skip this check when `SubmissionDescriptor.flags & FE_VALIDATED` is set (frontend has pre-validated); debug builds always run. Cycles raise `AdmissionStatus::REJECT(Validation)` with sub-code `CyclicDependency`.

> [UPDATED: A3-P15: NONE-mode hidden-dependency debug cross-check]
> In debug builds, admission under `dep_mode = NONE` performs a dry-run `DATA`-style lookup against `producer_index`; any hit is logged at `WARN` with `NoneModeHiddenDependency producer=... consumer=... buffer=...` but is not enforced. Release builds skip this check. See [§7.2 Cross-Cutting Concerns](../07-cross-cutting-concerns.md) for the debug observability contract.

> [UPDATED: A5-P8: admission-pressure degradation policy]
> Admission-side back-pressure is parameterized by `admission_pressure_policy ∈ {REJECT, COALESCE, DEFER}` with `max_deferred` bound to prevent memory exhaust. `REJECT` is the default and corresponds to the behavior above. `COALESCE` folds duplicate `REMOTE_SUBMIT` retries using the dedup key. `DEFER` stages the Submission in a bounded deferred queue sized by `max_deferred` and re-admits on `SUBMISSION_RETIRED`. Each policy value is tied to an A8-P5 `AlertRule`; see `modules/scheduler.md` for rule bindings.

**Earliest-first completion bias**

The window is most useful when the earliest outstanding Submission is actively driven to completion, because retirement of the earliest Submission is what unblocks the next admission. To support this, the default `ITaskSchedulePolicy` sorts DEP_READY tasks by ascending `submission_id` (ties broken by ready-queue arrival order). At decision points that would normally be symmetric (ready-queue draining, worker allocation among equivalent candidates, resource contention between equal-priority tasks), the Scheduler consistently prefers tasks from the earliest outstanding Submission. Custom policies are free to relax this bias but should preserve a monotone tendency — without it, the window can degenerate into head-of-line blocking.

**Accounting**
- A `SUBMISSION_RETIRED` event is emitted exactly once per Submission, when the last of its tasks reaches RETIRED.
- The ResourceManager's `GreedyResourceAllocationPolicy` exposes `outstanding_submissions` and `admission_queue_depth` in `ResourceSnapshot` so policies can observe window pressure.
- Per-Submission state (`submission_id`, task-remaining counter, workspace handle, dep_mode, boundary mask) is allocated from the Layer's control-plane Memory Region with the same placement rules as Task slots ([§2.1.8](04-memory.md#218-access-efficiency-considerations)).

### 2.1.3.1.B Dependency Resolution per Submission

On admission, each Submission's `dep_mode` is translated by the TaskManager into a concrete set of external edges attached to `boundary_in_tasks`. Intra-group edges (`intra_edges`) are always installed verbatim regardless of mode.

| Mode | TaskManager Action | Cost |
|------|--------------------|------|
| `BARRIER` | Snapshot the set `P` of all Tasks in all currently outstanding Submissions other than `S`. For each `b ∈ S.boundary_in_tasks`, install fan-in edges `P → b` (optimized as a single synthetic "barrier token" per prior Submission that counts down as that Submission completes). | O(W × B) edges, where W is outstanding-window depth and B is boundary-in count; optimized with barrier tokens to O(W + B). |
| `DATA` | For each `b ∈ S.boundary_in_tasks`, iterate `b.args` and for every `IN` / `INOUT` tensor whose `BufferRef` matches an output tensor of a Task in some outstanding Submission, install a producer→consumer edge. Tensor identity is carried by `BufferRef` and the TaskManager maintains a per-Layer `BufferRef → producer_task` index so lookups are O(1) per argument. **RAW-only in v1**: `DATA` mode installs only producer→consumer (RAW) edges; cross-Submission WAR / WAW are not tracked at the runtime boundary and are deferred ([§2.10.6](12-dependency-model.md#2106-scope-limits-v1)). | O(B × A) where A is the average boundary-in argument count. |
| `NONE` | Install no external edges. Intra-group edges still apply. | O(1). |

**Producer index.** The TaskManager maintains a hash map `producer_index : BufferRef → (submission_id, task_handle)` updated on Task admission (for each `OUT` / `INOUT` argument) and cleared on Submission retirement. `DATA`-mode admission consults this map; `BARRIER` does not need it (it uses Submission-level completion counters); `NONE` skips it entirely.

> [UPDATED: A1-P1 + A1-P14 + A10-P10: pre-size, CONTROL placement, capacity formula]
> `producer_index` is pre-allocated at Layer init with `capacity = max_outstanding_submissions × expected_outputs_per_submission × 2` (mirrored as the `producer_index_capacity` config field) and placed in the Layer's **CONTROL region** (CONTROL_AREA at Chip/Device; CONTROL_HEAP at Host) with `isolate_cache_line=true`. **Rehash on the hot path is forbidden.** Overflow uses bounded open-addressed probing (load factor L < 0.5, probe length ≤ 2) with cache-line-sized buckets for single-probe average; probe exhaustion returns `ErrorCode::ResourceExhausted` and triggers `IResourceAllocationPolicy.should_admit` back-pressure. A runtime assertion fires on occupancy overflow in debug builds.

**Producer-index lifecycle (normative).** The `producer_index` is the only cross-Submission dependency state the runtime owns. Its lifecycle is:

1. **Insertion (at admission).** For each Task admitted as part of `S`, for each `OUT` / `INOUT` argument, upsert `producer_index[BufferRef] = (S.submission_id, task_handle)`. Most-recent-writer wins; this is safe within a single Submission because intra-Submission WAW hazards are already serialized by frontend-supplied `intra_edges` ([§2.10.2](12-dependency-model.md#2102-frontend-contract)).
2. **Lookup (at admission of a later Submission).** `DATA`-mode resolution walks `boundary_in_tasks` and consults `producer_index` by `BufferRef`. A hit whose `(slot_index, generation)` no longer matches a live Task is treated as stale and discarded (generation guard, [§2.4.2](07-task-model.md#242-task-key-and-handle)).
3. **Eviction (on `SUBMISSION_RETIRED`).** When `S` retires, TaskManager walks `S`'s recorded `OUT` / `INOUT` `BufferRef`s and removes each `producer_index` entry whose stored `(submission_id, task_handle)` still identifies a Task in `S`. Entries already overwritten by a later Submission (Step 1 of that Submission) are left untouched — the newer writer is now the authoritative producer.

This eviction rule keeps the `producer_index` size bounded by the set of `OUT` / `INOUT` `BufferRef`s produced by currently-outstanding Submissions, and avoids per-buffer ref-count tracking. See [§2.10.3](12-dependency-model.md#2103-runtime-construction--cross-submission-data-mode) and [§2.10.4](12-dependency-model.md#2104-dependency-maintenance) for the full contract.

**RAW-only scope.** The single-valued `producer_index` inherently expresses only producer → consumer (RAW) edges. Modeling cross-Submission WAR / WAW would require a per-`BufferRef` version list (producer + consumer sets, as in [`tensor-dependency.md` §3.2](../../../tensor-dependency.md)); this is deferred ([09-open-questions.md](../09-open-questions.md)). Soundness with a single producer relies on the **non-aliasing intermediate-memref invariant** ([§2.1.6](04-memory.md#216-memory-manager), [§2.10.5](12-dependency-model.md#2105-invariants)): if two `BufferRef`s do not alias, there is no WAR/WAW hazard for the runtime to track.

**Non-boundary inputs.** Because non-boundary tasks, by construction, consume only tensors produced by other tasks **in the same Submission**, they never participate in external edge resolution regardless of mode. The DATA-mode scan is therefore bounded by `B` (boundary-in count), not by `|S.tasks|`.

**Policy hook.** `IResourceAllocationPolicy.should_admit(submission, ResourceSnapshot)` is consulted before the admission performs slot allocation; policies may reject submissions under resource pressure (e.g., workspace bytes unavailable), request back-pressure, or gate by `dep_mode` (e.g., reject `NONE` submissions above a watermark).

**Concurrency and locking.** `DATA`-mode admission is the only dep-mode path that reads and writes the runtime's cross-Submission dependency metadata (`producer_index` and the fan-in counters of the boundary-in Tasks being edged into) concurrently with completion handlers that are mutating the *same* metadata for prior outstanding Submissions (decrementing fan-in counters as predecessors retire, evicting their `producer_index` entries). The three modes therefore have different synchronization obligations on the admission critical path:

| Mode | Shared state touched at admission | Synchronization required |
|------|-----------------------------------|--------------------------|
| `BARRIER` | Outstanding-Submission set and the Submission-level completion counter only (no per-task dep metadata). | Atomic snapshot of the outstanding-Submission list (or a single barrier-token allocation) — **no dep-metadata lock**. Completion handlers of prior Submissions decrement the shared barrier token via an atomic RMW; they do not contend with admission on per-task structures. |
| `DATA` | `producer_index` hash map (lookup per boundary-in argument; upsert per `OUT`/`INOUT` argument) **and** per-Task fan-in counters on the newly admitted boundary-in Tasks (increment per external edge). Concurrently, prior tasks' completion handlers decrement producer-index entries and fan-in counters of their successors. | **Dep-metadata lock required.** A per-Layer reader/writer lock (or an equivalent fine-grained scheme — see below) protects `producer_index`; per-Task fan-in counters are atomic RMWs. The lock is held only for the duration of external edge construction for one Submission. Retirement-driven eviction takes the same lock in write mode. |
| `NONE` | None (no external edges are installed; `intra_edges` are installed on freshly allocated slots that are not yet visible to any other handler). | **No lock required.** Intra-group edges target slots that were just allocated atomically in step 2 of admission ([§2.1.3.1.A](#2131a-submission-admission--outstanding-window)); no concurrent reader can observe them until admission publishes the Submission. |

The lock is **not** the event-loop serialization. Under the default single-threaded Stage B deployment ([§2.1.3.5](#2135-event-driven-execution-model)), admission and completion handlers are naturally serialized by the event loop and the lock degenerates to an uncontended acquire/release. When `scheduler_thread_count > 1` (multi-threaded Stage B), or when the `submit(...)` entry point performs admission inline on the caller's thread rather than dispatching through the event loop, the lock is what keeps `producer_index` and fan-in counters consistent. The scheme MUST admit:

1. **Shared reads during `DATA` lookup** — multiple admissions performing only lookups may proceed concurrently as long as no eviction is in flight.
2. **Exclusive writes during upsert and eviction** — upsert on admission and removal on `SUBMISSION_RETIRED` are serialized against each other and against concurrent lookups.
3. **Lock-free fan-in decrement** — completion handlers decrement successor fan-in counters via atomic RMW and do not take the dep-metadata lock; the admission-side increment is performed while the lock is held so that no completion can observe a half-built edge set.

Implementations MAY partition `producer_index` by `BufferRef` hash to reduce contention (each shard has its own RW lock); the contract above holds per shard. The `generation` guard in `TaskHandle` ([§2.4.2](07-task-model.md#242-task-key-and-handle)) remains the correctness backstop for any lookup that slips past eviction — it is an independent mechanism from the dep-metadata lock and covers sequences such as "eviction happened between a `DATA` lookup and the edge install."

> [UPDATED: A1-P2 + A10-P1 + A10-P7: bounded lock-hold + shard count `LevelParam` + default/deployment cue]
> Shard count is exposed as `LevelParams.producer_index_shards` (alias `admission_shards`). **Default = 1** at Chip level and below (preserves today's single-thread fast path); default = 4 at Device level and 8 at Host level; default formula `max(1, scheduler_thread_count)` when `scheduler_thread_count > 1`. The opt-in multi-shard path is triggered by the deployment cue recorded in `05-physical-view.md` (ADR-019): `concurrent_submitters × cluster_nodes ≥ 64`. Lock-hold budgets are bounded: shared-mode admission hold ≤ `B × A` probes (≤ 500 ns for B=4, A=4); exclusive-mode hold bounded by `B × A` fan-in RMWs + `|S.OUT ∪ S.INOUT|` removals; p99 reader ≤ 3 μs, p99 writer ≤ 10 μs per shard under 8 shards. See ADR-019 "Admission shard default + deployment cue".

The contract is summarized by mode in [§2.4.B](07-task-model.md#24b-dependency-modes) and referenced from [§2.10.4](12-dependency-model.md#2104-dependency-maintenance).

> [UPDATED: A3-P4: producer-failure → consumer propagation (incl. SPMD aggregation)]
> On `TASK_FAILED(producer)`, TaskManager walks the producer's precomputed **successor list** — stored inline in the Task cold tail (see `07-task-model.md §2.4.1` cold-field split under A1-P4) — and emits `DEP_FAILED(producer_key, ErrorCode)` for each successor. Consumers in `PENDING` transition to `ERROR` (per `07-task-model.md §2.4.4`); propagation is recursive. For SPMD Submissions, the parent task is notified on **any** sub-task failure: the SPMD aggregation edge participates in successor-walk inlining, so a single failing rank fails the aggregated result. The successor walk itself is on the cold path (no hot-path allocation); precomputation occurs during admission as part of edge construction.

## 2.1.3.2 WorkerManager

The **WorkerManager** owns the Worker state machine ([§2.1.4.1](03-worker.md#2141-worker-state-machine)), the worker pool, Worker Types ([§2.1.4.2](03-worker.md#2142-heterogeneous-worker-types-and-worker-groups)), Worker Groups ([§2.1.4.2](03-worker.md#2142-heterogeneous-worker-types-and-worker-groups)), and worker availability tracking. It receives dispatch requests from the TaskManager and assigns Tasks to available Workers, respecting heterogeneous type constraints and group affinity rules.

Responsibilities:
1. **Manage** the worker pool: track each Worker's current state, Worker Type, and Worker Group membership.
2. **Track per-group availability**: maintain an availability index for each Worker Group — which members are IDLE, EXECUTING, etc. — enabling fast evaluation of whether a group can satisfy a `TaskExecType`'s `required_slots`.
3. **Select** Workers for a dispatched Task (via `IWorkerSelectionPolicy`, [§2.6.3](09-interfaces.md#263-schedule-policy-interfaces)). For single-worker Tasks, return a single Worker ID. For multi-worker Tasks with `requires_worker_group = true`, return a **set** of Worker IDs from the same Worker Group.
4. **Enforce the Resource Exclusion Rule** ([§2.1.4.2](03-worker.md#2142-heterogeneous-worker-types-and-worker-groups)): when a Worker is allocated, update the group's available capacity; when a Worker is released, restore it.
5. **Dispatch** the Task to the selected Worker(s) via the appropriate mechanism (register write, shared memory, DMA, network message).
6. **Track** Worker execution: detect completion, failure, and timeout.
7. **Report** Worker state changes back to TaskManager (completion triggers task state transitions) and update per-group availability.
8. **Handle** Worker failures: transition failed Workers through RECOVERING or to UNAVAILABLE, updating group availability accordingly.

At decision points, WorkerManager invokes an `IWorkerSelectionPolicy` ([§2.6.3](09-interfaces.md#263-schedule-policy-interfaces)):
- **Worker selection**: which Worker(s) to assign for a given Task, given its `TaskExecType` and the current per-group availability.
- **Observation**: notification on every Worker state change (including group-level derived state changes).

## 2.1.3.3 ResourceManager

The **ResourceManager** is a coordinator that decides when a Task has sufficient resources to proceed. It does **not** own memory allocation — it queries the `IMemoryManager` (from the `memory/` module) for buffer availability and the WorkerManager for Worker availability.

Responsibilities:
1. **Evaluate** resource readiness: when TaskManager signals a Task is DEP_READY, ResourceManager checks whether memory (buffers, task slots) and Workers are available.
2. **Allocate** resources: request buffer allocation from `IMemoryManager`, reserve a Worker via WorkerManager.
3. **Transition** the Task to RESOURCE_READY and notify TaskManager.
4. **Apply back-pressure**: when resources are insufficient, hold the Task in DEP_READY until resources become available or reject the submission.
5. **Release** resources on Task retirement (delegate to `IMemoryManager` and WorkerManager).

At decision points, ResourceManager invokes an `IResourceAllocationPolicy` ([§2.6.3](09-interfaces.md#263-schedule-policy-interfaces)):
- **Allocation strategy**: how to allocate memory and workers jointly (greedy, batched, priority-aware).
- **Back-pressure**: when to reject new submissions vs. queue them.
- **Observation**: notification on resource state changes.

## 2.1.3.4 Sub-Component Interaction

The internal flow for a Submission (and its Tasks) from submission to dispatch:

```
submit(SubmissionDescriptor)
  │
  ▼
TaskManager: check Outstanding Window (§2.1.3.1.A)
  │   ├── window full → enqueue on admission queue; wait for SUBMISSION_RETIRED
  │   └── window has a slot:
  │        admit atomically: alloc slots for all S.tasks,
  │        install S.intra_edges,
  │        resolve S.dep_mode → external edges on S.boundary_in_tasks,
  │        (optional) alloc Group Workspace for non-boundary tensors
  │
  ▼
TaskManager: each task → SUBMITTED → register deps → PENDING
  │                                           │
  │                          deps satisfied ◄──┘
  │                                │
  │                     ITaskSchedulePolicy.rank_ready_tasks()
  │                      (default: earliest submission_id first)
  │                                │
  ▼                                ▼
TaskManager: Task DEP_READY → request resources from ResourceManager
  │
  ▼
ResourceManager: query IMemoryManager + WorkerManager
  │               IResourceAllocationPolicy.allocate_resources()
  │
  ▼ (resources available)
TaskManager: Task RESOURCE_READY → dispatch to WorkerManager
  │
  ▼
WorkerManager: IWorkerSelectionPolicy.select_worker()
  │              dispatch to selected Worker → Task DISPATCHED
  │
  ▼ (Worker completes)
WorkerManager: report completion → TaskManager
  │
  ▼
TaskManager: Task COMPLETED → notify dependents → RETIRED
  │
  │  (last task of its Submission?)
  ▼
TaskManager: emit SUBMISSION_RETIRED → free Group Workspace,
             decrement outstanding_count, drain admission queue
```

> [UPDATED: A5-P6: scheduler-thread watchdog / deadman timer (elision cross-ref)]
> A monotonic timestamp is written to a deadman slot at the top of each event-loop cycle. A low-priority monitor thread checks at `scheduler_watchdog_ms` (default 250 ms) and raises `SchedulerStalled` when the delta exceeds budget. The deadman store is normatively **elided into the per-cycle TSC capture** (skip > K=16 cycles on burst, ≤ 1.6 ms < 250 ms watchdog); see `modules/scheduler.md §5` for the normative specification and A1-handoff budget.

## 2.1.3.5 Event-Driven Execution Model

The Scheduler's state machines (Task and Worker) are driven by **events**. An event represents a discrete occurrence that triggers one or more state transitions. The Scheduler is structured as an **event loop** that retrieves and processes events sequentially.

### Scheduler Event

```cpp
struct SchedulerEvent {
    enum Type {
        SUBMISSION_ADMITTED,   // a Submission cleared the outstanding window and was admitted
        SUBMISSION_RETIRED,    // last task of a Submission reached RETIRED; window slot freed
        TASK_SUBMITTED,        // new task accepted (within an admitted Submission)
        DEP_SATISFIED,         // dependency edge resolved
        RESOURCE_AVAILABLE,    // memory or worker became available
        WORKER_COMPLETED,      // worker finished executing a task
        WORKER_FAILED,         // worker encountered a fault
        CHILD_COMPLETED,       // child-layer task completed (from Vertical Channel)
        REMOTE_COMPLETED,      // remote task completed (from Horizontal Channel)
        TIMER_EXPIRED,         // timeout or periodic tick
        SCOPE_EXIT,            // scope boundary reached
        DRAIN_REQUESTED,       // drain() called — flush all pending work
    };
    Type        type;
    uint32_t    payload_id;    // task handle, worker ID, submission handle, or channel message ID
    uint64_t    timestamp;     // monotonic timestamp for ordering and profiling
};
```

### Event Sources and Delivery Modes

Events originate from two categories of sources:

1. **Internal sources** — generated within the Scheduler itself (e.g., a state transition handler produces a follow-on event such as DEP_SATISFIED after WORKER_COMPLETED).
2. **External sources** — generated outside the Scheduler (e.g., a Worker reporting completion, a parent-layer Worker submitting a task, a Vertical/Horizontal Channel delivering a message).

The mechanism for delivering events to the Scheduler is **configurable per source** via `IEventSource` ([§2.6.4](09-interfaces.md#264-event-and-execution-interfaces)):

| Delivery Mode | Mechanism | When to Use |
|---------------|-----------|-------------|
| **Queue** | Producer pushes event into a bounded queue (`IEventQueue`); the Scheduler's event loop dequeues. | External sources where the producer and Scheduler run on different execution contexts (threads, cores, nodes). Default for Worker completion, Channel messages. |
| **Poll** | The Scheduler's event loop calls `IEventSource.poll()` to check for events. | Hardware-level sources where the producer signals via registers or flags (e.g., AICore COND register, DMA completion flag). Avoids interrupt overhead. |

Each Machine Level registers its event sources at construction time. The Scheduler's event loop iterates over all registered sources every cycle.

### Event Handling Modes

When the Scheduler retrieves an event, the corresponding handler can be invoked in two modes, **configurable per event type**:

| Handling Mode | Behavior | When to Use |
|---------------|----------|-------------|
| **Inline** | Handler executes immediately within the event loop iteration. Follow-on events produced by the handler are available for processing in subsequent iterations. | Low-latency critical path events (e.g., DEP_SATISFIED → dispatch chain). Default for internal events. |
| **Deferred** | Event is placed in an internal **pending queue**. The Scheduler processes the pending queue at a policy-determined point (e.g., after draining all external events, or at fixed intervals). | Batching opportunities (e.g., accumulate multiple DEP_SATISFIED events before running the ready-queue ranking policy). Default for non-critical-path events. |

The choice is configured per event type per Machine Level via `EventHandlingConfig` in `LevelParams`.

### Scheduler Event Loop

The Scheduler event loop is defined as a composition of **three logical stages** connected by well-typed internal queues. Each stage has a pluggable policy, and the mapping of stages to physical execution units is a deployment decision — stages may share a thread (default) or run on separate threads/cores that are connected via the internal queues.

```
   ┌─────────────────────────────┐     inline_dispatch_queue    ┌─────────────────────────────┐
   │ Stage A: Collection         │  ──────────────────────────► │ Stage B: Classify & Inline  │
   │ (poll/dequeue from sources) │                              │ (run inline handlers,       │
   │  • IEventCollectionPolicy   │                              │   emit deferred events)     │
   └─────────────────────────────┘                              └──────────────┬──────────────┘
                                                                               │
                                                              deferred_queue   ▼
                                                                 ┌─────────────────────────────┐
                                                                 │ Stage C: Deferred Drain     │
                                                                 │ (batch-process, invoke      │
                                                                 │   deferred handlers)        │
                                                                 └─────────────────────────────┘
```

#### Stage A — Collection

Stage A retrieves events from all registered `IEventSource`s and hands them to Stage B through an **inline dispatch queue**. Collection behavior is not hard-coded; it is driven by an `IEventCollectionPolicy` ([§2.6.4](09-interfaces.md#264-event-and-execution-interfaces)) configured via a `SourceCollectionConfig` per Machine Level:

| Knob | Meaning |
|------|---------|
| **Source ordering** | Explicit ordered list of source IDs, or a strategy tag: `FIXED_PRIORITY`, `ROUND_ROBIN`, `WEIGHTED_ROUND_ROBIN`, `CUSTOM` (delegates ordering to the policy object). |
| **Per-source max per cycle** | Upper bound on the number of events drained from a single source in one collection pass. Prevents a high-rate source (e.g., Worker completions) from starving lower-rate sources (e.g., Channel messages) or delaying the deferred drain. |
| **Per-source weight** | Integer weight used when the strategy is `WEIGHTED_ROUND_ROBIN`. |
| **Cycle budget** | Total max events collected per cycle across all sources (optional hard cap that bounds loop latency). |
| **Empty-source backoff** | Skip / deprioritize / exponentially back off a source that returned no events recently, to reduce polling overhead. |

The policy returns, on each invocation, the next `(source, max_events)` pair to drain; the collection stage loops until the policy returns `DONE` for the cycle. This makes source handling declarative and avoids re-writing the loop for every scheduling heuristic.

#### Stage B — Classify & Inline Handling

Stage B consumes events from the inline dispatch queue (or receives them directly from Stage A when the two stages share a thread). For each event it looks up the handling mode from `EventHandlingConfig`:

- **Inline** events are handled immediately; any follow-on events the handler produces are re-entered into the appropriate queue (internal inline events typically short-circuit back into the inline path; events to other stages go through the normal queues).
- **Deferred** events are enqueued into the **deferred queue** and are not run here.

Stage B is the only stage that mutates Scheduler state machines (Task/Worker/Resource). Stages A and C do not hold state-machine locks; they only move events across queue boundaries.

#### Stage C — Deferred Drain

Stage C drains the deferred queue according to the configured batching policy (drain fully, drain up to N events, drain at fixed intervals, drain when Stage B idles, etc.). Handlers invoked here may, in turn, produce inline follow-on events that are pushed back to Stage B.

#### Stage Deployment

Stage-to-execution-unit mapping is expressed as an `EventLoopDeploymentConfig` ([§2.6.4](09-interfaces.md#264-event-and-execution-interfaces)), independent of the stage logic itself:

| Deployment | Stages A / B / C mapping | Typical use |
|------------|--------------------------|-------------|
| **Single-threaded** (default) | One thread runs A → B → C sequentially; the internal queues degenerate to direct handoffs. | Latency-critical or resource-constrained levels (e.g., Core level on AICPU). |
| **Split collection** | A on a dedicated thread/core; B + C on the Scheduler thread. Handoff via the inline dispatch queue (MPSC). | Host/Device level when many external producers (Workers, Channels, NICs) would otherwise force B to spin polling. |
| **Split deferred drain** | A + B on the Scheduler thread; C on its own thread. Handoff via the deferred queue. | When deferred handlers are coarse-grained (e.g., ready-queue re-ranking across many tasks) and should not block the inline critical path. |
| **Fully split** | A, B, and C each on their own execution unit, connected by queues. | Distributed/root levels where collection, decision-making, and batched policy work have different SLOs. |

Constraints:
- Stages must still behave **as if** executed in A → B → C order for a given event; the internal queues preserve per-source FIFO for correctness-sensitive sequences (e.g., `DEP_SATISFIED` must observe the `WORKER_COMPLETED` that caused it).
- The **state-machine mutation invariant** holds regardless of deployment: only Stage B mutates Task/Worker/Resource state, so no additional locks are introduced when A or C are split out.
- Queue types are selected to match the deployment (`SPSC` when one stage feeds another, `MPSC` when multiple producers feed Stage A or B).

#### Reference Pseudocode

```
// Stage A — Collection (bindable to its own execution unit)
loop {
    while (src, max_n) = collection_policy.next():   // DONE ends the cycle
        drained = 0
        if src.mode == QUEUE:
            while drained < max_n and (e = src.queue.try_dequeue()):
                inline_dispatch_queue.push(e); drained += 1
        else: // POLL
            while drained < max_n and src.poll(&e):
                inline_dispatch_queue.push(e); drained += 1
        collection_policy.report(src, drained)
    if nothing_collected: yield_policy.wait()
}

// Stage B — Classify & Inline Handling (Scheduler state owner)
loop {
    while e = inline_dispatch_queue.try_dequeue():
        mode = event_handling_config.mode_for(e.type)
        if mode == INLINE: invoke_handler(e)          // mutates state machines
        else:              deferred_queue.push(e)
    if nothing_handled: yield_policy.wait()
}

// Stage C — Deferred Drain (bindable to its own execution unit)
loop {
    batch = deferred_drain_policy.next_batch(deferred_queue)
    for e in batch: invoke_handler(e)                 // may push back to inline
    if batch.empty(): yield_policy.wait()
}
```

When stages share a thread, the three `loop { … }` bodies are composed into a single iteration and the queues become direct function calls, yielding the original simple loop as a special case.

The `inline_dispatch_queue` and `deferred_queue` shown above are backed by buffers allocated from the Layer's control-plane Memory Region. When Stages A / B / C are deployed on different execution units (split modes in the table above), each queue's head and tail pointers are allocated with `PlacementHint.isolate_cache_line = true` so producer and consumer do not repeatedly invalidate each other's cache lines. Cache-line size is read from the region descriptor rather than hard-coded — see [§2.1.8 Access-Efficiency Considerations](04-memory.md#218-access-efficiency-considerations).

This composition is the Scheduler's **sole execution entry point**. All state machine transitions are driven through Stage B — there are no direct method calls into the Scheduler from external threads (external callers enqueue events into Stage A sources or set hardware flags that Stage A polls).

> [UPDATED: A9-P2: closed `PolicyKind` / `DeploymentMode` enums; single `IEventLoopDriver` test seam]
> Deployment modes and policy pluggability are **closed** for v1: `enum DeploymentMode { SINGLE_THREADED, SPLIT_COLLECTION }` (defer `SPLIT_DEFERRED`, `FULLY_SPLIT` to appendix-listed future extensions in ADR-010). `IEventCollectionPolicy` and `IExecutionPolicy` pluggability is replaced with a closed `enum PolicyKind` that selects a built-in implementation. The sole retained interface is a single test-only seam **`IEventLoopDriver`** — gated on `enable_test_driver`, providing `step(max_events)` + `RecordedEventSource` for deterministic replay — see `09-interfaces.md §2.6.4` and `modules/scheduler.md §2.5`. Release binaries are unaffected. ADR-010 appendix lists the future-extension interfaces that would be reintroduced if/when a concrete second implementation appears.

> [UPDATED: A1-P5: latency budgets for SPMD fan-out + event-loop stages]
> Per-stage steady-state budgets apply to the event loop defined above and MUST be validated by `modules/profiling.md §2.5` phase timings. **Event-loop iteration:** Stage A ≤ 50 ns/source; Stage B ≤ 100 ns/event; Stage C ≤ 200 ns/event; idle iteration ≤ 300 ns. **SPMD Chip → N × AICore end-to-end ≤ 5 μs** (batch prep ≤ 1 μs; register-bank block write ≤ 2 μs; 108-bit-popcount ACK ≤ 2 μs; a5 108-AICore case uses AND + 2 × 64 `ctzll` fallback — both paths < 10 ns). Cross-reference `04-process-view.md §4.8.6/7` for the end-to-end fan-out budget breakdown.

## 2.1.3.6 Scheduler–Worker Deployment Flexibility

The Scheduler and Worker are **logical roles**, not fixed bindings to physical execution units. A logical role defines a specific job:

- **Scheduler role**: retrieve events, evaluate policies, drive state machine transitions, dispatch tasks.
- **Worker role**: execute a task's function body, report completion.

The mapping from logical roles to physical execution units (OS threads, hardware pipelines, CPU cores) is a **deployment decision**, configurable per Machine Level:

| Deployment Mode | Description | Example |
|-----------------|-------------|---------|
| **Dedicated** | Separate physical execution units for Scheduler and Workers. The Scheduler loop runs on its own thread(s); Workers run on separate thread(s) or hardware units. | Host level: 1 scheduler thread + N worker threads. Core level: AICPU runs scheduler, AICores are dedicated hardware workers. |
| **Interleaved** | A single physical execution unit alternates between Scheduler and Worker roles according to an `IExecutionPolicy`. | Device level (alternative): an AICPU thread runs the scheduler loop, and between event-loop iterations, executes a ready task inline before returning to the loop. |

### IExecutionPolicy

The `IExecutionPolicy` ([§2.6.4](09-interfaces.md#264-event-and-execution-interfaces)) governs how a shared execution unit allocates time between roles:

```cpp
class IExecutionPolicy {
public:
    virtual ~IExecutionPolicy() = default;

    // After processing events, decide what to do next.
    // Returns CONTINUE_SCHEDULING, EXECUTE_TASK, or YIELD.
    enum Action { CONTINUE_SCHEDULING, EXECUTE_TASK, YIELD };
    virtual Action next_action(const SchedulerStats& stats,
                               const TaskHandle* ready_task) = 0;
};
```

**Default implementations:**
- `DedicatedExecutionPolicy`: always returns `CONTINUE_SCHEDULING` (never executes tasks inline; Workers are separate).
- `InterleavedExecutionPolicy`: executes one ready task after each event-loop drain, then returns to scheduling.
- `BatchedExecutionPolicy`: accumulates N ready tasks, dispatches all, then executes them inline before returning.

This separation ensures that the architecture is portable across execution environments: on a device with many hardware cores and a dedicated scheduler core, use Dedicated mode; on a resource-constrained AICPU with limited threads, use Interleaved mode to avoid thread overhead.
