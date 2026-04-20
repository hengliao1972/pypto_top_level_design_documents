# Round 2 — Consolidated Extract for Synthesis

Source: the ten round-2 reviewer files under `round-2/reviews/`.
Purpose: sole input for the round-2 synthesis phase.

Conventions:
- `own` = the proposal's owner (self-vote not recorded).
- `missing` = the reviewer did not include an explicit row for that proposal; treat as `abstain (implicit)`.
- All vote tallies here are re-counted from the raw review tables; where a review's own summary-tally is internally inconsistent, the tables below use the raw row-level votes, not the self-tally.

---

## (A) Revisions by Proposal

| proposal_id | owner | round_2_action | amended_summary (if amend/split) | reason |
|-------------|-------|----------------|----------------------------------|--------|
| A1-P1 | A1 | defend | — | No blocking peer objection; A10-P10 (now merged into A1-P14) reinforces via cap+layout; pre-sizing closes the admission-path rehash allocation (rule X2). |
| A1-P2 | A1 | amend | Default `producer_index_shards = 1`; when `scheduler_thread_count > 1`, `shards = max(1, scheduler_thread_count)` with caps 8/4/1 at Host/Device/Chip+Core; normative per-shard lock bounds reader p99 ≤ 3 µs / writer p99 ≤ 10 µs. | Addresses A9 YAGNI (single-shard default) and A10-P1 convergence (`shard_count` as `LevelParam`) while keeping A5's uncontended RW-lock fast path. |
| A1-P3 | A1 | defend | — | Synthesis routed to fast-track; no peer dispute. |
| A1-P4 | A1 | amend | Move normative hot/cold split into `modules/core.md §8` only; Logical View stays abstract; hot prefix (first 64 B) enumerated (`state, fan_in_counter, submission_id, exec_type_id, worker_id, dispatch_ts_ns`); AoS justified. | A9 prescription-before-benchmark pushback + A7 module-cohesion; synthesis suggestion to localize the relayout. |
| A1-P5 | A1 | defend | — | No blocking objection; A3 and A8-P12 reinforce. |
| A1-P6 | A1 | amend + merge-accepted | Merge with A10-P5 → "Distributed payload hygiene": cap REMOTE_SUBMIT, REMOTE_BINARY_PUSH staging channel, `descriptor_template_id` dedup, per-peer projection (A10-P5 contribution), fan-out=1 fast path. | Synthesis merge register; A10 is co-owner on per-peer projection. |
| A1-P7 | A1 | defend | — | No blocking objection; A8-P1/A8-P12 add no cross-socket atomic. |
| A1-P8 | A1 | defend | — | Synthesizer fast-tracked; no peer objected. |
| A1-P9 | A1 | amend | Spec explicit 2×64-bitmap fallback for G>64 groups per slot-type; above 128 groups: N/64 words (O(slot_types × ⌈G/64⌉)); fallback documented in `02-logical-view/03-worker.md §2.1.4.2`. | A2-P3 extensibility (ceiling not hard cap) and A9 premature-layout concern. |
| A1-P10 | A1 | defend | — | Fast-track; ties A8-P9 / A5-P5 cleanly. |
| A1-P11 | A1 | amend + merge-accepted | Partial merge with A6-P8 (A6-P8a): combined per-arg budget covers both latency (scalar ≤20 ns, DLPack contiguous ≤200 ns, BufferRef ≤30 ns) and S3 boundary validation (dtype/device/shape/stride/capsule). | Synthesis "natural fusion"; addresses A6-P8 validation while holding the 200 ns envelope. |
| A1-P12 | A1 | amend | Two-tier: fast-path default = `Dedicated` (no knob, admission §4.8.1 2 µs unchanged); slow-path = `Batched` only when explicitly selected (default `max_batch=8` at Chip, tail ≤ `max_batch × 2 µs`); Deployment-Modes table gets a "default Dedicated" column. | A9 tunable-proliferation pushback; A3 scenario concern. |
| A1-P13 | A1 | amend | HAL contract: `args_size ≤ 1 KiB` fast-path uses pre-registered ring slot from CONTROL pool (`hal_args_ring_capacity`, default 1024); 1 KiB < args ≤ 4 KiB slow-path stages via `BufferRef`; > 4 KiB rejected at admission (`ArgsTooLarge`). | A7 HAL-contract tightening; A5 timeout interaction (inherits `worker_timeout_ms`). |
| A1-P14 | A1 | amend + merge-accepted | Merge with A10-P10: `producer_index` in CONTROL region, default `shards = max(1, scheduler_thread_count)`, capacity ≥ `max_outstanding_submissions × expected_out_per_submission × 2`, open-addressed cache-line buckets, `isolate_cache_line = true`. | Synthesis merge register; A10-P10 is the occupancy half of the same doc-only change. |
| A2-P1 | A2 | defend | — | Core E6 finding; reinforced by six additive-field peer proposals. Fixed offset + compile-time constant ⇒ zero hot-path branch. |
| A2-P2 | A2 | amend | Closed-for-v1 (ADR-backed enum list) with schema-registry transition plan for v2; v1 carries reserved `LevelOverrides.extension_map` byte placeholder so v2 adoption is non-breaking. | Synthesis suggestion ("closed for v1, schema-registered for v2 via Q6"); neutralizes A9 YAGNI. |
| A2-P3 | A2 | defend | — | Pre-compromise (closed `DepMode`, open extension-point enums); likely agreement. |
| A2-P4 | A2 | defend | — | Fast-track; doc obligation under E5. |
| A2-P5 | A2 | defend | — | Fast-track; every additive-field vote presumes A2-P5 lands. |
| A2-P6 | A2 | amend | `IDistributedProtocolHandler` declared only inside `distributed_scheduler`; v1 = single concrete backend with CRTP / `final` devirtualization; no plugin registry v1; handler-registry lands when a second backend is named. | Addresses A1 zero-virtual-dispatch and A9 YAGNI-on-registry; preserves OCP seam. |
| A2-P7 | A2 | amend | Reframe as Q-record in `09-open-questions.md` naming the two future policy axes (event-collection mode, execution-policy plug-in); no interface added in v1. | Synthesis suggestion; answers A9 (no speculative interface) and A1 (no dead code). |
| A2-P8 | A2 | defend | — | Fast-track; bookkeeping-only, Rule Exceptions procedure. |
| A2-P9 | A2 | defend | — | Fast-track; unblocks A6-P12 + A8-P12. |
| A3-P1 | A3 | split | A3-P1a (blocker): add `ERROR` state with transitions from `DISPATCHED/EXECUTING/COMPLETING`. A3-P1b (medium): add optional `CANCELLED` state + `on_cancel` handler. | No peer dissent on `ERROR`; splitting preserves blocker even if sibling-cancellation (A3-P5) is deferred. |
| A3-P2 | A3 | amend | Commit option (b): `std::expected<SubmissionHandle, ErrorContext>` (or `Status submit(..., SubmissionHandle*)`); Python binding keeps throwing convenience; adopt A9-P5 unified `AdmissionStatus` payload; record as new ADR. | Satisfies A2 (E1/E4) and A9 (single contract); unblocks A7-P2+A9-P1 merge. |
| A3-P3 | A3 | defend | — | Fast-track; no pushback. |
| A3-P4 | A3 | amend | `DEP_FAILED(producer_key, ErrorCode)` walks a precomputed successor list stored on each Task slot at admission; success fast path = 0 ops; failure slow path O(successors). | Two-tier to pre-empt A1 hot-path concern; aligns with A1-P4/P8. |
| A3-P5 | A3 | amend | Already-`EXECUTING` siblings run to completion and results are discarded; only `PENDING/DEP_READY/DISPATCHED` siblings → `CANCELLED`; aggregation waits on max{exec completion, one event-loop tick}. | Addresses A1 (no forced device-side cancel) and A5 (`idempotent` siblings may be retried elsewhere). |
| A3-P6 | A3 | amend | Traceability matrix required; if A9-P6 agreed, FR-10 shrinks to FUNCTIONAL+ONBOARD and §6.1.5 collapses. | Don't over-commit scenarios that may be removed. |
| A3-P7 | A3 | amend | Merge with A1-P11 and A6-P8 at Python↔C boundary; A3 owns precondition catalog (empty-tasks, self-loop, index-OOB, workspace-subrange-OOB, sum-overflow) + distinct `AdmissionRejected` sub-codes; A1 owns latency; A6 owns threat model. | Three-way convergence at one boundary. |
| A3-P8 | A3 | amend | Release builds skip cycle detection when `SubmissionDescriptor.flags & FE_VALIDATED` is set by trusted frontend; debug/untrusted always run O(V+E) DFS with `max_intra_edges` `LevelParam` cap. | Two-tier answers A1 HPI; FE_VALIDATED is an S3-controlled opt-in. |
| A3-P9 | A3 | defend | — | Clean with A9-P4 (`spmd.has_value()` is orthogonal to reserved slots). |
| A3-P10 | A3 | defend | — | No objections; reinforces A6-P1 and A5 error paths. |
| A3-P11 | A3 | defend | — | Trivial doc edit. |
| A3-P12 | A3 | amend | `drain()` atomic w.r.t. distributed submissions: remote peers observe drain before accepting further `REMOTE_SUBMIT`; in-flight continue to retirement; add `DrainInProgress` to `modules/error.md` Core domain. | A5-P8 + A10-P2 need a drain semantic surviving coordinator failover. |
| A3-P13 | A3 | amend | Pin per-link FIFO + idempotent delivery + duplicate detection via `TaskKey.generation`; align with A5-P10 DS4 contract. | Avoids duplicating A5-P10/A8-P6; one ordering model, three consumers. |
| A3-P14 | A3 | defend | — | Pure doc fix. |
| A3-P15 | A3 | defend | — | Debug-only; no hot-path cost. |
| A4-P1 | A4 | defend | — | Pure prose edit; no peer objected. |
| A4-P2 | A4 | defend | — | Blocker-severity anchor fix; trivial. |
| A4-P3 | A4 | defend | — | Prose count fix. |
| A4-P4 | A4 | defend | — | No peer objected; no cross-ref breakage. |
| A4-P5 | A4 | amend + split | Add glossary for `EventLoopDeploymentConfig` and `IEventCollectionPolicy`; `SourceCollectionConfig` entry conditional — kept iff A9-P7 is rejected; if A9-P2 agreed, `IEventCollectionPolicy` becomes a one-line "deprecated in v1, folded into `EventHandlingConfig` enum" note. | Preserves D7 under whichever shape wins A2↔A9 debate. |
| A4-P6 | A4 | defend | — | No pre-empt; A9 bulk concern pre-empted by inline citation only. |
| A4-P7 | A4 | defend | — | A7 independently surfaces the same concern. |
| A4-P8 | A4 | amend | Update Appendix-B annotation to enumerate 10 lifecycle states + `FAILED` + `ERROR` (+ `CANCELLED` if A3-P1b adopted); cross-ref `modules/core.md §2.3` and `04-process-view.md §4.3`. | A3-P1 introduces new states; Appendix-B must not drift immediately after. |
| A4-P9 | A4 | amend | Expand `Task State` glossary to `FREE → SUBMITTED → PENDING → DEP_READY → RESOURCE_READY → DISPATCHED → EXECUTING → COMPLETING → COMPLETED → RETIRED` + `FAILED` + (conditional) `ERROR`/`CANCELLED`. | Mirror `modules/core.md` after A3-P1. |
| A5-P1 | A5 | defend | — | No peer objection; failure-path only. |
| A5-P2 | A5 | amend | Steady-state `CLOSED` check = single relaxed-atomic load on cache-resident counter behind `thread_local` last-checked TSC (constant in `CLOSED`); auth-fail (A6-P2) counts as hard-fail with ×K multiplier. | A1 hot-path discipline; integrates A6-P2 + A10-P6 fast-fail. |
| A5-P3 | A5 | amend + merge-accepted | Absorb A10-P2 (co-owner). v1 = deterministic fail-fast (`CoordinatorLost` within `heartbeat_timeout_ms` on every surviving peer); v2 = decentralized quorum `cluster_view` generation (A10-P2). Record v2 as Q entry with trigger. | Balances A9 YAGNI, A10 scale goal, R5 no-silent-hang. |
| A5-P4 | A5 | amend | `idempotent: bool = true` on `TaskDescriptor`; scheduler MAY single-node-retry only if `idempotent == true`; on `idempotent == false`, worker-fail → `ERROR` (A3-P1a) + `DEP_FAILED` (A3-P4) with no retry. Cross-ref A9-P5 `AdmissionStatus`. | Resolves A3-P1/P4/P5 dependency. |
| A5-P5 | A5 | amend + co-own | Fuse `IFaultInjector` with A8-P7 as shared sim-only seam: A8-P7 owns the seam, A5-P5 owns scenario matrix + DR drill cadence. If A9-P6 passes, matrix commits FUNCTIONAL-only. | Clean mechanism/contract split with A8; no two seams. |
| A5-P6 | A5 | amend + merge-accepted | Merge with A8-P4 (co-owner). Scheduler deadman written every K=16 cycles; `SchedulerStalled` alert surfaced via `dump_state().scheduler.deadman_age_ns`; alert rule externalized per A8-P5. | A8-P4 gives watchdog its evidence surface. |
| A5-P7 | A5 | defend | — | No peer objection; A1 tension resolved R1. |
| A5-P8 | A5 | amend | Enums `AdmissionPressurePolicy { REJECT, COALESCE, DEFER }` + `GroupPartialAvailabilityPolicy { FAIL_SUBMISSION, SHRINK_AND_RETRY, DEGRADED_CONTINUE }`; each value tied to an A8-P5 `AlertRule`. Defaults remain today's behavior. | Integrates A8-P5 + preserves BC. |
| A5-P9 | A5 | defend | — | A9 YAGNI anticipated; counter: minimum R3 compliance at Worker granularity; already reduced to severity `low`. |
| A5-P10 | A5 | defend | — | Pure doc lint; no conflict. |
| A6-P1 | A6 | defend | — | No peer disagreement; boundary enumeration non-negotiable (S1). |
| A6-P2 | A6 | defend | — | No blocking objection; primitive = mTLS cert pinned; SPIFFE tagged as v2 extension. |
| A6-P3 | A6 | amend | Collapse per-field bound checks into single length-prefix guard at `MessageHeader` receive entry (per-type ceiling table); reject before `message_factory` allocates. HPI = `extends(≤1 compare)` accept / `none` reject. | Synthesis entry-gate resolution; A1 no-alloc-on-reject. |
| A6-P4 | A6 | amend | Two-tier: TCP/control-plane `require_encrypted_transport: bool = true` default, fail-closed multi-host plaintext; RDMA data-plane TLS explicitly NOT applied — security = A6-P5 rkey scoping + A6-P2 node auth. Doc in `10-known-deviations.md` Deviation 3. | Synthesis; addresses A1 RDMA-latency concern. |
| A6-P5 | A6 | amend | Rotation cadence = per-`SUBMISSION_RETIRED`; `IMemoryOps::deregister_peer_read` called from `distributed_scheduler` on retirement; steady-state rate ≪ control-plane budget. | Synthesis; no new FSM; retirement is already slow path. |
| A6-P6 | A6 | defend | — | v1 ships only `IAuditSink` + in-memory pre-allocated ring; file/syslog sinks as OCP extensions. |
| A6-P7 | A6 | amend | Gate behind `DeploymentConfig.trust_boundary.multi_tenant: bool = false`; single-tenant/SIM/dev: `allow_unsigned_functions = true` by default; multi-tenant: false. `FunctionDesc.Attestation` optional; v1 ships field/verifier/single signer; rotation/SPIFFE deferred. | Synthesis; neutralizes A9 YAGNI; A1 registration-time only. |
| A6-P8 | A6 | split | A6-P8a: per-arg structural validation fused into A1-P11 (merge accepted). A6-P8b: `DeploymentConfig.max_import_bytes` cap — standalone config-only, remains A6-owned. | Synthesis merge; A6-P8b preserves S3 byte-cap without per-arg cost. |
| A6-P9 | A6 | defend | — | ≤ 5 ns compare on framing slow path after A6-P3 guard; HPI = none. |
| A6-P10 | A6 | defend | — | API change source-level only; pairs with A8-P10 and A8-P5. |
| A6-P11 | A6 | defend | — | Reinforces A2-P6 init-only stance. |
| A6-P12 | A6 | amend | Conditional on A9-P6: if A9-P6 passes → A6-P12 moves to v2 item in deviations; if A9-P6 fails → apply as written, compose with A2-P9 (trace schema version) as signature binding target. | Synthesis; minimal-overhead shape. |
| A6-P13 | A6 | amend | Fold per-tenant quota into existing admission-accounting counter — tag the per-Submission counter with `logical_system_id` and partition; quota breach surfaces via `AdmissionDecision::REJECTED` (A9-P5). Opt-in via `DeploymentConfig.per_tenant_submit_qps` (default `None` = disabled). | Synthesis; HPI = none; A9 YAGNI mitigated by opt-in. |
| A6-P14 | A6 | defend | — | Doc-only ADR. |
| A7-P1 | A7 | defend | — | No peer disputed the cycle; hard rule D6; A2 (E4), A10 (P4), A5 (R5), A8 (X5) all benefit. |
| A7-P2 | A7 | amend + merge-accepted | Split `ISchedulerLayer` at **header level only** into `ISchedulerWiring`/`ISchedulerSubmit`/`ISchedulerCompletion`/`ISchedulerLifecycle`/`ISchedulerScope`; retain aggregator `ISchedulerLayer` via multiple inheritance. Absorb A9-P1: single `submit(SubmissionDescriptor)` entry; `submit_group/submit_spmd` become free-function builders in `runtime/`. | Merge-register; same vtable count (A1 concern) + no new classes (A9 concern); every consumer compiles against narrowest role header. |
| A7-P3 | A7 | defend | — | No peer objection; A1 implicitly welcomes (core/ leaf). |
| A7-P4 | A7 | amend | Move `Remote*Payload` / `HeartbeatPayload` from `transport/` to `distributed/include/distributed/protocol_payloads.h`; `transport/` keeps only `MessageHeader` + `MessageType` + framing. Anchor A2-P6 registry, A6-P3 bounds table, A1-P6 `REMOTE_BINARY_PUSH` + descriptor-template cache in `distributed/`, not `transport/`. | Three peer-owned follow-ons each need SRP preservation. |
| A7-P5 | A7 | defend | — | No peer objection; ADR-008 already claimed it. |
| A7-P6 | A7 | amend | Prefer strict sub-namespace `runtime::composition` with single include tree; escalate to separate compiled module only if `registry/descriptor/parser` grows distinct dependencies. | A9 "new top-level module" tension; sub-namespace gives D1/D5 wins without CMake ceremony. |
| A7-P7 | A7 | defend | — | No peer objection; DAG-truthfulness fix. |
| A7-P8 | A7 | defend | — | D7 hygiene. |
| A7-P9 | A7 | defend | — | A3-P10 expands mapping but does not resolve the specific duplication. |
| A8-P1 | A8 | defend | — | No peer objection; link-time HAL specialization keeps release zero-cost. |
| A8-P2 | A8 | amend | `EventLoopRunner::step(max_events)` exposed as the **single** `IEventLoopDriver` test-only seam (`enable_test_driver` build flag), paired with `RecordedEventSource` in `core/`. Only event-loop driveability surface in v1. | Synthesis bridge with A9-P2 (single seam, closed policy enums). |
| A8-P3 | A8 | defend | — | Hot-path tier documented (branchless insert ≤ 5 ns, pre-alloc, cold percentile); A1 vetoes exact insert sequence only. |
| A8-P4 | A8 | defend (accept pairing with A5-P6) | — | Merge-register pairs A5-P6 watchdog with A8-P4 evidence surface. |
| A8-P5 | A8 | amend | Alert rules declared in external file referenced by `DeploymentConfig.alerting_rules_path` (4-field schema `{metric, op, threshold, window_ms, severity}`); OpenMetrics/Prometheus sink recorded in `10-known-deviations.md` as opt-in. `Runtime::on_alert(callback)` retained. | Synthesis; addresses A9 "no plugin infrastructure". |
| A8-P6 | A8 | amend | Specify alignment algorithm against A3-P13 FIFO assumption; merge owner = coordinator when centralized (v1) and **any-owner-deterministic** when A10-P2 decentralization lands; require `IClockSync::offset_ns()` + bounded `skew_max_ns`. | Forward-compatibility with A10-P2 + A3-P13 ordering. |
| A8-P7 | A8 | amend (joint co-ownership with A5-P5) | `IFaultInjector` sim-only seam owned by A8; scenario matrix owned by A5-P5. | Synthesis co-owner split. |
| A8-P8 | A8 | defend | — | Level-2 opt-in; default L1 strips. |
| A8-P9 | A8 | defend | — | Peer review supportive; one relaxed-atomic read cold. |
| A8-P10 | A8 | amend | `Logger::log_kv(Severity, category, {kv...}, correlation_id)` as primary surface; every KV event carries `{correlation_id, layer_id, category, severity, logical_system_id}` so A6-P10 capability-scoped sinks filter by branch not parse. | Fuses O2 with S2. |
| A8-P11 | A8 | defend | — | CI plumbing; peer uniformly supportive. |
| A8-P12 | A8 | defend | — | Depends on A1-P7; if A1-P7 rejected, amend to L2-only. |
| A9-P1 | A9 | amend + merge-accepted | Combined with A7-P2: role-split `ISchedulerLayer` into `ISubmissionSink`/`ITaskLifecycleManager`/`ISchedulerQuery`; submission role carries exactly one virtual `submit(SubmissionDescriptor)`; `submit_group/submit_spmd/submit(TaskDescriptor)` become free-function builders in `runtime/`. | Merge-register; orthogonal ISP split + overload collapse; strictly simpler than either alone. |
| A9-P2 | A9 | amend | Retain core simplifications (cut `FULLY_SPLIT`/`SPLIT_DEFERRED`; collapse `IEventCollectionPolicy`/`IExecutionPolicy` to closed enums + config). Keep **single** `IEventLoopDriver` as test-only seam satisfying A8-P2 (one prod impl + one test impl via `RecordedEventSource` + `step()`); document future extension interfaces in an appendix. | A8 X5 seam is legitimate; single test-double preserves driveability without pluggable policy surface; addresses A2 via "extension-points-when-needed" appendix. |
| A9-P3 | A9 | defend | — | Add ADR committing collectives as Orchestration Functions in v1; hardware-native `ICollectiveOps` re-enters when HCCL data demands. |
| A9-P4 | A9 | defend | — | `spmd.has_value()` carries the same info; A3-P7 validated to still work. |
| A9-P5 | A9 | defend | — | Low-controversy fast-track. |
| A9-P6 | A9 | amend | Ship v1 with `FUNCTIONAL` only; new ADR-011-R2 formally defers `PERFORMANCE` and `REPLAY` with trigger conditions (PERFORMANCE: concrete perf-modeling consumer; REPLAY: named post-mortem debugging customer). Move both to `09-open-questions.md`. | Synthesis; addresses A8 via deferral-with-trigger; A6-P12 auto-moot. |
| A9-P7 | A9 | amend | Fold `SourceCollectionConfig` into `EventHandlingConfig`; coordinate with A4-P5 — glossary drops `SourceCollectionConfig` entry. | Synthesis A4 coordination. |
| A9-P8 | A9 | defend | — | Low-controversy fast-track. |
| A10-P1 | A10 | defend | — | Co-owned with A1-P14; keeps `shards=1` at Device/Chip by default (YAGNI-bounded). |
| A10-P2 | A10 | split + concede-to-merge | A10-P2a (v1): deterministic fail-fast + ADR + Q-record — merges into A5-P3 as alias. A10-P2b (v2): sticky + quorum `cluster_view` generation — roadmap-only ADR. | Synth + A9 YAGNI + A1 architecture-change weight make v1 fail-fast the landing point; v2 preserved. |
| A10-P3 | A10 | defend | — | DS6 minimum; tension with A9 resolved by P3+P4+P8 merger in amended A10-P8. |
| A10-P4 | A10 | defend | — | DS1; zero hot-path cost. |
| A10-P5 | A10 | concede-to-merge | Folded into A1-P6 (alias). | Same scope/rule basis (P6). |
| A10-P6 | A10 | amend | Split heartbeat cadence from detection budget; add transport fast-fail signal + hysteresis back to slow-path; fast-fail path runs on dedicated heartbeat thread (does not touch scheduler event loop). | HPI=none preserved; addresses A1/A5 concerns. |
| A10-P7 | A10 | amend | Two-tier sharded TaskManager: default `admission_shards=1` (zero-overhead); opt-in `admission_shards>1` at Host Layer only via `LevelParams` (align A1-P2); global outstanding-Submission window single counter; per-shard ordering + global `submission_id` tiebreak. | HPI=`relayouts` → `none` on default; LevelParam reuses A1-P2. |
| A10-P8 | A10 | amend | Single combined "Data & State Reference" section in `07-cross-cutting-concerns.md` absorbs P3 (consistency table) + P4 (stateful/stateless table) + P8 (ownership diagram); submodules backlink once. | Addresses A9 simplicity; co-locates DS1/DS6/DS7 audit. |
| A10-P9 | A10 | defend | — | Doc + log entry; no peer dissent. |
| A10-P10 | A10 | concede-to-merge | Folded into A1-P14 (alias). | Same artifact (producer_index placement + shard default). |

