# Aspect A7: Modularity & SOLID — Round 2

## Metadata

- **Reviewer:** A7
- **Round:** 2
- **Target:** `/data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/`
- **Runtime Mode:** yes
- **Timestamp (UTC):** 2026-04-19T00:00:00Z

## 1. Rubric Checks (delta vs. Round 1)

No new evidence since Round 1 changed any rubric result. The R1 grid still stands:

| # | Check | Result | Notes for Round 2 |
|---|-------|--------|-------------------|
| 1 | SRP — single reason to change per module | Weak | Still pending A7-P4 (transport payloads) and A7-P6 (runtime/ split). |
| 2 | DIP — high-level modules depend on abstractions only | Weak | Pending A7-P3 (core/ ↔ hal/) and A7-P5 (distributed → ISchedulerLayer). |
| 3 | DAG — no cycles | **Fail** | A7-P1 is the blocker; no peer disputed the cycle. |
| 4 | ISP — role-focused interfaces | Weak | A7-P2, once amended to header-only split, converges with A9-P1 (merge register). |
| 5 | Cohesion | Weak | Same drivers as #1. |
| 6 | Coupling — narrow, stable seams | Weak | A7-P7 (forward-decl contract) + A7-P5 close the DAG-truthfulness gap. |

Round 1 evidence (file:line citations) is unchanged; see `round-1/reviews/A7-modularity.md §1`.

## 2. Pros (additions from peer signal)

- **A2-P5 (BC policy table)** directly reinforces A7's D2/D4 posture: per-interface stability class is the mechanism that turns today's narrow interfaces into *durably* narrow interfaces. (D2, D4, E2)
- **A8-P1 (`IClock`) + A8-P2 (`RecordedEventSource`/`step()`) + A8-P11 (HAL contract test)** complete the X5 story at module boundaries: each is a narrow, test-double-friendly seam, not a new plugin registry. (D2, X5)
- **A9 (Simplicity)** converges with A7 on several points: A9-P1 (drop submit overloads) is the implementer-side mirror of A7-P2; A9-P3 (remove collectives from `IHorizontalChannel`) is pure ISP; A9-P4/P5/P7 reduce duplicate concepts (D7 / DRY). Cross-aspect net: A7's ISP/DRY items are *agreed by both* aspects that normally pull against each other.

## 3. Cons (additions from peer signal)

- **A2-P6 (pluggable `IDistributedProtocolHandler`)** is presented as *one extra seam* (good) but its interaction with A7-P4 (move payloads to `distributed/`) must be pinned: the handler registry must sit in `distributed/`, not `transport/`, or SRP regresses. See Tensions §7.
- **A6-P6 (audit trail) / A8-P5 (Prometheus sink)** both propose *new sinks on top of profiling sinks*. Modularity-wise this is clean only if `IAuditSink` / `IMetricsSink` are first-class sibling interfaces to `ILogSink` in `profiling/` (not `runtime/`). Called out as tension.
- **A5-P2 (per-peer circuit breaker) + A5-P9 (QUARANTINED worker state)** land in two different modules (`distributed/` and `scheduler/`). That is *correct* SRP (peer health ≠ worker health), but the synthesis conflict register does not flag this; risk is that a future review conflates the two. Clarify in the BC policy (A2-P5).

## 4. Proposals — New in Round 2

No new proposals in Round 2. Round-1 proposals A7-P1 … A7-P9 are revised in §5.

## 5. Revisions of own proposals

