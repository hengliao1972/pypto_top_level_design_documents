# Aspect A10: Scalability & Data Flow — Round 1

## Metadata

- **Reviewer:** A10
- **Round:** 1
- **Target:** `/data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/`
- **Runtime Mode:** yes
- **Timestamp (UTC):** 2026-04-19T00:00:00Z

## 1. Rubric Checks

| # | Check | Result | Evidence (file:line) | Rule(s) |
|---|-------|--------|----------------------|---------|
| 1 | Horizontal scaling first (scale-out path documented across levels) | Weak | `docs/pypto-runtime-design/05-physical-view.md:150,159,169`; coordinator SPOF `docs/pypto-runtime-design/05-physical-view.md:187` | P4 |
| 2 | Stateless services; state externalized with explicit justification | Fail | Stateful `TaskManager` / `producer_index` / proxy pool carry runtime-critical state with no externalization; `docs/pypto-runtime-design/02-logical-view/02-scheduler.md:91–99`; only recovery deviation acknowledged, not statelessness — `docs/pypto-runtime-design/10-known-deviations.md:19` | DS1 |
| 3 | Async messaging where caller does not need an immediate response | Pass | Event-loop admission, async submit, `copy_async()` first-class; `docs/pypto-runtime-design/02-logical-view/08-communication.md:47,51`; `docs/pypto-runtime-design/04-process-view.md` (event-driven Scheduler stages) | DS2 |
| 4 | Minimize data movement; colocate compute with data | Pass | Control/data plane separation via `IVerticalChannel` + `IMemoryOps`; `DataLocalityPartitioner` colocates compute to largest input; `docs/pypto-runtime-design/modules/distributed.md:55`; `docs/pypto-runtime-design/02-logical-view/08-communication.md:47,64,71`; group-workspace arena collapses per-tensor allocations — `docs/pypto-runtime-design/02-logical-view/04-memory.md:151,156` | P6 |
| 5 | Explicit consistency model per data-store/service interaction | Weak | Only one statement: causal per task chain, eventual cross-chain, `docs/pypto-runtime-design/04-process-view.md:561`; no per-data-element matrix for `producer_index`, `cluster_view`, task-slot pool, group-availability table, remote-proxy pool | DS6 |
| 6 | Single authoritative owner per data element | Pass | `producer_index` owned by per-Layer `TaskManager` (`02-logical-view/02-scheduler.md:91,93`); non-aliasing intermediate-memref invariant gives every live `BufferRef` a unique live producer (`02-logical-view/04-memory.md:68`); proxy pool owned by `distributed_scheduler` (`modules/distributed.md:139`) | DS7 |

## 2. Pros

- **Distributed-first is structural, not bolted-on.** `distributed_scheduler` is a plain `ISchedulerLayer` variant that partitions work across peer Nodes; the Pod-level layer's `worker_count` is literally "more Nodes" (`05-physical-view.md:159`). This satisfies P4's intent at the Pod tier — adding nodes does add throughput to the same logical Layer.
- **Data-plane vs control-plane are architecturally separated.** Control travels on `IVerticalChannel` / `IHorizontalChannel`; bulk tensor data travels on `IMemoryOps` / RDMA (`02-logical-view/08-communication.md:47,64,71`). This is the textbook realization of P6 and keeps the hot path free of serialization work for payloads.
- **Async-by-default on the hot path.** The Scheduler event loop, the outstanding-Submission window, and `complete_in_future` on data-movement Tasks all remove synchronous waits from the critical path (`02-logical-view/02-scheduler.md:39–57`; `02-logical-view/08-communication.md:51`). DS2 is cleanly met.
- **Data ownership is single and named.** The `producer_index` lifecycle is spelled out with insertion / lookup / eviction rules and makes "most-recent-writer wins" explicit (`02-logical-view/02-scheduler.md:95–97`). Combined with the non-aliasing intermediate-memref invariant (`02-logical-view/04-memory.md:68`), DS7 is satisfied for the critical cross-Submission state.
- **Sharding is reserved for when scaling demands it.** `producer_index` MAY be partitioned by `BufferRef` hash (`02-logical-view/02-scheduler.md:121`) — the contract is pre-written so that adding shards requires no semantic change. Good scaling headroom.
- **Deterministic vs non-deterministic partitioners are explicitly labeled.** `DataLocalityPartitioner` is deterministic, `WorkStealingPartitioner` is not and is forbidden for replayable runs (`modules/distributed.md:55–56`). This prevents a silent scalability/idempotency clash later.
- **Cross-node back-pressure is a first-class citizen.** Proxy-pool exhaustion returns `SlotPoolExhausted` and blocks admission (`modules/distributed.md:392`), i.e., bounded-queue semantics rather than unbounded growth. This scales safely into many-Pod deployments.