---

## (B) Vote Matrix

Notation: `own` = owner of the proposal; `missing` = no explicit row present (treat as `abstain (implicit)`); `agree/disagree/abstain`; `blocking` and `override_request` are boolean unless marked.

### A1-P1

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | own | — | — | owner |
| A2 | agree | false | false | "pre-sizing bounded/doc-first; size as `LevelParam`; no OCP regression" |
| A3 | agree | false | false | "pre-size + no hot-path rehash; no LSP impact" |
| A4 | agree | false | false | "pure doc additions; glossary cross-file needed" |
| A5 | agree | false | false | "X2 + no admission-time alloc; improves R4 under burst" |
| A6 | agree | false | false | "X2; eliminates allocator-spike DoS amplification (S1-adj.)" |
| A7 | agree | false | false | "X2; local to `scheduler/`; no module impact" |
| A8 | agree | false | false | "deterministic sizing lets tests predict `ResourceExhausted`" |
| A9 | agree | false | false | "fixed cap simpler than resizable map (G2)" |
| A10 | agree | false | false | "kills rehash on admission hot path; P6+X2; aligns absorbed A10-P10" |

### A1-P2

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | own | — | — | owner |
| A2 | agree | false | false | "shard_count as `LevelParam` meets E3" |
| A3 | agree | false | false | "strengthens admission-concurrency (P4/DS6); composes with A10-P1" |
| A4 | agree | false | false | "adds `producer_index_shards` LevelParam; glossary row" |
| A5 | agree | false | false | "bounded admission tail latency feeds SLO alerts (O5)" |
| A6 | agree | false | false | "bounded lock hold prerequisite for R1/R3 timeouts" |
| A7 | agree | false | false | "narrow lock contract; modularity hygiene win" |
| A8 | agree | false | false | "makes FakeClock-driven watchdog tests deterministic (A8-P1)" |
| A9 | agree | false | false | "bounds worst-case tail (X2/X9); no new abstractions" |
| A10 | agree | false | false | "exact surface A10-P1/P7 need; P4+X3" |

### A1-P3

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | own | — | — | owner |
| A2 | agree | false | false | "additive + swappable LRU; OCP-friendly" |
| A3 | agree | false | false | "closes implicit assumption at `modules/profiling.md`; G3" |
| A4 | agree | false | false | "adds `function_cache_bytes` + extends `HeartbeatPayload`; glossary" |
| A5 | agree | false | false | "P2 cache policy; Bloom-filter avoids hot-path RT" |
| A6 | agree | false | false | "LRU+HEARTBEAT presence prevents unbounded memory-growth (S1 DoS)" |
| A7 | agree | false | false | "presumes Function Cache in `hal/` + Bloom in `distributed/` — clean" |
| A8 | agree | false | false | "per-peer cache observability (drop + miss metric)" |
| A9 | agree | false | false | "LRU+capacity+HEARTBEAT makes ADR-007 concrete; no new surface" |
| A10 | agree | false | false | "DS7 (bounded owned state) + P6 (no unbounded growth)" |

### A1-P4

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | own | — | — | owner |
| A2 | agree | false | false | "one-time layout doc in `modules/core.md §8` per synthesis amendment; E4-neutral" |
| A3 | agree | false | false | "layout-only; precomputed-successor array in cold tier — no tension" |
| A4 | agree | false | false | "enumerates fields → reduces D7 drift between task-model / core / process" |
| A5 | agree | false | false | "doc-only on logical view; no reliability impact" |
| A6 | abstain | false | false | "pure cache-efficiency; no security surface" |
| A7 | agree | false | false | "X4; doc-only in `modules/core.md §8`" |
| A8 | agree | false | false | "hot/cold enumeration makes `dump_state()` safe to snapshot" |
| A9 | disagree | false | false | "HPI=`relayouts`; hot/cold split prescribed in Logical View before benchmark (G2/P3); flip to agree under synth amendment (moved to modules/core.md §8)" |
| A10 | agree | false | false | "X4; doc-only; one-time relayout; cache-efficiency at fan-out" |

