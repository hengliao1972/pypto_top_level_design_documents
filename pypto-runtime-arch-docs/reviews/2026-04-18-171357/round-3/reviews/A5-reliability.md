# Aspect A5: Reliability & Fault Tolerance — Round 3 (Stress Round)

## Metadata

- **Reviewer:** A5
- **Round:** 3 (MANDATORY STRESS ROUND)
- **Target:** `/data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/`
- **Runtime Mode:** yes
- **Timestamp (UTC):** 2026-04-19T00:00:00Z

---

## 1. Rubric Checks

Target docs are unchanged between rounds 2 and 3 (round-3 is reviewer-only; Phase-5 edits have not applied). Row results therefore match round 2. The round-3 delta column records whether the *amended* proposal text from round-2 §5 still closes each gap under stress-attack replay (§8).

| # | Check | R1 result | R2 amended-text closes? | R3 stress-attack verdict |
|---|-------|-----------|-------------------------|--------------------------|
| 1 | Timeouts on every remote / IPC / device call | Weak | A5-P7 + A5-P6 close it | **holds** (6.2.4 replay; see §8) |
| 2 | Retries bounded with exp. backoff + jitter | Fail | A5-P1 closes it | **holds** (6.2.2 replay with `RETRY_ELSEWHERE`; see §8) |
| 3 | Circuit breakers for failing dependencies | Fail | A5-P2 + A5-P9 close it | **holds** with one amendment — unified peer-health FSM must be authored day 1 (§8, §4 A5-P11) |
| 4 | Graceful degradation per dependency | Weak | A5-P8 + A6-P13 + A10-P6 close it | **holds** (6.2.2 CONTINUE_REDUCED; §8) |
| 5 | No SPOF on critical path | Weak | A5-P3 ⊕ A10-P2 closes via v1 fail-fast + v2 roadmap | **holds** (coordinator-kill replay; §8) |
| 6 | Idempotency under retry | Weak | A5-P4 + A5-P10 + A3-P4 + A3-P5 close it | **holds as bundle** — A5-P4 must not land if any of A3-P1/A3-P4/A3-P5 is rejected (hard dep; §5) |
| 7 | Failure-mode test plan | Weak | A5-P5 ⇔ A8-P7 close it | **holds under option (iii)** for A9-P6 — FUNCTIONAL + RecordedEventSource is sufficient for the R6 matrix (§8, §6 disputed vote) |

---

## 2. Pros

Round-1 and round-2 pros stand. Round 3 adds these stress-round observations:

- **Every hot-path-touching A5 proposal carries a two-tier bridge in its R2 amendment.** A5-P2 (breaker CLOSED-state single-load gated by `thread_local` TSC), A5-P6 (deadman write every Kth cycle, K=16), A5-P7 (`poll(zero)` unchanged), A5-P9 (`IDLE`-only selector filter). Re-reading these against A1's R2 review (`round-2/reviews/A1-performance.md`) — A1 voted `agree` on each with explicit acknowledgement of the two-tier. Stress posture: **no new hot-path costs hidden in the amendments.**
- **A5-P3 ⊕ A10-P2 merge produces a testable v1 contract.** Option-B fail-fast (`CoordinatorLost` within `heartbeat_timeout_ms`) is deterministic, measurable against `06-scenario-view.md:100-114`, and shippable without a lease protocol. The v2 roadmap as a Q-record (with named trigger `cluster_size ≥ N` or `availability_slo ≥ 99.9%`) satisfies R5 today *and* establishes an A2-E5 migration plan for tomorrow — both rubrics satisfied without over-building.
- **A3-P1 (`ERROR` state, severity blocker) unblocks A5-P4 and A5-P9.** The terminal state these two A5 proposals reference now exists under an agreed-unanimous proposal. The hard dependency I flagged in round-2 §7 is satisfied pre-stress.
- **A5-P5 ⇔ A8-P7 role-split (chaos matrix owner A5; `IFaultInjector` seam owner A8) survives the A9-P6 contest.** Option (iii) for A9-P6 (ship FUNCTIONAL only, keep REPLAY enum + scaffolding) preserves every A5-P5 matrix row under `FUNCTIONAL + RecordedEventSource` — the A8-P2 driveable event-loop is the reproducibility substrate, not REPLAY data. A5 no longer opposes A9-P6 in option (iii).
- **A5-P6 ⇔ A8-P4 pair produces a single diagnostic surface.** Round-3 re-reading of `round-2/reviews/A8-testability.md` confirms `dump_state().scheduler.deadman_age_ns` is the load-bearing field; A8 voted `agree` on both A5-P6 and A8-P4 with the pair annotation. This removes my round-2 concern that watchdog + dump would diverge.

