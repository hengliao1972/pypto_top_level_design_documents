# Round 1 Synthesis

## Metadata

- **Round:** 1
- **Timestamp (UTC):** 2026-04-18 17:30:00
- **Runtime Mode:** yes
- **Reviewers that returned output:** A1, A2, A3, A4, A5, A6, A7, A8, A9, A10
- **Reviewers missing / failed:** none

## Classification Summary

Round 1 is the baseline catalog; no votes yet. `status` is provisional based on the synthesizer's surface scan for rule-cited findings and obvious dups. Formal classification arrives in round-2 after voting.

| Status | Count | Proposal IDs |
|--------|-------|--------------|
| agreed (provisional, pending R2 vote) | 0 | — |
| rejected | 0 | — |
| disputed (expected R2 contention) | 30 | see Conflict Register |
| candidate-fast-track (likely uncontroversial edits) | 37 | see Proposal Catalog §status=fast |
| candidate-regular | 43 | all others |
| incomplete | 0 | — |

Totals: **110 proposals** across 10 reviewers (A1:14, A2:9, A3:15, A4:9, A5:10, A6:14, A7:9, A8:12, A9:8, A10:10).

## Convergence Status

- **Status:** in-progress
- **Next action:** run round 2 (cross-critique + vote). All reviewers must vote on every peer proposal.

## Proposal Catalog

Severity key: B=blocker, H=high, M=medium, L=low. HPI=hot_path_impact.

