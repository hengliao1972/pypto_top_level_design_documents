# 6. Scenario View

This view validates the architecture by walking through concrete, end-to-end scenarios. Each scenario traces through all four structural views (Logical, Development, Process, Physical) to verify consistency and completeness.

---

## 6.1 Key Scenarios

### 6.1.1 Scenario: Single-Node Hierarchical Kernel Execution

**Actor:** Python user
**Goal:** Execute an orchestrated computation (e.g., a fused attention kernel) on a single Ascend NPU device.
**Preconditions:** Runtime initialized with single-node 4-level configuration (Host, Device, Chip, Core). Functions registered. Device memory allocated.

**Steps:**

| Step | Action | Logical | Development | Process | Physical |
|------|--------|---------|-------------|---------|----------|
| 1 | Python calls `runtime.submit(orch_func, tensors)` | Host Function submitted as Task to Host Scheduler | `bindings/` → `runtime/` → `scheduler/host_scheduler` | Host thread creates Task in Host task slot pool; state → SUBMITTED | Host CPU |
| 2 | Host Scheduler dispatches to Device Worker | Task transitions SUBMITTED → PENDING → DEP_READY → RESOURCE_READY → DISPATCHED | `scheduler/host_scheduler` → `transport/` (DMA Control Channel) | DMA transfer of Task descriptor to Device memory; MMIO signal to AICPU | Host → Device via PCIe DMA |
| 3 | AICPU Thread executes Orchestration Function | Device Worker runs orch_func; body submits child compute Tasks to Device Scheduler | `scheduler/device_orchestrator` → `core/` (Task creation) | AICPU thread calls `scheduler.submit(compute_A)`, `scheduler.submit(compute_B, deps=[A])` | Device AICPU |
| 4 | Device Scheduler dispatches compute_A to Chip level | Task forwarded via Vertical Channel (SharedMemory) to Chip Scheduler | `scheduler/device_orchestrator` → `transport/` (SharedMemory Channel) → `scheduler/aicpu_scheduler` | Chip Scheduler writes dispatch payload to shared memory | Device → Chip (on-chip) |
| 5 | Chip Scheduler dispatches compute_A to AICore₀ | Chip Scheduler selects AICore₀, writes `DATA_MAIN_BASE` register | `scheduler/aicpu_scheduler` → `transport/` (RegisterBank Channel) → `hal/` (register write) | Register Bank write triggers AICore₀ polling loop to detect new task | Chip → Core (register bank) |
| 6 | AICore₀ executes kernel | AICore Function runs on compute pipeline | `hal/` (IExecutionEngine) | AICore processes tiles; reads tensors from shared memory | AICore₀ hardware |
| 7 | AICore₀ signals completion | Write FIN value to `COND` register | `transport/` (RegisterBank Channel) | Chip Scheduler polls COND, detects FIN | Core → Chip |
| 8 | compute_B becomes DEP_READY | Chip Scheduler: `notify_dep_satisfied(compute_B, compute_A)` | `scheduler/aicpu_scheduler` → `core/` (fan-in decrement) | Atomic decrement of compute_B's fan-in counter → 0 → DEP_READY | Chip level |
| 9 | compute_B dispatched and completes | Same as steps 5–7 for compute_B | Same modules | Same flow | AICore₁ |
| 10 | Children complete → parent completes | Chip Scheduler: `notify_child_complete(parent)` → Device Scheduler processes | `scheduler/aicpu_scheduler` → `transport/` → `scheduler/device_orchestrator` | Parent Task COMPLETING → COMPLETED; propagates to Host via DMA Control Channel | Device → Host |
| 11 | Python receives result | Host Scheduler: Task COMPLETED → RETIRED; Python `task.result()` returns | `scheduler/host_scheduler` → `bindings/` | GIL acquired, result returned to Python | Host CPU |

**Postconditions:** Output tensors contain computed results in Device memory. Task slots recycled. Profiling data flushed.

---

### 6.1.2 Scenario: Distributed Multi-Node Execution

**Actor:** Python user
**Goal:** Execute a data-parallel computation across 2 nodes in a Pod.
**Preconditions:** Runtime initialized with 5-level configuration (Core, Chip, Device, Host, Pod). 2 nodes connected via RDMA. Functions registered on all nodes.

**Steps:**

