# Simpler Distributed Runtime: HostWorker / DistWorker Implementation Design

This document describes the distributed runtime implementation design in the `simpler` repository (Phase 2: HostWorker), covering the L3-level DistWorker scheduling engine, HostSubWorker fork+shm parallel execution, the unified Worker interface, and usage patterns.

This document synthesizes the following design documents from `simpler/.docs/`: `UNIFIED_RUNTIME_PLAN.md`, `PHASE2_HOST_WORKER.md`, `HOSTSUB_FORK_SHM_DESIGN.md`, and `PLATFORM_SERVICES_DESIGN.md`. It is intended for upper-layer (pypto / linqu) developers to understand the underlying distributed execution capabilities.

---

## 1. Background and Positioning

### 1.1 Level Definitions

| Level | Resource | Orchestration | Implementation |
|-------|----------|---------------|----------------|
| L1 | Single AICore (AIC/AIV) | None | Kernel binary, scheduled by L2 |
| L2 | Single Chip (N AICores) | C++ orch on AICPU | **ChipWorker** (completed) |
| L3 | Single Host (M Chips) | Python orch on host CPU | **HostWorker / DistWorker** (Phase 2) |
| L4+ | Multiple Hosts | Isomorphic extension | Future |

### 1.2 Core Design Principles

- **Isomorphic with L2**: L3 reuses L2's entire execution model (scope + ringbuffer + tensormap + submit) — no new concepts invented
- **Unified Worker model**: Every level's Worker exposes the same `run(task)` blocking interface, fully isomorphic with ChipWorker
- **Do not modify simpler**: All L3+ design builds on top of simpler's L0-L2, never modifying existing L2 code

### 1.3 Implementation Progress

| Phase | Status | Notes |
|-------|--------|-------|
| Phase 0: Pre-built Runtime Binaries | Done | PR #386 |
| Phase 0.5: TaskArgs Type Refactor | Done | PR #391, #392, #395 |
| Phase 1a: Callable Type | Done | PR #408, #413 |
| Phase 1b: ChipWorker | Done | feat/chip-worker branch |
| **Phase 2: HostWorker (DistWorker)** | **Not started** | Focus of this document |

---

## 2. Unified Worker Interface

### 2.1 IWorker Interface

All Worker implementations share a single interface, fully transparent to callers:

```cpp
class IWorker {
 public:
    virtual ~IWorker() = default;
    // Called in the worker's own thread, blocks until task completion
    virtual void run(const WorkerPayload& payload) = 0;
};
```

### 2.2 Worker Hierarchy

```
IWorker (interface)
  run(payload)   <- blocks in the worker's own thread

  +-- ChipWorker     -> L2 hardware. run() = blocking C API call
  +-- SubWorker      -> fork/shm Python function. run() = write mailbox + spin-poll until TASK_DONE
  +-- DistWorker     -> Any-level node. run() = execute(host_task) with internal orch + drain
                       int level_ (3=L3, 4=L4, ...)
                       L3 instance: sub_workers = [ChipWorker x N, SubWorker x M]
                       L4 instance: sub_workers = [DistWorker(level=3) x K, SubWorker x M]
```

Each IWorker instance has its own worker thread. The Scheduler only routes tasks — it never executes `run()`.

### 2.3 run() Implementation per Worker

```
ChipWorker::run(payload):
    chip_worker_impl.run(callable, args, config)   <- C API itself blocks

SubWorker::run(payload):
    write_mailbox(TASK_READY, callable_id)          <- dispatch to forked child process
    while (read_mailbox() != TASK_DONE) sleep(50us) <- spin-poll, blocks this worker thread
    write_mailbox(IDLE)

DistWorker::run(payload):                           <- when used as an L4 sub-worker
    execute(host_task)                              <- internal orch + drain, blocks
```

---

## 3. Phase 2 Architecture: DistWorker Scheduling Engine

### 3.1 Component Overview