| proposal_id | owner | co_owners | severity | HPI | status | summary |
|-------------|-------|-----------|----------|-----|--------|---------|
| A1-P1 | A1 | — | H | allocates→none | disputed | Pre-size `producer_index`; forbid rehash on hot path |
| A1-P2 | A1 | — | H | blocks(bounded) | disputed | Cap DATA-mode lock hold time; shard count as `LevelParam` |
| A1-P3 | A1 | A6 (P7 adjacency) | M | none | fast | Function Cache LRU + capacity + HEARTBEAT presence |
| A1-P4 | A1 | A7 | M | relayouts | disputed | Enumerate `Task` hot/cold field split; justify AoS |
| A1-P5 | A1 | A3 | H | none | fast | Latency budgets for SPMD fan-out + event-loop stages |
| A1-P6 | A1 | A10 (P5) | M | extends→none | disputed | Cap REMOTE_SUBMIT; stage binaries; dedup templates |
| A1-P7 | A1 | — | M | extends→none | fast | Per-thread local sequence + offline merge in profiling |
| A1-P8 | A1 | — | M | allocates→none | disputed | Pre-size OutstandingWindow/ReadyQueue; CONTROL placement |
| A1-P9 | A1 | — | L | relayouts | disputed | Bitmask-encode per-slot-type WorkerGroup availability |
| A1-P10 | A1 | A8 | L | none | fast | Workload-level profiling-overhead CI gate |
| A1-P11 | A1 | — | L | extends→none | fast | Per-arg marshaling budget at Python↔C boundary |
| A1-P12 | A1 | — | M | extends-latency | disputed | `BatchedExecutionPolicy.max_batch` + tail budget |
| A1-P13 | A1 | — | M | extends-latency | disputed | Bound `args_blob` copy inside DEP_READY→DISPATCHED |
| A1-P14 | A1 | A10 (P10) | M | none | fast | Document `producer_index` placement+shard default |
| A2-P1 | A2 | — | H | none | disputed | Version every public data contract |
| A2-P2 | A2 | — | M | none | disputed | Schema-registered `LevelOverrides` |
| A2-P3 | A2 | — | M | none | disputed | Open extension-point enums; keep `DepMode` closed |
| A2-P4 | A2 | — | H | none | fast | Migration & Transition Plan |
| A2-P5 | A2 | — | M | none | fast | Interface Evolution & Backward-Compat Policy |
| A2-P6 | A2 | — | M | none | disputed | Pluggable `IDistributedProtocolHandler` registry |
| A2-P7 | A2 | — | M | none | disputed | Reserve async-policy extension seam |
| A2-P8 | A2 | — | L | none | fast | Record intentional closures as known deviations |
| A2-P9 | A2 | A4 (P5), A8 | M | none | fast | Versioned trace schema |
| A3-P1 | A3 | — | B | none | fast | Add `ERROR` (+ opt `CANCELLED`) state to Task FSM |
| A3-P2 | A3 | — | H | none | fast | Clarify `submit()` return vs `AdmissionDecision` |
| A3-P3 | A3 | A5 | H | none | fast | Add admission-path failure scenario |
| A3-P4 | A3 | — | H | none | fast | Specify producer-failure → consumer propagation |
| A3-P5 | A3 | — | H | none | fast | Specify sibling cancellation policy |
| A3-P6 | A3 | — | M | none | fast | Requirement ↔ Scenario traceability matrix |
| A3-P7 | A3 | — | M | none | fast | Submission precondition validation |
| A3-P8 | A3 | — | M | extends(bounded) | disputed | Cyclic `intra_edges` detection algorithm + cost |
| A3-P9 | A3 | — | M | none | fast | SPMD index/size delivery contract |
| A3-P10 | A3 | A6 | M | none | fast | Python exception mapping completeness |
| A3-P11 | A3 | — | L | none | fast | `[ASSUMPTION]` marker at `complete_in_future` |
| A3-P12 | A3 | — | M | none | fast | `drain()`/`submit()` concurrency contract |
| A3-P13 | A3 | A10 | M | none | fast | Cross-node ordering assumptions |
| A3-P14 | A3 | — | L | none | fast | `COMPLETING`-skip transition at leaf |
| A3-P15 | A3 | — | L | none | fast | Debug-mode `NONE`-dep-mode cross-check |
| A4-P1 | A4 | — | H | none | fast | Canonicalize HAL enum member casing |
| A4-P2 | A4 | — | B | none | fast | Fix broken anchor in task-model x-ref |
| A4-P3 | A4 | — | H | none | fast | Fix invariant count prose ("Two"→"Three") |
| A4-P4 | A4 | — | M | none | fast | Reorder Open Questions numerically |
| A4-P5 | A4 | A2 (P9) | M | none | fast | Add glossary entries for event-loop plumbing |
| A4-P6 | A4 | — | M | none | fast | Cross-link top-level views to ADRs they encode |
| A4-P7 | A4 | — | L | none | fast | Unify L0 component label ("Scheduler: AicoreDispatcher") |
| A4-P8 | A4 | — | L | none | fast | Sync Appendix-B TaskState count |
| A4-P9 | A4 | — | L | none | fast | Expand `Task State` glossary entry |
| A5-P1 | A5 | — | H | none | fast | Exponential backoff + jitter for remote retries |
| A5-P2 | A5 | — | H | none | fast | Per-peer circuit breaker in `distributed/` |
| A5-P3 | A5 | — | H | none | disputed | Commit coordinator-failover OR deterministic fail-fast |
| A5-P4 | A5 | — | M | none | fast | `idempotent: bool` on `TaskDescriptor` gates retry |
| A5-P5 | A5 | A8 (P7) | M | none | fast | Chaos / fault-injection harness + mandatory scenarios |
| A5-P6 | A5 | A8 (P4) | M | none | fast | Scheduler-thread watchdog / deadman timer |
| A5-P7 | A5 | — | M | none | fast | `Timeout` on `IMemoryOps` async |
| A5-P8 | A5 | — | M | none | fast | Degradation specs for admission saturation + WG loss |
| A5-P9 | A5 | — | L | none | fast | QUARANTINED Worker state between FAILED and UNAVAILABLE |
| A5-P10 | A5 | — | M | none | fast | DS4 protocol contract — per-REMOTE_* idempotency |
| A6-P1 | A6 | — | H | none | fast | Complete the trust-boundary threat model |
| A6-P2 | A6 | — | H | none | fast | Concrete node authentication in `HANDSHAKE` |
| A6-P3 | A6 | — | H | extends(bounded) | disputed | Bounded variable-length payload parsing |
| A6-P4 | A6 | — | H | extends-latency | disputed | Secure-default TLS for multi-host TCP |
| A6-P5 | A6 | — | M | extends(minor) | disputed | Scoped, revocable RDMA `rkey` |
| A6-P6 | A6 | — | M | none | fast | Security audit trail |
| A6-P7 | A6 | — | M | extends(startup) | disputed | Function-binary attestation |
| A6-P8 | A6 | — | H | extends | disputed | Boundary validation for DLPack / CAI imports |
| A6-P9 | A6 | — | M | none | fast | Enforce Logical System isolation in wire + code |
| A6-P10 | A6 | — | L | none | fast | Capability-scoped log / trace sinks |
| A6-P11 | A6 | — | M | none | fast | Gated `register_factory` |
| A6-P12 | A6 | — | M | none | disputed | Signed / schema-validated REPLAY trace |
| A6-P13 | A6 | — | M | none | disputed | Per-tenant submit rate-limit |
| A6-P14 | A6 | — | M | none | fast | Key-material lifecycle ADR |
| A7-P1 | A7 | — | H | none | fast | Break `scheduler/` ↔ `distributed/` cycle |
| A7-P2 | A7 | — | H | none | disputed | Split `ISchedulerLayer` into role interfaces |
| A7-P3 | A7 | — | M | none | fast | Invert `core/` ↔ `hal/` dependency for handle types |
| A7-P4 | A7 | — | M | none | fast | Move distributed payload structs to `distributed/` |
| A7-P5 | A7 | — | H | none | fast | `distributed_scheduler` depends only on `ISchedulerLayer` |
| A7-P6 | A7 | — | M | none | fast | Extract MLR + deployment parser |
| A7-P7 | A7 | — | L | none | fast | Forward-decl contract for `core/i_scheduler_layer.h` |
| A7-P8 | A7 | — | M | none | fast | Consolidate `ScopeHandle` ownership |
| A7-P9 | A7 | — | L | none | fast | Deduplicate Python `MemoryError` class names |
| A8-P1 | A8 | A5 | M | none | fast | Injectable `IClock` interface |
| A8-P2 | A8 | — | M | none | fast | Driveable event-loop (`step()` + `RecordedEventSource`) |
| A8-P3 | A8 | A1 | M | none | fast | Stats structs + bucketed latency histograms |
| A8-P4 | A8 | A5 (P6) | M | none | fast | Runtime `dump_state()` diagnostic endpoint |
| A8-P5 | A8 | — | M | none | disputed | Externalize alert rules + Prometheus/OTEL sink |
| A8-P6 | A8 | — | M | none | fast | Distributed trace time-alignment contract |
| A8-P7 | A8 | A5 | M | none | fast | Uniform `IFaultInjector` test seam (sim-only) |
| A8-P8 | A8 | — | M | none | fast | Commit AICore in-core trace upload protocol |
| A8-P9 | A8 | — | L | none | fast | Profiling drop/degraded state as first-class alerts |
| A8-P10 | A8 | — | L | none | fast | Structured KV logging primary surface |
| A8-P11 | A8 | — | M | none | fast | HAL contract test suite (sim + onboard) |
| A8-P12 | A8 | — | L | none | fast | Stable `PhaseId`s for Submission lifecycle |
| A9-P1 | A9 | — | M | none | disputed | Drop `submit_group`/`submit_spmd` overloads |
| A9-P2 | A9 | — | H | none | disputed | Cut FULLY_SPLIT/SPLIT_DEFERRED + pluggable policies |
| A9-P3 | A9 | — | H | none | disputed | Remove collectives from `IHorizontalChannel` (Q4=A) |
| A9-P4 | A9 | — | M | none | disputed | Drop `SubmissionDescriptor::Kind` |
| A9-P5 | A9 | — | M | none | fast | Unify admission enums (DRY) |
| A9-P6 | A9 | — | H | none | disputed | Defer PERFORMANCE/REPLAY simulation to post-v1 |
| A9-P7 | A9 | — | M | none | disputed | Fold `SourceCollectionConfig` into `EventHandlingConfig` |
| A9-P8 | A9 | — | L | none | fast | Move AICore companion-artifacts obligation out of design |
| A10-P1 | A10 | A1 (P14) | M | none | fast | Default `producer_index` sharding |
| A10-P2 | A10 | A5 (P3) | H | none | disputed | Decentralize Pod-level coordinator |
| A10-P3 | A10 | — | M | none | fast | Explicit per-data-element consistency model |
| A10-P4 | A10 | — | M | none | fast | Stateful / stateless module classification |
| A10-P5 | A10 | A1 (P6) | M | extends→none | disputed | Per-peer projection of `REMOTE_SUBMIT` |
| A10-P6 | A10 | — | M | none | fast | Faster configurable peer-failure detection |
| A10-P7 | A10 | — | M | relayouts | disputed | Sharded TaskManager path |
| A10-P8 | A10 | — | M | none | fast | Aggregated data-flow + ownership diagram |
| A10-P9 | A10 | — | L | none | fast | Gate `WorkStealing` against `RETRY_ELSEWHERE` |
| A10-P10 | A10 | A1 (P14) | M | none | fast | Cap + layout guidance for `producer_index` |