| Step | Action | Logical | Process | Physical |
|------|--------|---------|---------|----------|
| 1 | Python calls `runtime.submit(dist_orch_func, tensors)` | Distributed Orchestration Function submitted to Pod Scheduler | Pod Scheduler thread creates Task; state → SUBMITTED | Coordinator node |
| 2 | Pod Scheduler partitions work | IPartitioner (DATA_LOCALITY) assigns partition A → Node₀, partition B → Node₁ | Scheduler evaluates data locations; creates 2 child tasks | Coordinator node CPU |
| 3 | Local partition submitted | Child task for partition A submitted to Node₀'s Host Scheduler via Vertical Channel (TCP) | Local IVerticalChannel.submit_to_child() | Node₀ |
| 4 | Remote partition submitted | `REMOTE_SUBMIT` message sent to Node₁ via Horizontal Channel (RDMA) | Message serialized and sent via IHorizontalChannel | Node₀ → Node₁ via RDMA NIC |
| 5 | Node₁ receives remote task | Node₁'s Pod-level handler deserializes Task, submits to local Host Scheduler | Local task slot allocated; normal FSM begins | Node₁ |
| 6 | Both nodes execute locally | Each node runs Host → Device → Chip → Core hierarchy independently | Parallel execution on both nodes | Node₀ + Node₁ |
| 7 | Cross-node data exchange | Data Movement Task: tensor T from Node₀ → Node₁ via `IMemoryOps.copy_to_peer()` | RDMA_WRITE; `REMOTE_DATA_READY` notification on completion | Node₀ NIC → Node₁ NIC (RDMA) |
| 8 | Node₁ completes | Node₁ sends `REMOTE_COMPLETE` to Coordinator | Message via Horizontal Channel | Node₁ → Node₀ |
| 9 | Node₀ completes locally | Local child task completes; `notify_child_complete(parent)` | Pod Scheduler decrements parent pending counter | Node₀ |
| 10 | All children complete | Pod-level parent Task: COMPLETING → COMPLETED | Propagate to Python | Coordinator node |
| 11 | Python receives aggregated result | `task.result()` returns with results from both nodes | GIL acquired | Coordinator host CPU |

**Postconditions:** Output tensors available on coordinator node. Remote node task slots recycled. Distributed trace correlation IDs recorded.

---

### 6.1.3 Scenario: SPMD Kernel Launch

**Actor:** Orchestration Function running on AICPU
**Goal:** Launch the same kernel across 24 AICores (block_dim=24) with different data tiles.
**Preconditions:** Device-level Orchestration Task is EXECUTING. Tensor data partitioned in shared memory.

**Steps:**

| Step | Action |
|------|--------|
| 1 | Orchestration Function calls `scheduler.submit_spmd(kernel_func, block_dim=24, per_tile_args)` |
| 2 | Chip Scheduler expands into 24 sub-tasks, each with `spmd_index` 0–23 and `spmd_size` 24 |
| 3 | Sub-tasks dispatched to AICores 0–23 via Register Bank (parallel dispatch) |
| 4 | Each AICore executes kernel with its tile's data; reads from shared memory at offset based on `spmd_index` |
| 5 | Each AICore signals FIN via `COND` register |
| 6 | Chip Scheduler tracks group completion: all 24 sub-tasks COMPLETED |
| 6a | **[UPDATED: A3-P4]** On **any** sub-task failure, the SPMD aggregation edge fires: the parent Orchestration Task is notified with a `DEP_FAILED(producer_key, ErrorCode)` event and transitions `COMPLETING → ERROR`. Surviving siblings that are still `EXECUTING` run to completion but their outputs are discarded per A3-P5 sibling-cancellation policy. Siblings that had not yet dispatched are cancelled before dispatch. |
| 6b | **[UPDATED: A3-P5]** Cancelled / discarded siblings contribute a `CANCELLED` entry to `ErrorContext.remote_chain` that **retains the sibling's `spmd_index`** so the parent (and user) can identify which SPMD shard was cancelled. The `spmd_index` is preserved for the full 24-way SPMD edge case. |
| 7 | SPMD group complete → parent Orchestration Task's pending child counter decremented |

---

## 6.2 Failure Scenarios

### 6.2.1 Failure: AICore Hang During Kernel Execution

**Context:** Single-node execution; AICore₃ hangs during compute_A (step 6 of Scenario 6.1.1).

| Step | What Happens |
|------|-------------|
| 1 | Chip Scheduler polls `COND` register for AICore₃'s FIN value |
| 2 | Timeout expires (configurable per level in `LevelParams`) |
| 3 | Chip Scheduler marks AICore₃ as FAILED; creates `ErrorContext{HardwareFault, core_id=3, task_key=compute_A}` |
| 4 | `notify_parent_error(parent, ErrorContext)` sent via Vertical Channel to Device Scheduler |
| 5 | Device Scheduler: parent Orchestration Task transitions COMPLETING → ERROR |
| 6 | Error propagated to Host Scheduler via Vertical Channel |
| 7 | **[UPDATED: A4-R3-P2]** Host Scheduler: parent Task transitions `COMPLETING → ERROR` (canonical spelling per ADR-016; was "COMPLETED(error)"); `notify_parent_complete` with error context |
| 8 | Python: `task.result()` raises `simpler.DeviceError` with message including core_id and task details |

**System state after failure:** AICore₃ marked unavailable for future dispatches (until device reset). Other AICores continue operating. Resources for failed task are cleaned up via `on_retire` with error flag.

### 6.2.2 Failure: Remote Node Crash During Distributed Execution

**Context:** Distributed execution across 2 nodes (Scenario 6.1.2); Node₁ crashes during step 6.

