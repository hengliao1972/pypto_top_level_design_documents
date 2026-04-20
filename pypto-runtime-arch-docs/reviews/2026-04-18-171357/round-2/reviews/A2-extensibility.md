# Aspect A2: Extensibility & Evolvability — Round 2

## Metadata

- **Reviewer:** A2
- **Round:** 2
- **Target:** `/data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/`
- **Runtime Mode:** yes
- **Timestamp (UTC):** 2026-04-19T00:00:00Z

## 1. Rubric Checks

Re-run against the current design state (no R1 amendments have yet landed — the design docs are unchanged). Results are therefore close to R1, but peer findings from R1 sharpen the evidence.

| # | Check | Result | Evidence (file:line) | Rule(s) |
|---|-------|--------|----------------------|---------|
| 1 | Interfaces designed for change; OCP at known change points | Weak | `02-logical-view/02-scheduler.md:21–57` pluggable `ISchedulerLayer`, strategy policies present; but `IHorizontalChannel` conflates transport + collectives per A9's R1 finding (§A9-P3), and `ISchedulerLayer` is a "fat interface" per A7-P2 (OCP at wrong granularity) | E1, §1.2 OCP, §1.4 ISP |
| 2 | New functionality added by new code, not modifying existing | Weak | `modules/transport.md` `MessageType` enum is closed; adding a message still requires editing the enum + every switch site; `register_factory` is init-only — same gap A6-P11 identifies for security gating | E1, E4, §1.2 OCP |
| 3 | Versioning on protocols, APIs, schemas, configs | Weak | Only `HandshakePayload.protocol_version` and sandboxed trace `version` tag exist (`modules/distributed.md` §4.1, `modules/observability.md`); `SubmissionDescriptor`, `TaskDescriptor`, `FunctionDescriptor`, `MessageHeader`, stats/metrics schema have no version field — confirmed by A5-P4 (adding `idempotent` field will break old consumers), A6-P9 (needs `logical_system_id` → MessageHeader bump), A6-P7 (FunctionDesc attestation field), A8-P3 (stats schema) | E6 |
| 4 | Configuration over hardcoding (constants, thresholds externalized) | Weak | Many thresholds still inline or code-adjacent: heartbeat interval/threshold (`modules/distributed.md:407`), circuit-breaker defaults (A5-P2 new), alert thresholds (A8-P5), shard count (A1-P2/A10-P1), `max_outstanding_submissions`, `BatchedExecutionPolicy.max_batch` (A1-P12). R1 surfaced many of these; the remedy is a consolidated `LevelOverrides`/policy config with schema-registered extension (A2-P2). | E3 |
| 5 | Migration plan for transitions; incremental, reversible | Fail | No migration / versioning transition plan document; only ad-hoc Q-records (Q5 coordinator election, Q6 overrides). Cross-cutting changes flagged in R2 (`SubmissionDescriptor` version bump, coordinator failover path, registry dynamic-registration) have no declared phasing. | E5 |
| 6 | Backward-compatibility guarantees stated | Fail | No document names what is stable vs unstable, nor deprecation windows. R1 peer proposals that add fields (A5-P4, A6-P7, A6-P9, A5-P9 QUARANTINED state, A5-P8 new degradation states) each independently widen contracts with no central BC policy to shelter consumers. | E2, E6 |

Net verdict unchanged from R1: strong OCP at the macro layer level, weak versioning/BC/migration discipline at the data-contract level. R1 peer reviews provide additional ammunition (multiple additive field proposals that all need E6 clearance) — strengthens the case for A2-P1 and A2-P5.

## 2. Pros

Unchanged from R1; noting the ones peer reviews now corroborate.