## Agreement Matrix

Omitted — rounds ≥ 2 only.

## Conflict Register

Round-1 conflicts are detected by the synthesizer scanning for cross-aspect collisions, duplicated proposals, and hot-path-impact flags that invite an A1 veto. Each entry is routed to round 2 with a specific debate prompt.

### A1-P4 — enumerate `Task` hot/cold field split; justify AoS

- **Owner:** A1
- **Dissenters (expected):** A7, A9 (touches core type layout; possible overkill)
- **Blocking objections (expected):** none hard; A9 may label as over-spec
- **Tension pair:** A1 vs A9 (layout prescription)
- **Synthesizer observation:** HPI=`relayouts` invites A1 self-gate; proposal is doc-only, so the relayout is one-time. Bridge: keep enumeration in `modules/core.md` §8 (layout) rather than Logical.

### A1-P6 / A10-P5 — cap REMOTE_SUBMIT + staging channel / per-peer projection

- **Owner:** A1, A10 co-owners
- **Dissenters (expected):** A2 (adds protocol surface), A9 (YAGNI)
- **Tension pair:** A1 vs A2, A1 vs A9
- **Synthesizer observation:** A1-P6 and A10-P5 overlap strongly. Suggest merging under A1-P6 with A10-P5 contributing the per-peer projection rationale. Route as a single combined proposal for R2.