---

## 3. Cons

Round-2 cons stand. Round 3 adds these stress-round residuals:

- **Unified peer-health state machine is owed but un-authored.** Round-2 §7 (A5) identifies A5-P2 + A5-P9 + A10-P6 + A6-P2 as a livelock-risk quartet if each proposal lands without a single FSM diagram. The four owners voted `agree` but no single owner was charged with the joint diagram. Under 6.2.2 replay (§8) I prove the livelock is latent if A10-P6's hysteresis is authored inconsistently with A5-P2's `HALF_OPEN` cooldown. Raising a new proposal A5-P11 in §4 to charge A5 with the joint diagram.
- **A1-P13's slow-path `args_blob` still lacks a reliability contract.** Round-2 Con 1 remains open: if the staging `BufferRef` alloc fails for `args_size > 1 KiB`, the surfaced error class is not named. A1 voted `agree` on my raising this; no amendment landed. Raising this as a follow-up in §4 (A5-P12).
- **A6-P2 ↔ A5-P2 contract — "`AuthenticationFailed` trips the breaker with 10× cooldown" — is only recorded in my R2 §7 table, not in any target doc.** Under 6.2.2 replay with a flapping credential (§8), A5-P2's amendment counts auth-fail ×K; the "×K multiplier" (my §5 amended text used "10×") needs a home. Either A6-P2 or A5-P2 section must land the cross-reference. Minor; calling it out in §4 (A5-P13).
- **A9-P3 (drop `IHorizontalChannel` collectives) is agreed but leaves `FailurePolicy::CONTINUE_REDUCED` for Orchestration-composed collectives unspecified.** My round-2 §3 Con 6 flagged this; no peer proposed the follow-up. The A9-P3 amendment says "follow-up ADR"; the ADR is not scheduled. Raising in §4 as A5-P14 (Q-record in `09-open-questions.md`).
- **Heartbeat cadence vs. coordinator fail-fast bound is implicit.** A5-P3 option-B says "every surviving peer surfaces `CoordinatorLost` within `heartbeat_timeout_ms`." A10-P6 reduces detection floor to ~800 ms. If `heartbeat_timeout_ms` default remains 10 s (as documented at `modules/transport.md:399`), A5-P3 option-B's fail-fast bound is *10 seconds*, which is worse than A10-P6's own target. The two values must be reconciled so that `CoordinatorLost` surfaces within `max(heartbeat_interval_ms × hysteresis, 1 s)` — not within the coarser lifeness heartbeat. Minor but material; addressed via §5 amendment to A5-P3.

---

## 4. Proposals

This round raises four NEW follow-up proposals (A5-P11 … A5-P14). All are doc-only and hot-path `none`.