| Component | Location | Language | Responsibility |
|-----------|----------|----------|----------------|
| **Orch** | Main process, main thread | Python calling C++ | submit (tensormap + fanin wiring), scope |
| **Scheduler** | Main process, dedicated C++ thread | C++ | Pop ready_queue -> push to worker task_queue; receive completion_queue notifications -> fanout release |
| **ChipWorker threads x N** | Main process, C++ thread per device | C++ | loop: pop task_queue -> `run()` blocks -> push completion_queue |
| **SubWorker threads x M** | Main process, C++ thread per SubWorker | C++ | loop: pop task_queue -> `run()` writes mailbox + spin-polls TASK_DONE -> push completion_queue |
| **HostSubWorker processes x M** | Forked child processes | Python | loop: spin-poll mailbox TASK_READY -> execute callable -> write TASK_DONE |

### 3.2 Communication Model

```
Orch --ready_queue.push--> Scheduler --task_queue.push--> Worker thread
                                                                |
                                      completion_queue.push <--+ (after run() completes)
                                      cv.notify()
     <--ring.release------ Scheduler <--completion_queue.pop--
                            fanout release -> ready_queue.push
```

The Scheduler waits on a condition variable, woken when either ready_queue or completion_queue has content.

### 3.3 Architecture Diagram

```
Main Process
+-- Main thread: Orch (Python -> C++)
|     submit():
|       1. alloc slot + output (CV-blocks when ring is full)
|       2. tensormap lookup (find producer -> fanin)
|       3. tensormap insert (register output)
|       4. write task slot (fanin_count, payload)
|       5. finalize fanin (lock fanout_mu, attach consumer list)
|       6. if fanin_count==0 -> state=READY, push ready_queue + cv.notify
|     scope_begin/scope_end
|
+-- Scheduler thread (C++)
|     loop: wait on cv
|       drain completion_queue -> on_task_complete -> fanout release
|       drain ready_queue -> pick idle worker -> task_queue.push
|     all done -> notify execute() to return
|
+-- ChipWorker threads x N (per device)
|     loop: task_queue.pop() -> run(payload) -> completion_queue.push + cv.notify
|
+-- SubWorker threads x M
|     loop: task_queue.pop()
|           -> write_mailbox(TASK_READY)
|           -> spin-poll mailbox until TASK_DONE
|           -> write_mailbox(IDLE)
|           -> completion_queue.push + cv.notify
|
HostSubWorker processes x M (forked child processes, each with independent GIL)
      <- fork happens in HostWorker.__init__(), before any C++ thread starts
      loop: spin-poll mailbox.state == TASK_READY
            -> callable_registry[callable_id]()
            -> mailbox.state = TASK_DONE
```

### 3.4 Correspondence with L2

| L2 (AICPU, C++) | L3 (Host) |
|---|---|
| Orch thread -> `pto2_submit_mixed_task()` | Main thread -> `submit()` |
| aicpu_executor thread | **Scheduler C++ thread** |
| Poll AICore COND register (hardware has no threads, must poll) | **completion_queue + CV notification** (has threads, no polling needed) |
| AICore hardware execution | **Worker threads** (one per Worker) |
| dispatch entry + COND register | worker task_queue + completion_queue |

### 3.5 Parallelism

| Scenario | True Parallel | Mechanism |
|----------|---------------|-----------|
| Orch (submit) <-> Scheduler | Yes | Independent C++ threads, communicate via ready_queue + CV |
| ChipWorker[0] <-> ChipWorker[1] | Yes | Separate worker threads each running `run()` independently |
| ChipWorker <-> SubWorker | Yes | Worker threads each block independently, no interference |
| SubWorker thread <-> forked child | Yes | Thread spin-polls mailbox; child process has independent GIL |
| Task chain pipeline | Yes | completion -> fanout -> ready_queue, Orch not involved |

---

## 4. HostSubWorker: Fork + Shared Memory

### 4.1 Approach Selection

| Approach | GIL Parallel | Tensor Zero-Copy | Callable Constraints |
|----------|-------------|------------------|---------------------|
| C++ thread + `gil_scoped_acquire` | No (Python serialized) | Yes | None |
| spawn new process | Yes | No (requires serialization) | Must be picklable |
| **fork + shm (adopted)** | Yes | Yes | None (already in memory before fork) |

### 4.2 Fork Timing (Critical Constraint)

`fork()` is unsafe in multi-threaded environments. **All forks must complete in `HostWorker.__init__()`, before any C++ thread starts:**

