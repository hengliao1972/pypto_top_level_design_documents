# 7. Cross-Cutting Concerns

This section addresses concerns that span all modules and views: security, observability, and performance.

---

## 7.1 Security

### 7.1.1 Trust Boundaries

| Boundary | Trust Level | Threats |
|----------|------------|---------|
| Python ↔ C API (`pto_runtime_api.h`) | User code ↔ Runtime | Invalid arguments, resource exhaustion, injection via function names |
| Host ↔ Device (DMA) | Trusted runtime ↔ Trusted hardware | DMA corruption, address space violations |
| Node ↔ Node (network) | Trusted peers within cluster | Message tampering, replay attacks, unauthorized node joining |
| Runtime ↔ External Data Services | Runtime ↔ External systems | Data corruption, unauthorized access |

### 7.1.2 Threat Model (Rule S1)

Per-boundary threat enumeration using STRIDE categories:

| Boundary | Threat | STRIDE | Risk | Mitigation |
|----------|--------|--------|------|------------|
| Python ↔ C API | Malformed tensor shapes cause buffer overflow | Tampering | Medium | Input validation: bounds-check all shape dimensions, reject negative sizes, validate alignment (§7.1.3 bullet 1) |
| Python ↔ C API | Exhaustion via rapid task submission | Denial of Service | Medium | Task slot pool limits per Logical System; configurable pool sizes in `LevelParams` |
| Python ↔ C API | Function name injection (path traversal in binary loading) | Tampering | Low | Sanitize function names; restrict binary loading to registered paths |
| Host ↔ Device (DMA) | DMA to invalid device address | Tampering | Low | HAL validates device address ranges before DMA setup; IOMMU protection on supported platforms |
| Host ↔ Device (DMA) | DMA descriptor corruption | Tampering | Low | Shared trust domain; mitigated by memory protection in CANN runtime |
| Node ↔ Node (network) | Unauthorized node joins cluster | Spoofing | High | Node identity verification during peer discovery; reject unknown node IDs |
| Node ↔ Node (network) | Message replay causing duplicate task execution | Replay | Medium | Task generation counters detect stale handles; idempotent task slot allocation |
| Node ↔ Node (network) | Message tampering alters task descriptors | Tampering | High | [ASSUMPTION] Encrypted backends (TLS/IPsec) when deployed in untrusted networks |
| Node ↔ Node (network) | Resource exhaustion via flood of REMOTE_SUBMIT | Denial of Service | Medium | Per-node rate limiting on incoming remote submissions; configurable in `DistributedConfig` |
| Runtime ↔ External Data | Data corruption in external storage | Tampering | Medium | Checksum verification on data read; application-level integrity checks |

### 7.1.3 Mitigations

- **Input validation at C API boundary (Rule S3):** All arguments validated before entering runtime internals (tensor shapes, function IDs, configuration parameters).
- **Least privilege (Rule S2):** Each component accesses only its own level's Memory Scope. Cross-level access is mediated through `IMemoryOps`.
- **Node authentication (distributed):** Peer discovery includes node identity verification. `REMOTE_SUBMIT` messages carry authenticated node IDs.
- **Memory isolation:** Logical System namespaces prevent cross-tenant data access.
- **Secure defaults (Rule S4):** Default configuration disables external network access; distributed mode requires explicit opt-in via `DistributedConfig`.

### 7.1.3 Known Limitations

- Intra-node trust: The runtime assumes all components within a single node (Host, Device, Chip, Core) are mutually trusted. There is no encryption or authentication on intra-node channels (shared memory, register bank, DMA).
- [ASSUMPTION] Network encryption (TLS/mTLS) for inter-node communication is deferred to the network backend implementation, not enforced by the runtime framework.

> [UPDATED: A8-P4: `dump_state()` diagnostic endpoint is tenant-scoped by default.]
> `Runtime::dump_state(DumpOptions) → StateDump` (see `modules/runtime.md §2.1`) defaults its scope to the caller's `logical_system_id` — the returned JSON includes only tasks, workers, and state owned by the caller's Logical System. A cross-tenant (unscoped) dump, including an aggregate view across all Logical Systems, requires a `diagnostic_admin_token` carried in `DumpOptions`; requests without the token MUST be rejected with `ErrorCode::PermissionDenied`. Token issuance and revocation follow the security audit trail (A6-P6) and the capability-scoped sink policy (A6-P10).

---

## 7.2 Observability