### A1-P5

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | own | — | — | owner |
| A2 | agree | false | false | "prose SLOs; new stages later remain additive (E4)" |
| A3 | agree | false | false | "X9+P5; supports G1/V3 traceability; A3-P6 co-owns" |
| A4 | agree | false | false | "new §4.8.6/§4.8.7; critical paths → budget row" |
| A5 | agree | false | false | "X9+P5; prerequisites for SLO-alerts (O5)" |
| A6 | agree | false | false | "X9; needed to validate A6-P3 bounds check cost fits" |
| A7 | agree | false | false | "X9 doc-only; no D-impact" |
| A8 | agree | false | false | "SPMD / event-loop budgets directly measurable via A8-P12 `PhaseId`s" |
| A9 | agree | false | false | "decomposes hot paths into measurable targets; doc only" |
| A10 | agree | false | false | "X9+DS6 (bounds explicit); required to verify P4" |

### A1-P6

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | own | — | — | owner (merge-absorber of A10-P5) |
| A2 | agree | false | false | "cap REMOTE_SUBMIT + staging bounded; new msg type needs E6 handshake" |
| A3 | agree | false | false | "DS3, P6; merged form accepted" |
| A4 | agree | false | false | "merge with A10-P5; glossary entries for `REMOTE_BINARY_PUSH`, `descriptor_template_id`" |
| A5 | agree | false | false | "bounded message avoids chunked-reassembly slow path; agrees with merge" |
| A6 | agree | false | false | "P6+S3; staging channel natural hook for A6-P7 attestation gating" |
| A7 | agree | false | false | "P6; payload owner = `distributed/` per A7-P4" |
| A8 | agree | false | false | "staging + template dedup reduces REMOTE_SUBMIT bytes; dedup counters feed O3" |
| A9 | agree | false | false | "merges cleanly with A10-P5; KISS-friendly" |
| A10 | agree | false | false | "combined payload hygiene + per-peer projection; strong P6 support" |

### A1-P7

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | own | — | — | owner |
| A2 | agree | false | false | "per-thread local seq internal; no contract change" |
| A3 | agree | false | false | "doesn't alter correctness; O1 preserved via correlation_id" |
| A4 | agree | false | false | "no doc-consistency risk; module-internal detail" |
| A5 | agree | false | false | "X3 no cross-socket ping-pong; improves observability for R6" |
| A6 | abstain | false | false | "pure profiling perf; no security implications" |
| A7 | agree | false | false | "X3/P1 local to `profiling/`; D5 preserved" |
| A8 | agree | false | false | "**structurally required** for A8-P12 at L2 on multi-socket hosts" |
| A9 | agree | false | false | "well-understood pattern; removes hot-path contention (G4)" |
| A10 | agree | false | false | "X3 + P4 (scales with submitters)" |

### A1-P8

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | own | — | — | owner |
| A2 | agree | false | false | "pre-sizing bounded; externalize via `LevelParam` (E3)" |
| A3 | agree | false | false | "X2 pre-sizing + CONTROL placement" |
| A4 | agree | false | false | "adds capacity/placement in `modules/scheduler.md §4`" |
| A5 | agree | false | false | "explicit `WouldBlock` is a defined R4 back-pressure response" |
| A6 | agree | false | false | "bounded OutstandingWindow/ReadyQueue closes admission-saturation DoS (S1)" |
| A7 | agree | false | false | "X2; local to `scheduler/`" |
| A8 | agree | false | false | "same as A1-P1; `WouldBlock` surface becomes first-class observable" |
| A9 | agree | false | false | "KISS parallels A1-P1" |
| A10 | agree | false | false | "pre-sized OutstandingWindow/ReadyQueue + placement is X2+P6" |

### A1-P9

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | own | — | — | owner |
| A2 | disagree | false | **true** | "≤64-group cap creates E1 ceiling; needs 2×64 fallback spec in R2 edit sketch" |
| A3 | agree | false | false | "layout choice; 2×64 fallback per synthesis — request it be in edit" |
| A4 | agree | false | false | "no doc-consistency risk; one sub-bullet in `03-worker.md §2.1.4.2`" |
| A5 | agree | false | false | "X4; hot-path neutral; 2×64 fallback adequate for scale" |
| A6 | abstain | false | false | "pure layout optimisation; no security surface" |
| A7 | agree | false | false | "X4; local to `scheduler/`; >64-group fallback requested by A2" |
| A8 | agree | false | false | "X4; `select_workers` under 10 ns accelerates Chip→Core" |
| A9 | disagree | false | false | "premature layout (P3) for ≤64-group cap with no workload evidence; defer" |
| A10 | agree | false | false | "conditional on 2×64 fallback for >64 groups (E4)" |

### A1-P10

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | own | — | — | owner |
| A2 | agree | false | false | "CI gate; no contract impact" |
| A3 | agree | false | false | "benefits V4/O3" |
| A4 | agree | false | false | "adds §9.2.4 in `modules/profiling.md`; no D7 risk" |
| A5 | agree | false | false | "P1 SLO enforcement; supports R6" |
| A6 | agree | false | false | "P1 SLO; tangential to security" |
| A7 | agree | false | false | "P1 CI gate" |
| A8 | agree | false | false | "**A8 co-owner per synthesis**; enforces NFR-1 ≤1% L1, ≤5% L2" |
| A9 | agree | false | false | "low-cost; caught-early; X5/O3" |
| A10 | agree | false | false | "O3+X1; catches hot-path regressions that silently hurt scaling" |

### A1-P11

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | own | — | — | owner (merge-absorber of A6-P8a) |
| A2 | agree | false | false | "natural home for A6-P8 validation per merge register" |
| A3 | agree | false | false | "natural home for A3-P7 + A6-P8 validation; A3 accepts joint ownership" |
| A4 | agree | false | false | "new `modules/bindings.md` subsection; merged with A6-P8" |
| A5 | agree | false | false | "X9 per-arg decomposition; feeds A6-P8 validation budget" |
| A6 | agree | false | false | "absorbs A6-P8a structural DLPack validation into per-arg budget" |
| A7 | agree | false | false | "X9; absorbs A6-P8 at `bindings/` boundary" |
| A8 | agree | false | false | "lets A8-P12 emit `ARG_MARSHAL` phase with per-arg timing" |
| A9 | agree | false | false | "single doc constraint; simpler overall" |
| A10 | agree | false | false | "P6+X9; natural fusion with A6-P8" |

### A1-P12

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | own | — | — | owner |
| A2 | agree | false | false | "conditional: Dedicated default, `max_batch` only when Batched selected" |
| A3 | agree | false | false | "requires fast-path default `Dedicated` per synthesis" |
| A4 | agree | false | false | "adds row to Deployment-Modes table; glossary update" |
| A5 | agree | false | false | "X9+P5; tail-latency budget SLO-keyed; prefer Dedicated default" |
| A6 | abstain | false | false | "tail-latency; no security surface (max_batch mildly S1-positive)" |
| A7 | agree | false | false | "X9/P5; LevelParam scope" |
| A8 | agree | false | false | "X9, P5; exposing `max_batch` makes tail latency first-class SLO" |
| A9 | disagree | false | false | "tunable proliferation (G2) as written; synth amendment (Dedicated default + max_batch only in Batched) would flip to agree" |
| A10 | agree | false | false | "P5+X9; gated to Batched; Dedicated hot path untouched" |

### A1-P13

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | own | — | — | owner |
| A2 | agree | false | false | "HAL contract tightening; requires E2 deprecation window" |
| A3 | agree | false | false | "strengthens X9; precondition for `WorkspaceSizeMismatch` sub-code" |
| A4 | agree | false | false | "updates `modules/hal.md §2.4` + §4.8.1 in same edit" |
| A5 | agree | false | false | "amend on my side: staging-`BufferRef` must surface `ResourceExhausted` deterministically" |
| A6 | abstain | false | false | "HAL contract; OK if `args_size` ceiling enforced at boundary" |
| A7 | agree | false | false | "P6/X9; tightens HAL contract narrowly (D4)" |
| A8 | agree | false | false | "bounds `args_blob` copy; observable via `ARGS_COPY` PhaseId" |
| A9 | agree | false | false | "tightening HAL contract per synth amendment; net-simpler invariant" |
| A10 | agree | false | false | "bounded `args_blob` copy (ring-slot or ≤4 KiB); P6+X2" |

### A1-P14

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | own | — | — | owner (merge-absorber of A10-P10) |
| A2 | agree | false | false | "doc-only; pairs with A10-P10 absorbed" |
| A3 | agree | false | false | "doc `producer_index` placement + shard default; merges with A10-P10" |
| A4 | agree | false | false | "doc-only; directly supports D7 layout specificity" |
| A5 | agree | false | false | "X4 doc-only" |
| A6 | agree | false | false | "X4 — documentation, no security concern" |
| A7 | agree | false | false | "X4 doc-only" |
| A8 | agree | false | false | "X4; closes evidence gap" |
| A9 | agree | false | false | "document-only placement guidance" |
| A10 | agree | false | false | "doc-only; P4+X4" |

### A2-P1

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "E6; version validated only at bindings↔C + handshake; static_assert at C++ site → zero hot-path branch" |
| A2 | own | — | — | owner |
| A3 | agree | false | false | "prerequisite for A3-P2 amended `submit()` (ErrorContext wire version)" |
| A4 | agree | false | false | "creates per-type BC row in new `appendix-c-compatibility.md`" |
| A5 | agree | false | false | "E6; `ProtocolVersionMismatch` instead of UB" |
| A6 | agree | false | false | "E6; needed for A6-P3 + A6-P12 binding; validation stays at boundary" |
| A7 | agree | false | false | "E6; versions touch cold/init-path types" |
| A8 | agree | false | false | "supports REPLAY + distributed trace correlation; handshake-only validation" |
| A9 | disagree | false | false | "YAGNI (G2); no contract has evolved; fold into A2-P5 policy" |
| A10 | agree | false | false | "zero hot-path bytes if out-of-band; essential for cross-version Pod rollouts" |

### A2-P2

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "schema validation init-time only; `LevelOverrides` never on hot path" |
| A2 | own | — | — | owner |
| A3 | agree | false | false | "accept synthesis phasing (closed for v1, registered for v2)" |
| A4 | agree | false | false | "requires glossary update (`LevelOverridesSchema`)" |
| A5 | agree | false | false | "E4; future knobs ship with level factory" |
| A6 | agree | false | false | "E4; compatible with A6-P11 if init-only" |
| A7 | agree | false | false | "E4/OCP; matches Q6 pattern" |
| A8 | agree | false | false | "E4; per-level profiling/tracing knobs extend without editing `runtime/`" |
| A9 | disagree | false | false | "YAGNI for v1; Q6 ADR is right vehicle; synth amendment (closed v1, registered v2) would flip" |
| A10 | agree | false | false | "E3+E6; startup-only cost; supports deployment elasticity" |

### A2-P3

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "DepMode stays closed (hot-path 3-arm table); opened enums resolved at init → no hot-path dispatch" |
| A2 | own | — | — | owner |
| A3 | agree | false | false | "DepMode closed preserves G3 runtime contract" |
| A4 | agree | false | false | "keeping DepMode closed preserves glossary; D7 positive" |
| A5 | agree | false | false | "FailurePolicy resolved to function-pointer table; neutral for A5" |
| A6 | agree | false | false | "conditional: TransportBackend must declare `require_encrypted_transport` at registration (S4 fail-closed)" |
| A7 | agree | false | false | "E4; DepMode closed keeps ISP on extension axes" |
| A8 | agree | false | false | "opening init-time enums without touching hot path aligns A8 test backends" |
| A9 | abstain | false | false | "harmless if strictly documentation; YAGNI if registry mechanism added — vote depends on wording" |
| A10 | agree | false | false | "E4 balanced with G2; right trade" |

### A2-P4

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "migration plan is pure doc; no runtime effect" |
| A2 | own | — | — | owner |
| A3 | agree | false | false | "V1/V5 doc obligation" |
| A4 | agree | false | false | "migration plan in `03-development-view.md`; supports V2 cross-view mapping" |
| A5 | agree | false | false | "mandatory E5; feature flags reduce deployment-time risk" |
| A6 | agree | false | false | "E5; security scope boundary (legacy vs new)" |
| A7 | agree | false | false | "E5 migration plan" |
| A8 | agree | false | false | "orthogonal to A8; improves testability of transition" |
| A9 | agree | false | false | "documentation every real runtime needs (E5); no code surface added" |
| A10 | agree | false | false | "E5; v1→v2 decentralization path needs this scaffolding" |

### A2-P5

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "BC policy doc; no runtime effect" |
| A2 | own | — | — | owner |
| A3 | agree | false | false | "required for A3-P2 ABI-widening choice" |
| A4 | agree | false | false | "direct alignment with A4-P6 (views ↔ decisions back-link)" |
| A5 | agree | false | false | "reliability artefact for external MLR authors" |
| A6 | agree | false | false | "BC policy should explicitly classify security-load-bearing interfaces" |
| A7 | agree | false | false | "E2; **direct modularity win** — per-interface BC class" |
| A8 | agree | false | false | "BC policy table lets A8-P11 enumerate stable vs evolving interfaces" |
| A9 | agree | false | false | "absorbs A2-P1 motivation; G2 satisfied if A2-P1 withdrawn into A2-P5" |
| A10 | agree | false | false | "E2+E6; required for multi-node heterogeneous rollouts" |

### A2-P6

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "registry init-only; handler lookup = function-pointer table keyed by `MessageType` — no v-call" |
| A2 | own | — | — | owner |
| A3 | abstain | false | false | "outside correctness scope; A9 devirt concern valid but not LSP" |
| A4 | agree | false | false | "adds `DistributedMessageHandler` typedef; glossary row required" |
| A5 | agree | false | false | "E4; two-tier short-circuit keeps hot path unchanged" |
| A6 | agree | false | false | "registry init-only + idempotency class declaration (A5-P10) + A6-P3 bounds" |
| A7 | agree | false | false | "E4/D4; registry replaces central switch; must sit in `distributed/` (A7-P4)" |
| A8 | agree | false | false | "pluggable registry enables fault-injection handler (A8-P7) plug" |
| A9 | disagree | false | false | "canonical YAGNI violation (G2) — one v1 impl; call it internal abstraction, not extension point" |
| A10 | agree | false | false | "E4; off-hot-path; conditional on single-impl devirt" |

### A2-P7

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "reserved interface doc-only; sync fast path default" |
| A2 | own | — | — | owner |
| A3 | abstain | false | false | "design-evolution choice; no correctness impact absent concrete consumer" |
| A4 | abstain | false | false | "YAGNI debate A2↔A9; no D7/V5 stake if in Q11 companion" |
| A5 | abstain | false | false | "no A5-specific interest; A9 YAGNI debate owns this" |
| A6 | abstain | false | false | "future interface reservation; doc-level; no security dimension" |
| A7 | disagree | false | false | "declaring dead interface violates D3 (SRP); defer until Q11 resolves" |
| A8 | abstain | false | false | "G2/E1 tension; orthogonal to A8 rubrics" |
| A9 | disagree | false | false | "async-policy seam without named future extender is speculative (G2); convert to Q-record" |
| A10 | abstain | false | false | "no named future extender tied to Q; speculative vs G2 YAGNI; would flip if A2 names extender" |

