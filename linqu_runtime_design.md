# Linqu Distributed Runtime: Full System Design

This document provides a comprehensive design for the **Linqu Distributed Runtime** — the software system that executes PyPTO programs across a hierarchical cluster of machines. It synthesizes and extends the Linqu machine hierarchy model, the PyPTO scope/function grammar, the multi-layer ring buffer memory management system, and the distributed cluster orchestration architecture.

---

## 1. Design Goals and Principles

### 1.1 Hierarchical Symmetry

The runtime software architecture must **mirror** the physical Linqu machine hierarchy (Levels 0–6). Every runtime component — task identity, memory management, scheduling, and communication — is structured around the same hierarchy that the hardware and compiler use. There is no separate "software topology" distinct from the machine model.

### 1.2 Deterministic O(1) Resource Management

Resource lifetime and reclamation use **ring buffers** indexed by scope depth, not garbage collection. Allocation and retirement are O(1) per operation. Inner scopes can retire independently of outer scopes.

### 1.3 Zero-Configuration Cluster Discovery

Nodes identify themselves and discover peers using **deterministic, rule-based IP-to-coordinate mapping** and **gossip-based membership**. No static IP lists, no centralized DNS, no manual topology files.

### 1.4 Code and Data Residency

Minimize network overhead via a **two-phase model**: register code/data blobs once (by content hash), invoke many times by handle. Remote nodes are treated as **persistent environments**, not ephemeral task executors.

### 1.5 Logical Isolation (Multi-Tenancy)

A single physical cluster supports multiple **Logical Linqu Systems** identified by namespace strings. Different logical systems sharing the same hardware cannot observe or interfere with each other's memory, tasks, or ring buffers.

### 1.6 Unified Programming Model

The same `pl.Level.*` hierarchy parameter, `pl.at()` scope grammar, and `@pl.function(level=...)` decorator work consistently from core-level kernels (Level 0) up to cross-rack cluster programs (Level 6). The compiler generates **hierarchy labels** on all functions, reserved for the runtime to dispatch at the appropriate level.

---

## 2. The Linqu Machine Hierarchy

### 2.1 Levels Defined Bottom-Up

| Level | Name | Description | pl.Level aliases |
|-------|------|-------------|------------------|
| **0** | **Core / Core-group** | **Core:** single execution unit (AIV or AIC). **Core-group:** scheduling unit grouping multiple cores with local affinity (e.g. 1 AIC + 2 AIV), sharing TPUSH/TPOP ring; runs InCoreFunctionGroup. | `AIV`, `AIC`, `CORE_GROUP` |
| **1** | **Chip die** | One die; contains multiple cores/core-groups. May be **omitted** in single-die chip models. | `CHIP_DIE`, `L2CACHE` |
| **2** | **Chip** | One chip; contains one or more dies or directly multiple cores. Unified memory address space (UMA). | `CHIP`, `PROCESSOR`, `UMA` |
| **3** | **Host** | A single OS instance; one or more chips; runs orchestration. | `HOST`, `NODE` |
| **4** | **Cluster-level-0** | First cluster tier; usually a single server or pod; high bandwidth, tight coupling. | `CLUSTER_0`, `POD` |
| **5** | **Cluster-level-1** | Second cluster tier; usually a supernode; high-bandwidth domain across nodes. | `CLUSTER_1`, `CLOS1` |
| **6** | **Cluster-level-2** | Third cluster tier; cross-rack or wider-area with contracted bandwidth. | `CLUSTER_2`, `CLOS2` |

### 2.2 Recursive Enclosure

Each level is a logical machine that **encloses several instances** of the level below, recursively forming the complete system:

```
Cluster-level-2  encloses several →  Cluster-level-1
Cluster-level-1  encloses several →  Cluster-level-0
Cluster-level-0  encloses several →  Host
Host             encloses several →  Chip
Chip          (opt) encloses several →  Chip die
Chip / Chip die  encloses several →  Core / Core-group
```

### 2.3 Bandwidth and Coupling Gradient

As level number increases, the domain grows and interconnect bandwidth becomes more constrained:

- **Level 0** (Core-group): intra-cluster DMA, TPUSH/TPOP — highest bandwidth, lowest latency.
- **Level 1–2** (Chip die / Chip): on-chip interconnect, shared L2 cache or UMA memory.
- **Level 3** (Host): PCIe, NVLink, or similar host-to-device links.
- **Level 4** (Pod): high-bandwidth intra-server or intra-pod network (e.g. NVLink, InfiniBand within rack).
- **Level 5** (Supernode): high-bandwidth domain across nodes (e.g. fat-tree spine within supernode).
- **Level 6** (Cross-rack): contracted bandwidth, multi-hop routing.

The runtime uses this gradient to make communication decisions: prefer local data transfer at lower levels; serialize and RPC at higher levels.

---

## 3. Existing Runtime (`simpler`), Project Scope, and Extension Plan

### 3.1 The `simpler` Runtime: Level 0–2 (Do Not Modify)

The existing runtime, implemented in the **`pypto_workspace/simpler`** repository, already handles **Levels 0–2** of the Linqu hierarchy:

- **Level 0** (Core / Core-group): InCore functions, AIC/AIV kernels, and InCoreFunctionGroups execute on cores and core-groups. TPUSH/TPOP co-scheduling within a core-group.
- **Level 2** (Chip): **Orchestration functions** run at the chip level. The `simpler` runtime manages task submission, ring buffers (single-ring today), tensor lifetime, scope-exit semantics, and producer/consumer tracking — all within a single chip.
- **Level 1** (Chip die): **Omitted** on current chip models. In the future, the `simpler` repo will be enhanced to support Level 1 when multi-die chips become available. This is an internal `simpler` concern.

**Guiding rule: Do not change `simpler`.** The `simpler` design and codebase are treated as a **fixed, existing capability**. This project **adapts to** the `simpler` interfaces and **builds on top of** them, never modifying them.

### 3.2 This Project: Build the Level 2–6 Runtime

This project's scope is to build a **coherent runtime that covers Levels 2 through 6** (Chip through Cluster-level-2) in one unified design:

- **Level 2** (Chip): the interface layer — this project **wraps** the `simpler` runtime, treating it as the Level 0–2 execution engine. The `simpler` API for submitting orchestration functions to a chip becomes the "chip-level backend" that this project calls.
- **Level 3** (Host): multi-chip coordination within a single OS instance. This is the first new capability.
- **Levels 4–6** (Cluster-level-0/1/2): multi-host coordination across pods, supernodes, and racks. These are built as extensions of Level 3.