### A1-P12 — `BatchedExecutionPolicy.max_batch` + tail-latency budget

- **Owner:** A1
- **Dissenters (expected):** A9 (more tunables), possibly A3 (new parameter without scenario)
- **Tension pair:** A1 vs A9
- **Synthesizer observation:** HPI=`extends-latency`; A1's own proposal invites self-veto. A1 must provide a fast-path default (no knob) + slow-path tunable sketch in R2.

### A1-P13 — bound `args_blob` copy in DEP_READY→DISPATCHED

- **Owner:** A1
- **Dissenters (expected):** A7 (HAL contract widening), A5 (timeout interaction)
- **Tension pair:** A1 vs A7
- **Synthesizer observation:** tighten the HAL interface contract; A1 must show the pre-registered ring-slot alternative so A7 can judge modularity impact.

### A1-P1, A1-P8 — pre-sizing + placement (allocates→none)

- **Tension pair:** A1 vs A9 (YAGNI — fixed caps)
- **Synthesizer observation:** the R1 HPI transition (allocates→none) already justifies on X2. Expect R2 agreement; route to fast track unless A9 specifically objects.

### A1-P9 — bitmask-encode WorkerGroup availability

- **Dissenters (expected):** A2 (≤64 group cap), A9 (premature layout)
- **Tension pair:** A1 vs A2, A1 vs A9
- **Synthesizer observation:** A1 must show the split-group fallback for >64 groups; otherwise A2 may tag an extensibility regression.

### A2-P1 / A2-P9 — version every data contract / versioned trace schema

- **Dissenters (expected):** A9 (YAGNI for header fields not yet evolved)
- **Tension pair:** A2 vs A9
- **Synthesizer observation:** likely coalesces if A2 commits to a single uniform version discipline that costs no hot-path bytes (A1 should confirm).

### A2-P2 — schema-registered `LevelOverrides`