### 7.2.1 Profiling Levels

| Level | Scope | Overhead | Details |
|-------|-------|----------|---------|
| 0 | Off | Zero | No profiling code executed |
| 1 | Task-level timing | Minimal | Per-task start/end timestamps |
| 2 | Phase-level | Low | Scheduler, dispatch, compute phase timing |
| 3 | Full trace | Moderate | Every register write, memory op, state transition |
| 4 | Distributed trace | Moderate | Cross-node message latencies, data movement bandwidth |

Profiling level is runtime-switchable (Rule O4).

> [UPDATED: A4-P1: Canonical casing for HAL enums used across Observability and Performance sections.]
> All HAL enums are referenced in this document using their canonical UPPER_SNAKE_CASE spellings: `Variant::{ONBOARD, SIM}` and `SimulationMode::{PERFORMANCE, FUNCTIONAL, REPLAY}`. Any occurrence of `Onboard`, `Sim`, `Performance`, `Functional`, or `Replay` in older snippets SHOULD be read as the corresponding canonical casing. Authoritative definition in `modules/hal.md §2.7` + ADR-011.

### 7.2.2 Trace Event Model

Events are compatible with Chrome Trace format / Perfetto:

```
Event {
    timestamp:  uint64_t    // nanosecond timestamp
    duration:   uint64_t    // for duration events
    category:   string      // "scheduler", "dispatch", "compute", "data_movement", "network"
    name:       string      // "submit_task", "aicore_kernel", "rdma_write"
    tid:        uint32_t    // thread ID
    pid:        uint32_t    // process ID
    node_id:    uint32_t    // node ID (distributed)
    layer_id:   LayerId     // which layer generated this event
    args:       map         // additional key-value context
}
```

Event types:
- Phase events (B/E): begin/end pairs.
- Duration events (X): self-contained with duration.
- Instant events (i): point-in-time markers.
- Flow events (s/t/f): cross-node/cross-thread causality arrows.

### 7.2.3 Collection Architecture

```
AICore Collector ─┐
                  │ device ring buffers
AICPU Collector ──┤
                  │ drain via DMA/shared memory
                  ▼
Host Collector ────► Local Trace Buffer ────► Export
                                                │
                                   (distributed mode)
                                                ▼
                                   Coordinator Node merges
                                   all node traces with
                                   global timestamp alignment
```

- Per-target collectors (Host, AICPU, AICore) write to local ring buffers.
- Host collector drains device buffers via DMA/shared memory.
- Distributed collector: each node exports local trace; coordinator merges with global timestamp alignment.

### 7.2.4 Logging Framework

Format: `[timestamp][level][node_id][layer_id][module][file:line] message`

Levels: `FATAL`, `ERROR`, `WARN`, `INFO`, `DEBUG`, `TRACE`.

- Per-module enable/disable.
- Device-side logging with limited buffer (ring buffer on AICPU, minimal on AICore).
- Structured key-value format for machine parsing.

### 7.2.5 Zero-Overhead Strategy (Rule NFR-1)

- **Compile-time:** `#if PROFILING_ENABLED` guards profiling code paths.
- **Runtime:** Check profiling level before recording; branchless recording with pre-allocated slots.
- **No allocation in hot path:** All profiling buffers are pre-allocated at initialization.

### 7.2.6 Export Formats

