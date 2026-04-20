# Final Proposals — pypto-runtime-design

## Metadata

- **Run ID:** 2026-04-18-171357
- **Target:** `docs/pypto-runtime-design/`
- **Runtime Mode:** yes
- **Rounds Run:** 3
- **Convergence Status:** converged
- **Timestamp (UTC):** 2026-04-19

Weighted voting used A1 ×2, all others ×1. Stress round (R3) confirmed all 107 R2-agreed proposals hold, resolved 3 disputed proposals at `agree`, and introduced 6 new proposals. **Zero blocking objections; zero A1 hot-path vetoes; zero rejections; zero deferrals. Total agreed proposals: 116.**

---

## 1. Agreed Proposals (all 116)

Each entry is ≤ 12 lines. "Final amended text" folds in every R2 and R3 amendment from `round-2/extract.md`, `round-3/extract.md`, and `round-3/tally.md §3`.

### A1 — Performance & Hot Path (14)

### A1-P1 — Pre-size `producer_index`; forbid rehash on hot path
- **Owner / co-owners:** A1
- **Severity:** high
- **Hot-path impact:** none (was allocates)
- **Rule basis:** X2, X4, P5
- **Target doc(s):** `docs/pypto-runtime-design/02-logical-view/02-scheduler.md`; `docs/pypto-runtime-design/modules/scheduler.md`
- **Target section/anchor:** §2.1.3.1.B "Producer index" paragraph (~line 91); mirror in `modules/scheduler.md §8`
- **Final amended text (as of R3):** Allocate `producer_index` at Layer init with `capacity = max_outstanding_submissions × expected_outputs_per_submission × 2`, place in CONTROL region with `isolate_cache_line=true`, no rehash; overflow uses bounded open-addressed probing (L<0.5, probe ≤2), probe exhaustion returns `ErrorCode::ResourceExhausted` and triggers `IResourceAllocationPolicy.should_admit` back-pressure. Mirror config field `producer_index_capacity`.
- **Cross-references:** A1-P14 (placement/shard default), A10-P1 (shard default), A1-P8 (admission queue sizing)
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__02-logical-view__02-scheduler.md.diff.md`

### A1-P2 — Bound DATA-mode lock hold time; shard count as `LevelParam`
- **Owner / co-owners:** A1, A10
- **Severity:** high
- **Hot-path impact:** blocks (bounded)
- **Rule basis:** X3, P5
- **Target doc(s):** `02-logical-view/02-scheduler.md`; `modules/scheduler.md`
- **Target section/anchor:** §2.1.3.1.B "Concurrency and locking" block (end, after ~line 121)
- **Final amended text (as of R3):** Shared-mode admission hold ≤ `B×A` probes (≤ 500 ns for B=4,A=4); exclusive-mode hold bounded by `B×A` fan-in RMWs + `|S.OUT∪S.INOUT|` removals; default `producer_index_shards = max(1, scheduler_thread_count)`; p99 reader ≤ 3 μs, p99 writer ≤ 10 μs per shard under 8 shards.
- **Cross-references:** A1-P1, A10-P1, A10-P7
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__02-logical-view__02-scheduler.md.diff.md`

### A1-P3 — Function Cache LRU + capacity + HEARTBEAT presence
- **Owner / co-owners:** A1, A6
- **Severity:** medium
- **Hot-path impact:** none
- **Rule basis:** P2
- **Target doc(s):** `04-process-view.md`; `modules/hal.md`; `modules/distributed.md`; `modules/transport.md`
- **Target section/anchor:** §4.2.4 Function Registration & Caching; `modules/hal.md §8`; `modules/transport.md §2.3` HeartbeatPayload
- **Final amended text (as of R3):** Function Cache bounded by `function_cache_bytes` (default 64 MiB) with LRU; evictions recorded in `RuntimeStats`; peer cache presence published in `HeartbeatPayload.function_bloom[4]` (uint64_t); coordinator Bloom-checks before deciding to inline a binary in `REMOTE_SUBMIT`.
- **Cross-references:** A1-P6 (REMOTE_BINARY_PUSH staging), A2-P1 (header version bump)
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__04-process-view.md.diff.md`

### A1-P4 — Enumerate Task hot/cold field split; justify AoS
- **Owner / co-owners:** A1, A7
- **Severity:** medium
- **Hot-path impact:** relayouts (one-time doc)
- **Rule basis:** X4
- **Target doc(s):** `02-logical-view/07-task-model.md`; `modules/core.md`
- **Target section/anchor:** §2.4.1 Task struct; `modules/core.md §8` (normative)
- **Final amended text (as of R3):** Normative hot 64-B tier: `state, fan_in_counter, submission_id, exec_type_id, worker_id, dispatch_ts_ns`; cold tail: `function_id, args_blob_ptr, parent_task_key, dbg_name, trace_flags, error_ctx_ptr`. AoS per-slot with first cache line hot; state-transition handlers touch only hot line; ErrorContext written only on FAILED (cold tail). SoA for `fan_in_counter` considered and rejected with bench citation placeholder.
- **Cross-references:** A3-P4 (successor walk inlined in cold tail), A7-P3 (handle types in core)
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__modules__core.md.diff.md`

### A1-P5 — Latency budgets for SPMD fan-out + event-loop stages
- **Owner / co-owners:** A1, A3
- **Severity:** high
- **Hot-path impact:** none
- **Rule basis:** X9
- **Target doc(s):** `04-process-view.md`; `02-logical-view/02-scheduler.md`
- **Target section/anchor:** new §4.8.6 (SPMD fan-out) and §4.8.7 (event-loop iteration)
- **Final amended text (as of R3):** §4.8.6 SPMD Chip→N×AICore end-to-end ≤ 5 μs (batch prep ≤ 1 μs; register-bank block write ≤ 2 μs; 108-bit-popcount ACK ≤ 2 μs; a5 108-AICore case uses AND + 2×64 ctzll fallback, both paths < 10 ns). §4.8.7 event-loop iteration: Stage 1 ≤ 50 ns/source; Stage 2 ≤ 100 ns/event; Stage 3 ≤ 200 ns/event; idle iteration ≤ 300 ns.
- **Cross-references:** A1-P9 (bitmask availability), A3-P4/P5 (SPMD failure path)
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__04-process-view.md.diff.md`

### A1-P6 — Cap REMOTE_SUBMIT; stage binaries; dedup descriptor templates
- **Owner / co-owners:** A1, A10 (absorbed A10-P5)
- **Severity:** medium
- **Hot-path impact:** none (was extends)
- **Rule basis:** P6, X9
- **Target doc(s):** `modules/transport.md`; `04-process-view.md`; `modules/distributed.md`
- **Target section/anchor:** `transport.md §2.3` new `MessageType::REMOTE_BINARY_PUSH`; `§4.3` template table; `process-view §4.2.4`, `§4.8.2`
- **Final amended text (as of R3):** Function binaries travel on new `REMOTE_BINARY_PUSH` before first `REMOTE_SUBMIT` that needs them, gated by receiver `function_bloom` (A1-P3). `RemoteSubmitPayload` gains `uint32_t descriptor_template_id` + delta-encoded `TaskDescriptor[]`; per-peer template registry. Per-peer projection (ex-A10-P5) emits only tasks/edges/boundaries touching that peer. **R3 amendment:** add cold-start budget row "first-use template-miss ≤ 15 μs" in §4.8.2.
- **Cross-references:** A1-P3, A10-P5 (absorbed), A2-P1 (version bump)
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__modules__transport.md.diff.md`

### A1-P7 — Per-thread local seq + offline merge
- **Owner / co-owners:** A1, A8
- **Severity:** medium
- **Hot-path impact:** none (was extends)
- **Rule basis:** X3, P1
- **Target doc(s):** `modules/profiling.md`
- **Target section/anchor:** §4.2 SequenceAllocator; §5.1; §6 sink merge
- **Final amended text (as of R3):** Each `TlsRingBuffer` owns its own `uint64_t local_seq`; remove the global `SequenceAllocator::next` atomic. Sink collector merges per-thread streams ordered by `(timestamp_ns, thread_id, local_seq)` at drain; total order defined with `thread_id` as deterministic tiebreak.
- **Cross-references:** A8-P6 (time-alignment), A8-P12 (stable PhaseIds)
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__modules__profiling.md.diff.md`

### A1-P8 — Pre-size Outstanding/Ready; CONTROL placement
- **Owner / co-owners:** A1
- **Severity:** medium
- **Hot-path impact:** none (was allocates)
- **Rule basis:** X2, X4
- **Target doc(s):** `modules/scheduler.md`; `02-logical-view/02-scheduler.md`
- **Target section/anchor:** `scheduler.md §4` "Capacity & placement" per DS
- **Final amended text (as of R3):** `ReadyQueue` = intrusive min-heap over Task slot indices, capacity `max_outstanding_submissions × expected_tasks_per_submission × 2`, CONTROL region, isolate_cache_line. `Admission queue` = bounded SPSC ring of `SubmissionDescriptor*`, capacity `max_admission_backlog` (default 64), overflow returns `WouldBlock`. `OutstandingWindow` = fixed array indexed by `submission_id % max_outstanding_submissions`. **R3:** adds diagnostic sub-codes `TaskSlotExhausted`/`ReadyQueueExhausted` (routed via A3-P7).
- **Cross-references:** A1-P1, A1-P14, A3-P7
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__modules__scheduler.md.diff.md`

### A1-P9 — Bitmask-encode WorkerGroup availability per slot-type
- **Owner / co-owners:** A1
- **Severity:** low
- **Hot-path impact:** relayouts (bounded; 2×64 fallback)
- **Rule basis:** X4, X9
- **Target doc(s):** `02-logical-view/03-worker.md`; `modules/scheduler.md`
- **Target section/anchor:** §2.1.4.2 "Group-level availability"
- **Final amended text (as of R3):** Per-slot-type availability encoded as `uint64_t` bitmasks across groups (one bitmask per Worker Type + required-slot count tier); `select_workers` ANDs required masks and scans via `__builtin_ctzll`; complexity O(slot_types). >64 groups handled via 2×64 bitmap split (AND+ctzll fallback, both paths < 10 ns). Caps ≤ 64 groups per slot-type per bitmap.
- **Cross-references:** A1-P5, A7-P2
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__02-logical-view__03-worker.md.diff.md`

### A1-P10 — Workload-level profiling-overhead CI gate
- **Owner / co-owners:** A1, A8
- **Severity:** low
- **Hot-path impact:** none
- **Rule basis:** P1
- **Target doc(s):** `modules/profiling.md`; `07-cross-cutting-concerns.md`
- **Target section/anchor:** `profiling.md §9.2.4` (new)
- **Final amended text (as of R3):** Add `bench_profiling_overhead.cpp` driving 10 s of 1 M task submissions at PROFILING_LEVEL L0/L1/L2 on sim target; CI asserts L1 ≤ 1.01× L0 wall time and L2 ≤ 1.05× L0. Regression blocks merge.
- **Cross-references:** A8-P3, A8-P9
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__modules__profiling.md.diff.md`

### A1-P11 — Per-arg Python↔C budget + per-arg validation
- **Owner / co-owners:** A1, A3, A6 (absorbed A6-P8a)
- **Severity:** low
- **Hot-path impact:** none (was extends)
- **Rule basis:** X9, S3
- **Target doc(s):** `modules/bindings.md`; `04-process-view.md §4.8.1`
- **Target section/anchor:** `bindings.md §5.2, §9.5, §10`
- **Final amended text (as of R3):** Per-argument marshaling budget: scalar ≤ 20 ns; DLPack contiguous tensor import ≤ 200 ns; `BufferRef` passthrough ≤ 30 ns; total submit marshaling ≤ 500 ns for ≤ 8 args; above 8 args, Python→C stage adds `0.2 μs + 0.05 μs × extra_args`. Absorbs A6-P8a structural validation (dtype / device / shape × stride overflow / capsule ownership / zero-length dtype) inline under 30 ns; A6-P8b byte-cap remains A6-owned. **R3:** single validation pass enforced; SPMD shape sub-codes go here.
- **Cross-references:** A6-P8b, A3-P7
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__modules__bindings.md.diff.md`

### A1-P12 — `BatchedExecutionPolicy.max_batch` + tail budget
- **Owner / co-owners:** A1
- **Severity:** medium
- **Hot-path impact:** extends-latency (slow path only)
- **Rule basis:** X9, P5
- **Target doc(s):** `04-process-view.md §4.1.4, §4.8.1`
- **Target section/anchor:** Deployment-Modes table + §4.8.1 note
- **Final amended text (as of R3):** `BatchedExecutionPolicy.max_batch` default = 8 at Chip level. Tail latency for a batched task ≤ `max_batch × Chip→Core stage` (≤ 16 μs at max_batch=8). §4.8.1 uses adjusted stage target when BatchedExecutionPolicy active. **R3:** `Dedicated` remains the default at §4.1.4; no path silently enables Batched.
- **Cross-references:** A1-P5
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__04-process-view.md.diff.md`

### A1-P13 — Bound `args_blob` copy cost
- **Owner / co-owners:** A1, A7
- **Severity:** medium
- **Hot-path impact:** extends-latency (>1 KiB slow path only)
- **Rule basis:** P6, X9
- **Target doc(s):** `modules/hal.md`; `04-process-view.md §4.8.1`
- **Target section/anchor:** HAL `KernelDispatch` contract + §4.8.1 Notes column
- **Final amended text (as of R3):** `args_size ≤ 1 KiB` fast path — HAL copies through pre-registered ring slot (Host→Chip DMA ≥ 4 GB/s → ≤ 250 ns). For `args_size > 1 KiB`, scheduler stages args in data-plane `BufferRef` and passes handle; §4.8.1 budget extends by `0.5 μs/KiB` above threshold. **R3 (via A5-P12):** staging-allocation failure surfaces `ErrorCode::ResourceExhausted`.
- **Cross-references:** A5-P12
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__modules__hal.md.diff.md`

### A1-P14 — `producer_index` CONTROL placement + shard default
- **Owner / co-owners:** A1, A10 (absorbed A10-P10)
- **Severity:** medium
- **Hot-path impact:** none
- **Rule basis:** X4
- **Target doc(s):** `02-logical-view/02-scheduler.md`; `modules/scheduler.md`
- **Target section/anchor:** Producer-index paragraph; `scheduler.md §8` Configuration
- **Final amended text (as of R3):** `producer_index` resides in the Layer's CONTROL region (CONTROL_AREA at Chip/Device, CONTROL_HEAP at Host). Default shard count `max(1, scheduler_thread_count)`. Rehash disabled (A1-P1). Admission reads fit in CONTROL region.
- **Cross-references:** A1-P1, A1-P2, A10-P1
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__02-logical-view__02-scheduler.md.diff.md`

### A2 — Extensibility & Evolvability (9)

### A2-P1 — Version every public data contract
- **Owner / co-owners:** A2
- **Severity:** high
- **Hot-path impact:** none
- **Rule basis:** E6, E2
- **Target doc(s):** `02-logical-view/09-interfaces.md`; `modules/runtime.md`; `07-cross-cutting-concerns.md`; `modules/profiling.md`; `modules/error.md`; new `appendix-c-compatibility.md`
- **Target section/anchor:** `SubmissionDescriptor` def (~line 98); `MachineLevelDescriptor`; `DeploymentConfig`; `Event` header
- **Final amended text (as of R3):** Add `uint16_t schema_version` to cross-module contracts. **R3 (A9 amendment):** narrow blanket obligation → scope limited to multi-node wire messages (`MessageHeader`, `TaskDescriptor`, `FunctionDescriptor`, stats) and persistent artifacts (trace header); non-wire in-process types covered by A2-P5 checklist rather than blanket versioning. Validation runs at bindings/handshake boundaries only.
- **Cross-references:** A2-P5, A2-P9, A1-P6
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__appendix-c-compatibility.md.diff.md`

### A2-P2 — `LevelOverrides`: closed-for-v1, schema-registered v2
- **Owner / co-owners:** A2
- **Severity:** medium
- **Hot-path impact:** none
- **Rule basis:** E4, OCP, E5
- **Target doc(s):** `modules/runtime.md`; `02-logical-view/05-machine-level-registry.md`; `09-open-questions.md`
- **Target section/anchor:** `LevelOverrides` struct; Q6
- **Final amended text (as of R3):** Keep `LevelOverrides` typed and closed for v1 (satisfies A9 YAGNI); schema-registered v2 plan recorded in Q6 with trigger (first concrete out-of-tree level). Validation happens at `deployment_parser` init only.
- **Cross-references:** A9-P2 (closed enums), A2-P5
- **New ADR required?:** no
- **New open question required?:** no (extends existing Q6)
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__modules__runtime.md.diff.md`

