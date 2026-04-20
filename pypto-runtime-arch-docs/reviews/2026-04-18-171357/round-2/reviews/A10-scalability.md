# Aspect A10: Scalability & Data Flow — Round 2

## Metadata

- **Reviewer:** A10
- **Round:** 2
- **Target:** `/data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/`
- **Runtime Mode:** yes
- **Timestamp (UTC):** 2026-04-19T00:00:00Z

## 1. Rubric Checks

(Unchanged from R1 — A10 rubric results still apply; cross-referenced for A1 HPI weighting.)

| # | Check | Result | Evidence (file:line) | Rule(s) |
|---|-------|--------|----------------------|---------|
| 1 | Horizontal scaling first | Weak | `05-physical-view.md:150,159,169,187` | P4 |
| 2 | Stateless services / externalized state | Fail | `02-logical-view/02-scheduler.md:91-99`; `10-known-deviations.md:19` | DS1 |
| 3 | Async messaging | Pass | `02-logical-view/08-communication.md:47,51`; `04-process-view.md` | DS2 |
| 4 | Minimize data movement; colocate compute with data | Pass | `02-logical-view/08-communication.md:47,64,71`; `modules/distributed.md:55` | P6 |
| 5 | Explicit consistency model per data element | Weak | `04-process-view.md:561` only | DS6 |
| 6 | Single authoritative owner | Pass | `02-logical-view/02-scheduler.md:91,93`; `02-logical-view/04-memory.md:68`; `modules/distributed.md:139` | DS7 |

## 2. Pros

(Carry forward from R1; no revisions after reading peer feedback.)

- Distributed-first is structural; `distributed_scheduler` is a `ISchedulerLayer` variant (P4, `05-physical-view.md:159`).
- Control/data-plane split matches P6 (`02-logical-view/08-communication.md:47,64,71`).
- Async-by-default hot path meets DS2.
- Single authoritative owner per data element (DS7).
- Sharding seam pre-written for `producer_index` (P4 headroom).
- Deterministic vs non-deterministic partitioners labeled explicitly.
- Bounded back-pressure via `SlotPoolExhausted` (admission-queue discipline, P4).

## 3. Cons

(Carry forward from R1; peer reviews added no new scalability/data-flow defects not already captured by A10-P1..P10.)

- Pod-coordinator SPOF (`05-physical-view.md:187`) — P4/R5.
- Registry frozen at init (`05-physical-view.md:169`) — P4 ceiling.
- DS1 classification missing.
- DS6 per-element consistency table missing.
- `producer_index` RAW-only scope serializes cross-Submission reuse (P4).
- 3-second fail detection dominates straggler latency at scale.
- `REMOTE_SUBMIT` full-payload fan-out (P6 at scale).
- Per-Layer `TaskManager` is single admission point (P4 ingress cap).
- No aggregated data-flow/ownership diagram (DS7 audit-ability).
- No explicit cap/layout on `producer_index` (X4/P6).

## 4. Proposals

_No NEW proposals in R2. All A10 open items are revisions of A10-P1..P10 in Section 5._

## 5. Revisions of own proposals (round ≥ 2 only)

### 5.1 Merge-Register response (A10-involved merges from `round-1/synthesis.md`)

| merge row | decision | note |
|-----------|----------|------|
| `A1-P6 ⊇ A10-P5` (per-peer `REMOTE_SUBMIT` projection) | **accept** | A10-P5 becomes an alias of A1-P6. The per-peer projection rationale rolls up into A1-P6's combined payload-hygiene proposal. |
| `A1-P14 ⊇ A10-P10` (producer_index placement + cap + layout) | **accept** | A10-P10 becomes an alias of A1-P14. Capacity-bound + open-addressed cache-line-aligned buckets merge into A1-P14's placement/shard-default document. |
| `A5-P3 ⊇ A10-P2` (coordinator failover / decentralize) | **accept with amendment** | A10-P2 becomes an alias of A5-P3 **and** A10-P2 amended to the pragmatic form below: v1 = deterministic fail-fast + Q-record; v2 = decentralized membership. |