- **Dissenters (expected):** A9 (YAGNI), A1 (startup cost minor, OK at hot-path)
- **Tension pair:** A2 vs A9
- **Synthesizer observation:** the design already flags Q6 as open. A9 may push for "close the enum for v1 and use the Q6 ADR plan". Route: A2 must justify relative to Q6's horizon.

### A2-P3 — open extension-point enums, keep `DepMode` closed

- **Dissenters (expected):** A9 (YAGNI for the two kept-open enums); A7 (role-interface question)
- **Tension pair:** A2 vs A9
- **Synthesizer observation:** A2 already proposed closed-`DepMode` as a known deviation; this is the pre-compromise. Likely agreement.

### A2-P6 — Pluggable `IDistributedProtocolHandler` registry

- **Dissenters (expected):** A9 (YAGNI — one backend in v1), A1 (indirection on fallback path only; must confirm hot path untouched)
- **Tension pair:** A2 vs A9
- **Synthesizer observation:** hard A9 challenge. A2 must show single-implementer design keeps zero virtual dispatch on the message-handling hot path.

### A2-P7 — reserve async-policy extension seam

- **Dissenters (expected):** A9 (YAGNI); A1 (HPI only if seam is on hot path)
- **Tension pair:** A2 vs A9
- **Synthesizer observation:** similar to A2-P6. Need concrete YAGNI exemption (named future extender or existing Q).

### A3-P8 — cyclic `intra_edges` detection algorithm + cost

- **Dissenters (expected):** A1 (admission-path cost)
- **Tension pair:** A3 vs A1
- **Synthesizer observation:** HPI=`extends(bounded)`. A3 must commit to O(E) on the admission path with upper bound tied to `max_intra_edges` `LevelParam`.

### A5-P3 / A10-P2 — coordinator failover / decentralize Pod coordinator

- **Dissenters (expected):** A9 (YAGNI for v1), A1 (architecture change)
- **Tension pair:** A5 vs A9, A10 vs A9, A10 vs A1
- **Synthesizer observation:** dup semantics. Merge A10-P2 into A5-P3 as co-owner; R2 must decide between (a) fail-fast v1 + explicit roadmap for decentralized (b) Raft/Paxos v1. A9 pushes (a); A10 pushes (b). R2 decision likely: (a) with Q-record.

### A6-P3 — bounded variable-length payload parsing

- **Tension pair:** A6 vs A1 (A1 will check HPI=extends-bounded is truly fast-path-neutral)
- **Synthesizer observation:** safe provided cap is checked at trust-boundary entry, not per field. A1 must confirm.

### A6-P4 — secure-default TLS for multi-host TCP

- **Dissenters (expected):** A1 (extends-latency on multi-host TCP — but this is not the hot RDMA path)
- **Tension pair:** A6 vs A1
- **Synthesizer observation:** must show TLS is OFF on RDMA paths and ON for TCP control/submission only. Two-tier path specified.

### A6-P5 — scoped, revocable RDMA `rkey`

- **Dissenters (expected):** A1 (extends minor); A9 (added lifecycle complexity)
- **Tension pair:** A6 vs A1
- **Synthesizer observation:** revocation frequency limits are hot-path material. A6 must document rkey-rotation cadence.

### A6-P7 — function-binary attestation

- **Dissenters (expected):** A9 (YAGNI without trust boundary named); A1 (startup cost only, not hot path)
- **Tension pair:** A6 vs A1, A6 vs A9
- **Synthesizer observation:** A6 must show the threat it defends, and cite the rule (S2 least-privilege).

### A6-P8 — boundary validation for DLPack / CAI imports

- **Dissenters (expected):** A1 (adds marshaling cost); A9 (YAGNI inside single process)
- **Tension pair:** A6 vs A1
- **Synthesizer observation:** validation is at Python↔C boundary, budget P11 (A1-P11) already requires per-arg budgets. Natural fusion with A1-P11.

### A6-P12 — signed / schema-validated REPLAY trace

- **Tension with A9-P6** (which defers REPLAY entirely for v1).
- **Synthesizer observation:** A6-P12 is moot if A9-P6 agreed. Route: decide A9-P6 first in R2; if REPLAY kept, A6-P12 applies.

### A6-P13 — per-tenant submit rate-limit