### A2-P3 — Open extension-point enums; keep `DepMode` closed
- **Owner / co-owners:** A2
- **Severity:** medium
- **Hot-path impact:** none
- **Rule basis:** E4, OCP, X3
- **Target doc(s):** `modules/runtime.md`; `modules/hal.md`; `modules/distributed.md`; `08-design-decisions.md` (new ADR)
- **Target section/anchor:** `FailurePolicy`, `TransportBackend`, `SimulationMode`, `NodeRole` → string-keyed factories; `DepMode` stays closed
- **Final amended text (as of R3):** Deployment-time enums (`FailurePolicy`, `TransportBackend`, `NodeRole`) become string IDs resolved via registry at `Runtime::init`; `SimulationMode` stays open (per A9-P6 option iii). `DepMode` remains closed `enum class` with rationale recorded in A2-P8 known deviations and ADR-017 (closed-enum-in-hot-path policy).
- **Cross-references:** A9-P4, A9-P5, A9-P6, A2-P8
- **New ADR required?:** yes — ADR-017 "Closed-enum-in-hot-path policy" (co-driver with A9-P4)
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__modules__runtime.md.diff.md`

### A2-P4 — Migration & Transition Plan
- **Owner / co-owners:** A2
- **Severity:** high
- **Hot-path impact:** none
- **Rule basis:** E5, NFR-5
- **Target doc(s):** `03-development-view.md`; `appendix-b-codebase-mapping.md`; `10-known-deviations.md`
- **Target section/anchor:** new `§3.5 Migration & Transition Plan`
- **Final amended text (as of R3):** Add 3–5 phase table {phase, feature flag (`PTO_USE_NEW_SCHEDULER`, canary, on), current-class retired, new-class replacing, rollback gate, validation criterion}. Each of `DistScheduler`, `PTO2Runtime`, `AicpuExecutor` maps to a retirement phase with named rollback. Cross-link from Appendix-B and remove any implicit big-bang stance in `10-known-deviations.md`.
- **Cross-references:** A4-P8
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__03-development-view.md.diff.md`

### A2-P5 — Interface Evolution & BC Policy
- **Owner / co-owners:** A2
- **Severity:** medium
- **Hot-path impact:** none
- **Rule basis:** E2
- **Target doc(s):** new `appendix-c-compatibility.md`; `03-development-view.md`; every `modules/*.md`
- **Target section/anchor:** new §9 BC Policy table
- **Final amended text (as of R3):** One row per public interface with stability class `{stable-frozen, stable-additive, versioned-subinterface, unstable}` + deprecation procedure ("mark, ship warning for one minor release, remove"). **R3:** add cross-reference checklist listing every header that must receive version byte (A2-P1 narrow scope); add unknown-tag tolerance clause (v1 readers ignore unknown additive fields); add header-independence lint rule (A8-P11) to enforce `distributed/` header non-includable from `transport/`.
- **Cross-references:** A2-P1, A2-P6, A4-P6, A7-P4, A8-P11
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__appendix-c-compatibility.md.diff.md`

### A2-P6 — `IDistributedProtocolHandler` abstract boundary (v1 free function, v2 abstract class)
- **Owner / co-owners:** A2
- **Severity:** medium
- **Hot-path impact:** none
- **Rule basis:** E4, OCP, D3
- **Target doc(s):** `modules/distributed.md`; `modules/transport.md`; `02-logical-view/09-interfaces.md`
- **Target section/anchor:** new §2.6.2.1 Distributed Message Handler Registry
- **Final amended text (as of R3):** v1 ships a **free function** `DistributedMessageHandler` in `distributed/protocol.hpp` keyed by `MessageType` (O(1) table; top-N builtin short-circuit + indexed load + indirect branch). **R3 (A9 amendment):** demote abstract `IDistributedProtocolHandler` class → declared as free function only until second backend exists and ADR records adoption trigger. Header-independence invariant I-DIST-1 enforces `distributed/` header non-includable from `transport/` (A7-P4). CRTP / `final` devirt on single v1 impl.
- **Cross-references:** A7-P4, A2-P5, ADR-015 I-DIST-1
- **New ADR required?:** yes — ADR-015 Distributed header registration policy (driver shared with A7-P4)
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__modules__distributed.md.diff.md`

### A2-P7 — Reserve async-policy extension seam (Q-record only)
- **Owner / co-owners:** A2
- **Severity:** medium
- **Hot-path impact:** none
- **Rule basis:** E1, E5, G3
- **Target doc(s):** `09-open-questions.md`
- **Target section/anchor:** new Q15 entry (append at end)
- **Final amended text (as of R3):** **Q-record only in `09-open-questions.md`; no v1 interface shipped.** Q15 names three extension axes: async-policy (Q11), transport-capability semantics (R3 amendment — RDMA rkey vs TCP TLS binding), and async-submit return path. Reviewers A3/A5/A9 flip back to disagree if Phase-5 final text reintroduces any v1 interface slot.
- **Cross-references:** Q11 (existing)
- **New ADR required?:** no
- **New open question required?:** yes — Q15 "Async-policy extension seam with transport-capability axis"
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__09-open-questions.md.diff.md`

### A2-P8 — Record intentional closures as known deviations
- **Owner / co-owners:** A2
- **Severity:** low
- **Hot-path impact:** none
- **Rule basis:** E1, Rule-Exceptions procedure
- **Target doc(s):** `10-known-deviations.md`; cross-link from `09-open-questions.md` Q5/Q11/Q12
- **Target section/anchor:** append three deviation entries
- **Final amended text (as of R3):** Add four-part entries (rule, violation, justification, mitigation) for: (a) Registry frozen at init (Q5); (b) Synchronous-only policies (Q11); (c) `DepMode` closed enum (hot path — links ADR-017).
- **Cross-references:** A2-P3, ADR-017
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__10-known-deviations.md.diff.md`

### A2-P9 — Versioned trace schema
- **Owner / co-owners:** A2, A4, A8
- **Severity:** medium
- **Hot-path impact:** none
- **Rule basis:** E6
- **Target doc(s):** `07-cross-cutting-concerns.md §7.2.2`; `modules/profiling.md`; `modules/hal.md`
- **Target section/anchor:** Trace-file header struct; §4.8.5 reference
- **Final amended text (as of R3):** Prepend one-off trace-file header `{magic, trace_schema_version: uint16_t, platform, mode}`. `replay_engine` validates header at REPLAY init; mismatched major version → `ProtocolVersionMismatch`. Header-only, not per-event.
- **Cross-references:** A6-P12, A9-P6 (REPLAY deferred v2), A2-P1
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__modules__profiling.md.diff.md`

### A3 — Functional Sanity & Correctness (15)

### A3-P1 — Add `ERROR` state (+ optional `CANCELLED`) to Task FSM
- **Owner / co-owners:** A3, A4
- **Severity:** blocker
- **Hot-path impact:** none
- **Rule basis:** LSP, D7, V5
- **Target doc(s):** `04-process-view.md §4.3.1-.2`; `02-logical-view/07-task-model.md §2.4.4`
- **Target section/anchor:** state diagram; handler table
- **Final amended text (as of R3):** P1a (blocker): add `ERROR` reachable from `DISPATCHED`, `EXECUTING`, `COMPLETING`; add `on_fail(task, error)` handler. P1b (medium, conditional): optional `CANCELLED` from `PENDING`, `DEP_READY`, `RESOURCE_READY`, `DISPATCHED`, `EXECUTING` with `on_cancel(task, reason)` handler. **R3 (A4-R3-P2):** canonical spelling `COMPLETING → ERROR` (Task) + `FAILED` (Worker); unify across `06-scenario-view.md:91/93/95/108/111`.
- **Cross-references:** A3-P4, A3-P5, A4-P8, A4-P9, A4-R3-P2, ADR-016
- **New ADR required?:** yes — ADR-016 "TaskState canonical spelling + transitions" (co-driven by A3-P1 + A4-R3-P2)
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__04-process-view.md.diff.md`

### A3-P2 — `std::expected<SubmissionHandle, ErrorContext>` for `submit()`; ADR
- **Owner / co-owners:** A3, A9
- **Severity:** high
- **Hot-path impact:** none
- **Rule basis:** LSP, D7
- **Target doc(s):** `02-logical-view/09-interfaces.md §2.6.1, §2.6.3`; `04-process-view.md §4.5.6`; `08-design-decisions.md`
- **Target section/anchor:** `ISchedulerLayer::submit` signature
- **Final amended text (as of R3):** **§5 amendment pins `std::expected<SubmissionHandle, ErrorContext>` as the normative signature.** `AdmissionDecision::WAIT` blocks; `REJECT` returns `unexpected(ErrorContext{AdmissionRejected, ...})`. Convenience `throw`ing overload provided. Record as ADR (cross-reference from interfaces and process views).
- **Cross-references:** A3-P7, A7-P2, A9-P5 (AdmissionStatus unification), A3-P10
- **New ADR required?:** no (recorded as inline ADR reference; covered by existing ADR list + ADR-017)
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__02-logical-view__09-interfaces.md.diff.md`

### A3-P3 — Admission-path failure scenario
- **Owner / co-owners:** A3, A5
- **Severity:** high
- **Hot-path impact:** none
- **Rule basis:** V4, R6
- **Target doc(s):** `06-scenario-view.md §6.2` (new §6.2.5–§6.2.7)
- **Target section/anchor:** new sub-sections
- **Final amended text (as of R3):** New §6.2.5 "Admission Rollback on Cyclic `intra_edges`"; §6.2.6 "Admission Rejection on Workspace Exhaustion"; §6.2.7 "`NONE` Mode Misuse (Debug Assertion)". Each cites `AdmissionRejected / WorkspaceExhausted / CyclicDependency` codes. **R3:** cross-references §6.2.3 admission storm.
- **Cross-references:** A3-P7, A3-P8, A9-P5
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__06-scenario-view.md.diff.md`

### A3-P4 — Producer-failure → consumer propagation
- **Owner / co-owners:** A3
- **Severity:** high
- **Hot-path impact:** none (precomputed successor list; cold path)
- **Rule basis:** R1, DS3, G3
- **Target doc(s):** `02-logical-view/02-scheduler.md §2.1.3.5`; `04-process-view.md §4.5, §4.7.4`; `06-scenario-view.md §6.2.4`
- **Target section/anchor:** `SchedulerEvent::Type` enum; Task FSM edge
- **Final amended text (as of R3):** Add `DEP_FAILED(producer_key, ErrorCode)` event; TaskManager on `TASK_FAILED(producer)` walks precomputed successor list (inlined small-vector in A1-P4 cold tail) and emits `DEP_FAILED`; consumer `PENDING → ERROR`; recursive propagation. **R3:** extend edit sketch to include SPMD aggregation edge — parent is notified on any sub-task failure.
- **Cross-references:** A3-P5, A5-P4, A1-P4
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__02-logical-view__02-scheduler.md.diff.md`

### A3-P5 — Sibling cancellation policy
- **Owner / co-owners:** A3
- **Severity:** high
- **Hot-path impact:** none
- **Rule basis:** DS4, G3
- **Target doc(s):** `04-process-view.md §4.7.4, §4.3.2`; `02-logical-view/02-scheduler.md §2.1.3.1`; `06-scenario-view.md §6.2.1/6.2.2`
- **Target section/anchor:** `on_completion` handler section
- **Final amended text (as of R3):** Default: on first child failure, set parent `child_error_detected`; subsequent sibling dispatches for same parent cancelled before dispatch; siblings already EXECUTING complete but outputs discarded and contribute `CANCELLED` entries to `ErrorContext.remote_chain`. Policy hook: `IResourceAllocationPolicy::on_child_error()`. **R3:** cancelled siblings retain their `spmd_index` in `ErrorContext.remote_chain` (24-way SPMD edge closed).
- **Cross-references:** A3-P4, A3-P1 (CANCELLED state), A8-P7
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__04-process-view.md.diff.md`

### A3-P6 — Requirement ↔ Scenario traceability matrix
- **Owner / co-owners:** A3
- **Severity:** medium
- **Hot-path impact:** none
- **Rule basis:** G1, V3
- **Target doc(s):** `06-scenario-view.md §6.0` (new); new §6.1.4–§6.1.7
- **Target section/anchor:** new §6.0 table
- **Final amended text (as of R3):** New §6.0 table maps each FR/NFR to a scenario or cite an ADR / module test. New §6.1.4 Function Registration (FR-7); §6.1.5 Simulation Mode Walkthrough (FR-10, NFR-3); §6.1.6 Add-a-Level OCP (NFR-2); §6.1.7 Multi-Tenancy Isolation (FR-9). **R3:** under A9-P6 option (iii), FR-10 maps to §6.1.1–6.1.3 (FUNCTIONAL) + ADR-011-R2.
- **Cross-references:** A9-P6, A6-P9
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__06-scenario-view.md.diff.md`

### A3-P7 — Submission preconditions at Python↔C boundary
- **Owner / co-owners:** A3, A1, A6
- **Severity:** medium
- **Hot-path impact:** none (single validation pass on cold pre-admission path; `FE_VALIDATED` fast-path bit skips in release)
- **Rule basis:** G3, S3
- **Target doc(s):** `02-logical-view/09-interfaces.md §2.6.1.A`; `02-logical-view/02-scheduler.md §2.1.3.1.A`; `modules/error.md §2.1`
- **Target section/anchor:** SubmissionDescriptor Preconditions section
- **Final amended text (as of R3):** Preconditions: `tasks.size() ≥ 1`; `intra_edges[i].producer ≠ consumer`; indices in `[0, tasks.size())`; boundary subsets valid; workspace subrange bounds. Sub-codes `EmptyTasks / SelfLoopEdge / EdgeIndexOOB / BoundaryIndexOOB / WorkspaceSubrangeOOB / WorkspaceSizeMismatch`. **R3:** add SPMD sub-codes `SpmdIndexOOB`, `SpmdSizeMismatch`, `SpmdBlockDimZero`, plus routed `TaskSlotExhausted`/`ReadyQueueExhausted` from A1-P8.
- **Cross-references:** A3-P2, A1-P11, A6-P8b, A1-P8
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__02-logical-view__09-interfaces.md.diff.md`

### A3-P8 — Cyclic `intra_edges` detection (debug / FE_VALIDATED)
- **Owner / co-owners:** A3
- **Severity:** medium
- **Hot-path impact:** bounded; FE_VALIDATED fast path skips
- **Rule basis:** G3, X9
- **Target doc(s):** `02-logical-view/02-scheduler.md §2.1.3.1.A`; `04-process-view.md §4.8.4`
- **Target section/anchor:** admit() step 2
- **Final amended text (as of R3):** DFS with white/grey/black; cost O(|tasks|+|intra_edges|); ≤ 50 ns per edge amortized. Release builds skip when `SubmissionDescriptor.flags & FE_VALIDATED` set; debug builds always run. §4.8.4 table row added.
- **Cross-references:** A3-P7 (FE_VALIDATED flag)
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__02-logical-view__02-scheduler.md.diff.md`

### A3-P9 — SPMD index/size delivery contract
- **Owner / co-owners:** A3
- **Severity:** medium
- **Hot-path impact:** none
- **Rule basis:** G3, LSP
- **Target doc(s):** `02-logical-view/06-function-types.md`; `02-logical-view/07-task-model.md §2.4.6`
- **Target section/anchor:** AICore Function ABI; SPMD Submission subsection
- **Final amended text (as of R3):** `spmd_index` and `spmd_size` delivered in `TaskArgs.scalar_args` at reserved indices 0 and 1. ABI preserved under A9-P4 `Kind` drop (derivable from `spmd.has_value()`).
- **Cross-references:** A9-P4, A1-P11
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__02-logical-view__06-function-types.md.diff.md`

