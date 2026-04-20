# 5. Physical View

This view describes how the Simpler runtime maps to infrastructure — nodes, networks, devices, and physical resources.

---

## 5.1 Deployment Topology

The runtime supports three deployment classes, each described as a configuration of the Machine Level Registry.

### 5.1.1 Single-Node Deployment (4 Levels)

The minimal production deployment: one host machine with one or more Ascend NPU devices.

```
┌─────────────────────────────────────────────────────────────────┐
│ "Host" (ordinal 3)                                               │
│ Scheduler: HostScheduler          Memory: Host DRAM              │
│ Workers: [Device₀, Device₁, ...]                                 │
│ Vertical↓: DMA Control Channel                                   │
│                                                                   │
│   ┌───────────────────────────────────────────────────────────┐  │
│   │ "Device" (ordinal 2, per Device instance)                  │  │
│   │ Scheduler: DeviceOrchestrator   Memory: HBM                │  │
│   │ Workers: [AICPU Thread₀, Thread₁, ...]                    │  │
│   │ Vertical↓: SharedMemory Channel                            │  │
│   │                                                             │  │
│   │   ┌─────────────────────────────────────────────────────┐  │  │
│   │   │ "Chip" (ordinal 1)                                   │  │  │
│   │   │ Scheduler: AicpuScheduler   Memory: Shared Memory    │  │  │
│   │   │ Workers: [AICore₀, AICore₁, ..., AICore₇₁]          │  │  │
│   │   │ Vertical↓: RegisterBank Channel (ACK/FIN)            │  │  │
│   │   │ Horizontal: TPUSH/TPOP Channel (AIC↔AIV)            │  │  │
│   │   │                                                       │  │  │
│   │   │   ┌─────────────────────────────────────────────┐    │  │  │
│   │   │   │ "Core" (ordinal 0, leaf)                     │    │  │  │
│   │   │   │ Dispatcher: AICore dispatch (register poll)  │    │  │  │
│   │   │   │ Memory: L1/L0A/L0B/L0C (Scratchpad)         │    │  │  │
│   │   │   │ Execution: Cube | Vector | Scalar            │    │  │  │
│   │   │   └─────────────────────────────────────────────┘    │  │  │
│   │   └─────────────────────────────────────────────────────┘  │  │
│   └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

**Resource requirements (a2a3):**

| Level | Workers | Memory | Threads |
|-------|---------|--------|---------|
| Host | 1 Device | Host DRAM | 1 scheduler + 1 worker |
| Device | 4 AICPU threads | HBM (32 GB typical) | 4 worker threads |
| Chip | 72 AICores | Shared Memory (SRAM) | 1 scheduler thread |
| Core | 1 compute pipeline | L1/L0A/L0B/L0C (scratchpad) | Hardware pipeline |

### 5.1.2 Multi-Node Deployment (5 Levels)

Adds a `"Pod"` level above `"Host"` for cross-node scheduling:

```
                    ┌────────────────────────────────────┐
                    │ "Pod" (ordinal 4)                   │
                    │ Scheduler: DistributedScheduler     │
                    │ Memory: RDMA-registered buffers     │
                    │ Workers: [Node₀, Node₁, ...]       │
                    │ Horizontal: RDMA Channel            │
                    │ Vertical↓: TCP Message Channel      │
                    └──────┬──────────────┬──────────────┘
                           │              │
              ┌────────────▼──┐     ┌─────▼──────────┐
              │ Node₀         │     │ Node₁          │
              │ ┌───────────┐ │     │ ┌───────────┐  │
              │ │"Host"     │ │     │ │"Host"     │  │
              │ │ ┌───────┐ │ │     │ │ ┌───────┐ │  │
              │ │ │"Device"│ │ │     │ │ │"Device"│ │  │
              │ │ │┌─────┐│ │ │     │ │ │┌─────┐│ │  │
              │ │ ││"Chip"││ │ │     │ │ ││"Chip"││ │  │
              │ │ │└─────┘│ │ │     │ │ │└─────┘│ │  │
              │ │ └───────┘ │ │     │ │ └───────┘ │  │
              │ └───────────┘ │     │ └───────────┘  │
              └───────┬───────┘     └────────┬───────┘
                      │  Horizontal Channel  │
                      └──────────────────────┘
                        (RDMA / TCP / SHM)