- **OCP/DIP at layer boundary is clean** — `ISchedulerLayer` + factory/registry pattern (`modules/scheduler.md:36–41`). A7's R1 finding does not challenge this pattern, only its granularity (A7-P2 splits into role interfaces — fully compatible with A2).
- **Strategy patterns for admission, partitioning, placement** (`modules/scheduler.md:60–96`, `modules/distributed.md:55`) — the right OCP seams for the right change axes. §1.2 OCP.
- **Handshake carries `protocol_version`** (`modules/distributed.md` §4.1) — E6 existence proof; A2-P1 extends the pattern uniformly.
- **Sandboxed trace has schema `version`** (`modules/observability.md` sandboxed section) — another E6 existence proof; A2-P9 extends to the primary trace.
- **Level Controller and Coordinator separation supports in-place topology change** — fits E1/E5 if a migration plan is written (A2-P4).
- **Open Questions ledger exists** — provides a ready vehicle for A2-P4/A2-P5 without inventing new document structure.

## 3. Cons

R1 list stands. Two items strengthened by R2 reading:

- **Additive-field pressure is high and unbudgeted (E6).** R1 peers propose at least six additive schema changes: `idempotent` on `TaskDescriptor` (A5-P4), `logical_system_id` on `MessageHeader` (A6-P9), attestation fields on `FunctionDescriptor` (A6-P7), `QUARANTINED` Worker state (A5-P9), degradation states (A5-P8), signed-trace metadata (A6-P12). Without A2-P1's uniform version discipline, these proposals ship six independent partial fixes and old consumers break six different ways.
- **Test/observability seams want extension points (E1/E4).** A8-P1 (`IClock`), A8-P2 (`IEventLoopDriver`/`step()`), A8-P7 (`IFaultInjector`), A8-P5 (alert-rule externalization), A5-P5 (chaos hook) all require OCP seams the current design has not declared. A2 is the natural aspect to require they be declared consistently rather than ad-hoc per feature.
- **A7 cycle-break and role-split reveal interface coupling (§1.4 ISP, §1.5 DIP).** A7-P1/-P2/-P3/-P5 confirm A2's R1 finding that `ISchedulerLayer` is doing too much; I treat A7's interventions as complementary to my proposals (A2-P6, A2-P8) rather than competing.

## 4. Proposals (NEW in this round)

No new proposals. My R1 set (A2-P1..P9) covers the R2 themes. Revisions are in §5.

## 5. Revisions of own proposals

| id | action | amended_summary | reason |
|----|--------|-----------------|--------|
| A2-P1 | defend | — | Core E6 finding; reinforced by six additive-field R1 proposals that otherwise ship uncoordinated partial fixes. HPI=`none`: version field is 2 bytes inside an already-present 32-byte header, zero hot-path cost; I commit to fixed offset + compile-time constant so A1 need not veto. |
| A2-P2 | amend | "Phase as closed-for-v1 (ADR-backed enum list) with a schema-registry transition plan for v2; v1 carries a reserved `LevelOverrides.extension_map` byte placeholder so v2 adoption is non-breaking." | Synthesis §A2-P2 suggested "closed for v1, schema-registered for v2 with Q6 as vehicle." I accept. The amendment preserves E6 (version-bump path exists) without paying E3 complexity cost in v1. |
| A2-P3 | defend | — | Already the pre-compromise: `DepMode` closed, extension-point enums open. Synthesis marks this as likely agreement. No change. |
| A2-P4 | defend | — | Synthesis routes fast-track; no peer dissent identified. E5 requires a migration plan to exist at all — the doc gap is concrete. |
| A2-P5 | defend | — | Fast-track per synthesis. R1 peer proposals that add contract fields (see §3) cannot land safely without this policy; every R2 vote on those assumes A2-P5 lands. |
| A2-P6 | amend | "Declare `IDistributedProtocolHandler` as an abstract boundary inside `distributed_scheduler` only. v1 delivers exactly one concrete backend (current protocol) with dispatch inlined via CRTP or `final` + devirtualization. No plugin registry in v1; registry lands when a second backend is named. Preserve the abstraction in headers so v2 registration is additive." | Synthesis amendment explicit. This addresses A1 (zero virtual dispatch on hot path) and A9 (YAGNI — no registry until needed) while preserving OCP at the right boundary. E1 satisfied via the declared seam; E4 satisfied because the registry is a later additive change. |
| A2-P7 | amend | "Reframe as a Q-record (new Q-entry in `09-open-questions.md`) naming the two future policy axes (event-collection mode, execution-policy plug-in) and the Q that will resolve each. No interface added in v1." | Synthesis explicitly suggested this. Reduces speculative-interface charge from A9 while keeping E5 (named future evolution step). Eliminates extensibility/simplicity collision at source. |
| A2-P8 | defend | — | Fast-track. `known-deviations.md` already exists; adding rows for closed `MessageType`, init-only registry, closed `LevelOverrides` is purely documentary, no HPI, supports E1-exception discipline per `04-agent-rules.md` §"Rule Exceptions." |
| A2-P9 | defend | — | Fast-track. The sandboxed trace already has `version`; primary trace lacks it. Directly unblocks A6-P12 (signed REPLAY) and A8-P12 (stable PhaseIds) which both presume a versioned trace envelope. |