| id | action | amended_summary (if amend/split) | reason |
|----|--------|----------------------------------|--------|
| A7-P1 | **defend** | — | No peer disputed the cycle; A2 (E4), A10 (P4), A5 (R5), A8 (X5) all benefit from a strict DAG. Hard rule D6 governs; `hot_path_impact: none`. |
| A7-P2 | **amend** | Split `ISchedulerLayer` at the **header level only** into role-focused headers (`ISchedulerWiring`, `ISchedulerSubmit`, `ISchedulerCompletion`, `ISchedulerLifecycle`, `ISchedulerScope`); **retain the aggregator `ISchedulerLayer` for implementers** via multiple inheritance. Absorb A9-P1: the aggregator gets exactly one `submit(SubmissionDescriptor)` entry (no `submit_group` / `submit_spmd` virtual overloads); build-time helpers `build_single/build_group/build_spmd` live in `runtime/`. | (a) Merge register accepted — A7-P2 ⟵ A9-P1 (see §"Merge Register Response"). (b) Same vtable count as today (A1 concern absorbed); (c) A9 ceremony objection absorbed since no new classes at the implementer side; (d) every consumer compiles against the narrowest role header — ISP win. `hot_path_impact: none`. |
| A7-P3 | **defend** | — | No peer objection. A1 implicitly welcomes it (core/ becomes leaf — one fewer transitive link on hot-path users). `hot_path_impact: none`. |
| A7-P4 | **amend** | Move `RemoteSubmitPayload` / `RemoteDepNotifyPayload` / `RemoteCompletePayload` / `RemoteDataReadyPayload` / `RemoteErrorPayload` / `HeartbeatPayload` from `transport/` to `distributed/include/distributed/protocol_payloads.h`; `transport/` keeps only `MessageHeader` + `MessageType` + framing. **Add:** explicitly anchor A2-P6's registry, A6-P3's bounds table, and A1-P6's `REMOTE_BINARY_PUSH` + descriptor-template cache inside `distributed/`, not `transport/`, so SRP is preserved across all three peer-owned follow-ons. | Peer reviews added three payload-layer responsibilities (pluggable dispatch, bounds validation, staging binaries); each belongs to "protocol semantics", not "framing/reliability". Extending A7-P4 to explicitly scope them prevents future SRP drift. |
| A7-P5 | **defend** | — | No peer objection; A2 and A5 both benefit (testability of `distributed_scheduler` without a full `scheduler/` link). ADR-008 text makes the claim already — A7-P5 just enforces it in the code layout. |
| A7-P6 | **amend** | Prefer a strict **sub-namespace `runtime::composition`** (single include tree, no cross-includes into runtime orchestration code) as the minimum edit; escalate to a separate compiled module only if `registry/descriptor/parser` grows a dependency set distinct from `runtime/`'s. | A9 tension on "new top-level module" is real; sub-namespace gives identical D1/D5 wins without CMake-library ceremony. Preserves A9 while satisfying D3. |
| A7-P7 | **defend** | — | No peer objection. Pure documentation + header-discipline fix; closes a DAG-truthfulness gap against D6. `hot_path_impact: none`. |
| A7-P8 | **defend** | — | No peer objection. D7 naming hygiene; single definition in `core/scope.h`, `memory/` consumes. |
| A7-P9 | **defend** | — | A3-P10 (complete Python exception mapping) supersedes the scope but does not resolve the specific `MemoryError` duplication; keep A7-P9 as the minimum rename obligation and let A3-P10 expand the mapping table. |

### Merge Register Response (A7-relevant entries)

| Merge | Verdict | Justification |
|-------|---------|---------------|
| A7-P2 ⟵ A9-P1 | **accept** | Already called out as a tension in R1 §7. Combined form: one `submit(SubmissionDescriptor)` entry + role-focused headers over a single aggregator. Addresses ISP (A7) and G2 (A9) with one delta. |
| A1-P6 ⟵ A10-P5 | accept | A7 is not the owner but notes: payload sits in `distributed/` post-A7-P4. OK. |
| A1-P14 ⟵ A10-P10 | accept | Purely scheduler-internal placement doc. |
| A5-P3 ⟵ A10-P2 | accept | Both live in `distributed/`; single policy surface preferred (D4). |
| A5-P6 ⟵ A8-P4 | accept | `dump_state()` is the evidence surface for the watchdog; keep in `runtime/` + per-module `describe()`. |
| A6-P8 ⟵ A1-P11 (partial) | accept | DLPack validation fits the per-arg budget at `bindings/` entry; one boundary, two rubrics satisfied. |

## 6. Votes on peer proposals

Notation: cite rule IDs per `04-agent-rules.md`. `blocking=true` iff the objection rests on a hard rule from §04 (D-rules, R1-6, DS-rules, S-rules, V-rules, X-rules). A7 is non-A1; `override_request` may be set.