- Chrome Trace JSON (single-node and multi-node merged).
- CSV summary for post-processing.
- Binary dump for offline analysis.
- Python API to access profiling data programmatically.
- **Replay-compatible trace:** Binary dumps captured at Level 2 or higher (§7.2.1) are a valid input for the `REPLAY` Simulation Mode ([§2.8.1](02-logical-view/10-platform.md#281-simulation-modes), [ADR-011](08-design-decisions.md#adr-011-simulation-facility-with-three-leaf-execution-modes)). The event schema in §7.2.2 is the contract shared by live profiling, `PERFORMANCE`-mode simulation output, and `REPLAY`-mode reconstructed output.

### 7.2.8 Alerting Strategy (Rule O5)

Alerts fire when SLOs are at risk, not on arbitrary thresholds. Alert definitions are based on the SLO targets in §7.3.3 and the latency budgets in [Process View §4.8](04-process-view.md#48-latency-budgets-rule-x9).

| Alert | Condition | Severity | Action |
|-------|-----------|----------|--------|
| Task dispatch latency degraded | P95 Host→AICore dispatch > 15 μs over 1-minute window | Warning | Investigate scheduler queue depth; check for resource contention |
| Task dispatch latency critical | P99 Host→AICore dispatch > 30 μs over 1-minute window | Critical | Reduce submission rate; check for task slot exhaustion |
| Task slot pool near exhaustion | Any level's pool utilization > 80% | Warning | Increase pool size in `LevelParams`; investigate task completion backlog |
| Cross-node message latency degraded | P95 REMOTE_SUBMIT latency > 50 μs | Warning | Check network health; verify RDMA registration |
| Node heartbeat missed | No HEARTBEAT from peer for > 2× heartbeat interval | Critical | Trigger failure detection; evaluate `failure_policy` |
| Error rate spike | Error rate > 1% of tasks over 1-minute window | Warning | Inspect `ErrorContext` for patterns; check hardware health |
| Profiling overhead exceeded | Level 1 overhead > 1% wall time | Warning | Verify profiling buffer sizes; check for contention in collectors |
| Memory pressure | Available memory < 10% of total at any level | Warning | Check for tensor lifetime leaks; review scope exit timing |

Alerts are exposed via:
- **Metrics endpoint:** `LayerStats.get_stats()` provides all alert-relevant counters.
- **Log-based alerts:** Structured log entries with severity WARN/ERROR for external monitoring.
- **Python callbacks:** Optional alert callback registration via `Runtime.on_alert()`.

### 7.2.7 Integration Points

Profiling hooks are inserted at:
- Task state transitions (all 10 states).
- Scheduler decisions (dispatch, dependency resolution).
- Kernel dispatch (register writes, DMA setup).
- Memory operations (allocation, transfer, free).
- Ring buffer operations (head/tail advancement).
- Distributed message send/recv.
- Data movement start/end.
- Cross-layer submit/complete.

---

## 7.3 Performance

### 7.3.1 Identified Hot Paths

| Hot Path | Criticality | Optimization Strategy |
|----------|------------|----------------------|
| Task state machine transitions | Every task, every layer | Lock-free atomic state transitions; inline handlers for common paths |
| Scheduler ready queue dispatch | Every task dispatch | Lock-free SPSC/MPSC ring queues; zero-copy descriptor passing |
| Register Bank write/poll (Chip→Core) | Every AICore dispatch | Minimal MMIO writes; batched polling with spin-wait |
| DMA setup (Host→Device) | Every host-device transfer | Batched DMA descriptors; pre-registered memory regions |
| Fan-in counter decrement | Every dependency edge | Atomic decrement; branch-free notification |
| Ring buffer head advancement | Every scope exit / tensor reclaim | Cache-aligned ring pointers; single writer |

### 7.3.2 Data Movement Minimization (Rule P6)

- Tensor data stays in the Memory Scope where it was produced until explicitly moved.
- Cross-level transfers are explicit (Data Movement Tasks) and scheduled like compute tasks, enabling overlap with computation.
- Ring buffer memory management avoids unnecessary copies within a level.
- RDMA enables zero-copy network transfers for distributed data movement.

### 7.3.3 SLO Targets

| Metric | Target | Measurement Point |
|--------|--------|-------------------|
| Task submission latency | < 1 μs (host level) | `submit()` call to TaskHandle return |
| Scheduler dispatch latency | < 10 μs (host→device) | DEP_READY to DISPATCHED |
| AICore dispatch latency | < 100 ns (chip→core) | Register write to ACK |
| Cross-node message latency | < 10 μs (RDMA) | REMOTE_SUBMIT send to receive |
| Profiling overhead (Level 1) | < 1% wall time | Comparison with Level 0 |

### 7.3.4 Concurrency Optimization

- Scheduler and Workers run on separate threads to prevent mutual blocking.
- Completion path handlers are asynchronous by default, preventing blocking in dispatch threads.
- SPMD dispatch batches multiple register writes for parallel AICore launch.
- Distributed operations overlap computation with data movement via async `IMemoryOps`.

---

## 7.4 Chaos / Fault-Injection Scenario Matrix

> [UPDATED: A5-P5: Normative chaos / fault-injection matrix exercised via `IFaultInjector` (A8-P7).]
> Every row is a mandatory scenario with a pass criterion; rows are executed against the sim target (`a2a3sim`, `a5sim`) and a subset against onboard CI. Faults are scheduled through the sim-only `IFaultInjector` seam (§7.4.1).

| # | Category | Fault | Pass Criterion |
|---|----------|-------|----------------|
| 1 | Availability | Node kill (hard crash) during mid-flight SPMD | Surviving peers surface `CoordinatorLost` or `RemoteNodeFailure` within `heartbeat_timeout_ms`; no silent hang (A5-P3). |
| 2 | Network | Packet drop at {1, 5, 20, 50}% | Retries succeed within `RetryPolicy.max_retries = 5`; `max_ms = 2000` cap respected (A5-P1). |
| 3 | Network | Byzantine slow peer (10× RTT) | Circuit breaker transitions `CLOSED → OPEN → HALF_OPEN` per `CircuitBreaker.fail_threshold = 5` (A5-P2). |
| 4 | Time | Clock step ±1 s during heartbeat | Trace-alignment holds `skew_max_ns ≤ 100 µs` (A8-P6); no duplicated task dispatch. |
| 5 | Resource | Task-slot pool exhaust at {0.9, 1.0, 1.1}× capacity | `AdmissionStatus::REJECT(Exhaustion)` returned; no memory growth beyond `max_deferred` (A5-P8, A9-P5). |
| 6 | DMA / MMIO | DMA completion bit-flip | HAL surfaces `HardwareFault`; parent task transitions `COMPLETING → ERROR`. |
| 7 | RDMA | RDMA CRC corrupt | Per-peer dedup window (A3-P13) rejects duplicate; transport retries under A5-P1. |
| 8 | Availability | Coordinator isolation (partition) | `cluster_view` generation bumps on surviving coordinators per A10-P2; no split-brain dispatch. |
| 9 | Memory | Memory watermark breach | A8-P5 "Memory pressure" alert fires; submissions throttled via `admission_pressure_policy`. |

### 7.4.1 A6-Owned Security Fault Categories

Five additional rows cover the A6 trust-boundary perimeter and are required alongside the core matrix above:

| # | Category | Fault | Pass Criterion |
|---|----------|-------|----------------|
| S1 | Handshake downgrade | Peer negotiates weaker cipher than policy | Connection refused; `AuthenticationFailed` with breaker weight `breaker_auth_fail_weight = 10` (A5-P13). |
| S2 | Cert rotation | mTLS certificate rotated mid-session | New session established within `heartbeat_timeout_ms`; no admission loss (A6-P2, A6-P14). |
| S3 | RDMA `rkey` race | `rkey` revoked while peer mid-transfer | Transfer fails with `InvalidRkey`; no silent data read (A6-P5). |
| S4 | Quota skew | Per-tenant submit rate spikes past per-tenant quota | Offending tenant rejected with `AdmissionStatus::REJECT(Exhaustion)`; neighbor tenants unaffected (A6-P13). |
| S5 | Audit sink | Audit sink back-pressure / drop | `TraceEvents_dropped` / `LogEvents_dropped` fire A8-P9 alert; no silent loss of security audit records (A6-P6). |

### 7.4.2 `IFaultInjector` Sim-Only Seam

> [UPDATED: A8-P7: `IFaultInjector` is compiled only under the simulation targets and drives the matrix above.]
> `IFaultInjector::schedule_fault(FaultSpec)` is the single seam for reproducibly injecting the faults enumerated in §7.4 and §7.4.1. The symbol is **never linked into onboard release binaries**; it is hooked into `hal/sim`, `transport/` (sim-only deliveries), and `distributed/` dedup paths. Scenarios use `IFaultInjector` together with `IClock` (A8-P1) and `RecordedEventSource` (A8-P2) to produce bit-identical replays of chaos runs.

---

## 7.5 Data & State Reference (Cross-Reference)

> [UPDATED: A10-P8: Canonical one-page summary for data and state.]
> A single canonical **"Data & State Reference"** page has been introduced under A10-P8 that absorbs the previous "Per-data-element consistency model" (A10-P3) and the "Stateful / Stateless classification" (A10-P4). The page contains a Mermaid flowchart, the `{data_element, owner_module, writers, readers, consistency, sync primitive}` table (with R3 rows for `cluster_view`, `group_availability`, `remote_proxy_pool`, `outstanding_submission_set`), and the stateful/stateless classification in one place. A reader can trace any `BufferRef` through producer → consumer → retirement from this single page. Subsequent views (03–06) that reference `producer_index`, `cluster_view`, or `task_slot_pool` link here rather than duplicating the table.