### A3-P10 — Python exception mapping completeness
- **Owner / co-owners:** A3, A6
- **Severity:** medium
- **Hot-path impact:** none
- **Rule basis:** G1, D7
- **Target doc(s):** `04-process-view.md §4.7.6`; `modules/bindings.md §2.2`
- **Target section/anchor:** exception mapping table
- **Final amended text (as of R3):** One row per `Domain` value mapping to a `simpler.*Error` subclass; satisfies `python_mapping_completeness` test. **R3:** scenario 6.2.3 text updated — Scheduler-domain errors raise `simpler.SchedulerError` (not generic `simpler.RuntimeError`) to match full domain mapping.
- **Cross-references:** A3-P2, A6-P1
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__modules__bindings.md.diff.md`

### A3-P11 — `[ASSUMPTION]` marker at `complete_in_future`
- **Owner / co-owners:** A3
- **Severity:** low
- **Hot-path impact:** none
- **Rule basis:** G3
- **Target doc(s):** `02-logical-view/07-task-model.md §2.4.5`; `09-open-questions.md` Q8
- **Target section/anchor:** Deferred Completion paragraph
- **Final amended text (as of R3):** Prepend `[ASSUMPTION]` tag at point of use; add anchor link to Q8 `#q8-deferred-completion-tracking`.
- **Cross-references:** Q8
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__02-logical-view__07-task-model.md.diff.md`

### A3-P12 — `drain()` / `submit()` concurrency + distributed semantics
- **Owner / co-owners:** A3, A5, A10
- **Severity:** medium
- **Hot-path impact:** none
- **Rule basis:** G3, R4
- **Target doc(s):** `02-logical-view/09-interfaces.md §2.6.1`; `modules/error.md §2.1`
- **Target section/anchor:** behavioral contracts
- **Final amended text (as of R3):** After `drain()` entered, `submit()` returns `AdmissionRejected(DrainInProgress)`; drain flag sticky; multiple concurrent `drain()` idempotent; `shutdown()` after `drain()` safe. Add `DrainInProgress` to Core domain. Cross-Pod drain semantics: drain flag propagates via `REMOTE_DRAIN` with same sticky guarantee.
- **Cross-references:** A10-P3, A5-P8
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__02-logical-view__09-interfaces.md.diff.md`

### A3-P13 — Cross-node ordering (FIFO + idempotent + dup-detect)
- **Owner / co-owners:** A3, A5, A8, A10
- **Severity:** medium
- **Hot-path impact:** none
- **Rule basis:** DS3, G3
- **Target doc(s):** `04-process-view.md §4.6.1–§4.6.3`
- **Target section/anchor:** message table; consistency guarantee
- **Final amended text (as of R3):** `[ASSUMPTION] Per-(source, destination, task-chain) FIFO; idempotent delivery; duplicate detection via TaskKey.generation.` Reorder buffer size `remote_reorder_window` (default 64) → drop with `ProtocolViolation` on overflow. **R3 (A10 handoff):** dup-detect window tagged **per-peer-pair** so 28 pairs at 8× Pods do not overflow a node-global window.
- **Cross-references:** A5-P10, A10-P3
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__04-process-view.md.diff.md`

### A3-P14 — `COMPLETING`-skip transition at leaf
- **Owner / co-owners:** A3
- **Severity:** low
- **Hot-path impact:** none
- **Rule basis:** LSP, V5
- **Target doc(s):** `04-process-view.md §4.3.5, §4.3.1`
- **Target section/anchor:** Task/Worker state notes
- **Final amended text (as of R3):** Add edge `EXECUTING → COMPLETED` labelled "leaf worker, no children" alongside `EXECUTING → COMPLETING`; §4.3.5 table gains `Skip COMPLETING?` column stating AICore/leaf workers take skip edge.
- **Cross-references:** A3-P1
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__04-process-view.md.diff.md`

### A3-P15 — Debug-mode `NONE`-dep cross-check
- **Owner / co-owners:** A3
- **Severity:** low
- **Hot-path impact:** none (debug-only cold path)
- **Rule basis:** G3, X6
- **Target doc(s):** `02-logical-view/02-scheduler.md §2.1.3.1.B`; `07-cross-cutting-concerns.md §7.2`
- **Target section/anchor:** DATA/BARRIER/NONE table
- **Final amended text (as of R3):** Debug builds: `NONE` admission performs dry-run `DATA` lookup against `producer_index`; any hit logged `WARN NoneModeHiddenDependency producer=... consumer=... buffer=...`, not enforced. Release unaffected.
- **Cross-references:** A8-P7
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__02-logical-view__02-scheduler.md.diff.md`

### A4 — Document Consistency (11)

### A4-P1 — Canonicalize HAL enum member casing
- **Owner / co-owners:** A4
- **Severity:** high
- **Hot-path impact:** none
- **Rule basis:** D7, V5
- **Target doc(s):** `modules/hal.md`
- **Target section/anchor:** §2.7 Public Data Types (lines 163-164); §5 Config table (lines 394-395)
- **Final amended text (as of R3):** Rewrite `Variant { Onboard, Sim }` → `{ ONBOARD, SIM }`; `SimulationMode { Performance, Functional, Replay }` → `{ PERFORMANCE, FUNCTIONAL, REPLAY }` to match every other view and ADR-011. Config column examples updated accordingly. Rename contained in `modules/hal.md` + ADR-011.
- **Cross-references:** ADR-011-R2
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__modules__hal.md.diff.md`

### A4-P2 — Fix broken anchor
- **Owner / co-owners:** A4
- **Severity:** blocker
- **Hot-path impact:** none
- **Rule basis:** D7, V5
- **Target doc(s):** `02-logical-view/07-task-model.md`
- **Target section/anchor:** line 45 Outstanding Submission Window pointer
- **Final amended text (as of R3):** `#21_3_1_a-submission-admission--outstanding-window` → `#2131a-submission-admission--outstanding-window`.
- **Cross-references:** —
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__02-logical-view__07-task-model.md.diff.md`

### A4-P3 — Fix invariant count prose
- **Owner / co-owners:** A4
- **Severity:** high
- **Hot-path impact:** none
- **Rule basis:** D7, G4
- **Target doc(s):** `02-logical-view/12-dependency-model.md`
- **Target section/anchor:** line 107
- **Final amended text (as of R3):** "Two invariants hold…" → "Three invariants hold across the pipeline and are relied upon by every stage:" to match enumerated I1/I2/I3.
- **Cross-references:** —
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__02-logical-view__12-dependency-model.md.diff.md`

### A4-P4 — Reorder Open Questions numerically
- **Owner / co-owners:** A4
- **Severity:** medium
- **Hot-path impact:** none
- **Rule basis:** D7, V5
- **Target doc(s):** `09-open-questions.md`
- **Target section/anchor:** Q9 section (currently line 193)
- **Final amended text (as of R3):** Move Q9 block between Q8 (line 99) and Q10 (line 112) so section order is strictly ascending Q1..Q14.
- **Cross-references:** —
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__09-open-questions.md.diff.md`

### A4-P5 — Event-loop plumbing glossary
- **Owner / co-owners:** A4, A2
- **Severity:** medium
- **Hot-path impact:** none
- **Rule basis:** D7
- **Target doc(s):** `appendix-a-glossary.md`
- **Target section/anchor:** after EventHandlingConfig entry (~line 28)
- **Final amended text (as of R3):** Add three alphabetical rows: `EventLoopDeploymentConfig`, `SourceCollectionConfig` (pointer-only row, since A9-P7 folds it), `IEventCollectionPolicy` (one-line pointer per A9-P2 collapse to `IEventLoopDriver`). **R3:** conditional branch resolved — `IEventCollectionPolicy` entry reads "(retired; see `IEventLoopDriver` test-only seam + appendix of future extensions in ADR-010)".
- **Cross-references:** A9-P2, A9-P7
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__appendix-a-glossary.md.diff.md`

### A4-P6 — Cross-link top-level views to ADRs
- **Owner / co-owners:** A4
- **Severity:** medium
- **Hot-path impact:** none
- **Rule basis:** V2, G5
- **Target doc(s):** `03-development-view.md`; `04-process-view.md`; `05-physical-view.md`; `06-scenario-view.md`
- **Target section/anchor:** end-of-file "Related ADRs" paragraph (minimal inline cites)
- **Final amended text (as of R3):** 03 cites ADR-011, ADR-011-R2; 04 cites ADR-010, ADR-012, ADR-013; 05 cites ADR-005, ADR-011, ADR-019 (admission shard deployment cue); 06 cites ADR-001, ADR-008, ADR-010, ADR-012, ADR-016 (TaskState). Single-citation only where decision is local.
- **Cross-references:** A2-P5 (BC appendix), A4-P5
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__04-process-view.md.diff.md`

### A4-P7 — Unify L0 component label
- **Owner / co-owners:** A4
- **Severity:** low
- **Hot-path impact:** none
- **Rule basis:** V5, D7
- **Target doc(s):** `05-physical-view.md`; `06-scenario-view.md`
- **Target section/anchor:** Single-Node Deployment diagram lines 35–40
- **Final amended text (as of R3):** `Dispatcher: AICore dispatch (register poll)` → `Scheduler: AicoreDispatcher (register poll)` across all boxes. Role-interface list (A7-P2) sits in glossary, not the diagram.
- **Cross-references:** A7-P2
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__05-physical-view.md.diff.md`

### A4-P8 — Sync Appendix-B TaskState count + new states
- **Owner / co-owners:** A4, A3
- **Severity:** low
- **Hot-path impact:** none
- **Rule basis:** D7
- **Target doc(s):** `appendix-b-codebase-mapping.md`
- **Target section/anchor:** lines 42-43 (TaskState Host/Device rows)
- **Final amended text (as of R3):** `core/task.h (10 states)` → `core/task.h (10 lifecycle states + ERROR; CANCELLED if adopted; see modules/core.md §2.3)` — add explicit footnote "if A3-P1b adopted" for CANCELLED.
- **Cross-references:** A3-P1, A4-R3-P1
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__appendix-b-codebase-mapping.md.diff.md`

### A4-P9 — Expand `Task State` glossary entry
- **Owner / co-owners:** A4, A3
- **Severity:** low
- **Hot-path impact:** none
- **Rule basis:** D7
- **Target doc(s):** `appendix-a-glossary.md`
- **Target section/anchor:** `Task State` entry (line 88)
- **Final amended text (as of R3):** Enumerate canonical lifecycle: `FREE → SUBMITTED → PENDING → DEP_READY → RESOURCE_READY → DISPATCHED → EXECUTING → COMPLETING → COMPLETED → RETIRED` (plus `ERROR`; optional `CANCELLED`). Link to `modules/core.md §2.3` and ADR-016.
- **Cross-references:** A3-P1, A4-R3-P2
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__appendix-a-glossary.md.diff.md`

### A4-R3-P1 — Add `WorkerState` row to Appendix-B + glossary
- **Owner / co-owners:** A4
- **Severity:** medium
- **Hot-path impact:** none
- **Rule basis:** D7, V5
- **Target doc(s):** `appendix-b-codebase-mapping.md`; `appendix-a-glossary.md`
- **Target section/anchor:** append row next to TaskState
- **Final amended text (as of R3):** Append `WorkerState` row enumerating `READY, BUSY, DRAINING, RETIRED, FAILED, UNAVAILABLE + {Permanent, Quarantine(duration)}` per A5-P9 DRY-fold amendment. Symmetry with TaskState entry; cross-reference A5-P11 FSM.
- **Cross-references:** A5-P9, A5-P11
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__appendix-b-codebase-mapping.md.diff.md`

### A4-R3-P2 — Unify terminal-failed-Task spelling across scenario view
- **Owner / co-owners:** A4
- **Severity:** low
- **Hot-path impact:** none
- **Rule basis:** D7, V5
- **Target doc(s):** `06-scenario-view.md`
- **Target section/anchor:** lines 91, 93, 95, 108, 111
- **Final amended text (as of R3):** Collapse three spellings (`COMPLETED(error)`, `ERROR`, `FAILED`) to canonical `COMPLETING → ERROR` for Task and `FAILED` for Worker (per A3-P1 + ADR-016).
- **Cross-references:** A3-P1, ADR-016
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__06-scenario-view.md.diff.md`

### A5 — Reliability & Fault Tolerance (14)

### A5-P1 — Exponential backoff + jitter on remote retries
- **Owner / co-owners:** A5
- **Severity:** high
- **Hot-path impact:** none (failure path)
- **Rule basis:** R2
- **Target doc(s):** `modules/distributed.md`; `modules/runtime.md`; `05-physical-view.md`
- **Target section/anchor:** `DeploymentConfig` new `RetryPolicy` struct; distributed §4 Config table
- **Final amended text (as of R3):** `struct RetryPolicy { uint32_t base_ms=50; uint32_t max_ms=2000; double jitter=0.3; uint32_t max_retries=5; };` The n-th retry waits `min(max_ms, base_ms · 2^n) · (1 + U[-jitter,+jitter])`. **R3:** cap `max_ms=2000` confirmed; A8-P5 AlertRule surfaces SLO breach at n=4.
- **Cross-references:** A5-P8, A8-P5
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__modules__distributed.md.diff.md`

### A5-P2 — Per-peer circuit breaker
- **Owner / co-owners:** A5, A6, A10
- **Severity:** high
- **Hot-path impact:** none (thread_local TSC short-circuit)
- **Rule basis:** R3
- **Target doc(s):** `modules/distributed.md`; `modules/runtime.md`; `02-logical-view/02-scheduler.md`
- **Target section/anchor:** new §3.4 "Peer health & circuit breaker"; `DeploymentConfig`
- **Final amended text (as of R3):** `struct CircuitBreaker { uint32_t fail_threshold=5; uint32_t cooldown_ms=10000; uint32_t half_open_max_in_flight=1; };` States `{CLOSED, OPEN, HALF_OPEN}` per peer. Fast path: `thread_local` last-checked TSC short-circuits steady-state CLOSED. **R3 (via A5-P13):** `breaker_auth_fail_weight=10` weights `AuthenticationFailed` to prevent flapping-credential storms defeating fail_threshold. Subsumed into unified peer-health FSM (A5-P11).
- **Cross-references:** A5-P11, A5-P13, A6-P2
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__modules__distributed.md.diff.md`

### A5-P3 — v1 deterministic coordinator fail-fast (absorbed A10-P2a)
- **Owner / co-owners:** A5, A10
- **Severity:** high
- **Hot-path impact:** none
- **Rule basis:** R5
- **Target doc(s):** `modules/distributed.md` new §3.5; `09-open-questions.md` Q5; `10-known-deviations.md`; `05-physical-view.md §5.4.3`
- **Target section/anchor:** Coordinator membership & failover
- **Final amended text (as of R3):** v1 = deterministic fail-fast — every surviving peer surfaces `CoordinatorLost` within `heartbeat_timeout_ms`; Python driver observes `DistributedError`; no silent hang. **R3 (A5):** add `coordinator_liveness_timeout_ms < heartbeat_timeout_ms`, default `3 × heartbeat_interval_ms`. **R3 (A10-P2a):** scope pinned to failed Pod only; `cluster_view` generation bumps to surviving-coordinator list (no cluster-wide fail-closed). v2 = decentralized roadmap recorded as ADR-005 extension.
- **Cross-references:** A10-P2a (absorbed), A5-P11, A10-P6, A6-P2
- **New ADR required?:** no (extends ADR-005)
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__modules__distributed.md.diff.md`

### A5-P4 — `idempotent: bool` on `TaskDescriptor`
- **Owner / co-owners:** A5, A3
- **Severity:** medium
- **Hot-path impact:** none (checked only on retry)
- **Rule basis:** DS4
- **Target doc(s):** `02-logical-view/07-task-model.md §2.4.1`; `modules/scheduler.md §5`; `modules/bindings.md §2.1`; `05-physical-view.md §5.4.3`
- **Target section/anchor:** `TaskDescriptor` / `TaskArgs` struct
- **Final amended text (as of R3):** Add `bool idempotent = true;` Scheduler MAY retry only when `idempotent == true`; Python `simpler.Task(..., idempotent=True)`. **R3 (A3 handoff):** scenario 6.2.3:125 clarified — caller retry of rejected submission is **independent of** the `idempotent` flag (that flag gates scheduler-initiated retry only).
- **Cross-references:** A3-P4, A5-P1, A5-P10
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__02-logical-view__07-task-model.md.diff.md`

