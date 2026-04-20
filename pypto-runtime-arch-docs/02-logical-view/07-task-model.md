# 2.4 Task Model

> Part of the [Logical View](../02-logical-view.md). This module describes the **Task** — the unit of schedulable work: a calling instance of a [Function](06-function-types.md) with specific arguments, dependencies, resource requirements, and a Worker-type requirement expressed via Task Execution Types — and the **Submission** — the unit of admission accepted by a Scheduler, which may wrap one or more Tasks.

A **Task** is the unit of schedulable work — a single invocation of a Function with specific arguments and declared dependencies. A Task is bound to a specific Layer and managed by that Layer's Scheduler.

A **Submission** is the unit of admission: a single call to `ISchedulerLayer.submit(...)` delivers exactly one Submission, which in turn wraps one or more Tasks. Tasks are the scheduling granularity for dispatch and completion; Submissions are the granularity for admission control, dependency-mode resolution, and group-level workspace memory (see [§2.4.A](#24a-submission-model) through [§2.4.D](#24d-group-workspace-memory)).

## 2.4.A Submission Model

A **Submission** is what a caller hands to a Scheduler. One Submission always carries:

- A stable, monotonically increasing `submission_id` assigned by the Scheduler at admission time.
- A **Dependency Mode** (`dep_mode`) declaring how this Submission relates to previously admitted-but-not-retired Submissions on the same Layer ([§2.4.B](#24b-dependency-modes)).
- A set of one or more Tasks and, optionally, pre-built intra-Submission dependency edges among them.
- An optional **Workspace Request** describing the group's non-boundary tensor storage ([§2.4.D](#24d-group-workspace-memory)).

> **Who fills in `intra_edges` and boundary masks?** The **frontend dependency analyzer** (above `bindings/`) is the normative owner of intra-Submission hazard analysis and is responsible for populating `intra_edges`, `boundary_in_tasks`, and `boundary_out_tasks`. See [§2.10 Dependency Model](12-dependency-model.md) for the frontend ↔ runtime contract and [`tensor-dependency.md`](../../../tensor-dependency.md) for the algorithm the frontend runs.

### Submission Kinds

| Kind | Tasks | Intra-Submission Edges | Typical Use |
|------|-------|------------------------|-------------|
| **Single** | Exactly 1 Task | None | Ordinary one-shot launch. |
| **Group** | N Tasks (N ≥ 1) with a pre-built DAG | Arbitrary acyclic edges supplied by caller | Fused operator graphs; op blocks whose producer/consumer structure is known at submission time. |
| **SPMD** | N sub-tasks of the same Function | None (or only replica-broadcast edges) | Data-parallel fan-out; see [§2.4.6](#246-spmd-submission). SPMD is a specialization of Group. |

> [UPDATED: A9-P4: drop `SubmissionDescriptor::Kind` discriminator]
> The `Kind` enum above is a **documentary classification only** and is **not** represented as a struct field on `SubmissionDescriptor`. The `kind` member and `enum Kind` are removed: SPMD-ness is inferred from `spmd.has_value()`; Single = `tasks.size() == 1 && !spmd.has_value()`; Group = `tasks.size() ≥ 1 && !spmd.has_value() && tasks.size() > 1`. The edge case `tasks.size() == 1 && spmd.has_value() && spmd->size == 1` (one-way SPMD) is accepted in the precondition catalog as a valid SPMD-with-size-1 submission. See ADR-017 "Closed-enum-in-hot-path policy" for rationale.

### Submission Fields

| Field | Type | Description |
|-------|------|-------------|
| `submission_id` | `uint64_t` | Monotonic per-Layer identifier; used for admission-order and earliest-first bias. |
| `dep_mode` | `DepMode` | `BARRIER`, `DATA`, or `NONE` — see [§2.4.B](#24b-dependency-modes). |
| `tasks` | `TaskDescriptor[]` | One or more Tasks contained in this Submission. |
| `intra_edges` | `IntraGroupEdge[]` | Producer→consumer edges among `tasks` (empty for Single, optional for Group). Must be acyclic. |
| `boundary_in_tasks` | `bitset` | Tasks in this Submission whose inputs may be produced by **prior** Submissions. |
| `boundary_out_tasks` | `bitset` | Tasks whose outputs may be consumed by **later** Submissions. |
| `workspace_request` | `WorkspaceRequest?` | Optional group workspace sizing/placement ([§2.4.D](#24d-group-workspace-memory)). |

The classification `boundary_in_tasks` / `boundary_out_tasks` is supplied (or computed) at submission time; a task is **non-boundary** when it is neither in `boundary_in_tasks` nor in `boundary_out_tasks` — i.e., all of its inputs come from other tasks in the same Submission and all of its outputs are consumed exclusively within the Submission. Non-boundary tasks are the ones eligible for workspace-carved tensor storage ([§2.4.D](#24d-group-workspace-memory)).

### Admission Contract

- A Submission is admitted **atomically**: either every Task in it is accepted into the Scheduler and every `intra_edges` entry is installed, or the Submission is rejected as a whole (no partial admission).
- Admission is gated by the per-Layer **Outstanding Submission Window** ([§2.1.3.1.A](02-scheduler.md#2131a-submission-admission--outstanding-window)).
<!-- [UPDATED: A4-P2: fix broken anchor — #21_3_1_a-... → #2131a-...] -->

- Once admitted, all Tasks in the Submission follow the standard Task state machine ([§2.4.4](#244-task-state-summary)) independently.
- A Submission is considered **retired** when every one of its Tasks has reached RETIRED; only then does its window slot free up.

## 2.4.B Dependency Modes

The `dep_mode` field of a Submission governs how the Scheduler establishes external dependency edges (i.e., edges from Tasks in this Submission to Tasks in previously admitted Submissions). Intra-Submission edges (`intra_edges`) are unaffected by `dep_mode` — they are always installed as given.

| Mode | External Edge Construction | Runtime Cost | Admission-Time Synchronization | Safety Contract |
|------|----------------------------|--------------|--------------------------------|-----------------|
| **`BARRIER`** | Join every Task in `boundary_in_tasks` on the completion of **all** previously admitted, not-yet-retired Submissions on the same Layer. Equivalent to a barrier immediately before the Submission's entry frontier. | O(W) where W = outstanding-window size (one fan-in edge per prior Submission). | Atomic snapshot of the outstanding-Submission set plus a single barrier-token allocation. **No dep-metadata lock.** Prior-task completion handlers decrement the shared barrier token via atomic RMW and do not contend with admission on per-task structures. | Always safe; serializes against prior work. |
| **`DATA`** | For each Task in `boundary_in_tasks`, scan its input tensors; for every tensor whose producer is a Task in a still-outstanding Submission, install a producer→consumer edge. Tensor identity is matched by `BufferRef` (not by bytewise address). **RAW-only in v1** — WAR / WAW across Submissions are not tracked at the runtime boundary; see [§2.10.6](12-dependency-model.md#2106-scope-limits-v1). | O(I × P) where I = boundary input count and P = producers tracked in the outstanding window. | **Dep-metadata lock required.** Admission reads and writes the shared `producer_index` and increments fan-in counters on newly admitted boundary-in Tasks concurrently with prior-Submission completion handlers mutating the same structures. A per-Layer RW lock (or per-shard equivalent) protects `producer_index`; the lock is held only for the duration of external-edge construction for one Submission. See [§2.1.3.1.B](02-scheduler.md#2131b-dependency-resolution-per-submission) for the full scheme. | Safe iff (a) tensor identity via `BufferRef` captures all real hazards (matches the runtime's tensor-lifetime model, [§2.4.5](#245-tensor-lifecycle)) **and** (b) the non-aliasing intermediate-memref invariant holds ([§2.10.5](12-dependency-model.md#2105-invariants), [§2.1.6](04-memory.md#216-memory-manager)). |
| **`NONE`** | No external edges are installed. | O(1) — hot-path cost is a single bit check. | **No lock required.** No shared dep metadata is read or written beyond the Submission's own freshly allocated slots, which are not yet visible to any other handler until admission publishes the Submission. | Caller-asserted: the caller guarantees the Submission has no true dependency on any prior outstanding Submission. Using `NONE` when a real dependency exists is a correctness bug, not a runtime-detected error. |

**Default.** Layers declare a default `DepMode` in `LevelParams.default_dep_mode`. When a submission omits `dep_mode` the layer default is used (`DATA` is the recommended default; `NONE` is recommended only for streams of known-independent work such as iteration-level data loaders).

**Interaction with intra-group edges.** Every `intra_edges` entry is always installed; the intra-group DAG is therefore always enforced even when `dep_mode = NONE`.

**Interaction with Boundary-Only rule.** For Group Submissions, `DATA` mode walks only the boundary-in tasks; non-boundary tasks inside the Submission already have all of their real producers in the same Submission and therefore never need an external edge. This is the single most important optimization enabled by the group/boundary split.

**Intra-Submission hazards.** Every intra-Submission RAW / WAR / WAW hazard — including sub-range overlap in assemble-style operations — MUST be expressed as an `intra_edges` entry by the frontend. The runtime does not attempt intra-Submission hazard detection; a missing `intra_edges` entry is a frontend correctness bug, not a runtime-recoverable condition. See [§2.10 Dependency Model](12-dependency-model.md) and [`tensor-dependency.md`](../../../tensor-dependency.md) for the frontend analysis that produces these edges.

## 2.4.C Group Submission Semantics

A **Group Submission** is a Submission whose `tasks` vector is a pre-built DAG. It is the general form; Single and SPMD are special cases.

**Intra-group edges**
- Supplied by the caller as `(producer_index, consumer_index)` pairs referencing indices into `tasks`.
- Must form an acyclic graph over `tasks`.
- Installed atomically at admission, before the Submission's tasks become eligible for dependency resolution against prior Submissions.

**Boundary classification**
- `boundary_in_tasks` — tasks that have at least one external input (from a prior Submission or from host-level buffers).
- `boundary_out_tasks` — tasks that have at least one external output consumed outside this Submission (by a later Submission or read back by the host).
- Non-boundary tasks — neither; their inputs and outputs are completely internal.
- Classification is either supplied by the submitter (preferred; the caller typically already knows) or computed by the Scheduler from `intra_edges` and `boundary_in_mask`/`boundary_out_mask` hints on per-task tensor arguments.

**External dependency attachment**
- `BARRIER` / `DATA` modes attach edges **only** to `boundary_in_tasks`. Non-boundary tasks never gain external fan-in edges.
- `BARRIER` attaches a single synthetic join against all outstanding Submissions' completion sets, shared by every boundary-in task in the Submission.
- `DATA` attaches per-tensor producer→consumer edges driven by the boundary-in task's `IN` / `INOUT` arguments.
- `NONE` attaches nothing.

**Atomicity**
- The entire Submission is admitted as one unit — if the admission policy or resource reservation fails for any task or for the workspace, the whole Submission is rejected and no Task slots are allocated.
- Conversely, the Submission's window slot is freed only when **every** Task in it has reached RETIRED.

## 2.4.D Group Workspace Memory

Non-boundary tensors inside a Group Submission have a lifetime that is **exactly** the lifetime of the Submission — they are produced and consumed strictly by tasks in the Submission. The runtime exploits this with a **Group Workspace**: a single coherent allocation from the Layer's data region, sized once at admission and freed once at retirement.

**Structure**
- One `WorkspaceHandle` per Submission (absent when the Submission has no non-boundary tensors).
- Internally a bump/offset arena: each non-boundary tensor is given `(workspace_handle, offset, size)` and exposed to tasks as an ordinary `BufferRef` via `IMemoryManager::workspace_subrange(...)` ([§2.1.6.A](04-memory.md#216a-group-workspace-memory)).
- Boundary tensors (inputs read from prior Submissions, outputs consumed by later Submissions) are **not** carved from the workspace — they follow the normal tensor lifecycle in [§2.4.5](#245-tensor-lifecycle).

**Lifecycle**
1. At admission, the Scheduler computes total workspace bytes from the Submission's `WorkspaceRequest` (sum of non-boundary tensor sizes with per-tensor alignment), then calls `IMemoryManager::alloc_workspace(size, hint)`.
2. Each non-boundary tensor's `BufferRef` is produced via `workspace_subrange(handle, offset, size)`.
3. When the last Task in the Submission reaches RETIRED, the Scheduler calls `IMemoryManager::free_workspace(handle)`, reclaiming all non-boundary tensor storage as one operation.

**Invariants**
- Sub-range `BufferRef`s obtained from a workspace MUST NOT be retained past Submission retirement. Any cross-Submission reference necessarily implies the underlying task was boundary, not non-boundary.
- The workspace is bulk data-plane memory; it lives in a `DATA_*` region (typical placement table row in [§2.1.8](04-memory.md#218-access-efficiency-considerations)) and does not require cache-line isolation.
- Workspace allocation obeys the same `IResourceAllocationPolicy` back-pressure as ordinary buffer allocation: if total workspace bytes are not currently available, admission waits (or the Submission is rejected per policy).

**Benefits**
- One allocation and one free per Submission, instead of N per non-boundary tensor.
- Eliminates per-tensor free ordering concerns inside the Submission — the runtime does not need to track ref-counts for non-boundary tensors.
- Cuts memory-manager traffic on the critical admission/retirement paths.

## 2.4.1 Task Structure

> [UPDATED: A1-P4: hot/cold field split + `idempotent` flag]
> The Task struct's hot 64-byte tier (first cache line) carries only `state, fan_in_counter, submission_id, exec_type_id, worker_id, dispatch_ts_ns`. The cold tail carries `function_id, args_blob_ptr, parent_task_key, dbg_name, trace_flags, error_ctx_ptr`, plus the precomputed **successor list** used by A3-P4 failure propagation. Layout is AoS per-slot with state-transition handlers touching only the hot line; `ErrorContext` is written only on FAILED and lives in the cold tail. See `modules/core.md §8` for the normative field ordering, sizes, and alignment. SoA for `fan_in_counter` was considered and rejected (see bench citation in `modules/core.md §8`).
>
> **[UPDATED: A5-P4: `idempotent: bool` on `TaskDescriptor`]** `TaskDescriptor` carries `bool idempotent = true;` — the Scheduler MAY retry a failed Task only when `idempotent == true`; Python bindings expose this as `simpler.Task(..., idempotent=True)`. The **caller-initiated** retry of a rejected submission is independent of this flag (the flag gates scheduler-initiated retry only; see `06-scenario-view.md §6.2.3`).

| Field | Type | Description |
|-------|------|-------------|
| `task_key` | `TaskKey` | `(layer_id, slot_index, generation)` — unique across the entire machine |
| `layer_id` | `LayerId` | The Layer this Task is bound to |
| `submission_id` | `uint64_t` | Identifier of the Submission that contains this Task ([§2.4.A](#24a-submission-model)); used for earliest-first ordering and window accounting |
| `is_boundary_in` | `bool` | Task is a boundary-in task of its Submission (eligible for external fan-in edges); false for non-boundary tasks |
| `is_boundary_out` | `bool` | Task is a boundary-out task of its Submission (its outputs may be consumed by later Submissions) |
| `func_ref` | `FunctionRef` | Reference to the Function to invoke |
| `args` | `TaskArgs` | Tensor arguments and scalar arguments |
| `deps` | `DepList` | Predecessor TaskKeys (intra-layer dependencies) — includes both intra-Submission and dep-mode-derived external edges |
| `parent_key` | `TaskKey?` | Parent Task's key in the Layer above (nullable) |
| `state` | `TaskState` | Current lifecycle state |
| `resource_req` | `ResourceReq` | Memory requirements and resource constraints |
| `workspace_ref` | `WorkspaceHandle?` | Group workspace this Task's non-boundary tensors are carved from (null for Single Submissions and boundary tasks) |
| `exec_type_id` | `uint32_t` | References a `TaskExecType` ([§2.4.8](#248-task-execution-types)) — which Worker Types and quantities are needed |
| `source_node` | `NodeId?` | Originating Node (for distributed Tasks) |
| `profiling` | `ProfilingMeta` | Timestamps, counters |
| `error` | `ErrorContext?` | Error information if Task failed |

## 2.4.2 Task Key and Handle

**Task Key** uniquely identifies a Task across the entire machine:
```
TaskKey = { layer_id: LayerId, slot_index: uint32_t, generation: uint32_t }
```

**Task Handle** is the layer-local portion:
```
TaskHandle = { slot_index: uint32_t, generation: uint32_t }
```

The generation counter increments on slot recycling, preventing stale handle access.

## 2.4.3 Task Arguments and ContinuousTensor

Task arguments include:
- **Tensor Arguments**: `ContinuousTensor` descriptors with `ArgDirection` (`IN`, `OUT`, `INOUT`).
- **Scalar Arguments**: Fixed-size values passed by value.

```cpp
struct ContinuousTensor {
    void*       data_address;
    uint32_t    shapes[MAX_TENSOR_DIMS];       // storage shape (512B-aligned)
    uint32_t    valid_shapes[MAX_TENSOR_DIMS]; // logical data extent
    uint8_t     ndims;
    DataType    dtype;
};
```

The `shapes` field is the storage layout; `valid_shapes` is the logical data extent where `valid_shapes[i] <= shapes[i]`. This decoupling supports:
- Tail tiles in tiling loops.
- Masking in attention operations and variable-length sequences.
- Compiler optimization (dead data elimination, predicated stores).

## 2.4.4 Task State (Summary)

| State | Description |
|-------|-------------|
| `FREE` | Slot unallocated |
| `SUBMITTED` | Accepted by Scheduler; deps unresolved |
| `PENDING` | Deps registered; waiting for predecessors |
| `DEP_READY` | All deps satisfied; waiting for resources |
| `RESOURCE_READY` | Resources allocated; ready for dispatch |
| `DISPATCHED` | Handed to Worker; execution not confirmed |
| `EXECUTING` | Worker actively running the Function |
| `COMPLETING` | Function body finished; waiting for children |
| `COMPLETED` | Execution and all children done; dependents notified |
| `RETIRED` | Resources freed; slot eligible for recycling |

> [UPDATED: A3-P1: canonical Task FSM + `ERROR` / optional `CANCELLED` states]
> The canonical Task terminal transition is `COMPLETING → ERROR` on failure (paired with Worker-side `FAILED`) — **not** `COMPLETED(error)`. `ERROR` is reachable from `DISPATCHED`, `EXECUTING`, and `COMPLETING` via `on_fail(task, error)`. An optional `CANCELLED` state is reachable from `PENDING`, `DEP_READY`, `RESOURCE_READY`, `DISPATCHED`, and `EXECUTING` via `on_cancel(task, reason)`. `FAILED` is retained **only as a Worker-side spelling** for backward-compat traces, per ADR-016. `06-scenario-view.md` uses the canonical spelling uniformly (per A4-R3-P2).
>
> **[UPDATED: A3-P11: drain semantics note]** During `drain()`, in-flight Tasks complete naturally; no new Tasks are admitted, and the drain flag is sticky (`AdmissionStatus::REJECT(Drain)` with sub-code `DrainInProgress` is returned to new `submit()` calls). See `09-interfaces.md §2.6.1` for the normative drain contract.

The same states are used at every Layer. See [Process View §4.3](../04-process-view.md#43-task-state-machine) for the full state machine.

## 2.4.5 Tensor Lifecycle

Tensor lifetime is governed by two complementary mechanisms:

**Implicit Scope Lifetime:** By default, a tensor's lifetime is tied to the scope in which its producer Task was created. A buffer is reclaimable when:
1. The scope token has been applied (via scope exit or explicit `free()`), AND
2. All declared consumers have completed (`ref_count == fanout_count`).

**Explicit Free:** `memory_manager.mark_tensor_free(buffer_ref)` applies the scope-exit token immediately, without waiting for the enclosing scope to exit. Idempotent and does not bypass fanout safety.

**Ring Stack Interaction:** When using ring buffer Memory Managers, each scope depth has its own task ring and buffer ring. Explicit `free()` reduces logical lifetime within a scope; ring stack eliminates cross-scope structural blocking.

**Deferred Completion:** `[ASSUMPTION]` Tasks marked `complete_in_future` release the Worker on function return but do not trigger completion until all registered hardware completion conditions are satisfied (e.g., SDMA, RoCE). Tensor buffers remain live until the deferred completion fires. See [Q8 Deferred Completion Tracking](../09-open-questions.md#q8-deferred-completion-tracking) for the open question on hardware-specific completion condition enumeration.
<!-- [UPDATED: A3-P11: [ASSUMPTION] marker + Q8 cross-ref] -->

**Workspace Tensors (Group Submissions):** Non-boundary tensors inside a Group Submission are sub-ranges of a single Group Workspace ([§2.4.D](#24d-group-workspace-memory)). Their lifetime is governed by the enclosing Submission — they are valid from admission to Submission retirement and are freed as a unit, bypassing per-tensor ref-count and scope-exit paths. Boundary tensors in a Group Submission continue to use the implicit-scope / explicit-free rules above.

## 2.4.6 SPMD Submission

An **SPMD Submission** is a specialization of the Group Submission ([§2.4.A](#24a-submission-model), [§2.4.C](#24c-group-submission-semantics)): N sub-tasks sharing the same Function with per-instance data, all admitted and accounted for as a single Submission.

| Field | Type | Description |
|-------|------|-------------|
| `func_ref` | `FunctionRef` | Single Function for all sub-tasks |
| `spmd_count` | `uint32_t` | Number of sub-task instances |
| `per_instance_args` | `TaskArgs[]` | Per-instance tensor arguments |
| `shared_args` | `TaskArgs` | Arguments shared across all sub-tasks |
| `dep_mode` | `DepMode` | Inherited from Submission ([§2.4.B](#24b-dependency-modes)); default `DATA` |

Each sub-task receives implicit `spmd_index` and `spmd_size`. The Scheduler expands it into individual sub-tasks.

> [UPDATED: A3-P9: SPMD index/size delivery contract + sibling tagging]
> `spmd_index` and `spmd_size` are delivered in `TaskArgs.scalar_args` at **reserved indices 0 and 1** (see [§2.3.1](06-function-types.md#231-aicore-function)). SPMD sibling sub-tasks are tagged at admission with `(parent_submission_id, spmd_index)` so cancelled siblings under A3-P5 retain their `spmd_index` in `ErrorContext.remote_chain`. The ABI is preserved under A9-P4's `Kind` removal — SPMD-ness is derivable from `spmd.has_value()`.

**Relation to the Submission Model.**
- SPMD typically has **no intra-group edges** — sub-tasks are mutually independent.
- All sub-tasks are boundary tasks of the Submission by default (each reads shared inputs and writes per-instance outputs that are typically consumed outside the Submission).
- The outstanding-window accounting and `dep_mode` behavior are identical to any other Submission: an SPMD Submission occupies exactly one window slot regardless of `spmd_count`.
- An SPMD Submission **may** request a group workspace when the Function needs per-instance scratch storage that the compiler has sized ahead of time; the workspace is then freed when the whole SPMD set retires.

## 2.4.7 Data Types

| DataType | Enum | Size (bytes) |
|----------|------|-------------|
| `FLOAT32` | 0 | 4 |
| `FLOAT16` | 1 | 2 |
| `INT32` | 2 | 4 |
| `INT16` | 3 | 2 |
| `INT8` | 4 | 1 |
| `UINT8` | 5 | 1 |
| `BFLOAT16` | 6 | 2 |
| `INT64` | 7 | 8 |
| `UINT64` | 8 | 8 |

## 2.4.8 Task Execution Types

A **Task Execution Type** (`TaskExecType`) declares which Worker Types a Task requires and in what quantities. It is associated with a Task through its Function ([§2.3](06-function-types.md)) and stored in the Task's `exec_type_id` field ([§2.4.1](#241-task-structure)).

```cpp
struct TaskExecType {
    std::string  type_name;            // e.g., "VectorOnly", "CubeOnly", "FullCoreWrap"
    uint32_t     type_id;
    struct Slot {
        uint32_t worker_type_id;       // references a WorkerTypeDescriptor (§2.1.4.2)
        uint32_t count;                // number of Workers of this type required
    };
    std::vector<Slot> required_slots;  // list of (worker_type_id, count) pairs
    bool requires_worker_group;        // if true, all slots must come from the same WorkerGroup
};
```

**Semantics:**
- `required_slots` enumerates the Worker Types needed and how many of each. The Task cannot be dispatched until all slots are satisfiable.
- `requires_worker_group = true` means all required Workers must belong to the **same** WorkerGroup ([§2.1.4.2](03-worker.md#2142-heterogeneous-worker-types-and-worker-groups)). The WorkerManager enforces this via the Resource Exclusion Rule.
- `requires_worker_group = false` means Workers may come from any group or be ungrouped.

**Concrete instances for the Chip level (Ascend a2a3):**

| TaskExecType | `required_slots` | `requires_worker_group` | Description |
|-------------|-------------------|-------------------------|-------------|
| `VectorOnly` | `[{AIV, 1}]` | false | Single vector core operation |
| `CubeOnly` | `[{AIC, 1}]` | false | Single cube core operation |
| `CubeVector` | `[{AIC, 1}, {AIV, 1}]` | **true** | Cube + vector fusion; must be same Core Wrap |
| `FullCoreWrap` | `[{AIC, 1}, {AIV, 2}]` | **true** | Full core wrap utilization; must be same Core Wrap |

**Homogeneous levels** define a single `TaskExecType` with one slot of the single Worker Type and `requires_worker_group = false`, preserving backward compatibility.