The architecture is:

```
┌─────────────────────────────────────────────────────────────────────┐
│  This project: Linqu Distributed Runtime (Levels 2–6)               │
│  Level 6: Cluster-level-2 (cross-rack)                              │
│  Level 5: Cluster-level-1 (supernode)                               │
│  Level 4: Cluster-level-0 (pod)                                     │
│  Level 3: Host (multi-chip coordination)                            │
│  Level 2: Chip (adaptation layer — wraps simpler)                   │
├─────────────────────────────────────────────────────────────────────┤
│  simpler runtime (Levels 0–2, DO NOT MODIFY)                        │
│  Level 2: Chip orchestration (task ring, buffer ring, scope mgmt)   │
│  Level 1: Chip die (omitted for now; simpler will add later)        │
│  Level 0: Core / Core-group (InCore, AIC, AIV, TPUSH/TPOP)         │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.3 Adaptation to `simpler`

The Linqu runtime **adapts to** the `simpler` runtime at the Level 2 boundary:

1. **Calling convention:** When the Linqu runtime needs to execute a chip-level (Level 2) function, it calls into `simpler`'s existing orchestration submission API. The Linqu runtime does **not** re-implement task rings, buffer rings, or scope-exit logic for Level 0–2 — it delegates to `simpler`.

2. **Identity mapping:** `simpler` uses its own task identity scheme internally (e.g. `task_id` within a chip). The Linqu runtime wraps this with the full `TaskKey(logical_system, L6..L0, scope_depth, task_id)` coordinate. The `simpler`-internal `task_id` becomes the `L0/L2` portion of the full key.

3. **Ring buffer layering:** For Levels 0–2, `simpler` manages its own ring buffers. For Levels 3–6, the Linqu runtime manages **its own** ring buffers (`task_ring[L][d]`, `buffer_ring[L][d]` for `L ≥ 3`). The two ring systems are independent; the Linqu runtime does not modify `simpler`'s rings.

4. **Scope nesting:** `simpler` manages scope depth within a chip. The Linqu runtime manages **higher-level scopes** (e.g. a host-level scope that contains multiple chip-level scopes). When the Linqu runtime exits a Level 3 scope, it performs Level 3 ring retirement; the chip-level scope exits inside `simpler` are handled by `simpler` independently.

5. **Future Level 1 support:** When `simpler` adds Level 1 (chip-die) support, it will be transparent to the Linqu runtime. The Linqu runtime only interacts with `simpler` at the Level 2 boundary.

### 3.4 Hardware Verification Environment Limitation

The full 7-layer Linqu system (Levels 0–6) is the **target architecture**, but the physical hardware for all seven layers is **not yet available**:

- **Levels 0–2**: Available today via `simpler` on current chips (Level 1 omitted on single-die models).
- **Levels 3–6**: While individual servers exist, the full multi-level cluster hardware with distinct Level 4/5/6 network fabrics is **not yet ready** for verification.

Therefore, the **first implementation** of the Linqu runtime targets a hardware environment with **Level 3 only** — a single host (one OS instance) managing one or more chips via `simpler`. This is the most constrained but immediately verifiable environment.

### 3.5 Forward-Compatible Software Design Requirement

Despite the Level 3–only hardware verification environment, the **software design must be compatible with a hardware environment that has all 7 layers** (Levels 0–6). This means:

1. **All data structures are parameterized by hierarchy level.** Ring buffers are `task_ring[L][d]` and `buffer_ring[L][d]` where `L` ranges over Levels 3–6 (managed by this project) plus the adaptation interface to `simpler` for Level 0–2. In the first implementation, only `L = 3` has non-zero capacity in the Linqu runtime; Levels 4–6 have zero-capacity rings but the indexing and data structures exist.

2. **All APIs accept `pl.Level` for any level.** The `pl.at(level=...)`, `@pl.function(level=...)`, and runtime dispatch interfaces accept any `pl.Level` value. For `L ∈ {0, 1, 2}`: dispatch is delegated to `simpler`. For `L = 3`: handled by the Linqu runtime. For `L ∈ {4, 5, 6}`: raise a clear `NotYetSupported` error at **runtime** (not compile time) in the first implementation.

3. **Task identity uses the full coordinate.** `TaskKey` always includes all level indices, even if most are zero in the first implementation.

4. **The RPC protocol header includes all level fields.** The `LinquHeader` struct carries `l6_idx`, `l5_idx`, `l4_idx`, `l3_idx` from day one. In the first implementation, `l4–l6` are always zero, but the wire format is stable.

5. **The PeerRegistry and gossip protocol are designed for multi-level.** In the first implementation, discovery is trivial (single host, no network gossip needed), but the data structures exist and are tested with mock multi-level topologies.

6. **The compiler generates hierarchy labels for all levels.** Even if the runtime ignores Level 4–6 labels today, the compiler emits them.

### 3.6 Compiler Contract: Hierarchy Labels

The compiler **must** generate a **hierarchy label** (a `pl.Level` tag) on every function it outlines or assigns to a level. This label is attached to the function in the IR and emitted metadata. Even if the runtime does not yet support a given level, the label is **present and reserved** so that:

- Future runtime versions can dispatch and schedule using these labels without changing the codegen contract.
- The same compiled binary can run on runtimes with different levels of multi-level support.
- Tools (profilers, debuggers) can display the hierarchy level of each function.

### 3.7 Extension Roadmap

| Phase | Levels Exercised | Hardware Environment | Capability |
|-------|-----------------|---------------------|------------|
| **Phase 0 (first impl.)** | 0–2 via `simpler`, 3 via Linqu runtime | Single host, one or more chips | Core/core-group execution via `simpler`; chip orchestration via `simpler`; **host-level multi-chip coordination** via Linqu runtime. Software design forward-compatible with all 7 layers. |
| **Phase 1** | 0–2 via `simpler` (with L1), 3 via Linqu | Host with multi-die chips | Level 1 added inside `simpler` when multi-die chips ship. Transparent to Linqu runtime. |
| **Phase 2** | 0–2 via `simpler`, 3–4 via Linqu | Multi-host pod/server | Cluster-level-0: intra-pod multi-host coordination. Activate gossip discovery, distributed SCOPE_EXIT, SPMD fan-out. |
| **Phase 3** | 0–2 via `simpler`, 3–5 via Linqu | Supernode | Cluster-level-1: hierarchical dispatch through Level-5 leaders. |
| **Phase 4** | 0–2 via `simpler`, 3–6 via Linqu | Full cluster | Cluster-level-2: cross-rack dispatch, aggregation nodes, contracted-bandwidth-aware scheduling. |

---

## 4. Task Identity and Coordinate System

### 4.1 The Task Key

Every task in the system is uniquely identified by a **hierarchical coordinate**:

```
TaskKey = (logical_system, L6, L5, L4, L3, L2, L1, L0, scope_depth, task_id)
```

In practice, many levels may be elided (e.g. if Level 1 is omitted, or if the program runs on a single host). The minimal form used within a single chip is:

```
TaskKey = (scope_depth, task_id)
```

This dual tag is the **canonical identifier** for all task lookup, producer/consumer references, ring buffer indexing, and retirement decisions. `task_id` alone is **never** used for correctness decisions; it is only unique within a scope layer.

### 4.2 User-Defined Coordinate Mapping

For cluster-level operation (Levels 3–6), each node computes its position in the hierarchy from its IP address using a **user-defined** mapping function:

```python
def get_my_coordinates(ip_address: str) -> dict:
    parts = [int(x) for x in ip_address.split('.')]
    return {
        "level_5": parts[1],   # Cluster-level-1 (supernode)
        "level_4": parts[2],   # Cluster-level-0 (pod/rack)
        "level_3": parts[3],   # Host
    }