| id | severity | summary | affected_docs | hot_path_impact | trade_offs | sanity_test |
|----|----------|---------|---------------|-----------------|------------|-------------|
| A5-P11 | medium | Author a single unified peer-health FSM section co-owned by A5 (breaker) + A10 (heartbeat) + A6 (auth) | `modules/distributed.md` new §3.5 "Peer Health"; cross-link from `02-logical-view/03-worker.md` §"Worker Health" | none | Prevents livelock between A5-P2 HALF_OPEN, A5-P9 QUARANTINED, A10-P6 heartbeat hysteresis, A6-P2 auth-fail trip | Walk a flapping-peer trace through all four state machines; assert at most one transition per tick and no oscillation under jittered RTT |
| A5-P12 | low | Name `ResourceExhausted` as the failure class for staging `BufferRef` allocation failure on A1-P13's slow path | `modules/hal.md` §`args_blob` (the 1 KiB fast-path section); `modules/scheduler.md` §5 error table | none | Closes round-2 Con 1 residual; one sentence | Inject a failed staging alloc; assert error class = `ResourceExhausted`, not a generic fault |
| A5-P13 | low | Record the "`AuthenticationFailed` trips breaker with `breaker_auth_fail_weight` multiplier (default 10)" cross-reference in **both** A6-P2 text and A5-P2 text | `modules/distributed.md` §3.4 (A5-P2 landing site) and §3.x (A6-P2 handshake); `modules/runtime.md` config | none | Prevents credential-rollout retry storms from defeating A5-P2's `fail_threshold` budget | Run a 5-minute credential rollout that returns `AuthenticationFailed` for 30 s; assert the breaker trips within `fail_threshold / 10` attempts (not `fail_threshold`) |
| A5-P14 | medium | Q-record the "Orchestration-composed collective partial-failure → FailurePolicy mapping" as a v2 ADR item triggered by A9-P3 landing | `09-open-questions.md` new Q entry; cross-link from `modules/distributed.md` `IHorizontalChannel` section | none | Pre-empts a round-1 R4 Con becoming a post-v1 defect; follow-up ADR is scoped, not written today | Doc-lint: Q entry exists and names the three policies (`FAIL_SUBMISSION`, `DEGRADED_CONTINUE`, `RETRY_PARTITION_ELSEWHERE`) |

### Proposal detail

#### A5-P11: Unified peer-health FSM

- **Rationale:** R3 (circuit breaker) + R5 (no SPOF) + R4 (graceful degradation) converge on one per-peer state. Without a unified FSM, A5-P2 (CLOSED/HALF_OPEN/OPEN), A5-P9 (QUARANTINED Worker), A10-P6 (SUSPECT/LOST heartbeat), and A6-P2 (TRUSTED/AUTH_FAIL) live as four parallel FSMs; their transitions interleave and can livelock (§8 6.2.2 replay shows a HALF_OPEN probe contending with a SUSPECT heartbeat for the same `cooldown_ms` window).
- **Edit sketch:**
  - File: `modules/distributed.md`
  - Location: new §3.5 "Peer Health"
  - Delta: one FSM diagram with states `{HEALTHY, SUSPECT, OPEN, HALF_OPEN, QUARANTINED, LOST, AUTH_REVOKED}` and a transition table keyed on `{heartbeat_timeout, breaker_trip, probe_success, probe_fail, auth_fail, auth_rotate}` events. Explicitly: a peer in `LOST` skips the breaker entirely; a peer in `HALF_OPEN` is invisible to the heartbeat monitor; `AUTH_REVOKED` routes to `OPEN` with 10× cooldown (lands A5-P13).
- **Trade-offs:**
  - Gain: eliminates cross-proposal livelock; one diagram replaces four.
  - Give up: ~1 page of doc work; one owner assigned (A5 primary; A10 + A6 review).
- **Sanity test:** For each of five scripted fault traces (peer flap, credential rollout, partition, GC pause, recovering worker), the unified FSM must produce exactly one transition per tick and terminate in a steady state.

#### A5-P12: Name `ResourceExhausted` for slow-path `args_blob` alloc failure

- **Rationale:** R4 — every dependency-unavailability must have a named response. A1-P13 creates a slow path that inherits its failure class from an unspecified location.
- **Edit sketch:**
  - File: `modules/hal.md` §`args_blob` (1 KiB fast-path section) — add "If `args_size > 1 KiB`, the staging `BufferRef` is allocated from the outstanding-ring slab; allocation failure surfaces as `ErrorCode::ResourceExhausted` and the Submission is rejected via A5-P8's admission-pressure policy."
  - File: `modules/scheduler.md` §5 error table — add row for `ResourceExhausted` with admission-time origin.
- **Trade-offs:**
  - Gain: fills round-2 Con 1 residual; aligns with A5-P8.
  - Give up: one table row, one sentence.