### 5.2 Revision table

| id | action | amended_summary (if amend/split) | reason |
|----|--------|----------------------------------|--------|
| A10-P1 | defend | — | Co-owned with A1-P14; `MAY → SHOULD` default sharding is the minimum change to make horizontal admission safe (P4). Peer A9 dissent is addressed by keeping `shards=1` at Device/Chip by default (YAGNI-bounded). |
| A10-P2 | split + concede-to-merge | **A10-P2a (v1, into A5-P3):** deterministic fail-fast + ADR + Q-record (Q5) — blocker-severity (R5/P4). **A10-P2b (v2, roadmap-only):** sticky + quorum `cluster_view` generation; parked as ADR for post-v1 (P4). | Synthesizer observation + A9 YAGNI + A1 architecture-change weight make v1 fail-fast the landing point; v2 elasticity is preserved as a roadmap entry rather than dropped. |
| A10-P3 | defend | — | DS6 cannot be met implicitly; one consistency table in `07-cross-cutting-concerns.md` is the minimum. Tension with A9 resolved by merging P3+P4+P8 into one "Data & State Reference" section (see §7). |
| A10-P4 | defend | — | DS1 requires explicit justification for every stateful module; one classification table is zero hot-path cost. |
| A10-P5 | concede-to-merge | Folded into A1-P6. | Same scope, same rule basis (P6). Single combined proposal avoids protocol-surface dispute being litigated twice. |
| A10-P6 | amend | "Split heartbeat **cadence** from **detection budget**; add transport-fast-fail signal and **hysteresis** back to slow-path; document that the fast-fail path runs on a dedicated heartbeat thread and does **not** touch the scheduler event loop (HPI=none preserved)." | A1 will vote on HPI; explicit hysteresis addresses false-positive concern under 5% loss, and the dedicated-thread clause neutralizes any implicit admission-path cost. |
| A10-P7 | amend | "**Two-tier sharded TaskManager**: default `admission_shards=1` (current, zero-overhead behavior); opt-in `admission_shards>1` at **Host Layer only** via `LevelParams` (align with A1-P2). The outstanding-Submission window remains a single global counter so the window bound holds cluster-wide. Earliest-first completion bias preserved by per-shard ordering + global `submission_id` tiebreak." | HPI=`relayouts` → `none` on default; explicit fast-path quiets A1 self-veto; LevelParam surface reuses A1-P2's already-proposed extensibility. A9 YAGNI addressed because shards>1 is opt-in. |
| A10-P8 | amend | "**Single combined section** in `07-cross-cutting-concerns.md` titled 'Data & State Reference' absorbing P3 (consistency table) + P4 (stateful/stateless table) + P8 (ownership diagram), referenced from submodules via a single backlink." | Addresses A9 simplicity tension: three sprinkled additions become one page; DS1/DS6/DS7 audit is co-located. |
| A10-P9 | defend | — | Low-cost, doc + log entry; prevents a silent scalability/idempotency clash when both WorkStealing and RETRY_ELSEWHERE are enabled. No peer dissent expected. |
| A10-P10 | concede-to-merge | Folded into A1-P14. | A1-P14 is the same artifact (producer_index placement + shard default). Aggregating capacity bound + open-addressed + cache-line bucket shape into A1-P14 avoids documentation drift. |

Net effect of revisions: **A10 open items reduce from 10 → 7** (P1, P2a, P3, P4, P6, P7, P8, P9 plus roadmap-only P2b). P5/P10 are absorbed; P2 is split with v1 merged into A5-P3.

## 6. Votes on peer proposals (round ≥ 2 only)

**Vote discipline (A10 lens):** P4 (horizontal scale), P6 (minimize data movement), DS1 (stateless), DS2 (async), DS6 (explicit consistency), DS7 (ownership), R5 (no SPOF). A10 does **not** hold hot-path veto; `override_request` is used only when a blocking A1 veto appears likely to be unjustified.