| proposal_id | vote | rationale (cite rule id) | blocking | override_request |
|-------------|------|--------------------------|----------|-------------------|
| A1-P1 | agree | X2 pre-sizing; local to `scheduler/`, no module impact. | false | false |
| A1-P2 | agree | X3 documented hold-time is a modularity hygiene win (narrow lock contract). | false | false |
| A1-P3 | agree | P2; presumes Function Cache lives in `hal/` and cross-peer Bloom in `distributed/` — both clean. | false | false |
| A1-P4 | agree | X4; doc-only in `modules/core.md §8`. | false | false |
| A1-P5 | agree | X9 doc-only; no D-impact. | false | false |
| A1-P6 | agree | P6; payload owner = `distributed/` per A7-P4 amendment. | false | false |
| A1-P7 | agree | X3/P1 local to `profiling/`; D5 preserved. | false | false |
| A1-P8 | agree | X2; local to `scheduler/`. | false | false |
| A1-P9 | agree | X4; local to `scheduler/`; fallback for >64 groups requested by A2 sanity. | false | false |
| A1-P10 | agree | P1 CI gate. | false | false |
| A1-P11 | agree | X9; also absorbs A6-P8 at the `bindings/` boundary. | false | false |
| A1-P12 | agree | X9/P5; LevelParam is appropriate scope. | false | false |
| A1-P13 | agree | P6/X9; tightens HAL contract narrowly (D4). | false | false |
| A1-P14 | agree | X4 doc-only. | false | false |
| A2-P1 | agree | E6; versions touch cold/init-path types. | false | false |
| A2-P2 | agree | E4/OCP — schema-registered per-level knobs match the Q6 pattern. | false | false |
| A2-P3 | agree | E4; `DepMode` stays closed (hot-path). Keeps ISP on extension axes. | false | false |
| A2-P4 | agree | E5 migration plan. | false | false |
| A2-P5 | agree | E2; **direct modularity win** — per-interface BC class. | false | false |
| A2-P6 | agree | E4/D4 — registry-dispatched handler replaces a central switch with a narrow seam. Must sit in `distributed/` (A7-P4 amendment). | false | false |
| A2-P7 | disagree | D3/G2 — declaring a dead `IAsyncTaskSchedulePolicy` before any extender violates SRP (ships an abstraction without a current reason to change); defer until Q11 resolves. | false | false |
| A2-P8 | agree | Discipline-only; Rule Exceptions procedure. | false | false |
| A2-P9 | agree | E6. | false | false |
| A3-P1 | agree | LSP, D7, V5 — state machine uniformity. | false | false |
| A3-P2 | agree | LSP/D7; prefer option (b) `std::expected<>` for OCP on future `AdmissionDecision` values. | false | false |
| A3-P3 | agree | V4/R6. | false | false |
| A3-P4 | agree | DS3/R1; `DEP_FAILED` is a single additive event-enum value — minimal D-surface. | false | false |
| A3-P5 | agree | R4/DS4 via `IResourceAllocationPolicy::on_child_error()` — clean policy seam. | false | false |
| A3-P6 | agree | V3/G1. | false | false |
| A3-P7 | agree | G3/S3; admission-only, does not widen `ISchedulerLayer`. | false | false |
| A3-P8 | agree | G3/X9; local to admission. | false | false |
| A3-P9 | agree | G3/LSP; function ABI contract. | false | false |
| A3-P10 | agree | D7; complementary to A7-P9. | false | false |
| A3-P11 | agree | G3. | false | false |
| A3-P12 | agree | G3/R4. | false | false |
| A3-P13 | agree | DS3/G3. | false | false |
| A3-P14 | agree | LSP/V5. | false | false |
| A3-P15 | agree | X6 debug-only; zero release cost. | false | false |
| A4-P1 | agree | D7/V5 naming. | false | false |
| A4-P2 | agree | D7 anchor fix. | false | false |
| A4-P3 | agree | D7/G4 count fix. | false | false |
| A4-P4 | agree | D7 order. | false | false |
| A4-P5 | agree | D7 glossary. | false | false |
| A4-P6 | agree | V2/G5 back-links. | false | false |
| A4-P7 | agree | V5/D7 — "Scheduler: AicoreDispatcher" matches A7's view that L0 is an `ISchedulerLayer` implementation. | false | false |
| A4-P8 | agree | D7 count. | false | false |
| A4-P9 | agree | D7 enumeration. | false | false |
| A5-P1 | agree | R2; failure-path only. | false | false |
| A5-P2 | agree | R3; per-peer state lives in `distributed/`, clean D1. | false | false |
| A5-P3 | agree | R5; prefer option B (deterministic fail-fast) for minimal D1/D5 impact in v1; A10-P2 absorbed. | false | false |
| A5-P4 | agree | DS4; one boolean on `TaskDescriptor`. | false | false |
| A5-P5 | agree | R6; `IFaultInjector` is sim-only symbol — zero release surface. | false | false |
| A5-P6 | agree | R5; paired with A8-P4 per merge register. | false | false |
| A5-P7 | agree | R1; narrow extension to `IMemoryOps`. | false | false |
| A5-P8 | agree | R4 policies. | false | false |
| A5-P9 | agree | R3; one enum state in `WorkerState`. | false | false |
| A5-P10 | agree | DS4 contract discipline. | false | false |
| A6-P1 | agree | S1 completeness. | false | false |
| A6-P2 | agree | S1/S2; handshake-time only. | false | false |
| A6-P3 | agree | S3; lives in `distributed/` after A7-P4. | false | false |
| A6-P4 | agree | S4/S5. | false | false |
| A6-P5 | agree | S2; narrow extension on `IMemoryOps`. | false | false |
| A6-P6 | agree | S6; `IAuditSink` must sit in `profiling/` sibling to `ILogSink` (D5). | false | false |
| A6-P7 | agree | S1/S3; `Attestation` field on `FunctionDesc` — minimal surface. | false | false |
| A6-P8 | agree | S3; absorbed into A1-P11 per merge register. | false | false |
| A6-P9 | agree | S1/S2; 4-byte field addition to `MessageHeader`. | false | false |
| A6-P10 | agree | S2; scoped sinks match A7's D4 narrow-surface preference. | false | false |
| A6-P11 | agree | S2/S4. | false | false |
| A6-P12 | abstain | Contingent on A9-P6: if REPLAY is deferred, this is moot; if kept, it is a straightforward S1/S3 addition to the REPLAY engine only. | false | false |
| A6-P13 | agree | S1 DoS; admission-path accounting, not a new branch. | false | false |
| A6-P14 | agree | G5/S5 ADR. | false | false |
| A8-P1 | agree | X5; textbook DIP seam (D2). | false | false |
| A8-P2 | agree | X5; test-only; preserves D1/D5. | false | false |
| A8-P3 | agree | O3; stats struct enumeration is a per-module internal concern — fits D5. | false | false |
| A8-P4 | agree | O4/X6; `describe()` on each module honors D4 (narrow diagnostic surface per module). | false | false |
| A8-P5 | agree | O5/E3/X8; `IMetricsSink` sits next to `ILogSink`/`IAuditSink` in `profiling/` (D5). | false | false |
| A8-P6 | agree | O1; `IClockSync` co-located with `IClock` (A8-P1) in `hal/`. | false | false |
| A8-P7 | agree | X5/R6; sim-only symbol. | false | false |
| A8-P8 | agree | O3; HAL capability bit is the right seam. | false | false |
| A8-P9 | agree | O3/O5. | false | false |
| A8-P10 | agree | O2. | false | false |
| A8-P11 | agree | X5; contract tests are the keystone of D2 HAL discipline. | false | false |
| A8-P12 | agree | O1 stable IDs. | false | false |
| A9-P1 | agree | D4/G2 — merged into A7-P2 per merge register (single `submit(SubmissionDescriptor)`). | false | false |
| A9-P2 | agree | D4/D5/G2 — removing `IEventCollectionPolicy`/`IExecutionPolicy` interfaces with one extender each reduces fat interfaces and speculative seams. A2 pushback mitigated by A2-P2's schema-registered pattern: re-introduce via new code, not by keeping an unused interface today. | false | false |
| A9-P3 | agree | D4 — `IHorizontalChannel` shrinks to `send/recv/poll/transfer`. Collectives, if revived, land as `ICollectiveOps` sibling (OCP via new interface, not amendment). | false | false |
| A9-P4 | agree | G2/D7 — `Kind` is a vestigial discriminant. | false | false |
| A9-P5 | agree | D7/DRY. | false | false |
| A9-P6 | agree | G2 — reduces `hal/sim/` SRP burden; PERFORMANCE/REPLAY deferred to v1.x. | false | false |
| A9-P7 | agree | D5/D7. | false | false |
| A9-P8 | agree | D1 (scope of runtime design doc). | false | false |
| A10-P1 | agree | P4/DS6; sharded lock is an internal `scheduler/` concern (D1 clean). | false | false |
| A10-P2 | agree | Merged into A5-P3. | false | false |
| A10-P3 | agree | DS6; doc in `07-cross-cutting-concerns.md`. | false | false |
| A10-P4 | agree | DS1; doc in `03-development-view.md`. | false | false |
| A10-P5 | agree | P6; merged into A1-P6. | false | false |
| A10-P6 | agree | R5/P4; transport-level fast-fail piggyback is a narrow addition. | false | false |
| A10-P7 | agree | P4; two-tier (`shards=1` default) preserves D1 simplicity for single-node. | false | false |
| A10-P8 | agree | DS7 diagram. | false | false |
| A10-P9 | agree | DS6 — correct incompatibility declaration; adds a minimal assignment-log seam in `distributed/`. | false | false |
| A10-P10 | agree | P6/X4; merged with A1-P14. | false | false |