- **Sanity test:** Force staging-slab alloc to fail; assert the Submission returns `ResourceExhausted` (not a generic fault or a silent retry).

#### A5-P13: `breaker_auth_fail_weight` cross-ref

- **Rationale:** R3 + S1/S2 interaction. Without an explicit multiplier, a routine credential rollout (auth temporarily fails) consumes the same `fail_threshold` budget as a real peer fault, producing false positives during planned events.
- **Edit sketch:**
  - File: `modules/distributed.md` §3.4 (A5-P2 landing site) — add "`AuthenticationFailed` is counted with weight `breaker_auth_fail_weight` (default 10); all other failure classes count with weight 1."
  - File: `modules/distributed.md` §3.x (A6-P2 handshake) — cross-link to A5-P2 §3.4.
  - File: `modules/runtime.md` `CircuitBreaker` struct — add `uint32_t auth_fail_weight = 10;`.
- **Trade-offs:**
  - Gain: credential-rollout robustness; no new enum.
  - Give up: one config field, two doc sentences.
- **Sanity test:** Simulate a 30-second credential rollout on one peer; assert the breaker trips to OPEN within `fail_threshold / auth_fail_weight` rollover attempts, not after `fail_threshold` individual failures.

#### A5-P14: Q-record for A9-P3 collective-partial-failure follow-up

- **Rationale:** R4 — when collectives move from `IHorizontalChannel` to Orchestration-level composition (A9-P3), the partial-failure semantics of the composed collective inherit `FailurePolicy` from the parent Submission. The mapping is not documented and will be a post-v1 defect if left implicit.
- **Edit sketch:**
  - File: `09-open-questions.md`
  - Location: new Q entry after the existing Q5.
  - Delta: "Q<next>: For Orchestration-composed collectives (post A9-P3), how does a single child-peer failure map to `FailurePolicy::{ABORT_ALL, CONTINUE_REDUCED, RETRY_ELSEWHERE}`? Trigger: first production deployment using a non-trivial collective. Candidates: `FAIL_SUBMISSION`, `DEGRADED_CONTINUE` (with partial reduction), `RETRY_PARTITION_ELSEWHERE`."
  - File: `modules/distributed.md` `IHorizontalChannel` section — cross-link to the Q entry.
- **Trade-offs:**
  - Gain: pre-empts a known R4 gap; follow-up ADR is triggered, not pre-written.
  - Give up: one Q entry, one cross-link.
- **Sanity test:** Doc-lint: Q entry exists and names at least three candidate policies.

---

## 5. Revisions of own proposals