- **Dissenters (expected):** A1 (new admission branch), A9 (no multi-tenancy scenario)
- **Tension pair:** A6 vs A1
- **Synthesizer observation:** HPI on admission path. A6 must define the rate-limit check as the existing admission accounting (no new branch) or tag as HPI=extends-latency.

### A7-P2 — split `ISchedulerLayer` into role interfaces

- **Dissenters (expected):** A9 (ceremony); A1 (if split introduces virtual dispatch)
- **Tension pair:** A7 vs A9, A7 vs A1
- **Synthesizer observation:** conflicts with A9-P1 (drop overloads). The two are orthogonal: ISP role-split keeps a single `submit(SubmissionDescriptor)` but separates submit vs manage vs query. Candidate to combine A9-P1+A7-P2 into an agreed proposal.

### A8-P5 — externalize alert rules + Prom/OTEL sink

- **Dissenters (expected):** A9 (YAGNI — no operator story yet); A1 (metrics read path only, none HPI)
- **Tension pair:** A8 vs A9
- **Synthesizer observation:** A8 must show which metrics are already exposed in the current design and demonstrate externalization is a format choice, not new code.

### A9-P1 — drop `submit_group`/`submit_spmd` overloads

- **Dissenters (expected):** A7 (interested in role split — see A7-P2), A3 (impact on submit return contract — see A3-P2)
- **Tension pair:** A9 vs A7
- **Synthesizer observation:** merge candidate with A7-P2 (net: one submit, split roles).

### A9-P2 — defer FULLY_SPLIT/SPLIT_DEFERRED + pluggable policies

- **Dissenters (expected):** A2 (OCP: removing the pluggable seam is a regression), A8 (test-driveable event loop wants `IEventCollectionPolicy`)
- **Tension pair:** A9 vs A2, A9 vs A8
- **Synthesizer observation:** A8-P2 ("driveable event-loop") is compatible with A9-P2 if the driveable seam is a single test double, not a full pluggable policy registry. Bridge: one `IEventLoopDriver` boundary (test-only) + closed enums for policies.

### A9-P3 — remove collectives from `IHorizontalChannel`

- **Dissenters (expected):** A2 (may prefer keeping as extension interface), potentially A1 (HCCL native path)
- **Tension pair:** A9 vs A2
- **Synthesizer observation:** Q4 direction. Likely agreement if roadmap note is kept.

### A9-P4 — drop `SubmissionDescriptor::Kind`

- **Dissenters (expected):** A3 (affects precondition validation — see A3-P7), A1 (one fewer branch on admission — agrees)
- **Tension pair:** A9 vs A3
- **Synthesizer observation:** likely agreement; A3's precondition check can rely on `spmd.has_value()`.

### A9-P6 — defer PERFORMANCE/REPLAY simulation

- **Dissenters (expected):** A8 (replay for debugging), A6 (P12 depends on REPLAY)
- **Tension pair:** A9 vs A8, A9 vs A6
- **Synthesizer observation:** central R2 debate. If agreed, A6-P12 is auto-rejected; if rejected, A6-P12 proceeds.

### A9-P7 — fold `SourceCollectionConfig` into `EventHandlingConfig`

- **Dissenters (expected):** A4 (glossary update scope — see A4-P5)
- **Tension pair:** A9 vs A4
- **Synthesizer observation:** A4-P5 adds entries for both; if A9-P7 agreed, A4-P5 drops `SourceCollectionConfig` entry.

### A10-P7 — sharded TaskManager path

- **Dissenters (expected):** A9 (v1 scope), A1 (`relayouts`, potential hot-path regress)
- **Tension pair:** A10 vs A1, A10 vs A9
- **Synthesizer observation:** HPI=relayouts invites A1 veto. A10 must provide a fast-path (unsharded, single-scheduler-thread) and slow-path (sharded) two-tier sketch.

## Open Disputes — Next-Round Focus

For each disputed proposal, what evidence/amendment would change the vote:

### A1-P4 (Task hot/cold split)
- **Required engagement:** A7 (module cohesion), A9 (is 64B hot-tier evidence-based?)
- **Evidence needed:** benchmark citation or L1-hit model
- **Suggested amendment:** move prescription to `modules/core.md` §8 only; keep logical view abstract