### A2-P8

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "Rule Exceptions procedure; bookkeeping" |
| A2 | own | — | — | owner |
| A3 | agree | false | false | "recording intentional closures satisfies G3" |
| A4 | agree | false | false | "direct alignment with A4's R1 positive observation" |
| A5 | agree | false | false | "supports A5-P3 amendment (record decentralization as v2 Q)" |
| A6 | agree | false | false | "G5; Rule Exceptions supports S4 reasoning" |
| A7 | agree | false | false | "discipline-only; Rule Exceptions" |
| A8 | agree | false | false | "G3; design-level traceability" |
| A9 | agree | false | false | "KISS-compatible documentation discipline (G2/G5)" |
| A10 | agree | false | false | "known-deviations ledger; zero cost" |

### A2-P9

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "per-file header only; no per-event hot-path cost" |
| A2 | own | — | — | owner |
| A3 | agree | false | false | "required by A8-P12 + A3-P11 Q8" |
| A4 | agree | false | false | "adds header struct in `07-cross-cutting-concerns.md §7.2.2`" |
| A5 | agree | false | false | "E6; REPLAY can refuse incompatible traces" |
| A6 | agree | false | false | "E6; exact hook A6-P12 needs to attach signature" |
| A7 | agree | false | false | "E6" |
| A8 | agree | false | false | "**prerequisite** for A6-P12 + A8-P6" |
| A9 | agree | false | false | "one byte/field cheap; traces *do* evolve" |
| A10 | agree | false | false | "E6; cross-version cluster rollouts" |

### A3-P1

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "`ERROR`/`CANCELLED` already implicit; codifying adds no hot-path branches" |
| A2 | agree | false | false | "additive to FSM; forces A2-P5 deprecation of old-state assumptions" |
| A3 | own | — | — | owner (split into P1a blocker + P1b medium) |
| A4 | agree | false | false | "closes 5 distinct cross-view drifts; A4-P8/P9 anticipate" |
| A5 | agree | false | false | "terminal A5-P4 depends on; without A3-P1 A5-P4 cannot land" |
| A6 | agree | false | false | "makes `AuditEvent.outcome` reliably computable" |
| A7 | agree | false | false | "LSP, D7, V5; state machine uniformity" |
| A8 | agree | false | false | "`TraceEvent.task_state` transitions become complete" |
| A9 | agree | false | false | "correctness fix (G1 overrides G2); FSM stays small" |
| A10 | agree | false | false | "DS3 partial-failure behavior defined; critical at scale" |

### A3-P2

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "prefer option (b) `std::expected` with trivial success path" |
| A2 | agree | false | false | "prefer option (b) `Result<SubmissionId, AdmissionError>`" |
| A3 | own | — | — | owner |
| A4 | agree | false | false | "removes ambiguous narrative at `09-interfaces.md:65`" |
| A5 | agree | false | false | "deterministic return lets callers handle WAIT/REJECT from A5-P8" |
| A6 | agree | false | false | "option (b) future-proof `AdmissionStatus`" |
| A7 | agree | false | false | "option (b) for OCP on future `AdmissionDecision` values" |
| A8 | agree | false | false | "directly measurable via `SUBMISSION_ADMIT` PhaseId" |
| A9 | agree | false | false | "documentation only; aligns with A9-P5 unified enum" |
| A10 | agree | false | false | "D4 narrow stable interface" |

### A3-P3

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "V4, R6; scenario text, no runtime effect" |
| A2 | agree | false | false | "documentary; no contract regression" |
| A3 | own | — | — | owner |
| A4 | agree | false | false | "adds admission failure scenario flagged missing in rubric #2" |
| A5 | agree | false | false | "V4+R6; closes admission-path gap" |
| A6 | agree | false | false | "natural location for A6-P3 bounds-violation scenarios" |
| A7 | agree | false | false | "V4/R6" |
| A8 | agree | false | false | "closes V4 gap A8-P7 matrix would otherwise leave incomplete" |
| A9 | agree | false | false | "fills scenario gap; no runtime surface" |
| A10 | agree | false | false | "V4+DS3" |

### A3-P4

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "`DEP_FAILED` cold path only; successor walk O(fan_out)" |
| A2 | agree | false | false | "additive (E4); must land under A2-P1 version discipline" |
| A3 | own | — | — | owner |
| A4 | agree | false | false | "adds `DEP_FAILED` to `SchedulerEvent::Type`" |
| A5 | agree | false | false | "critical for A5; gives `DEP_FAILED` for non-idempotent Task failure" |
| A6 | agree | false | false | "R1, DS3; removes unbounded wait (S1 DoS surface)" |
| A7 | agree | false | false | "DS3/R1; single additive event-enum value" |
| A8 | agree | false | false | "replaces 'eventually timeout' hand-wave; correlation_id already threads" |
| A9 | agree | false | false | "correctness requirement (DS3); shrinks later rework" |
| A10 | agree | false | false | "DS3; prevents silent stalls at scale" |

### A3-P5

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "DS4, G3; cancellation cold path; policy hook policy-time" |
| A2 | agree | false | false | "exactly the policy A2-P7 Q-records" |
| A3 | own | — | — | owner |
| A4 | agree | false | false | "adds `IResourceAllocationPolicy::on_child_error()` — glossary row" |
| A5 | agree | false | false | "critical; gives `idempotent=false` retry-suppression deterministic fan-in" |
| A6 | agree | false | false | "DS4; bounds error propagation time; prereq for auditable trails" |
| A7 | agree | false | false | "R4/DS4 via `on_child_error()` — clean policy seam" |
| A8 | agree | false | false | "deterministic `ErrorContext.remote_chain` fan-in" |
| A9 | agree | false | false | "prevents undefined behavior; correctness trumps simplicity" |
| A10 | agree | false | false | "DS3+DS6 explicit semantics" |

### A3-P6

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "G1, V3; scenario doc" |
| A2 | agree | false | false | "supports E5 (migrations preserve traced reqs)" |
| A3 | own | — | — | owner |
| A4 | agree | false | false | "directly satisfies A4 rubric-adjacent trace" |
| A5 | agree | false | false | "V3+G1; reliability-audit artefact" |
| A6 | agree | false | false | "enumerates requirement coverage incl. FR-9 multi-tenant" |
| A7 | agree | false | false | "V3/G1" |
| A8 | agree | false | false | "missing table A8-P11 cross-references" |
| A9 | agree | false | false | "single doc artifact; G5 benefits" |
| A10 | agree | false | false | "V3" |

### A3-P7

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "G3, S3; `FE_VALIDATED` bit two-tier; fuses with A6-P8" |
| A2 | agree | false | false | "additive; supports E2 (stable preconditions documented)" |
| A3 | own | — | — | owner |
| A4 | agree | false | false | "adds 6 sub-codes in `modules/error.md §2.1`" |
| A5 | agree | false | false | "folds into A5-P8 REJECT path without new branch" |
| A6 | agree | false | false | "S3+G3; fully overlaps A6 input-validation; co-owner" |
| A7 | agree | false | false | "admission-only; does not widen `ISchedulerLayer`" |
| A8 | agree | false | false | "A8-P7 fault-injection can count distinct codes" |
| A9 | agree | false | false | "compatible with A9-P4 (uses `spmd.has_value()`)" |
| A10 | agree | false | false | "S3+G3" |

### A3-P8

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "G3, X9; `intra_edges.empty()` fast-path single branch; `FE_VALIDATED` elides in release" |
| A2 | agree | false | false | "conditional on synthesis amendment: debug-only + `max_intra_edges` cap" |
| A3 | own | — | — | owner |
| A4 | agree | false | false | "no D7 impact; improves §4.8.4 table specificity" |
| A5 | agree | false | false | "bounded DFS on admission keeps R4 deterministic" |
| A6 | agree | false | false | "unbounded check itself is DoS surface; O(V+E) with cap" |
| A7 | agree | false | false | "G3/X9; local to admission" |
| A8 | agree | false | false | "measurable via `DATA_DEP_SCAN` PhaseId" |
| A9 | disagree | false | false | "HPI=`extends(bounded)` on admission; synth amendment (debug-only + fast-path skip) would flip; as written: disagree" |
| A10 | agree | false | false | "conditional on debug-only default (two-tier)" |

### A3-P9

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "G3, LSP; reserved scalar slots; compile-time zero runtime cost" |
| A2 | agree | false | false | "SPMD ABI stability commitment; strengthens E6 for SPMD" |
| A3 | own | — | — | owner |
| A4 | agree | false | false | "adds to `06-function-types.md`; aligns rubric #2" |
| A5 | agree | false | false | "deterministic SPMD ABI = idempotent retry at Task level" |
| A6 | agree | false | false | "G3; adds determinism" |
| A7 | agree | false | false | "G3/LSP; function ABI contract" |
| A8 | agree | false | false | "A8-P11 HAL contract tests can assert ABI conformance" |
| A9 | agree | false | false | "clarifies ambiguity; doc-only" |
| A10 | agree | false | false | "V3+D7 consistent naming across partitions" |

### A3-P10

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "G1, D7; exception mapping doc" |
| A2 | agree | false | false | "supports BC when new exceptions added" |
| A3 | own | — | — | owner |
| A4 | agree | false | false | "closes `Domain` ↔ exception map" |
| A5 | agree | false | false | "D7+G1; reliability guarantee for Python caller" |
| A6 | agree | false | false | "prereq for honest audit-log outcomes" |
| A7 | agree | false | false | "D7; complementary to A7-P9" |
| A8 | agree | false | false | "directly testable by `python_mapping_completeness`" |
| A9 | agree | false | false | "prevents silent failures; lightweight" |
| A10 | agree | false | false | "S3 hygiene" |

### A3-P11

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "`[ASSUMPTION]` marker; doc only" |
| A2 | agree | false | false | "documentary marker; zero cost" |
| A3 | own | — | — | owner |
| A4 | agree | false | false | "trivial doc edit" |
| A5 | agree | false | false | "preserves honesty about reliability-critical unresolved mechanism" |
| A6 | agree | false | false | "G3; no tension" |
| A7 | agree | false | false | "G3" |
| A8 | agree | false | false | "matches A8 observability discipline" |
| A9 | agree | false | false | "aligns with G3" |
| A10 | agree | false | false | "G3" |

### A3-P12

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "one atomic flag read on admission" |
| A2 | agree | false | false | "concurrency contract strengthens E2 stability" |
| A3 | own | — | — | owner |
| A4 | agree | false | false | "adds `DrainInProgress` error code" |
| A5 | agree | false | false | "G3+R4; `DrainInProgress` gives shutdown deterministic contract" |
| A6 | agree | false | false | "S2 least-privilege (no admission after teardown)" |
| A7 | agree | false | false | "G3/R4" |
| A8 | agree | false | false | "deterministic state transition observable via A8-P4 `dump_state()`" |
| A9 | agree | false | false | "correctness gap; lightweight" |
| A10 | agree | false | false | "DS6; directly addresses A10's DS6 gap" |

### A3-P13

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "DS3, G3; reorder buffer lives on slow path" |
| A2 | agree | false | false | "protocol-level contract; pairs with A2-P1 + E6" |
| A3 | own | — | — | owner |
| A4 | agree | false | false | "A4-friendly `[ASSUMPTION]` discipline" |
| A5 | agree | false | false | "causal-consistency must be explicit for R4/R6 reproducibility" |
| A6 | agree | false | false | "bounded reorder window caps replay-injection attack" |
| A7 | agree | false | false | "DS3/G3" |
| A8 | agree | false | false | "**prerequisite for A8-P6**" |
| A9 | agree | false | false | "necessary for distributed correctness reasoning (DS6)" |
| A10 | agree | false | false | "DS6+P6; co-owned by A10" |

### A3-P14

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "LSP, V5; diagram edge only" |
| A2 | agree | false | false | "small FSM addition, additive" |
| A3 | own | — | — | owner |
| A4 | agree | false | false | "addresses V5 claim called out in R1" |
| A5 | agree | false | false | "cleans uniform state machine claim" |
| A6 | agree | false | false | "LSP; one extra FSM edge; no security impact" |
| A7 | agree | false | false | "LSP/V5" |
| A8 | agree | false | false | "closes state-machine inconsistency for A8-P12 PhaseId boundaries" |
| A9 | agree | false | false | "minor FSM clarification" |
| A10 | agree | false | false | "DS6 state-transition completeness" |

### A3-P15

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "G3, X6; debug-only, release build strips" |
| A2 | agree | false | false | "debug-mode cross-check; E4-neutral" |
| A3 | own | — | — | owner |
| A4 | agree | false | false | "no hot-path impact; doc addition" |
| A5 | agree | false | false | "cheap reliability defence; zero prod cost" |
| A6 | agree | false | false | "G3/X6; no security concern" |
| A7 | agree | false | false | "X6 debug-only; zero release cost" |
| A8 | agree | false | false | "textbook X6 production-debuggability hook" |
| A9 | agree | false | false | "debug-only; no hot-path impact" |
| A10 | agree | false | false | "X6 production-debuggable" |

### A4-P1

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "D7, V5; enum casing fix" |
| A2 | agree | false | false | "E6 naming stability precondition" |
| A3 | agree | false | false | "D7/V5" |
| A4 | own | — | — | owner |
| A5 | agree | false | false | "D7; pure doc fix" |
| A6 | agree | false | false | "naming consistency" |
| A7 | agree | false | false | "D7/V5 naming" |
| A8 | agree | false | false | "avoids mismatched trace-string comparisons" |
| A9 | agree | false | false | "D7 (consistent naming)" |
| A10 | agree | false | false | "D7" |

### A4-P2

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "D7, V5; anchor fix" |
| A2 | agree | false | false | "blocker-severity anchor fix; trivial" |
| A3 | agree | false | false | "D7/V5" |
| A4 | own | — | — | owner |
| A5 | agree | false | false | "documentation-reliability defect" |
| A6 | agree | false | false | "D7, V5; pure doc fix" |
| A7 | agree | false | false | "D7 anchor fix" |
| A8 | agree | false | false | "pure correctness fix" |
| A9 | agree | false | false | "pure maintenance" |
| A10 | agree | false | false | "V5" |

### A4-P3

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "D7, G4; prose count fix" |
| A2 | agree | false | false | "prose count fix" |
| A3 | agree | false | false | "V5/LSP-adjacent; confirms non-aliasing invariant" |
| A4 | own | — | — | owner |
| A5 | agree | false | false | "D7+G4; trivial" |
| A6 | agree | false | false | "D7, G4; prose arithmetic fix" |
| A7 | agree | false | false | "D7/G4 count fix" |
| A8 | agree | false | false | "pure accuracy fix" |
| A9 | agree | false | false | "invariant-count prose fix" |
| A10 | agree | false | false | "V5" |