## 6. Votes on peer proposals

Voting discipline:

- HPI-touching proposals: vote agrees only if a two-tier path is sketched in R1 or the R2 synthesis amendment provides one.
- `blocking=true` only when the objection rests on a hard rule from `04-agent-rules.md`.
- `override_request` is set when a non-A1 aspect (including this one) believes an A1 hot-path concern is over-applied and should be debated explicitly.

### 6.1 Votes on A1 (Performance)

| proposal_id | vote | rationale (cite rule id) | blocking | override_request |
|-------------|------|--------------------------|----------|-------------------|
| A1-P1 | agree | Pre-sizing `producer_index` is bounded and doc-first; E3 (size as `LevelParam`) supports externalization. No OCP regression. | false | false |
| A1-P2 | agree | Cap + shard count as `LevelParam` directly meets E3; complements A2-P1 on uniform config discipline. | false | false |
| A1-P3 | agree | HEARTBEAT presence is additive, LRU policy swappable — OCP-friendly; requires A2-P5's BC policy for HEARTBEAT opcode rollout. | false | false |
| A1-P4 | agree | One-time layout doc in `modules/core.md` §8 (per synthesis amendment); does not bind logical view. E4-neutral. | false | false |
| A1-P5 | agree | Budgets are prose SLOs; adding new event-loop stages later remains additive (E4). | false | false |
| A1-P6 | agree | Capping REMOTE_SUBMIT + staging channel is bounded; new message type requires E6 handshake — aligns with A2-P1. Merge of A10-P5 in. | false | false |
| A1-P7 | agree | Per-thread local sequence is internal; no contract change. E1/E4 unaffected. | false | false |
| A1-P8 | agree | Pre-sizing bounded; externalize limits via `LevelParam` (E3). | false | false |
| A1-P9 | disagree | Bitmask encoding at ≤64 groups creates an E1 ceiling. The synthesis requires a split-group fallback; until A1 commits the 2×64 fallback in the edit sketch (not prose), this proposal encodes a hard cap the roadmap already expects to exceed. Amend to include the multi-bitmap fallback spec. | false | true |
| A1-P10 | agree | CI gate, no contract impact; E4-neutral. | false | false |
| A1-P11 | agree | Per-arg marshaling budget is a boundary contract — A2 views this as the right place to land A6-P8 validation (synthesis merge). | false | false |
| A1-P12 | agree | Agreed conditional on synthesis amendment: `max_batch` applies only when Batched explicitly selected; Dedicated default preserves closed-policy simplicity. E3 satisfied. | false | false |
| A1-P13 | agree | Bound is a HAL contract tightening; requires E2 deprecation window for any existing consumer assuming larger blob. A2-P5 provides the policy. | false | false |
| A1-P14 | agree | Doc-only; pairs with A10-P10 (absorbed). | false | false |

### 6.2 Votes on A3 (Functional Sanity)