```

The mapping function is **not hard-coded** into the runtime. Different cluster deployments may use different IP assignment schemes. The runtime loads this function at startup.

### 4.3 Logical and Physical System Names

To support multi-tenancy on shared physical infrastructure, each node carries two namespace strings:

- **`linqu_physical_system_name`**: identifies the hardware (e.g. `"dc-north-rack-05"`). Used for operational monitoring, failure diagnosis, and hardware-level addressing.
- **`linqu_logical_system_name`**: identifies the application or tenant (e.g. `"llm-training-v2"`). Used for **runtime isolation** — ring buffers, code caches, task keys, and membership tables are scoped to the logical system.

A single physical cluster can host multiple logical Linqu systems simultaneously. A node may participate in multiple logical systems.

---

## 5. Memory and Task Management: The Multi-Layer Ring Stack

### 5.1 Overview

Instead of a global heap or garbage collector, the runtime manages memory using a **multi-layer ring stack**. For every hierarchy level `L` and every scope depth `d` within that level, the runtime maintains:

- `task_ring[L][d]`: ring buffer of task metadata and execution status.
- `buffer_ring[L][d]`: ring buffer of output tensor / data buffer slots.
- `last_task_alive[L][d]`: the retirement head pointer for that ring.

### 5.2 Scope Depth and Ring Layers

Programs use `pl.scope()` to create nested scopes. Each scope depth `d` maps to a ring layer:

**On `scope.enter`:**
1. `current_scope_depth += 1`
2. Bind all new allocations to ring layer `d = current_scope_depth`.

**On task creation at depth `d`:**
1. Allocate task slot from `task_ring[L][d]`.
2. Allocate output buffers from `buffer_ring[L][d]`.
3. Assign task key as `(d, task_id_in_layer)`.
4. Initialize `fanout_count = 1` (the mandatory scope-exit token).
5. Register producer/consumer metadata.

**On `scope.exit`:**
1. Mark tasks/tensors in this scope frame as out-of-scope.
2. Apply scope-exit token: for each task in scope, if `task_freed == 0`, increment `ref_count += 1`.
3. Trigger layer-local retirement scan for depth `d`.
4. `current_scope_depth -= 1`.

### 5.3 Retirement Rules

A task and its output buffers in `buffer_ring[L][d]` are reclaimable when **both** conditions hold:

1. **Scope token applied**: either `scope.exit()` has been reached, or `pl.free(tensor)` was called for the tensor's producer task.
2. **Reference count satisfied**: `ref_count == fanout_count` (all consumers have completed).

Retirement is **layer-local**: progress in an inner scope (depth `d+1`) is **not** blocked by a stalled task in an outer scope (depth `d`). Each ring advances its own `last_task_alive` pointer independently.

### 5.4 The `pl.free(tensor)` Optimization

`pl.free(tensor)` allows the programmer to explicitly end a tensor's scope lifetime before the lexical scope exits. This is useful for tensors in broad, high-level scopes whose memory would otherwise be pinned until the scope naturally closes.

**Runtime API: `pto_rt.free(outbuf)`**

1. Lookup `(task_scope_level, task_id)` from `outbuf` via tensor map.
2. If `task_freed == true`, return (idempotent).
3. Set `task_freed = true`.
4. Increment `ref_count += 1`.
5. Return.

**Interaction with `scope.exit()`:**

At scope exit, when iterating tasks:
- If `task_freed == false`: apply scope token (`ref_count += 1`).
- If `task_freed == true`: skip (token was already applied by `pl.free`).

This guarantees **exactly-once** scope token application.

### 5.5 Producer/Consumer Metadata

All references use the dual-tag `TaskKey(scope_level, task_id)`, never `task_id` alone:

| Record | From (legacy) | To (multi-layer) |
|--------|---------------|-------------------|
| Tensor producer | `producer_task_id` | `producer_task_key = (scope_level, task_id)` |
| Tensor consumer list | `consumer_task_id[]` | `consumer_task_key[] = (scope_level, task_id)[]` |
| Tensor map | N/A | `outbuf -> (task_scope_level, task_id)` |
| Debug/profile events | `task_id` | `(scope_level, task_id)` |

### 5.6 Correctness Invariants

The following invariants must hold at all times:

1. **No early free**: never reclaim unless `ref_count == fanout_count`.
2. **Exactly-once scope token**: `pl.free()` + `scope.exit()` applies the token exactly once per task.
3. **Layer isolation**: `last_task_alive[d]` only controls layer `d`.
4. **Deterministic behavior**: same program order yields same reclamation decisions.
5. **Task identity uniqueness**: `TaskKey(scope_level, task_id)` is globally unique at runtime.
6. **Compatibility**: programs without `pl.free()` behave identically to current single-ring runtime.

---

## 6. The Unified Programming Grammar

### 6.1 Hierarchy Level Enum: `pl.Level`

A first-class enum aligns with the Linqu hierarchy and is used in both explicit and implicit function declarations:

| Symbol | Linqu Level | Readability Aliases |
|--------|-------------|---------------------|
| `pl.Level.AIV` | 0 (Core) | — |
| `pl.Level.AIC` | 0 (Core) | — |
| `pl.Level.CORE_GROUP` | 0 (Core-group) | — |
| `pl.Level.CHIP_DIE` | 1 | `pl.Level.L2CACHE` |
| `pl.Level.CHIP` | 2 | `pl.Level.PROCESSOR`, `pl.Level.UMA` |
| `pl.Level.HOST` | 3 | `pl.Level.NODE` |
| `pl.Level.CLUSTER_0` | 4 | `pl.Level.POD` |
| `pl.Level.CLUSTER_1` | 5 | `pl.Level.CLOS1` |
| `pl.Level.CLUSTER_2` | 6 | `pl.Level.CLOS2` |

### 6.2 Explicit Function Declaration

```python
@pl.function(level=pl.Level.AIV)
def my_aiv_kernel(...): ...