### 6.1 A1 (Performance & Hot Path)

| proposal_id | vote | rationale (cite rule id) | blocking | override_request |
|-------------|------|--------------------------|----------|-------------------|
| A1-P1 | agree | Pre-sized `producer_index` kills rehash on admission hot path — direct support for P6 + X2; aligns with absorbed A10-P10. | false | false |
| A1-P2 | agree | Bounded DATA-mode hold time + `shard_count` as `LevelParam` is the exact surface A10-P1 and amended A10-P7 need; P4 + X3. | false | false |
| A1-P3 | agree | LRU + capacity on Function Cache is DS7 (bounded owned state) and P6 (no unbounded memory growth under scale). | false | false |
| A1-P4 | agree | Hot/cold split + AoS justification is X4; doc-only, one-time relayout. Scalability-neutral but supports cache efficiency at fan-out. | false | false |
| A1-P5 | agree | SPMD fan-out + event-loop stage budgets are X9 and DS6 (bounds are explicit). Required to verify P4 scale claims. | false | false |
| A1-P6 | agree | Combined payload hygiene + per-peer projection (A10-P5 absorbed). Strong P6 support at fan-out. | false | false |
| A1-P7 | agree | Per-thread local sequence + offline merge is X3 (no cross-thread lock) and P4 (scales with submitters). | false | false |
| A1-P8 | agree | Pre-sized OutstandingWindow/ReadyQueue + placement is X2 + P6; no scalability downside. | false | false |
| A1-P9 | agree | Bitmask encoding is P6/X4. Conditional on A1 shipping the 2×64-bitmap fallback for >64 groups (E4 extensibility); my agreement is tied to that fallback being spec'd in R2 response. | false | false |
| A1-P10 | agree | Profiling-overhead CI gate is O3 + X1; catches hot-path regressions that silently hurt scaling. | false | false |
| A1-P11 | agree | Per-arg marshaling budget at Py↔C boundary is P6 + X9. Natural fusion with A6-P8. | false | false |
| A1-P12 | agree | `BatchedExecutionPolicy.max_batch` + tail budget is P5 + X9 — gated to Batched policy so Dedicated hot path is untouched (two-tier in synthesis). | false | false |
| A1-P13 | agree | Bounded `args_blob` copy (ring-slot or ≤4 KiB) is P6 + X2. Resolves an implicit HAL size ambiguity at scale. | false | false |
| A1-P14 | agree | `producer_index` placement + shard default (A10-P10 absorbed). Doc-only; P4 + X4. | false | false |

### 6.2 A2 (Extensibility & Evolvability)

| proposal_id | vote | rationale (cite rule id) | blocking | override_request |
|-------------|------|--------------------------|----------|-------------------|
| A2-P1 | agree | Versioning every public data contract is E6; zero hot-path bytes if versions are out-of-band or fixed-width header. Essential for cross-version Pod rollouts (P4). | false | false |
| A2-P2 | agree | Schema-registered `LevelOverrides` is E3 + E6; startup-only cost. Supports deployment elasticity under P4. | false | false |
| A2-P3 | agree | Open extension-point enums + closed `DepMode` is E4 balanced with G2 (KISS where correctness matters). Right trade. | false | false |
| A2-P4 | agree | Migration & transition plan is E5; v1→v2 decentralization path (amended A10-P2b) needs this scaffolding. | false | false |
| A2-P5 | agree | Interface evolution + backward-compat policy is E2 + E6; required for multi-node heterogeneous rollouts. | false | false |
| A2-P6 | agree | Pluggable `IDistributedProtocolHandler` is E4; off-hot-path. Conditional on single-impl devirt being shown (synthesis required engagement). | false | false |
| A2-P7 | abstain | No named future extender tied to a Q record; E4 benefit speculative vs G2 YAGNI cost. Would flip to agree if A2 names an R2 extender. | false | false |
| A2-P8 | agree | Known-deviations ledger is X-exceptions hygiene; zero cost. | false | false |
| A2-P9 | agree | Versioned trace schema is E6; consumer-compat across cluster rollouts directly supports P4. | false | false |