| proposal_id | vote | rationale (cite rule id) | blocking | override_request |
|-------------|------|--------------------------|----------|-------------------|
| A3-P1 | agree | Adding `ERROR`/`CANCELLED` to Task FSM is additive; E4 pattern. Forces A2-P5 deprecation of old-state assumptions → good. | false | false |
| A3-P2 | agree | Clarifies return contract; prefer option (b) `Result<SubmissionId, AdmissionError>` — richer extension point than `throw`. §1.2 OCP preserved. | false | false |
| A3-P3 | agree | Failure-scenario addition is purely documentary; no contract regression. | false | false |
| A3-P4 | agree | New `DEP_FAILED` event is additive (E4). Must land under A2-P1 version discipline so old observers don't crash on the new event tag. | false | false |
| A3-P5 | agree | Sibling-cancellation policy is exactly the kind of named policy A2-P7 Q-records. Landing A3-P5 also motivates A2-P7. | false | false |
| A3-P6 | agree | Traceability matrix supports E5 (migrations must preserve traced requirements). | false | false |
| A3-P7 | agree | Admission precondition validation is additive and supports E2 (stable preconditions documented). | false | false |
| A3-P8 | agree | Accept subject to synthesis amendment: detection confined to debug mode with O(\|V\|+\|E\|) and cap via `max_intra_edges` `LevelParam` (E3). | false | false |
| A3-P9 | agree | SPMD index/size delivery is a stability commitment — directly strengthens E6 for SPMD ABI. | false | false |
| A3-P10 | agree | Exception-mapping table supports BC when new exceptions are added later (E2/E6). | false | false |
| A3-P11 | agree | Documentary marker; zero cost. | false | false |
| A3-P12 | agree | Concurrency contract is a contract — strengthens E2 stability. | false | false |
| A3-P13 | agree | Cross-node ordering assumption is a protocol-level contract; pairs with A2-P1 (version) and E6. | false | false |
| A3-P14 | agree | Small FSM addition, additive. | false | false |
| A3-P15 | agree | Debug-mode cross-check; E4-neutral. | false | false |

### 6.3 Votes on A4 (Document Consistency)

| proposal_id | vote | rationale (cite rule id) | blocking | override_request |
|-------------|------|--------------------------|----------|-------------------|
| A4-P1 | agree | Enum casing canonicalization — E6 naming stability precondition. | false | false |
| A4-P2 | agree | Broken anchor fix; blocker in A4's severity but trivial edit. | false | false |
| A4-P3 | agree | Prose count fix. | false | false |
| A4-P4 | agree | Reorder open questions; no contract change. | false | false |
| A4-P5 | agree | Glossary entries for event-loop plumbing; co-owned with A2-P9. If A9-P7 agrees, the `SourceCollectionConfig` glossary entry drops (per synthesis). | false | false |
| A4-P6 | agree | Cross-link views to ADRs — supports E5/E1 traceability. | false | false |
| A4-P7 | agree | Label unification (D7/DRY). | false | false |
| A4-P8 | agree | Sync Appendix-B count. | false | false |
| A4-P9 | agree | Expand glossary. | false | false |

### 6.4 Votes on A5 (Reliability)