### A5-P5 — Chaos / fault-injection scenario matrix
- **Owner / co-owners:** A5, A8
- **Severity:** medium
- **Hot-path impact:** none (test-only seam)
- **Rule basis:** R6
- **Target doc(s):** `07-cross-cutting-concerns.md §7.4` (new); `modules/transport.md §9`; `modules/hal.md §9`; `modules/distributed.md §9`
- **Target section/anchor:** new mandatory chaos matrix
- **Final amended text (as of R3):** Matrix rows: node kill; packet drop {1,5,20,50}%; Byzantine slow peer 10×RTT; clock step ±1 s during heartbeat; task-slot pool exhaust {0.9,1.0,1.1}× capacity; DMA completion bit-flip; RDMA CRC corrupt; coordinator isolation; memory watermark breach. Pass criteria per row; `IFaultInjector` seam (A8-P7). Enumerates 5 A6 categories (handshake-downgrade, cert-rotation, rkey-race, quota-skew, audit-sink). **R3:** option (iii) keeps `RecordedEventSource + A8-P2 + A8-P1` as the reproduction vehicle.
- **Cross-references:** A8-P7, A8-P1, A8-P2, A6-P1
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__07-cross-cutting-concerns.md.diff.md`

### A5-P6 — Scheduler-thread watchdog / deadman timer
- **Owner / co-owners:** A5, A8
- **Severity:** medium
- **Hot-path impact:** none
- **Rule basis:** R5, O5
- **Target doc(s):** `02-logical-view/02-scheduler.md §2.1.3.4`; `modules/scheduler.md §4/§5`; `07-cross-cutting-concerns.md §7.3`
- **Target section/anchor:** event-loop description
- **Final amended text (as of R3):** Scheduler writes a monotonic timestamp to a deadman slot at top of each event-loop cycle. Low-priority monitor thread checks at `scheduler_watchdog_ms` (default 250 ms); raises `SchedulerStalled` when delta > budget. **R3 (normative elision, A1 handoff):** store elided into per-cycle TSC capture (skip >K=16 cycles on burst, 1.6 ms < 250 ms watchdog); stated normatively in `modules/scheduler.md`.
- **Cross-references:** A5-P11, A8-P4, A8-P5
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__modules__scheduler.md.diff.md`

### A5-P7 — `Timeout` on `IMemoryOps` async
- **Owner / co-owners:** A5
- **Severity:** medium
- **Hot-path impact:** none
- **Rule basis:** R1
- **Target doc(s):** `02-logical-view/04-memory.md`; `modules/memory.md`
- **Target section/anchor:** `IMemoryOps` method table (lines 102-103)
- **Final amended text (as of R3):** `wait(AsyncHandle) → Status` → `wait(AsyncHandle, Timeout) → Status`; `poll(AsyncHandle) → bool` → `poll(AsyncHandle, Timeout = Timeout::zero()) → bool`. Returns `TransportTimeout` on deadline. Dedup window + A5-P10 idempotency covers retry side-effects.
- **Cross-references:** A5-P10
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__modules__memory.md.diff.md`

### A5-P8 — Degradation specs + alert rules
- **Owner / co-owners:** A5, A8
- **Severity:** medium
- **Hot-path impact:** none (only fires on saturation)
- **Rule basis:** R4
- **Target doc(s):** `02-logical-view/02-scheduler.md §2.1.3.1.A`; `02-logical-view/03-worker.md §2.1.4.2`; `modules/scheduler.md`
- **Target section/anchor:** admission-pressure subsection; partial group availability subsection
- **Final amended text (as of R3):** `admission_pressure_policy ∈ {REJECT, COALESCE, DEFER}` with `max_deferred` bound to prevent memory exhaust. `partial_group_policy ∈ {FAIL_SUBMISSION, SHRINK_AND_RETRY, DEGRADED_CONTINUE}`. Externalized rules file parameterizes both; each value tied to an A8-P5 AlertRule.
- **Cross-references:** A8-P5, A5-P1
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__02-logical-view__02-scheduler.md.diff.md`

### A5-P9 — QUARANTINED Worker state (DRY-folded into UNAVAILABLE)
- **Owner / co-owners:** A5
- **Severity:** low
- **Hot-path impact:** none
- **Rule basis:** R3, R4
- **Target doc(s):** `02-logical-view/03-worker.md §2.1.4.1`; `modules/scheduler.md §3.2`
- **Target section/anchor:** Worker FSM
- **Final amended text (as of R3):** **R3 (A9 DRY amendment):** fold into `UNAVAILABLE + {Permanent, Quarantine(duration)}` policy variants rather than a new state. Transitions: `RECOVERING → UNAVAILABLE(Quarantine(worker_quarantine_ms))`; after probe success → `IDLE`; probe failure / `max_quarantine_cycles` → `UNAVAILABLE(Permanent)`. Config knobs `worker_quarantine_ms`, `worker_probe_retries`.
- **Cross-references:** A5-P11, A4-R3-P1
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__02-logical-view__03-worker.md.diff.md`

### A5-P10 — DS4 per-`REMOTE_*` idempotency
- **Owner / co-owners:** A5, A3
- **Severity:** medium
- **Hot-path impact:** none
- **Rule basis:** DS4, G3
- **Target doc(s):** `modules/distributed.md §3.3`; `04-process-view.md §3.4`
- **Target section/anchor:** Idempotent message handling
- **Final amended text (as of R3):** Every `REMOTE_*` handler MUST declare `idempotency ∈ {safe, at-most-once, at-least-once}` with link to dedup key. Doc-lint CI rule enforces annotation (attack vs. future `REMOTE_COLLECTIVE_FLUSH` moot).
- **Cross-references:** A3-P13 (per-peer-pair window), A5-P7, A8-P11
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__modules__distributed.md.diff.md`

### A5-P11 — Unified peer-health FSM (new R3)
- **Owner / co-owners:** A5, A10 (heartbeat), A6 (auth)
- **Severity:** medium
- **Hot-path impact:** none
- **Rule basis:** R3, R5
- **Target doc(s):** `modules/distributed.md §3.5` (new)
- **Target section/anchor:** new subsection
- **Final amended text (as of R3):** Single unified FSM with states `{HEALTHY, SUSPECT, OPEN, HALF_OPEN, UNAVAILABLE(Quarantine), LOST, AUTH_REVOKED}`. Co-owned across breaker (A5-P2), heartbeat (A10-P6), auth (A6-P2). Prevents oscillation under 6.2.2 replay (recovering network + credential churn).
- **Cross-references:** A5-P2, A5-P9, A5-P13, A6-P2, A10-P6
- **New ADR required?:** yes — ADR-018 "Single unified peer-health FSM"
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__modules__distributed.md.diff.md`

### A5-P12 — Name `ErrorCode::ResourceExhausted` for `BufferRef` staging alloc (new R3)
- **Owner / co-owners:** A5
- **Severity:** low
- **Hot-path impact:** none (slow path)
- **Rule basis:** G3
- **Target doc(s):** `modules/hal.md`; `modules/scheduler.md §5`
- **Target section/anchor:** HAL slow-path notes + error table
- **Final amended text (as of R3):** Close R2 Con 1 residual: staging `BufferRef` allocation failure on A1-P13's >1 KiB slow path surfaces `ErrorCode::ResourceExhausted`. One sentence in `modules/hal.md`; one row in scheduler §5.
- **Cross-references:** A1-P13
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__modules__hal.md.diff.md`

### A5-P13 — `breaker_auth_fail_weight` config field (new R3)
- **Owner / co-owners:** A5
- **Severity:** low
- **Hot-path impact:** none (compile-time-constant read on slow breaker path)
- **Rule basis:** R3
- **Target doc(s):** `modules/distributed.md` CB section + auth section cross-ref
- **Target section/anchor:** CircuitBreaker config
- **Final amended text (as of R3):** Add `breaker_auth_fail_weight` (default 10); `AuthenticationFailed` events count 10× in breaker accumulator so a flapping credential trips OPEN before normal `fail_threshold=5` would be exhausted by benign retries.
- **Cross-references:** A5-P2, A6-P2, A5-P11
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__modules__distributed.md.diff.md`

### A5-P14 — Q-record for collective partial-failure FailurePolicy mapping (new R3)
- **Owner / co-owners:** A5
- **Severity:** medium
- **Hot-path impact:** none
- **Rule basis:** R4
- **Target doc(s):** `09-open-questions.md`
- **Target section/anchor:** new Q16
- **Final amended text (as of R3):** Q-record: "Orchestration-composed collective partial-failure → FailurePolicy mapping" as v2 ADR trigger post A9-P3. Pre-empts a known R4 gap; follow-up ADR triggered, not pre-written.
- **Cross-references:** A9-P3
- **New ADR required?:** no
- **New open question required?:** yes — Q16
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__09-open-questions.md.diff.md`

### A6 — Security & Trust Boundaries (14)

### A6-P1 — Trust-boundary threat model completeness
- **Owner / co-owners:** A6
- **Severity:** high
- **Hot-path impact:** none
- **Rule basis:** S1, G6
- **Target doc(s):** `07-cross-cutting-concerns.md §7.1`
- **Target section/anchor:** STRIDE table
- **Final amended text (as of R3):** Extend STRIDE with six missing boundaries: (a) plug-in factory via `register_factory`, (b) REPLAY trace-file input, (c) Python-callable log/trace sinks, (d) multi-tenant Logical System isolation, (e) function-binary provenance, (f) per-service External-Data endpoints (DFS/KV/DSM/block storage).
- **Cross-references:** A6-P6, A6-P7, A6-P9, A6-P10, A6-P11, A6-P12
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__07-cross-cutting-concerns.md.diff.md`

### A6-P2 — Concrete node authentication in HANDSHAKE (mTLS cert-pinned v1; SPIFFE v2)
- **Owner / co-owners:** A6, A5
- **Severity:** high
- **Hot-path impact:** none (handshake-time only)
- **Rule basis:** S1, S2, S5
- **Target doc(s):** `modules/transport.md §2.3, §5.3`; `modules/distributed.md §2.4`; `07-cross-cutting-concerns.md §7.1.3`; `10-known-deviations.md`
- **Target section/anchor:** HANDSHAKE payload
- **Final amended text (as of R3):** `HandshakePayload { uint64_t node_id; bytes nonce; bytes signature_over_nonce_and_version; uint32_t credential_id; uint64_t coordinator_generation; }`. v1 = mTLS cert-pinned against `DeploymentConfig.peers[].public_key`; v2 = SPIFFE. Failure → `ErrorCode::AuthenticationFailed`. `ClusterView` gains `verified: bool`. **R3:** add `coordinator_generation` + `StaleCoordinatorClaim` reject rule to defeat malicious demoted-coordinator replay.
- **Cross-references:** A5-P11, A5-P13, A6-P4, ADR-020
- **New ADR required?:** yes — ADR-020 "Coordinator generation + stale-coordinator reject"
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__modules__transport.md.diff.md`

### A6-P3 — Bounded payload parsing (entry-gate guard)
- **Owner / co-owners:** A6, A1
- **Severity:** high
- **Hot-path impact:** extends (≤ 1 compare at entry; none on reject)
- **Rule basis:** S3, X2
- **Target doc(s):** `modules/transport.md §2.3, §8`
- **Target section/anchor:** payload structs; Configuration table
- **Final amended text (as of R3):** Single length-prefix guard per message. Config `max_task_count_per_remote_submit` (1024); `max_edge_count_per_remote_submit` (4096); `max_remote_error_message_bytes` (4 KiB); `max_tensor_meta_bytes` (16 KiB). Overflow → drop frame with `TransportCorrupted` before allocation; metric `corrupted_frames` increments.
- **Cross-references:** A2-P1, A6-P9
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__modules__transport.md.diff.md`

### A6-P4 — Default-encrypted TCP control; explicit RDMA exemption
- **Owner / co-owners:** A6
- **Severity:** high
- **Hot-path impact:** none (TCP only; RDMA untouched)
- **Rule basis:** S4, S5
- **Target doc(s):** `modules/runtime.md`; `modules/transport.md §8`; `07-cross-cutting-concerns.md §7.1.3`; `10-known-deviations.md`
- **Target section/anchor:** DeploymentConfig + init check
- **Final amended text (as of R3):** Add `require_encrypted_transport: bool = true`; multi-host plaintext TCP fails init with `ErrorCode::InsecureTransport`. RDMA uses RC + rkey scoping (A6-P5) — explicit exemption. Loopback unaffected. Deviation 3 reflects "enforced at init unless explicitly disabled".
- **Cross-references:** A6-P2, A6-P5, A6-P14
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__modules__runtime.md.diff.md`

### A6-P5 — Scoped, revocable RDMA `rkey`
- **Owner / co-owners:** A6
- **Severity:** medium
- **Hot-path impact:** none (retirement-only rotation)
- **Rule basis:** S2, S3
- **Target doc(s):** `modules/memory.md`; `modules/transport.md §2.3`; `modules/distributed.md §5.3`
- **Target section/anchor:** `IMemoryOps` API; `RemoteDataReadyPayload`
- **Final amended text (as of R3):** `register_peer_read(BufferRef, NodeId, Scope) → RkeyToken` + `deregister_peer_read(RkeyToken)`. `distributed_scheduler` deregisters on `SUBMISSION_RETIRED`. Replayed `REMOTE_DATA_READY` after retirement fails with `RkeyRevoked`.
- **Cross-references:** A5-P10, A6-P14
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__modules__memory.md.diff.md`

### A6-P6 — Security audit trail
- **Owner / co-owners:** A6
- **Severity:** medium
- **Hot-path impact:** none
- **Rule basis:** S6
- **Target doc(s):** `07-cross-cutting-concerns.md §7.1.4` (new); `modules/runtime.md §5`; `modules/profiling.md §3`
- **Target section/anchor:** new Audit Trail section
- **Final amended text (as of R3):** `AuditEvent { timestamp_ns; sequence_no; principal_id; action; subject; outcome; prev_hash; }` hash-chained per-boot. Pre-allocated audit ring; `IAuditSink` alongside `ILogSink`; fission trigger at N≥6 sinks. Emit at: node_join, factory_register, function_register, config_change, peer_lost, shutdown, sink_register, tenant_switch. 5 named event classes cover 6.1.2 replay.
- **Cross-references:** A8-P5, A8-P10
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__07-cross-cutting-concerns.md.diff.md`