### 6.3 A3 (Functional Sanity & Correctness)

| proposal_id | vote | rationale (cite rule id) | blocking | override_request |
|-------------|------|--------------------------|----------|-------------------|
| A3-P1 | agree | `ERROR` (+ optional `CANCELLED`) is DS3 (partial-failure behavior is defined). Critical for cross-node failure propagation under scale. | false | false |
| A3-P2 | agree | `submit()` return contract clarity is D4 (narrow stable interface). | false | false |
| A3-P3 | agree | Admission-path failure scenario is V4 + DS3. | false | false |
| A3-P4 | agree | Producer-failure → consumer propagation is DS3; prevents silent stalls at scale. | false | false |
| A3-P5 | agree | Sibling cancellation policy is DS3 + DS6 (explicit semantics). | false | false |
| A3-P6 | agree | Req↔scenario traceability is V3. | false | false |
| A3-P7 | agree | Submission precondition validation is S3 + G3. | false | false |
| A3-P8 | agree | Cyclic `intra_edges` detection — conditional on debug-only default (two-tier in synthesis). O(E) cost bounded by `max_intra_edges` LevelParam keeps HPI off. | false | false |
| A3-P9 | agree | SPMD index/size contract is V3 + D7 (consistent naming across partitions). | false | false |
| A3-P10 | agree | Python exception mapping is S3 hygiene. | false | false |
| A3-P11 | agree | `[ASSUMPTION]` marker is G3. | false | false |
| A3-P12 | agree | `drain()`/`submit()` concurrency contract is DS6 — directly addresses A10's DS6 gap. | false | false |
| A3-P13 | agree | Cross-node ordering assumptions is DS6 + P6 — co-owned by A10; strongly supports horizontal-scale correctness. | false | false |
| A3-P14 | agree | `COMPLETING`-skip at leaf is a simplification; DS6 (state-transition completeness). | false | false |
| A3-P15 | agree | Debug `NONE`-dep-mode cross-check is X6 (production-debuggable by enabling). | false | false |

### 6.4 A4 (Document Consistency)

| proposal_id | vote | rationale (cite rule id) | blocking | override_request |
|-------------|------|--------------------------|----------|-------------------|
| A4-P1 | agree | Canonical HAL enum casing is D7. | false | false |
| A4-P2 | agree | Broken anchor fix — V5. | false | false |
| A4-P3 | agree | Invariant count prose fix — V5. | false | false |
| A4-P4 | agree | Numerically ordered Open Questions — doc hygiene. | false | false |
| A4-P5 | agree | Glossary entries (co-owned with A2-P9) — D7. A4 should drop `SourceCollectionConfig` entry if A9-P7 passes. | false | false |
| A4-P6 | agree | Cross-link views → ADRs — V2. | false | false |
| A4-P7 | agree | Unify L0 label — V5 + D7. | false | false |
| A4-P8 | agree | Appendix-B TaskState count sync — V5. | false | false |
| A4-P9 | agree | Expand Task State glossary — D7. | false | false |

### 6.5 A5 (Reliability & Fault Tolerance)