## 3. Cons

- **Coordinator Node in a Pod is a declared SPOF.** `05-physical-view.md:187` calls this out and defers election to Q5. In a "scale-out first" design this is the exact component that should scale horizontally; as written, a Pod with N nodes still has one coordinator. Violates P4 spirit and R5.
- **Registry is frozen at init.** `05-physical-view.md:169` — "Dynamic node addition/removal at runtime is not in scope." Horizontal scaling under P4 normally includes online elasticity; this is a hard scalability ceiling, not just deferred work.
- **Stateless classification (DS1) is absent.** The design has many internally stateful modules (`TaskManager`, `RemoteTaskProxyPool`, `cluster_view`, producer_index, outstanding-Submission window) but no table separating stateful from stateless components and no justification for each statefulness. DS1 requires explicit justification, not implicit acceptance.
- **Per-data-element consistency model is incomplete.** `04-process-view.md:561` covers task-chain causality only. `producer_index`, `cluster_view`, `group_availability`, `remote_proxy_pool`, `outstanding_submission` metadata are all shared-mutable — DS6 requires an explicit consistency statement for each. The current doc hides this under "dep-metadata lock" (`02-logical-view/02-scheduler.md:107–115`) and a seqlock for `cluster_view`, without an aggregated model.
- **Single-valued `producer_index` restricts cross-Submission parallelism.** RAW-only scope is acknowledged (`02-logical-view/02-scheduler.md:101`) and pushed to Q12. In a scaled training workload with re-used boundary tensors (e.g., parameter buffers written then read by many micro-steps), WAR/WAW must be enforced by the frontend via `BARRIER`. This serializes what could otherwise parallelize across Submissions and directly reduces throughput at scale. P4.
- **Faint-to-dead peer detection is 3 s.** `heartbeat_interval_ms=500` × `heartbeat_miss_threshold=6` (`modules/distributed.md:407–408`) gives a 3-second worst-case until `NodeLost` fires. In a many-Pod cluster the tail-failure window dominates latency of straggler handling and idempotent retries.
- **REMOTE_SUBMIT payload ships whole SubmissionDescriptor to every assigned peer.** §5.1 sends `{submission, task_subset}` (`modules/distributed.md:302`). When a Submission is split across many peers this duplicates edges/boundaries to each peer. No contract to project per-peer task subsets with their locally-relevant edges only. Hurts P6 as fan-out grows.
- **Outstanding Submission Window is per-Layer and single-managed.** Defaults 8–64 at Host, 2–16 at Device/Chip (`02-logical-view/02-scheduler.md:43`). The TaskManager is a single serialization point per Layer and Stage-B admission is default-single-threaded; with a large number of concurrent user submitters, this caps ingress throughput even when ample hardware parallelism exists downstream. No sharded-TaskManager path is specified.
- **No data-flow/data-ownership diagram.** DS7 is met per field but there is no single place (e.g., `02-logical-view.md` or `07-cross-cutting-concerns.md`) listing {data element, owner module, access pattern, consistency, scaling axis}. A reader cannot audit scalability without re-deriving it from six files.
- **No explicit bound on per-Layer `producer_index` memory footprint under heavy OUT/INOUT fan-out.** The eviction rule (`02-logical-view/02-scheduler.md:97`) bounds it by outstanding-Submission OUT/INOUT sets, but for Group Submissions with many OUT tensors and a wide window (Host=64), this is a large hot-map that must stay cache-friendly. No sizing guidance.

