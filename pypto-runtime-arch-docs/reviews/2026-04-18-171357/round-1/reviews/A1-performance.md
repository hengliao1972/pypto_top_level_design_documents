# Aspect A1: Performance & Hot Path — Round 1

## Metadata

- **Reviewer:** A1
- **Round:** 1
- **Target:** `/data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design`
- **Runtime Mode:** yes
- **Timestamp (UTC):** 2026-04-18T17:18:54Z

## 1. Rubric Checks

| # | Check | Result | Evidence (file:line) | Rule(s) |
|---|-------|--------|----------------------|---------|
| 1 | Critical paths identified in Process View with latency budgets decomposed per stage | Weak | `04-process-view.md:637-701` (§4.8 budgets for 4 paths); `07-cross-cutting-concerns.md:187-196` (SLO table). Missing budgets for SPMD fan-out, BatchedExecutionPolicy tail, event-loop Stage B/C, and Chip-level scope-exit | X9, P5 |
| 2 | Hot path allocation-free and blocking-free | Weak | Good: `modules/scheduler.md:§10` ("no heap allocation on hot path"), `02-scheduler.md:30` (pre-allocated slots in CONTROL region), `modules/transport.md:§10` (pre-registered RDMA MRs), `modules/profiling.md:§10` ("no heap on hot path"). Gaps: `02-scheduler.md:91,115-121` (DATA-mode RW lock hold time **not bounded** when `scheduler_thread_count > 1`); `modules/scheduler.md:§4` (`OutstandingWindow`/`ReadyQueue` capacity and backing region unspecified); `02-scheduler.md:91` (`producer_index` hash map not pre-sized — first resize allocates on admission hot path) | X2, X3 |
| 3 | Hot data laid out for cache efficiency (contiguous, aligned, hot/cold separated) | Weak | Good: `modules/profiling.md:82-99` (`alignas(64) TraceEvent`), `modules/profiling.md:261-271` (`alignas(64) TlsRingBuffer` with cache-line-isolated tail), `modules/transport.md:247-258` (SPSC ring head/tail on separate cache lines), `02-scheduler.md:30` (atomic counters with `isolate_cache_line = true`), `02-logical-view/04-memory.md:39-58` (cache_line_bytes is a region-descriptor fact, not hardcoded — satisfies X7+X4). Gaps: `modules/core.md` and `02-logical-view/07-task-model.md` reference "hot/cold field separation" for `Task` but **do not enumerate** which fields belong to which cache line; no AoS-vs-SoA justification for the Task slot pool or the fan-in-counter array; `WorkerGroupAvailability` layout not documented despite O(G × S) evaluation per dispatch (`04-process-view.md:374-380`) | X4 |
| 4 | Data movement minimized on hot path | Weak | Good: `07-cross-cutting-concerns.md:180-186` (P6 rollup), `02-logical-view/08-communication.md:33` (Vertical = control-plane only; bulk via IMemoryOps), `modules/transport.md:§10` (zero-copy for pre-registered payloads), `modules/bindings.md:§10` (DLPack zero-copy). Gaps: `04-process-view.md:214-225` + `modules/transport.md:103-112` (REMOTE_SUBMIT inlines full `TaskDescriptor[]` + `IntraGroupEdge[]` + boundary bitsets + optional function binary; will exceed `max_message_bytes=65536` for large Groups and trigger `modules/transport.md:287-288` chunked reassembly copies); no Submission-descriptor dedup for repeating iteration submits | P6 |
| 5 | Caches justified with R/W ratio, TTL, invalidation | Weak | Good: `02-scheduler.md:93-99` (`producer_index` lifecycle explicit — insertion/lookup/eviction rules normative); `modules/memory.md:§5.2` (TensorMap fanout + scope-exit reclaim). Gaps: `04-process-view.md:207-225` (Function/Device Function Cache) — **no eviction policy, no TTL, no capacity bound, no invalidation for re-registration**, no config field for size in `modules/hal.md` or `modules/runtime.md`; `04-process-view.md:661` ("Pre-computed locality tables") — no invalidation on `on_cluster_change`; per-peer function-cache presence ("Check if Node₁ cache has content_hash") has no stated mechanism | P2 |
| 6 | Performance measured, not assumed (benchmarks + profiling hooks) | Pass (with gaps) | Good: `modules/profiling.md` (full trace/counter/annotation infrastructure; compile- and runtime-level gating; Chrome/Perfetto compatible); `04-process-view.md:696-701` (budget validation strategy explicitly `[ASSUMPTION]` pending benchmark); `modules/profiling.md:§9.1-9.3` (microbench `test_overhead` at L2); `modules/bindings.md:§9.5` (submit-overhead test). Gaps: no workload-level L1 overhead test for the `< 1% wall time` SLO (`07-cross-cutting-concerns.md:195`); no benchmark harness for DATA-mode lock contention under `scheduler_thread_count > 1`; no reproducible microbench for the 2 μs Chip→Core path; no perf-regression CI gate for any latency budget | P1, P3 |

