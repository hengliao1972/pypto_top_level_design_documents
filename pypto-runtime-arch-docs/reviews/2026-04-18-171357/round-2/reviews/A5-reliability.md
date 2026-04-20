# Aspect A5: Reliability & Fault Tolerance — Round 2

## Metadata

- **Reviewer:** A5
- **Round:** 2
- **Target:** `/data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/`
- **Runtime Mode:** yes
- **Timestamp (UTC):** 2026-04-19T00:00:00Z

---

## 1. Rubric Checks

Round-1 rubric findings stand. Round-2 does not re-evaluate rubric results (the target docs have not been edited between rounds). The rows below are restated in compressed form for cross-round readability; see round-1 A5 §1 for full evidence.

| # | Check | Round-1 result | Round-2 delta |
|---|-------|----------------|---------------|
| 1 | Timeouts on every remote / IPC / device call | Weak | unchanged; closed by A5-P7 + A5-P6 if adopted |
| 2 | Retries bounded with exp. backoff + jitter | Fail | unchanged; closed by A5-P1 if adopted |
| 3 | Circuit breakers for failing dependencies | Fail | unchanged; closed by A5-P2 + A5-P9 if adopted |
| 4 | Graceful degradation per dependency | Weak | unchanged; closed by A5-P8 + A6-P13 + A10-P6 if adopted |
| 5 | No SPOF on critical path | Weak | unchanged; partially closed by A5-P3⊕A10-P2 merge if adopted |
| 6 | Idempotency under retry | Weak | unchanged; closed by A5-P4 + A5-P10 + A3-P4 + A3-P5 if adopted |
| 7 | Failure-mode test plan | Weak | unchanged; closed by A5-P5 ⇔ A8-P7 (co-owned) if adopted |

---

## 2. Pros

Round-1 pros stand unchanged; not repeated. Peer reviews reinforce the following A5-relevant strengths newly surfaced in round 1:

- **A3-P4 + A3-P5** converge on deterministic producer-failure / sibling-cancel propagation, supplying the missing mechanism behind my round-1 R1/DS3 concern ("Consumer-of-failed-producer is under-specified"). Cross-aspect alignment.
- **A8-P1 (`IClock`) + A8-P2 (driveable event-loop) + A8-P7 (`IFaultInjector`)** provide the test infrastructure that A5-P5's chaos matrix needs. The three are joint enablers.
- **A10-P2 (decentralize Pod coordinator) + A10-P6 (faster peer-failure detection)** directly attack two of the biggest round-1 R5 gaps I named (coordinator SPOF, 3 s detection window).
- **A7-P1 (break scheduler ↔ distributed cycle)** removes a D6 defect that would have otherwise hidden reliability state flows across the module boundary. A healthier module graph makes the cross-module reliability contracts enforceable.

---

## 3. Cons

Round-1 cons stand. Round-2 surfaces these additional A5-relevant concerns from reading peer reviews:

- **A1-P13 (HAL `args_blob` ≤ 1 KiB fast path) leaves the slow path open-ended** without specifying a reliability contract for the staging `BufferRef` when `args_size > 1 KiB`. If the staging `BufferRef` allocation fails, the scheduler must surface a bounded failure (likely `ResourceExhausted`); this is not documented. Cites R1, R4.
- **A8-P5 (`AlertRule` schema) tightens O5 but does not require a `DependencyUnavailable` alert family** tied to R4's graceful-degradation decisions. If A5-P8 adds `CONTINUE_REDUCED` / `COALESCE` / `DEFER` as admission-pressure responses, the alert schema must name them so operators can see *which* degradation is active. Cites O5, R4.
- **A6-P2 (`HandshakePayload` credentials) implies a fail-closed on auth failure** but the interaction with A5's `FailurePolicy::RETRY_ELSEWHERE` is not drawn: a peer with a flapping credential should not be retried-ELSEWHERE indefinitely — the auth-fail signal must feed A5-P2's circuit-breaker state transition to OPEN, not count as a generic remote failure. Cites R3, S1.
- **A10-P6 (fast-fail heartbeat via transport signals) lowers the detection floor to ~800 ms** but introduces a hysteresis requirement that is not elaborated: when does a quarantined/OPEN peer transition back to healthy? The answer must agree with A5-P2 (circuit breaker `HALF_OPEN` probe) and A5-P9 (Worker `QUARANTINED`). Without a single state machine these two paths can livelock. Cites R3, R5.
- **A9-P6 (defer `PERFORMANCE`/`REPLAY` sim modes)** is attractive for simplicity but eliminates one of the vehicles for deterministic reliability testing (trace-driven replay of a failure scenario). If A9-P6 passes, A5-P5's chaos matrix must commit to `FUNCTIONAL`-only reproduction. Cites R6.
- **A9-P3 (remove collectives from `IHorizontalChannel`) risks pushing `FailurePolicy::CONTINUE_REDUCED`'s all-reduce fallback onto Orchestration composition without specifying how a partial-failure collective behaves.** If a collective is a Group Submission of `transfer()`s, one child-peer failure must map cleanly to `CONTINUE_REDUCED` with a defined reduced result (or `ABORT_ALL`). Cites R4.