### A4-P4

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "D7, V5; section reorder" |
| A2 | agree | false | false | "reorder open questions; no contract change" |
| A3 | agree | false | false | "V5" |
| A4 | own | — | — | owner |
| A5 | agree | false | false | "D7" |
| A6 | agree | false | false | "D7, V5; numeric ordering" |
| A7 | agree | false | false | "D7 order" |
| A8 | agree | false | false | "pure housekeeping" |
| A9 | agree | false | false | "cosmetic but cheap" |
| A10 | agree | false | false | "doc hygiene" |

### A4-P5

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "D7; glossary additions" |
| A2 | agree | false | false | "if A9-P7 agrees `SourceCollectionConfig` entry drops" |
| A3 | agree | false | false | "if A9-P7 agrees one glossary entry drops" |
| A4 | own | — | — | owner |
| A5 | agree | false | false | "D7" |
| A6 | agree | false | false | "D7; new A6 types must also be added" |
| A7 | agree | false | false | "D7 glossary" |
| A8 | agree | false | false | "needed by A8-P2 test-only seam" |
| A9 | agree | false | false | "coordinated in A9-P7 revision" |
| A10 | agree | false | false | "D7; drop `SourceCollectionConfig` entry if A9-P7 passes" |

### A4-P6

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "V2, G5; ADR back-links doc only" |
| A2 | agree | false | false | "supports E5/E1 traceability" |
| A3 | agree | false | false | "V2/V5" |
| A4 | own | — | — | owner |
| A5 | agree | false | false | "V2+G5; ADR back-refs aid reliability audits" |
| A6 | agree | false | false | "V2, G5; when A6 proposals land, each ADR cites §7.1.x" |
| A7 | agree | false | false | "V2/G5 back-links" |
| A8 | agree | false | false | "makes A8-P12 PhaseId design discoverable" |
| A9 | agree | false | false | "improves G5 justify decisions" |
| A10 | agree | false | false | "V2" |

### A4-P7

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "V5, D7; label unification" |
| A2 | agree | false | false | "D7/DRY" |
| A3 | agree | false | false | "D7" |
| A4 | own | — | — | owner |
| A5 | agree | false | false | "V5" |
| A6 | agree | false | false | "V5, D7; L0 label unification" |
| A7 | agree | false | false | "matches A7 view that L0 is `ISchedulerLayer` impl" |
| A8 | agree | false | false | "helps A8-P11 contract-test targeting" |
| A9 | agree | false | false | "D7 consistent naming" |
| A10 | agree | false | false | "V5+D7" |

### A4-P8

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "D7; appendix annotation" |
| A2 | agree | false | false | "sync Appendix-B count" |
| A3 | agree | false | false | "diff A3-P1 depends on" |
| A4 | own | — | — | owner |
| A5 | agree | false | false | "D7" |
| A6 | agree | false | false | "D7; state count sync" |
| A7 | agree | false | false | "D7 count" |
| A8 | agree | false | false | "fixes drift against A3-P1 `ERROR` state" |
| A9 | agree | false | false | "Appendix-B sync" |
| A10 | agree | false | false | "V5" |

### A4-P9

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "D7; glossary expansion" |
| A2 | agree | false | false | "expand glossary" |
| A3 | agree | false | false | "aligns with A3-P1" |
| A4 | own | — | — | owner |
| A5 | agree | false | false | "D7" |
| A6 | agree | false | false | "D7; glossary expansion" |
| A7 | agree | false | false | "D7 enumeration" |
| A8 | agree | false | false | "enumerates states used by A8-P12 PhaseIds" |
| A9 | agree | false | false | "glossary expansion for `Task State`" |
| A10 | agree | false | false | "D7" |

### A5-P1

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "R2; failure-path only; no success cost" |
| A2 | agree | false | false | "policy params as config (E3)" |
| A3 | agree | false | false | "composes with A3-P4 on producer retry" |
| A4 | agree | false | false | "adds `RetryPolicy` struct" |
| A5 | own | — | — | owner |
| A6 | agree | false | false | "R2; prevents correlated retry storms (S1 adj.)" |
| A7 | agree | false | false | "R2; failure-path only" |
| A8 | agree | false | false | "retry counter histograms fit A8-P3" |
| A9 | agree | false | false | "textbook R2; no simplicity cost" |
| A10 | agree | false | false | "R2; prevents cascading retry storms" |

### A5-P2

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "R3 steady-state CLOSED = `thread_local` TSC gate; single relaxed atomic" |
| A2 | agree | false | false | "additive policy; thresholds config-externalized (A2-P2)" |
| A3 | agree | false | false | "R3; composes with P1 on producer retry" |
| A4 | agree | false | false | "adds `CircuitBreaker` config + `{CLOSED, OPEN, HALF_OPEN}` states" |
| A5 | own | — | — | owner |
| A6 | agree | false | false | "R3; breaker state transitions audit-log sites (A6-P6)" |
| A7 | agree | false | false | "R3; per-peer state in `distributed/`; clean D1" |
| A8 | agree | false | false | "per-peer state metric for A8-P3" |
| A9 | agree | false | false | "scoped to distributed layer, not hot path" |
| A10 | agree | false | false | "R3; essential under large peer count" |

### A5-P3

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "R5; control plane off hot path; v1 fail-fast + Q-record" |
| A2 | agree | false | false | "v1 fail-fast + Q-record is E5 incremental migration; absorbs A10-P2" |
| A3 | agree | false | false | "prefer v1 fail-fast; provides deterministic FSM A3-P12 extends" |
| A4 | agree | false | false | "closes Q5; adds `CoordinatorLost` error code" |
| A5 | own | — | — | owner (merge-absorber of A10-P2) |
| A6 | agree | false | false | "R5; either path a new authenticated-role boundary; A6-P2 must bind current-coordinator claim" |
| A7 | agree | false | false | "R5; prefer option B (deterministic fail-fast) for v1" |
| A8 | agree | false | false | "closes design ambiguity A8-P6 trace merge depends on" |
| A9 | agree | false | false | "fail-fast v1 is KISS option" |
| A10 | agree | false | false | "R5+P4; absorbs A10-P2 (v1 fail-fast + v2 roadmap)" |

### A5-P4

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "`idempotent` flag cold; checked only on retry" |
| A2 | agree | false | false | "requires E6 version bump; presumes A2-P1" |
| A3 | agree | false | false | "prerequisite for A3-P4 decision (`DEP_FAILED` immediately or after retry)" |
| A4 | agree | false | false | "glossary row + Python API doc update" |
| A5 | own | — | — | owner |
| A6 | agree | false | false | "DS4; must be captured in audit-log for retry decisions" |
| A7 | agree | false | false | "DS4; one boolean on `TaskDescriptor`" |
| A8 | agree | false | false | "gates retry deterministically; becomes observable attribute" |
| A9 | agree | false | false | "minimum-cost maximum-correctness (DS4)" |
| A10 | agree | false | false | "DS4" |

### A5-P5

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "R6 test-build only; stripped from production" |
| A2 | agree | false | false | "textbook OCP seam; sim-only keeps hot path clean" |
| A3 | agree | false | false | "V4/R6 support for A3-P3 admission-failure scenarios" |
| A4 | agree | false | false | "new `11-chaos-plan.md`; supports V4" |
| A5 | own | — | — | owner (fuses with A8-P7) |
| A6 | agree | false | false | "R6; sim-only; helps chaos-test auth/rkey/quota paths" |
| A7 | agree | false | false | "R6; `IFaultInjector` sim-only symbol; zero release surface" |
| A8 | agree | false | false | "**jointly owned with A8-P7**; matrix = what, seam = how" |
| A9 | agree | false | false | "sim-only; does not add hot-path surface" |
| A10 | agree | false | false | "R6; required to verify P4 claims at scale" |

### A5-P6

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "deadman written every Kth cycle; lift into existing TSC capture" |
| A2 | agree | false | false | "policy-parameterized monitor (E3); pairs with A8-P4" |
| A3 | agree | false | false | "paired with A8-P4 per merge register" |
| A4 | agree | false | false | "merged with A8-P4; watchdog evidence surface = `dump_state`" |
| A5 | own | — | — | owner (merges with A8-P4) |
| A6 | agree | false | false | "new audit-worthy event on fire; fold alert emission into A6-P6" |
| A7 | agree | false | false | "R5; paired with A8-P4 per merge register" |
| A8 | agree | false | false | "**paired with A8-P4**; detect+diagnose roles" |
| A9 | agree | false | false | "merges with A8-P4's dump_state evidence surface; one paired addition" |
| A10 | agree | false | false | "R5 single-point protection" |

### A5-P7

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "R1; zero cost on success; `poll(zero)` = today's HW flag check" |
| A2 | agree | false | false | "contract widening; amend to new overload `submit_with_timeout()`" |
| A3 | agree | false | false | "closes implicit assumption in rubric #4" |
| A4 | agree | false | false | "interface change must propagate to `04-memory.md` + `modules/memory.md §2`" |
| A5 | own | — | — | owner |
| A6 | agree | false | false | "closes silent-stall window (S1 DoS)" |
| A7 | agree | false | false | "R1; narrow extension to `IMemoryOps`" |
| A8 | agree | false | false | "R1; uses `IClock` under hood" |
| A9 | agree | false | false | "textbook R1 (timeout all remote calls)" |
| A10 | agree | false | false | "R1; no bounded wait = no bounded scale" |

### A5-P8

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "R4; fires only on saturation / partial-group fail — cold paths" |
| A2 | agree | false | false | "additive states (E4); requires A2-P1 for observers" |
| A3 | agree | false | false | "directly covers A3-P3 §4.8.4 gap" |
| A4 | agree | false | false | "adds enums; glossary rows" |
| A5 | own | — | — | owner |
| A6 | agree | false | false | "R4; `admission_pressure_policy` must be per-tenant-aware" |
| A7 | agree | false | false | "R4 policies" |
| A8 | agree | false | false | "become A8-observable state transitions" |
| A9 | agree | false | false | "documentation of behavior that must exist anyway" |
| A10 | agree | false | false | "R4" |

### A5-P9

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "R3, R4; `on_fail` only; `QUARANTINED` invisible to selector" |
| A2 | agree | false | false | "QUARANTINED additive (E4)" |
| A3 | agree | false | false | "LSP-friendly Worker FSM extension; consistent with A3-P1" |
| A4 | agree | false | false | "state update in scheduler.md §3.2 + worker.md §2.1.4.1 + glossary" |
| A5 | own | — | — | owner |
| A6 | agree | false | false | "R3; intra-node equivalent of A6-P2/A5-P2 circuit-breaking" |
| A7 | agree | false | false | "R3; one enum state in `WorkerState`" |
| A8 | agree | false | false | "fits A8-P12 PhaseId + `dump_state()` matrix" |
| A9 | disagree | false | false | "FSM inflation (G2, rubric check 5); express as timestamp field on UNAVAILABLE" |
| A10 | agree | false | false | "R3" |

### A5-P10

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "DS4; doc lint only" |
| A2 | agree | false | false | "stability commitment (E2); clarifies E6 scope" |
| A3 | agree | false | false | "contract A3-P13 assumes; accept as co-owning" |
| A4 | agree | false | false | "pure doc-discipline improvement" |
| A5 | own | — | — | owner |
| A6 | agree | false | false | "DS4; foundational for retry-under-attack reasoning" |
| A7 | agree | false | false | "DS4 contract discipline" |
| A8 | agree | false | false | "CI-checkable assertion; aligns A8-P11" |
| A9 | agree | false | false | "correctness documentation; lightweight" |
| A10 | agree | false | false | "DS4; scale-relevant" |

### A6-P1

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "S1; STRIDE doc, no runtime effect" |
| A2 | agree | false | false | "complete threat model; no contract change" |
| A3 | agree | false | false | "prerequisite for A3-P7 validation pass" |
| A4 | agree | false | false | "supports V1 non-empty rubric" |
| A5 | agree | false | false | "S1; completeness of boundary roster" |
| A6 | own | — | — | owner |
| A7 | agree | false | false | "S1 completeness" |
| A8 | agree | false | false | "complete STRIDE table; boundary crossings enumerated" |
| A9 | agree | false | false | "foundational documentation (S1); keeps later security work scoped" |
| A10 | agree | false | false | "S1" |

### A6-P2

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "S1, S2; handshake-time only; slow path" |
| A2 | agree | false | false | "additive via `HandshakePayload` extension (E6)" |
| A3 | agree | false | false | "S2/S3" |
| A4 | agree | false | false | "new `HandshakePayload` + `credential_id` field; glossary rows" |
| A5 | agree | false | false | "S1/S2/R5; flapping-credential peer must not be generic remote-failure" |
| A6 | own | — | — | owner |
| A7 | agree | false | false | "S1/S2; handshake-time only" |
| A8 | agree | false | false | "auditable handshake outcomes" |
| A9 | agree | false | false | "S2 least privilege on existing boundary" |
| A10 | agree | false | false | "S1+S2" |

### A6-P3

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "S3, X2; single length-prefix guard per message (entry-gate two-tier)" |
| A2 | agree | false | false | "synthesis amendment preserves E1; bounded at trust boundary entry" |
| A3 | agree | false | false | "fold cap check into A1-P11 per-arg budget per synthesis" |
| A4 | agree | false | false | "adds 4 config keys in `modules/transport.md §8`" |
| A5 | agree | false | false | "S3+R4; bounded parsing is DoS-resistance property" |
| A6 | own | — | — | owner |
| A7 | agree | false | false | "S3; lives in `distributed/` after A7-P4" |
| A8 | agree | false | false | "adds `corrupted_frames` metric; A8-P3/P9 alert targets" |
| A9 | agree | false | false | "single length-prefix guard at boundary entry is simple + sound" |
| A10 | agree | false | false | "S3+P6 at boundary entry" |

### A6-P4

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "S4, S5; TLS on TCP only; RDMA uses RC+rkey; explicit in Deviation 3" |
| A2 | agree | false | false | "synthesis amendment explicit (RDMA path unaffected); config downgrade for test-only" |
| A3 | agree | false | false | "two-tier synth amendment; S5 satisfied without hot-path impact" |
| A4 | agree | false | false | "`DeploymentConfig.require_encrypted_transport` + glossary; updates Deviation 3" |
| A5 | agree | false | false | "S4/S5; fail-closed at init is R5 posture for multi-host" |
| A6 | own | — | — | owner |
| A7 | agree | false | false | "S4/S5" |
| A8 | agree | false | false | "startup check, not hot-path cost" |
| A9 | agree | false | false | "TLS on TCP only; two-tier path; KISS-compatible" |
| A10 | agree | false | false | "S5; two-tier; scalability-neutral once RDMA excluded" |