## 2. Pros

- **Latency budgets decomposed per stage** for four critical paths (Host→AICore 15 μs, Pod→Remote 50 μs, AICore→Python 10 μs, Admission) — `04-process-view.md:637-701` satisfies X9/P5 at the architecture layer.
- **Initial budgets flagged `[ASSUMPTION]`** pending benchmarks — `04-process-view.md:701` demonstrates P1 honesty.
- **Control-plane region with ON_CHIP_SRAM backing at Chip level** drives sub-μs Chip→Core dispatch — `02-scheduler.md:30`; structurally satisfies X4 + X9 for the register-bank path.
- **Cache-line isolation driven by `MemoryRegionDescriptor` facts, not hardcoded constants** — `02-logical-view/04-memory.md:39-58`, `02-logical-view/08-communication.md:33`; satisfies X4 + X7 (portability).
- **Group Workspace collapses N non-boundary-tensor allocations to one bump/arena allocation** — `02-logical-view/07-task-model.md:92-114`, `modules/memory.md:§5.1`; direct win on X2 + P6.
- **Vertical channels are control-plane only; bulk data routed through `IMemoryOps`** — `modules/transport.md:236-238`; avoids copy ping-pong on the 2 μs Chip→Core path (P6).
- **RDMA MRs registered once at init; pre-allocated work-request pool per peer** — `modules/transport.md:§4.2,§10`; eliminates per-task registration cost (P6 + X2).
- **Profiling TLS ring buffer is per-thread SPSC with compile-time level strip** — `modules/profiling.md:§3.3,§10`; implements Rule NFR-1 "zero overhead when disabled" + X2/X3.
- **`producer_index` eviction rule is bounded by outstanding-window OUT/INOUT set** — `02-scheduler.md:93-99`; prevents unbounded hash growth (P2 + X2).
- **Earliest-first completion bias in default `FifoTaskSchedulePolicy`** prevents window head-of-line blocking — `02-scheduler.md:73-74`; preserves throughput target in P5.
- **SLO-driven alerts, not arbitrary thresholds** — `07-cross-cutting-concerns.md:133-147` aligned with O5/P5.
- **Non-Aliasing Intermediate-Memref Invariant is the structural enabler of the single-valued `producer_index`** — `02-logical-view/04-memory.md:68`, `modules/memory.md:§2.1`; keeps DATA-mode admission O(1) per lookup (X9 for §4.8.4 target).

## 3. Cons