```

**Additional resources per Pod:**

| Component | Requirement |
|-----------|-------------|
| Network | RDMA-capable NIC (InfiniBand / RoCE) or TCP fallback |
| Memory | RDMA-registered host memory pool per node |
| Threads | Network I/O threads + DistributedScheduler thread per node |

### 5.1.3 Full Hierarchy Deployment (8 Levels)

For large-scale clusters (Linqu 8-level conceptual hierarchy):

| Ordinal | Level Name | Notes |
|---------|-----------|-------|
| 0 | `"Core"` | AICore compute pipeline (leaf) |
| 1 | `"CoreGroup"` | Co-scheduled AIC + AIV group; horizontal TPUSH/TPOP |
| 2 | `"ChipDie"` | Single die within multi-die chip (elided on single-die) |
| 3 | `"Chip"` | Full chip with AICPU + AICore clusters |
| 4 | `"Host"` | Host machine, multi-device |
| 5 | `"Pod"` | Rack-level cluster |
| 6 | `"Supernode"` | Multi-rack |
| 7 | `"Global"` | Global coordinator |

Levels not needed in a deployment are elided via `instance_count = 0`.

---

## 5.2 Network Topology

### 5.2.1 Intra-Node Communication

| Path | Protocol | Bandwidth | Latency |
|------|----------|-----------|---------|
| Host ↔ Device | DMA via CANN runtime | ~16 GB/s (PCIe Gen4 x16) | ~μs (DMA setup) |
| Device ↔ Chip (AICPU ↔ AICore) | Shared memory + polling | On-chip bandwidth | ~ns |
| Core ↔ Core (within CoreGroup) | TPUSH/TPOP flag ring buffer | On-chip SRAM bandwidth | ~ns |
| Chip ↔ Core (AICPU → AICore) | Register Bank (MMIO) | Register write latency | ~ns |

### 5.2.2 Inter-Node Communication

| Path | Protocol Options | Typical Bandwidth | Typical Latency |
|------|-----------------|-------------------|-----------------|
| Host ↔ Host (within Pod) | RDMA (InfiniBand/RoCE), TCP | 100–400 Gbps (RDMA) | 1–5 μs (RDMA) |
| Pod ↔ Pod (Supernode) | TCP/RDMA across switches | 100 Gbps | 5–50 μs |
| Supernode ↔ Supernode (Cluster) | Compressed TCP, WAN-optimized | Variable | ms range |

### 5.2.3 Platform Capability Discovery

Each Platform provides a capability struct describing:
- Core count, memory sizes, supported features.
- Profiling capabilities, simulation mode flag.
- Network topology (neighbors, bandwidth, latency).
- Inter-device link information.

Used by the runtime and distributed scheduler to adapt behavior (e.g., partition strategy selection based on data locality and link bandwidth).

---

## 5.3 Scaling Strategy

### 5.3.1 Vertical Scaling (Within a Node)

| Dimension | Mechanism |
|-----------|-----------|
| More AICores | Increase `worker_count` at Chip level (platform-determined) |
| More AICPU threads | Increase `worker_thread_count` at Device level |
| More devices per host | Add Device instances under Host level |
| More memory | Larger HBM / host DRAM (platform-determined) |

### 5.3.2 Horizontal Scaling (Across Nodes)

| Dimension | Mechanism |
|-----------|-----------|
| More nodes in Pod | Increase `worker_count` at Pod level; DistributedScheduler partitions work across additional nodes |
| More Pods in Supernode | Add Pod instances under Supernode level |
| Cross-cluster | Add Supernode/Cluster levels to the registry |

**Scaling trigger:** Adding capacity requires only updating the Machine Level Registry configuration (worker counts, instance counts) — no code changes.

### 5.3.3 Dynamic Scaling

- Level elision enables runtime adaptation (e.g., disable `"Pod"` level for single-node runs).
- Partition strategies (ROUND_ROBIN, DATA_LOCALITY, WORK_STEALING) adapt to changing node counts.
- [ASSUMPTION] Dynamic node addition/removal at runtime is not in scope for the initial design. The registry is frozen at initialization.

> [UPDATED: A10-P7: Deployment cue for opt-in admission sharding (two-tier TaskManager).]
> Enable the opt-in sharded `TaskManager` path when `concurrent_submitters × cluster_nodes ≥ 64`. Below this threshold, the default single-threaded fast path (`shards = 1`) remains normative. Shard count defaults: **Host = 8, Device = 4, Chip and below = 1** (see A10-P1). See ADR-019 ("Admission shard default + deployment cue") and `02-logical-view/02-scheduler.md` "Concurrency and locking".
>
> [UPDATED: A10-P9: `WorkStealingPartitioner × RETRY_ELSEWHERE` requires a live assignment log; FUNCTIONAL mode preserves it.]
> This combination is admissible only if an assignment log `(submission_id, task_index) → final_node_id` is written **before** dispatch. Under A9-P6 option (iii) the runtime ships `FUNCTIONAL` simulation mode only, and the log is live (in-memory) — REPLAY is not required. The log is preserved across retries so that `RETRY_ELSEWHERE` can honor per-task idempotency semantics (A5-P4). Authoritative definition in `modules/distributed.md §3.1`.

---

## 5.4 Availability and Disaster Recovery

### 5.4.0 Critical Path SPOF Analysis (Rule R5)

Every component on the critical path must have a redundancy or fallback strategy. The following table identifies single points of failure and their mitigations:

| Component | SPOF? | Criticality | Redundancy / Fallback Strategy |
|-----------|-------|-------------|-------------------------------|
| Host process | **Yes** | Critical | External process monitor restarts; in-flight tasks lost (see [Known Deviation 2](10-known-deviations.md)). Application-level re-submission. |
| Host Scheduler thread | **Yes** | Critical | Configurable `scheduler_thread_count` (default 1). Single thread is SPOF for that level. Mitigation: crash detection triggers runtime shutdown with error propagation to Python. |
| Device (NPU) | No (multi-device) | High | Multiple devices under Host level. Scheduler avoids failed devices; remaining devices continue. Degraded capacity, not total failure. |
| AICPU thread | No (multi-thread) | Medium | Multiple worker threads per Device. Single thread crash propagates error to parent task; other threads continue. |
| AICore | No (multi-core) | Low | 72+ AICores per Chip. Failed core marked unavailable; Scheduler dispatches to remaining cores. |
| Network link (distributed) | **Yes** (per-link) | High | Multiple network paths (RDMA + TCP fallback). If all paths fail, heartbeat timeout triggers `failure_policy`. |
| Coordinator node (Pod) | **Yes** | Critical | No automatic coordinator failover in initial design. [ASSUMPTION] Coordinator election is deferred to future work (see Q5 in [09-open-questions.md](09-open-questions.md)). Mitigation: application-level re-submission from a different coordinator. |

**Summary:** The primary SPOFs are (1) the host process (inherent to a computation engine — mitigated by external monitoring), (2) the coordinator node in distributed mode (mitigated by application-level restart), and (3) individual network links (mitigated by transport fallback). All hardware-level components (Devices, AICores, AICPU threads) have built-in redundancy through multiple instances.

> [UPDATED: A10-P2: Deployment-mode variants for coordinator failover; `cluster_view` bumps on coordinator demote.]
> The v1 coordinator-failure model (fail-fast, absorbed into A5-P3) has **three deployment-mode variants**, selected in `DeploymentConfig`:
>
> 1. **`SingleCoordinator` (default v1):** one coordinator per Pod; on failure, surviving peers surface `CoordinatorLost` within `heartbeat_timeout_ms`; Python driver sees `DistributedError`; application-level re-submission.
> 2. **`StickyCoordinator` (v1 opt-in):** coordinator role pinned to one member of a small sticky-routing set; on failure, **scope is pinned to the failed Pod only** and `cluster_view` generation bumps to the surviving-coordinator list. No cluster-wide fail-closed.
> 3. **`QuorumCoordinator` (v2 roadmap, not implemented in v1):** decentralized via quorum-promoted `cluster_view` generation; recorded as an ADR-005 extension.
>
> On every coordinator demote / promote in modes (2)–(3), `cluster_view.generation` monotonically bumps and is included in the next `HEARTBEAT` and outbound `REMOTE_*` envelopes; peers with an older generation retry with backoff (A5-P1). See `modules/distributed.md` Coordinator Membership.

### 5.4.1 Single-Node Failure Modes

| Failure | Detection | Response |
|---------|-----------|----------|
| AICore hang | Timeout on Task dispatch (ACK/FIN register) | Mark core unavailable; Scheduler avoids dispatching to it |
| AICPU thread crash | Worker thread exit detection | Propagate error to parent Task; attempt re-execution if idempotent |
| Device failure | HAL health check failure | Propagate `HardwareFault` to Host; drain remaining tasks |
| Host process crash | OS-level crash | External process monitor restarts; in-flight tasks are lost |

### 5.4.2 Distributed Failure Modes

| Failure | Detection | Response |
|---------|-----------|----------|
| Node unreachable | Heartbeat timeout | DistributedScheduler marks node FAILED; policy: abort, retry on alternate, or continue reduced |
| Network partition | Message delivery timeout | Affected cross-node dependencies time out; `REMOTE_ERROR` propagated |
| Partial failure | Heartbeat from subset of nodes | Policy-dependent: abort all, continue with available nodes, or retry failed partitions |

### 5.4.3 Recovery Strategy

- **Task-level recovery:** Recoverable errors (timeout, resource busy) may trigger retry with backoff (Rule R2).
- **Node-level recovery:** Failed node's tasks are re-submitted to alternate nodes if partition strategy supports it.
- **No persistent state recovery:** The runtime does not persist task state across process restarts. Recovery is at the Python-level (user re-submits the computation).

---

## 5.5 Deployment Configuration

A deployment is fully described by a **Deployment Configuration**:

1. **Machine Level selection** — which levels to register and which to elide.
2. **Implementation binding** — factory registrations for all six component interfaces per level.
3. **Parameter configuration** — `LevelParams` per level (thread counts, queue sizes, memory sizes, DSL aliases).
4. **Layer Stack assembly** — runtime constructs Layer instances top-down, wires channels, initializes.
5. **Function deployment** — register Functions into each level's Function Cache.

Configuration sources:
- C++ initialization sequence (compile-time binding).
- YAML/JSON configuration file (runtime loading for dynamic deployments).
- Combination: static factory registration with runtime-supplied parameters.

### 5.5.1 Peer Discovery

For distributed levels:
- **Config-file-based** (default): Static peer list in deployment configuration. Simple, predictable.
- **Gossip-based** (optional): Dynamic discovery via SWIM protocol. Enabled by `discovery_mode = "gossip"` in `LevelParams`.

### 5.5.2 Multi-Tenancy

Multiple **Logical Systems** (named, isolated namespaces) MAY share physical Machine Level instances:
- TaskKeys scoped to Logical System.
- Ring buffers, tensor buffers, Function Caches partitioned per Logical System.
- Scheduler dispatches only within the same Logical System.
- Horizontal Channel messages filtered by `logical_system_name`.

Multi-tenancy is optional; single-tenant deployments use a default Logical System name with no overhead.