| proposal_id | vote | rationale (cite rule id) | blocking | override_request |
|-------------|------|--------------------------|----------|-------------------|
| A5-P1 | agree | Backoff + jitter for remote retries is R2 — prevents cascading retry storms at scale. | false | false |
| A5-P2 | agree | Per-peer circuit breaker is R3 — essential under large peer count (P4). | false | false |
| A5-P3 | agree | Coordinator failover or fail-fast is R5 + P4 — absorbs A10-P2 (v1 = fail-fast + Q-record; v2 roadmap for decentralize). | false | false |
| A5-P4 | agree | `idempotent:bool` gating retry is DS4. | false | false |
| A5-P5 | agree | Chaos/fault-injection harness is R6; required to verify P4 claims at scale. | false | false |
| A5-P6 | agree | Scheduler-thread watchdog is R5 (single-point protection). | false | false |
| A5-P7 | agree | Timeout on `IMemoryOps` async is R1; no bounded wait = no bounded scale. | false | false |
| A5-P8 | agree | Degradation specs for admission saturation + WG loss is R4. | false | false |
| A5-P9 | agree | QUARANTINED Worker state is R3. | false | false |
| A5-P10 | agree | Per-REMOTE_* idempotency contract is DS4 — directly scale-relevant. | false | false |

### 6.6 A6 (Security & Trust Boundaries)

| proposal_id | vote | rationale (cite rule id) | blocking | override_request |
|-------------|------|--------------------------|----------|-------------------|
| A6-P1 | agree | Trust-boundary threat model is S1. | false | false |
| A6-P2 | agree | Concrete node auth in `HANDSHAKE` is S1 + S2. | false | false |
| A6-P3 | agree | Bounded variable-length payload parsing is S3 + also P6 at boundary entry (protects against unbounded memory consumption under adversarial fan-in). HPI bounded via single length-prefix guard per message (synthesis). | false | false |
| A6-P4 | agree | TLS-default on TCP is S5; must not touch RDMA path (two-tier). Scalability-neutral once RDMA is excluded. | false | false |
| A6-P5 | agree | Scoped/revocable RDMA `rkey` is S2; rotation on Submission retirement keeps HPI minor. | false | false |
| A6-P6 | agree | Security audit trail is S6. | false | false |
| A6-P7 | abstain | Function-binary attestation (S2) is valuable only when multi-tenant is a scenario; no such scenario in v1. Would agree if gated behind `trust_boundary.multi_tenant=true` config (off by default). | false | false |
| A6-P8 | agree | DLPack/CAI boundary validation is S3 — fuse with A1-P11 per-arg budget. | false | false |
| A6-P9 | agree | Logical System isolation enforcement is S2 — also supports P4 by preventing cross-system interference. | false | false |
| A6-P10 | agree | Capability-scoped log/trace sinks is S2. | false | false |
| A6-P11 | agree | Gated `register_factory` is S2 + S4. | false | false |
| A6-P12 | abstain | Depends on A9-P6 outcome. If REPLAY kept in v1 → agree (S6). If REPLAY deferred → moot. | false | false |
| A6-P13 | abstain | No declared multi-tenant scenario in v1 (G2 YAGNI); if kept, must fold into existing admission accounting to avoid a new admission branch. | false | false |
| A6-P14 | agree | Key-material lifecycle ADR is S5 + S6. | false | false |

### 6.7 A7 (Modularity & SOLID)

| proposal_id | vote | rationale (cite rule id) | blocking | override_request |
|-------------|------|--------------------------|----------|-------------------|
| A7-P1 | agree | Break scheduler↔distributed cycle is D6 — supports independent scale evolution of distributed layer (P4). | false | false |
| A7-P2 | agree | Split `ISchedulerLayer` into role interfaces is D3 (SRP); merge-candidate with A9-P1. | false | false |
| A7-P3 | agree | Invert core↔HAL for handles is D2. | false | false |
| A7-P4 | agree | Move distributed payload structs to `distributed/` is D5 (cohesion). | false | false |
| A7-P5 | agree | `distributed_scheduler` depends only on `ISchedulerLayer` is D2 + D4. | false | false |
| A7-P6 | agree | Extract MLR + deployment parser is D3 — supports P4 (rolling-config changes without core rebuild). | false | false |
| A7-P7 | agree | Forward-decl contract is D6 (hygiene). | false | false |
| A7-P8 | agree | Consolidate `ScopeHandle` ownership is DS7 — perfectly aligned with A10 lens. | false | false |
| A7-P9 | agree | Dedup Python `MemoryError` classes is D7. | false | false |