- **`producer_index` hash map has no pre-sizing or growth policy specified** — `02-scheduler.md:91-99`, `modules/scheduler.md:§4`; first-time resize allocates on the admission hot path. Violates X2.
- **DATA-mode dep-metadata lock has no documented maximum hold time or contention model** — `02-scheduler.md:107-121`; Rule X3 explicitly requires "If a lock is unavoidable, document maximum hold time and contention analysis" — unmet when `scheduler_thread_count > 1`.
- **Function Cache has no eviction policy, TTL, capacity bound, or invalidation** — `04-process-view.md:207-225`; Rule P2 violated ("Define TTL and invalidation for every cache").
- **Task structure hot/cold split is referenced but not enumerated** — `02-logical-view/07-task-model.md:§2.4.1`, `modules/core.md` (Task structure mentioned as cache-line-aligned but no field-by-field layout). X4 requires explicit separation + AoS/SoA justification.
- **`OutstandingWindow`, `ReadyQueue`, and admission queue have no capacity bound, no placement hint, and no cache-line layout** — `modules/scheduler.md:§4`; possible allocation on admission or retirement (X2) and unjustified layout (X4).
- **SPMD fan-out dispatch (up to 72 AICore on a2a3, 108 on a5) has no per-stage latency budget** — called out as hot path in `07-cross-cutting-concerns.md:175,201`; no budget violates X9.
- **Event-loop Stage B (classify & inline) and Stage C (deferred drain) have no per-stage ns target** — `02-scheduler.md:§2.1.3.5` and `04-process-view.md:§4.1.4`; drains are on every dispatch (X9).
- **`BatchedExecutionPolicy` at Chip level has no tail-latency budget** — `04-process-view.md:95-101`; batch of N silently inflates last-task p99 by factor of N (X9 + P5).
- **REMOTE_SUBMIT payload is unbounded**: inlines full `TaskDescriptor[]` + `IntraGroupEdge[]` + boundary bitsets + optional `WorkspaceRequest`, and per `04-process-view.md:222-225` may include a full function binary on first miss — `modules/transport.md:103-112`. Above `max_message_bytes=65536` (`modules/transport.md:395`) this triggers chunked reassembly with copies (`modules/transport.md:287-288`), extending the 5 μs network-transit budget in §4.8.2 (P6).
- **Global atomic sequence counter in profiling** — `modules/profiling.md:277-286`; `fetch_add` on every `Phase` emit under multi-threaded load becomes a cross-socket cache-line ping-pong that can dominate the < 100 ns per-Phase target (X3 + P1).
- **WorkerGroup availability evaluated O(G × S) per `select_workers`** — `04-process-view.md:374-380`; on a2a3 with 24 groups × 2 slot types this is 48 integer compares inside the 2 μs Chip→Core budget (X4 + X9).
- **Argument marshaling at Python↔C boundary has no per-arg budget** — `modules/bindings.md:§5.2,§9.5`; §4.8.1 allocates 0.5 μs to "Python → C API boundary" but the stage is not decomposed per argument type (X9).
- **No profiling-overhead test at workload scale for the Level-1 ≤ 1% SLO** — `07-cross-cutting-concerns.md:195`, `modules/profiling.md:§9.1` only covers microbench at L2 (P1).
- **`IExecutionEngine::submit_kernel` `args_blob` memory lifecycle is implicit** — `modules/hal.md:277-288`; "layout owned by scheduler; HAL copies to engine queue" implies a copy per dispatch; not quantified in the 5 μs `DEP_READY → DISPATCHED` stage (P6 + X9).
- **RW lock eviction on `SUBMISSION_RETIRED` is exclusive** — `02-scheduler.md:112-121`; under bursty retirements can stall concurrent admissions without a quantified bound.

## 4. Proposals

| id | severity | summary | affected_docs | hot_path_impact | trade_offs | sanity_test |
|----|----------|---------|---------------|-----------------|------------|-------------|
| A1-P1 | high | Pre-size `producer_index` and forbid rehash on hot path | `02-logical-view/02-scheduler.md`, `modules/scheduler.md` | allocates (before) → none (after) | Fixed memory for unused capacity vs. zero-alloc admission | Admission storm; assert zero `alloc_buffer` from scheduler thread |
| A1-P2 | high | Specify max DATA-mode lock hold time; make shard count a first-class `LevelParam` | `02-logical-view/02-scheduler.md`, `modules/scheduler.md` | blocks (bounded after) | Extra tunable + per-shard stats vs. verifiable X3 bound | Vary `scheduler_thread_count`; measure p99 admission lock hold via phase traces |
| A1-P3 | medium | Define Function Cache LRU + capacity + per-peer presence in HEARTBEAT | `04-process-view.md`, `modules/hal.md`, `modules/distributed.md` | none | One config knob vs. bounded memory + explicit invalidation | Register > capacity distinct functions; observe LRU eviction in stats |
| A1-P4 | medium | Enumerate `Task` hot-64B / cold-tail fields and justify AoS | `02-logical-view/07-task-model.md`, `modules/core.md` | relayouts (one-time; net positive) | Slot footprint fixed at ≥ 2 cache lines vs. measurable L1 hit rate | perf-stat L1-dcache-miss on `on_dep_ready`+`on_dispatch` bench; ≥ 20% reduction |
| A1-P5 | high | Add latency budgets for SPMD fan-out and event-loop stages | `04-process-view.md`, `02-logical-view/02-scheduler.md` | none | More budgets to defend vs. full X9 coverage | Launch 72-way SPMD + 1000 empty-task burst; per-stage phase traces meet budget at p95 |
| A1-P6 | medium | Cap REMOTE_SUBMIT size; route function binaries on a staging channel; add descriptor template dedup | `modules/transport.md`, `04-process-view.md`, `modules/distributed.md` | extends-latency (before) → none (after) | Extra message type + template registry vs. 1–5 μs RDMA floor in §4.8.2 | 1000 identical remote submits; bytes/wire < 1 KB after dedup |
| A1-P7 | medium | Replace global atomic sequence with per-thread local + sink-side merge | `modules/profiling.md` | extends-latency (before) → none (after) | More complex sink merge vs. no cross-socket ping-pong | 8-thread 10 M-emits/s bench; p99 per-Phase < 100 ns after fix |
| A1-P8 | medium | Pre-size `OutstandingWindow`, `ReadyQueue`, admission queue; place in CONTROL region with cache-line isolation | `modules/scheduler.md`, `02-logical-view/02-scheduler.md` | allocates (before) → none (after) | Tunable ceiling + `WouldBlock` overflow path vs. zero-alloc admission | 4× window back-pressure test; no alloc events from scheduler thread |
| A1-P9 | low | Bitmask-encode WorkerGroup availability per slot-type (O(slot_types) selection) | `02-logical-view/03-worker.md`, `modules/scheduler.md` | relayouts (net positive) | Caps G ≤ 64 per slot-type bitmap (split otherwise) vs. 3–5× faster `select_workers` | Microbench 24-group `select_workers` before/after; full Chip→Core p95 ≤ 2 μs |
| A1-P10 | low | Add workload-level profiling-overhead CI gate for L1 ≤ 1% and L2 ≤ 5% | `modules/profiling.md`, `07-cross-cutting-concerns.md` | none | +~30 s CI vs. enforcement of the stated SLO | Test fails when `PROFILING_LEVEL=L2` with undersized disk buffer |
| A1-P11 | low | Decompose Python→C boundary budget per argument type and cap fan-in | `modules/bindings.md` | extends-latency (implicit bound before) → none | Exposes arg-count as a tuning dimension vs. predictable submit cost | Submit with {1,4,8,16} tensor args; boundary Phase stays within compound of §4.8.1 |
| A1-P12 | medium | Specify `BatchedExecutionPolicy.max_batch` and a dedicated tail-latency budget | `04-process-view.md`, `02-logical-view/02-scheduler.md` | extends-latency | Users pick throughput vs. latency vs. tail visible in SLOs | 1000-task burst; p99 Chip→Core ≤ `max_batch × 2 μs` at Batched; ≤ 2 μs at Dedicated |
| A1-P13 | medium | Quantify `args_blob` copy cost inside the 5 μs `DEP_READY → DISPATCHED` stage or pass by pre-registered ring slot | `modules/hal.md`, `04-process-view.md` | extends-latency | HAL contract tighter vs. bounded copy cost | Vary `args_size` ∈ {64, 256, 1024, 4096}; assert stage ≤ 5 μs at all sizes |
| A1-P14 | medium | Document `producer_index` placement (CONTROL region) and sharding default | `02-logical-view/02-scheduler.md`, `modules/scheduler.md` | none (doc) | Doc debt vs. X4 evidence | Review diff lists region + shard default explicitly |

