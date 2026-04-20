# Applied Changes — `docs/pypto-runtime-design/07-cross-cutting-concerns.md`

Run folder: `reviews/2026-04-18-171357/`.
Proposals applied (from `final/final-proposal.md`): **A4-P1**, **A5-P5**, **A8-P4**, **A8-P7**, **A10-P8**.

Line-count delta: 202 → 251 (+49).

Each change is a tightly-scoped `> [UPDATED: <id>: <reason>]` callout or a small dedicated section for matrices; no paragraph rewrites.

---

## Change 1 — A8-P4 (`dump_state()` tenant scope + admin token)

**Anchor:** §7.1.3 Known Limitations — immediately after the two existing bullets, before the `---` separator.

### Diff

```diff
 - Intra-node trust: The runtime assumes all components within a single node (Host, Device, Chip, Core) are mutually trusted. There is no encryption or authentication on intra-node channels (shared memory, register bank, DMA).
 - [ASSUMPTION] Network encryption (TLS/mTLS) for inter-node communication is deferred to the network backend implementation, not enforced by the runtime framework.

+> [UPDATED: A8-P4: `dump_state()` diagnostic endpoint is tenant-scoped by default.]
+> `Runtime::dump_state(DumpOptions) → StateDump` (see `modules/runtime.md §2.1`) defaults its scope to the caller's `logical_system_id` — the returned JSON includes only tasks, workers, and state owned by the caller's Logical System. A cross-tenant (unscoped) dump, including an aggregate view across all Logical Systems, requires a `diagnostic_admin_token` carried in `DumpOptions`; requests without the token MUST be rejected with `ErrorCode::PermissionDenied`. Token issuance and revocation follow the security audit trail (A6-P6) and the capability-scoped sink policy (A6-P10).
+
 ---
```

---

## Change 2 — A4-P1 (HAL enum canonical casing, read in this view)

**Anchor:** §7.2.1 Profiling Levels — immediately after the "Profiling level is runtime-switchable" line.

### Diff

```diff
 Profiling level is runtime-switchable (Rule O4).

+> [UPDATED: A4-P1: Canonical casing for HAL enums used across Observability and Performance sections.]
+> All HAL enums are referenced in this document using their canonical UPPER_SNAKE_CASE spellings: `Variant::{ONBOARD, SIM}` and `SimulationMode::{PERFORMANCE, FUNCTIONAL, REPLAY}`. Any occurrence of `Onboard`, `Sim`, `Performance`, `Functional`, or `Replay` in older snippets SHOULD be read as the corresponding canonical casing. Authoritative definition in `modules/hal.md §2.7` + ADR-011.
```

---

## Change 3 — A5-P5 (chaos matrix) + A8-P7 (IFaultInjector) — new §7.4

**Anchor:** end of file, after §7.3.4 Concurrency Optimization. New normative section §7.4 with two sub-sections.

### Diff