## 4. Proposals

| id | severity | summary | affected_docs | hot_path_impact | trade_offs | sanity_test |
|----|----------|---------|---------------|-----------------|------------|-------------|
| A10-P1 | high | Make `producer_index` sharding the default and specify shard count | `02-logical-view/02-scheduler.md`, `modules/scheduler.md` | none | +contention relief at scale; −more RW-lock objects (~cache-line each) | Under simulated 8-thread admission, measure `submit` p99 at Host Layer with sharded vs unsharded; p99 should not rise vs single-threaded, whereas unsharded rises superlinearly |
| A10-P2 | high | Decentralize Pod-level coordination (sticky routing + quorum cluster_view) | `05-physical-view.md`, `modules/distributed.md`, `09-open-questions.md` (Q5) | none (off hot path) | +removes SPOF, enables true horizontal scaling; −adds coordination protocol complexity (agree on `cluster_view` generation) | Kill the old coordinator; verify another node continues accepting new Pod-level Submissions within one heartbeat tick; no user-visible error |
| A10-P3 | medium | Add an explicit per-data-element consistency table | `07-cross-cutting-concerns.md` (new section), referenced from `02-logical-view.md` | none (documentation) | +audit-ability; covers DS6 gap; −one table to maintain | Reviewer can name any runtime-shared data element and find a row specifying consistency and sync primitive |
| A10-P4 | medium | Classify modules as stateful / stateless with justification | `03-development-view.md` or `02-logical-view.md`, `10-known-deviations.md` | none | +DS1 compliance; −documentation debt | For every module, reader finds "stateful because …" or "stateless" in one place |
| A10-P5 | medium | Project per-peer `RemoteSubmitPayload` — ship only the edges and boundaries that peer actually needs | `modules/distributed.md` §5.1, `02-logical-view/08-communication.md` | extends-latency (slow path: partition step grows; fast-single-peer path unchanged) | +bytes on wire drop roughly linearly with peer count; −partitioner does additional projection pass | For an 8-peer partition, measure aggregate REMOTE_SUBMIT bytes; expect < (1/4 × N × full-payload) once projection is enabled |
| A10-P6 | medium | Make heartbeat detection time configurable separately from cadence, and add transport-level fast-fail piggyback | `modules/distributed.md` §config; `modules/transport.md` | none (dedicated heartbeat thread) | +sub-second NodeLost at scale; −risk of false positives on noisy links → must come with hysteresis | Kill a peer; observe `NodeLost` event timestamp − kill time < configured budget (e.g., 800 ms) while no false positives occur under simulated 5 % packet loss on a healthy peer |
| A10-P7 | medium | Document sharded / multi-writer TaskManager path and specify when to enable | `02-logical-view/02-scheduler.md`, `modules/scheduler.md` | none on default single-thread; `blocks` becomes `none` on sharded path | +removes per-Layer ingress serialization ceiling; −implementation complexity; MUST preserve earliest-first completion bias | Under 16-submitter stress, sharded TaskManager sustains ≥ 4× single-Layer ingress throughput without breaking the outstanding-window bound (admission is still rejected once total outstanding ≥ cap) |
| A10-P8 | medium | Add an aggregated data-flow + ownership diagram | `02-logical-view.md`, `07-cross-cutting-concerns.md` | none (documentation) | +DS7 visibility and cross-Submission RAW/WAR/WAW audit in one place; −one diagram to maintain | Reader traces any `BufferRef` through producer → consumer → retirement on a single page |
| A10-P9 | low | Flag `WorkStealingPartitioner` as incompatible with `RETRY_ELSEWHERE` unless the final assignment is logged | `modules/distributed.md` §3.1 and §5 | none | +prevents replay-time double execution of a stolen task on a different peer; −requires a lightweight assignment log in distributed scheduler | Kill a peer mid-run under `WorkStealing + RETRY_ELSEWHERE`; replay: verify every Task has exactly one final assignment written to the log |
| A10-P10 | low | Require an explicit cap and cache-line-aware layout for per-Layer `producer_index` | `02-logical-view/02-scheduler.md`, `modules/memory.md` | none (admission reads are hash lookups either way) | +predictable footprint under wide outstanding windows; −adds sizing guidance to config | Allocate `max_outstanding_submissions × max_OUT_per_submission` entries; verify structure fits expected working-set bound and single-bucket probe stays O(1) |