**Vote summary:** 101 votes cast — 99 agree, 1 disagree (A2-P7), 1 abstain (A6-P12). Zero blocking objections from A7 in this round. Zero override requests (A1 has not overused the veto yet).

## 7. Cross-aspect tensions

| pair | with_proposal | preferred_resolution |
|------|---------------|----------------------|
| A7 vs A9 | A7-P2 (amended) + A9-P1 | Merged: single `submit(SubmissionDescriptor)` + role-focused *headers* (not new classes) over aggregator `ISchedulerLayer`. Zero new implementer-side surface; narrowest consumer surface per site. |
| A7 vs A9 | A7-P6 (amended) | Drop top-level new module; use sub-namespace `runtime::composition` with its own include tree. No new CMake library. |
| A7 vs A2 | A2-P6 (protocol registry) ↔ A7-P4 | `IDistributedProtocolHandler` registry lives in `distributed/`, not `transport/`. A2's OCP win + A7's SRP win both hold only under this placement. |
| A7 vs A2 | A2-P7 (reserved async seam) | A7 disagrees: reserving a dead interface violates D3 (single reason to change). Compromise: record as a *shape sketch* in `09-open-questions.md` (Q11) rather than in `09-interfaces.md`. Matches A9-P2 bridge. |
| A7 vs A1 | A1-P13 (HAL `args_blob` contract) | Two-tier HAL contract: `args_size ≤ 1 KiB` fast path keeps `IExecutionEngine::submit_kernel` signature unchanged; large args pass `BufferRef` handle. Narrow interface preserved (D4). |
| A7 vs A5 | A5-P2 (peer circuit breaker) + A5-P9 (worker QUARANTINED) | Two distinct concerns in two distinct modules (`distributed/` peer health vs `scheduler/` worker health). Preserve the split; document both in the BC policy table (A2-P5). |
| A7 vs A6 | A6-P6 (audit) + A8-P5 (metrics) | `IAuditSink` and `IMetricsSink` are sibling interfaces to `ILogSink` in `profiling/` — one module owns all sinks (D5). Do not create `audit/` or `metrics/` top-level modules. |
| A7 vs A8 | A8-P4 (`dump_state`/`describe()`) | Each module exposes `describe()` behind its own facade; `Runtime::dump_state()` is pure aggregation. No cross-module state snooping; D4 narrow diagnostic surface preserved. |
| A7 vs A3 | A3-P2 option (b) (`expected<>`) | Prefer (b): widening the return channel keeps `ISchedulerLayer::submit` a single-method signature for future `AdmissionDecision` values without ABI break (E2/D4). |
| A7 vs A10 | A10-P2 (decentralized coordinator) merged into A5-P3 | Option B (fail-fast) in v1 keeps `distributed/` narrow; quorum-based decentralization lands as new code in v2 via the BC policy, not by mutating today's `ClusterView`. |

## 9. Status

- **Satisfied with current design?** no (unchanged from R1)
- **Open items expected in next round (R3 stress-attack candidates from A7's vantage):**
  - **A7-P1** (blocker, D6): round 3 should stress-test alternative cycle breaks (e.g., keep `distributed/` depending on `scheduler/` only and promote `SchedulerEvent` into `core/` — what breaks?).
  - **A7-P2 + A9-P1** (merged): stress the header-only split against a hypothetical 11th ISchedulerLayer consumer that needs `Wiring + Completion` but not `Submit` — does the aggregator still pull in the full vtable?
  - **A7-P4 extension**: stress whether A2-P6's handler registry + A6-P3's bounds + A1-P6's binary staging can all co-exist in `distributed/` without recreating a fat `DistributedFacade`.
  - **A7-P6 (amended sub-namespace)**: stress against A4-P6 (ADR back-references) — does the sub-namespace decision need its own ADR?
  - **A2-P7** (the only A7-disagree): stress from A2's side: is there a Q11-specific extender that would justify the reservation?