### 6.8 A8 (Testability & Observability)

| proposal_id | vote | rationale (cite rule id) | blocking | override_request |
|-------------|------|--------------------------|----------|-------------------|
| A8-P1 | agree | Injectable `IClock` is X5; enables sim-accelerated large-scale tests (P4 validation). | false | false |
| A8-P2 | agree | Driveable event-loop is X5; required test surface for A10-P7 verification. | false | false |
| A8-P3 | agree | Stats structs + bucketed latency histograms is O3 — direct input to verifying P4 scaling claims. | false | false |
| A8-P4 | agree | Runtime `dump_state()` is X6 + O4. | false | false |
| A8-P5 | agree | Externalize alert rules + Prom/OTEL sink is O5 — operability-at-scale belongs with P4. | false | false |
| A8-P6 | agree | Distributed trace time-alignment is O1 — essential for cross-node scalability debugging. Strongly aligned with A10. | false | false |
| A8-P7 | agree | `IFaultInjector` (sim-only) is R6. | false | false |
| A8-P8 | agree | AICore in-core trace upload protocol — O4 + X6. | false | false |
| A8-P9 | agree | Profiling drop/degraded alerts is O5. | false | false |
| A8-P10 | agree | Structured KV logging is O2. | false | false |
| A8-P11 | agree | HAL contract test suite is X5 + X7. | false | false |
| A8-P12 | agree | Stable `PhaseId`s is O1. | false | false |

### 6.9 A9 (Simplicity — KISS/YAGNI)

| proposal_id | vote | rationale (cite rule id) | blocking | override_request |
|-------------|------|--------------------------|----------|-------------------|
| A9-P1 | agree | Drop `submit_group`/`submit_spmd` overloads is G2 (KISS); single `submit(SubmissionDescriptor)` is cleaner. Merge-candidate with A7-P2. | false | false |
| A9-P2 | abstain | Cutting FULLY_SPLIT/SPLIT_DEFERRED + pluggable policies simplifies, but A8-P2 requires a driveable event-loop seam. Would agree if A9 commits to the bridge (keep one `IEventLoopDriver` test-only seam). | false | false |
| A9-P3 | agree | Remove collectives from `IHorizontalChannel` is G2; roadmap note for Orchestration-Functions-as-collectives in v1 satisfies A2 concern. | false | false |
| A9-P4 | agree | Drop `SubmissionDescriptor::Kind` is G2 + DRY; `spmd.has_value()` carries the same information. | false | false |
| A9-P5 | agree | Unify admission enums is DRY. | false | false |
| A9-P6 | agree | Defer PERFORMANCE/REPLAY simulation is G2 YAGNI for v1; ADR-deferral is sufficient. A6-P12 becomes moot consequently. | false | false |
| A9-P7 | agree | Fold `SourceCollectionConfig` into `EventHandlingConfig` is G2. | false | false |
| A9-P8 | agree | Move AICore companion-artifacts obligation out of design is D3 (separate concern). | false | false |

### 6.10 Vote summary

| — | agree | disagree | abstain | total |
|---|-------|----------|---------|-------|
| A1 | 14 | 0 | 0 | 14 |
| A2 | 8 | 0 | 1 | 9 |
| A3 | 15 | 0 | 0 | 15 |
| A4 | 9 | 0 | 0 | 9 |
| A5 | 10 | 0 | 0 | 10 |
| A6 | 11 | 0 | 3 | 14 |
| A7 | 9 | 0 | 0 | 9 |
| A8 | 12 | 0 | 0 | 12 |
| A9 | 7 | 0 | 1 | 8 |
| **Total** | **95** | **0** | **5** | **100** |

No blocking votes; no `override_request` requests (A10 is not A1 and has raised no hot-path objections).

## 7. Cross-aspect tensions

(R1 tensions that are now resolved by revisions are noted; newly observed tensions are added.)