### Proposal detail

#### A10-P1: Default `producer_index` sharding

- **Rationale:** P4, P6, DS6/DS7. Today `producer_index` is guarded by a per-Layer RW lock (`02-logical-view/02-scheduler.md:107–115`); sharding is a MAY (`02-logical-view/02-scheduler.md:121`). As soon as multi-threaded Stage B or inline-caller admission is enabled, the single RW lock is the first contention point. Making N-way sharding the default closes the scale-out gap without changing correctness.
- **Edit sketch:**
  - File: `docs/pypto-runtime-design/02-logical-view/02-scheduler.md`
  - Location: §"Concurrency and locking" (around line 121)
  - Delta: promote "MAY partition" to "SHOULD partition with default shard count = 8 at Host / 4 at Device / 1 at Chip and below"; add one line: "Each shard has its own RW lock; admission takes the shard lock keyed by `hash(BufferRef) mod N`."
- **Trade-offs:**
  - Gain: contention relief under multi-threaded admission; retains single-lock semantics per shard.
  - Give up: N lock objects per Layer (cache-line-padded); small steady-state memory increase.
- **Sanity test:** Under simulated 8 concurrent submitters at Host Layer, measure admission p99. With shards=8, p99 must not degrade more than 10 % vs single-threaded; unsharded will trend with a lock-contention curve.

#### A10-P2: Decentralize Pod-level coordinator

- **Rationale:** P4 + R5. `05-physical-view.md:187` concedes a Pod coordinator SPOF. In a horizontal-scaling-first design this is the exact axis that must scale.
- **Edit sketch:**
  - File: `docs/pypto-runtime-design/05-physical-view.md` (§5.4.0) and `docs/pypto-runtime-design/modules/distributed.md` (new §"Coordinator Membership").
  - Delta: define a membership view where any peer can accept a Pod-level Submission; route the same `correlation_id` always to the same current-owner peer (sticky), with ownership transferred via `cluster_view` generation bump and quorum ack; on owner loss, the next peer in the membership picks up.
- **Trade-offs:**
  - Gain: removes declared SPOF; Pod throughput scales truly horizontally.
  - Give up: needs agreement on `cluster_view` generation (quorum), additional protocol complexity; explicit mention in ADR.
- **Sanity test:** Kill current owner; verify the Pod accepts a new Submission within one heartbeat tick on a different peer without user-visible error.

#### A10-P3: Explicit per-data-element consistency model

- **Rationale:** DS6. The only consistency statement found is `04-process-view.md:561` — causal per task chain, eventual cross-chain — which does not cover runtime-shared data structures that are accessed concurrently by admission and completion handlers.
- **Edit sketch:**
  - File: `docs/pypto-runtime-design/07-cross-cutting-concerns.md`
  - Location: new subsection "Consistency Model by Data Element".
  - Delta: table with columns `{data_element, owner_module, writers, readers, consistency (strong / causal / eventual), sync primitive}`. Minimum rows: `producer_index`, `cluster_view`, `group_availability`, `remote_proxy_pool`, `outstanding_submission_set`, `task_slot_pool.generation`, `completion_counters`.
- **Trade-offs:** documentation maintenance; no hot-path cost.
- **Sanity test:** For any name listed, a reviewer finds exactly one row and can answer "what happens if a reader observes a stale value?" from the table.

#### A10-P4: Stateful / stateless module classification