### A6-P5

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "S2, S3; rotation scoped to `SUBMISSION_RETIRED` cold path" |
| A2 | agree | false | false | "rotation on Submission retirement (synth); rotation cadence = versioned policy" |
| A3 | agree | false | false | "rotation on Submission retirement (synth) avoids hot-path cost" |
| A4 | agree | false | false | "new `IMemoryOps::register_peer_read`/`deregister_peer_read` methods" |
| A5 | agree | false | false | "S2+R4 blast-radius bound; aligns layer-lifecycle" |
| A6 | own | — | — | owner |
| A7 | agree | false | false | "S2; narrow extension on `IMemoryOps`" |
| A8 | agree | false | false | "`rkey_inflight` counter feeds A8-P3" |
| A9 | agree | false | false | "lifecycle already implied by Submission completion; minimal extra ceremony" |
| A10 | agree | false | false | "S2; rotation on retirement keeps HPI minor" |

### A6-P6

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "S6; pre-allocated audit ring; rare events" |
| A2 | agree | false | false | "`IAuditSink` is OCP/E1 extension seam done right" |
| A3 | agree | false | false | "composes with A8-P10 structured logging" |
| A4 | agree | false | false | "new §7.1.4 + `AuditEvent` + `IAuditSink`" |
| A5 | agree | false | false | "S6; reliability-postmortem artefact" |
| A6 | own | — | — | owner |
| A7 | agree | false | false | "S6; `IAuditSink` must sit in `profiling/` sibling to `ILogSink`" |
| A8 | agree | false | false | "`IAuditSink` uses exact sink plug-in surface A8 specifies" |
| A9 | agree | false | false | "S6 documentation of structured logging; integrates A8-P10" |
| A10 | agree | false | false | "S6" |

### A6-P7

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "S1, S3; registration is startup/slow path" |
| A2 | agree | false | false | "attestation additive — require A2-P1 FunctionDesc version bump" |
| A3 | agree | false | false | "gated by `multi_tenant=true`; off by default" |
| A4 | abstain | false | false | "YAGNI-vs-security tension; no direct D7/V5 stake" |
| A5 | abstain | false | false | "A5 neutrality; trust property, not reliability property" |
| A6 | own | — | — | owner |
| A7 | agree | false | false | "S1/S3; `Attestation` field on `FunctionDesc` — minimal surface" |
| A8 | agree | false | false | "attestation failures new audit/error class" |
| A9 | disagree | false | false | "YAGNI (G2) without named multi-tenant scenario; synth amendment (gate behind flag, default off) would flip" |
| A10 | abstain | false | false | "no multi-tenant scenario in v1; would agree if gated behind `trust_boundary.multi_tenant=true`" |

### A6-P8

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "S3; fuses with A1-P11; combined 200 ns envelope" |
| A2 | agree | false | false | "merge with A1-P11 accepted — correct place to land" |
| A3 | agree | false | false | "merges with A3-P7 + A1-P11 per synthesis" |
| A4 | agree | false | false | "merged with A1-P11 per register" |
| A5 | agree | false | false | "S3+R4; DLPack validation prevents UB" |
| A6 | own | — | — | owner (split into P8a merged + P8b standalone) |
| A7 | agree | false | false | "S3; absorbed into A1-P11 per register" |
| A8 | agree | false | false | "Python→C stage check measured via `ARG_MARSHAL`" |
| A9 | agree | false | false | "merged into A1-P11; DRY+S3" |
| A10 | agree | false | false | "S3; fuse with A1-P11 per-arg budget" |

### A6-P9

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "S1, S2, S3; conditional: `logical_system_id` outside 64B cache line with `correlation_id`/`source_node`" |
| A2 | agree | false | false | "prototypical additive field; presumes A2-P1" |
| A3 | agree | false | false | "directly supports FR-9 traceability gap" |
| A4 | agree | false | false | "adds `logical_system_id` to `MessageHeader`; critical V5/D7 impact" |
| A5 | agree | false | false | "tenant isolation = reliability property" |
| A6 | own | — | — | owner |
| A7 | agree | false | false | "S1/S2; 4-byte field addition to `MessageHeader`" |
| A8 | agree | false | false | "capability key A6-P10 + A8-P10 KV sinks filter on" |
| A9 | agree | false | false | "S2; isolation invariant exists and needs enforcement hooks" |
| A10 | agree | false | false | "S2; supports P4 by preventing cross-system interference" |

### A6-P10

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "S2; sink dispatch on flush thread, not hot path" |
| A2 | agree | false | false | "extension interface; §1.2 OCP satisfied" |
| A3 | agree | false | false | "S2/O2" |
| A4 | agree | false | false | "`add_sink` signature change; doc updates together" |
| A5 | agree | false | false | "S2; scoped sinks avoid cross-tenant leak" |
| A6 | own | — | — | owner |
| A7 | agree | false | false | "S2; scoped sinks match A7 D4 narrow-surface preference" |
| A8 | agree | false | false | "**aligns with A8-P10 amendment** (KV+capability-scoped)" |
| A9 | agree | false | false | "S2 applied to observability" |
| A10 | agree | false | false | "S2" |

### A6-P11

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "S2, S4; init-time gate" |
| A2 | agree | false | false | "complements A2-P8 with controlled open door; E1 boundary preserved" |
| A3 | agree | false | false | "S2/S4" |
| A4 | agree | false | false | "`RegistrationToken` + `factory_registration_token`; glossary rows" |
| A5 | agree | false | false | "S2+S4; reduces attack surface" |
| A6 | own | — | — | owner |
| A7 | agree | false | false | "S2/S4" |
| A8 | agree | false | false | "pure init-time guard; no hot-path cost" |
| A9 | agree | false | false | "S2/S4 on surface already present" |
| A10 | agree | false | false | "S2+S4" |

### A6-P12

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "S1, S3; conditional on A9-P6 outcome" |
| A2 | agree | false | false | "conditional on A9-P6; if REPLAY stays, needs A2-P9 first" |
| A3 | abstain | false | false | "moot if A9-P6 defers REPLAY; vote tied to A9-P6 resolution" |
| A4 | abstain | false | false | "contingent on A9-P6; cannot vote on conditional" |
| A5 | agree | false | false | "conditional on A9-P6; signing trace preserves R6 reproducibility" |
| A6 | own | — | — | owner |
| A7 | abstain | false | false | "contingent on A9-P6" |
| A8 | agree | false | false | "complements A2-P9 + A8-P6" |
| A9 | disagree | false | false | "moot if A9-P6 passes; YAGNI without named multi-tenant debugger" |
| A10 | abstain | false | false | "depends on A9-P6; if REPLAY kept → agree, else moot" |

### A6-P13

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "S1 two-tier: relaxed counter inc fast path; leaky-bucket only on refill tick" |
| A2 | abstain | false | false | "depends on multi-tenant scope decision in v1" |
| A3 | abstain | false | false | "lacks v1 multi-tenant scenario; orthogonal to A3" |
| A4 | abstain | false | false | "hot-path cost debate A1/A9; A4 unaffected" |
| A5 | agree | false | false | "S1+R4 graceful-degradation; reliability quota" |
| A6 | own | — | — | owner |
| A7 | agree | false | false | "S1 DoS; admission-path accounting, not new branch" |
| A8 | abstain | false | false | "primarily security control; no testability stake" |
| A9 | disagree | false | false | "no multi-tenant scenario in v1 → no rate-limit needed; synth suggests deferral; A9 concurs" |
| A10 | abstain | false | false | "no multi-tenant scenario in v1 (G2); if kept, must fold into existing admission accounting" |

### A6-P14

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "G5, S5; ADR-only" |
| A2 | agree | false | false | "exactly E5 migration-plan pattern for specific asset class" |
| A3 | agree | false | false | "G5/S5" |
| A4 | agree | false | false | "new ADR cross-linked from Deviation 3 and `transport.md §12`" |
| A5 | agree | false | false | "G5+S5; doc-level, no functional impact" |
| A6 | own | — | — | owner |
| A7 | agree | false | false | "G5/S5 ADR" |
| A8 | agree | false | false | "pure documentation; improves audit-trail coverage" |
| A9 | agree | false | false | "documentation; necessary once any key exists" |
| A10 | agree | false | false | "S5+S6" |

### A7-P1

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "D6 (hard rule); strengthens hot-path compile isolation" |
| A2 | agree | **true** | false | "D6 forbids dependency cycles (hard rule); blocker" |
| A3 | agree | false | false | "D6" |
| A4 | agree | false | false | "hard-rule fix in D7/V2 neighborhood; coordinated edit needed" |
| A5 | agree | false | false | "cycle concealed reliability state flows; prereq for breaker/quarantine" |
| A6 | agree | **true** | false | "D6 hard rule; cycle removal clarifies trust-flow direction; foundational for A6-P9" |
| A7 | own | — | — | owner |
| A8 | agree | false | false | "D6 hard-rule fix; `scheduler/` unit tests can mock `distributed/`" |
| A9 | agree | false | false | "D6 hard rule; cycle harder to reason about than DAG" |
| A10 | agree | false | false | "D6; supports independent scale evolution of distributed layer" |

### A7-P2

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "ISP, D4; aggregator preserves today's vtable; no new indirection" |
| A2 | agree | false | false | "role-split is ISP + improves OCP by narrower seams; merge with A9-P1" |
| A3 | agree | false | false | "accept **only if** A3-P2 resolved first; otherwise LSP drift worse" |
| A4 | agree | false | false | "improves D7 narrower interfaces; each role in glossary + Interfaces-at-a-Glance" |
| A5 | agree | false | false | "ISP+D4; merge with A9-P1; neutral for A5" |
| A6 | agree | false | false | "ISP; reduces `bindings/` attack surface" |
| A7 | own | — | — | owner (merges A9-P1) |
| A8 | agree | false | false | "narrows test-double surface" |
| A9 | agree | false | false | "merged with A9-P1; fewer v-methods than either alone" |
| A10 | agree | false | false | "D3 SRP; merge-candidate with A9-P1" |

### A7-P3

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "D2; typedefs only" |
| A2 | agree | false | false | "D2 abstractions don't depend on details" |
| A3 | agree | false | false | "D2" |
| A4 | agree | false | false | "pure DAG edit; Appendix-B must sync" |
| A5 | agree | false | false | "D2 neutral for A5" |
| A6 | agree | false | false | "D2; no security dimension but cleanly satisfies DIP" |
| A7 | own | — | — | owner |
| A8 | agree | false | false | "A8 test doubles compile without `hal/`" |
| A9 | agree | false | false | "D2; aligns cohesion (D5)" |
| A10 | agree | false | false | "D2" |

### A7-P4

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "D3, D5; struct relocation" |
| A2 | agree | false | false | "lets distributed protocol evolve independently (E4, SRP)" |
| A3 | agree | false | false | "D3" |
| A4 | agree | false | false | "reduces two-owner drift in `transport.md §2.3`" |
| A5 | agree | false | false | "D3/D5; neutral for A5" |
| A6 | agree | false | false | "sharpens trust boundary A6-P1 enumerates; strong A6 synergy" |
| A7 | own | — | — | owner |
| A8 | agree | false | false | "transport narrowed to framing; A8-P7 fault injector backend-agnostic" |
| A9 | agree | false | false | "cohesion (D5)" |
| A10 | agree | false | false | "D5 cohesion" |

### A7-P5

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "D2; no effect on REMOTE_* dispatch hot path" |
| A2 | agree | false | false | "D2/D6" |
| A3 | agree | false | false | "D2" |
| A4 | agree | false | false | "unlocks ADR-008 claim" |
| A5 | agree | false | false | "D2+ADR-008; partial-failure semantics depend on crisp cross-module contracts" |
| A6 | agree | false | false | "narrows attack surface for A6-P9 tenant isolation later" |
| A7 | own | — | — | owner |
| A8 | agree | false | false | "mock `ISchedulerLayer` suffices to test `distributed/` in isolation" |
| A9 | agree | false | false | "D4 narrow coupling" |
| A10 | agree | false | false | "D2+D4" |

### A7-P6

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "D3, D5; module extraction" |
| A2 | agree | false | false | "SRP; natural home for A2-P2 schema" |
| A3 | agree | false | false | "D3/D5" |
| A4 | agree | false | false | "new module doc must follow per-module template" |
| A5 | agree | false | false | "D3/D5; narrows `runtime/` to reliability concerns" |
| A6 | agree | false | false | "D3/D5; no direct security concern" |
| A7 | own | — | — | owner |
| A8 | agree | false | false | "isolates code A8-P5 extends" |
| A9 | agree | false | false | "SRP (D3)" |
| A10 | agree | false | false | "D3; supports rolling-config changes without core rebuild" |

### A7-P7

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "D2, D6; forward-decl contract" |
| A2 | agree | false | false | "D7" |
| A3 | agree | false | false | "D6" |
| A4 | agree | false | false | "eliminates invisible edge" |
| A5 | agree | false | false | "D2/D6 consistency" |
| A6 | agree | false | false | "D2, D6; forward-decl contract explicit" |
| A7 | own | — | — | owner |
| A8 | agree | false | false | "removes hidden `core → memory/transport` edges for test builds" |
| A9 | agree | false | false | "maintenance" |
| A10 | agree | false | false | "D6 hygiene" |

### A7-P8

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "D7; handle-type consolidation" |
| A2 | agree | false | false | "DRY" |
| A3 | agree | false | false | "single-owner invariant supports DS7" |
| A4 | agree | false | false | "textbook D7 rubric-#1 case" |
| A5 | agree | false | false | "D7" |
| A6 | agree | false | false | "D7; `ScopeHandle` consolidation" |
| A7 | own | — | — | owner |
| A8 | agree | false | false | "glossary hygiene A8-P11 benefits from" |
| A9 | agree | false | false | "reduces coupling (D4)" |
| A10 | agree | false | false | "DS7 aligned with A10 lens" |

### A7-P9

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "D7; Python class rename" |
| A2 | agree | false | false | "dedup Python `MemoryError`" |
| A3 | agree | false | false | "D7; aligns with A3-P10 mapping table" |
| A4 | agree | false | false | "trivially right" |
| A5 | agree | false | false | "D7 neutral" |
| A6 | agree | false | false | "D7; helps clear error-mapping for audit" |
| A7 | own | — | — | owner |
| A8 | agree | false | false | "pure Python-surface fix" |
| A9 | agree | false | false | "DRY" |
| A10 | agree | false | false | "D7" |

### A8-P1

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "X5; link-time HAL specialization inlines `CLOCK_MONOTONIC_RAW` — no v-call" |
| A2 | agree | false | false | "DIP; adds one abstract seam at right boundary" |
| A3 | agree | false | false | "X5; drain/submit race test needs it" |
| A4 | agree | false | false | "adds `hal::IClock`; glossary rows; helps determinism claims" |
| A5 | agree | false | false | "X5; prereq for A5-P5 chaos matrix (time-warp tests)" |
| A6 | agree | false | false | "prereq to deterministically test A5-P1 backoff, A6-P2 handshake timeouts, A6-P5 rotation" |
| A7 | agree | false | false | "X5; textbook DIP seam (D2)" |
| A8 | own | — | — | owner |
| A9 | agree | false | false | "single test seam; one interface, two impls" |
| A10 | agree | false | false | "X5; enables sim-accelerated large-scale tests" |