### A6-P7 — Function-binary attestation (multi-tenant gated)
- **Owner / co-owners:** A6
- **Severity:** medium
- **Hot-path impact:** none (startup only)
- **Rule basis:** S1, S3
- **Target doc(s):** `modules/core.md §2.6`; `modules/runtime.md §2.4`; `07-cross-cutting-concerns.md §7.1.3`
- **Target section/anchor:** `FunctionDesc` + `DeploymentConfig`
- **Final amended text (as of R3):** `FunctionDesc.Attestation { key_id; signature_over_hash; }`. `DeploymentConfig.allow_unsigned_functions` defaults **false** in multi-tenant, **true** in single-tenant / SIM. Unsigned + `allow_unsigned=false` → `FunctionNotAttested`. Gated by `trust_boundary.multi_tenant`.
- **Cross-references:** A6-P1, A6-P14
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__modules__core.md.diff.md`

### A6-P8 — Boundary validation for DLPack / CAI imports (structural half absorbed into A1-P11; byte-cap A6-owned)
- **Owner / co-owners:** A6, A1, A3
- **Severity:** high
- **Hot-path impact:** none (absorbed into A1-P11)
- **Rule basis:** S3
- **Target doc(s):** `modules/bindings.md §2.4, §7`
- **Target section/anchor:** DLPack interop
- **Final amended text (as of R3):** A6-P8a (structural validation) absorbed into A1-P11. A6-P8b remains: `DeploymentConfig.max_import_bytes` (default 1 GiB); stride × shape overflow guarded by A1-P11 compare stack + this cap. Rejects negative/mismatched-device/overflowing tensors with `ValueError`.
- **Cross-references:** A1-P11
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__modules__bindings.md.diff.md`

### A6-P9 — Enforce Logical System isolation in wire + code
- **Owner / co-owners:** A6
- **Severity:** medium
- **Hot-path impact:** none (one compare on slow framing path; off hot 64-B line)
- **Rule basis:** S1, S2, S3
- **Target doc(s):** `modules/transport.md §2.3`; `modules/distributed.md §2.2`; `05-physical-view.md §5.5.2`
- **Target section/anchor:** `MessageHeader`; inbound dispatch
- **Final amended text (as of R3):** Add `uint32_t logical_system_id` to `MessageHeader` at offset placed outside 64-B hot cache line; bump header version. Protocol handler drops cross-tenant inbound → metric `cross_tenant_rejected` + `CrossTenantDenied`. `ISchedulerLayer::submit` checks caller tenant.
- **Cross-references:** A6-P1, A6-P3, A2-P1
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__modules__transport.md.diff.md`

### A6-P10 — Capability-scoped log / trace sinks
- **Owner / co-owners:** A6, A8
- **Severity:** low
- **Hot-path impact:** none
- **Rule basis:** S2
- **Target doc(s):** `modules/bindings.md §2.1`; `modules/profiling.md §2`; `07-cross-cutting-concerns.md §7.2.4`
- **Target section/anchor:** `add_sink` signature
- **Final amended text (as of R3):** `add_sink(callable, *, severity_min='INFO', domains=None, logical_system_id=None) → SinkHandle`. Dispatch filters events by capability. **R3:** caller-scoped default (depends on A8-P4 adopting caller scope); unscoped admin sinks must carry `diagnostic_admin_token`.
- **Cross-references:** A8-P4, A6-P9
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__modules__bindings.md.diff.md`

### A6-P11 — Gated `register_factory`
- **Owner / co-owners:** A6
- **Severity:** medium
- **Hot-path impact:** none (init-time)
- **Rule basis:** S2, S4
- **Target doc(s):** `modules/runtime.md §2.2`; `modules/bindings.md:51`
- **Target section/anchor:** `register_factory` signature
- **Final amended text (as of R3):** `register_factory(descriptor, *, registration_token)`. Token from `DeploymentConfig.factory_registration_token` set by deployer. Missing/wrong → `Unauthorized`. Post-`freeze()` → `RegistrationClosed`. Single guard function in `runtime/` emits audit event (A6-P6).
- **Cross-references:** A6-P6
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__modules__runtime.md.diff.md`

### A6-P12 — Signed / schema-validated REPLAY trace (v1 schema frozen; engine v2)
- **Owner / co-owners:** A6
- **Severity:** medium
- **Hot-path impact:** none
- **Rule basis:** S1, S3
- **Target doc(s):** `02-logical-view/10-platform.md §2.8.1`; `07-cross-cutting-concerns.md §7.2.6`; ADR-011-R2
- **Target section/anchor:** REPLAY mode description
- **Final amended text (as of R3):** **R3 (rescoped):** v1 ships **signed envelope + format frozen at ADR-011-R2 time**; no REPLAY engine in v1. Schema carries `{schema_version, capture_node_id, content_hash, detached_signature}`. REPLAY engine implementation deferred to v2 per A9-P6 option (iii).
- **Cross-references:** A9-P6, A2-P9, ADR-011-R2
- **New ADR required?:** no (amends existing ADR-011 → ADR-011-R2)
- **New open question required?:** yes — Q17 (REPLAY engine v2 triggers)
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__02-logical-view__10-platform.md.diff.md`

### A6-P13 — Per-tenant submit rate-limit (multi-tenant only)
- **Owner / co-owners:** A6
- **Severity:** medium
- **Hot-path impact:** none (gated behind multi-tenant; fast path = relaxed counter increment)
- **Rule basis:** S1
- **Target doc(s):** `modules/bindings.md §2.1`; `modules/scheduler.md` (admission path); `07-cross-cutting-concerns.md §7.1.2`
- **Target section/anchor:** submit admission
- **Final amended text (as of R3):** `DeploymentConfig.per_tenant_submit_qps: uint32 | None`. Leaky-bucket relaxed counter; on breach → `AdmissionDecision::REJECTED` with `TenantQuotaExceeded`. Single-tenant default = no limit. Partitioned counter so tenant-A flood does not affect tenant-B.
- **Cross-references:** A6-P9, A5-P8
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__modules__scheduler.md.diff.md`

### A6-P14 — Key-material lifecycle ADR
- **Owner / co-owners:** A6
- **Severity:** medium
- **Hot-path impact:** none
- **Rule basis:** G5, S5
- **Target doc(s):** `08-design-decisions.md`; `10-known-deviations.md §Deviation 3`; `modules/transport.md §12`
- **Target section/anchor:** new ADR + cross-links
- **Final amended text (as of R3):** New ADR: "Transport credentials (TLS certs, RDMA key material) externally provisioned via `DeploymentConfig`; runtime never generates or persists keys; rotation = restart-based in v1; multi-rotation without restart is a v2 extension."
- **Cross-references:** A6-P2, A6-P4
- **New ADR required?:** no (folds into ADR-011-R2 / ADR-020 companion note; captured in `10-known-deviations.md` update)
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__08-design-decisions.md.diff.md`

### A7 — Modularity & SOLID (9)

### A7-P1 — Break `scheduler/` ↔ `distributed/` cycle
- **Owner / co-owners:** A7, A2 (D6 citation), A6 (D6 citation)
- **Severity:** high
- **Hot-path impact:** none
- **Rule basis:** D6
- **Target doc(s):** `03-development-view.md §3.2`; `modules/distributed.md`; `modules/scheduler.md`
- **Target section/anchor:** `distributed/` Depended-on-by row
- **Final amended text (as of R3):** Strictly descending edges `bindings > runtime > scheduler > distributed > transport > hal > core`. Replace "`distributed/` depended on by `scheduler/` (for cross-node coordination), `runtime/`" with "`runtime/`" only. Remote notifications surface as `SchedulerEvent`s produced by `distributed/` and consumed by `scheduler/` via `core/` types. ADR note: `scheduler/` must not `#include` `distributed/`.
- **Cross-references:** A7-P5, A7-P4
- **New ADR required?:** no (clarifies existing ADR-008)
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__03-development-view.md.diff.md`

### A7-P2 — Split `ISchedulerLayer` into role interfaces (absorbs A9-P1)
- **Owner / co-owners:** A7, A9
- **Severity:** high
- **Hot-path impact:** none
- **Rule basis:** ISP, D4
- **Target doc(s):** `modules/core.md`; `02-logical-view/09-interfaces.md`
- **Target section/anchor:** `ISchedulerLayer` definition
- **Final amended text (as of R3):** Role interfaces `ISchedulerWiring`, `ISchedulerSubmit` (single `submit(SubmissionDescriptor)` — A9-P1 absorbed, overloads dropped), `ISchedulerCompletion`, `ISchedulerLifecycle`, `ISchedulerScope`. Aggregator `ISchedulerLayer : public Wiring, Submit, Completion, Lifecycle, Scope`. Implementation-side vtable layout unchanged; consumers include narrow headers. **R3:** atomic PR requirement — all role interfaces land together; ordered after A3-P2 (or co-commit).
- **Cross-references:** A3-P2, A7-P1, A9-P1 (absorbed)
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__02-logical-view__09-interfaces.md.diff.md`

### A7-P3 — Invert `core/` ↔ `hal/` for handle types
- **Owner / co-owners:** A7
- **Severity:** medium
- **Hot-path impact:** none
- **Rule basis:** D2
- **Target doc(s):** `modules/core.md`; `modules/hal.md`; `03-development-view.md`
- **Target section/anchor:** handle-type declarations
- **Final amended text (as of R3):** Move `using DeviceAddress = std::uint64_t;` and `using NodeId = std::uint64_t;` from `hal/` to `core/types.h`. HAL re-exports from its own `types.h` (includes `core/types.h`). `core::TaskHandle` compiles without hal.
- **Cross-references:** A7-P7
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__modules__core.md.diff.md`

### A7-P4 — Move distributed payload structs to `distributed/`
- **Owner / co-owners:** A7
- **Severity:** medium
- **Hot-path impact:** none
- **Rule basis:** D3, D5, SRP
- **Target doc(s):** `modules/transport.md §2.3`; `modules/distributed.md`; `02-logical-view/08-communication.md`
- **Target section/anchor:** payload struct definitions
- **Final amended text (as of R3):** Remove all `RemoteXxxPayload` / `HeartbeatPayload` from `transport/messages.h`; keep only `MessageHeader`, framing, `MessageType` tags. Move to `distributed/include/distributed/protocol_payloads.h`. Transport API: `send(peer, MessageType, span<const std::byte>, Timeout)`. **R3:** Invariant I-DIST-1 enforces `distributed/` header non-includable from `transport/` via IWYU-CI; Appendix-B ownership column updated.
- **Cross-references:** A7-P1, A2-P6, ADR-015
- **New ADR required?:** yes — ADR-015 "Distributed header registration policy (Invariant I-DIST-1)"
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__modules__transport.md.diff.md`

### A7-P5 — `distributed_scheduler` depends only on `ISchedulerLayer`
- **Owner / co-owners:** A7
- **Severity:** high
- **Hot-path impact:** none
- **Rule basis:** D2
- **Target doc(s):** `modules/distributed.md §1.3, §3.2, §3.3`; `08-design-decisions.md` ADR-008
- **Target section/anchor:** depends-on list
- **Final amended text (as of R3):** Replace `schedBase = scheduler/ TaskManager+WorkerManager+ResourceManager` with `ischedLayer = core/ ISchedulerLayer` (+ role interfaces from A7-P2). Shared machinery, if needed, lifts to `scheduler/core/` abstract base. `distributed_scheduler` stress-links only against `ISchedulerLayer.h`.
- **Cross-references:** A7-P1, A7-P2
- **New ADR required?:** no (ADR-008 enforcement)
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__modules__distributed.md.diff.md`

### A7-P6 — Extract MLR + deployment parser (`runtime::composition` sub-namespace)
- **Owner / co-owners:** A7
- **Severity:** medium
- **Hot-path impact:** none
- **Rule basis:** D3, D5
- **Target doc(s):** `modules/runtime.md`; `03-development-view.md §3.1`; `02-logical-view/05-machine-level-registry.md`
- **Target section/anchor:** module listing + dependency graph
- **Final amended text (as of R3):** Extract `MachineLevelRegistry`, `MachineLevelDescriptor`, `DeploymentConfig`, `deployment_parser` into `runtime::composition` sub-namespace with its own `include/`; forbid cross-includes via IWYU rule. **R3:** freeze sub-namespace via **ADR-014 (ADR-A7-R3)** with promotion triggers — promote to top-level `composition/` module if cross-consumer list grows ≥ 2 or deployment schema churn exceeds N/quarter.
- **Cross-references:** A2-P2, A2-P4
- **New ADR required?:** yes — ADR-014 (ADR-A7-R3) "Freeze `runtime::composition` sub-namespace + promotion triggers"
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__modules__runtime.md.diff.md`

### A7-P7 — Forward-decl contract
- **Owner / co-owners:** A7
- **Severity:** low
- **Hot-path impact:** none
- **Rule basis:** D2, D6 consistency
- **Target doc(s):** `modules/core.md §3.3`; `02-logical-view.md §2.1.1`; `03-development-view.md`
- **Target section/anchor:** wiring-interface section + DAG
- **Final amended text (as of R3):** `core/i_scheduler_layer.h` forward-declares `memory::IMemoryManager`, `memory::IMemoryOps`, `transport::IVerticalChannel`, `transport::IHorizontalChannel`; full definitions only included by implementers. Add interface-reference edges to logical DAG annotated "interface reference only".
- **Cross-references:** A7-P1, A7-P2
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__modules__core.md.diff.md`

### A7-P8 — Consolidate `ScopeHandle` ownership
- **Owner / co-owners:** A7
- **Severity:** medium
- **Hot-path impact:** none
- **Rule basis:** D7
- **Target doc(s):** `modules/core.md`; `modules/memory.md`; `appendix-a-glossary.md`
- **Target section/anchor:** `ScopeHandle` references
- **Final amended text (as of R3):** Declare `ScopeHandle` in `core/include/core/scope.h`; `memory/` consumes via `#include <core/scope.h>`. Glossary lists single canonical definition.
- **Cross-references:** A7-P3
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__modules__core.md.diff.md`

### A7-P9 — Dedupe Python `MemoryError` class names
- **Owner / co-owners:** A7
- **Severity:** low
- **Hot-path impact:** none
- **Rule basis:** D7
- **Target doc(s):** `modules/bindings.md §2.2`
- **Target section/anchor:** exception-mapping table
- **Final amended text (as of R3):** Rename `simpler.errors.MemoryError(SimplerError)` → `simpler.errors.SimplerMemoryAccessError(SimplerError)`; keep `simpler.errors.SimplerOutOfMemory(builtins.MemoryError)` distinct.
- **Cross-references:** A3-P10
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__modules__bindings.md.diff.md`

### A8 — Testability & Observability (12)

### A8-P1 — Injectable `IClock`
- **Owner / co-owners:** A8, A5
- **Severity:** medium
- **Hot-path impact:** none (release-build link-time specialization inlines to `CLOCK_MONOTONIC_RAW`)
- **Rule basis:** X5
- **Target doc(s):** `modules/hal.md`; `modules/profiling.md`; `modules/scheduler.md`; `modules/transport.md`; `modules/runtime.md`; `07-cross-cutting-concerns.md`
- **Target section/anchor:** `hal.md §2.8` (new `IClock`)
- **Final amended text (as of R3):** `IClock::now_ns(), monotonic_raw_ns(), steady_tick()` with `SystemClock`, `FakeClock`, `PerformanceSimClock` realizations. Threaded via `ProfilingConfig`, `worker_timeout_ms`, heartbeat, Timeouts. Link-time specialized so release bin identical to today.
- **Cross-references:** A8-P2, A8-P6, A8-P7
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__modules__hal.md.diff.md`

### A8-P2 — Driveable event-loop (single `IEventLoopDriver`)
- **Owner / co-owners:** A8, A9
- **Severity:** medium
- **Hot-path impact:** none (test-only; gated on `enable_test_driver`)
- **Rule basis:** X5, §8.2 DfT
- **Target doc(s):** `modules/scheduler.md §2.5, §5.2`; `modules/core.md §2`; `02-logical-view/02-scheduler.md`; `02-logical-view/09-interfaces.md`
- **Target section/anchor:** new `EventLoopRunner::step(max_events)` + `RecordedEventSource`
- **Final amended text (as of R3):** Single test seam `IEventLoopDriver` + `step()` + `RecordedEventSource` (ex-A9-P2 landing). Release binary unaffected. Serialized trace replayed produces bit-identical state transitions across deployments.
- **Cross-references:** A9-P2, A5-P5, A8-P1
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__modules__scheduler.md.diff.md`