@pl.function(level=pl.Level.CHIP)
def my_orchestration(...): ...

@pl.function(level=pl.Level.CLUSTER_0)
def my_cluster_function(...): ...
```

The compiler tags the function with its hierarchy label. The runtime dispatches it to the appropriate level.

### 6.3 Implicit Function Declaration: `pl.at()`

A single context manager with an optional `optimization` parameter:

```python
with pl.at(level=pl.Level.CORE_GROUP):
    # One function at core-group level.
    ...

with pl.at(level=pl.Level.CORE_GROUP, optimization=pl.chunked_loop_optimizer):
    # Compiler splits loops, interchanges, outlines one or more functions.
    ...

with pl.at(level=pl.Level.CLUSTER_0):
    # One function dispatched to all nodes in cluster-level-0.
    ...
```

**Parameters:**

| Parameter | Meaning |
|-----------|---------|
| `level` | Target hierarchy level. Block is outlined as function(s) at this level. |
| `optimization` | Optional. Transformation before outlining: `pl.chunked_loop_optimizer`, `pl.fully_unroll_static_loop`, etc. Omit for a single outlined function. |

**Backward compatibility:**
- `with pl.incore` → `with pl.at(level=pl.Level.CORE_GROUP)`
- `with pl.auto_incore` → `with pl.at(level=pl.Level.CORE_GROUP, optimization=pl.chunked_loop_optimizer)`

---

## 7. Distributed Runtime Architecture (Levels 3–6)

### 7.1 Component Stack

The distributed runtime consists of four layers:

```
┌──────────────────────────────────────────────────────────┐
│  Orchestrator (Level 3+ code path)                        │
│  - Executes main control flow                             │
│  - Manages pl.scope, dispatches SPMD blocks               │
│  - Sends REG_CODE / REG_DATA / CALL_TASK / SCOPE_EXIT     │
├──────────────────────────────────────────────────────────┤
│  Node Daemon (Level 3, runs on every host)                │
│  - Listens for tasks                                      │
│  - Manages local buffer_ring[L][d] and task_ring[L][d]    │
│  - Executes code blobs locally on chips                   │
│  - Reports heartbeats via gossip                          │
├──────────────────────────────────────────────────────────┤
│  Rendezvous Ledger (Distributed, gossip-based)            │
│  - Membership table: (logical_system, coord) → IP/status  │
│  - No central server; peer-to-peer convergence            │
├──────────────────────────────────────────────────────────┤
│  Transport (Binary RPC over TCP/UDP)                      │
│  - Zero-copy binary protocol (FlatBuffers or equivalent)  │
│  - Handle-based: register once, invoke many               │
└──────────────────────────────────────────────────────────┘
```

### 7.2 The Orchestrator

The **Orchestrator** runs the top-level PyPTO program. It is the entity that:

1. Executes `pl.scope()` enter/exit at the cluster level, managing `current_scope_depth` for distributed scopes.
2. Resolves `pl.at(level=pl.Level.CLUSTER_0)` by querying the Rendezvous Ledger for live nodes at the target level and dispatching the outlined function to all of them.
3. Sends `REG_CODE` and `REG_DATA` messages to register code blobs and data buffers on remote nodes.
4. Sends `CALL_TASK` messages containing only the blob hash, data handles, and SPMD index — not the full binary.
5. Sends `SCOPE_EXIT` signals when a scope closes, triggering distributed ring layer retirement.

### 7.3 The Node Daemon

Every host runs a lightweight **Node Daemon** that:

1. On startup, calls the user-defined `get_my_coordinates(ip)` to compute its position in the hierarchy.
2. Joins the cluster by probing deterministic seed IPs (see §8) and participating in gossip.
3. Maintains local `task_ring[L][d]` and `buffer_ring[L][d]` for the hierarchy levels it manages.
4. Stores received code blobs in a `CodeCache[blob_hash] → FunctionPointer` map.
5. Stores received data in a `DataCache[data_handle] → BufferPointer` map.
6. On `CALL_TASK` receipt: looks up the blob hash in CodeCache, looks up data handles in DataCache, executes the function with the given `spmd_idx`.
7. On `SCOPE_EXIT` receipt at depth `d`: retires all tasks and buffers at that scope depth, advancing `last_task_alive[L][d]` and freeing ring slots.

### 7.4 Communication Model by Level

| Level boundary | Transport mechanism | Serialization |
|----------------|---------------------|---------------|
| Level 0 (intra-core-group) | TPUSH/TPOP hardware DMA | None (on-chip) |
| Level 0–2 (intra-chip) | Shared memory / on-chip interconnect | None (pointer passing via buffer_ring) |
| Level 2–3 (chip → host) | PCIe / NVLink DMA | Minimal header |
| Level 3–4 (intra-pod) | RDMA or high-speed TCP | Binary RPC (FlatBuffers) |
| Level 4–5 (intra-supernode) | TCP / RDMA fabric | Binary RPC |
| Level 5–6 (cross-rack) | TCP with possible compression | Binary RPC |

---

## 8. Cluster Discovery and Topology

### 8.1 Rule-Based IP-to-Coordinate Mapping

Linqu network hierarchy boundaries are **not** defined by subnet boundaries. Instead, IP addresses are **pre-planned and assigned** according to the hierarchy. The runtime uses a **user-defined rule formula** to compute the index at each level from an IP address.

Example schema for IPv4 `A.B.C.D`:

| Level | Name | IP segment | Formula |
|-------|------|------------|---------|
| 6 | Cluster-level-2 | (implicit or from A) | `idx_L6 = f(A)` |
| 5 | Cluster-level-1 | Octet B | `idx_L5 = B` |
| 4 | Cluster-level-0 | Octet C | `idx_L4 = C` |
| 3 | Host | Octet D | `idx_L3 = D` |

The mapping function is provided by the user at deployment time:

```python
def get_my_coordinates(ip_address: str) -> dict:
    parts = [int(x) for x in ip_address.split('.')]
    return {
        "level_6": 0,          # single L2 cluster in this deployment
        "level_5": parts[1],
        "level_4": parts[2],
        "level_3": parts[3],
    }