| proposal_id | vote | rationale (cite rule id) | blocking | override_request |
|-------------|------|--------------------------|----------|-------------------|
| A5-P1 | agree | Backoff + jitter; policy parameters as config (E3). | false | false |
| A5-P2 | agree | Per-peer circuit breaker is an additive policy (E4); thresholds must be config-externalized (A2-P2 alignment). | false | false |
| A5-P3 | agree | Accept synthesis amendment: v1 fail-fast + Q-record, v2 decentralize. This is precisely E5 (incremental migration, named transition step). Absorbs A10-P2. | false | false |
| A5-P4 | agree | Adding `idempotent: bool` to `TaskDescriptor` requires E6 version bump — exactly A2-P1's vehicle. Vote presumes A2-P1 lands. | false | false |
| A5-P5 | agree | `IFaultInjector` is a textbook OCP seam; supports E1. Sim-only keeps hot path clean. | false | false |
| A5-P6 | agree | Watchdog is a policy-parameterized monitor (E3); pairs with A8-P4 (merge). | false | false |
| A5-P7 | agree | Adding `Timeout` on `IMemoryOps` is a contract widening — requires E2 deprecation window. Amend to land as a new overload `submit_with_timeout()` rather than replacing the existing signature. | false | false |
| A5-P8 | agree | Degradation specs are additive states (E4); requires A2-P1 version discipline for observers. | false | false |
| A5-P9 | agree | QUARANTINED state is additive (E4). | false | false |
| A5-P10 | agree | Per-REMOTE_* idempotency contract is stability commitment (E2); clarifies E6 scope boundaries. | false | false |

### 6.5 Votes on A6 (Security)

| proposal_id | vote | rationale (cite rule id) | blocking | override_request |
|-------------|------|--------------------------|----------|-------------------|
| A6-P1 | agree | Complete threat model; no contract change. | false | false |
| A6-P2 | agree | Node auth in HANDSHAKE is additive via `HandshakePayload` extension — this is exactly E6 (the `protocol_version` already exists). | false | false |
| A6-P3 | agree | Bounded parsing at trust boundary; synthesis amendment (single length-prefix guard per message) keeps HPI inside entry gate. E1 preserved. | false | false |
| A6-P4 | agree | TLS default on TCP; synthesis amendment explicit that RDMA path unaffected — two-tier honored. A2 supports: extension via config (E3) to downgrade for test-only. | false | false |
| A6-P5 | agree | Scoped rkey with per-submission rotation (synthesis amendment). Rotation cadence becomes a versioned policy — aligns E3/E6. | false | false |
| A6-P6 | agree | `IAuditSink` is OCP/E1 extension seam done right. | false | false |
| A6-P7 | agree | Attestation fields are additive — require A2-P1 FunctionDescriptor version bump. Synthesis amendment (config-gated behind multi-tenant flag) fits E3. | false | false |
| A6-P8 | agree | Boundary validation; synthesis merge with A1-P11 accepted — correct place to land. | false | false |
| A6-P9 | agree | `logical_system_id` on `MessageHeader` is the prototypical additive field that breaks silently without A2-P1. Vote presumes A2-P1. | false | false |
| A6-P10 | agree | Capability-scoped sinks — extension interface; §1.2 OCP satisfied. | false | false |
| A6-P11 | agree | Gated `register_factory` complements A2-P8 (closure as known deviation) with a controlled open door. E1 boundary preserved. | false | false |
| A6-P12 | agree | Signed REPLAY trace is moot if A9-P6 agrees. A2 prefers REPLAY stay in v1 scope (see vote on A9-P6). If REPLAY stays, A6-P12 requires A2-P9 (versioned trace schema) to land first. | false | false |
| A6-P13 | abstain | Per-tenant rate-limit depends on multi-tenant scope decision in v1; neither E-rule is violated either way. Re-vote once scope decided. | false | false |
| A6-P14 | agree | Key-material lifecycle ADR is exactly the E5 migration-plan pattern for a specific asset class. | false | false |

### 6.6 Votes on A7 (Modularity)