### A8-P3 — Stats structs + latency histograms
- **Owner / co-owners:** A8, A1
- **Severity:** medium
- **Hot-path impact:** none (init-time pre-alloc; branchless bucket-increment < 5 ns)
- **Rule basis:** O3, O5
- **Target doc(s):** `modules/core.md §2.5`; `modules/scheduler.md §2.6, §4.3`; `modules/memory.md §2.5`; `modules/distributed.md §2.6`; `modules/runtime.md`; `modules/transport.md`; `07-cross-cutting-concerns.md §7.2.8`; `modules/profiling.md`
- **Target section/anchor:** stats struct definitions
- **Final amended text (as of R3):** Enumerate `LayerStats`, `TaskManagerStats`, `WorkerStats`, `MemoryStats`, `ChannelStats`, `DistributedStats`, `RuntimeStats` with concrete fields + units. Add `LatencyHistogram` (pre-allocated log-scale buckets; branchless insert; seqlock snapshot). **R3:** histogram bucket array pinned to CONTROL region in `modules/profiling.md` (cache-line-resident, no competition with admission-queue drain).
- **Cross-references:** A8-P5, A8-P9, A8-P4
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__modules__profiling.md.diff.md`

### A8-P4 — `dump_state()` diagnostic endpoint
- **Owner / co-owners:** A8, A5
- **Severity:** medium
- **Hot-path impact:** none (seqlock / relaxed-atomic snapshot reads)
- **Rule basis:** O4, X6, §8.3 DfD
- **Target doc(s):** `modules/runtime.md §2.1`; `modules/scheduler.md §2.5, §3`; `modules/memory.md`; `modules/distributed.md`; `modules/bindings.md §2.1`; `07-cross-cutting-concerns.md §7.2`
- **Target section/anchor:** new endpoint
- **Final amended text (as of R3):** `Runtime::dump_state(DumpOptions) → StateDump` returning structured JSON; `ISchedulerLayer::describe()`; `IMemoryManager::describe_state()`; `DistributedRuntime::describe_cluster_state()`; Python `simpler.Runtime.dump_state() -> dict`. **R3 (A6 amendment):** default scope = caller's `logical_system_id`; unscoped (cross-tenant) dump requires `diagnostic_admin_token`.
- **Cross-references:** A5-P6, A6-P10, A6-P6
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__modules__runtime.md.diff.md`

### A8-P5 — External alert-rules file; Prom/OTEL as opt-in deviation
- **Owner / co-owners:** A8, A5
- **Severity:** medium
- **Hot-path impact:** none (collector thread)
- **Rule basis:** O5, E3, X8
- **Target doc(s):** `modules/runtime.md`; `modules/bindings.md §2.1`; `modules/profiling.md`; `07-cross-cutting-concerns.md`
- **Target section/anchor:** `DeploymentConfig::alerting`; `on_alert` API
- **Final amended text (as of R3):** `AlertRule { metric, op, threshold, window_ms, severity, logical_system_id }` (4-field base + tenant scope). `Runtime::on_alert(callback)`. `PrometheusSink` / `OpenMetricsSink` emit `RuntimeStats` — recorded as deviation in `10-known-deviations.md`. Rules externalized (parameterize A5-P8 policies). **R3:** `AlertRule.logical_system_id` field added (tenant isolation).
- **Cross-references:** A5-P1, A5-P8, A6-P9
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__modules__runtime.md.diff.md`

### A8-P6 — Distributed trace time-alignment contract
- **Owner / co-owners:** A8, A3
- **Severity:** medium
- **Hot-path impact:** none
- **Rule basis:** O1
- **Target doc(s):** `07-cross-cutting-concerns.md §7.2.3`; `modules/profiling.md`; `modules/distributed.md §3.3`; `modules/hal.md §2.8`
- **Target section/anchor:** trace-alignment algorithm
- **Final amended text (as of R3):** PTP/NTP dependency declared; `skew_max_ns ≤ 100 µs` bounded; merge algorithm: primary sort by `(sequence, correlation_id, happens_before)` with skew-windowed reorder. **R3:** tie-breaker rule `min(node_id)` in youngest all-online epoch. Correlation-id chain dominates timestamp ties. `IClockSync::offset_ns()` added alongside A8-P1.
- **Cross-references:** A8-P1, A10-P3
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__07-cross-cutting-concerns.md.diff.md`

### A8-P7 — `IFaultInjector` sim-only seam
- **Owner / co-owners:** A8, A5
- **Severity:** medium
- **Hot-path impact:** none (sim-only symbol; never linked in onboard release)
- **Rule basis:** X5, DfT
- **Target doc(s):** `modules/hal.md`; `modules/transport.md`; `modules/distributed.md`; `modules/scheduler.md`; `06-scenario-view.md §6.2`
- **Target section/anchor:** `IFaultInjector` definition
- **Final amended text (as of R3):** `schedule_fault(FaultSpec)` compiled only under sim. Covers DMA-timeout, register-read-fault, RDMA-loss, heartbeat-miss, AICore-hang, slot-pool-exhausted; hooks into `hal/sim`, `transport/`, `distributed/` dedup. Drives A5-P5 matrix.
- **Cross-references:** A5-P5
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__modules__hal.md.diff.md`

### A8-P8 — AICore in-core trace upload protocol
- **Owner / co-owners:** A8
- **Severity:** medium
- **Hot-path impact:** none (L2 opt-in; L1 strips emits)
- **Rule basis:** O3, §4.8.5
- **Target doc(s):** `modules/profiling.md §3.1, §3.4`; `modules/hal.md §2.4`; `07-cross-cutting-concerns.md`; `04-process-view.md §4.8.5`
- **Target section/anchor:** AICore event ring + DMA uploader
- **Final amended text (as of R3):** Fixed-size AICore event ring + Chip-level DMA uploader at `sink_flush_interval_ms`. Capability bit `AICORE_TRACE_RING_SUPPORTED`. Default `PROFILING_LEVEL = L1_Coarse` compile-time strips Level-2 emits; L2 ring write < 50 ns amortized.
- **Cross-references:** A1-P7, A1-P10, A8-P12
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__modules__profiling.md.diff.md`

### A8-P9 — Profiling drop/degraded as first-class alerts
- **Owner / co-owners:** A8
- **Severity:** low
- **Hot-path impact:** none
- **Rule basis:** O3, O5
- **Target doc(s):** `modules/profiling.md §4.1, §7`; `07-cross-cutting-concerns.md §7.2.8`; `modules/runtime.md`
- **Target section/anchor:** ProfilingStats → RuntimeStats
- **Final amended text (as of R3):** Expose `TraceEvents_dropped`, `TraceSink_degraded_ms`, `LogEvents_dropped`. Alert rules: "Trace drop rate > 0 over 1 min → Warning"; "Sink degraded > 5 s → Warning".
- **Cross-references:** A8-P3, A8-P5
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__modules__profiling.md.diff.md`

### A8-P10 — Structured KV logging
- **Owner / co-owners:** A8, A6
- **Severity:** low
- **Hot-path impact:** none (severity-gate unchanged)
- **Rule basis:** O2
- **Target doc(s):** `modules/profiling.md §2.3`; `07-cross-cutting-concerns.md §7.2.4`
- **Target section/anchor:** `Logger::log_kv`
- **Final amended text (as of R3):** `Logger::log_kv(Severity, category, {kv...}, correlation_id)`; string-message shim retained for legacy sites. JSON-line sink default.
- **Cross-references:** A6-P6, A6-P10
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__modules__profiling.md.diff.md`

### A8-P11 — HAL contract test suite (sim + onboard)
- **Owner / co-owners:** A8
- **Severity:** medium
- **Hot-path impact:** none
- **Rule basis:** X5, DfT
- **Target doc(s):** `modules/hal.md §9`; `03-development-view.md §3.3.6`
- **Target section/anchor:** Contract Test Suite bullet
- **Final amended text (as of R3):** Single `.cpp` suite runnable against both `a2a3sim` and `a2a3` onboard CI. Header-independence lint (enforces A2-P6 / A7-P4 I-DIST-1). CI required check.
- **Cross-references:** A2-P5, A2-P6, A7-P4
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__modules__hal.md.diff.md`

### A8-P12 — Stable `PhaseId`s for Submission lifecycle
- **Owner / co-owners:** A8, A1
- **Severity:** low
- **Hot-path impact:** none (L2+ stripped at L1)
- **Rule basis:** O1
- **Target doc(s):** `modules/profiling.md §2.5`; `modules/scheduler.md §5.1, §5.3`
- **Target section/anchor:** `PhaseId` constants
- **Final amended text (as of R3):** `SUBMISSION_ADMIT`, `SUBMISSION_RETIRE`, `DATA_DEP_SCAN`, `BARRIER_ATTACH`, `NONE_ATTACH`, `WORKSPACE_ALLOC`, `ADMISSION_QUEUE_DRAIN`, `WORKER_DISPATCH`, `WORKER_COMPLETE`. Per-phase durations sum to admission latency within 1%.
- **Cross-references:** A1-P7, A8-P8
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__modules__profiling.md.diff.md`

### A9 — Simplicity (KISS/YAGNI) (8)

### A9-P1 — Drop `submit_group`/`submit_spmd` overloads (absorbed into A7-P2)
- **Owner / co-owners:** A9, A7
- **Severity:** medium
- **Hot-path impact:** none
- **Rule basis:** G2, G4
- **Target doc(s):** `02-logical-view/09-interfaces.md`; `02-logical-view/07-task-model.md`
- **Target section/anchor:** `ISchedulerLayer` method list lines 28-41
- **Final amended text (as of R3):** Only `submit(const SubmissionDescriptor&)` remains on the interface; free helpers `build_single/build_group/build_spmd` in `runtime/`. Absorbed into A7-P2 role-split (ISP).
- **Cross-references:** A7-P2 (absorbing proposal)
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__02-logical-view__09-interfaces.md.diff.md`

### A9-P2 — Collapse policy-enum / interface pluggability (landing: single `IEventLoopDriver` + closed enums)
- **Owner / co-owners:** A9
- **Severity:** high
- **Hot-path impact:** none (reduces 2 vcalls on Stage B)
- **Rule basis:** G2
- **Target doc(s):** `02-logical-view/02-scheduler.md`; `02-logical-view/09-interfaces.md`; `02-logical-view/05-machine-level-registry.md`; `08-design-decisions.md` ADR-010
- **Target section/anchor:** §2.1.3.5 / §2.1.3.6
- **Final amended text (as of R3):** Reduce deployment modes to closed `enum DeploymentMode { SINGLE_THREADED, SPLIT_COLLECTION }` (defer `SPLIT_DEFERRED`, `FULLY_SPLIT`). Replace `IEventCollectionPolicy` / `IExecutionPolicy` pluggability with closed `enum PolicyKind`. Keep **single `IEventLoopDriver` test-only seam + `step()` + `RecordedEventSource`** (satisfies X5). ADR-010 appendix lists **future-extension interfaces** (A2 OCP satisfied). Release-path unaffected.
- **Cross-references:** A8-P2, A9-P7, ADR-017
- **New ADR required?:** no (extends ADR-010; ADR-017 covers closed-enum-in-hot-path policy)
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__02-logical-view__02-scheduler.md.diff.md`

### A9-P3 — Remove collectives from `IHorizontalChannel` (Q4 option A) + ADR
- **Owner / co-owners:** A9
- **Severity:** high
- **Hot-path impact:** none
- **Rule basis:** G2
- **Target doc(s):** `02-logical-view/05-machine-level-registry.md`; `09-open-questions.md` Q4; `modules/transport.md`; `08-design-decisions.md`
- **Target section/anchor:** `IHorizontalChannel` definition lines 134-149
- **Final amended text (as of R3):** Remove `barrier()` / `all_reduce()`; keep only `send`/`recv`/`poll`/`transfer`. Q4 resolved to option A (composition at Orchestration level). Future `ICollectiveOps` sibling appendix-listed. HCCL native hook re-added only when concrete perf data arrives.
- **Cross-references:** A5-P14, Q4
- **New ADR required?:** no (resolves Q4)
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__02-logical-view__05-machine-level-registry.md.diff.md`

### A9-P4 — Drop `SubmissionDescriptor::Kind`
- **Owner / co-owners:** A9
- **Severity:** medium
- **Hot-path impact:** none
- **Rule basis:** G2, DRY
- **Target doc(s):** `02-logical-view/09-interfaces.md` lines 98-120; `02-logical-view/07-task-model.md`
- **Target section/anchor:** `SubmissionDescriptor` struct
- **Final amended text (as of R3):** Delete `enum Kind` and `kind` member. SPMD = `spmd.has_value()`; Single = `tasks.size()==1 && !spmd.has_value()`. **R3 edge:** `tasks.size()==1 && spmd.has_value() && spmd->size==1` (one-way SPMD) accepted in precondition catalog as a valid SPMD-with-size-1 submission. Closed-enum-in-hot-path policy (ADR-017) documents the removed `Kind`.
- **Cross-references:** A3-P7, A3-P9, ADR-017
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__02-logical-view__09-interfaces.md.diff.md`

### A9-P5 — Unify admission enums (`AdmissionStatus`)
- **Owner / co-owners:** A9
- **Severity:** medium
- **Hot-path impact:** none
- **Rule basis:** G2, DRY
- **Target doc(s):** `02-logical-view/09-interfaces.md` lines 242-297
- **Target section/anchor:** `IResourceAllocationPolicy::should_admit`; `ResourceAllocationResult::status`
- **Final amended text (as of R3):** Single `enum class AdmissionStatus { ADMIT, WAIT, REJECT(Exhaustion|Validation|Drain) }`; both APIs use it. Drop `SourceCollectionConfig` glossary ghosts (no prior entry). **R3:** update 6.2.3:123 scenario text — use `AdmissionStatus::REJECT(Exhaustion)` instead of `ResourceExhausted`.
- **Cross-references:** A3-P2, A3-P7, A3-P12
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__02-logical-view__09-interfaces.md.diff.md`

### A9-P6 — Defer PERFORMANCE/REPLAY simulation (option iii)
- **Owner / co-owners:** A9
- **Severity:** high
- **Hot-path impact:** none
- **Rule basis:** G2
- **Target doc(s):** `08-design-decisions.md` ADR-011 → ADR-011-R2; `02-logical-view/10-platform.md §2.8.1`; `07-cross-cutting-concerns.md §7.2`; `09-open-questions.md`
- **Target section/anchor:** ADR-011 Decision
- **Final amended text (as of R3):** **Option (iii):** v1 ships `FUNCTIONAL` only; `SimulationMode` enum stays **open**; REPLAY scaffolding (placeholder interfaces declared) **not factory-registered** in v1. ADR-011-R2 names triggers (Q17). Scaffolding setting `REPLAY` in v1 rejects explicitly (no silent fallback). A6-P12 rescoped to "signed schema + frozen format v1; REPLAY engine v2". Conditional re-vote triggers on A5/A7/A8 if Phase-5 text drifts.
- **Cross-references:** A6-P12, A2-P9, A3-P6, ADR-011-R2
- **New ADR required?:** no (amends ADR-011 → ADR-011-R2)
- **New open question required?:** yes — Q17 (REPLAY engine v2 concrete triggers + schema evolution)
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__08-design-decisions.md.diff.md`

### A9-P7 — Fold `SourceCollectionConfig` into `EventHandlingConfig`
- **Owner / co-owners:** A9, A4
- **Severity:** medium
- **Hot-path impact:** none
- **Rule basis:** G2, DRY
- **Target doc(s):** `02-logical-view/09-interfaces.md` lines 358-427; `02-logical-view/05-machine-level-registry.md:21`
- **Target section/anchor:** config struct definitions
- **Final amended text (as of R3):** Extend `EventHandlingConfig` with source list, per-source weight, max events per cycle; delete `SourceCollectionConfig`. Startup validation collapses to single pass. Merged struct cold-start only.
- **Cross-references:** A4-P5
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__02-logical-view__09-interfaces.md.diff.md`

### A9-P8 — Move AICore companion-artifacts obligation out of design
- **Owner / co-owners:** A9
- **Severity:** low
- **Hot-path impact:** none
- **Rule basis:** G2, SRP
- **Target doc(s):** `08-design-decisions.md` ADR-011 Negative #1 (line 487)
- **Target section/anchor:** ADR-011 Negative
- **Final amended text (as of R3):** Replace "Each AICore Function now has up to three companion artifacts…" with "Leaf `IExecutionEngine` implementations consume platform-specific artifacts defined in the compiler/kernel-tooling spec [link]." Cross-reference frontend design.
- **Cross-references:** ADR-011-R2
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__08-design-decisions.md.diff.md`