| Step | What Happens |
|------|-------------|
| 1 | Coordinator (Node₀) stops receiving `HEARTBEAT` from Node₁ |
| 2 | Heartbeat timeout expires (configurable: default 10s) |
| 3 | DistributedScheduler marks Node₁ as FAILED |
| 4 | Creates `ErrorContext{RemoteNodeFailure, node_id=1}` |
| 5 | **Policy evaluation:** Check `DistributedConfig.failure_policy` |
| 6a | **[UPDATED: A4-R3-P2]** **If ABORT_ALL:** All in-flight tasks on Node₀ are cancelled; parent Task transitions `COMPLETING → ERROR` (canonical spelling per ADR-016); Python gets `simpler.DistributedError` |
| 6b | **If CONTINUE_REDUCED:** Node₀'s local partition continues; Node₁'s partition is marked FAILED; parent Task completes with partial results and error context |
| 6c | **If RETRY_ALTERNATE:** Node₁'s partition is re-submitted to an alternate node (if available); execution continues |
| 7 | Error context includes `remote_error_chain` with Node₁'s last known state |

### 6.2.3 Failure: Task Slot Pool Exhaustion

**Context:** High-throughput submission; Chip-level task slot pool is full.

> [UPDATED: A3-P3: Cross-references to new admission-path failure scenarios §6.2.5–§6.2.7.]
> This scenario captures the **resource-exhaustion** axis of admission failure. Three additional admission-path failure scenarios are added by A3-P3 and sit as siblings in §6.2:
>
> - **§6.2.5** Admission Rollback on Cyclic `intra_edges` → `AdmissionStatus::REJECT(Validation::CyclicDependency)` (see A3-P7, A3-P8).
> - **§6.2.6** Admission Rejection on Workspace Exhaustion → `AdmissionStatus::REJECT(Exhaustion::WorkspaceExhausted)`.
> - **§6.2.7** `NONE` Mode Misuse (debug-only assertion, A3-P15) → `AdmissionStatus::REJECT(Validation::NoneModeHiddenDependency)`.
>
> All three cross-reference the admission storm behavior described in this §6.2.3.

| Step | What Happens |
|------|-------------|
| 1 | Orchestration Function calls `scheduler.submit(child_task)` |
| 2 | **[UPDATED: A9-P5]** `IMemoryManager.alloc_task_slot()` fails → admission returns `AdmissionStatus::REJECT(Exhaustion)` (was `ResourceExhausted`; enums unified per A9-P5). `SubmissionDescriptor.flags` sets the `TaskSlotExhausted` sub-code (A1-P8, routed via A3-P7). |
| 3 | `on_submit` handler detects failure; returns error status to Orchestration Function |
| 4 | Orchestration Function MAY: (a) wait and retry (back-pressure), (b) propagate error upward (retry of a rejected submission by the caller is independent of the `idempotent` flag per A5-P4). |
| 5 | **[UPDATED: A3-P10]** If propagated: parent Task transitions `COMPLETING → ERROR` → `simpler.SchedulerError("task slot pool exhausted at Chip level")` (Scheduler-domain errors raise `simpler.SchedulerError`, not generic `simpler.RuntimeError`, per full Domain→Python mapping). |

**Mitigation:** Pool sizes are configurable via `LevelParams`. Monitoring via `LayerStats.get_stats()` provides pool utilization for capacity planning.

### 6.2.4 Failure: Network Timeout During Cross-Node Data Movement

**Context:** RDMA write from Node₀ to Node₁ during Scenario 6.1.2 step 7.

| Step | What Happens |
|------|-------------|
| 1 | `IMemoryOps.copy_to_peer()` initiates RDMA write |
| 2 | `poll()` returns no completion within timeout period (Rule R1) |
| 3 | Data Movement Task → ERROR with `CommunicationError` |
| 4 | Dependent tasks on Node₁ receive `REMOTE_ERROR` instead of `REMOTE_DATA_READY` |
| 5 | Consumer tasks' dependency counter is not decremented; they eventually timeout |
| 6 | Error aggregated at Pod level; parent Task → ERROR |
| 7 | Python: `simpler.DistributedError("RDMA write to Node₁ timed out")` |

---

## 6.3 Cross-View Consistency Checks

| Check | Verified By |
|-------|-------------|
| Logical components → code modules | Each domain concept (Scheduler, Worker, Memory Manager, etc.) maps to a specific module in `03-development-view.md` |
| Code modules → runtime processes | Each module's execution context (host thread, AICPU thread, AICore pipeline) is specified in `04-process-view.md` |
| Runtime processes → deployment nodes | Thread-to-hardware mapping is specified per deployment configuration in `05-physical-view.md` |
| Interface consistency | `ISchedulerLayer`, `IMemoryManager`, `IMemoryOps`, `IVerticalChannel`, `IHorizontalChannel` are defined in Logical View, implemented in Development View modules, exercised in Process View flows |
| Scenario coverage | All scenarios trace through all four views; each scenario exercises key interfaces |