```
HostWorker.__init__():
    1. User registers callables (hw.register(name, fn) -> callable_id)
    2. Allocate shm mailbox (SharedMemory, /dev/shm)
    3. Fork M HostSubWorker processes            <- single-threaded, safe
    4. Create C++ HostWorkerEngine (nanobind)     <- C++ threads start after this
    5. Create ChipWorker x N
```

### 4.3 Mailbox Communication

Each HostSubWorker has an independent 256-byte mailbox (cache-line aligned):

```
Mailbox layout:
  offset 0:   int32  state        # IDLE=0, TASK_READY=1, TASK_DONE=2, SHUTDOWN=3
  offset 4:   int32  callable_id  # Pre-registered callable ID
  offset 8:   int64  args_shm_fd  # fd of shm containing tensor data
  offset 16:  int64  args_offset  # TaskArgs offset within shm
  offset 24:  int64  result_addr  # Result tensor write-back address
  offset 32:  int32  error_code   # 0=ok, nonzero=failed
  offset 64:  char[192] error_msg # Error message
  total: 256 bytes (4 cache lines)
```

State machine (isomorphic with L2 AICore dispatch):

```
       Scheduler writes                Worker reads
IDLE --> TASK_READY --> Worker polls --> execute callable --> TASK_DONE
  ^                                                            |
  +----------------- Scheduler polls <-------------------------+
```

### 4.4 Tensor Zero-Copy

- **torch tensor**: `tensor.share_memory_()` -> storage moves to `/dev/shm` -> child process directly accesses the same physical pages after fork
- **Mailbox**: `SharedMemory` (`/dev/shm` named file, `MAP_SHARED`) -> parent and child processes share physical pages

### 4.5 Callable Registration

```python
hw = HostWorker(device_ids=[0, 1], ...)
cid = hw.register("postprocess", postprocess_fn)   # Register before fork
# Child process inherits callable_registry via COW after fork — no pickle needed, lambdas/closures work

def my_orch(hw, args):
    hw.submit(HOST_SUB, cid, args_c)  # Pass callable_id, not the function object
```

### 4.6 POC Validation Results

| Test Case | Result |
|-----------|--------|
| SharedMemory bidirectionally visible after fork | Pass |
| `share_memory_()` tensor zero-copy | Pass |
| Lambda/closure callable after fork without pickle | Pass |
| IDLE -> TASK_READY -> TASK_DONE multi-round cycle | Pass |
| 3 workers parallel wall time < serial x 0.7 | Pass |
| Start Python/C++ threads after fork without deadlock | Pass |

---

## 5. Data Ownership and Synchronization

### 5.1 Data Ownership (Consistent with L2)

| Data Structure | Ownership | Accessed By | Protection |
|----------------|-----------|-------------|------------|
| **TensorMap** | **Orch exclusive** | Only submit() | **No lock needed** |
| **Scope stack** | **Orch exclusive** | submit() / scope_begin/end() | **No lock needed** |
| Task slot state | Shared | Orch + Scheduler | Per-task mutex + atomic |
| Ready queue | Shared | Orch push + Scheduler pop | mutex + CV |
| Completion queue | Shared | Worker push + Scheduler pop | mutex + CV (same CV) |
| Ring flow control | Shared | Orch alloc + Scheduler release | atomic + CV |
| Worker task_queue | Scheduler -> Worker | Scheduler push + Worker pop | Per-worker mutex |
| SubWorker mailbox | Thread <-> forked child | MAP_SHARED | acquire/release store |
| Tensor data | Shared memory | Orch + Worker | `share_memory_()` |

### 5.2 Dependency Inference and Scheduling

Fully consistent with L2, based on tensormap automatic dependency inference + eager dispatch:

1. On submit, for each INPUT tensor: query tensormap to find producer -> establish fanin dependency
2. On submit, for each OUTPUT tensor: write to tensormap to register as producer
3. After task submission, immediately check fanin_refcount — dispatch if all satisfied
4. After task completion, walk fanout list, release consumers -> newly ready tasks dispatch immediately

L3 does not use an independent DAG data structure. Dependencies are managed entirely by tensormap.

---

## 6. Scope Management

### 6.1 L2 Scope (Existing, Unchanged)

Device-side ring buffer intermediate buffer lifecycle management. Scope depth maps to different ring layers (up to 4), with inner scopes retiring independently.

### 6.2 L3 Scope (New, Isomorphic with L2)