```

Every node can compute its own coordinates — and the coordinates of any other node — using this function. No central topology server is required.

### 8.2 Deterministic Peer Mapping

Because the IP-to-coordinate function is known to all nodes, peer addressing is **mathematical**:

- To find the "next host" in the same pod: `neighbor_ip = f"10.{my_L5}.{my_L4}.{my_L3 + 1}"`
- To find all hosts in pod 5 of supernode 1: enumerate `10.1.5.*`

The runtime does not need DNS or a name service for address resolution.

### 8.3 Zero-Configuration Rendezvous

Even though coordinates are deterministic, the runtime needs to know which nodes are **currently alive**. This is solved with a **decentralized gossip-based membership protocol** (e.g. SWIM):

**Phase A: Bootstrap ("First Citizen")**

By convention, the first valid IP in each Level-4 subnet is the **implicit rendezvous seed** for that scope:
- Node `10.1.5.20` knows its Level-4 seed is `10.1.5.1`.
- On boot, the node probes `10.1.5.1`. If `.1` is down, it tries `.2`, `.3`, etc.
- Once any active peer responds, they swap membership lists.

**Phase B: Gossip Protocol**

Once bootstrapped, nodes use periodic gossip to maintain the membership table:
- Every ~1 second, a node picks 2–3 random peers and exchanges its "Node List" for the same `linqu_logical_system_name`.
- Information about new or failed nodes spreads exponentially.
- Even in a 1,000-node cluster, full convergence happens in seconds.

**Phase C: Failure Detection**

- If Node A hasn't received a gossip from Node B in `T_suspect` seconds, it marks B as "Suspect."
- After `T_dead` more seconds without response, it marks B as "Dead" and propagates this.
- The Orchestrator's `pl.at(Level.CLUSTER_0)` call automatically skips dead nodes.

### 8.4 The Rendezvous Ledger Structure

Each node maintains a local `PeerRegistry` — a thread-safe map:

```
Key:   (linqu_logical_system_name, L6_idx, L5_idx, L4_idx, L3_idx)
Value: {
    ip: str,
    physical_system_name: str,
    status: "live" | "suspect" | "dead",
    last_heartbeat: timestamp,
    resources: ResourceVector,  # e.g. {"chips": 8, "mem_gb": 512}
}
```

The Orchestrator queries this map when resolving `pl.at(level=...)`:
1. Filter entries by `linqu_logical_system_name`.
2. Filter by target level and index range.
3. Exclude "dead" entries.
4. Return the list of live IPs for dispatch.

### 8.5 Hierarchical Aggregation for Scale

In very large clusters (Level 6), flat gossip becomes expensive. The runtime uses **aggregation nodes**:

- **Level 4 Leader**: one designated node in each Cluster-level-0 acts as a sub-registrar. It maintains the full membership list for its pod and reports a summary to Level 5.
- **Level 5 Leader**: aggregates Level-4 summaries for its supernode, reports up to Level 6.

Queries are level-scoped: an orchestrator targeting Level 4 only queries the Level-4 leader, not every individual host.

---

## 9. The Efficient RPC Protocol

### 9.1 Design Principles

- **Zero-copy**: use FlatBuffers or Cap'n Proto so fields can be read directly from the network buffer without deserialization into new objects.
- **Handle-based**: code and data are registered once and referenced by hash/handle thereafter.
- **Hierarchy-aware**: every message carries the sender's Linqu coordinate so the receiver knows the message's position in the hierarchy.
- **Tiny metadata path**: the common case (`CALL_TASK`) is a small header + handles, not a full binary blob.

### 9.2 Common Message Header

Every RPC message begins with a fixed 24-byte header:

```c
struct LinquHeader {
    uint32_t magic;            // Protocol version identifier
    uint64_t logical_sys_id;   // Hash of linqu_logical_system_name
    uint8_t  l6_idx;           // Cluster-level-2 index
    uint8_t  l5_idx;           // Cluster-level-1 index
    uint8_t  l4_idx;           // Cluster-level-0 index
    uint8_t  l3_idx;           // Host index
    uint16_t msg_type;         // Message type enum
    uint32_t payload_size;     // Payload bytes following header
};
```

### 9.3 Message Types

| msg_type | Name | Direction | Payload | Frequency |
|----------|------|-----------|---------|-----------|
| 0x01 | `HEARTBEAT` | Node → Peers | `(phys_name, logic_name, resources, timestamp)` | Periodic (~1s) |
| 0x02 | `REG_CODE` | Orchestrator → Node | `(blob_hash, code_binary)` | Once per code version |
| 0x03 | `REG_DATA` | Orchestrator → Node | `(data_handle, buffer_bytes, level, scope_depth)` | Once per dataset |
| 0x04 | `CALL_TASK` | Orchestrator → Node | `(blob_hash, data_handle[], spmd_idx, scope_depth, task_id)` | Every invocation |
| 0x05 | `SCOPE_EXIT` | Orchestrator → Nodes | `(scope_depth)` | Every scope close |
| 0x06 | `RETRY_WITH_CODE` | Node → Orchestrator | `(blob_hash)` | On cache miss |
| 0x07 | `TASK_COMPLETE` | Node → Orchestrator | `(task_key, status)` | Every task completion |

### 9.4 The "Register Once, Invoke Many" Lifecycle

```
Step 1  SETUP:      Orchestrator calls rt.deploy_environment(cluster_id)
                    → Node daemons are already running; gossip membership converges.

Step 2  CODE PREP:  Orchestrator calls rt.register(my_kernel_blob)
                    → REG_CODE sent with full binary + blob_hash.
                    → Node stores in CodeCache[blob_hash] → FunctionPointer.