### Proposal detail

#### A1-P1: Pre-size `producer_index` and forbid rehash on hot path

- **Rationale:** Rule X2 ("No allocation on the hot path"). `producer_index` is consulted on every DATA-mode admission (`02-scheduler.md:88-99`), and admission sits on the `submit()` hot path per `04-process-view.md:681-694`. An unspecified-capacity hash map will reallocate on first growth — a latent allocator spike inside the 100 ns amortized admission target.
- **Edit sketch:**
  - File: `docs/pypto-runtime-design/02-logical-view/02-scheduler.md`
  - Location: §2.1.3.1.B after the "Producer index" paragraph (around line 91)
  - Delta: Add "**Pre-sizing (normative).** `producer_index` is allocated at Layer init with `capacity = max_outstanding_submissions × expected_outputs_per_submission × 2`, placed in the control-plane region with `PlacementHint{isolate_cache_line=true}`, and runs without rehash. Overflow uses bounded open-addressed probing; exhausting the probe limit returns `ErrorCode::ResourceExhausted` to admission and triggers back-pressure via `IResourceAllocationPolicy.should_admit`." Mirror config field (`producer_index_capacity`) in `modules/scheduler.md §8`.
- **Trade-offs:**
  - Gain: zero-alloc DATA-mode admission, predictable 95p admission latency under burst.
  - Give up: fixed control-region footprint even for under-used layers; need a sensible default per level.
- **Sanity test:** Run an admission storm of 10× `max_outstanding_submissions` Group Submissions under DATA mode with `scheduler_thread_count = 4`. Count `alloc_buffer` / `malloc` events attributed to the scheduler thread via `test_overhead.cpp` — expected 0.

#### A1-P2: Bound DATA-mode lock hold time and promote shard count to `LevelParams`

- **Rationale:** Rule X3 (documented max hold time + contention analysis required when a lock is unavoidable). `02-scheduler.md:107-121` identifies the RW lock but says it "degenerates to an uncontended acquire/release" only under single-threaded Stage B. For `scheduler_thread_count > 1` the scheme is specified but the hold time and contention model are not.
- **Edit sketch:**
  - File: `docs/pypto-runtime-design/02-logical-view/02-scheduler.md`
  - Location: §2.1.3.1.B end of the "Concurrency and locking" block (after line 121)
  - Delta: Add "**Lock hold-time budget.** In shared (reader) mode: one DATA admission holds the lock for at most `B × A` probes (≤ 500 ns for B=4, A=4). In exclusive (writer) mode: admission's external-edge install is bounded by `B × A` fan-in RMWs; eviction is bounded by `|S.OUT ∪ S.INOUT|` removals. Default shard count `producer_index_shards = max(1, scheduler_thread_count)`. With W=16, B=4, A=4, and 2 shards, expected p95 shared-mode hold < 1 μs and p99 writer hold < 3 μs."
