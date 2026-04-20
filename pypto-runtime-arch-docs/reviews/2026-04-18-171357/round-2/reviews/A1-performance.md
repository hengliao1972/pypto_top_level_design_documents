# Aspect A1: Performance & Hot Path — Round 2

## Metadata

- **Reviewer:** A1
- **Round:** 2
- **Target:** `/data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/`
- **Runtime Mode:** yes (A1 has ×2 weight and hot-path veto)
- **Timestamp (UTC):** 2026-04-19T00:00:00Z

## 1. Rubric Checks (carry-over summary)

Rubric positions unchanged from Round 1 — see `round-1/reviews/A1-performance.md §1`. Round-1 findings still valid after reading all peer reviews; the open items are now the proposals revised in §5 below. Principal unresolved hot-path items: A1-P1 (producer_index pre-sizing), A1-P2 (DATA-mode lock bound), A1-P5 (SPMD fan-out + event-loop budgets), A1-P6 (REMOTE_SUBMIT hygiene), A1-P7 (profiling seq ping-pong), A1-P8 (window/ready-queue capacity), A1-P12 (batched tail), A1-P13 (args_blob copy).

Runtime-Mode check: every peer proposal is labelled with `hot_path_impact` in §6. Twelve peer items touch the hot path per synthesis audit; each has been re-evaluated against the two-tier requirement (see §6 blocking column and §7 tensions).

## 2. Pros (carry-over)

Unchanged from Round 1. Cross-aspect reinforcements observed:

- **A7-P1** (break `scheduler/` ↔ `distributed/` cycle) strengthens the hot-path's memory layout discipline — breaking the cycle removes a transitively-included header set that would otherwise inflate the `scheduler/` translation-unit compile closure and weaken A1-P4's cache-line claims.
- **A9-P1 + A7-P2** (single `submit()` + role-split) reduces the `ISchedulerLayer` vtable and removes two non-hot-path overloads, which A1 treats as a hot-path-neutral simplification that also shortens the hot-path instruction footprint.
- **A9-P4** (drop `SubmissionDescriptor::Kind`) removes one admission branch — a direct A1 win — and aligns with A1-P1's pre-sizing work.
- **A10-P10** (producer_index cap+layout) overlaps and strengthens A1-P1/A1-P14; see merge acceptance in §9.
- **A8-P2** (driveable event-loop + `RecordedEventSource`) is the primary mechanism that will let A1's SPMD fan-out and event-loop-stage budgets (A1-P5) be validated reproducibly in CI.

## 3. Cons (carry-over with new observations)

Round-1 cons stand. Two additional observations from cross-review:

- **A2-P1 version-field proposal is only hot-path-safe if A2's own two-tier stance holds** — version is validated at the Python↔C bindings boundary and at inter-node handshake, never on the per-submit in-process hot path. A1 accepts this provided A2 codifies it in `appendix-c-compatibility.md` with a `static_assert` at the C++ call site (no per-submit branch).
- **A6-P9 `logical_system_id` on `MessageHeader` is hot-path-safe only if placed outside the cache line that holds `correlation_id` / `source_node`** — otherwise the framing hot path suffers an extra miss. A1 conditions agreement on a specified header layout; see §7.

## 4. Proposals (new this round)

No net-new proposals this round. Round 2 focus is revision + voting. One clarifying sub-proposal is raised only if the merge register is rejected:

| id | severity | summary | affected_docs | hot_path_impact | trade_offs | sanity_test |
|----|----------|---------|---------------|-----------------|------------|-------------|
| A1-P15 | low (conditional — raised only if A10-P10 merge is rejected) | Keep A1-P14 and A10-P10 as two distinct entries with A1-P14 owning region/shard defaults and A10-P10 owning capacity/occupancy invariants | `02-logical-view/02-scheduler.md`, `modules/scheduler.md` | none | Gain: two narrowly-scoped proposals easier to review | Two PRs land side-by-side; neither duplicates the other's normative text |

A1-P15 exists only as a branch in case the synthesiser's merge (A1-P14 ← A10-P10) is rejected by A10. A1's preferred outcome is acceptance of the merge (see §9).

## 5. Revisions of own proposals