- **Rationale:** DS1. Runtime has many stateful components (TaskManager, ProxyPool, ClusterView, ProducerIndex, OutstandingWindow); DS1 requires explicit justification for each.
- **Edit sketch:**
  - File: `docs/pypto-runtime-design/03-development-view.md`
  - Location: new subsection "State Ownership & DS1 Compliance".
  - Delta: a table `{module, stateful?, justification, externalization notes}`. For stateful entries, note why externalization is not done (e.g., "microsecond path, persistence would exceed budget — Deviation 2").
- **Trade-offs:** none functional; a few pages of prose.
- **Sanity test:** For every module in `02-logical-view.md`, the classification table has an entry; stateful entries have a justification tied to hot-path budgets or correctness.

#### A10-P5: Per-peer projection of `REMOTE_SUBMIT`

- **Rationale:** P6. At high fan-out, shipping the full `SubmissionDescriptor` to every target peer duplicates edges/boundaries per peer (`modules/distributed.md:302`). At N-peer partition this is O(N × |submission|) bytes across the cluster where O(|submission|) suffices in aggregate.
- **Edit sketch:**
  - File: `docs/pypto-runtime-design/modules/distributed.md` §5.1
  - Delta: specify that the partitioner emits a per-peer sub-submission with only the tasks assigned there and only the edges/boundaries touching that subset; add a note that `correlation_id` is shared so that `REMOTE_DEP_NOTIFY` still joins across peers.
- **Trade-offs:**
  - Gain: linear reduction in aggregate REMOTE_SUBMIT bytes with fan-out; better P6.
  - Give up: partition step does extra projection work (O(|tasks| + |edges|)); off the single-node hot path.
- **Sanity test:** For an 8-peer partition of a 10,000-task Submission, measured aggregate REMOTE_SUBMIT bytes must be under `1.5 × full-payload` (vs `8 × full-payload` today).

#### A10-P6: Faster, configurable peer-failure detection

- **Rationale:** P4 (indirectly) and R5. `modules/distributed.md:407–408` yield ~3 s detection. In scaled deployments this dominates straggler handling latency.
- **Edit sketch:**
  - File: `docs/pypto-runtime-design/modules/distributed.md` §config (around line 407)
  - Delta: add `heartbeat_miss_threshold_fast` (default 2) gated by transport-level TCP-keepalive / RDMA completion error; on fast-fail signal, miss threshold drops from 6 to 2. Preserve current threshold as the slow-path baseline.
- **Trade-offs:**
  - Gain: sub-second NodeLost under hard failures.
  - Give up: possible false positive on a noisy link; requires hysteresis back to slow-path on recovery.
- **Sanity test:** Kill a peer; `NodeLost` event timestamp − kill time < 800 ms. Inject 5 % packet loss on a healthy peer for 10 s; no `NodeLost` fires.

#### A10-P7: Sharded TaskManager path

- **Rationale:** P4. Default single-threaded Stage B makes TaskManager a per-Layer ingress bottleneck (`02-logical-view/02-scheduler.md:39–57`). At scale, ingress must shard.
- **Edit sketch:**
  - File: `docs/pypto-runtime-design/02-logical-view/02-scheduler.md`
  - Delta: add subsection "Sharded TaskManager (opt-in)". Shards admit independently; the outstanding-Submission window is a global counter (atomic) so the bound holds cluster-wide on the Layer. Earliest-first completion bias is preserved by per-shard ordering; cross-shard ordering uses the global `submission_id`.
- **Trade-offs:**
  - Gain: removes the per-Layer ingress serialization ceiling.
  - Give up: added complexity; test surface grows.
- **Sanity test:** Under 16 concurrent submitters, sharded path sustains ≥ 4× single-Layer ingress throughput; admission is still rejected once total outstanding ≥ `max_outstanding_submissions`.

#### A10-P8: Aggregated data-flow + ownership diagram

- **Rationale:** DS7 + P6. Today ownership is correct but scattered across six files.
- **Edit sketch:**
  - File: `docs/pypto-runtime-design/02-logical-view.md`
  - Delta: a Mermaid `flowchart` showing each persistent data element, its owner module, its writers/readers, and its movement path across layers; referenced from every submodule.
