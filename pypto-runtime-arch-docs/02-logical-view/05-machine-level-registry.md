# 2.2 Machine Level Registry

> Part of the [Logical View](../02-logical-view.md). This module describes how the Layer hierarchy is composed from **Machine Levels** — named, registered layer types with pluggable factories for Scheduler, Workers, Memory Manager, Memory Operations, and communication channels (Vertical and Horizontal).

The Abstract Machine does **not** hardcode a fixed set of topology levels. Instead, the topology is built from **Machine Levels** — named, registered layer types with pluggable implementations. This allows the same framework to model a 3-level single-chip deployment, a 5-level single-node deployment, or an 8-level multi-cluster deployment without structural code changes.

## 2.2.1 Machine Level

A Machine Level is a registered layer type that defines one tier in the hierarchy:

| Property | Type | Description |
|----------|------|-------------|
| `level_name` | `string` | Unique name (e.g., `"Core"`, `"Chip"`, `"Host"`, `"Pod"`) |
| `ordinal` | `uint32_t` | Position in the Layer Stack (0 = leaf, N = root) |
| `scheduler_factory` | `ISchedulerFactory*` | Creates the `ISchedulerLayer` implementation |
| `worker_factory` | `IWorkerFactory*` | Creates Workers |
| `memory_manager_factory` | `IMemoryManagerFactory*` | Creates the Memory Manager |
| `memory_ops_factory` | `IMemoryOpsFactory*` | Creates the Memory Operations handler |
| `vertical_channel_factory` | `IVerticalChannelFactory*` | Creates the parent↔child control channel (`nullptr` for leaf) |
| `horizontal_channel_factory` | `IHorizontalChannelFactory*` | Creates the sibling↔sibling control channel (`nullptr` if no sibling comm) |
| `level_params` | `LevelParams` | Per-level configuration (thread counts, queue sizes, memory sizes, `event_queue_capacity`, `event_handling_config`, `event_loop_deployment`). **[UPDATED: A9-P7]** The former separate `source_collection_config` is folded into `event_handling_config` (source list, per-source weight, max events per cycle). **[UPDATED: A2-P2]** `LevelOverrides` remains typed and closed for v1; schema-registered v2 is recorded in Q6 with a trigger (first concrete out-of-tree level). Validation runs at `deployment_parser` init only. **[UPDATED: A9-P2]** Deployment and policy pluggability collapse to closed enums `DeploymentMode { SINGLE_THREADED, SPLIT_COLLECTION }` and `PolicyKind { ... }`; future-extension interfaces are appendix-listed in ADR-010. |
| `is_leaf` | `bool` | `true` for the lowest level |
| `task_schedule_policy_factory` | `ITaskSchedulePolicyFactory*` | Creates the task scheduling policy (`nullptr` → default FIFO) |
| `worker_selection_policy_factory` | `IWorkerSelectionPolicyFactory*` | Creates the worker selection policy (`nullptr` → default round-robin) |
| `resource_allocation_policy_factory` | `IResourceAllocationPolicyFactory*` | Creates the resource allocation policy (`nullptr` → default greedy) |
| `worker_types` | `std::vector<WorkerTypeDescriptor>` | Worker Types available at this level ([§2.1.4.2](03-worker.md#2142-heterogeneous-worker-types-and-worker-groups)). Empty = single implicit homogeneous type |
| `task_exec_types` | `std::vector<TaskExecType>` | Task Execution Types supported at this level ([§2.4.8](07-task-model.md#248-task-execution-types)). Empty = single implicit type |
| `worker_groups` | `std::vector<WorkerGroup>` | Worker Group definitions ([§2.1.4.2](03-worker.md#2142-heterogeneous-worker-types-and-worker-groups)). Empty = no grouping (flat pool) |
| `event_source_factories` | `std::vector<IEventSourceFactory*>` | Creates event sources for the Scheduler's event loop ([§2.1.3.5](02-scheduler.md#2135-event-driven-execution-model), [§2.6.4](09-interfaces.md#264-event-and-execution-interfaces)). Empty = default queue-based source |
| `execution_policy_factory` | `IExecutionPolicyFactory*` | Creates the execution policy for Scheduler–Worker deployment mode ([§2.1.3.6](02-scheduler.md#2136-schedulerworker-deployment-flexibility), [§2.6.4](09-interfaces.md#264-event-and-execution-interfaces)). `nullptr` → `DedicatedExecutionPolicy` |

Each Machine Level defines **six** core component interfaces, up to **three** optional scheduling policy interfaces, optional worker topology, and optional event/execution configuration:

```
Machine Level
├── Execution
│   ├── ISchedulerFactory     → creates ISchedulerLayer
│   └── IWorkerFactory        → creates Workers
├── Memory
│   ├── IMemoryManagerFactory → creates IMemoryManager
│   └── IMemoryOpsFactory     → creates IMemoryOps
├── Communication
│   ├── IVerticalChannelFactory   → creates IVerticalChannel
│   └── IHorizontalChannelFactory → creates IHorizontalChannel
├── Policy (optional — defaults provided)
│   ├── ITaskSchedulePolicyFactory     → creates ITaskSchedulePolicy
│   ├── IWorkerSelectionPolicyFactory  → creates IWorkerSelectionPolicy
│   └── IResourceAllocationPolicyFactory → creates IResourceAllocationPolicy
├── Worker Topology (optional — defaults to homogeneous flat pool)
│   ├── worker_types[]               → WorkerTypeDescriptor definitions
│   ├── task_exec_types[]            → TaskExecType definitions
│   └── worker_groups[]              → WorkerGroup definitions
└── Event & Execution (optional — defaults provided)
    ├── IEventSourceFactory[]        → creates IEventSource instances
    └── IExecutionPolicyFactory      → creates IExecutionPolicy
```

Policy factories are optional. When `nullptr`, the Scheduler uses built-in defaults: FIFO task ordering, round-robin worker selection, greedy resource allocation. Custom policies are injected at construction time and remain fixed for the lifetime of the Layer instance.

A Machine Level is a **type** (blueprint), not an instance. At runtime, each is instantiated zero or more times depending on physical topology.

## 2.2.2 Registration

Machine Levels are registered into a **Machine Level Registry** at compile time or system initialization, before the Layer Stack is assembled. Registration order defines the hierarchy (leaf first, root last):

```cpp
class MachineLevelRegistry {
public:
    void register_level(const MachineLevelDescriptor& desc);
    size_t level_count() const;
    const MachineLevelDescriptor& get_level(uint32_t ordinal) const;
    const MachineLevelDescriptor* find_level(const std::string& name) const;
    void freeze();  // No more registrations; validates stack integrity
};
```

Constraints:
- Exactly one leaf level (`is_leaf = true` on ordinal 0).
- All non-leaf levels supply a `vertical_channel_factory`.
- `horizontal_channel_factory` is optional at any level.

## 2.2.3 DSL Level Alias (`pl.Level`)

Each Machine Level MAY be assigned DSL-level aliases (`pl.Level` enum values) as part of deployment configuration. The mapping from `pl.Level` to Machine Level name is **not hardcoded** — it is established at registration time via the `dsl_aliases` field in `MachineLevelDescriptor`.

The compiler tags every outlined function with a `pl.Level` label. At runtime, this label is resolved to a Machine Level ordinal via the registry's alias table.

## 2.2.4 Level Elision

A registered Machine Level MAY be elided by setting `instance_count = 0` in `LevelParams`. An elided level is skipped when the Layer Stack is assembled — the parent connects directly to the next non-elided child.

## 2.2.5 Vertical Channel (Parent–Child Communication)

A **Vertical Channel** (`IVerticalChannel`) is the control communication mechanism between adjacent levels:

- **Downward**: Task submissions, dispatch commands, scope signals, data movement requests.
- **Upward**: Task completions, dependency notifications, error reports, profiling data.

```cpp
class IVerticalChannel {
public:
    virtual ~IVerticalChannel() = default;

    virtual Status submit_to_child(const TaskDescriptor& desc) = 0;
    virtual Status send_scope_signal(ScopeSignal signal) = 0;
    virtual Status send_data_movement(const DataMovementRequest& req) = 0;

    virtual Status notify_parent_complete(TaskHandle child, const CompletionInfo& info) = 0;
    virtual Status notify_parent_error(TaskHandle child, const ErrorContext& err) = 0;

    virtual void init(const VerticalChannelConfig& config) = 0;
    virtual void shutdown() = 0;
};
```

Typical implementations per boundary:

| Parent → Child | Implementation |
|----------------|----------------|
| Host → Device | DMA commands via HAL, MMIO control registers |
| Device → Chip | On-chip shared memory writes, interrupt/polling |
| Chip → Core | Register Bank writes (ACK/FIN protocol) |
| Pod → Host | TCP/Unix socket message passing |
| Supernode → Pod | TCP/RDMA message passing |

## 2.2.6 Horizontal Channel (Sibling Communication)

A **Horizontal Channel** (`IHorizontalChannel`) connects sibling instances at the same Machine Level:

- **Data movement**: Tensor transfers (inter-node RDMA, intra-chip TPUSH/TPOP).
- **Coordination**: Collective operations, peer-to-peer scheduling messages, barriers.

```cpp
class IHorizontalChannel {
public:
    virtual ~IHorizontalChannel() = default;

    virtual Status send(uint32_t peer_index, const Message& msg) = 0;
    virtual Status recv(uint32_t peer_index, Message* out) = 0;
    virtual bool   poll(uint32_t peer_index) = 0;

    virtual Status transfer(uint32_t peer_index, const DataTransferRequest& req) = 0;

    virtual void init(const HorizontalChannelConfig& config) = 0;
    virtual void shutdown() = 0;
};
```

> [UPDATED: A9-P3: collectives removed from `IHorizontalChannel`; Q4 resolved option A]
> The speculative collective operations `barrier()` and `all_reduce()` are **removed** from `IHorizontalChannel`. The interface keeps only `send` / `recv` / `poll` / `transfer`. Collective operations are composed at the Orchestration level (Q4 option A); a future `ICollectiveOps` sibling interface is appendix-listed in ADR-010. Native HCCL (or equivalent) hooks will be re-added only when concrete performance data arrives. See ADR-008 and `09-open-questions.md` Q4 for the resolution trail; see `modules/distributed.md` for the composition path.

> [UPDATED: A2-P2: `extension_map` + closed v1 variants in `MachineLevelDescriptor`]
> `MachineLevelDescriptor` exposes an opaque `extension_map : string → bytes` for forward-compat payloads (open for extension) while all first-class fields above remain **closed** variants for v1. New fields require an explicit versioned promotion from `extension_map` in a minor release (see `appendix-c-compatibility.md` BC policy). This keeps the hot-path factory wiring static while admitting out-of-tree experiments.

| Level | Implementation | Use Case |
|-------|---------------|----------|
| `"Core"` / `"CoreGroup"` | TPUSH/TPOP hardware flag ring buffer | Intra-cluster data exchange |
| `"Host"` | Shared memory IPC, Unix domain sockets | Multi-process coordination |
| `"Pod"` | RDMA / InfiniBand | Inter-node tensor transfer |
| `"Supernode"` | TCP / RDMA across racks | Cross-pod data movement |
| `"Cluster"` | Compressed TCP, WAN-optimized | Cross-rack with contracted bandwidth |

## 2.2.7 Machine Level Instance

A Machine Level Instance is a runtime instantiation identified by `(level_ordinal, instance_index)`. At runtime, the Layer Stack assembler creates instances by walking the registry top-down, creating the registered number of instances per level, and wiring vertical channels between parents and children.

## 2.2.8 Topology as Configuration, Not Code

The machine topology is fully described by:
1. The ordered sequence of Machine Levels in the registry.
2. The `LevelParams` per level.
3. The factory bindings per level.

Adding a new level requires only implementing the six component interfaces and registering. No existing code is modified.