| proposal_id | vote | rationale (cite rule id) | blocking | override_request |
|-------------|------|--------------------------|----------|-------------------|
| A7-P1 | agree | D6 forbids dependency cycles (hard rule). Blocker-level. | true | false |
| A7-P2 | agree | Role-split on `ISchedulerLayer` is ISP (§1.4) and improves OCP by making each extension seam narrower. Combine with A9-P1 per synthesis merge register — one submit, split roles. | false | false |
| A7-P3 | agree | Invert core↔hal dependency — D2 (abstractions don't depend on details). | false | false |
| A7-P4 | agree | Move payload structs to `distributed/` — lets distributed protocol evolve independently (E4, §1.1 SRP). | false | false |
| A7-P5 | agree | `distributed_scheduler` depends only on `ISchedulerLayer` — D2/D6. | false | false |
| A7-P6 | agree | Extract MLR + deployment parser — SRP; also creates the natural home for A2-P2 `LevelOverrides` schema. | false | false |
| A7-P7 | agree | Forward-decl contract; D7. | false | false |
| A7-P8 | agree | Consolidate `ScopeHandle` ownership — DRY. | false | false |
| A7-P9 | agree | Dedup Python `MemoryError` class names. | false | false |

### 6.7 Votes on A8 (Testability & Observability)

| proposal_id | vote | rationale (cite rule id) | blocking | override_request |
|-------------|------|--------------------------|----------|-------------------|
| A8-P1 | agree | `IClock` interface — §1.5 DIP; adds exactly one abstract seam at the right boundary. E1 satisfied. | false | false |
| A8-P2 | agree | `step()` + `IEventLoopDriver` is exactly the minimal test seam that A9-P2 synthesis amendment endorses — one seam, not a pluggable policy registry. Compatible with A2-P7 amendment. | false | false |
| A8-P3 | agree | Stats structs + histograms imply a schema — adopt under A2-P9 (versioned stats schema extension). | false | false |
| A8-P4 | agree | `dump_state()` endpoint is additive (E4); merge with A5-P6. | false | false |
| A8-P5 | agree | Externalizing alert rules to a file is E3 (config over hardcoding); OTEL sink is E4 (new sink interface). Accept synthesis amendment: sink ADR + known-deviation ledger. | false | false |
| A8-P6 | agree | Trace time-alignment contract (E6). | false | false |
| A8-P7 | agree | `IFaultInjector` seam — coincides with A5-P5; agree once under either owner. | false | false |
| A8-P8 | agree | In-core trace upload protocol — new protocol, must carry version (E6) and a handshake gate (E2). | false | false |
| A8-P9 | agree | Degraded profiling as first-class alert — additive, no contract break. | false | false |
| A8-P10 | agree | Structured KV logging — E6 stability of log field names. | false | false |
| A8-P11 | agree | HAL contract test suite is the enforcement mechanism for A2-P5's BC policy. | false | false |
| A8-P12 | agree | Stable `PhaseId`s are E6 schema stability for phase-identifying enums. | false | false |

### 6.8 Votes on A9 (Simplicity)

| proposal_id | vote | rationale (cite rule id) | blocking | override_request |
|-------------|------|--------------------------|----------|-------------------|
| A9-P1 | agree | Merged with A7-P2 per synthesis register: one `submit(SubmissionDescriptor)`, role-split interfaces. A2 view: this sharpens OCP rather than weakens it (fewer overload collisions, one extension axis per role). | false | false |
| A9-P2 | disagree | Removing `IEventCollectionPolicy`/`IExecutionPolicy` *as interfaces* is an OCP regression at four already-documented change axes (FULLY_SPLIT, SPLIT_DEFERRED, Batched, Dedicated). Synthesis bridge accepted for A8-P2 (single `IEventLoopDriver` test seam) — but A9-P2 goes further and closes the policy enums entirely. Amend to: keep policy enums closed for v1 *with ADR-recorded future-extension names* (E5 migration plan); preserve the abstract policy hook behind a `final` concrete v1 implementation so v2 is additive. Without this amendment, A9-P2 violates E1/E5 discipline. | false | true |
| A9-P3 | agree | Removing collectives from `IHorizontalChannel` aligns ISP (§1.4) with A2's E1 principle: collectives land as Orchestration Functions / `ICollectiveOps` later (synthesis amendment: roadmap ADR). Interface narrowed, extension path preserved. | false | false |
| A9-P4 | agree | `SubmissionDescriptor::Kind` discriminant is redundant with `spmd.has_value()`; removal is E4-compatible because re-adding later is additive. | false | false |
| A9-P5 | agree | Unify admission enums (D7/DRY); removes a known drift risk for E6 when new admission outcomes are added. | false | false |
| A9-P6 | disagree | Deferring PERFORMANCE/REPLAY wholesale closes two documented extension points without the migration plan E5 requires. REPLAY also underpins A6-P12 (signed trace) and A8 debugging harness (A8-P2/P7). Amend to: ship FUNCTIONAL-only *implementations* but keep the `SimulationMode` enum open + ADR declaring v2 scope with named Q records. That satisfies A9's YAGNI (no code yet) and A2's E5 (named transition). | false | true |
| A9-P7 | agree | Folding `SourceCollectionConfig` into `EventHandlingConfig` is DRY; if the two fields need re-separation later, that is an additive struct extension under E6. | false | false |
| A9-P8 | agree | Moving companion-artifact obligations out of the core design — scope discipline; no E-rule impact. | false | false |

### 6.9 Votes on A10 (Scalability & Data Flow)

| proposal_id | vote | rationale (cite rule id) | blocking | override_request |
|-------------|------|--------------------------|----------|-------------------|
| A10-P1 | agree | Default `producer_index` sharding; shard count as `LevelParam` meets E3. Merged with A1-P14 per register. | false | false |
| A10-P2 | abstain | Merged into A5-P3 per register; voted via A5-P3 (agree under synthesis amendment: v1 fail-fast + Q-record, v2 decentralize). This row is the alias. | false | false |
| A10-P3 | agree | Per-data-element consistency model — documentary; supports E6 audit. | false | false |
| A10-P4 | agree | Stateful/stateless classification — supports E1 (identify change axes). | false | false |
| A10-P5 | agree | Per-peer REMOTE_SUBMIT projection; merged into A1-P6 per register. Vote via A1-P6. | false | false |
| A10-P6 | agree | Faster configurable peer-failure detection; thresholds as config (E3). | false | false |
| A10-P7 | agree | Sharded TaskManager; synthesis amendment requires `shards=1` default + `LevelParam` opt-in. E1 preserves fast path, E3 externalizes scaling knob. | false | false |
| A10-P8 | agree | Aggregated data-flow / ownership diagram; supports E5 change-impact analysis. | false | false |
| A10-P9 | agree | Gate `WorkStealing` against `RETRY_ELSEWHERE` — explicit incompatibility; E2/E5 disciplined. | false | false |
| A10-P10 | agree | Cap + layout guidance for `producer_index`; merged into A1-P14. | false | false |

**Vote tally:** agree = 93, disagree = 3, abstain = 5, blocking = 1 (A7-P1, hard-rule D6), override_request = 3 (A1-P9, A9-P2, A9-P6).

## 7. Cross-aspect tensions

| pair | with_proposal | preferred_resolution |
|------|---------------|----------------------|
| A2 vs A9 | A9-P2 (cut FULLY_SPLIT/SPLIT_DEFERRED + pluggable policies) | Keep policy enums closed for v1 **with ADR-listed v2 extensions named by Q-number**. Preserve the policy abstract class behind a `final`-marked v1 implementation; no registry. This reconciles A9's YAGNI (no new code runs in v1) with A2's E5 (named transition step on the record). A8's driveable seam stays as a single `IEventLoopDriver` test double (synthesis bridge) — not a policy registry. |
| A2 vs A9 | A9-P6 (defer PERFORMANCE/REPLAY) | Ship FUNCTIONAL-only implementations in v1; keep `SimulationMode` enum open; record v2 REPLAY/PERFORMANCE bring-up as a Q-entry per mode. A6-P12 conditioned on REPLAY being brought up in v2. Avoids both premature build and unrecorded deferment. |
| A2 vs A1 | A1-P9 (bitmask ≤64 WorkerGroup availability) | Accept bitmask as fast path only if synthesis amendment is fulfilled in R2: multi-bitmap fallback (2×64, growable) specified in the edit sketch with a switchover rule keyed on `num_groups > 64`. Without the fallback, we are encoding a scale ceiling the roadmap already expects to cross — E1 regression. |
| A2 vs A5 | A5-P7 (Timeout on IMemoryOps) | Add as a new overload `submit_with_timeout(...)` instead of changing the existing signature; keep the zero-timeout path on the existing method. E2 deprecation applies only to the new code path; existing callers unchanged. |
| A2 vs A6 | A6-P9 / A6-P7 / A5-P4 (additive fields to MessageHeader / FunctionDescriptor / TaskDescriptor) | Bundle behind A2-P1: these three proposals must cite the version-bumped contract to which they apply. A2-P1 provides one uniform `schema_version` + reserved-field discipline; each R2 additive proposal writes one sentence naming the new version number it introduces. Otherwise old receivers break three different ways. |
| A2 vs A7 | A7-P2 (split ISchedulerLayer) + A9-P1 (merge) | Agree with merged form. The role-split is ISP (§1.4) and **narrows** each extension seam — improves OCP compared to the current fat interface. Explicitly not in tension with A2. |
| A2 vs A8 | A8-P2 (driveable event loop) | Scope the test seam to one `IEventLoopDriver` abstraction (test-only); do not expose it as a runtime plugin point. Removes A9-P2 overlap while preserving A2-P7-style named future policy extension in the Q register. |
| A2 vs A3 | A3-P5 (sibling cancellation policy) + A2-P7 | A3-P5's policy is the first concrete named extension that A2-P7's Q-record points to. Resolve by landing A3-P5's policy with a `CancellationPolicy` enum (closed, three values) + Q-record for future additions. E5 path is explicit. |

## 7a. Merge Register response

Decisions on the six merges proposed in the synthesis (A2 acts as reviewer; no merge absorbs an A2 proposal):

| Merge | Accept / Reject | Reason (from A2 viewpoint) |
|-------|-----------------|-----------------------------|
| A1-P6 ⟵ A10-P5 | accept | Same scope (REMOTE_SUBMIT payload hygiene). New message variants still require E6 handshake gate — A2 vote applies to the merged proposal. |
| A1-P14 ⟵ A10-P10 | accept | Same scope (`producer_index` layout / sizing). Doc-only; E3 externalization via `LevelParam` preserved. |
| A5-P3 ⟵ A10-P2 | accept | Same architectural issue (Pod coordinator failover). Synthesis amendment (v1 fail-fast + Q-record) is E5-compliant; A10's full-decentralization becomes the v2 target named in the Q-record. |
| A5-P6 ⟷ A8-P4 | accept (pair-merge) | `dump_state()` is the evidence surface the watchdog consumes; coherent change. |
| A7-P2 ⟵ A9-P1 | accept | Role-split + single `submit()` is one coherent change. From A2's standpoint it **improves** OCP (narrower seams, §1.4 ISP) rather than weakening it. |
| A6-P8 ⟵ A1-P11 (partial) | accept | Per-arg boundary validation fits inside A1-P11's per-arg budget; single boundary contract under one vote. |

All six merges accepted. None absorb an A2 proposal.

## 8. Stress-attack on emerging consensus

Skipped — not applicable until Round 3.

## 9. Status

- **Satisfied with current design?** partially
- **Open items expected in next round:**
  - **Own:** A2-P1 (defended), A2-P2 (amended), A2-P6 (amended), A2-P7 (amended) — require R3 confirmation that the amendments answer A9's YAGNI and A1's hot-path concerns. A2-P4, A2-P5, A2-P8, A2-P9 expected fast-track agreement.
  - **Peer disagreements to resolve:** A1-P9 (needs fallback spec), A9-P2 (needs ADR-listed v2 extensions), A9-P6 (needs enum-open + Q-record amendment).
  - **Conditional votes:** A6-P12 depends on A9-P6 outcome; A6-P13 abstained pending multi-tenant scope decision; A5-P4 / A6-P7 / A6-P9 / A5-P8 all depend on A2-P1 landing.
