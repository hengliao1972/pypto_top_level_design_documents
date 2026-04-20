# 2.1.4 Worker

> Part of the [Logical View](../02-logical-view.md). This module describes Workers — the execution agents within each Layer — together with their lifecycle state machine, heterogeneous Worker Types, and Worker Group co-location constraints.

A Worker is an execution agent within a Layer. It receives a dispatched Task, executes the Task's Function, and reports completion. During execution, a Worker MAY:

- Access the Layer's Memory Scope.
- Submit child Tasks to its own Layer's Scheduler (hierarchical execution).
- Produce output data in the Layer's Memory Scope.

Worker type is not hardcoded — each Machine Level defines its worker type string and semantics at registration time (e.g., `"AICore"`, `"AICPU"`, `"Device"`, `"Node"`).

Thread/worker counts are configurable per level:
- `scheduler_thread_count`: threads running the Scheduler's ready-queue loop.
- `worker_thread_count`: threads (or hardware units) that concurrently execute Tasks.
- `child_worker_count`: number of child-level Workers managed by this level.

## 2.1.4.1 Worker State Machine

Each Worker has a formal lifecycle managed by the WorkerManager ([§2.1.3.2](02-scheduler.md#2132-workermanager)):

```
    ┌──────────┐
    │   IDLE   │ ◄──────────────────────────────┐
    └────┬─────┘                                │
         │ WorkerManager assigns task           │
         ▼                                      │
    ┌──────────┐                                │
    │ ASSIGNED │                                │
    └────┬─────┘                                │
         │ Worker begins execution              │
         ▼                                      │
    ┌──────────┐                                │
    │EXECUTING │                                │
    └────┬─────┘                                │
         │ Function body returns                │
         ├──────────────────────┐               │
         ▼                      ▼               │
    ┌───────────┐          ┌────────┐           │
    │COMPLETING │          │ FAILED │           │
    └────┬──────┘          └───┬────┘           │
         │ all children done   │ cleanup        │
         │                     ▼                │
         │               ┌────────────┐         │
         │               │ RECOVERING │         │
         │               └────┬───────┘         │
         │                    │                 │
         │           ┌────────┴────────┐        │
         │           ▼                 ▼        │
         │     ┌─────────────┐    (recovered)───┘
         │     │ UNAVAILABLE │
         │     └─────────────┘
         │
         └──────────────────────────────────────┘
```

| State | Description |
|-------|-------------|
| `IDLE` | Worker available for dispatch. WorkerManager reports this Worker as available for selection. |
| `ASSIGNED` | Task dispatched to Worker; execution has not yet begun (e.g., register write issued but AICore has not ACKed). |
| `EXECUTING` | Worker is actively running a Function. The Worker is occupied — no other Task can be dispatched to it. |
| `COMPLETING` | Function body has returned; Worker is waiting for child Tasks to complete. Still occupied. |
| `FAILED` | An error occurred during execution (hardware fault, timeout, runtime error). |
| `RECOVERING` | Cleanup in progress after failure (resource release, error context construction). |
| `UNAVAILABLE` | Permanently removed from the worker pool (e.g., hardware fault on an AICore). WorkerManager will not select this Worker. |

**Worker state transition handlers** (owned by WorkerManager):

| Handler | Trigger | Actions |
|---------|---------|---------|
| `on_assign(worker, task)` | Task dispatched by WorkerManager | Write dispatch payload; start execution timeout timer |
| `on_execute(worker)` | Worker confirms execution started (e.g., AICore ACK) | Record execution start timestamp |
| `on_complete(worker, task)` | Function returns or all children done | Report completion to TaskManager; transition to IDLE |
| `on_fail(worker, error)` | Timeout or hardware fault | Create ErrorContext; transition to RECOVERING |
| `on_recover(worker)` | Cleanup complete | Transition to IDLE (if recoverable) or UNAVAILABLE (if permanent) |

The Worker state machine is **uniform across all levels**, though not all states are meaningful at every level (e.g., AICore Workers have no COMPLETING state — they are leaf executors).

> [UPDATED: A5-P9: `UNAVAILABLE` variants + QUARANTINED DRY-fold]
> `UNAVAILABLE` is a single sink state parameterized by a policy variant: `UNAVAILABLE + { Permanent, Quarantine(duration) }`. The prior proposal for a distinct `QUARANTINED` state is folded into this variant to avoid FSM duplication. Transitions: `RECOVERING → UNAVAILABLE(Quarantine(worker_quarantine_ms))`; after a successful probe the Worker returns to `IDLE`; probe failure or exceeding `worker_probe_retries` (a.k.a. `max_quarantine_cycles`) transitions to `UNAVAILABLE(Permanent)`. Config knobs `worker_quarantine_ms`, `worker_probe_retries` are exposed in `LevelParams`. See the unified peer-health FSM in ADR-018 for the cross-module consolidation.

## 2.1.4.2 Heterogeneous Worker Types and Worker Groups

Workers within a Layer are not necessarily homogeneous. The architecture supports **heterogeneous** Workers with different execution capabilities, organized into **Worker Groups** that enforce co-location constraints for multi-worker Tasks.

### Worker Type

A **Worker Type** (`WorkerTypeDescriptor`) declares the execution capability of a class of physical Workers:

```cpp
struct WorkerTypeDescriptor {
    std::string  type_name;       // e.g., "AIC", "AIV", "AICPU", "Device"
    uint32_t     type_id;         // numeric ID for fast matching
    uint32_t     capability_mask; // bitmask of supported execution capabilities
};
```

Worker Types are registered per Machine Level in `MachineLevelDescriptor` ([§2.2.1](05-machine-level-registry.md#221-machine-level)). Each physical Worker instance carries a `worker_type_id` identifying which type it belongs to.

**Worker struct extension:**

```cpp
struct Worker {
    uint32_t     worker_id;       // unique within the Layer
    uint32_t     worker_type_id;  // references a WorkerTypeDescriptor
    uint32_t     group_id;        // references a WorkerGroup (0 if ungrouped)
    WorkerState  state;           // current lifecycle state (§2.1.4.1)
};
```

### Worker Group

A **Worker Group** is a named, non-overlapping partition of physical Workers within a Layer. Groups model hardware topology constraints where certain Workers must be co-allocated for multi-worker Tasks.

```cpp
struct WorkerGroup {
    uint32_t                  group_id;
    std::string               group_type;          // e.g., "CoreWrap"
    std::vector<uint32_t>     member_worker_ids;   // physical Worker IDs in this group
};
```

**Invariants:**
- Each physical Worker belongs to **at most one** Worker Group (non-overlapping).
- Workers not assigned to any group have `group_id = 0` and are treated as independent.
- Worker Groups are defined at registration time and are immutable for the Layer's lifetime.

### Resource Exclusion Rule

When the WorkerManager assigns Workers within a Worker Group to a Task, the remaining available Workers in that group are reduced accordingly. The key constraint:

> A Task with `requires_worker_group = true` ([§2.4.8](07-task-model.md#248-task-execution-types)) can only be scheduled if **all** its `required_slots` can be satisfied by **idle** members of a **single** WorkerGroup.

Assigning individual Workers from a group to single-worker Tasks reduces the group's capacity for multi-worker Tasks. The WorkerManager must track per-group availability to enforce this rule.

> [UPDATED: A1-P9: bitmask-encoded WorkerGroup availability per slot-type (108-core ACK note)]
> Per-slot-type availability is encoded as `uint64_t` bitmasks across Worker Groups (one bitmask per Worker Type × required-slot-count tier). `select_workers` ANDs the required masks and scans via `__builtin_ctzll`; complexity is O(slot_types). For deployments with **> 64 groups per slot-type** (e.g., the a5 108-AICore case), the bitmap is split into two `uint64_t`s and scanned with an AND-then-`ctzll` fallback — both the single-word and 2 × 64 paths stay < 10 ns. Single `uint64_t` representation caps at ≤ 64 groups per slot-type per bitmap. The same 108-bit-popcount ACK bitmap is used on the SPMD fan-out ACK path; see `02-scheduler.md` SPMD latency budget (A1-P5).

> [UPDATED: A5-P8: partial-group availability degradation policy]
> When the WorkerManager detects partial availability of a required group mid-Submission (member loss, hardware fault), the `partial_group_policy ∈ {FAIL_SUBMISSION, SHRINK_AND_RETRY, DEGRADED_CONTINUE}` governs the response. `FAIL_SUBMISSION` is the default and surfaces `WorkerGroupPartial` via the admission error catalog. `SHRINK_AND_RETRY` re-selects a smaller group if the `TaskExecType` allows it. `DEGRADED_CONTINUE` proceeds with the surviving members and records the degradation in `ErrorContext.remote_chain`. Each value is tied to an A8-P5 `AlertRule`; see the cross-reference in `02-scheduler.md §2.1.3.1.A` for the admission-side counterpart.

### Canonical Example: Ascend NPU Core Wraps

On the Ascend a2a3 Chip level, the 72 cores are organized as:

| Worker Type | Type ID | Count | Capability |
|-------------|---------|-------|------------|
| `AIC` (cube core) | 1 | 24 | Matrix/cube compute |
| `AIV` (vector core) | 2 | 48 | Vector compute |

These 72 cores are partitioned into **24 Core Wraps** (Worker Groups), each containing 1 AIC + 2 AIV:

```
Core Wrap 0:  { AIC_0,  AIV_0,  AIV_1  }
Core Wrap 1:  { AIC_1,  AIV_2,  AIV_3  }
...
Core Wrap 23: { AIC_23, AIV_46, AIV_47 }
```

No AIC or AIV is shared between Core Wraps. When a `VectorOnly` task is dispatched to `AIV_0`, Core Wrap 0's available capacity becomes `{AIC_0(IDLE), AIV_0(EXECUTING), AIV_1(IDLE)}` — it can still accept a `CubeOnly` or `CubeVector` task but not a `FullCoreWrap` task (which requires 1 AIC + 2 AIV all idle).

### Homogeneous Levels as Degenerate Case

Levels with homogeneous Workers (e.g., Host level with identical Device workers, Pod level with identical Node workers) register a single Worker Type and no Worker Groups. The WorkerManager degrades to flat-pool behavior with no group constraints — fully backward compatible.