Manages host-side intermediate tensor lifetimes:
- scope_begin -> record scope_tasks position, scope_stack_top++
- Tasks submitted within scope acquire a scope reference (fanout_count +1)
- scope_end -> iterate tasks in scope, release_scope_ref (fanout_refcount--)
- When fanout_refcount reaches fanout_count -> task transitions to CONSUMED, heap reclaimable
- Scope depth -> ring layer mapping (consistent with L2, independent ring retirement)

---

## 7. Usage

### 7.1 Worker Factory

All levels are created uniformly via `Worker(level, ...)` factory with a fully symmetric interface:

```python
worker = Worker(level=x, ...)   # Factory: constructs the appropriate Worker based on level
worker.register(fn) -> int      # Register SubWorker callable (call before init)
worker.init()                   # Allocate resources, fork child processes, start C++ threads
worker.run(task)                # Blocking execution of a single task
worker.close()                  # Release resources
```

### 7.2 L2 Single-Task Usage

```python
w2 = Worker(level=2, device_id=0, platform="a2a3sim", runtime="tmr")
w2.init()

for batch in batches:
    w2.run(WorkerPayload(callable=chip_callable, args=batch_args))

w2.close()
```

### 7.3 L3 Multi-Chip + SubWorker Usage

```python
w3 = Worker(level=3,
            chip_workers=[Worker(level=2, device_id=i, platform="a2a3sim", runtime="tmr")
                          for i in range(4)],
            num_sub_workers=4)

# Register SubWorker callables before fork
sub_cids = [w3.register(fn_i) for fn_i in postprocess_fns]
w3.init()

def my_orch(w, args):
    chip_results = []
    for i in range(4):
        r = w.submit(WorkerType.CHIP, chip_payload(i),
                     inputs=[args.input_ptrs[i]], outputs=[4*1024*1024])
        chip_results.append(r)

    for i in range(4):
        w.submit(WorkerType.SUB,
                 sub_payload(sub_cids[i]),
                 inputs=[chip_results[i].outputs[0].ptr])  # Automatic dependency inference

# run() is fully isomorphic with L2 ChipWorker.run(): accepts a task, returns on completion
w3.run(Task(orch=my_orch, args=my_args))

# Multiple rounds
for batch in batches:
    w3.run(Task(orch=my_orch, args=batch))

w3.close()
```

### 7.4 L4 Recursive Composition

```python
w4 = Worker(level=4,
            dist_workers=[
                Worker(level=3, chip_workers=[...], num_sub_workers=2),
                Worker(level=3, chip_workers=[...], num_sub_workers=2),
            ])
w4.init()

def l4_orch(w, args):
    # Each submit dispatches to an L3 node; the L3 node expands internally
    w.submit(WorkerType.DIST, l3_task_payload_a, inputs=[args.part_a])
    w.submit(WorkerType.DIST, l3_task_payload_b, inputs=[args.part_b])

w4.run(Task(orch=l4_orch, args=big_args))
w4.close()
```

### 7.5 Level Symmetry

```
w = Worker(level=N, ...)
w.run(task)
```

From the outside, a `Worker` at any level looks identical: accepts a task, blocks, returns on completion.
The caller does not need to know whether the task is executed by AICore, ChipWorker, or DistWorker.

---

## 8. Key Types

### 8.1 Type Name Mapping

| Document Name | Actual Type Name | Definition Location |
|---------------|-----------------|---------------------|
| `TensorArg` | `ContinuousTensor` | `src/common/task_interface/tensor_arg.h` |
| `SeparatedArgs` | `TaskArgs` | `src/common/task_interface/task_args.h` |
| `OrchArgs` | `DynamicTaskArgs` | `TaskArgs<ContinuousTensor, uint64_t, 0, 0>` |
| `OrchArgStorage` | `ChipStorageTaskArgs` | `TaskArgs<ContinuousTensor, uint64_t, 16, 128>` |

### 8.2 ChipCallable

ChipCallable replaces the binary payload role of ChipTask, containing the orch function and kernel binaries.
Invoked via `init_runtime(RuntimeHandle, ChipCallable*, ChipStorageTaskArgs*)` with 3 parameters.

---

## 9. Platform Services Layering

### 9.1 Independent .so per Platform Capability Domain