---

## 4. Proposals

No NEW A5 proposals in round 2. Existing proposals A5-P1 … A5-P10 remain as round-1, with revisions tracked in §5.

---

## 5. Revisions of own proposals

| id | action | amended_summary | reason |
|----|--------|-----------------|--------|
| A5-P1 | defend | — | No peer objection; hot-path impact `none` by construction (failure path only); A1 tension already resolved in round-1 §7. |
| A5-P2 | amend | Keep circuit breaker; pre-empt A1 hot-path concern by **requiring the steady-state `CLOSED` check to be a single relaxed-atomic load on a cache-resident counter, gated behind a `thread_local` last-checked TSC so the check collapses to a branch-predicted constant in the `CLOSED` state**. Auth-failure signals (A6-P2) feed the breaker as "hard fail" (counts ×K vs normal fail). | A1 hot-path discipline; integrates with A6-P2 (auth-failure fast-fails without polluting generic retry counts); integrates with A10-P6 (fast-fail signal). |
| A5-P3 | amend + merge | Accept synthesis merge with **A10-P2** (co-owner). Amended target: **v1 ships a deterministic fail-fast (`CoordinatorLost` within `heartbeat_timeout_ms` on every surviving peer; all outstanding `REMOTE_*` fail with that code; `Python driver receives `DistributedError`); v2 opens the decentralized, quorum-`cluster_view`-generation path that A10-P2 specifies**. Q5 resolved toward v1 fail-fast + explicit v2 roadmap; record A10-P2's decentralization as a v2 Q entry with trigger condition. | Balances A9 (YAGNI for v1), A10 (horizontal scale goal), R5 (no silent hang). Fail-fast is the minimum-viable R5 compliance; decentralization is the R5-plus-P4 solution when the cluster size demands it. |
| A5-P4 | amend | Add `idempotent: bool = true` on `TaskDescriptor`; **the Scheduler MAY single-node-retry only if `idempotent == true`**. On `idempotent == false`, on Worker failure the Task transitions `EXECUTING → ERROR` (leveraging A3-P1's new `ERROR` state) and the producer-failure propagation (A3-P4) fires `DEP_FAILED` to consumers without retry. Cross-reference the new `AdmissionStatus` enum from A9-P5 if that proposal lands. | Resolves the interaction with A3-P1/P4/P5 (which I explicitly needed: "ERROR state + DEP_FAILED + sibling cancellation" is the substrate my DS4 contract plugs into). Without A3-P1 the flag has no terminal state to select. |
| A5-P5 | amend + co-own | Keep the mandatory-scenario matrix + DR drill cadence owned by A5. **Fuse the `IFaultInjector` seam with A8-P7 as the shared test interface** (A8-P7 defines the seam; A5-P5 defines the scenario matrix and required outcomes). If A9-P6 passes (defer REPLAY), the matrix commits to `FUNCTIONAL`-only reproduction; if A9-P6 fails, REPLAY becomes the preferred determinism path. | A8-P7 is the cleaner owner of the *mechanism* (uniform sim-only compile target). A5 remains the owner of the *contract* (what faults must be covered and what response is required). This avoids two seams. |
| A5-P6 | amend + co-own | Accept synthesis merge with **A8-P4** (co-owner). Amended target: scheduler deadman slot written every Kth cycle (K=16 default; A1 hot-path discipline); **`SchedulerStalled` alert surfaced via the same `dump_state()` / `Runtime::dump_state()` endpoint that A8-P4 introduces**, so the watchdog's evidence surface is `dump_state().scheduler.deadman_age_ns`. Alert rule externalized per A8-P5. | A8-P4 gives the watchdog the diagnostic surface it needs; co-ownership keeps the two proposals coherent rather than duplicative. |
| A5-P7 | defend | — | No peer objection; A1 tension resolved round-1 §7 (zero cost on success `poll(zero)`). |
| A5-P8 | amend | Specify degradation-policy enum at the scheduler layer: `AdmissionPressurePolicy { REJECT, COALESCE, DEFER }` and `GroupPartialAvailabilityPolicy { FAIL_SUBMISSION, SHRINK_AND_RETRY, DEGRADED_CONTINUE }`. **Tie each enum value to an `AlertRule` per A8-P5 so operators can observe which degradation is active** (addresses the round-2 Con I raised above). Default policies remain the current behaviour (`REJECT`, `FAIL_SUBMISSION`) so round-1 consumers see no change without opting in. | Integrates with A8-P5 (externalized alerts); preserves backward compatibility for A2; gives A3 a deterministic admission-result to validate against. |
| A5-P9 | defend | — | A9 YAGNI objection anticipated; counter-argument stands: `QUARANTINED` is the minimum R3 compliance at Worker granularity. Cost = one FSM state + one `on_probe` path; the `WorkerManager::on_assign` selector already filters on `state == IDLE`, so `QUARANTINED` Workers are invisible to the selector at zero hot-path cost. Severity `low` is already a concession; I will not concede further. |
| A5-P10 | defend | — | Pure doc-lint contract; no hot-path impact; does not conflict with any peer proposal. |

---

## 6. Votes on peer proposals

Votes on every peer proposal (100 total: A1×14, A2×9, A3×15, A4×9, A6×14, A7×9, A8×12, A9×8, A10×10). Rationale cites the rule(s) an A5 reviewer weights. `blocking=true` only where the objection rests on a hard rule from `04-agent-rules.md`. `override_request` applies only if I disagree with a hypothetical future A1 veto; left `false` everywhere in round 2 since A1 has not yet voted.

### A1 — Performance & Hot Path

| proposal_id | vote | rationale (cite rule id) | blocking | override_request |
|-------------|------|--------------------------|----------|-------------------|
| A1-P1 | agree | X2 (pre-size = no admission-time alloc); also improves R4 under burst (no latency cliff). | false | false |
| A1-P2 | agree | X3 (bounded lock hold); reliability win — bounded admission tail latency feeds SLO alerts (O5). | false | false |
| A1-P3 | agree | P2 (cache policy explicit); Bloom-filter presence avoids round-trip on hot path and keeps Function-Cache miss recovery bounded (R1-adjacent). | false | false |
| A1-P4 | agree | X4 (layout explicit); doc-only on logical view, no reliability impact; improves testability of A8-P3 histograms by pinning hot fields. | false | false |
| A1-P5 | agree | X9 + P5; SPMD fan-out and event-loop stage budgets are prerequisites for SLO-alerts (O5) that A5 relies on. | false | false |
| A1-P6 | agree | P6 + R4 (bounded message avoids the chunked-reassembly slow path that interacts with R1 `remote_timeout_ms`); agrees with synthesis merge absorbing A10-P5. | false | false |
| A1-P7 | agree | X3 (no cross-socket ping-pong); profiling fidelity preserved; improves observability for R6 chaos tests. | false | false |
| A1-P8 | agree | X2 + X4; explicit `WouldBlock` is a defined R4 back-pressure response. | false | false |
| A1-P9 | agree | X4; hot-path neutral for A5; 2×64 bitmap fallback for >64 groups adequately addresses scale. | false | false |
| A1-P10 | agree | P1 (SLO enforcement); directly supports R6 by failing CI when profiling breaks SLO. | false | false |
| A1-P11 | agree | X9 per-arg decomposition; feeds A6-P8 validation budget. | false | false |
| A1-P12 | agree | X9 + P5; tail-latency budget is an SLO the R4 graceful-degradation alerts must be keyed to. Prefer Dedicated default with opt-in Batched. | false | false |
| A1-P13 | agree | P6 + X9, but **amend on my side**: the staging-`BufferRef` slow path must surface `ResourceExhausted` deterministically if the ring slot is full (this is the round-2 Con I raised under §3). | false | false |
| A1-P14 | agree | X4 doc-only. | false | false |

### A2 — Extensibility & Evolvability

| proposal_id | vote | rationale (cite rule id) | blocking | override_request |
|-------------|------|--------------------------|----------|-------------------|
| A2-P1 | agree | E6; version fields at handshake boundaries let reliability code fail with a specific `ProtocolVersionMismatch` rather than UB. | false | false |
| A2-P2 | agree | E4; same shape as Q6; future reliability knobs (quarantine, breaker) can ship with their level's factory. | false | false |
| A2-P3 | agree | E4; keeping `DepMode` closed preserves hot-path; opening `FailurePolicy` to string-ID resolution is neutral for A5 because the resolved `IFailurePolicy` remains a single function-pointer table. | false | false |
| A2-P4 | agree | E5 (mandatory — "never plan a big-bang rewrite without a fallback"); migration plan with feature flags directly reduces deployment-time reliability risk. | false | false |
| A2-P5 | agree | E2; BC policy is a reliability artefact for external Machine-Level authors. | false | false |
| A2-P6 | agree | E4; two-tier short-circuit (A2's own resolution) keeps hot path unchanged. Neutral for A5 provided the registry is init-time-only. | false | false |
| A2-P7 | abstain | No A5-specific interest; A9 YAGNI debate owns this. One-line reason: reserving a dead-code interface today does not affect round-1 reliability posture. | false | false |
| A2-P8 | agree | Rule Exceptions procedure in `04-agent-rules.md`; deviations should be recorded not hidden. Directly supports A5-P3 amendment (record decentralization as v2 Q). | false | false |
| A2-P9 | agree | E6; versioned trace schema allows `REPLAY` to refuse incompatible traces — reliability aid for A5-P5 chaos reproduction. | false | false |

### A3 — Functional Sanity & Correctness

| proposal_id | vote | rationale (cite rule id) | blocking | override_request |
|-------------|------|--------------------------|----------|-------------------|
| A3-P1 | agree | LSP + D7 + V5; `ERROR`/`CANCELLED` state is the terminal A5-P4 depends on. **Without A3-P1 my A5-P4 cannot land.** Severity blocker matches. | false | false |
| A3-P2 | agree | LSP; deterministic `submit()` return contract lets callers handle `WAIT`/`REJECT` from A5-P8's admission-pressure policy. Prefer option (b) (`Expected<>`-style) for extensibility. | false | false |
| A3-P3 | agree | V4 + R6 (failure scenario per critical path); closes the admission-path gap I named in round-1 §1 row 4. | false | false |
| A3-P4 | agree | R1 + DS3; resolves "eventually timeout" hand-wave and provides the `DEP_FAILED` event that A5-P4 relies on for non-idempotent Task failure propagation. Critical for A5. | false | false |
| A3-P5 | agree | DS4 + R4; sibling cancellation policy gives A5-P4's `idempotent=false` retry-suppression a deterministic fan-in semantic. Critical for A5. | false | false |
| A3-P6 | agree | V3 + G1; traceability is a reliability-audit artefact (every FR/NFR has a validation path = R6 baseline). | false | false |
| A3-P7 | agree | S3 + G3; preconditions at admission fold into A5-P8's `REJECT` path without adding a new branch. | false | false |
| A3-P8 | agree | X9 + G3; bounded DFS cost on admission keeps R4 admission-pressure deterministic. | false | false |
| A3-P9 | agree | G3 + LSP; deterministic SPMD ABI = idempotent retry at Task level (DS4). | false | false |
| A3-P10 | agree | D7 + G1; complete domain-to-exception mapping is a reliability guarantee for the Python caller. | false | false |
| A3-P11 | agree | G3; explicit `[ASSUMPTION]` on `complete_in_future` preserves honesty about a reliability-critical unresolved mechanism. | false | false |
| A3-P12 | agree | G3 + R4; `DrainInProgress` gives shutdown a deterministic contract. | false | false |
| A3-P13 | agree | DS3 + G3; causal-consistency assumptions must be explicit for A5's R4 / R6 scenarios to be reproducible. | false | false |
| A3-P14 | agree | LSP + V5; cleans up the "uniform state machine" claim without introducing new reliability surface. | false | false |
| A3-P15 | agree | G3 + X6; debug-mode `NONE`-misuse detection is a cheap reliability defence with zero prod cost. | false | false |

### A4 — Document Consistency

| proposal_id | vote | rationale (cite rule id) | blocking | override_request |
|-------------|------|--------------------------|----------|-------------------|
| A4-P1 | agree | D7; pure doc fix. | false | false |
| A4-P2 | agree | D7 + V5; broken anchor is a documentation-reliability defect. | false | false |
| A4-P3 | agree | D7 + G4; prose count error; trivial. | false | false |
| A4-P4 | agree | D7. | false | false |
| A4-P5 | agree | D7. | false | false |
| A4-P6 | agree | V2 + G5; ADR back-references aid reliability audits (decision rationale accessible from any view). | false | false |
| A4-P7 | agree | V5. | false | false |
| A4-P8 | agree | D7. | false | false |
| A4-P9 | agree | D7. | false | false |

### A6 — Security & Trust Boundaries

| proposal_id | vote | rationale (cite rule id) | blocking | override_request |
|-------------|------|--------------------------|----------|-------------------|
| A6-P1 | agree | S1; completeness of the boundary roster is a reliability-adjacent artefact (R6 must include each boundary). | false | false |
| A6-P2 | agree | S1/S2/R5; a flapping-credential peer must never be absorbed as a generic remote-failure by A5-P2's breaker. Required for sane failure classification. | false | false |
| A6-P3 | agree | S3 + R4; bounded payload parsing is a DoS-resistance reliability property. | false | false |
| A6-P4 | agree | S4/S5; fail-closed at init is the R5 posture for a multi-host cluster. A5 explicitly agrees that `Fatal` classification per §4.7.5 is the right surface — no graceful degradation to plaintext. | false | false |
| A6-P5 | agree | S2 + R4 (blast-radius bound); revocation on `SUBMISSION_RETIRED` aligns with A5's layer-lifecycle model. | false | false |
| A6-P6 | agree | S6; audit trail is a reliability-postmortem artefact (forensic reconstruction of a failure cascade). | false | false |
| A6-P7 | abstain | A5 neutrality; A9 YAGNI contest owns this in single-tenant mode. One-line reason: attestation is a trust property, not a reliability property; single-tenant posture does not change R1–R6 conformance. | false | false |
| A6-P8 | agree | S3 + R4; DLPack-import validation prevents malformed-tensor-induced undefined behaviour. Fuses naturally with A1-P11. | false | false |
| A6-P9 | agree | S1/S2/S3; tenant isolation enforcement is a reliability property (one tenant's failure cannot manifest as another's). | false | false |
| A6-P10 | agree | S2; scoped sinks avoid cross-tenant info-leak. | false | false |
| A6-P11 | agree | S2 + S4; reduces the attack surface, collaterally reducing the reliability-failure-mode surface. | false | false |
| A6-P12 | agree | S1/S3, **conditional on A9-P6 outcome** (moot if A9-P6 agreed). If REPLAY kept, signing the trace preserves R6 reproducibility. | false | false |
| A6-P13 | agree | S1 + R4 graceful-degradation; per-tenant rate-limit is a reliability quota. | false | false |
| A6-P14 | agree | G5 + S5; ADR for key lifecycle is doc-level, no functional impact. | false | false |

### A7 — Modularity & SOLID

| proposal_id | vote | rationale (cite rule id) | blocking | override_request |
|-------------|------|--------------------------|----------|-------------------|
| A7-P1 | agree | D6 (hard rule — blocker in A7's review); A5 perspective: a cycle between scheduler and distributed would conceal reliability state flows, so resolving it is a prerequisite for any circuit-breaker or quarantine work. | false | false |
| A7-P2 | agree | ISP + D4; merge with A9-P1 per synthesis. Role-split does not change reliability surface; neutral. | false | false |
| A7-P3 | agree | D2. Neutral for A5. | false | false |
| A7-P4 | agree | D3/D5; distributed payloads with distributed/ module is cleaner; neutral for A5. | false | false |
| A7-P5 | agree | D2 + ADR-008 alignment; `distributed_scheduler` should not reach into scheduler internals (the round-1 R4/R5 partial-failure semantics depend on crisp cross-module contracts). | false | false |
| A7-P6 | agree | D3/D5; extracting composition/ module narrows `runtime/` to reliability-adjacent concerns (wiring, elision). | false | false |
| A7-P7 | agree | D2/D6 consistency. Neutral. | false | false |
| A7-P8 | agree | D7. Neutral. | false | false |
| A7-P9 | agree | D7. Neutral. | false | false |

### A8 — Testability & Observability

| proposal_id | vote | rationale (cite rule id) | blocking | override_request |
|-------------|------|--------------------------|----------|-------------------|
| A8-P1 | agree | X5; `IClock` is a prerequisite for A5-P5 chaos matrix (time-warp watchdog tests, skew injection for A5-P6). | false | false |
| A8-P2 | agree | X5 + DfT; driveable event loop is the deterministic-replay substrate for A5-P5 failure scenarios. | false | false |
| A8-P3 | agree | O3 + O5; latency histograms are what R4/R5 SLO-breach alerts must be keyed to. | false | false |
| A8-P4 | agree | O4 + X6; co-owned with A5-P6 (dump_state is the evidence surface for `SchedulerStalled`). | false | false |
| A8-P5 | agree | O5 + E3 + X8; externalized alerts let A5-P8's degradation policies surface operationally. | false | false |
| A8-P6 | agree | O1 + DS3; clock-sync contract is required for merged distributed traces to respect causality under A5-P5 cross-node fault tests. | false | false |
| A8-P7 | agree | X5 + DfT; co-owned with A5-P5 (A8-P7 = seam, A5-P5 = matrix + outcomes). | false | false |
| A8-P8 | agree | O3; AICore trace-upload closes the on-core observability gap for R4 leaf-level failure detection. | false | false |
| A8-P9 | agree | O3 + O5; silent profiling drops invalidate R6 budget validation — making them alertable is a reliability gain. | false | false |
| A8-P10 | agree | O2; structured logging improves R6 post-mortem quality. | false | false |
| A8-P11 | agree | X5; HAL contract test suite running against sim + onboard is a reliability safeguard against sim-drift. | false | false |
| A8-P12 | agree | O1 + X9; stable `PhaseId`s let A5-P6's deadman timer key off a documented phase. | false | false |

### A9 — Simplicity (KISS / YAGNI)

| proposal_id | vote | rationale (cite rule id) | blocking | override_request |
|-------------|------|--------------------------|----------|-------------------|
| A9-P1 | agree | G2; drop submit overloads; merges with A7-P2 per synthesis. Neutral for A5. | false | false |
| A9-P2 | abstain | A2 vs A9 contest; A5-neutral provided the `SINGLE_THREADED + SPLIT_COLLECTION` subset retained still supports A5-P6 watchdog cadence. One-line reason: deployment-mode count does not affect R1–R6 directly. | false | false |
| A9-P3 | abstain | A2 vs A9 contest; A5-neutral provided that a partial-failure collective (round-2 Con above) maps cleanly to `FailurePolicy`. One-line reason: collective location is an interface question, not a reliability question, **but** if A9-P3 lands I want a follow-up ADR specifying how Orchestration-composed collectives handle partial-failure. | false | false |
| A9-P4 | agree | G2 + DRY; drop `Kind` discriminant. Neutral for A5. | false | false |
| A9-P5 | agree | DRY; unify admission enums. Helps A5-P8 avoid defining a third enum. | false | false |
| A9-P6 | abstain | A9 vs A8 vs A6 contest; A5 slightly leans against (REPLAY is a reliability-testing vehicle) but accepts A5-P5 can commit to `FUNCTIONAL`-only. One-line reason: R6 is reachable with `FUNCTIONAL` alone if A8-P2 driveable event-loop lands. | false | false |
| A9-P7 | agree | G2. Neutral for A5. | false | false |
| A9-P8 | agree | G2 + SRP; moving artifact obligation out of runtime doc does not affect R1–R6. | false | false |

### A10 — Scalability & Data Flow

| proposal_id | vote | rationale (cite rule id) | blocking | override_request |
|-------------|------|--------------------------|----------|-------------------|
| A10-P1 | agree | P4; sharded `producer_index` default is reliability-adjacent via scalability; no R* impact. | false | false |
| A10-P2 | agree | R5 + P4; accept synthesis merge into A5-P3 (co-owned). Critical for R5 in multi-Pod. | false | false |
| A10-P3 | agree | DS6; per-element consistency table is a reliability audit artefact. | false | false |
| A10-P4 | agree | DS1; stateful/stateless classification documents where persistence-recovery deviations (Deviation 2) apply. | false | false |
| A10-P5 | agree | P6; merged into A1-P6 per synthesis. Neutral for A5. | false | false |
| A10-P6 | agree | R5; sub-second `NodeLost` detection is a direct reliability win. **Must specify hysteresis interaction with A5-P2 breaker and A5-P9 QUARANTINED** (see round-2 Con §3 and tensions §7). | false | false |
| A10-P7 | agree | P4; sharded TaskManager with single-shard default two-tier path. Neutral for A5. | false | false |
| A10-P8 | agree | DS7 + P6; ownership diagram is a reliability-audit artefact. | false | false |
| A10-P9 | agree | DS6 + DS4; gating `WorkStealingPartitioner` against `RETRY_ELSEWHERE` resolves a non-idempotency hazard I would otherwise have raised. | false | false |
| A10-P10 | agree | P6 + DS6; capacity + layout guidance for `producer_index` is the logical partner to A1-P1. | false | false |

**Vote totals (for this review):** agree = 95; disagree = 0; abstain = 5. Blocking = 0.

---

## 7. Cross-aspect tensions (round-2 additions)

Round-1 tensions carry forward; the table below adds only the tensions newly observed in round 2.

| pair | with_proposal | preferred_resolution |
|------|---------------|----------------------|
| A5 vs A3 | A5-P4 + A3-P1 + A3-P4 + A3-P5 | **Hard dependency.** A5-P4 (`idempotent` flag) cannot land without A3-P1 (`ERROR` state) as the terminal, A3-P4 (`DEP_FAILED`) as the fan-out, and A3-P5 (sibling cancellation) as the aggregation semantic. Vote these as a bundle in round 3; if any one fails, A5-P4 must be re-scoped. |
| A5 vs A8 | A5-P5 ⇔ A8-P7 | **Merge roles.** A8-P7 owns the `IFaultInjector` seam (sim-only compile target); A5-P5 owns the mandatory scenario matrix + DR drill cadence. Neither proposal is a duplicate; they compose. |
| A5 vs A8 | A5-P6 ⇔ A8-P4 | **Merge evidence surface.** Accept synthesis register entry: deadman-age becomes a field in `dump_state().scheduler`. Watchdog alert externalized via A8-P5 `AlertRule`. |
| A5 vs A10 | A5-P3 ⇔ A10-P2 | **Merge and phase.** v1 = deterministic fail-fast (A5-P3 option B); v2 = decentralized quorum (A10-P2). Record v2 as a Q entry with trigger condition (cluster size ≥ N or availability SLO ≥ 99.9%). |
| A5 vs A10 | A5-P2 + A5-P9 vs A10-P6 | **Unified peer-health state machine.** The fast-fail heartbeat (A10-P6), per-peer circuit breaker (A5-P2), and Worker `QUARANTINED` (A5-P9) must compose into a single per-peer and per-Worker state machine with one set of transitions. Resolution: round-3 authors a joint section in `modules/distributed.md` §"Peer Health" + `02-logical-view/03-worker.md` §"Worker Health". Otherwise livelock risk. |
| A5 vs A6 | A5-P2 vs A6-P2 | **Auth-failure routes to breaker.** `ErrorCode::AuthenticationFailed` must flip the breaker to `OPEN` with a longer `cooldown_ms` than a generic remote failure (e.g., 10× multiplier) so a credential-rollout glitch does not produce retry storms. |
| A5 vs A6 | A5-P1 vs A6-P13 | **Backoff + rate-limit compose.** Per-tenant rate-limit (A6-P13) is an admission-time shed; exponential backoff (A5-P1) is a failure-time delay. They are orthogonal; no conflict but must both be named in A8-P5's alert taxonomy. |
| A5 vs A9 | A5-P9 vs A9 general | **Defend.** A9 will repeat the YAGNI attack on `QUARANTINED`; counter: it is the minimum R3 compliance at Worker granularity, severity already reduced to `low`, FSM cost = one enum value invisible to the selector. I will not concede further. |
| A5 vs A9 | A5-P5 vs A9-P6 | **Committed position.** If A9-P6 lands (ship FUNCTIONAL only), A5-P5 commits to FUNCTIONAL-only reproduction and a Q-record for REPLAY-based reproduction as v2. If A9-P6 fails, A5-P5 prefers REPLAY as the deterministic vehicle with A6-P12 signature enforcement. |
| A5 vs A1 | A5-P2 hot-path | **Confirmed two-tier.** Round-1 resolution stands: steady-state `CLOSED` reduces to a `thread_local` TSC gate; full state transitions only on the failure edge. Amended A5-P2 text above now makes this normative. |

---

## Merge Register — acceptance of synthesis-proposed merges involving A5

| Merge into | Absorbed | Decision | Reason |
|------------|----------|----------|--------|
| A5-P3 | A10-P2 | **Accept** (co-ownership) | Same architectural issue (Pod coordinator SPOF / decentralization). A5-P3 amended to v1 fail-fast + v2 decentralize roadmap; A10 co-owner on v2. |
| A5-P6 | A8-P4 | **Accept** (pair) | Watchdog needs a diagnostic surface; `dump_state()` is that surface. Keep both proposal ids; they converge in implementation. |
| A1-P6 | A10-P5 | **Accept** (not A5 primary but touches A5 via `remote_timeout_ms` / chunked-reassembly interaction) | Bounded REMOTE_SUBMIT is a reliability property as well as a performance one. |
| A1-P14 | A10-P10 | **Accept** (not A5 primary) | `producer_index` capacity + layout doc; no reliability objection. |
| A7-P2 | A9-P1 | **Accept** (not A5 primary) | Role-split + single `submit()`; no reliability objection. |
| A6-P8 | A1-P11 (partial) | **Accept** (not A5 primary) | Per-arg validation fits inside per-arg budget; reliability win (S3). |

No merges are rejected.

---

## 8. Stress-Attack on emerging consensus

*Skipped — round 3 only.*

---

## 9. Status

- **Satisfied with current design?** partially (unchanged from round 1).
- **Open items expected in next round:**
  - **Bundle-vote A3-P1 / A3-P4 / A3-P5 with A5-P4** — verify the hard dependency holds.
  - **Merged A5-P3 ⊕ A10-P2** — round-3 stress-attack on the v1 fail-fast option (can every surviving peer actually surface `CoordinatorLost` within `heartbeat_timeout_ms`?).
  - **Merged A5-P5 ⇔ A8-P7** — round-3 stress: can the chaos matrix be executed under `FUNCTIONAL`-only if A9-P6 lands?
  - **Merged A5-P6 ⇔ A8-P4** — round-3 stress: deadman cadence vs. A1 hot-path discipline (K=16 cycles sufficient?).
  - **Unified peer-health state machine** (A5-P2 + A5-P9 + A10-P6 + A6-P2) — needs a single authoritative diagram in round 3.
  - **A5-P7 / A5-P8 / A5-P10** — expect low-controversy convergence unless A9 flags admission-pressure enum as YAGNI.
  - **A5-P9** (QUARANTINED) — defended against expected A9 YAGNI attack.