### A10 — Scalability & Data Flow (10)

### A10-P1 — Default `producer_index` sharding policy
- **Owner / co-owners:** A10, A1
- **Severity:** medium
- **Hot-path impact:** none (default `shards=1` preserves fast path)
- **Rule basis:** P4, DS6
- **Target doc(s):** `02-logical-view/02-scheduler.md`; `modules/scheduler.md`
- **Target section/anchor:** §"Concurrency and locking" (~line 121)
- **Final amended text (as of R3):** Shard count default = 8 at Host, 4 at Device, 1 at Chip and below. Each shard has own RW lock; admission takes shard lock keyed by `hash(BufferRef) mod N`. Default `shards=1` preserves today's single-thread fast path. **R3 (via A10-P7):** deployment cue in 05-physical-view.md (A10-P7) triggers opt-in when `concurrent_submitters × cluster_nodes ≥ 64`.
- **Cross-references:** A1-P1, A1-P2, A1-P14, A10-P7, ADR-019
- **New ADR required?:** yes — ADR-019 "Admission shard default + deployment cue" (co-driven with A10-P7)
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__02-logical-view__02-scheduler.md.diff.md`

### A10-P2 — v1 coordinator fail-fast (absorbed into A5-P3); v2 decentralize ADR
- **Owner / co-owners:** A10, A5
- **Severity:** high
- **Hot-path impact:** none
- **Rule basis:** P4, R5
- **Target doc(s):** `05-physical-view.md §5.4.0`; `modules/distributed.md` Coordinator Membership; `08-design-decisions.md`
- **Target section/anchor:** coordinator membership
- **Final amended text (as of R3):** v1 = fail-fast (absorbed into A5-P3 with **R3 scope-pin** to failed Pod only; `cluster_view` generation bump). v2 = decentralize via sticky routing + quorum `cluster_view` generation, recorded as roadmap-only ADR extension (not implemented in v1).
- **Cross-references:** A5-P3, A10-P3, A10-P6, A6-P2
- **New ADR required?:** no (roadmap extension of ADR-005)
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__modules__distributed.md.diff.md`

### A10-P3 — Per-data-element consistency model
- **Owner / co-owners:** A10
- **Severity:** medium
- **Hot-path impact:** none (documentation)
- **Rule basis:** DS6
- **Target doc(s):** `07-cross-cutting-concerns.md` (new subsection)
- **Target section/anchor:** "Data & State Reference" section (single, absorbs A10-P4/A10-P8)
- **Final amended text (as of R3):** Table `{data_element, owner_module, writers, readers, consistency, sync primitive}`. Rows: `producer_index`, `task_slot_pool.generation`, `completion_counters`. **R3:** add rows `cluster_view`, `group_availability`, `remote_proxy_pool`, `outstanding_submission_set`.
- **Cross-references:** A10-P4, A10-P8, A8-P6
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__07-cross-cutting-concerns.md.diff.md`

### A10-P4 — Stateful / stateless classification
- **Owner / co-owners:** A10
- **Severity:** medium
- **Hot-path impact:** none
- **Rule basis:** DS1
- **Target doc(s):** `03-development-view.md` or `02-logical-view.md`; `10-known-deviations.md`
- **Target section/anchor:** new "State Ownership & DS1" subsection (folded into "Data & State Reference")
- **Final amended text (as of R3):** Table `{module, stateful?, justification, externalization notes}`. Stateful entries cross-reference µs-path budget or correctness. **R2 amendment:** coordinator role explicitly classified.
- **Cross-references:** A10-P3, A10-P8
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__07-cross-cutting-concerns.md.diff.md`

### A10-P5 — Per-peer REMOTE_SUBMIT projection (absorbed into A1-P6)
- **Owner / co-owners:** A10, A1
- **Severity:** medium
- **Hot-path impact:** none (was extends)
- **Rule basis:** P6
- **Target doc(s):** `modules/distributed.md §5.1`
- **Target section/anchor:** partitioner projection
- **Final amended text (as of R3):** Partitioner emits per-peer sub-submission with only tasks/edges/boundaries touching that peer subset; shared `correlation_id` so `REMOTE_DEP_NOTIFY` still joins across peers. Absorbed into A1-P6 under same diff file.
- **Cross-references:** A1-P6 (absorbing proposal)
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__modules__transport.md.diff.md`

### A10-P6 — Faster peer-failure detection with hysteresis
- **Owner / co-owners:** A10
- **Severity:** medium
- **Hot-path impact:** none (dedicated heartbeat thread; sharded)
- **Rule basis:** P4, R5
- **Target doc(s):** `modules/distributed.md §config`; `modules/transport.md`
- **Target section/anchor:** heartbeat config
- **Final amended text (as of R3):** `heartbeat_miss_threshold_fast` (default 2), gated by transport-level TCP-keepalive / RDMA completion error; current threshold (6) remains slow-path baseline. Sub-second `NodeLost` under hard failure; 5% packet loss tolerance via hysteresis. **R3:** shard heartbeat thread per 32 peers, thread-pool cap = 4; cap-4 assumes ≤ 128 peers per node (Q18 revisit trigger).
- **Cross-references:** A5-P3, A5-P11, Q18
- **New ADR required?:** no
- **New open question required?:** yes — Q18 (heartbeat shard cap beyond 128 peers)
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__modules__distributed.md.diff.md`

### A10-P7 — Two-tier sharded TaskManager
- **Owner / co-owners:** A10
- **Severity:** medium
- **Hot-path impact:** none (default `shards=1`)
- **Rule basis:** P4
- **Target doc(s):** `02-logical-view/02-scheduler.md`; `modules/scheduler.md`; `05-physical-view.md`
- **Target section/anchor:** new "Sharded TaskManager (opt-in)"
- **Final amended text (as of R3):** Opt-in sharded path; shards admit independently; outstanding-submission window is global atomic counter so Layer-wide bound holds. Earliest-first completion bias preserved per-shard; cross-shard ordering via global `submission_id`. **R3:** deployment cue in `05-physical-view.md` — trigger threshold `concurrent_submitters × cluster_nodes ≥ 64`.
- **Cross-references:** A10-P1, A1-P2, ADR-019
- **New ADR required?:** yes — ADR-019 (shared with A10-P1)
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__05-physical-view.md.diff.md`

### A10-P8 — Single "Data & State Reference" section (absorbs P3+P4+P8)
- **Owner / co-owners:** A10
- **Severity:** medium
- **Hot-path impact:** none
- **Rule basis:** DS7, P6
- **Target doc(s):** `07-cross-cutting-concerns.md` (single new section)
- **Target section/anchor:** "Data & State Reference" — consistency table (P3) + stateful classification (P4) + ownership diagram (P8)
- **Final amended text (as of R3):** Mermaid flowchart + consistency table + stateful/stateless table in one page. A reader traces any `BufferRef` through producer → consumer → retirement from one page. Authored by A4 per cross-aspect resolution.
- **Cross-references:** A10-P3, A10-P4
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__07-cross-cutting-concerns.md.diff.md`