### A8-P2

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "X5; test-only, gated on `enable_test_driver`" |
| A2 | agree | false | false | "exactly minimal seam A9-P2 synthesis amendment endorses" |
| A3 | agree | false | false | "A3-P4/P5 test plans need this" |
| A4 | agree | false | false | "test-only surface; glossary row for `RecordedEventSource`" |
| A5 | agree | false | false | "deterministic-replay substrate for A5-P5 failure scenarios" |
| A6 | agree | false | false | "X5; enables determinism tests of admission/quota paths" |
| A7 | agree | false | false | "X5; test-only; preserves D1/D5" |
| A8 | own | — | — | owner |
| A9 | agree | false | false | "compatible with A9-P2 amendment (single `IEventLoopDriver` seam)" |
| A10 | agree | false | false | "X5; required test surface for A10-P7 verification" |

### A8-P3

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "O3, O5; two-tier: log-scale bucket inc <5 ns; init-time alloc; A1 vetoes insert sequence" |
| A2 | agree | false | false | "adopt under A2-P9 (versioned stats schema)" |
| A3 | agree | false | false | "branchless insert + pre-allocated buckets per synthesis two-tier" |
| A4 | agree | false | false | "alerts in `07-cross-cutting-concerns.md` become traceable to concrete fields" |
| A5 | agree | false | false | "histograms keyed to SLO-breach alerts" |
| A6 | agree | false | false | "primitives for failed-handshake/breaker-trip/quota-breach/audit-degraded alerts" |
| A7 | agree | false | false | "O3; stats structs are per-module internal — D5 fit" |
| A8 | own | — | — | owner |
| A9 | agree | false | false | "single data-surface addition; no new runtime abstractions" |
| A10 | agree | false | false | "O3; direct input to verify P4 scaling claims" |

### A8-P4

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "O4, X6; cold diagnostic endpoint; seqlock/relaxed-atomic reads" |
| A2 | agree | false | false | "additive (E4); merge with A5-P6" |
| A3 | agree | false | false | "makes A3-P3/P4 scenarios inspectable" |
| A4 | agree | false | false | "glossary `StateDump`, `DumpOptions`; merged with A5-P6" |
| A5 | agree | false | false | "O4+X6; co-owned with A5-P6 (dump_state = evidence surface)" |
| A6 | agree | false | false | "operationally essential **but** default output must be scoped to single `logical_system_id`; unscoped requires admin token" |
| A7 | agree | false | false | "O4/X6; `describe()` on each module honors D4" |
| A8 | own | — | — | owner (pairs with A5-P6) |
| A9 | agree | false | false | "merged with A5-P6 watchdog; single paired fast-track" |
| A10 | agree | false | false | "X6+O4" |

### A8-P5

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "O5; alert eval on low-priority collector thread" |
| A2 | agree | false | false | "E3 config over hardcoding + E4 new sink interface; synth amendment: ADR + known-dev" |
| A3 | agree | false | false | "O5+E3" |
| A4 | agree | false | false | "`AlertRule` schema + `on_alert` API; glossary rows" |
| A5 | agree | false | false | "O5+E3+X8; surfaces A5-P8 degradation policies operationally" |
| A6 | agree | false | false | "composes with A6-P6 audit events" |
| A7 | agree | false | false | "O5/E3/X8; `IMetricsSink` sits next to `ILogSink`/`IAuditSink`" |
| A8 | own | — | — | owner |
| A9 | disagree | false | false | "ships two systems at once (rule format + wire sink); synth amendment (file location + OTEL as known-deviation) would flip to agree" |
| A10 | agree | false | false | "O5; operability-at-scale" |

### A8-P6

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "O1 distributed; offline/export merge; no hot-path" |
| A2 | agree | false | false | "trace time-alignment contract (E6)" |
| A3 | agree | false | false | "A3-P13 depends for causal-ordering validation" |
| A4 | agree | false | false | "adds PTP/NTP dependency + `IClockSync`; matches A3-P13" |
| A5 | agree | false | false | "O1+DS3; merged distributed traces respect causality" |
| A6 | agree | false | false | "prereq for forensic via A6-P12 REPLAY" |
| A7 | agree | false | false | "O1; `IClockSync` co-located with `IClock` in `hal/`" |
| A8 | own | — | — | owner |
| A9 | agree | false | false | "O1; doc + one contract" |
| A10 | agree | false | false | "O1; essential for cross-node scalability debugging" |

### A8-P7

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "X5, R6; sim-only; never compiled into onboard" |
| A2 | agree | false | false | "`IFaultInjector` seam coincides with A5-P5; agree once" |
| A3 | agree | false | false | "X5; reusable for A3-P3 + A5-P5" |
| A4 | agree | false | false | "sim-only; needs `modules/hal.md §2.x` cross-refs" |
| A5 | agree | false | false | "X5+DfT; co-owned with A5-P5 (seam vs matrix)" |
| A6 | agree | false | false | "R6; sim-only; zero prod cost; fuzzes A6-P3/P2" |
| A7 | agree | false | false | "X5/R6; sim-only symbol" |
| A8 | own | — | — | owner (co-owner with A5-P5) |
| A9 | agree | false | false | "sim-only; does not touch hot path" |
| A10 | agree | false | false | "R6 sim-only" |

### A8-P8

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "O3; two-tier: default L1 strips L2 AICore ring emits; < 50 ns amortized" |
| A2 | agree | false | false | "new protocol must carry version (E6) + handshake gate (E2)" |
| A3 | agree | false | false | "Level-2 opt-in; two-tier preserved" |
| A4 | agree | false | false | "closes open question at `modules/profiling.md:461`" |
| A5 | agree | false | false | "O3; closes on-core observability gap for R4 leaf-level" |
| A6 | agree | false | false | "Level-2 only; default L1 strip" |
| A7 | agree | false | false | "O3; HAL capability bit is the right seam" |
| A8 | own | — | — | owner |
| A9 | agree | false | false | "boundary commitment, not new runtime surface" |
| A10 | agree | false | false | "O4+X6" |

### A8-P9

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "O3, O5; existing drop counters + relaxed-atomic read cold" |
| A2 | agree | false | false | "additive; no contract break" |
| A3 | agree | false | false | "O3/O5" |
| A4 | agree | false | false | "two additional alert rows in §7.2.8" |
| A5 | agree | false | false | "silent profiling drops invalidate R6; making alertable is reliability gain" |
| A6 | agree | false | false | "same pattern for audit-sink degraded state (fold into A6-P6)" |
| A7 | agree | false | false | "O3/O5" |
| A8 | own | — | — | owner |
| A9 | agree | false | false | "O5 SLO-driven alerts; tiny addition" |
| A10 | agree | false | false | "O5" |

### A8-P10

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "O2; KV emit only above threshold" |
| A2 | agree | false | false | "E6 stability of log field names" |
| A3 | agree | false | false | "composes with A6-P6 audit trail" |
| A4 | agree | false | false | "closes `modules/profiling.md:463` open question" |
| A5 | agree | false | false | "O2; improves R6 post-mortem quality" |
| A6 | agree | false | false | "natural substrate for A6-P6 audit events" |
| A7 | agree | false | false | "O2" |
| A8 | own | — | — | owner |
| A9 | agree | false | false | "simpler than ad-hoc log strings once adopted" |
| A10 | agree | false | false | "O2" |

### A8-P11

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "X5; CI test-suite" |
| A2 | agree | false | false | "enforcement mechanism for A2-P5 BC policy" |
| A3 | agree | false | false | "X5/DfT" |
| A4 | agree | false | false | "adds `modules/hal.md §9`" |
| A5 | agree | false | false | "X5; safeguard against sim-drift" |
| A6 | agree | false | false | "X5; no security concern" |
| A7 | agree | false | false | "X5; keystone of D2 HAL discipline" |
| A8 | own | — | — | owner |
| A9 | agree | false | false | "X5; aligns with A1-P13 HAL tightening" |
| A10 | agree | false | false | "X5+X7" |

### A8-P12

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "O1; compiled out below L2; <100 ns at L2 per `modules/profiling.md:334`" |
| A2 | agree | false | false | "E6 schema stability for phase-identifying enums" |
| A3 | agree | false | false | "enables A3-P3/P6 traceability validation" |
| A4 | agree | false | false | "named IDs in `modules/profiling.md §2.5`; glossary row" |
| A5 | agree | false | false | "O1+X9; A5-P6 deadman keys off documented phase" |
| A6 | agree | false | false | "audit events can adopt same stable-id discipline" |
| A7 | agree | false | false | "O1 stable IDs" |
| A8 | own | — | — | owner |
| A9 | agree | false | false | "O1 trace IDs; lightweight" |
| A10 | agree | false | false | "O1" |

### A9-P1

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "G2; smaller vtable, shorter hot-path footprint; merge with A7-P2" |
| A2 | agree | false | false | "merged with A7-P2; sharpens OCP" |
| A3 | agree | false | false | "aligned with A3-P2 (one return contract); merges with A7-P2" |
| A4 | agree | false | false | "merged with A7-P2; improves D7" |
| A5 | agree | false | false | "G2; merges with A7-P2; neutral for A5" |
| A6 | agree | false | false | "G2; reduces C/Python boundary attack surface" |
| A7 | agree | false | false | "D4/G2; merged into A7-P2" |
| A8 | agree | false | false | "narrower test surface for A8-P2" |
| A9 | own | — | — | owner (merges with A7-P2) |
| A10 | agree | false | false | "G2 KISS; merge-candidate with A7-P2" |

### A9-P2

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "G2, G4; reduces v-dispatch; conditional on A8-P2 test seam" |
| A2 | disagree | false | **true** | "OCP regression at 4 change axes; amend to keep policy enums closed + ADR-listed v2 extensions" |
| A3 | abstain | false | false | "A2-vs-A9 question; A3 neutral if no LSP drift" |
| A4 | abstain | false | false | "core A2↔A9 extensibility debate" |
| A5 | abstain | false | false | "deployment-mode count does not affect R1–R6" |
| A6 | abstain | false | false | "no direct security dimension" |
| A7 | agree | false | false | "D4/D5/G2; removing interfaces with one extender each reduces fat interfaces" |
| A8 | disagree | false | false | "X5 §8.2 DfT; deletion acceptable only if single `IEventLoopDriver` test seam survives (synthesis bridge)" |
| A9 | own | — | — | owner (amended: keep single test seam) |
| A10 | abstain | false | false | "would agree if A9 commits to keeping one `IEventLoopDriver` test-only seam" |

### A9-P3

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "G2; HCCL native path can return later as `ICollectiveOps` extension" |
| A2 | agree | false | false | "ISP; collectives land as Orchestration Functions later" |
| A3 | abstain | false | false | "Q4 resolution; no direct correctness impact" |
| A4 | abstain | false | false | "depends on Q4 resolution" |
| A5 | abstain | false | false | "neutral provided partial-failure collective maps to FailurePolicy" |
| A6 | agree | false | false | "G2; minimal base transport interface" |
| A7 | agree | false | false | "D4; `IHorizontalChannel` shrinks; collectives as `ICollectiveOps` sibling" |
| A8 | abstain | false | false | "orthogonal to A8" |
| A9 | own | — | — | owner |
| A10 | agree | false | false | "G2; roadmap note satisfies A2 concern" |

### A9-P4

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "G2, DRY; one fewer admission branch" |
| A2 | agree | false | false | "re-adding later is additive (E4)" |
| A3 | agree | false | false | "A3-P7 works on `spmd.has_value()`+`tasks.size()`" |
| A4 | agree | false | false | "removes discriminant runtime doesn't branch on; D7 win" |
| A5 | agree | false | false | "G2+DRY; neutral for A5" |
| A6 | agree | false | false | "G2, DRY; no security dimension" |
| A7 | agree | false | false | "G2/D7; `Kind` is vestigial" |
| A8 | agree | false | false | "simplifies A8-P12 PhaseId attribution" |
| A9 | own | — | — | owner |
| A10 | agree | false | false | "G2+DRY" |

### A9-P5

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "G2, DRY; enum unification" |
| A2 | agree | false | false | "removes drift risk for E6 when new outcomes added" |
| A3 | agree | false | false | "required by A3-P2 amended return contract" |
| A4 | agree | false | false | "clearest D7-coded win in catalog" |
| A5 | agree | false | false | "DRY; helps A5-P8 avoid third enum" |
| A6 | agree | false | false | "G2, DRY; mandatory for A6-P13 reuse" |
| A7 | agree | false | false | "D7/DRY" |
| A8 | agree | false | false | "single status dimension for A8-P3 histograms" |
| A9 | own | — | — | owner |
| A10 | agree | false | false | "DRY" |

### A9-P6

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "G2; A1 no hot-path stake in sim modes; A6-P12 becomes moot" |
| A2 | disagree | false | **true** | "closes two extension points without migration plan; amend to enum-open + Q-record" |
| A3 | agree | false | false | "shrinks FR-10 scope; A6-P12 moot" |
| A4 | abstain | false | false | "neutral on scope; if accepted, edits mechanical" |
| A5 | abstain | false | false | "R6 reachable with FUNCTIONAL alone if A8-P2 lands" |
| A6 | disagree | false | false | "REPLAY is *one* forensic-analysis tactic; prefer FUNCTIONAL+REPLAY in v1, defer PERFORMANCE" |
| A7 | agree | false | false | "G2; reduces `hal/sim/` SRP burden" |
| A8 | disagree | false | false | "X5 §8.2 DfT 'Reproducible failures'; cancel A8 REPLAY-based tactic; prefer keep REPLAY, defer only PERFORMANCE" |
| A9 | own | — | — | owner (amended: ADR-011-R2 with named triggers) |
| A10 | agree | false | false | "G2 YAGNI for v1; ADR-deferral sufficient" |

### A9-P7

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "G2; startup-time only" |
| A2 | agree | false | false | "DRY; additive struct extension if re-separation needed" |
| A3 | agree | false | false | "no correctness impact" |
| A4 | agree | false | false | "A4-P5 amends to drop `SourceCollectionConfig` glossary row" |
| A5 | agree | false | false | "G2; neutral for A5" |
| A6 | agree | false | false | "G2; no security dimension" |
| A7 | agree | false | false | "D5/D7" |
| A8 | abstain | false | false | "config-struct refactor; A8-P2 references merged name trivially" |
| A9 | own | — | — | owner |
| A10 | agree | false | false | "G2" |

### A9-P8

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "G2, SRP; doc scope" |
| A2 | agree | false | false | "no E-rule impact" |
| A3 | agree | false | false | "SRP" |
| A4 | agree | false | false | "removes out-of-scope content" |
| A5 | agree | false | false | "G2+SRP; no R1-R6 impact" |
| A6 | agree | false | false | "G2, SRP; no security dimension" |
| A7 | agree | false | false | "D1 (scope of runtime doc)" |
| A8 | agree | false | false | "A8 doesn't depend on artifact format in-scope" |
| A9 | own | — | — | owner |
| A10 | agree | false | false | "D3 separate concern" |