Step 3  DATA PREP:  h = rt.put(large_tensor, level=Level.HOST, scope_depth=d)
                    → REG_DATA sent with buffer bytes.
                    → Node stores in buffer_ring[L][d] at the given scope.
                    → Returns data_handle h.

Step 4  RUN:        rt.call(my_kernel_blob, h, spmd_idx=i)
                    → CALL_TASK sent: only (blob_hash, [h], i, d, task_id).
                    → Tiny message: ~32 bytes of metadata.
                    → Node looks up CodeCache and DataCache, executes locally.

Step 5  CLEANUP:    Orchestrator exits pl.scope() at depth d.
                    → SCOPE_EXIT(d) broadcast to all nodes.
                    → Nodes clear buffer_ring[L][d] and task_ring[L][d].
```

### 9.5 Lazy Code Loading

When a node receives `CALL_TASK` but doesn't have the `blob_hash` in its CodeCache (e.g. it joined the cluster after registration), it responds with `RETRY_WITH_CODE`. The Orchestrator then sends `REG_CODE` for that hash, and re-sends the `CALL_TASK`. This lazy loading pattern keeps initial startup fast and handles late-joining nodes gracefully.

### 9.6 Code Outlining Integration

The compiler provides each outlined function with:
- A **static content hash** of the code block.
- A **capture list** (the variables the code reads from the enclosing scope).

The Orchestrator checks `if remote_node.has_hash(block_hash): skip_send()`. This further reduces transfers for code that hasn't changed between invocations.

---

## 10. SPMD Dispatch: `pl.at()` at Cluster Level

### 10.1 Execution Semantics

When the Orchestrator encounters `pl.at(level=pl.Level.CLUSTER_0)`, it:

1. **Resolves** the target scope: queries the PeerRegistry for all live nodes in the current Cluster-level-0 group (filtered by `linqu_logical_system_name`).
2. **Registers** code and data if not already cached on target nodes.
3. **Broadcasts** `CALL_TASK` to each target node, assigning a unique `spmd_idx` to each.
4. **Waits** for all `TASK_COMPLETE` responses (or handles failures).

### 10.2 Optimization via `pl.at(level=..., optimization=...)`

At cluster level, the `optimization` parameter can control how the work is distributed:

- `optimization=pl.chunked_loop_optimizer`: the runtime (or compiler) splits a large iteration space across nodes in chunks, similar to how chunked-loop splitting works at core level.
- No `optimization`: the block is dispatched as a single function per node.

### 10.3 Hierarchical Dispatch

For multi-level dispatch (e.g. `pl.at(level=pl.Level.CLUSTER_1)` that spans a supernode):

1. The Orchestrator sends to **Level 5 Leaders** (one per Cluster-level-0).
2. Each Level 5 Leader re-dispatches to its Level 4 nodes.
3. Each Level 4 node executes the function locally.

This hierarchical fan-out reduces the Orchestrator's bandwidth requirements for large clusters.

---

## 11. Ring Buffer Hierarchy at Cluster Level

### 11.1 Per-Level, Per-Depth Rings

At every hierarchy level, the runtime maintains ring buffers indexed by scope depth. For cluster operation:

```
task_ring[HOST][0], task_ring[HOST][1], ...      (Host-local rings)
task_ring[CLUSTER_0][0], task_ring[CLUSTER_0][1], ...  (Pod-level rings)
task_ring[CLUSTER_1][0], task_ring[CLUSTER_1][1], ...  (Supernode-level rings)
```

Data allocated within a `pl.at(level=pl.Level.CLUSTER_0)` scope at depth `d` is placed in `buffer_ring[CLUSTER_0][d]`.

### 11.2 Distributed Scope Exit

When the Orchestrator exits a `pl.scope()` at cluster level:

1. It broadcasts `SCOPE_EXIT(d)` to all nodes in the scope.
2. Each node locally applies scope-exit semantics: iterates tasks in `task_ring[L][d]`, applies scope tokens (respecting `task_freed` flags), triggers retirement scan.
3. Each node clears its local `buffer_ring[L][d]` slots once tasks are retired.
4. The Orchestrator decrements its own `current_scope_depth`.

This is the **distributed analog** of single-chip scope exit: each node handles its local portion independently.

### 11.3 Cross-Level Data Transfer

When a tensor produced at Level 3 (Host) is consumed by a function at Level 0 (Core), the runtime performs a multi-level transfer:

1. Host ring: tensor lives in `buffer_ring[HOST][d]`.
2. Chip: runtime DMA-transfers the tensor from host memory to chip HBM, allocating in `buffer_ring[CHIP][d']`.
3. Core: the InCore function reads from chip memory.

The hierarchy labels on functions tell the runtime which transfers are needed.

---

## 12. Profiling and Ring Capacity Tuning

### 12.1 Per-Layer Metrics

The runtime must collect the following metrics for each ring layer (hierarchy level `L`, scope depth `d`):

| Metric | Description |
|--------|-------------|
| `task_ring_capacity[L][d]` | Total slots allocated |
| `task_ring_peak_used[L][d]` | Maximum slots simultaneously occupied |
| `task_ring_peak_occupancy_pct[L][d]` | `peak_used / capacity × 100` |
| `task_ring_block_count[L][d]` | Number of times allocation blocked (ring full) |
| `task_ring_block_time_us[L][d]` | Total microseconds spent blocked |
| `buffer_ring_capacity_bytes[L][d]` | Total bytes allocated |
| `buffer_ring_peak_used_bytes[L][d]` | Maximum bytes simultaneously occupied |
| `buffer_ring_peak_occupancy_pct[L][d]` | `peak_used / capacity × 100` |
| `buffer_ring_block_count[L][d]` | Number of times allocation blocked |
| `buffer_ring_block_time_us[L][d]` | Total microseconds spent blocked |
| `retire_scan_calls[L][d]` | Number of retirement scans triggered |
| `retire_scan_reclaimed_tasks[L][d]` | Tasks reclaimed by retirement |
| `retire_scan_reclaimed_bytes[L][d]` | Bytes reclaimed |

### 12.2 Global Rollout Metrics

- Total blocked time across all layers.
- Maximum concurrent active scope depth.
- Program-level peak memory (sum over all layers).
- Per-operation / per-scope attribution of blocking hotspots.

### 12.3 Profiling Interface

Runtime flags:
- `runtime.profile_ring = true`
- `runtime.profile_ring_detail_level = {basic | verbose}`

Output formats:
- Machine-readable JSON (for CI and auto-tuning).
- Human-readable summary table.

Example JSON output:

```json
{
  "run_id": "xxx",
  "layers": [
    {
      "level": "HOST",
      "depth": 0,
      "task_ring_capacity": 4096,
      "task_ring_peak_used": 3012,
      "task_ring_block_count": 0,
      "buffer_ring_capacity_bytes": 1073741824,
      "buffer_ring_peak_used_bytes": 812646400,
      "buffer_ring_block_count": 3,
      "buffer_ring_block_time_us": 14320
    }
  ],
  "global": {
    "total_block_time_us": 20111,
    "max_active_scope_depth": 4,
    "peak_total_buffer_bytes": 1879048192
  }
}
```

### 12.4 Capacity Tuning Workflow

1. Run representative workloads with profiling enabled.
2. Collect p95/p99 of peak usage per layer.
3. Set deployment ring capacity with safety margin:
   - `task_capacity[L][d] = ceil(p99_task_peak[L][d] × margin)` (margin: 1.1–1.3).
   - `buffer_capacity[L][d] = ceil(p99_buffer_peak[L][d] × margin)`.
4. Re-validate: target `block_count == 0` for latency-sensitive paths.
5. Repeat for different model shapes / batch sizes; store profiles as deployment presets.

If `block_count` remains non-zero after capacity increase, inspect whether:
- Logical lifetime is too long (missing `pl.free` opportunities).
- Retire ordering is suboptimal.
- Do not only increase ring sizes; fix the root cause.

---

## 13. CI Gating and Regression Policy

### 13.1 Gating Rules

| Severity | Condition |
|----------|-----------|
| **Hard fail** | `buffer_ring_block_count_total == 0` for latency-critical pipelines |
| **Hard fail** | `task_ring_block_count_total == 0` |
| **Soft fail / warning** | Any layer `peak_occupancy_pct > 95%` |
| **Soft fail / warning** | Global `total_block_time_us` regresses by > X% from baseline |
| **Trend guard** | p99 `peak_total_buffer_bytes` regression > Y% over 7-day window |

### 13.2 Baseline Strategy

- Store a versioned profiling baseline per (model_shape, batch_size, cluster_config).
- Compare PR results against nearest baseline preset.
- Require explicit approval tag for intentional capacity increases.

### 13.3 Regression Triage

1. If block count increases: inspect missing/late `pl.free` opportunities; inspect ring layer mapping for changed scope behavior.
2. If occupancy increases without blocks: evaluate whether shape mix changed; adjust preset capacities only after confirming no logic regressions.
3. If only one layer regresses: tune that layer first; avoid global over-provisioning.

---

## 14. Implementation Plan

The plan is structured around two realities:

1. **`simpler` owns Levels 0–2 and must not be modified.** The Linqu runtime adapts to `simpler`'s existing interfaces for chip-level and core-level execution.
2. **The first implementation** (Phase 0) runs on a **Level 3 (single-host) environment only**, but the software is designed so that **all data structures, APIs, protocols, and identity formats are forward-compatible with the full 7-layer system**. Subsequent phases activate additional levels as hardware becomes available.

### Phase 0: First Implementation — Level 3 (Single-Host), Building on `simpler`

**Hardware target:** One host (one OS instance), one or more chips. No multi-host cluster. No Level 4–6 network fabric.

**Guiding rule:** Do not modify `simpler`. Use `simpler`'s API to execute Level 0–2 functions. Build Linqu runtime for Levels 2–6 around it.

| Task | Description |
|------|-------------|
| 0.1 | **Define the `simpler` adaptation interface.** Document the `simpler` APIs used to submit orchestration functions (Level 2) and InCore functions (Level 0) to a chip. This is a read-only contract: the Linqu runtime calls into `simpler` but does not modify it. The adaptation interface must also document how `simpler` reports task completion so the Linqu runtime can track host-level dependencies. |
| 0.2 | **Implement the `ChipBackend` adapter.** A thin wrapper that translates Linqu runtime dispatch calls (`level=pl.Level.CHIP`) into `simpler` API calls. This is the only component that directly touches `simpler`. All other Linqu runtime code interacts with this adapter, never with `simpler` directly. If `simpler` later adds Level 1 support, only this adapter needs to be aware. |
| 0.3 | Implement the `RingLayer` class managing `task_ring[L][d]` and `buffer_ring[L][d]` for **Levels 3–6**. Parameterized by hierarchy level `L` (3–6), but only allocate non-zero capacity for `L = 3` in this phase. Levels 4–6 have zero-capacity rings (data structures exist, just empty). For Levels 0–2, `simpler` manages its own rings — the Linqu runtime does **not** duplicate them. |
| 0.4 | Implement the `ScopeManager` for **host-level and above scopes** (Level 3+). Tracks `current_scope_depth` for Level 3+ scopes, handles `scope.enter` / `scope.exit` / `pl.free` signals at these levels. Chip-level scope depth (within a `simpler` orchestration function) is managed by `simpler` independently. |
| 0.5 | Implement the `TaskKey` identity model with **full coordinate** `(logical_system, L6..L0, scope_depth, task_id)`. The `L0/L2` portion wraps the `simpler`-internal task identity. In this phase, L4–L6 are always zero, but the struct and all comparisons use the full key. |
| 0.6 | Implement the retirement scan for Level 3+ rings: layer-local, respects `task_freed` flag, advances `last_task_alive[L][d]`. Level 0–2 retirement is handled by `simpler`. |
| 0.7 | Implement the `LinquCoordinate` struct and `get_my_coordinates(ip)` interface. In this phase, the host populates L3 (and queries `simpler` for chip/core topology to fill L0/L2); L4–L6 default to zero. |
| 0.8 | Implement `linqu_physical_system_name` and `linqu_logical_system_name` node identity. Even on a single host, the naming is set so that multi-host deployment requires no identity-format change. |
| 0.9 | Define and implement the `LinquHeader` binary message format (24-byte header) with **all level fields** (`l6_idx`, `l5_idx`, `l4_idx`, `l3_idx`). In this phase, l4–l6 are zero, but the wire format is stable. |
| 0.10 | Implement the `PeerRegistry` data structure, keyed by `(logical_system, full_coordinate)`. In this phase, it contains only the local host; tested with **mock multi-level topologies** for forward compatibility. |
| 0.11 | Implement the `pl.at(level=...)` dispatch path. For `L ∈ {0, 1, 2}`: delegate to `simpler` via the `ChipBackend` adapter. For `L = 3`: handle in the Linqu runtime (host-level multi-chip coordination). For `L ∈ {4, 5, 6}`: raise `NotYetSupported` at **runtime**. |
| 0.12 | Implement host-level multi-chip coordination: the Linqu runtime can submit work to **multiple chips** on the same host by making multiple calls to the `ChipBackend` adapter, managing host-level scope and inter-chip data dependencies. |
| 0.13 | Implement per-layer ring profiling metrics for Level 3. The metrics framework is parameterized for Levels 3–6. Level 0–2 profiling is `simpler`'s responsibility. |
| 0.14 | Write **forward-compatibility tests**: unit tests that create `TaskKey` and `LinquCoordinate` objects with non-zero L4–L6 fields, verify serialization/deserialization, ring indexing, and PeerRegistry lookup with full 7-level coordinates. These tests run on the single-host environment using mock data. |
| 0.15 | Write **`simpler` integration tests**: end-to-end tests that exercise Level 3 → Level 2 → Level 0 dispatch: Linqu runtime creates a host-level scope, submits chip-level work via `ChipBackend`, `simpler` executes it, Linqu runtime tracks completion and retires host-level rings. |

### Phase 1: Multi-Die Chips (Level 1 Added Inside `simpler`)

**Hardware target:** Host with multi-die chips.

**Guiding rule:** Level 1 is implemented **inside `simpler`**, not in the Linqu runtime. The Linqu runtime only needs minor adapter updates.

| Task | Description |
|------|-------------|
| 1.1 | Update the `ChipBackend` adapter if `simpler`'s API surface changes for multi-die dispatch (e.g. new die-index parameter). The Linqu runtime's `get_my_coordinates` populates L1 from `simpler`'s topology query. |
| 1.2 | Verify that `TaskKey` and `LinquCoordinate` correctly carry the L1 index through all Linqu runtime code paths (should already work via forward-compatible design). |

### Phase 2: Multi-Host Pod (Activate Level 4)

**Hardware target:** Multiple hosts in a pod/server, connected by high-bandwidth network.

| Task | Description |
|------|-------------|
| 2.1 | Activate Level 4 ring buffers (`task_ring[4][d]`, `buffer_ring[4][d]`). |
| 2.2 | Activate gossip-based discovery: UDP Heartbeat, deterministic seeding (probe Level-4 seed IPs), SWIM membership. |
| 2.3 | Implement `REG_CODE` / `REG_DATA` / `CALL_TASK` / `SCOPE_EXIT` message handlers on the Node Daemon (network RPC using the `LinquHeader` format defined in Phase 0). |
| 2.4 | Build `CodeCache[blob_hash]` and `DataCache[data_handle]` on remote nodes. |
| 2.5 | Implement `RETRY_WITH_CODE` lazy loading fallback. |
| 2.6 | Implement Orchestrator-side `pl.at(level=pl.Level.CLUSTER_0)` resolution: PeerRegistry query → IP list → broadcast. Each remote node runs its own `simpler` for Level 0–2 execution. |
| 2.7 | Implement distributed `SCOPE_EXIT` broadcast and per-node ring retirement (Level 3+ rings on each node). |
| 2.8 | Implement SPMD fan-out: assign `spmd_idx` per target node; broadcast `CALL_TASK`. |
| 2.9 | Implement `TASK_COMPLETE` collection and synchronization barriers. |
| 2.10 | Implement failure detection: suspect → dead transition; gossip propagation. |

### Phase 3: Supernode (Activate Level 5)

**Hardware target:** Supernode with high-bandwidth domain across pods.

| Task | Description |
|------|-------------|
| 3.1 | Activate Level 5 ring buffers. |
| 3.2 | Implement hierarchical dispatch: Orchestrator → Level-5 Leaders → Level-4 Nodes. |
| 3.3 | Implement Level-4 aggregation nodes (sub-registrars reporting up to Level 5). |
| 3.4 | Implement `optimization` parameter handling at cluster level (e.g. chunked distribution of iteration spaces across pods). |

### Phase 4: Full Cluster (Activate Level 6)

**Hardware target:** Cross-rack cluster with contracted bandwidth.

| Task | Description |
|------|-------------|
| 4.1 | Activate Level 6 ring buffers. |
| 4.2 | Implement Level-5 aggregation nodes reporting up to Level 6. |
| 4.3 | Implement bandwidth-aware scheduling: prefer local dispatch at lower levels; use compression or staged transfers at Level 6. |
| 4.4 | Implement RDMA transport optimization for Level 3–4 (intra-pod, zero-copy network). |

### Phase 5: Profiling, CI, and Production Hardening

| Task | Description |
|------|-------------|
| 5.1 | Extend ring profiling to all activated levels (3–6); full JSON and human-readable output. Level 0–2 profiling is `simpler`'s responsibility. |
| 5.2 | Define CI gating rules and baseline comparison framework (per-level, Level 3+). |
| 5.3 | Build capacity auto-tuning recommendations from profiling data. |
| 5.4 | Stress test: deep nesting, mixed `pl.free` + scope exits, node failure mid-scope, late-joining nodes, cross-level data transfer through `ChipBackend` → `simpler`. |
| 5.5 | Compatibility adapters for `simpler` profiling data (if needed for unified cross-level views). |

---

## 15. References

- **`simpler` runtime (Level 0–2, do not modify):** `pypto_workspace/simpler/` — existing chip-level and core-level runtime.
- **Machine hierarchy and function grammar:** `machine_hierarchy_and_function_hierarchy.md`
- **Multi-level runtime ring stack and `pl.free`:** `multi_level_runtime_ring_and_pypto_free_api.md`
- **ExpandMixedKernel and InCoreFunctionGroup:** `HL_new_feature_Expand_Mixed_Kernel_and_call_spmd.md`
- **TPUSH/TPOP (intra-cluster core communication):** `HL_ptoisa_newfeature20260306_TPUSH_TPOP.md`
- **Tensor valid_shape and alignment:** `tensor_valid_shape.md`
- **Gemini conversation (distributed runtime design exploration):** `Gemini_conversation.md`