### A10-P9 — Gate `WorkStealing` × `RETRY_ELSEWHERE`
- **Owner / co-owners:** A10
- **Severity:** low
- **Hot-path impact:** none
- **Rule basis:** DS6, P4
- **Target doc(s):** `modules/distributed.md §3.1 + Error model`
- **Target section/anchor:** partitioner / failure policy cross
- **Final amended text (as of R3):** Declare `WorkStealingPartitioner + RETRY_ELSEWHERE` incompatible unless an assignment log `(submission_id, task_index) → final_node_id` is written before dispatch. Live-only log (works under A9-P6 option iii FUNCTIONAL mode; REPLAY not required).
- **Cross-references:** A5-P10, A9-P6
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__modules__distributed.md.diff.md`

### A10-P10 — `producer_index` cap/layout (absorbed into A1-P14)
- **Owner / co-owners:** A10, A1
- **Severity:** medium
- **Hot-path impact:** none
- **Rule basis:** P6, DS6
- **Target doc(s):** `02-logical-view/02-scheduler.md`; `modules/memory.md`
- **Target section/anchor:** sizing line
- **Final amended text (as of R3):** `producer_index.capacity ≥ max_outstanding_submissions × expected_out_per_submission × 2`; open-addressed with cache-line-sized buckets; single-probe average. Absorbed into A1-P14; runtime assertion on occupancy overflow.
- **Cross-references:** A1-P14 (absorbing proposal), A1-P1
- **New ADR required?:** no
- **New open question required?:** no
- **Planned diff file:** `applied-changes/docs__pypto-runtime-design__02-logical-view__02-scheduler.md.diff.md`

---

## 2. Rejected Proposals

**No proposals were rejected in this run.**

| proposal_id | owner | reason rejected | rule_conflict |
|-------------|-------|------------------|---------------|
| — | — | — | — |

---

## 3. Deferred Proposals

**No proposals were deferred in this run.** All 116 proposals converged as `agreed` with documented R2/R3 amendments.

| proposal_id | owner | severity | last_agree_fraction | blocking_objections | risk_if_ignored |
|-------------|-------|----------|---------------------|---------------------|-----------------|
| — | — | — | — | — | — |

---

## 4. Applied Changes Index

Flattened paths use double-underscore for `/`. One diff file per target doc; proposals that share a target doc land in a single diff.

| Target file | Proposal IDs that drove the edit | Diff file |
|-------------|-----------------------------------|-----------|
| `docs/pypto-runtime-design/02-logical-view/02-scheduler.md` | A1-P1, A1-P2, A1-P5, A1-P8, A1-P14, A3-P4, A3-P7, A3-P8, A3-P15, A5-P6, A5-P8, A9-P2, A10-P1, A10-P7, A10-P10 | [applied-changes/docs__pypto-runtime-design__02-logical-view__02-scheduler.md.diff.md](applied-changes/docs__pypto-runtime-design__02-logical-view__02-scheduler.md.diff.md) |
| `docs/pypto-runtime-design/02-logical-view/03-worker.md` | A1-P9, A5-P8, A5-P9 | [applied-changes/docs__pypto-runtime-design__02-logical-view__03-worker.md.diff.md](applied-changes/docs__pypto-runtime-design__02-logical-view__03-worker.md.diff.md) |
| `docs/pypto-runtime-design/02-logical-view/04-memory.md` | A5-P7 | [applied-changes/docs__pypto-runtime-design__02-logical-view__04-memory.md.diff.md](applied-changes/docs__pypto-runtime-design__02-logical-view__04-memory.md.diff.md) |
| `docs/pypto-runtime-design/02-logical-view/05-machine-level-registry.md` | A2-P2, A9-P2, A9-P3, A9-P7 | [applied-changes/docs__pypto-runtime-design__02-logical-view__05-machine-level-registry.md.diff.md](applied-changes/docs__pypto-runtime-design__02-logical-view__05-machine-level-registry.md.diff.md) |
| `docs/pypto-runtime-design/02-logical-view/06-function-types.md` | A3-P9 | [applied-changes/docs__pypto-runtime-design__02-logical-view__06-function-types.md.diff.md](applied-changes/docs__pypto-runtime-design__02-logical-view__06-function-types.md.diff.md) |
| `docs/pypto-runtime-design/02-logical-view/07-task-model.md` | A1-P4, A3-P1, A3-P9, A3-P11, A4-P2, A5-P4, A9-P4 | [applied-changes/docs__pypto-runtime-design__02-logical-view__07-task-model.md.diff.md](applied-changes/docs__pypto-runtime-design__02-logical-view__07-task-model.md.diff.md) |
| `docs/pypto-runtime-design/02-logical-view/09-interfaces.md` | A2-P1, A2-P6, A2-P7, A3-P2, A3-P7, A3-P12, A7-P2, A9-P1, A9-P2, A9-P4, A9-P5, A9-P7 | [applied-changes/docs__pypto-runtime-design__02-logical-view__09-interfaces.md.diff.md](applied-changes/docs__pypto-runtime-design__02-logical-view__09-interfaces.md.diff.md) |
| `docs/pypto-runtime-design/02-logical-view/10-platform.md` | A6-P12, A9-P6 | [applied-changes/docs__pypto-runtime-design__02-logical-view__10-platform.md.diff.md](applied-changes/docs__pypto-runtime-design__02-logical-view__10-platform.md.diff.md) |
| `docs/pypto-runtime-design/02-logical-view/12-dependency-model.md` | A4-P3 | [applied-changes/docs__pypto-runtime-design__02-logical-view__12-dependency-model.md.diff.md](applied-changes/docs__pypto-runtime-design__02-logical-view__12-dependency-model.md.diff.md) |
| `docs/pypto-runtime-design/03-development-view.md` | A2-P4, A7-P1, A8-P11, A10-P4 | [applied-changes/docs__pypto-runtime-design__03-development-view.md.diff.md](applied-changes/docs__pypto-runtime-design__03-development-view.md.diff.md) |
| `docs/pypto-runtime-design/04-process-view.md` | A1-P3, A1-P5, A1-P6, A1-P12, A1-P13, A3-P1, A3-P5, A3-P13, A3-P14, A4-P6, A5-P1, A8-P8 | [applied-changes/docs__pypto-runtime-design__04-process-view.md.diff.md](applied-changes/docs__pypto-runtime-design__04-process-view.md.diff.md) |
| `docs/pypto-runtime-design/05-physical-view.md` | A4-P6, A4-P7, A5-P1, A5-P3, A6-P9, A10-P2, A10-P7 | [applied-changes/docs__pypto-runtime-design__05-physical-view.md.diff.md](applied-changes/docs__pypto-runtime-design__05-physical-view.md.diff.md) |
| `docs/pypto-runtime-design/06-scenario-view.md` | A3-P3, A3-P5, A3-P6, A3-P10, A4-P6, A4-R3-P2, A8-P7 | [applied-changes/docs__pypto-runtime-design__06-scenario-view.md.diff.md](applied-changes/docs__pypto-runtime-design__06-scenario-view.md.diff.md) |
| `docs/pypto-runtime-design/07-cross-cutting-concerns.md` | A1-P10, A2-P1, A2-P9, A5-P5, A5-P6, A6-P1, A6-P2, A6-P4, A6-P6, A6-P7, A6-P9, A6-P10, A6-P13, A8-P6, A8-P9, A8-P10, A10-P3, A10-P4, A10-P8 | [applied-changes/docs__pypto-runtime-design__07-cross-cutting-concerns.md.diff.md](applied-changes/docs__pypto-runtime-design__07-cross-cutting-concerns.md.diff.md) |
| `docs/pypto-runtime-design/08-design-decisions.md` | A2-P3, A2-P5, A3-P2, A4-P6, A5-P3, A5-P11, A6-P14, A7-P5, A7-P6, A9-P2, A9-P3, A9-P6, A9-P8, A10-P1, A10-P2, A10-P7 | [applied-changes/docs__pypto-runtime-design__08-design-decisions.md.diff.md](applied-changes/docs__pypto-runtime-design__08-design-decisions.md.diff.md) |
| `docs/pypto-runtime-design/09-open-questions.md` | A2-P2, A2-P7, A2-P8, A3-P11, A4-P4, A5-P14, A9-P6, A10-P6 | [applied-changes/docs__pypto-runtime-design__09-open-questions.md.diff.md](applied-changes/docs__pypto-runtime-design__09-open-questions.md.diff.md) |
| `docs/pypto-runtime-design/10-known-deviations.md` | A2-P4, A2-P8, A5-P3, A6-P4, A6-P14, A8-P5 | [applied-changes/docs__pypto-runtime-design__10-known-deviations.md.diff.md](applied-changes/docs__pypto-runtime-design__10-known-deviations.md.diff.md) |
| `docs/pypto-runtime-design/appendix-a-glossary.md` | A4-P5, A4-P9, A4-R3-P1, A7-P8 | [applied-changes/docs__pypto-runtime-design__appendix-a-glossary.md.diff.md](applied-changes/docs__pypto-runtime-design__appendix-a-glossary.md.diff.md) |
| `docs/pypto-runtime-design/appendix-b-codebase-mapping.md` | A2-P4, A4-P8, A4-R3-P1, A7-P4 | [applied-changes/docs__pypto-runtime-design__appendix-b-codebase-mapping.md.diff.md](applied-changes/docs__pypto-runtime-design__appendix-b-codebase-mapping.md.diff.md) |
| `docs/pypto-runtime-design/appendix-c-compatibility.md` (new) | A2-P1, A2-P5 | [applied-changes/docs__pypto-runtime-design__appendix-c-compatibility.md.diff.md](applied-changes/docs__pypto-runtime-design__appendix-c-compatibility.md.diff.md) |
| `docs/pypto-runtime-design/modules/hal.md` | A1-P3, A1-P13, A4-P1, A5-P12, A7-P3, A8-P1, A8-P6, A8-P7, A8-P11 | [applied-changes/docs__pypto-runtime-design__modules__hal.md.diff.md](applied-changes/docs__pypto-runtime-design__modules__hal.md.diff.md) |
| `docs/pypto-runtime-design/modules/core.md` | A1-P4, A3-P1, A6-P7, A7-P3, A7-P7, A7-P8, A8-P3 | [applied-changes/docs__pypto-runtime-design__modules__core.md.diff.md](applied-changes/docs__pypto-runtime-design__modules__core.md.diff.md) |
| `docs/pypto-runtime-design/modules/scheduler.md` | A1-P1, A1-P2, A1-P8, A1-P14, A5-P6, A5-P8, A5-P9, A5-P12, A6-P13, A8-P2, A8-P3, A8-P12, A10-P1 | [applied-changes/docs__pypto-runtime-design__modules__scheduler.md.diff.md](applied-changes/docs__pypto-runtime-design__modules__scheduler.md.diff.md) |
| `docs/pypto-runtime-design/modules/memory.md` | A3-P2, A5-P7, A6-P5, A8-P3 | [applied-changes/docs__pypto-runtime-design__modules__memory.md.diff.md](applied-changes/docs__pypto-runtime-design__modules__memory.md.diff.md) |
| `docs/pypto-runtime-design/modules/transport.md` | A1-P3, A1-P6, A2-P1, A6-P2, A6-P3, A6-P4, A6-P9, A7-P4, A8-P3, A10-P5 | [applied-changes/docs__pypto-runtime-design__modules__transport.md.diff.md](applied-changes/docs__pypto-runtime-design__modules__transport.md.diff.md) |
| `docs/pypto-runtime-design/modules/distributed.md` | A1-P3, A1-P6, A2-P3, A2-P6, A5-P1, A5-P2, A5-P3, A5-P10, A5-P11, A5-P13, A6-P2, A7-P1, A7-P4, A7-P5, A8-P3, A8-P6, A8-P7, A10-P2, A10-P6, A10-P9 | [applied-changes/docs__pypto-runtime-design__modules__distributed.md.diff.md](applied-changes/docs__pypto-runtime-design__modules__distributed.md.diff.md) |
| `docs/pypto-runtime-design/modules/profiling.md` | A1-P7, A1-P10, A2-P9, A8-P3, A8-P5, A8-P6, A8-P8, A8-P9, A8-P10, A8-P12 | [applied-changes/docs__pypto-runtime-design__modules__profiling.md.diff.md](applied-changes/docs__pypto-runtime-design__modules__profiling.md.diff.md) |
| `docs/pypto-runtime-design/modules/error.md` | A2-P1, A3-P2, A3-P7, A3-P10, A3-P12, A5-P12, A7-P9 | [applied-changes/docs__pypto-runtime-design__modules__error.md.diff.md](applied-changes/docs__pypto-runtime-design__modules__error.md.diff.md) |
| `docs/pypto-runtime-design/modules/runtime.md` | A2-P2, A2-P3, A2-P5, A5-P1, A5-P2, A6-P4, A6-P7, A6-P11, A6-P13, A7-P6, A8-P1, A8-P4, A8-P5 | [applied-changes/docs__pypto-runtime-design__modules__runtime.md.diff.md](applied-changes/docs__pypto-runtime-design__modules__runtime.md.diff.md) |
| `docs/pypto-runtime-design/modules/bindings.md` | A1-P11, A3-P7, A3-P10, A4-P6, A5-P4, A6-P8, A6-P10, A6-P11, A7-P9, A8-P4, A8-P5, A8-P10 | [applied-changes/docs__pypto-runtime-design__modules__bindings.md.diff.md](applied-changes/docs__pypto-runtime-design__modules__bindings.md.diff.md) |

---

## 5. New ADRs

Existing last ADR in `08-design-decisions.md` is **ADR-013**. New ADRs number from 014. ADR-011-R2 is an amendment to the existing ADR-011 and keeps that number with `-R2` suffix.

| ADR ID | Title | Driven By | Location |
|--------|-------|-----------|----------|
| ADR-011-R2 (amendment) | SimulationMode option (iii): v1 FUNCTIONAL only; enum open; REPLAY scaffolding non-factory-registered; REPLAY engine deferral triggers | A9-P6, A6-P12 | `08-design-decisions.md` (amends existing ADR-011) |
| ADR-014 (ADR-A7-R3) | Freeze `runtime::composition` sub-namespace + promotion triggers | A7-P6 | `08-design-decisions.md` |
| ADR-015 | Distributed header registration policy (Invariant I-DIST-1: `distributed/` header non-includable from `transport/`; IWYU-CI enforced) | A7-P4, A2-P6 | `08-design-decisions.md` |
| ADR-016 | TaskState canonical spelling + transitions (`COMPLETING → ERROR` for Task; `FAILED` for Worker; optional `CANCELLED`) | A3-P1, A4-R3-P2 | `08-design-decisions.md` |
| ADR-017 | Closed-enum-in-hot-path policy (`DepMode` closed; A9-P4 `Kind` removal; rationale via benchmark-or-veto gate) | A2-P3, A9-P4 | `08-design-decisions.md` |
| ADR-018 | Single unified peer-health FSM (states `{HEALTHY, SUSPECT, OPEN, HALF_OPEN, UNAVAILABLE(Quarantine), LOST, AUTH_REVOKED}`; co-owned by breaker + heartbeat + auth) | A5-P11 | `08-design-decisions.md` |
| ADR-019 | Admission shard default + deployment cue (shards=1 default; opt-in at `concurrent_submitters × cluster_nodes ≥ 64`) | A10-P1, A10-P7 | `08-design-decisions.md` |
| ADR-020 | Coordinator generation + stale-coordinator reject (`coordinator_generation: uint64_t` in `HandshakePayload`; `StaleCoordinatorClaim` reject rule) | A6-P2 | `08-design-decisions.md` |

---

## 6. New Open Questions

Existing last Q in `09-open-questions.md` is **Q14**. New Qs number from Q15.

| Q ID | Title | Driven By | Location |
|------|-------|-----------|----------|
| Q15 | Async-policy extension seam with transport-capability axis (three axes: async-policy, transport-capability semantics, async-submit return path) | A2-P7 | `09-open-questions.md` |
| Q16 | Orchestration-composed collective partial-failure → FailurePolicy mapping (v2 ADR trigger post A9-P3) | A5-P14 | `09-open-questions.md` |
| Q17 | REPLAY engine v2 concrete triggers and schema evolution (envelope + frozen format landed v1; engine deferred) | A9-P6, A6-P12 | `09-open-questions.md` |
| Q18 | Heartbeat shard cap beyond 128 peers (cap-4 thread-pool assumption revisit when topology grows) | A10-P6 | `09-open-questions.md` |

---

## 7. Glossary Updates

| Term | Definition | Source proposal |
|------|------------|-----------------|
| `WorkerState` | Worker FSM states: `READY, BUSY, DRAINING, RETIRED, FAILED, UNAVAILABLE + {Permanent, Quarantine(duration)}` | A4-R3-P1 |
| `TaskState` (expanded) | `FREE → SUBMITTED → PENDING → DEP_READY → RESOURCE_READY → DISPATCHED → EXECUTING → COMPLETING → COMPLETED → RETIRED` (+ `ERROR`; optional `CANCELLED`) — see ADR-016 | A3-P1, A4-P9, A4-R3-P2 |
| `SimulationMode` | Open enum `{PERFORMANCE, FUNCTIONAL, REPLAY}`; v1 registers FUNCTIONAL only; REPLAY scaffolding declared but not factory-registered | A9-P6 |
| `IEventLoopDriver` | Single test-only seam (gated on `enable_test_driver`) used by `RecordedEventSource.step()` to drive scheduler event loops deterministically; release binary unaffected | A9-P2, A8-P2 |
| `REMOTE_BINARY_PUSH` | New `MessageType` staging a function binary to a peer before the first `REMOTE_SUBMIT` that needs it; gated by receiver `function_bloom` | A1-P6 |
| `coordinator_generation` | `uint64_t` field in `HandshakePayload`; monotonically bumped on failover; `StaleCoordinatorClaim` reject rule for mismatches | A6-P2 |
| `breaker_auth_fail_weight` | Configurable weight (default 10) applied to `AuthenticationFailed` events in circuit-breaker accumulator | A5-P13 |
| `AdmissionStatus` | Unified admission-outcome enum `{ADMIT, WAIT, REJECT(Exhaustion|Validation|Drain)}` replacing `AdmissionDecision` + `ResourceAllocationResult::Status` | A9-P5 |
| `PeerHealthState` | States of unified peer-health FSM `{HEALTHY, SUSPECT, OPEN, HALF_OPEN, UNAVAILABLE(Quarantine), LOST, AUTH_REVOKED}` | A5-P11 |
| `cluster_view` | Generation-stamped snapshot of participating-peer list; bumped on coordinator failover; consistency eventual with quorum on generation | A10-P3 |
| `group_availability` | Per-slot-type availability bitmask across WorkerGroups; strong consistency under per-Layer RW lock | A10-P3, A1-P9 |
| `IFaultInjector` | Sim-only interface declaring `schedule_fault(FaultSpec)`; never linked in onboard release | A8-P7 |
| `AlertRule` | `{metric, op, threshold, window_ms, severity, logical_system_id}` schema for externalized alerting | A8-P5 |
| `ScopeHandle` | Canonical scope handle declared in `core/include/core/scope.h`; consumed by `memory/` | A7-P8 |
| `RecordedEventSource` | `IEventSource` implementation replaying a serialized `EventTrace`; used with `IEventLoopDriver::step()` | A5-P5, A8-P2 |
| `logical_system_id` | `uint32_t` tenant id in `MessageHeader` (outside 64-B hot line); drives `CrossTenantDenied` and `AlertRule.logical_system_id` | A6-P9 |
| `remote_proxy_pool` | Runtime data element owned by `distributed_scheduler`; consistency table row in A10-P3; bounded by `SlotPoolExhausted` | A10-P3 |
| `outstanding_submission_set` | Global atomic counter preserving window bound under sharded TaskManager | A10-P3, A10-P7 |
| `descriptor_template_id` | `uint32_t` key into per-peer template registry; enables delta-encoded `TaskDescriptor[]` in `RemoteSubmitPayload` | A1-P6 |
| `function_bloom` | `uint64_t[4]` Bloom filter in `HeartbeatPayload` advertising each peer's Function Cache presence | A1-P3 |
| `RetryPolicy` | `{base_ms=50, max_ms=2000, jitter=0.3, max_retries=5}` for distributed retries | A5-P1 |
| `CircuitBreaker` | `{fail_threshold=5, cooldown_ms=10000, half_open_max_in_flight=1}` + states `{CLOSED, OPEN, HALF_OPEN}` per peer | A5-P2 |
| `RegistrationToken` | Deployer-provisioned token for `register_factory`; absent → `Unauthorized` | A6-P11 |
| `diagnostic_admin_token` | Token required for cross-tenant / unscoped `dump_state()` reads | A8-P4, A6-P10 |
| `EventLoopDeploymentConfig` | Struct binding Stage A/B/C to execution units; scope narrowed to closed `DeploymentMode` enum `{SINGLE_THREADED, SPLIT_COLLECTION}` | A4-P5, A9-P2 |
| `FE_VALIDATED` | Flag bit on `SubmissionDescriptor.flags` indicating frontend has pre-validated cycles/preconditions; release builds skip re-validation when set | A3-P7, A3-P8 |
| `AICORE_TRACE_RING_SUPPORTED` | HAL capability bit for in-core trace upload | A8-P8 |
| `DrainInProgress` | Core-domain `ErrorCode` returned when `submit()` is called after `drain()`; sticky flag | A3-P12 |
| `TaskSlotExhausted` / `ReadyQueueExhausted` | Diagnostic sub-codes surfaced by A3-P7 precondition catalog (routed from A1-P8) | A1-P8, A3-P7 |
| `SpmdIndexOOB` / `SpmdSizeMismatch` / `SpmdBlockDimZero` | SPMD precondition sub-codes added to A3-P7 catalog | A3-P7 |
| `coordinator_liveness_timeout_ms` | `< heartbeat_timeout_ms`; default `3 × heartbeat_interval_ms`; bounds coordinator-loss detection latency | A5-P3 |
| `admission_pressure_policy` | Enum `{REJECT, COALESCE, DEFER}` with `max_deferred` cap | A5-P8 |
| `partial_group_policy` | Enum `{FAIL_SUBMISSION, SHRINK_AND_RETRY, DEGRADED_CONTINUE}` for WorkerGroup member loss mid-Submission | A5-P8 |
| `Invariant I-DIST-1` | `distributed/` header non-includable from `transport/`; IWYU-CI enforced | A7-P4 |
| `runtime::composition` | Sub-namespace of `runtime/` owning MLR + deployment parser + layer factory; frozen per ADR-014 | A7-P6 |

---

## 8. Consistency Spot-Check (D7, V2, V5)

**Result: pass.**

- **Naming consistency across views (D7):** Unified by A4-P1 (HAL enum casing), A4-P7 (L0 label), A4-P9 (TaskState enumeration), A7-P8 (ScopeHandle ownership), A7-P9 (MemoryError dedup), A4-R3-P2 (terminal-failed spelling). Glossary (§7) adds 30+ new canonical terms covering every new type introduced across the 116 agreed proposals.
- **Logical↔Development↔Process↔Physical mapping (V2):** A2-P4 phase table + A10-P3/P4/P8 "Data & State Reference" section + A4-P6 view-to-ADR cross-links close the V2 loop. Appendix-B updates (A4-P8, A4-R3-P1, A7-P4) preserve codebase mapping.
- **Diagram name consistency (V5):** A4-P7 (Physical "AicoreDispatcher"), A3-P14 (COMPLETING skip-edge), A4-R3-P2 (scenario-view terminal-state), A3-P1 (Task ERROR state diagram) align diagrams with prose.

**Deferred consistency items (routed to `residual-risks.md`):**
- Re-vote triggers on A9-P6 (A5/A7/A8) if Phase-5 text drifts from option (iii).
- Re-vote triggers on A2-P7 (A3/A5/A9) if Phase-5 text reintroduces any v1 interface slot.
- A5's 6.2.2 livelock bound depends on A5-P11 landing as co-owned section.
- A6-P12 REPLAY engine deferred to v2 — Q17 triggers must be tracked.
- A10-P6 heartbeat cap-4 assumes ≤ 128 peers/node — Q18 revisit trigger.

No systemic drift was found; all three disputed proposals converged at `agree`, every R2-agreed proposal held under R3 scenario replay, and no blocking objections remain open.