### A1-P6+A10-P5 (REMOTE_SUBMIT cap + staging)
- **Required engagement:** A2 (protocol versioning), A9 (YAGNI)
- **Evidence needed:** measured bytes-on-wire for 1000 identical remote submits
- **Suggested amendment:** merge into one proposal "distributed payload hygiene"

### A1-P12 (max_batch + tail budget)
- **Required engagement:** A9 (tunable proliferation), A3 (new scenario)
- **Evidence needed:** default value that satisfies the canonical burst scenario
- **Suggested amendment:** fast-path default of Dedicated; max_batch only applies when Batched is explicitly selected

### A1-P13 (bound args_blob copy)
- **Required engagement:** A7 (HAL contract), A5 (timeout interaction)
- **Evidence needed:** pre-registered ring-slot sketch
- **Suggested amendment:** HAL contract states `args_blob ≤ 4 KiB` or fast-path

### A1-P9 (bitmask WorkerGroup avail)
- **Required engagement:** A2 (extension to >64 groups), A9 (premature)
- **Evidence needed:** multi-bitmap fallback algorithm
- **Suggested amendment:** spec the 2×64 bitmap fallback

### A2-P2 (schema-registered LevelOverrides)
- **Required engagement:** A9, A1
- **Evidence needed:** size of startup cost; Q6 alignment
- **Suggested amendment:** phase as "closed for v1, schema-registered for v2" with Q6 as vehicle

### A2-P6 (pluggable IDistributedProtocolHandler)
- **Required engagement:** A9, A1
- **Evidence needed:** single-impl dispatch elimination (devirt)
- **Suggested amendment:** keep abstract only in distributed_scheduler; deliver one concrete backend with inlined dispatch

### A2-P7 (async-policy seam)
- **Required engagement:** A9, A1
- **Evidence needed:** named future extender (which Q does it resolve?)
- **Suggested amendment:** rename as Q record rather than new interface

### A3-P8 (cyclic intra_edges detection)
- **Required engagement:** A1 (admission cost)
- **Evidence needed:** O(|V|+|E|) with `max_intra_edges` cap
- **Suggested amendment:** confine detection to debug mode; fast-path skips

### A5-P3+A10-P2 (coordinator failover / decentralize)
- **Required engagement:** A9 (v1 scope), A1 (architecture upheaval)
- **Evidence needed:** fail-fast cost + roadmap
- **Suggested amendment:** v1 = fail-fast + Q-record; v2 = decentralize

### A6-P3 (bounded payload parsing)
- **Required engagement:** A1
- **Evidence needed:** cap check at boundary entry only
- **Suggested amendment:** single length-prefix guard per message

### A6-P4 (TLS on TCP)
- **Required engagement:** A1
- **Evidence needed:** RDMA path unaffected
- **Suggested amendment:** explicit "TLS on TCP only; RDMA uses RC + rkey scoping"

### A6-P5 (scoped rkey)
- **Required engagement:** A1, A9
- **Evidence needed:** revocation cadence
- **Suggested amendment:** rotation on Submission retirement, not per task

### A6-P7 (function-binary attestation)
- **Required engagement:** A9, A1
- **Evidence needed:** concrete threat (multi-tenant)
- **Suggested amendment:** gate behind a `trust_boundary.multi_tenant=true` config; off by default for single-tenant

### A6-P8 (boundary validation DLPack/CAI)
- **Required engagement:** A1 (fuse with A1-P11), A9
- **Evidence needed:** per-arg validation budget fits A1-P11
- **Suggested amendment:** merge with A1-P11

### A6-P12 (signed REPLAY trace)
- **Depends on A9-P6 outcome**

### A6-P13 (per-tenant rate-limit)
- **Required engagement:** A1 (admission branch), A9 (tenancy scope)
- **Evidence needed:** no new branch (fold into existing admission accounting) or defer
- **Suggested amendment:** defer to when multi-tenant deployment is a scenario

### A7-P2 (split ISchedulerLayer)
- **Required engagement:** A9 (ceremony), A1 (virtual dispatch count)
- **Evidence needed:** final vtable count vs current
- **Suggested amendment:** combine with A9-P1 into one proposal