- **Trade-offs:**
  - Gain: verifiable bound satisfying X3; parallel admission scales with `scheduler_thread_count` without requiring a redesign.
  - Give up: additional tunable; per-shard stats required for validation.
- **Sanity test:** For `scheduler_thread_count ∈ {1,2,4,8}` with mixed DATA/BARRIER/NONE submissions at 4:1:1 ratio, record admission-lock hold via a new phase `DEP_LOCK_HOLD`. Assert p99 (reader) ≤ 3 μs and p99 (writer) ≤ 10 μs.

#### A1-P3: Function Cache LRU + capacity + HEARTBEAT-published presence

- **Rationale:** Rule P2 ("Define TTL and invalidation for every cache"). `04-process-view.md:207-225` defines the Function Cache behavior but omits eviction, capacity, and how peer-cache presence is queried before `REMOTE_SUBMIT` inlines a binary.
- **Edit sketch:**
  - File: `docs/pypto-runtime-design/04-process-view.md` (§4.2.4) and `docs/pypto-runtime-design/modules/hal.md` (new Function Cache subsection or augment §8 Configuration)
  - Location: after the "On subsequent Tasks with same content_hash" line
  - Delta: Add "Function Cache is bounded by `function_cache_bytes` (default 64 MiB) with LRU eviction; evicted entries trigger re-transfer on next miss. Peer cache presence is published in `HeartbeatPayload` (extend `modules/transport.md §2.3` `HeartbeatPayload` with `uint64_t function_bloom[4]`); coordinator checks the Bloom filter before deciding to inline a binary in `REMOTE_SUBMIT`."
- **Trade-offs:**
  - Gain: bounded memory, predictable miss behavior, no round-trip query before remote submit.
  - Give up: Bloom-filter false positives force occasional re-transfer; one more wire-format bump.
- **Sanity test:** Register `N > capacity / avg_size` distinct functions; observe LRU evictions in `RuntimeStats`. Remote-submit for an evicted function; verify binary is re-transmitted and counter increments.

#### A1-P4: Enumerate Task hot/cold field split; justify AoS

- **Rationale:** Rule X4 ("Separate hot fields from cold fields. Justify AoS vs. SoA choices for performance-critical structures"). `02-logical-view/07-task-model.md:§2.4.1` says `Task` has "hot/cold field separation" but never enumerates which fields are hot. Without the layout, profiling targets (`< 1 μs submit → DEP_READY` in §4.8.1) cannot be structurally defended.
- **Edit sketch:**
  - File: `docs/pypto-runtime-design/02-logical-view/07-task-model.md` (§2.4.1) and `docs/pypto-runtime-design/modules/core.md` (Task structure)
  - Location: replace the single field table with "Hot (first 64 B)" and "Cold (remaining slot)" sub-tables.
  - Delta: Hot: `state`, `fan_in_counter`, `submission_id`, `exec_type_id`, `worker_id`, `dispatch_ts_ns`. Cold: `function_id`, `args_blob_ptr`, `parent_task_key`, `dbg_name`, trace flags. Justification: "AoS per-slot, first cache line hot; state-transition handlers touch the hot line only. SoA for `fan_in_counter` across the pool was benchmarked (see A1-P10 test plan) and rejected because ready-queue scans touch at most `max_outstanding_submissions × B` slots, amortized favorable for AoS with one cache-line prefetch per slot."
- **Trade-offs:**
  - Gain: measurable L1-miss reduction on the state-machine hot path.
  - Give up: slot footprint fixed at ≥ 2 cache lines; slight memory cost per idle slot.
- **Sanity test:** `perf stat -e L1-dcache-load-misses` before/after on a 100k-task empty-function bench; expect ≥ 20% reduction on `on_dep_ready` + `on_dispatch` handlers.

#### A1-P5: Latency budgets for SPMD fan-out and event-loop stages