| id | action | amended_summary | reason |
|----|--------|-----------------|--------|
| A5-P1 | defend | — | No peer objection; hot-path `none`; stress replay (§8 6.2.2) holds. |
| A5-P2 | defend | — | Round-2 amendment (thread-local TSC gate + auth-fail weighted ×K) survives the 6.2.2 stress replay. A5-P11 covers the FSM-unification concern. |
| A5-P3 | amend | Round-2 amendment already adopted v1 fail-fast + v2 Q-record. **Round-3 addition:** the fail-fast bound is `min(heartbeat_timeout_ms, coordinator_liveness_timeout_ms)` where `coordinator_liveness_timeout_ms` defaults to `3 × heartbeat_interval_ms` (so A10-P6's sub-second target applies for coordinator loss). Record the relationship `coordinator_liveness_timeout_ms < heartbeat_timeout_ms` as an invariant in `modules/transport.md:399` next to the existing `heartbeat_timeout_ms > heartbeat_interval_ms × 2` invariant. | Reconciles the tension noted in §3 Con 5 (coordinator fail-fast bound must not be coarser than peer-loss detection). Zero hot-path impact; failure-path only. |
| A5-P4 | defend (bundle) | — | Hard dependency on A3-P1 + A3-P4 + A3-P5, all of which are `agreed` in R2 synthesis. Stress replay §8 6.2.1 confirms the bundle is coherent. |
| A5-P5 | defend (option iii) | — | Option (iii) for A9-P6 keeps REPLAY enum + scaffolding; A5-P5's chaos matrix commits to `FUNCTIONAL + RecordedEventSource` reproduction for v1, with REPLAY scaffolding reserved for day-2 forensic replay. A8-P7's `IFaultInjector` seam is the mechanism; A5-P5 is the contract. No revision. |
| A5-P6 | defend (pair) | — | Pair with A8-P4 stands. Stress replay §8 6.2.1 shows `dump_state().scheduler.deadman_age_ns` visibility is sufficient for watchdog alerting within `2 × scheduler_watchdog_ms`. K=16 keeps A1 hot-path. |
| A5-P7 | defend | — | No peer objection. |
| A5-P8 | defend | — | Amendment text stands; ties into A8-P5 `AlertRule` per round-2 amendment. |
| A5-P9 | defend | — | A9-P2 stress (§8 disputed-vote row) does not impugn QUARANTINED; the state is load-bearing for R3 at Worker granularity and costs one enum value outside the hot path. |
| A5-P10 | defend | — | Pure doc-lint contract; no regressions. |

---

## 6. Votes on peer proposals (round 3 updates)

Round 2 already recorded full per-proposal votes (95 agree, 0 disagree, 5 abstain from A5). Round 3 revises only the three disputed rows where the stress-round instruction requires a re-vote. All other 104 R2 votes stand unchanged and are not repeated here.

### Disputed proposals — R3 re-vote

| proposal_id | R2 vote | R3 vote | rationale (cite rule id) | blocking | override_request |
|-------------|---------|---------|--------------------------|----------|-------------------|
| A2-P7 | abstain | **agree** | Synthesis §"A2-P7" recommends reframing as a Q-record in `09-open-questions.md` naming the two future async-policy axes (no new interface in v1). Under that framing the proposal becomes a pure doc-extensibility (E5 migration plan) entry and does not affect R1–R6. A5 agrees with the Q-record framing. If the owner's final text reintroduces a speculative interface in v1, I revert to abstain. | false | false |
| A9-P2 | abstain | **agree** | Synthesis §"A9-P2" recommends keeping a single `IEventLoopDriver` test seam + closed enums + appendix of v2 extensions. Under that framing: (a) A8-P2 (driveable event-loop) lands; (b) A5-P5's chaos matrix can drive the loop via `step()` + `RecordedEventSource` without REPLAY; (c) closed enums protect the hot path. The `SINGLE_THREADED + SPLIT_COLLECTION` subset is preserved, which is the only subset my A5-P6 watchdog cadence needs. **Flip from abstain to agree.** | false | false |
| A9-P6 | abstain | **agree (option iii)** | Synthesis's recommended landing zone is option (iii): ship FUNCTIONAL-only but keep REPLAY enum + scaffolding so A6-P12 and A8 can land day 2. Under option (iii), A5-P5 retains both FUNCTIONAL (present day) and REPLAY (day 2) reproduction vehicles; `RecordedEventSource` + A8-P2 driveable loop are sufficient for the R6 chaos matrix rows named in round-1. R6 is reachable. **Flip from abstain to agree, conditional on option (iii).** If the final text chooses option (i) (full defer including enum) A5 would disagree because A5-P5 would lose its day-2 REPLAY upgrade path without an enum. | false | false |

### Note on votes that remain unchanged

All 104 non-disputed votes stand as recorded in round-2 §6. No new disagrees. No blocking. No override_request.

---

## 7. Cross-aspect tensions (round-3 additions)

Round-1 and round-2 tensions carry forward. Round 3 adds:

| pair | with_proposal | preferred_resolution |
|------|---------------|----------------------|
| A5 vs A1 | A5-P11 (unified peer-health FSM) | Doc-only; zero hot-path impact. Existing two-tier of A5-P2 + A5-P9 is preserved inside the unified FSM — HEALTHY is the single-load fast path. |
| A5 vs A10 | A5-P3 amended (coordinator fail-fast bound) | `coordinator_liveness_timeout_ms < heartbeat_timeout_ms` invariant aligns A5-P3 with A10-P6 — same detection floor, no conflict. |
| A5 vs A6 | A5-P13 (`breaker_auth_fail_weight`) | Cross-reference between A5-P2 and A6-P2 sections; single config field. |
| A5 vs A9 | A5-P14 (collective partial-failure Q) | A9-P3 lands now; the Q-record is the ADR trigger. A9 agrees to Q-record framing per standard E5/G2 compromise. |
| A5 vs A2 | A5-P14 | Q-record pattern is exactly A2-P7's own preferred resolution — A5 using A2's machinery to pre-empt a future defect. |
| A5 vs A8 | A5-P11 | A8-P4 `dump_state()` surfaces every FSM state field; unified FSM fits inside existing A8-P4 schema with one new `peer_health` sub-struct. No new A8 machinery required. |

---

## Merge register — no new merges

No new merges or splits in round 3. The six R2 merges (A1-P6 ← A10-P5; A1-P14 ← A10-P10; A5-P3 ← A10-P2 as split; A5-P6 ↔ A8-P4 as pair; A7-P2 ← A9-P1; A1-P11 ← A6-P8a) stand.

---

## 8. Stress-Attack on emerging consensus

**Methodology.** For each A5-adjacent agreed proposal I (a) attempt the strongest attack my aspect can mount against the **amended text from round-2 §5**, (b) replay at least one scenario from `06-scenario-view.md` step by step under the amendment, (c) record a verdict `holds | breaks | uncertain`. Every row cites a scenario-view line range. Hot-path-touching proposals additionally note whether the two-tier bridge survives.

Scenarios used for replay:
- **S1 = 6.2.1 AICore Hang** (`06-scenario-view.md:83-98`)
- **S2 = 6.2.2 Remote Node Crash** (`06-scenario-view.md:100-114`)
- **S3 = 6.2.3 Task Slot Pool Exhaustion** (`06-scenario-view.md:116-128`)
- **S4 = 6.2.4 Network Timeout** (`06-scenario-view.md:130-142`)

| proposal_id | stress_attempt | scenario (file:line) | verdict |
|-------------|----------------|----------------------|---------|
| A5-P1 (backoff+jitter) | Under S2 `RETRY_ELSEWHERE`, if `base_ms · 2^n` exceeds a downstream SLO budget on retry n=4, the scheduler silently violates R4's latency budget without an alert. | `06-scenario-view.md:113` (6b RETRY_ALTERNATE) | **holds** — amendment caps at `max_ms = 2000` and exposes total retry budget = `Σ min(max_ms, base·2^n)(1+jitter)`; A8-P5 `AlertRule` surfaces an SLO-breach alert when budget is close to expiry. Two-tier: failure path only. |
| A5-P2 (per-peer breaker) | Under S2 with a flapping credential, `AuthenticationFailed` is counted with weight 1 and the breaker never trips until `fail_threshold` generic failures accumulate, during which time the cluster hammers the flapping peer. | `06-scenario-view.md:106-113` | **breaks → amendment** — A5-P13 (new) lands `breaker_auth_fail_weight = 10` in both A5-P2 and A6-P2 sections. With the amendment: **holds**. Two-tier preserved (CLOSED-state single `thread_local` TSC gate). |
| A5-P3 ⊕ A10-P2 (coordinator fail-fast + v2 roadmap) | SIGKILL the coordinator at S2 step 3. Surviving peers wait for `heartbeat_timeout_ms = 10 s` before surfacing `CoordinatorLost` — worse than A10-P6's sub-second target. | `06-scenario-view.md:105-109` | **breaks → amendment** — A5-P3 R3 amendment (§5) adds `coordinator_liveness_timeout_ms < heartbeat_timeout_ms` invariant and defaults it to `3 × heartbeat_interval_ms`. With the amendment: **holds** at ~800 ms. |
| A5-P4 (`idempotent` flag) | If A3-P4 (`DEP_FAILED` propagation) is amended to eagerly retry instead of propagating, A5-P4's `idempotent=false` path has no terminal. | `06-scenario-view.md:95-96` (S1 step 6-7, parent → ERROR) | **holds** — A3-P4 is `agreed` unanimously in R2; terminal `ERROR` state exists via A3-P1. Hard dependency bundle holds. Stress: vote A5-P4 only after A3-P1/A3-P4/A3-P5 pass Phase-5 edit application — flagged to parent. |
| A5-P5 ⇔ A8-P7 (chaos matrix + `IFaultInjector`) | Under A9-P6 option (i) (defer REPLAY + PERFORMANCE), the "deterministic reproduction" row of the chaos matrix loses its primary vehicle; A5-P5 would be uncommitted on reproduction fidelity. | `06-scenario-view.md:130-142` (S4 timeout; reproduction of network jitter) | **holds under option (iii)** — RecordedEventSource + A8-P2 driveable event-loop + A8-P1 injectable `IClock` = deterministic replay of S4 without REPLAY data. A5 votes A9-P6 agree **conditional on option (iii)** — see §6. |
| A5-P6 ⇔ A8-P4 (watchdog + dump_state) | If the deadman write is skipped for >K=16 consecutive cycles due to a pathological event burst, the deadman age appears stale even under a live scheduler. | `06-scenario-view.md:96-97` (S1 Host propagation; implies a live scheduler loop) | **holds** — K=16 at a 100 µs/cycle budget bounds the "stale but live" window to 1.6 ms, well below `scheduler_watchdog_ms = 250`. Two-tier preserved (write only every Kth cycle). A8-P4 `dump_state()` exposes `deadman_age_ns` and `last_cycle_ns` so an operator can distinguish "burst" from "stall". |
| A5-P7 (`IMemoryOps::wait` timeout) | Under S4 RDMA write, if the backend's internal retry adds 200 ms, the caller's `wait(100ms)` returns `TransportTimeout` but the hardware write is still pending — leading to a duplicate-delivery scenario. | `06-scenario-view.md:136` (S4 step 2 poll returns no completion) | **holds** — dedup window (`modules/distributed.md:406`, ring size 4096) catches the duplicate at the consumer side. A5-P10 DS4 contract requires each `REMOTE_*` handler to document its idempotency class; the duplicate write is covered. Two-tier preserved (`poll(zero)` unchanged). |
| A5-P8 (admission-pressure policy) | Under S3 task-slot exhaust, if the operator configures `admission_pressure_policy = DEFER` with no upper deferral bound, deferred submissions accumulate until memory exhaust. | `06-scenario-view.md:122-127` | **holds** — the amendment ties each policy value to an `AlertRule` per A8-P5; `DEFER` mode includes a bounded deferral queue with `max_deferred` config (implicit in the amendment's "default = current behaviour" clause, which is `REJECT`; any non-default must configure the bound). Verified against A3-P7 preconditions landing. |
| A5-P9 (QUARANTINED Worker state) | A9 YAGNI attack: FSM state proliferation; an IDLE-with-probe implementation would suffice. | `06-scenario-view.md:83-98` (S1 AICore hang → Chip Scheduler marks core FAILED) | **holds** — without QUARANTINED, an IDLE-with-probe Worker is re-assigned by `WorkerManager::on_assign` *before* the probe completes; the Worker fails the next real Task, not a probe. Costs 1 enum value invisible to the `state == IDLE` selector (two-tier preserved). R3 Worker-granularity is the load-bearing reason. Attack fails. |
| A5-P10 (per-handler DS4 class) | A future `REMOTE_COLLECTIVE_FLUSH` handler is added without a DS4 annotation; without CI doc-lint enforcement the annotation is skipped silently. | `06-scenario-view.md:130-142` (S4; duplicate delivery class depends on handler) | **holds** — amendment includes a doc-lint rule that every `REMOTE_*` handler declaration must carry `idempotency ∈ {safe, at-most-once, at-least-once}`. CI failure on missing annotation makes the attack moot. |
| A5-P11 *(new)* — unified peer-health FSM | Under S2 with one peer in HALF_OPEN, a heartbeat miss from that peer could oscillate between HALF_OPEN and SUSPECT if the FSM is not authored. | `06-scenario-view.md:106-113` | **holds with amendment** — this proposal exists to hold; without it, the livelock is reachable. |
| A5-P12 *(new)* — name `ResourceExhausted` on A1-P13 slow path | Slow-path alloc fail returns unnamed error → Python sees a generic RuntimeError. | `06-scenario-view.md:116-128` (S3 analog) | **holds with amendment** — the new proposal ensures the error has a name. |
| A5-P13 *(new)* — `breaker_auth_fail_weight` | (See A5-P2 row.) | `06-scenario-view.md:106-113` | **holds with amendment** (see A5-P2). |
| A5-P14 *(new)* — Q-record for A9-P3 partial-failure | Post-v1 Orchestration-composed collective has no documented partial-failure contract. | `06-scenario-view.md:100-114` (generalised to a collective) | **holds with amendment** — Q-record is the trigger; no day-1 cost. |
| — cross-proposal — A5-P2 + A5-P9 + A10-P6 + A6-P2 | Replay S2 step-by-step: Node₁ crashes → heartbeat misses → A10-P6 SUSPECT → A5-P2 breaker increments → A6-P2 handshake fails → A5-P9 QUARANTINED fires on Worker hosting the remote stub. Without A5-P11 the transitions can oscillate (SUSPECT ↔ HEALTHY) under a recovering network with credential churn. | `06-scenario-view.md:100-114` whole scenario | **breaks without A5-P11; holds with A5-P11.** Verdict drives the new proposal's severity to medium. |

**Global stress-attack verdict:** all 10 A5-owned proposals (A5-P1 … A5-P10) hold under amendment. Four new follow-up proposals (A5-P11 … A5-P14) are raised, three at low severity (doc-only cross-references) and one at medium (A5-P11 — the unified peer-health FSM). No A5 proposal requires `blocking=true`. No A5 override_request.

**A1 hot-path audit (from A5's side).** Re-reading every hot-path-touching A5 proposal against A1's round-2 review and the two-tier bridges:

| Proposal | Hot-path cost (steady state) | Two-tier bridge preserved? |
|----------|------------------------------|----------------------------|
| A5-P2 | one relaxed-atomic load on a cache-resident counter, amortised via `thread_local` TSC gate | **yes** |
| A5-P6 | 1 store per K=16 cycles on a cache-resident slot | **yes** |
| A5-P7 | zero change for `poll(zero)`; timeout branch is slow-path only | **yes** |
| A5-P9 | one enum compare only on failure edge; `IDLE` selector filter unchanged | **yes** |
| A5-P11 *(new)* | zero; doc-only | **N/A (doc)** |
| A5-P12 *(new)* | zero; doc-only | **N/A (doc)** |
| A5-P13 *(new)* | zero; one compile-time-constant config read at breaker-update path (already slow-path) | **N/A (slow path)** |

No new hot-path cost introduced by R3 amendments. No A1 veto expected.

---

## 9. Status

- **Satisfied with current design?** **yes, conditional on:**
  1. A5-P11 lands (unified peer-health FSM); without it the A5-P2 + A5-P9 + A10-P6 + A6-P2 quartet can livelock under the S2 flapping-peer replay.
  2. A5-P3 R3 amendment lands (`coordinator_liveness_timeout_ms < heartbeat_timeout_ms` invariant); without it the fail-fast bound exceeds A10-P6's detection floor.
  3. A9-P6 lands under **option (iii)** per the synthesis's recommended landing zone. A5 flips to disagree if final text chooses option (i) (full defer including enum).
  4. A5-P4 bundle-vote passes: A3-P1 + A3-P4 + A3-P5 all land.
- **Open items expected in next round (if any):**
  - Phase-5 edit application for A5-P11 through A5-P14 (owners: A5 primary, with A10 + A6 + A9 as reviewers for their respective cross-refs).
  - Verification that A9-P6 final text matches option (iii); re-vote trigger if not.
  - Joint diagram authorship for A5-P11 (charge A5 + A10 + A6).
  - No new disputed proposals raised in round 3 from A5's aspect; all A5 verdicts are `holds` or `holds with (my own) amendment`.
- **Convergence call:** from A5's side, convergence is achievable in round 3 under the four conditions above. No blocking objections. No A1 veto triggered. No A5 override_request.