### A8-P5 (externalize alert rules)
- **Required engagement:** A9
- **Evidence needed:** format choice only, not new subsystems
- **Suggested amendment:** declare external alert-rule file location; OTEL sink as a known deviation ledger entry

### A9-P1 (drop submit overloads)
- **Merge with A7-P2**

### A9-P2 (cut split policies + pluggables)
- **Required engagement:** A2, A8
- **Evidence needed:** A8 driveable seam survives with single test double
- **Suggested amendment:** keep one `IEventLoopDriver` test-only seam

### A9-P3 (remove collective from channel)
- **Required engagement:** A2
- **Evidence needed:** roadmap note
- **Suggested amendment:** keep proposal; add ADR that collectives land as Orchestration Functions in v1

### A9-P4 (drop Kind)
- **Required engagement:** A3
- **Evidence needed:** precondition validation via `spmd.has_value()` works
- **Suggested amendment:** drop

### A9-P6 (defer PERFORMANCE/REPLAY)
- **Required engagement:** A8, A6
- **Evidence needed:** no scenario in v1 needs them
- **Suggested amendment:** ship FUNCTIONAL-only; add ADR deferring PERFORMANCE/REPLAY to v2

### A9-P7 (fold SourceCollectionConfig)
- **Required engagement:** A4
- **Evidence needed:** clean merge
- **Suggested amendment:** merge; A4-P5 drops one entry

### A10-P7 (sharded TaskManager)
- **Required engagement:** A1, A9
- **Evidence needed:** fast-path single-shard default
- **Suggested amendment:** spec two-tier; A1-P2 covers shard_count as LevelParam

## Semantic-Duplicate Merge Register

The synthesizer proposes the following dedup actions before R2; owners may reject and keep separate.

| Merge into | Absorbed | Rationale |
|------------|----------|-----------|
| A1-P6 | A10-P5 | Same scope: REMOTE_SUBMIT payload hygiene. A10 is co-owner on the per-peer projection angle. |
| A1-P14 | A10-P10 | Both document `producer_index` layout/capacity; A10 is co-owner. |
| A5-P3 | A10-P2 | Coordinator-failover and Pod-coordinator-decentralization are the same architectural issue. |
| A5-P6 | A8-P4 | `dump_state()` gives the watchdog its evidence surface; keep as paired fast-track. |
| A7-P2 | A9-P1 | Role-split + single `submit()` overload is one coherent change. |
| A6-P8 | A1-P11 (partial) | Per-arg boundary validation fits inside A1-P11's per-arg budget. |

R2 reviewers should vote on the merged form; the absorbed id becomes an alias.

## Stress-Round Results

Omitted — round 3 only.

## Runtime-Mode Hot-Path Audit

| Proposal | HPI | Required two-tier? | Owner's response required in R2 |
|----------|-----|--------------------|----------------------------------|
| A1-P4 | relayouts | no (one-time doc) | no |
| A1-P9 | relayouts | yes (>64 groups) | yes |
| A1-P12 | extends-latency | yes (Batched vs Dedicated) | yes |
| A1-P13 | extends-latency | yes (ring-slot vs copy) | yes |
| A3-P8 | extends (bounded) | yes (debug vs prod) | yes |
| A6-P3 | extends (bounded) | no (entry-gate) | yes |
| A6-P4 | extends-latency | yes (TCP vs RDMA) | yes |
| A6-P5 | extends (minor) | yes (rotation cadence) | yes |
| A6-P7 | extends (startup) | no | yes |
| A6-P8 | extends | yes (fuse with A1-P11) | yes |
| A10-P7 | relayouts | yes (shard-count=1 default) | yes |
| A1-P6+A10-P5 | extends→none | completed | no |

All hot-path-touching proposals are captured here. A1 will weight these in R2 votes.

## Notes for the parent

- Runtime-mode hot-path audit: pass — every proposal carries HPI.
- A1 veto applications this round: 0 (no votes yet).
- Non-A1 override requests: 0.
- Deduplication actions proposed (6): listed above; owners may reject in R2.
- Round-2 launch conditions: give every reviewer all 10 reviews + this synthesis + the merge register; emphasise HPI scrutiny for the 12 flagged proposals.