```
host_runtime.so   -> L2 runtime execution       (existing)
host_comm.so      -> Inter-chip communication    (used by L3)
host_network.so   -> Cross-node networking       (used by L4+, future)
```

Each level dlopen's only the .so it needs, without intruding on other levels.

### 9.2 Build Artifacts

```
build/lib/{arch}/{variant}/
  {runtime}/                         # runtime-dependent
    host_runtime.so                  # L2
    aicpu.so
    aicore.o
  comm/                              # runtime-independent
    host_comm.so                     # L3
```

---

## 10. File Layout

### 10.1 C++ Layer

```
src/common/distributed/                  # Scheduling engine, shared by L3/L4/L5/L6
  dist_types.h                           # IWorker interface + common types
  dist_tensormap.{h,cpp}                 # TensorMap
  dist_ring.{h,cpp}                      # TaskAllocator + RingSet + DepListPool
  dist_scope.{h,cpp}                     # ScopeManager
  dist_orchestrator.{h,cpp}              # submit 7-step flow
  dist_scheduler.{h,cpp}                 # Scheduler thread
  dist_sub_worker.{h,cpp}               # SubWorker: fork/shm
  dist_worker.{h,cpp}                    # DistWorker: exposed to Python

src/common/worker/
  chip_worker.{h,cpp}                    # Existing, used only by L3
```

### 10.2 Python Layer

```
python/bindings/
  task_interface.cpp                     # Existing (data types + ChipWorker)
  dist_worker_bind.cpp                   # New: DistWorker nanobind bindings

python/
  task_interface.py                      # Append DistWorker re-export
  host_worker/
    __init__.py
    host_worker.py                       # Thin wrapper, delegates to DistWorker(level=3)
    host_task.py                         # HostTask dataclass
```

---

## 11. PR Split Plan

### PR 2-1: C++ DistWorker Core + Nanobind Bindings

Add all files under `src/common/distributed/` + nanobind bindings + C++ unit tests.

### PR 2-2: Python HostWorker Wrapper + HostTask + Integration Tests

HostWorker Python thin wrapper + HostTask + mock/sim end-to-end tests.

### PR 2-3: DeviceRunner thread_local + Multi-Device

Change DeviceRunner singleton to thread_local to support multi-device parallelism.

### PR 2-4: Platform comm .so

Independent communication library (separate owner).

---

## 12. Relationship with the Linqu Distributed Runtime

This design (simpler Phase 2) corresponds to the L3 extension of simpler's existing capabilities described in [linqu_runtime_design.md](linqu_runtime_design.md) Section 3.1:

| Relationship | Description |
|-------------|-------------|
| **simpler L0-L2** | Not modified; ChipWorker calls via dlsym C API |
| **simpler L3 (this design)** | DistWorker + HostSubWorker, implemented within the simpler repository |
| **Linqu L3-L6** | pypto_runtime_distributed repository, uses simpler's ChipWorker/DistWorker as execution backend |
| **Interface integration** | Linqu's `ChipBackend` adapter calls simpler's C API via dlopen/dlsym |

### Recursive Relationship

```
Linqu L4+ Orchestrator
    | CALL_TASK (RPC)
Linqu L3 NodeDaemon
    | calls simpler Worker API
simpler DistWorker (L3)
    | submit(CHIP, ...) / submit(HOST_SUB, ...)
simpler ChipWorker (L2) / HostSubWorker
    | C API
simpler L0-L2 Runtime (executes on device)
```

---

## 13. Design Highlights Summary

1. **Unified IWorker interface** — ChipWorker / SubWorker / DistWorker all implement the blocking `run(payload)` interface
2. **Fork before threading** — HostSubWorker forks in `__init__()`, before C++ threads start
3. **Tensor zero-copy** — Achieved via `share_memory_()` + `/dev/shm` after fork
4. **No callable pickle constraint** — Registered before fork, child process inherits via COW, only ID is passed
5. **Scheduler does not poll Workers** — Key difference from L2: L3 uses completion_queue + CV instead of COND register polling
6. **TensorMap / Scope / Ring isomorphic with L2** — No new concepts; L3 is the Python/C++ version of L2
7. **Recursive Worker composition per level** — L4's DistWorker contains multiple L3 DistWorkers, naturally forming a tree hierarchy