- **Rationale:** Rule X9 (every critical path needs end-to-end target + per-stage budgets). `07-cross-cutting-concerns.md:175,201` lists SPMD fan-out and task state-machine transitions as hot paths but §4.8 only budgets single-task dispatch.
- **Edit sketch:**
  - File: `docs/pypto-runtime-design/04-process-view.md` (§4.8)
  - Location: insert §4.8.6 and §4.8.7 after §4.8.4
  - Delta: "**§4.8.6 SPMD Fan-out Dispatch (Chip → N×AICore, N ≤ 108).** End-to-end ≤ 5 μs: batch descriptor prep ≤ 1 μs; register-bank block write ≤ 2 μs; per-core ACK collection ≤ 2 μs. **§4.8.7 Scheduler Event Loop Iteration.** Stage 1 collect ≤ 50 ns/source; Stage 2 classify+inline ≤ 100 ns/event; Stage 3 deferred drain ≤ 200 ns/event; full idle iteration ≤ 300 ns."
- **Trade-offs:** Gain: closes X9 coverage; explicit acceptance criteria. Give up: more budgets to profile and defend.
- **Sanity test:** Submit a 72-way SPMD and a 1000-task empty-function burst; capture phase traces for the new stages; assert p95 within budget.

#### A1-P6: Cap REMOTE_SUBMIT size; stage binaries; dedup descriptor templates

- **Rationale:** Rule P6 (minimize data movement on critical paths). REMOTE_SUBMIT currently inlines full descriptors and may embed function binaries (`04-process-view.md:222-225`, `modules/transport.md:103-112`); above `max_message_bytes=65536` (`modules/transport.md:395`) chunked reassembly copies data, extending the 5 μs network-transit budget in §4.8.2.
- **Edit sketch:**
  - File: `docs/pypto-runtime-design/modules/transport.md` (§2.3, §4.3) and `docs/pypto-runtime-design/04-process-view.md` (§4.2.4)
  - Location: new `MessageType::REMOTE_BINARY_PUSH` and descriptor template id in `RemoteSubmitPayload`
  - Delta: "Binaries travel on a dedicated `REMOTE_BINARY_PUSH` before the first `REMOTE_SUBMIT` that needs them, gated by the receiver's `function_bloom` (A1-P3). `RemoteSubmitPayload` gains `uint32_t descriptor_template_id` and a delta-encoded `TaskDescriptor[]`; templates are registered on first use and cached per peer."
- **Trade-offs:** Gain: REMOTE_SUBMIT fits in a single 64 KiB frame for realistic Groups; hits the 1–5 μs RDMA lower bound of §4.8.2. Give up: new message type + template registry state per peer.
- **Sanity test:** 1000 identical remote submissions of a 256-task Group; measure bytes-per-wire; expect < 1 KB amortized after dedup and zero chunk continuations.

#### A1-P7: Per-thread local sequence + offline merge

- **Rationale:** Rules X3 + P1. `modules/profiling.md:277-286` uses one `std::atomic<uint64_t>::fetch_add` per `Phase`. Under multi-threaded emit, the contended cache line dominates the < 100 ns per-Phase target (`modules/profiling.md:334,440`); the current microbench at `test_overhead.cpp` is single-threaded and does not expose this.
- **Edit sketch:**
  - File: `docs/pypto-runtime-design/modules/profiling.md` (§4.2, §5.1, §6)
  - Location: replace the global `SequenceAllocator`
  - Delta: Each `TlsRingBuffer` carries its own monotonic `local_seq` (plain `uint64_t` owned by the producer thread). Sink collector merges per-thread streams ordered by `(timestamp_ns, thread_id, local_seq)` at drain. Remove `SequenceAllocator::next`.
- **Trade-offs:** Gain: eliminates cross-socket cache-line ping-pong; Phase hot path drops back under 100 ns. Give up: slightly more complex merge logic in offline tools.
- **Sanity test:** Extend `test_overhead.cpp` to 8 threads × 10 M emits/s; p99 per-Phase ≤ 100 ns after fix (before: expected failure on multi-socket hosts).

#### A1-P8: Pre-size Outstanding Window, Ready Queue, and admission queue; CONTROL-region placement

- **Rationale:** Rules X2 + X4. `modules/scheduler.md:§4` names these data structures but omits capacity, backing region, and cache-line layout. Any growth triggers allocation on the retirement/admission path.
- **Edit sketch:**
  - File: `docs/pypto-runtime-design/modules/scheduler.md` (§4)
  - Location: add a "Capacity & placement" block to each data structure
  - Delta: "**ReadyQueue:** intrusive min-heap over Task slot indices with capacity `max_outstanding_submissions × expected_tasks_per_submission × 2`; CONTROL region, `isolate_cache_line = true`. **Admission queue:** bounded SPSC ring of `SubmissionDescriptor*` with capacity `max_admission_backlog` (default 64); overflow returns `WouldBlock` per `IResourceAllocationPolicy.should_admit`. **OutstandingWindow:** fixed-size array indexed by `submission_id % max_outstanding_submissions`."