| pair | with_proposal | preferred_resolution |
|------|---------------|----------------------|
| A1 vs A10 (R1) | A10-P1 (sharded `producer_index`) | **Resolved by A1-P2 + A10-P1 amendment.** A1-P2 already puts `shard_count` on `LevelParams`; A10-P1 keeps `shards=1` default at Device/Chip, `SHOULD partition` at Host — uncontended path identical to today. |
| A1 vs A10 (R1) | A10-P5 → A1-P6 | **Resolved by merge.** Projection runs in the distributed scheduler; fan-out=1 is a no-op fast path. |
| A9 vs A10 (R1) | A10-P3 + A10-P4 + A10-P8 | **Resolved by amended A10-P8.** Single "Data & State Reference" section in `07-cross-cutting-concerns.md` absorbs all three. |
| A1 vs A10 (R1) | A10-P6 (fast-fail heartbeat) | **Resolved by A10-P6 amendment.** Dedicated heartbeat thread + hysteresis; HPI=none preserved. |
| A2 vs A10 (R1) | A10-P7 (sharded TaskManager) | **Resolved by A10-P7 amendment.** `admission_shards` via `LevelParams`, default 1; A2 gains an extensibility seam, A1 keeps zero-overhead default. |
| A5 vs A10 (R1) | A10-P2 (decentralized coordinator) | **Resolved by concede-to-merge + split.** A10-P2a (fail-fast, v1) merges into A5-P3; A10-P2b (decentralized membership) becomes a v2 ADR roadmap entry. |
| A1 vs A10 (new) | A10-P7 (two-tier shards) vs A1-P2 (shard count as LevelParam) | **Converged.** A1-P2's LevelParam surface is the exact vehicle A10-P7 needs — deliver as one combined spec in `02-logical-view/02-scheduler.md` + `modules/scheduler.md`. |
| A6 vs A10 (new) | A6-P3 (bounded payload parsing) vs A10-P5→A1-P6 (per-peer projection) | **Complementary.** Per-peer projection *shrinks* the payload A6-P3 must bound-check. Deliver A6-P3's length-prefix guard in the same REMOTE_SUBMIT handler change as A1-P6's projection. |
| A8 vs A10 (new) | A8-P6 (distributed trace time-alignment) | **Strong alignment.** A8-P6 provides the *observability prerequisite* for A10-P2b (v2 decentralized coordinator): without aligned cross-node timestamps, membership-generation debates cannot be audited. Treat as soft dependency in the v1→v2 roadmap ADR. |
| A9 vs A10 (new, soft) | A9-P6 (defer PERFORMANCE/REPLAY) vs A10-P9 (WorkStealing × RETRY_ELSEWHERE log) | **Compatible.** A10-P9 is a REPLAY-adjacent safety gate. If A9-P6 passes (REPLAY deferred to v2), A10-P9 narrows to the FUNCTIONAL-mode assignment log only, which is still needed for RETRY_ELSEWHERE correctness on live runs. |
| A1 vs A10 (new, clarifying) | A1-P9 (bitmask WG availability) | **Agree conditional.** If cluster has >64 `WorkerGroup`s per Layer (possible at Pod scale), A1 must ship the multi-bitmap fallback; otherwise P4 is capped. Not a blocker; tracked for A1's R2 response. |

## 8. Stress-attack on emerging consensus (round 3+ only)

_Not applicable in round 2._

## 9. Status

- **Satisfied with current design?** partially — satisfied with the direction of A1, A3, A5, A7, A8, A9 converging proposals; remaining A10 scalability gaps (DS1 classification, DS6 table, Pod-coordinator SPOF path) only close after the amended A10-P1..P9 + merge-register absorptions are written into the design docs.
- **Open items expected in next round:** A10-P1, A10-P2a (via A5-P3), A10-P2b (v2 roadmap ADR), A10-P3, A10-P4, A10-P6, A10-P7, A10-P8, A10-P9.