| id | action | amended_summary (if amend/split) | reason |
|----|--------|----------------------------------|--------|
| A1-P1 | defend | — | No peer raised a blocking objection. A10-P10 reinforces via cap+layout guidance (merged into A1-P14 below). Pre-sizing closes the admission rehash allocation (X2). |
| A1-P2 | amend | Default `producer_index_shards = 1` (A9-compatible single-thread fast path); when `scheduler_thread_count > 1`, `shards = max(1, scheduler_thread_count)` with default cap 8/4/1 at Host/Device/Chip+Core (A10-P1 alignment). Lock hold bounds stated normatively: reader p99 ≤ 3 µs, writer p99 ≤ 10 µs per shard. | Addresses A9 tension ("single shard default"), A10-P1 convergence ("shard count as `LevelParam`"), A5 tension (default single-thread keeps uncontended RW-lock fast path). |
| A1-P3 | defend | — | No blocking objection. The synthesis routed this to fast-track; peers did not dispute. |
| A1-P4 | amend | Move the normative hot/cold split into `modules/core.md §8` only; keep `02-logical-view/07-task-model.md` abstract ("Task has cache-line-aligned hot prefix; see `modules/core.md §8`"). Hot (first 64 B): `state, fan_in_counter, submission_id, exec_type_id, worker_id, dispatch_ts_ns`. Cold tail unchanged. AoS justified because ready-queue scans touch `≤ max_outstanding_submissions × B` slots; one cache-line prefetch per slot amortizes favorably. | Synthesiser observation: doc-only `relayouts` can be localized. Addresses A7 (module cohesion — modules/core.md owns layout) and A9 (not over-specified in Logical View). L1-dcache-miss benchmark target unchanged (≥ 20 % reduction on `on_dep_ready + on_dispatch`). |
| A1-P5 | defend | — | No blocking objection; A3 and A8 both support (A3 via Req↔Scenario traceability, A8-P12 via stable `PhaseId`s that make the new budgets measurable). |
| A1-P6 | amend + merge | Accept merge with A10-P5. Combined proposal title: **"Distributed payload hygiene."** Combined scope: (1) cap REMOTE_SUBMIT size; (2) route function binaries on a dedicated `REMOTE_BINARY_PUSH` gated by `HeartbeatPayload.function_bloom`; (3) descriptor-template dedup (`uint32_t descriptor_template_id`) with per-peer template cache; (4) per-peer projection of `RemoteSubmitPayload` to only the tasks/edges/boundaries touching that peer (A10-P5's contribution); (5) fast-path single-peer case (fan-out = 1) ships descriptor unchanged (A10 two-tier). | Synthesis merge register proposed this; A1 is co-owner; A10-P5 contributes the per-peer projection angle. Evidence target for round 2 compliance: 1000 identical remote submits → bytes-per-wire < 1 KB amortized; for 8-peer fan-out of a 10k-task Group → aggregate bytes < 1.5× full-payload (vs 8× today). |
| A1-P7 | defend | — | No blocking objection. A8-P12 (stable `PhaseId`s) and A8-P1 (`IClock`) do not add any cross-socket atomic that would re-introduce the ping-pong; offline merge at `(timestamp_ns, thread_id, local_seq)` remains the total-order contract. |
| A1-P8 | defend | — | Synthesiser marked this as low-friction; no peer objected. Admission-queue `WouldBlock` path already in `IResourceAllocationPolicy.should_admit`. |
| A1-P9 | amend | Spec the **2×64 bitmap fallback** for `G > 64` groups per slot-type: two `uint64_t` bitmasks + one branch; `select_workers` AND-s required masks across both words and scans with `__builtin_ctzll`. Above 128 groups: N/64 words — still O(slot_types × ⌈G/64⌉). Document the fallback in `02-logical-view/03-worker.md §2.1.4.2`. | Addresses A2-P3 (extensibility: the 64-group ceiling is not a hard limit) and A9 (premature-layout concern softened by showing the explicit fallback). |
| A1-P10 | defend | — | Synthesiser routed to fast-track; no peer objected. Ties in with A8-P9 (drop-rate alerts) and A5-P5 (chaos harness) naturally. |
| A1-P11 | amend + merge | Accept partial merge with A6-P8 (per synthesis register). Combined per-argument budget covers **both** latency (scalar ≤ 20 ns; DLPack contiguous ≤ 200 ns; `BufferRef` passthrough ≤ 30 ns) **and** validation (dtype whitelist, device-type match, shape/stride overflow, zero-length product, capsule ownership). Fast-path: structural validation inlined inside the same 200 ns budget; slow-path: policy/tenant checks deferred to first-registration (one-time). §4.8.1 Python→C stage unchanged on fast path. | Aligns with the synthesiser's "natural fusion" finding. Addresses A6's S3 boundary validation while keeping A1's per-arg budget intact. |
| A1-P12 | amend | Two-tier default: **fast path = `Dedicated` execution policy** (admission targets §4.8.1 2 µs Chip→Core unchanged, no batching); **slow path = `Batched` only when explicitly selected** (`execution_mode = Batched`). When `Batched` is active, `max_batch` defaults to 8 at Chip level; tail-latency budget for last task ≤ `max_batch × 2 µs` (= 16 µs at default). Document in §4.1.4 Deployment-Modes table with an explicit "default Dedicated" column. | Addresses A9 tension (tunable proliferation): no new knob on the default path. Addresses A3 (new scenario): canonical burst scenario with `Dedicated` needs no new scenario; `Batched` mode has its own scenario walkthrough. |
| A1-P13 | amend | HAL contract states: **`args_size ≤ 1 KiB`** is the fast path; HAL copies via a pre-registered ring slot from a CONTROL-region pool (`hal_args_ring_capacity` config, default 1024 slots). For `args_size > 1 KiB`, scheduler stages args into a data-plane `BufferRef` and passes a handle; §4.8.1 per-stage budget extended by `0.5 µs/KiB` above threshold. Add `args_size ≤ 4 KiB` as the absolute maximum per `KernelDispatch` — larger submissions are rejected at admission with `ArgsTooLarge`. | Addresses A7 (HAL contract tightening; one additional invariant, no widening beyond the existing `submit_kernel` surface). Addresses A5 (timeout interaction: staging path inherits the enclosing Worker's `worker_timeout_ms`). |
| A1-P14 | amend + merge | Accept merge with A10-P10. Combined text: `producer_index` lives in the Layer's CONTROL region (`CONTROL_AREA` at Chip/Device, `CONTROL_HEAP` at Host); default shards = `max(1, scheduler_thread_count)`; rehash disabled (A1-P1); capacity ≥ `max_outstanding_submissions × expected_out_per_submission × 2`; open-addressed with cache-line-sized buckets; single-probe average; `isolate_cache_line = true` per shard. | Synthesis merge register proposed this; A10-P10 is functionally the capacity/occupancy half of the same docs-only change A1-P14 proposed. |

## 6. Votes on peer proposals

Every row below carries an `hpi` label (per Runtime Design Mode requirement). A1 veto rule: `blocking = true` if `hpi ∈ {allocates, blocks, relayouts, extends-latency}` and the proposer has **not** documented a compensating structural fix or a two-tier (fast/slow) path. When the proposer documented an acceptable two-tier bridge in Round 1 (or one is trivially available), `blocking = false` with the bridge cited in `rationale`.

| proposal_id | vote | hpi (A1 tag) | rationale (cite rule id) | blocking | override_request |
|-------------|------|--------------|--------------------------|----------|-------------------|
| A2-P1 | agree | none | E6. Version fields validated only at bindings↔C boundary + inter-node handshake (A2 §7 two-tier); static_assert at C++ call site → zero runtime branch on hot path. | false | n/a |
| A2-P2 | agree | none | E4. Schema validation is init-time only; `LevelOverrides` never touched on hot path. | false | n/a |
| A2-P3 | agree | none | E4. A1 specifically requires `DepMode` stays closed (hot-path 3-arm jump table). Opened enums (`FailurePolicy`/`TransportBackend`/`SimulationMode`/`NodeRole`) resolved to function-pointer tables at init; no hot-path dispatch. | false | n/a |
| A2-P4 | agree | none | E5. Migration plan is pure doc; no runtime effect. | false | n/a |
| A2-P5 | agree | none | E2. BC policy doc; no runtime effect. | false | n/a |
| A2-P6 | agree | none | E4. Registry is init-only; handler lookup is a function-pointer table keyed by `MessageType` — no virtual dispatch, no allocation. A2's §7 fast-path short-circuit for top-N builtins satisfies A1's hot-path budget. Proposer must confirm the lookup compiles to an indexed load + indirect branch, not a virtual call. | false | n/a |
| A2-P7 | agree | none | E1. Reserved interface is documentation-only; sync fast path remains default. | false | n/a |
| A2-P8 | agree | none | Rule Exceptions procedure. Deviations bookkeeping; no runtime effect. | false | n/a |
| A2-P9 | agree | none | E6. Per-file header only; no per-event hot-path cost. | false | n/a |
| A3-P1 | agree | none | LSP, D7, V5. `ERROR`/`CANCELLED` state is already implicit in narrative; codifying it does not add hot-path branches (failure states are cold path). | false | n/a |
| A3-P2 | agree | none | LSP. A1 prefers option (b) `std::expected<SubmissionHandle, ErrorContext>` with a trivial success path (no error path allocation) over option (a) throw; exception throwing in `submit()` risks unwind cost on the cold path but is acceptable. | false | n/a |
| A3-P3 | agree | none | V4, R6. Scenario text; no runtime effect. | false | n/a |
| A3-P4 | agree | none | R1, DS3. `DEP_FAILED` event fires only on the error cold path; successor walk is O(successors), bounded by the producer's `fan_out`. No effect on success path. A1 suggests the implementation store successors as a contiguous small-vector inline in the Task slot (SoA side table for large fan-out) — doc-level only, not a prerequisite for agreement. | false | n/a |
| A3-P5 | agree | none | DS4, G3. Cancellation traversal is cold path; policy hook is policy-time, not dispatch-time. | false | n/a |
| A3-P6 | agree | none | G1, V3. Scenario doc; no runtime effect. | false | n/a |
| A3-P7 | agree | extends-latency | G3, S3. A3 provides two-tier: `SubmissionDescriptor.flags & FE_VALIDATED` bit lets release builds skip the validation when the frontend has pre-checked; debug always validates. Fuses naturally with A6-P8 (and therefore with A1-P11 per the merge register). | false | n/a |
| A3-P8 | agree | extends-latency | G3, X9. A3 provides two-tier: DFS is O(V+E); `intra_edges.empty()` fast path is a single size-zero branch; large groups (>1k tasks) elide via `FE_VALIDATED` flag in release. Admission-budget row updated with "Cyclic intra_edges check" line. | false | n/a |
| A3-P9 | agree | none | G3, LSP. Reserved scalar slots; compile-time, zero runtime cost. | false | n/a |
| A3-P10 | agree | none | G1, D7. Exception mapping doc; no hot path. | false | n/a |
| A3-P11 | agree | none | G3. `[ASSUMPTION]` marker; doc only. | false | n/a |
| A3-P12 | agree | none | G3, R4. One atomic flag read on admission — on the already-slow admission path, not on Chip→Core. Cost dominated by existing admission work. | false | n/a |
| A3-P13 | agree | none | DS3, G3. Reorder buffer lives on the slow path; hot path unaffected. | false | n/a |
| A3-P14 | agree | none | LSP, V5. Diagram edge only. | false | n/a |
| A3-P15 | agree | none | G3, X6. Debug-only; release build strips. | false | n/a |
| A4-P1 | agree | none | D7, V5. Enum casing fix; no runtime effect. | false | n/a |
| A4-P2 | agree | none | D7, V5. Anchor fix; no runtime effect. | false | n/a |
| A4-P3 | agree | none | D7, G4. Prose count fix. | false | n/a |
| A4-P4 | agree | none | D7, V5. Section reorder. | false | n/a |
| A4-P5 | agree | none | D7. Glossary additions. | false | n/a |
| A4-P6 | agree | none | V2, G5. ADR back-links; doc only. | false | n/a |
| A4-P7 | agree | none | V5, D7. Label unification. | false | n/a |
| A4-P8 | agree | none | D7. Appendix annotation. | false | n/a |
| A4-P9 | agree | none | D7. Glossary expansion. | false | n/a |
| A5-P1 | agree | none | R2. Backoff+jitter runs only on the failure path; no cost on success. | false | n/a |
| A5-P2 | agree | extends-latency | R3. A5 provides two-tier: steady-state `CLOSED` path collapses to a `thread_local` timestamp check (compile-time constant for steady state); single relaxed atomic load amortized below the existing `REMOTE_SUBMIT` 10 µs budget. A1 accepts the bridge. | false | n/a |
| A5-P3 | agree | none | R5. Control plane; off hot path. Per synthesis, v1 = fail-fast + Q-record (option B); decentralized v2. Merge absorption of A10-P2 accepted. | false | n/a |
| A5-P4 | agree | none | DS4. `idempotent` flag is cold — checked only on retry decision, not on dispatch. | false | n/a |
| A5-P5 | agree | none | R6. Test-build only (`IFaultInjector`); stripped from production builds. Fuses with A8-P7. | false | n/a |
| A5-P6 | agree | extends-latency | R5, O5. A5 provides two-tier: deadman written every Kth cycle (default K=16), amortized below TSC-read noise. A1 prefers lifting the write into the existing per-cycle timestamp capture so the net cost is zero; this is an implementation detail inside the structural bridge. Merge with A8-P4 accepted (see §9). | false | n/a |
| A5-P7 | agree | none | R1. Zero cost on success — `poll(zero)` reduces to today's hardware-flag check; timeout branch lives on slow path. | false | n/a |
| A5-P8 | agree | none | R4. Fires only on saturation / partial-group failure — both cold paths. | false | n/a |
| A5-P9 | agree | none | R3, R4. FSM transition is `on_fail` only; `WorkerManager::on_assign` already filters by `state == IDLE`, so `QUARANTINED` workers are invisible to the selector. | false | n/a |
| A5-P10 | agree | none | DS4. Per-handler idempotency contract; doc lint only. | false | n/a |
| A6-P1 | agree | none | S1. STRIDE doc; no runtime effect. | false | n/a |
| A6-P2 | agree | none | S1, S2. Handshake-time only; slow path. | false | n/a |
| A6-P3 | agree | extends-latency | S3, X2. A6 §7 tension row documents "bounds check executes only in the codec framing stage which is already a slow path relative to the hot-path RDMA transit" — bridge is the entry-gate single length-prefix guard per message (not per field). ≤ 10 ns per frame fits the 50 µs cross-node budget. | false | n/a |
| A6-P4 | agree | extends-latency | S4, S5. A6 documents two-tier: TLS on TCP only; RDMA (the hot path) uses RC + rkey scoping with no TLS cost. Init-time failure if plaintext + multi-host. A1 requires the doc to explicitly state "RDMA path not wrapped in TLS" in `10-known-deviations.md` Deviation 3. | false | n/a |
| A6-P5 | agree | extends-latency | S2, S3. A6's rkey rotation scoped to `SUBMISSION_RETIRED` (retirement cold path per `04-process-view.md:491`). One MR deregister per retired buffer ≈ 1 µs, off the 5 µs network-transit hot path. | false | n/a |
| A6-P6 | agree | none | S6. Pre-allocated audit ring; security events are rare and non-critical-path; no hot-path allocation. | false | n/a |
| A6-P7 | agree | none | S1, S3. Registration is startup/slow path; no hot-path effect. | false | n/a |
| A6-P8 | agree | extends-latency | S3. Fuses with A1-P11 per synthesis merge register; combined per-arg budget covers both validation and marshaling in the same 200 ns envelope. Fast path: inline structural checks; slow path: policy/registration on first use. | false | n/a |
| A6-P9 | agree | extends-latency | S1, S2, S3. A1-conditional agreement: `logical_system_id` must be placed **outside** the cache line holding `correlation_id` / `source_node` (per A6 §7 response). One integer compare per inbound dispatch (≤ 5 ns), off the Chip→Core hot path. | false | n/a |
| A6-P10 | agree | none | S2. Sink dispatch is on the flush thread, not hot path. | false | n/a |
| A6-P11 | agree | none | S2, S4. Init-time gate; no runtime effect. | false | n/a |
| A6-P12 | agree | none | S1, S3. REPLAY init only; not production hot path. Conditional on A9-P6 outcome — if A9-P6 agreed (defer REPLAY), A6-P12 becomes moot; A1 has no preference on that resolution. | false | n/a |
| A6-P13 | agree | extends-latency | S1. A6 §4 text documents two-tier: fast path = single relaxed counter increment (≤ 1 ns); slow path = leaky-bucket recompute only on refill tick. A1 requires the counter be placed per-tenant cache-aligned, and the refill run on a dedicated low-priority thread (not the scheduler event loop). Folds into the existing admission accounting — no new branch on admission. | false | n/a |
| A6-P14 | agree | none | G5, S5. ADR-only. | false | n/a |
| A7-P1 | agree | none | D6 (hard rule). Cycle removal is a module-structure change with no runtime effect; strengthens hot-path compile isolation. | false | n/a |
| A7-P2 | agree | none | ISP, D4. A7 §7 clarifies the aggregator `ISchedulerLayer : public IWiring, ISubmit, ICompletion, ILifecycle, IScope` preserves today's vtable layout — no new indirection. Merge with A9-P1 accepted (§9). | false | n/a |
| A7-P3 | agree | none | D2. Typedefs only; zero runtime effect. | false | n/a |
| A7-P4 | agree | none | D3, D5. Struct relocation; no runtime effect. | false | n/a |
| A7-P5 | agree | none | D2. Module-boundary cleanup; no runtime effect on the DistributedRuntime hot path (`REMOTE_*` dispatch is already `IDistributedProtocolHandler`-mediated). | false | n/a |
| A7-P6 | agree | none | D3, D5. Module extraction; no runtime effect. | false | n/a |
| A7-P7 | agree | none | D2, D6. Forward-decl contract; no runtime effect. | false | n/a |
| A7-P8 | agree | none | D7. Handle-type consolidation; no runtime effect. | false | n/a |
| A7-P9 | agree | none | D7. Python class rename; no runtime effect. | false | n/a |
| A8-P1 | agree | none | X5, §8.2. A8 documents link-time HAL specialization (`modules/hal.md:430, 247` pattern) so release binaries inline to today's `CLOCK_MONOTONIC_RAW` call — no indirect call, no allocation. Required for A1-P10's deterministic CI gate. | false | n/a |
| A8-P2 | agree | none | X5, §8.2. Test-only, gated on `enable_test_driver` compile flag; release binary unaffected. Enables A1-P5 budget validation reproducibly. | false | n/a |
| A8-P3 | agree | allocates | O3, O5. A8 §7 documents two-tier: hot-path insert = single branchless log-scale bucket increment (< 5 ns); allocation is at init only (~1 KiB per histogram); percentile computation on cold reader side. Insert is on completion path (`on_complete`), which already accepts the sub-µs budget in §4.8.3. A1 requires the bucket-insert to **not** cross a cache line and **not** use a mutex — proposer must confirm via the hot-path sketch. | false | n/a |
| A8-P4 | agree | none | O4, X6, §8.3. Cold diagnostic endpoint; seqlock / relaxed-atomic snapshot reads. Merge with A5-P6 accepted (§9). | false | n/a |
| A8-P5 | agree | none | O5, E3, X8. Alert-rule evaluation on low-priority collector thread; no effect on scheduler or dispatch path. | false | n/a |
| A8-P6 | agree | none | O1 distributed. Offline / export-path merge; no hot-path effect. Pairs with A8-P1 `IClock` for deterministic skew. | false | n/a |
| A8-P7 | agree | none | X5, R6. Sim-only; never compiled into onboard builds. Fuses with A5-P5. | false | n/a |
| A8-P8 | agree | extends-latency | O3. Two-tier: default `PROFILING_LEVEL = L1_Coarse` strips Level-2 AICore emits entirely (fast path = today). Level-2 is opt-in by operators (e.g., triggered by A8-P5 alerts). A1 retains veto on the exact ring-emit instruction sequence — implementer must demonstrate < 50 ns amortized device-side ring write before default-on. | false | n/a |
| A8-P9 | agree | none | O3, O5. Exposing existing drop counters; one relaxed-atomic read in cold stats path. | false | n/a |
| A8-P10 | agree | none | O2. Severity gate unchanged; KV emit only above threshold. Addresses `modules/profiling.md:463` open question. | false | n/a |
| A8-P11 | agree | none | X5. CI test-suite; no runtime effect. | false | n/a |
| A8-P12 | agree | allocates | O1. Two-tier: emits compiled out below `L2_Phase`; at L2 each emit < 100 ns per `Phase` per `modules/profiling.md:334`. A1 requires the default `PROFILING_LEVEL = L1_Coarse` is preserved so the fast path is untouched. Enables A1-P5/A1-P12 budget validation. | false | n/a |
| A9-P1 | agree | none | G2. Fewer overloads → smaller vtable, shorter hot-path instruction footprint. Merge with A7-P2 accepted (§9). | false | n/a |
| A9-P2 | agree | none | G2, G4. Slightly **reduces** virtual dispatch on the event-loop hot path (two v-calls → one enum switch). A1 conditions agreement on preserving A8-P2's driveable-event-loop test seam (`IEventLoopDriver` single test double) per synthesis bridge. | false | n/a |
| A9-P3 | agree | none | G2. Collectives removed from hot-path `IHorizontalChannel`; HCCL native path can return later as `ICollectiveOps` extension without touching the base interface. | false | n/a |
| A9-P4 | agree | none | G2, DRY. Direct A1 win: one fewer admission branch. | false | n/a |
| A9-P5 | agree | none | G2, DRY. Enum unification; no runtime effect. | false | n/a |
| A9-P6 | agree | none | G2. Defers `PERFORMANCE`/`REPLAY` to post-v1. A1 has no hot-path stake in sim modes; agrees with the simplification. Implication: A6-P12 becomes moot (noted there). | false | n/a |
| A9-P7 | agree | none | G2. Config struct merge; startup-time only. | false | n/a |
| A9-P8 | agree | none | G2, SRP. Doc scope; no runtime effect. | false | n/a |
| A10-P1 | agree | relayouts | P4, P6, DS7. A10 §7 documents two-tier: default single-threaded Stage B retains the degenerate uncontended RW-lock fast path; sharded path opt-in when `scheduler_thread_count > 1`. Aligns with A1-P2's amended default (`shards = max(1, scheduler_thread_count)`). A1 owns the per-shard lock-hold-time bound. | false | n/a |
| A10-P2 | agree | none | P4, R5. Control plane; off hot path. Per synthesis, merged into A5-P3; v1 = fail-fast + Q-record (option B). | false | n/a |
| A10-P3 | agree | none | DS6. Consistency-model doc; no runtime effect. | false | n/a |
| A10-P4 | agree | none | DS1. Stateful/stateless classification doc; no runtime effect. | false | n/a |
| A10-P5 | agree | extends-latency | P6. Merged into A1-P6 per synthesis register. Per-peer projection runs in distributed scheduler off the single-node hot path; fan-out = 1 fast path ships descriptor unchanged. | false | n/a |
| A10-P6 | agree | none | P4, R5. Fast-fail path on dedicated heartbeat thread; zero cost to admission/dispatch. | false | n/a |
| A10-P7 | agree | relayouts | P4. Two-tier: `shards = 1` default (today's single-path); opt-in via `LevelParams` when scale demands. Outstanding-window global counter (atomic) preserves the bound. Aligns with A1-P2 (shard_count as LevelParam). | false | n/a |
| A10-P8 | agree | none | DS7, P6. Diagram only. | false | n/a |
| A10-P9 | agree | none | DS6, P4. Log-keyed assignment; no hot-path cost. | false | n/a |
| A10-P10 | agree | none | P6, DS6. Merged into A1-P14 per synthesis register. Cap + layout guidance is the capacity/occupancy half of A1-P14's placement guidance — naturally combines. | false | n/a |

**Vote summary.** 96 votes cast: 96 agree, 0 disagree, 0 abstain, 0 blocking. No A1 veto applied this round — every hot-path-touching proposal (HPI ∈ {allocates, blocks, relayouts, extends-latency}) carries a documented two-tier path or compensating structural fix from the proposer, and A1 accepts each bridge.

## 7. Cross-aspect tensions (new observations)

| pair | with_proposal | preferred_resolution |
|------|---------------|----------------------|
| A1 vs A3 | A3-P4 (`DEP_FAILED` event + successor walk) | A1-suggested structural optimization (non-blocking): store Task successors as an inline small-vector in the hot prefix of the Task slot; large fan-out spills to a SoA side-table. Addresses A3's "O(successors)" concern without changing the hot path (small-vector fits the A1-P4 cold tail; fan-out walk happens on failure only). |
| A1 vs A6 | A6-P9 (`logical_system_id` on `MessageHeader`) | **Layout constraint (A1-conditioned):** the new `uint32_t logical_system_id` must land **outside** the 64 B cache line holding `correlation_id` / `source_node` / `sequence` — preferably in the header tail already shared with `crc32`. One integer compare stays off the Chip→Core hot path. A1 does not block, but agreement is conditional on the layout commit. |
| A1 vs A5 | A5-P6 (scheduler watchdog) | **Elision opportunity:** the per-cycle event-loop already captures a monotonic timestamp for phase annotation (A1-P5 + A8-P12); the deadman write should lift into that same capture rather than adding a separate store. Net hot-path cost: zero. |
| A1 vs A8 | A8-P3 (bucketed histograms on completion path) | **Layout constraint:** histogram bucket array must be a contiguous `uint32_t[N_BUCKETS]` per stat, cache-line-padded, with a branchless `__builtin_ctz` bucket index. Insert path must not cross a cache line. A1 does not block but requires the implementation sketch in round 2+ to include a measured < 5 ns single-insert microbench before default-on at L1. |
| A1 vs A8 | A8-P1 (`IClock` interface) | **Link-time specialization required:** release build must resolve `IClock::now_ns()` to today's inlined `CLOCK_MONOTONIC_RAW` call — no virtual dispatch, no indirect branch. A8's §7 response already commits to this pattern; A1 notes that A1-P5 / A1-P7 / A8-P12 all depend on the no-indirection guarantee. |
| A1 vs A9 | A9-P2 (cut `IEventCollectionPolicy` / `IExecutionPolicy`) | **Bridge preserved:** A8-P2's `IEventLoopDriver` single test-double seam is retained (per synthesis bridge) so test-driveable event loop survives; runtime replaces virtual dispatch with enum switch — A1 wins (fewer vtable calls on hot path) while A8 retains determinism. |
| A1 vs A2 | A2-P6 (`IDistributedProtocolHandler` registry) | **Confirmed two-tier:** top-N builtin `MessageType`s (`REMOTE_SUBMIT`, `REMOTE_COMPLETE`, `REMOTE_DEP_NOTIFY`, `REMOTE_DATA_READY`, `HEARTBEAT`) short-circuit via if-ladder before the registry table lookup. Registered extensions use the table. Registry is init-only (frozen; no runtime mutation). Common-path latency identical to today's switch. |
| A1 vs A5 | A5-P2 (per-peer circuit breaker) | **Placement constraint:** the per-peer `atomic<uint32_t>` counter must share a cache line with the peer's existing send-path state (already touched on every `REMOTE_SUBMIT`). Compile to one relaxed load already in-cache, no additional cache-line pull. |

## 8. Stress-attack on emerging consensus

*Round 3 only — skipped per prompt.*

## 9. Merge Register decisions (this aspect)

| Merge | Decision | Reason |
|-------|----------|--------|
| A1-P6 ← A10-P5 | **accept** | A1 is co-owner; A10-P5 contributes the per-peer projection angle that A1-P6's descriptor-template dedup did not explicitly cover. Combined title: "Distributed payload hygiene" (see §5 amendment). |
| A1-P14 ← A10-P10 | **accept** | A10-P10's cap + cache-line layout is the occupancy half of A1-P14's region/shard placement. One coherent proposal; combined text in §5 amendment. |
| A5-P3 ← A10-P2 | **accept** (not A1-owned) | A1 endorses: coordinator failover vs decentralization is one architectural decision. A1 prefers v1 = fail-fast + Q-record (synthesis option A), v2 = decentralize — keeps v1 hot path unchanged and avoids a Raft/Paxos protocol on the critical path. |
| A5-P6 ← A8-P4 | **accept** (not A1-owned) | A1 endorses: `dump_state()` provides the evidence surface the scheduler watchdog fires on; pairing keeps the diagnostic stack coherent. |
| A7-P2 ← A9-P1 | **accept** (not A1-owned) | A1 endorses: role-split + single `submit()` is one coherent ISP/G2 win. Shrinks the `ISchedulerLayer` vtable and preserves the aggregator. |
| A6-P8 ← A1-P11 (partial) | **accept** (A1 co-owner) | A1-P11 covers per-arg **marshaling budget**; A6-P8 covers per-arg **validation**. Both live at the Python↔C boundary and share the same ≤ 200 ns fast-path envelope — fusion is the natural correct form. |

All six merges accepted.

## 10. Status

- **Satisfied with current design?** partially — same as Round 1. Round 2 resolved the debate topology: every hot-path-touching peer proposal has documented a two-tier bridge, and every A1 proposal has a concrete revision aligned with peer feedback. The architecture's latency-budget discipline plus the accepted proposals put the design on track to structurally meet X2 / X3 / X9; round 3 stress-attacks are expected to focus on (a) whether A8-P3 histogram insert truly fits < 5 ns at L1, (b) whether A1-P6's distributed payload hygiene meets the 1 KB/wire target under realistic workloads, and (c) whether A1-P2's per-shard lock-hold bounds hold under `scheduler_thread_count = 8`.
- **Open items expected in round 3 (stress-attack targets):** A1-P2, A1-P4, A1-P6, A1-P7, A1-P12, A1-P13 (A1-owned); A8-P3 (histogram hot-path cost), A5-P2 (circuit-breaker atomic placement), A6-P9 (logical_system_id layout commit), A2-P6 (single-impl dispatch devirt), A3-P8 (debug/prod split honesty), A6-P3 (entry-gate bounds check under max-size frames).
- **A1 veto applications this round:** 0 (all hot-path proposals accepted with documented two-tier bridges).
- **Non-A1 override requests received:** 0.