- **Trade-offs:** Gain: zero-alloc admission and dispatch. Give up: explicit `WouldBlock` path surfaces in policies.
- **Sanity test:** Back-pressure test at 4× window; scheduler thread sees zero `alloc_buffer`; `WouldBlock` reflected in `LayerStats`.

#### A1-P9: Bitmask-encode per-slot-type WorkerGroup availability

- **Rationale:** Rule X4. `04-process-view.md:374-380` and `02-logical-view/03-worker.md:§2.1.4.2` give O(G × S) per dispatch. On a2a3 with 24 groups × 2 slot types, that is 48 integer comparisons inside the 2 μs Chip→Core stage in §4.8.1.
- **Edit sketch:**
  - File: `docs/pypto-runtime-design/02-logical-view/03-worker.md` (§2.1.4.2)
  - Location: add a bullet under "Group-level availability"
  - Delta: "Per-slot-type availability is encoded as `uint64_t` bitmasks across groups (one bitmask per Worker Type + `required_slot` count tier). `select_workers` ANDs required masks and scans low bit (`__builtin_ctzll`) for the first eligible group. Complexity O(slot_types)."
- **Trade-offs:** Gain: < 10 ns `select_workers`; keeps Chip→Core under 2 μs at p99. Give up: ≤ 64 groups per slot-type bitmap (splits cleanly if exceeded).
- **Sanity test:** Microbench `select_workers` against 24-group fixture before/after; expect ≥ 3× speedup; Chip→Core p95 ≤ 2 μs on the burst test of A1-P5.

#### A1-P10: Workload-level profiling-overhead CI gate

- **Rationale:** Rule P1. `07-cross-cutting-concerns.md:195` states "Profiling overhead (Level 1) < 1% wall time" but `modules/profiling.md:§9.1` only tests < 100 ns at L2 microbench.
- **Edit sketch:**
  - File: `docs/pypto-runtime-design/modules/profiling.md` (§9)
  - Location: add §9.2.4
  - Delta: "`bench_profiling_overhead.cpp` runs 10 s of 1 M task submissions at L0, L1, L2 on the sim target; CI asserts L1 ≤ 1.01× L0 wall time and L2 ≤ 1.05× L0. Regression blocks merge."
- **Trade-offs:** Gain: SLO enforced. Give up: ~30 s CI extension.
- **Sanity test:** Shrink `sink_max_queue_bytes` to force sink back-pressure; CI must fail — the test is the gate.

#### A1-P11: Per-arg marshaling budget at Python↔C boundary

- **Rationale:** Rule X9. `04-process-view.md §4.8.1` gives a flat 0.5 μs for Python→C API but makes no statement about scaling with argument count.
- **Edit sketch:**
  - File: `docs/pypto-runtime-design/modules/bindings.md` (§5.2, §9.5, §10)
  - Location: new subsection "Per-argument marshaling budget"
  - Delta: "Scalar arg ≤ 20 ns; DLPack tensor import (contiguous) ≤ 200 ns; `BufferRef` passthrough ≤ 30 ns. Total submit marshaling ≤ 500 ns for ≤ 8 args; above 8 args, §4.8.1 Python→C stage adds a per-arg term (`0.2 μs + 0.05 μs × extra_args`)."
- **Trade-offs:** Gain: predictable submit cost; documentable stage in §4.8.1. Give up: arg count becomes a visible tuning dimension.
- **Sanity test:** Submit kernels with {1, 4, 8, 16} tensor args; verify the marshaling phase trace + §4.8.1 per-arg term holds.

#### A1-P12: Specify `BatchedExecutionPolicy.max_batch` and tail-latency budget

- **Rationale:** Rules X9 + P5. `04-process-view.md:95-101` introduces `BatchedExecutionPolicy` at Chip level without bounding `N`; tail latency for the last task in a batch scales linearly.
- **Edit sketch:**
  - File: `docs/pypto-runtime-design/04-process-view.md` (§4.1.4, §4.8.1)
  - Location: add "Batch size bound" to the Deployment-Modes table and a note in §4.8.1
  - Delta: "`BatchedExecutionPolicy.max_batch` default = 8 at Chip level; tail latency for a batched task ≤ `max_batch × Chip→Core stage` (≤ 16 μs for max_batch=8). When BatchedExecutionPolicy is active, §4.8.1 uses this adjusted stage target."
- **Trade-offs:** Gain: throughput vs. latency choice is explicit. Give up: one more tunable; users must opt in to batching.
- **Sanity test:** 1000-task burst; p99 Chip→Core dispatch ≤ `max_batch × 2 μs` at Batched; ≤ 2 μs at Dedicated.