### A10-P1

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "P4, P6, DS7; two-tier: default single-threaded retains RW-lock fast path; sharded opt-in" |
| A2 | agree | false | false | "shard count as `LevelParam` meets E3; merged with A1-P14" |
| A3 | agree | false | false | "aligns with A1-P2 shared `shard_count` LevelParam" |
| A4 | agree | false | false | "MAY→SHOULD in `02-scheduler.md`; aligns A1-P14" |
| A5 | agree | false | false | "P4 reliability-adjacent via scalability" |
| A6 | agree | false | false | "shard key should carry `logical_system_id` to avoid cross-tenant collisions" |
| A7 | agree | false | false | "P4/DS6; internal to `scheduler/`" |
| A8 | agree | false | false | "deterministic sizing observable via A8-P3 per-shard histograms" |
| A9 | agree | false | false | "doc-only; merged with A1-P14" |
| A10 | own | — | — | owner |

### A10-P2

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "P4, R5; control plane off hot path; merged into A5-P3" |
| A2 | abstain | false | false | "merged into A5-P3 per register; this row is alias" |
| A3 | agree | false | false | "accept compromise (v1 fail-fast + v2 decentralized); merges with A5-P3" |
| A4 | agree | false | false | "merged with A5-P3; adds ADR + §5.4.0 `Coordinator Membership`" |
| A5 | agree | false | false | "R5+P4; critical for R5 multi-Pod; merged with A5-P3" |
| A6 | agree | false | false | "A6-P2 handshake must bind `cluster_view.generation`" |
| A7 | agree | false | false | "merged into A5-P3" |
| A8 | agree | false | false | "A8-P6 amendment tracks this merge" |
| A9 | disagree | false | false | "aligned with fail-fast v1, not decentralised branch (G2 YAGNI for v1 scope)" |
| A10 | own | — | — | owner (split: P2a merged to A5-P3, P2b v2 roadmap) |

### A10-P3

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "DS6; consistency-model doc, no runtime effect" |
| A2 | agree | false | false | "documentary; supports E6 audit" |
| A3 | agree | false | false | "directly answers rubric #4 implicit assumptions finding" |
| A4 | agree | false | false | "direct A4 alignment; complements A10-P8 ownership diagram" |
| A5 | agree | false | false | "DS6 reliability-audit artefact" |
| A6 | agree | false | false | "explicitly add `audit_ring` + `rkey_scope_table` rows" |
| A7 | agree | false | false | "DS6; doc in `07-cross-cutting-concerns.md`" |
| A8 | agree | false | false | "closes references-counters-not-exposed gap" |
| A9 | agree | false | false | "DS6 (required)" |
| A10 | own | — | — | owner |

### A10-P4

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "DS1; classification doc" |
| A2 | agree | false | false | "supports E1 (identify change axes)" |
| A3 | agree | false | false | "DS1" |
| A4 | agree | false | false | "direct A4 alignment with rubric #2" |
| A5 | agree | false | false | "documents where Deviation 2 applies" |
| A6 | agree | false | false | "audit ring + rkey table must be classified" |
| A7 | agree | false | false | "DS1; doc in `03-development-view.md`" |
| A8 | agree | false | false | "aligns with A8-P4 `dump_state()` surface design" |
| A9 | agree | false | false | "helps reasoning about DS1" |
| A10 | own | — | — | owner |

### A10-P5

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "P6; merged into A1-P6; fan-out=1 fast path ships descriptor unchanged" |
| A2 | agree | false | false | "merged into A1-P6 per register" |
| A3 | agree | false | false | "merges into A1-P6 per register" |
| A4 | agree | false | false | "merged with A1-P6 per register" |
| A5 | agree | false | false | "merged into A1-P6; neutral for A5" |
| A6 | agree | false | false | "P6+S2; per-peer minimum-information — least-privilege aligned" |
| A7 | agree | false | false | "P6; merged into A1-P6" |
| A8 | agree | false | false | "observable via `bytes_per_peer` metric" |
| A9 | agree | false | false | "merged with A1-P6; net-simpler combined" |
| A10 | own | — | — | owner (absorbed into A1-P6) |

### A10-P6

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "P4, R5; dedicated heartbeat thread; zero admission/dispatch cost" |
| A2 | agree | false | false | "thresholds as config (E3)" |
| A3 | agree | false | false | "consistent with A5-P3 fail-fast" |
| A4 | agree | false | false | "`heartbeat_miss_threshold_fast` config + glossary" |
| A5 | agree | false | false | "R5 direct win; **hysteresis interaction with A5-P2/A5-P9 must be specified**" |
| A6 | agree | false | false | "R5; shortens stale-credential window" |
| A7 | agree | false | false | "R5/P4; transport-level fast-fail piggyback" |
| A8 | agree | false | false | "`NodeLost` observable within SLO" |
| A9 | agree | false | false | "single tunable (E3)" |
| A10 | own | — | — | owner |

### A10-P7

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "P4; two-tier: `shards=1` default; LevelParams opt-in; global outstanding counter preserves bound" |
| A2 | agree | false | false | "`shards=1` default + LevelParam opt-in; E1 fast path + E3 scaling knob" |
| A3 | agree | false | false | "two-tier answers A1 veto concern" |
| A4 | agree | false | false | "cross-linked from `modules/scheduler.md §4`" |
| A5 | agree | false | false | "P4; neutral for A5" |
| A6 | agree | false | false | "quota must stay partition-aware (A6-P13)" |
| A7 | agree | false | false | "P4; two-tier preserves D1 simplicity" |
| A8 | agree | false | false | "P4/DS6; preserves earliest-first + opens admission-throughput observability" |
| A9 | disagree | false | false | "sharded with HPI=`relayouts` for v1 with no bottleneck is premature; synth amendment (default shards=1) would flip" |
| A10 | own | — | — | owner |

### A10-P8

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "DS7, P6; diagram only" |
| A2 | agree | false | false | "supports E5 change-impact analysis" |
| A3 | agree | false | false | "complements LSP-across-layers audit" |
| A4 | agree | false | false | "direct A4 rubric #2 improvement; strong support" |
| A5 | agree | false | false | "DS7 reliability-audit artefact" |
| A6 | agree | false | false | "audit-log + rkey-scope paths should appear on diagram" |
| A7 | agree | false | false | "DS7 diagram" |
| A8 | agree | false | false | "pure O4 win" |
| A9 | agree | false | false | "documentation (DS7)" |
| A10 | own | — | — | owner |

### A10-P9

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "DS6, P4; log-keyed assignment" |
| A2 | agree | false | false | "E2/E5 disciplined incompatibility declaration" |
| A3 | agree | false | false | "correctness invariant for idempotency" |
| A4 | agree | false | false | "small but precise D7 discipline" |
| A5 | agree | false | false | "DS6+DS4; resolves non-idempotency hazard" |
| A6 | agree | false | false | "assignment log = audit-log emission site" |
| A7 | agree | false | false | "DS6; minimal assignment-log seam in `distributed/`" |
| A8 | agree | false | false | "auditable by A8-P11 contract tests" |
| A9 | agree | false | false | "small correctness guard" |
| A10 | own | — | — | owner |

### A10-P10

| reviewer | vote | blocking | override_request | rationale |
|----------|------|----------|------------------|-----------|
| A1 | agree | false | false | "P6, DS6; merged into A1-P14" |
| A2 | agree | false | false | "merged into A1-P14; doc-only" |
| A3 | agree | false | false | "merges into A1-P14 per register" |
| A4 | agree | false | false | "merged with A1-P14" |
| A5 | agree | false | false | "merged; no reliability objection" |
| A6 | agree | false | false | "P6; aligns with A6-P3 no-unbounded-growth discipline" |
| A7 | agree | false | false | "P6/X4; merged with A1-P14" |
| A8 | agree | false | false | "X4; foundational for A8-P3 cache-friendliness" |
| A9 | agree | false | false | "merged with A1-P14; doc only" |
| A10 | own | — | — | owner (absorbed into A1-P14) |

---

## (C) Override Requests Summary

Only three override-request rows were filed across all ten reviewers. All were filed by A2 against A1/A9 proposals; no other reviewer filed any. No override was filed against any A1 vote (A1 cast zero vetoes this round — every hot-path-touching proposal carries a documented two-tier bridge that A1 accepted).

| proposal_id | reviewer | vote | context |
|-------------|----------|------|---------|
| A1-P9 | A2 | disagree + override_request | "Bitmask encoding at ≤64 groups creates an E1 ceiling. Synthesis requires split-group fallback; until A1 commits the 2×64 fallback in the edit sketch (not prose), the proposal encodes a hard cap the roadmap already expects to exceed." |
| A9-P2 | A2 | disagree + override_request | "Removing `IEventCollectionPolicy`/`IExecutionPolicy` *as interfaces* is an OCP regression at four already-documented change axes (FULLY_SPLIT, SPLIT_DEFERRED, Batched, Dedicated). Amend to keep policy enums closed for v1 *with ADR-recorded future-extension names* and preserve the abstract policy hook behind a `final` concrete v1 impl." |
| A9-P6 | A2 | disagree + override_request | "Deferring PERFORMANCE/REPLAY wholesale closes two documented extension points without the migration plan E5 requires. REPLAY also underpins A6-P12 and A8 debugging harness. Amend to ship FUNCTIONAL-only *implementations* but keep `SimulationMode` enum open + ADR declaring v2 scope with named Q records." |

---

## (D) Blocking Objections Summary

Only two `blocking=true` rows were filed this round; both attach to the same proposal (A7-P1) and invoke the same hard rule (D6, no dependency cycles). A7-P1 is universally supported, so these blocking votes align with majority sentiment — they are "hard-rule" flags rather than vetoes against the proposal.

| proposal_id | reviewer | vote | rule | reason |
|-------------|----------|------|------|--------|
| A7-P1 | A2 | agree + blocking | D6 (hard rule) | "D6 forbids dependency cycles; cycle removal is blocker-level from A2's rubric." |
| A7-P1 | A6 | agree + blocking | D6 (hard rule) | "Cycle removal clarifies trust-flow direction between `scheduler/` (tenant-local) and `distributed/` (untrusted peer boundary). Foundational for A6-P9." |

A1 (hot-path veto authority) filed zero blocking objections: "A1 veto applications this round: 0 (all hot-path proposals accepted with documented two-tier bridges)."

All other reviewers (A3, A4, A5, A7, A8, A9, A10) also filed zero blocking votes in round 2.

---

## (E) Merge Register Outcomes

The six synthesis-proposed merges (from `round-1/synthesis.md` §Semantic-Duplicate Merge Register), and each reviewer's position. Every reviewer accepted every merge, with some conditionals noted.

| Merge | A1 | A2 | A3 | A4 | A5 | A6 | A7 | A8 | A9 | A10 |
|-------|----|----|----|----|----|----|----|----|----|-----|
| **A1-P6 ← A10-P5** (Distributed payload hygiene) | accept (owner) | accept | accept | accept | accept | accept | accept | accept | accept | accept (absorbed) |
| **A1-P14 ← A10-P10** (producer_index placement + cap + layout) | accept (owner) | accept | accept | accept | accept | accept | accept | accept | accept | accept (absorbed) |
| **A5-P3 ← A10-P2** (coordinator failover / decentralize) | accept (endorse; prefer v1 fail-fast, v2 decentralize) | accept (v1 fail-fast + Q-record is E5-compliant) | accept conditional on v1=fail-fast | accept (closes Q5 with single coherent ADR) | accept (co-owner; v1 fail-fast, v2 decentralized roadmap) | accept **conditional** that merged proposal bind A6-P2 handshake to `cluster_view.generation` | accept (both live in `distributed/`; D4) | accept (A8-P6 amendment tracks this; trace-merge owner adapts to decentralized coordination) | accept (A9 aligned with v1 fail-fast branch only, not decentralized) | accept **split**: P2a (v1 fail-fast) into A5-P3 + P2b (v2 decentralized) roadmap-only |
| **A5-P6 ↔ A8-P4** (watchdog + dump_state pair) | accept (endorse) | accept (pair-merge) | accept | accept | accept (co-owner) | accept **conditional** on A8-P4 adopting tenant-scoping (S2) | accept (`dump_state()` is evidence surface) | accept **as pairing, not absorption** (keep both IDs; A8 prefers absorbing A5-P6 into A8-P4 to keep `dump_state()` ID) | accept (merged with dump_state evidence surface) | accept |
| **A7-P2 ← A9-P1** (role-split + single `submit()`) | accept (endorse) | accept (role-split is ISP; improves OCP) | accept **conditional on A3-P2 resolved first** (option-b return contract) | accept (one edit to interface definition) | accept (neutral for A5) | accept (narrows `bindings/` attack surface) | accept (owner) | accept (narrower `ISchedulerLayer` helps test doubles) | accept (owner; A9-P1 absorbed into A7-P2) | accept |
| **A6-P8 ← A1-P11 (partial)** (per-arg validation fused with per-arg budget) | accept (co-owner; A1-P11 covers marshaling, A6-P8 covers validation — same 200 ns envelope) | accept (single boundary contract under one vote) | accept **and extend** to also absorb A3-P7 (three-way convergence on Python↔C boundary) | accept (avoids cross-proposal drift) | accept (S3 win) | accept **with split**: A6-P8a (structural validation) merges into A1-P11; A6-P8b (`max_import_bytes` deployment cap) remains standalone A6-owned | accept (DLPack fits per-arg budget at `bindings/`) | accept | accept (merged; DRY+S3) | accept |

Summary of conditions / notes on merges:

- **A5-P3 ← A10-P2:** Every reviewer accepts v1 fail-fast; A9 explicitly endorses only the fail-fast branch and disagrees with the decentralized-coordinator branch; A10 splits the proposal so v2 decentralization is a roadmap-only ADR rather than an in-scope v1 item; A6 adds a binding requirement (`cluster_view.generation` in handshake) without which it would block — this is the clearest cross-aspect tension on the merged form.
- **A5-P6 ↔ A8-P4:** A6 conditions acceptance on tenant-scoping of `dump_state()` (S2). A8 prefers the absorption direction be "A5-P6 into A8-P4" (to keep `dump_state()` ID as the artifact the watchdog references), but accepts either ordering.
- **A7-P2 ← A9-P1:** A3 conditions its acceptance on A3-P2 being resolved first (otherwise LSP drift between role interfaces would be worse than today's fat interface). A7 amended the merged form to split at the header level only (aggregator retained).
- **A6-P8 ← A1-P11:** A6 splits its own proposal — only the structural-validation half (A6-P8a) is absorbed into A1-P11; the byte-cap half (A6-P8b) remains a small standalone config-only proposal. A3 proposes extending the merge to also absorb A3-P7.

All six merges accepted by all ten reviewers. No merge was rejected by any reviewer.