- **Trade-offs:** documentation only; no hot-path cost.
- **Sanity test:** A new reviewer can answer "who writes `producer_index` and on what event" from one page.

#### A10-P9: Gate `WorkStealing` against `RETRY_ELSEWHERE`

- **Rationale:** DS6/P4. `WorkStealingPartitioner` is labeled non-deterministic (`modules/distributed.md:55–56`). Combined with `RETRY_ELSEWHERE`, a stolen-and-failed task could be retried on a different peer than the original assignment, producing observable non-idempotency at scale.
- **Edit sketch:**
  - File: `docs/pypto-runtime-design/modules/distributed.md` §3.1 and §"Error model"
  - Delta: declare the incompatibility explicitly; if both are enabled, require an assignment log keyed by `(submission_id, task_index) → final_node_id` written before dispatch.
- **Trade-offs:** documentation + small log; no hot-path cost.
- **Sanity test:** Under `WorkStealing + RETRY_ELSEWHERE`, kill a peer mid-run; replay: each task has exactly one final assignment row.

#### A10-P10: Cap and layout guidance for `producer_index`

- **Rationale:** P6 (cache efficiency on hot lookups) and DS6 (bounded state).
- **Edit sketch:**
  - File: `docs/pypto-runtime-design/02-logical-view/02-scheduler.md`
  - Delta: add a sizing line — `producer_index.capacity ≥ max_outstanding_submissions × expected_out_per_submission × 2`; require the hash map to be open-addressed with cache-line-sized buckets to keep the single-probe average.
- **Trade-offs:** slight memory pre-allocation; removes a rehash risk on the admission hot path.
- **Sanity test:** Static check that the configured capacity meets the bound; runtime assertion triggers if occupancy exceeds the bound.

## 5. Revisions of own proposals (round ≥ 2 only)

_Not applicable in round 1._

## 6. Votes on peer proposals (round ≥ 2 only)

_Not applicable in round 1._

## 7. Cross-aspect tensions

| pair | with_proposal | preferred_resolution |
|------|---------------|----------------------|
| A1 (Performance) vs A10 | A10-P1 (sharded producer_index) | Two-tier lock path: default single-threaded Stage B retains the degenerate uncontended RW-lock path (today's fast path, `02-logical-view/02-scheduler.md:115`); sharded path is compile/config-selected and costs one extra hash step only when multi-threaded admission is enabled. |
| A1 (Performance) vs A10 | A10-P5 (per-peer REMOTE_SUBMIT projection) | Projection runs in the distributed scheduler (off the single-node hot path). For fan-out = 1 the projection is a no-op fast path (ship the descriptor unchanged). |
| A9 (Simplicity) vs A10 | A10-P3 (consistency table) + A10-P4 (stateful classification) + A10-P8 (ownership diagram) | Merge all three into one "Data & State Reference" section in `07-cross-cutting-concerns.md` — one new section rather than three sprinkled additions. |
| A1 (Performance) vs A10 | A10-P6 (fast-fail heartbeat) | Fast-fail path is driven by transport-level signals on a dedicated heartbeat thread, not by the scheduler event loop — zero cost to admission/dispatch. |
| A2 (Extensibility) vs A10 | A10-P7 (sharded TaskManager) | Expose shard count via `LevelParams` so single-node deployments keep `shards=1` (current behavior) and only opt in when scale demands it. |
| A5 (Consistency) vs A10 | A10-P2 (decentralized coordinator) | Introduce quorum on `cluster_view` generation only; do not widen the protocol to a full Paxos variant in v1 — documented as an ADR with explicit scope. |

## 8. Stress-attack on emerging consensus (round 3+ only)

_Not applicable in round 1._

## 9. Status

- **Satisfied with current design?** partially
- **Open items expected in next round:** A10-P1, A10-P2, A10-P3, A10-P4, A10-P5, A10-P6, A10-P7, A10-P8, A10-P9, A10-P10