#### A1-P13: Bound `args_blob` copy cost inside DEP_READY → DISPATCHED budget

- **Rationale:** Rule P6 + X9. `modules/hal.md:277-288` says "HAL copies to engine queue", but the per-stage 5 μs budget in §4.8.1 does not price the copy against `args_size`. Large argument blobs can silently blow the budget.
- **Edit sketch:**
  - File: `docs/pypto-runtime-design/modules/hal.md` (§2.4, §10) and `docs/pypto-runtime-design/04-process-view.md` (§4.8.1)
  - Location: `KernelDispatch` contract and Notes column of §4.8.1
  - Delta: "`args_size` ≤ 1 KiB on the fast path; HAL copies through a pre-registered ring slot (not per-call allocation). For `args_size > 1 KiB`, scheduler stages args in a data-plane `BufferRef` and passes a handle; §4.8.1 per-stage budget extended by `0.5 μs/KiB` above threshold."
- **Trade-offs:** Gain: copy cost is bounded and visible. Give up: HAL has one more invariant to test; scheduler grows a staging path.
- **Sanity test:** Dispatch with `args_size ∈ {64, 256, 1024, 4096}`; stage latency stays within the adjusted budget at all sizes.

#### A1-P14: Document `producer_index` placement and sharding default

- **Rationale:** Rule X4 — layout must be explicit to be defended. `02-scheduler.md:91,115-121` leaves region placement and shard defaults implicit.
- **Edit sketch:**
  - File: `docs/pypto-runtime-design/02-logical-view/02-scheduler.md` and `docs/pypto-runtime-design/modules/scheduler.md`
  - Location: append to the Producer-index paragraph
  - Delta: "`producer_index` resides in the Layer's CONTROL region (CONTROL_AREA at Chip/Device, CONTROL_HEAP at Host). Default shard count `max(1, scheduler_thread_count)`. Rehash disabled (A1-P1)."
- **Trade-offs:** Gain: documented layout supports X4 evidence. Give up: doc churn only.
- **Sanity test:** Review diff mentions region + shard default; architecture audit confirms via `modules/scheduler.md §8` Configuration.

## 7. Cross-aspect tensions

| pair | with_proposal | preferred_resolution |
|------|---------------|----------------------|
| A1 vs A2 (Extensibility) | A1-P1 (pre-sized `producer_index`) | Parameterize capacity via `LevelParams`; keep `IMemoryManager` and scheduler interfaces stable. Expert-mode knob; default values ship sensible. |
| A1 vs A2 | A1-P6 (staged binaries + descriptor template) | Introduce as a new `MessageType` + optional field behind protocol-version flag so old peers continue to operate with inline binaries — preserves E2 backward-compat. |
| A1 vs A5 (Reliability) | A1-P2 (bounded DATA-mode lock hold) | Two-tier: fast path under uncontended event-loop (default single-threaded Stage B) keeps lock as no-op; reliability-driven retry/eviction runs in the slow path. R1/R2 remain enforced at transport; A1 bounds do not touch remote-call timeouts. |
| A1 vs A6 (Security) | A1-P11 (per-arg marshaling budget) | Two-tier at the C API: cheap structural validation (type + size + alignment) inline under ≤ 30 ns; deep semantic/policy validation deferred to slow-path gate called only on first registration. Security stays at boundary (S3), A1 keeps the hot path bounded. |
| A1 vs A8 (Observability) | A1-P7 (remove global sequence atomic) | Offline tools still reconstruct a total order from `(timestamp_ns, thread_id, local_seq)`; tracing fidelity preserved. Correlation id continues to flow end-to-end — no observability loss. |
| A1 vs A9 (Simplicity) | A1-P2 (shard-capable `producer_index`) | Default is single shard (simplest). Shard count only activates when `scheduler_thread_count > 1` — zero ceremony for the common single-node Sim case. |
| A1 vs A9 | A1-P6 (staged binaries + descriptor template) | Retain the inline-binary path as the simple fallback; template dedup only kicks in when the coordinator detects ≥ 3 identical consecutive submits. |

## 9. Status

- **Satisfied with current design?** partially — architecture's latency-budget discipline is good, but several core hot-path structures (`producer_index`, admission queue, SPMD fan-out, Function Cache) need the proposals above before the design can claim to structurally meet Rule X2 / X3 / X9.
- **Open items expected in next round:** A1-P1, A1-P2, A1-P4, A1-P5, A1-P6, A1-P7, A1-P8, A1-P12 — the blocking / high / medium-severity proposals that carry hot-path impact.