```diff
 ### 7.3.4 Concurrency Optimization

 - Scheduler and Workers run on separate threads to prevent mutual blocking.
 - Completion path handlers are asynchronous by default, preventing blocking in dispatch threads.
 - SPMD dispatch batches multiple register writes for parallel AICore launch.
 - Distributed operations overlap computation with data movement via async `IMemoryOps`.
+
+---
+
+## 7.4 Chaos / Fault-Injection Scenario Matrix
+
+> [UPDATED: A5-P5: Normative chaos / fault-injection matrix exercised via `IFaultInjector` (A8-P7).]
+> Every row is a mandatory scenario with a pass criterion; rows are executed against the sim target (`a2a3sim`, `a5sim`) and a subset against onboard CI. Faults are scheduled through the sim-only `IFaultInjector` seam (§7.4.1).
+
+| # | Category | Fault | Pass Criterion |
+|---|----------|-------|----------------|
+| 1 | Availability | Node kill (hard crash) during mid-flight SPMD | Surviving peers surface `CoordinatorLost` or `RemoteNodeFailure` within `heartbeat_timeout_ms`; no silent hang (A5-P3). |
+| 2 | Network | Packet drop at {1, 5, 20, 50}% | Retries succeed within `RetryPolicy.max_retries = 5`; `max_ms = 2000` cap respected (A5-P1). |
+| 3 | Network | Byzantine slow peer (10× RTT) | Circuit breaker transitions `CLOSED → OPEN → HALF_OPEN` per `CircuitBreaker.fail_threshold = 5` (A5-P2). |
+| 4 | Time | Clock step ±1 s during heartbeat | Trace-alignment holds `skew_max_ns ≤ 100 µs` (A8-P6); no duplicated task dispatch. |
+| 5 | Resource | Task-slot pool exhaust at {0.9, 1.0, 1.1}× capacity | `AdmissionStatus::REJECT(Exhaustion)` returned; no memory growth beyond `max_deferred` (A5-P8, A9-P5). |
+| 6 | DMA / MMIO | DMA completion bit-flip | HAL surfaces `HardwareFault`; parent task transitions `COMPLETING → ERROR`. |
+| 7 | RDMA | RDMA CRC corrupt | Per-peer dedup window (A3-P13) rejects duplicate; transport retries under A5-P1. |
+| 8 | Availability | Coordinator isolation (partition) | `cluster_view` generation bumps on surviving coordinators per A10-P2; no split-brain dispatch. |
+| 9 | Memory | Memory watermark breach | A8-P5 "Memory pressure" alert fires; submissions throttled via `admission_pressure_policy`. |
+
+### 7.4.1 A6-Owned Security Fault Categories
+
+Five additional rows cover the A6 trust-boundary perimeter and are required alongside the core matrix above:
+
+| # | Category | Fault | Pass Criterion |
+|---|----------|-------|----------------|
+| S1 | Handshake downgrade | Peer negotiates weaker cipher than policy | Connection refused; `AuthenticationFailed` with breaker weight `breaker_auth_fail_weight = 10` (A5-P13). |
+| S2 | Cert rotation | mTLS certificate rotated mid-session | New session established within `heartbeat_timeout_ms`; no admission loss (A6-P2, A6-P14). |
+| S3 | RDMA `rkey` race | `rkey` revoked while peer mid-transfer | Transfer fails with `InvalidRkey`; no silent data read (A6-P5). |
+| S4 | Quota skew | Per-tenant submit rate spikes past per-tenant quota | Offending tenant rejected with `AdmissionStatus::REJECT(Exhaustion)`; neighbor tenants unaffected (A6-P13). |
+| S5 | Audit sink | Audit sink back-pressure / drop | `TraceEvents_dropped` / `LogEvents_dropped` fire A8-P9 alert; no silent loss of security audit records (A6-P6). |
+
+### 7.4.2 `IFaultInjector` Sim-Only Seam
+
+> [UPDATED: A8-P7: `IFaultInjector` is compiled only under the simulation targets and drives the matrix above.]
+> `IFaultInjector::schedule_fault(FaultSpec)` is the single seam for reproducibly injecting the faults enumerated in §7.4 and §7.4.1. The symbol is **never linked into onboard release binaries**; it is hooked into `hal/sim`, `transport/` (sim-only deliveries), and `distributed/` dedup paths. Scenarios use `IFaultInjector` together with `IClock` (A8-P1) and `RecordedEventSource` (A8-P2) to produce bit-identical replays of chaos runs.
+
+---
+
+## 7.5 Data & State Reference (Cross-Reference)
+
+> [UPDATED: A10-P8: Canonical one-page summary for data and state.]
+> A single canonical **"Data & State Reference"** page has been introduced under A10-P8 that absorbs the previous "Per-data-element consistency model" (A10-P3) and the "Stateful / Stateless classification" (A10-P4). The page contains a Mermaid flowchart, the `{data_element, owner_module, writers, readers, consistency, sync primitive}` table (with R3 rows for `cluster_view`, `group_availability`, `remote_proxy_pool`, `outstanding_submission_set`), and the stateful/stateless classification in one place. A reader can trace any `BufferRef` through producer → consumer → retirement from this single page. Subsequent views (03–06) that reference `producer_index`, `cluster_view`, or `task_slot_pool` link here rather than duplicating the table.
```

---

## Cross-references updated

- `modules/hal.md §2.7` and ADR-011 / ADR-011-R2 (A4-P1 canonical casing).
- `modules/runtime.md §2.1` (A8-P4 `dump_state` primary owner).
- `modules/hal.md §9` / `modules/transport.md §9` / `modules/distributed.md §9` (A5-P5 / A8-P7 primary owners).
- A10-P3 / A10-P4 table content authored by A4 in the single "Data & State Reference" page (this root-view diff installs the cross-reference only; the full Mermaid + tables are authored in a follow-up slice).

## Not applied here (owner lives elsewhere)

- `modules/hal.md` rename of enum members is owned by the A4-P1 primary diff.
- `dump_state()` full API surface (parameters, JSON schema) is authored in `modules/runtime.md`.
- The full A10-P8 "Data & State Reference" body (Mermaid + tables) is authored by A4 per cross-aspect resolution; this file records the canonical cross-reference only.
