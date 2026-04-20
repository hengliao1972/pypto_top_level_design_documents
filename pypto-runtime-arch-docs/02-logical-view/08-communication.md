# 2.5 Communication Model

> Part of the [Logical View](../02-logical-view.md). This module describes the **communication model** layered over the Vertical and Horizontal Channels registered by each Machine Level: message types, transport memory models, reference transport implementations, data movement, and data access models.

Communication is organized through the channels registered by Machine Levels along two axes:

1. **Vertical** (parent↔child): between adjacent Layers via `IVerticalChannel` ([§2.2.5](05-machine-level-registry.md#225-vertical-channel-parentchild-communication)).
2. **Horizontal** (sibling↔sibling): between instances at the same Layer via `IHorizontalChannel` ([§2.2.6](05-machine-level-registry.md#226-horizontal-channel-sibling-communication)).

## 2.5.1 Message Types

| Category | Messages | Channel |
|----------|----------|---------|
| **Scheduling** | `TASK_SUBMIT`, `TASK_READY`, `TASK_DISPATCH`, `TASK_COMPLETE`, `TASK_RETIRE` | Vertical |
| **Remote Scheduling** | `REMOTE_SUBMIT`, `REMOTE_DEP_NOTIFY`, `REMOTE_COMPLETE`, `REMOTE_DATA_READY`, `REMOTE_ERROR` | Horizontal (distributed) |
| **Data Movement** | `DMA_REQUEST`, `DMA_COMPLETE`, `RDMA_WRITE`, `RDMA_READ` | Both |
| **Control** | `SHUTDOWN`, `BARRIER`, `HEARTBEAT` | Both |

The message set is extensible per Machine Level.

## 2.5.2 Transport Memory Models

Different transport implementations operate under different memory model assumptions:

| Memory Model | Typical Levels | Addressing | Synchronization | Back-Pressure |
|-------------|---------------|-----------|-----------------|---------------|
| Shared address space | L0–L2 (intra-chip) | Direct pointers | Hardware atomics | Spin on ring pointers |
| DMA-bridged domains | L2↔L3 (chip↔host) | Separate address spaces | DMA completion + MMIO | DMA queue back-pressure |
| Independent processes | L3↔L4+ (host↔pod) | Opaque handles | Message acknowledgment | Message-driven |

The abstract interfaces are agnostic to these models — each implementation encapsulates its model internally.

Channel message buffers, ring-buffer head/tail pointers, and DMA descriptor tables are drawn from the Layer's **control-plane Memory Region** (e.g. `CONTROL_AREA` or `MSG_BUFFERS`), not the bulk data region. When the producer and consumer of a channel run on different cores or threads, buffers are allocated with `PlacementHint.pad_to_cache_line = true` to avoid false sharing, and head/tail pointers with `isolate_cache_line = true` to enforce single-writer-per-line. Cache-line size is read from the region descriptor (`IMemoryManager::describe`) — see [§2.1.8 Access-Efficiency Considerations](04-memory.md#218-access-efficiency-considerations).

## 2.5.3 Reference Transport Implementations

| Transport | Channel Type | Boundary | Mechanism |
|-----------|-------------|----------|-----------|
| Register Bank | Vertical | Chip → Core | Memory-mapped registers (ACK/FIN protocol via `COND` register) |
| Ring Buffer | Vertical | Device → Chip | Bounded circular FIFO in shared memory |
| DMA Control | Vertical | Host → Device | MMIO writes + DMA descriptors + shared memory flags |
| TPUSH/TPOP | Horizontal | Core ↔ Core (CoreGroup) | Hardware flag ring buffers with split `tpop`/`tfree` |
| Network Link | Horizontal | Host ↔ Host (Pod+) | Message passing + RDMA + collective primitives |

## 2.5.4 Data Movement

Data Movement is the transfer of tensor data between Memory Scopes. It is modeled as a first-class Task type, scheduled like any other Task, and executed via `IMemoryOps` ([§2.1.7](04-memory.md#217-memory-operations)).

**Vertical Data Movement** uses `copy_to_child()` / `copy_from_child()`.
**Horizontal Data Movement** uses `copy_to_peer()` / `copy_from_peer()`.
**Asynchronous Data Movement** uses `copy_async()` / `wait()` / `poll()` with `complete_in_future` marking.

## 2.5.5 Data Access Models

Two complementary models coexist:

**Shared Memory Model:**
- Intra-level: all Workers share their Layer's Memory Scope.
- Cross-level: `IMemoryOps` address translation enables shared regions across adjacent levels.
- Distributed: shared memory services (OpenSHMEM-style) MAY expose cross-node regions.

**Distributed (Message-Passing) Model:**
- Private address spaces per instance.
- Explicit data movement via `IMemoryOps` or Data Movement Tasks.
- Handle-based references (opaque, not memory pointers).

**External Data Services** (integration points only):

| Service | Description | Integration Point |
|---------|-------------|-------------------|
| Distributed Shared Memory | Cross-node shared, accessible L0–L2 | `IMemoryOps` shared regions |
| Block Storage | Async block I/O to UB-attached devices | Data movement Task with tensor output |
| Distributed File System | POSIX-style file APIs | Host Function or Worker tasks |
| Key-Value Database | Redis-compatible K/V | Host Function or Worker tasks |
