# Aspect A8: Testability & Observability — Round 2

## Metadata

- **Reviewer:** A8
- **Round:** 2
- **Target:** `/data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/` (full doc set)
- **Runtime Mode:** yes
- **Timestamp (UTC):** 2026-04-19T00:00:00Z

---

## 1. Rubric Checks

Unchanged from round 1. The six rubric checks (clock/timer injection, trace-id plumbing, structured logs, health metrics, runtime debuggability, SLO-driven alerts) and the four A8 spot checks (step-driven scheduler, fault-injection seam, distributed trace alignment, drop-counter observability) remain **Weak** until the A8-P1..P12 edits land. Peer R1 reviews surface new findings that *reinforce* these gaps rather than close them:

- A3-P4 / A3-P13 expose observability holes (silent-timeout consumer, undefined cross-node ordering) that feed A8-P6's time-alignment and A8-P12's PhaseId proposals.
- A5-P5 / A5-P6 overlap with A8-P7 (`IFaultInjector`) and A8-P4 (`dump_state`) and are routed in the Merge Register.
- A6-P6 (audit trail), A6-P10 (capability-scoped sinks), A6-P12 (signed REPLAY trace), A2-P9 (versioned trace schema) all augment A8-P10 (structured KV) and A8-P6 (distributed trace alignment).
- A1-P7 (per-thread profiling sequence) structurally enables A8-P12 at Level ≥ 2.
- A7-P3 / A7-P7 tighten `core/` and forward-decl discipline that make `IClock` (A8-P1) trivially injectable without pulling `hal/` into test targets.

No new rubric-level regressions were introduced by round-1 peer proposals.

## 2. Pros (delta vs round 1)

Peer R1 proposals that strengthen A8's position and should be recorded:

- A1-P10 (workload-level profiling-overhead CI gate) is explicitly co-owned by A8 per the synthesis. Pass condition for the NFR-1 "zero-overhead when disabled" / `< 1%` L1 claim (`07-cross-cutting-concerns.md:195`) becomes enforceable. Cites **P1, O3**.
- A1-P7 (per-thread local sequence + offline merge) removes the cross-socket ping-pong that currently dominates `modules/profiling.md:277-286`. Without this, A8-P12 (stable PhaseIds) and any L2 emission exceed the `< 100 ns` per-Phase target on multi-socket hosts. Cites **X2, X3, O3**.
- A6-P6 (security audit trail via `IAuditSink` alongside `ILogSink`) lands naturally as a *sink variant* on the profiling sink plug-in interface — confirms A8's sink model scales to S6 without re-architecture. Cites **O1, S6**.
- A2-P9 + A6-P12 jointly formalize a versioned, signed REPLAY trace format. Combined with A8-P6 (time alignment) and A8's existing O1 correlation-id plumbing, distributed traces become machine-verifiable across nodes. Cites **O1, E6, S3**.
- A3-P4 (`DEP_FAILED` event) eliminates the "eventually timeout" hand-wave at `06-scenario-view.md:139-140`. Deterministic failure-propagation events are traceable with the existing `TraceEvent`/`ErrorContext` correlation-id — A8's foundation needs no change. Cites **O1**.
- A3-P13 (cross-node FIFO-per-chain ordering + `[ASSUMPTION]` markers) provides the transport-ordering premise that A8-P6's trace-merge algorithm relies on. Cites **O1**.
- A7-P3 (move `DeviceAddress` / `NodeId` typedefs from `hal/` into `core/`) and A7-P7 (documented forward-decl contract) eliminate a hidden `hal/` dependency that currently blocks building test doubles without the sim leaf. Cites **X5**.

## 3. Cons (delta vs round 1)

Peer R1 proposals that expose new A8-relevant defects:

- **Peer-failure-detection time (A10-P6) is 3 s today** (`modules/distributed.md:407-408`). A8's O5 alert rows at `07-cross-cutting-concerns.md:137-146` cite `NodeLost` but assume timely detection; at 3 s, the straggler-observability SLO is unmeetable at scale.
- **Coordinator trace-merge assumes a single owner (A5-P3 / A10-P2).** Cross-node trace alignment (A8-P6) currently relies on "Coordinator Node merges all node traces". Decentralizing coordination — the direction A10-P2 proposes — makes the merge author ambiguous unless A8-P6 explicitly picks *any-owner* semantics. Cross-reference added to A8-P6 amendment below.
- **`dump_state()` (A8-P4) must be paired with scheduler-thread watchdog (A5-P6)** to close O4 + R5 silent-hang gap; merge register proposes coupling them. Accepted — see Merge Register response.

## 4. Proposals — new in round 2

None. All round-2 A8 changes are revisions of A8-P1…A8-P12; no new `A8-PN` is added. Round-3 may add one if merges produce a composite.

## 5. Revisions of own proposals

| id | action | amended_summary (if amend/split) | reason |
|----|--------|----------------------------------|--------|
| A8-P1 | **defend** | — | No peer objections surfaced. Link-time HAL specialization (`modules/hal.md:247,430`) already makes `IClock` zero-cost in release builds; A1 hot-path veto is pre-empted by keeping `SystemClock::now_ns()` inlined. Required by A8-P2 (time-travel tests), A8-P6 (skew injection), A8-P7 (fault timing), A5-P6 (deadman slot), A5-P7 (`IMemoryOps` timeout). |
| A8-P2 | **amend** | Expose `EventLoopRunner::step(max_events)` as the **single** `IEventLoopDriver` test-only seam; pair with `RecordedEventSource` in `core/`. Commit that this seam is test-only (`enable_test_driver` build flag) and is the only event-loop driveability surface in v1, compatible with A9-P2's retention of one test seam. | A9-P2 would remove four deployment modes and the `IEventCollectionPolicy` / `IExecutionPolicy` pluggables. Per synthesis "Bridge: one `IEventLoopDriver` boundary (test-only) + closed enums for policies", A8-P2 is preserved if narrowed to a single test seam. Narrowing answers A9's YAGNI objection without losing X5 testability. |
| A8-P3 | **defend** | — | Hot-path tier already documented: branchless single-bucket increment ≤ 5 ns, histogram memory pre-allocated at init, percentile computation on the cold reader. A1's expected contest (round-1 tensions table) is addressed by the two-tier path; A1 can still veto the exact insert sequence, which A8 accepts as refinement. The "1 KB per histogram" overhead is explicit and configurable. |
| A8-P4 | **defend (accept pairing with A5-P6)** | Keep as written. Accept the Merge Register routing that pairs A5-P6 (scheduler watchdog) with A8-P4 (`dump_state`) as a fast-track bundle: the watchdog detects the silent hang; `dump_state()` supplies the evidence surface for the incident responder. | Synthesis merge register §"A5-P6 | A8-P4". Pairing is correct: same incident-response workflow, same O4 + R5 rubric. No proposal substance changes. |
| A8-P5 | **amend** | Declare alert rules in an **external file** referenced by `DeploymentConfig.alerting_rules_path` (4-field schema: `{metric, op, threshold, window_ms, severity}`); register **OpenMetrics/Prometheus sink as a known-deviation ledger entry** in `10-known-deviations.md` (opt-in; not on the default ship path). Keep `Runtime::on_alert(callback)` as the in-process hook. | Synthesis Open Disputes §"A8-P5: declare external alert-rule file location; OTEL sink as a known deviation ledger entry". Addresses A9's "no plugin infrastructure" objection: the primary change is *format*, not a new subsystem. |
| A8-P6 | **amend** | Specify distributed trace alignment algorithm against A3-P13's declared transport ordering assumptions (`[ASSUMPTION] per-(source, destination, task-chain) FIFO`). Declare that the merge owner is **the coordinator when centralized (v1) and any-owner-deterministic when A10-P2 decentralization lands**. Require `IClockSync::offset_ns()` (A8-P1 companion) and bounded `skew_max_ns`. | Two peer proposals (A3-P13 ordering, A10-P2 decentralization) directly affect merge semantics. Amendment makes A8-P6 forward-compatible with A10-P2 while keeping the v1 story unchanged. |
| A8-P7 | **amend (joint co-ownership with A5)** | Keep the `IFaultInjector` sim-only seam specification; route joint ownership with A5-P5 (chaos harness + mandatory-scenario matrix). A8-P7 defines the **seam**; A5-P5 defines the **matrix** of scenarios that must hit it. | Synthesis §A5-P5 co-owns A8-P7. Both reviewers converged in round-1 tensions. No substance change; clarifies division of work between the design-doc artifacts. |
| A8-P8 | **defend** | — | Level-2 opt-in; default `PROFILING_LEVEL = L1_Coarse` strips AICore ring emits entirely (`modules/profiling.md:399`). Fast path = today. A1 tension resolved by compile-time strip; Level-2 overhead is an opt-in slow path only, activated by operators (e.g., triggered by A8-P5 SLO alerts). Closes open question at `modules/profiling.md:461`. |
| A8-P9 | **defend** | — | Peer review uniformly supportive; makes A8-P3 alert rows implementable for the drop-counter case; A1 cost is one relaxed-atomic read in the cold stats path. |
| A8-P10 | **amend** | Adopt `Logger::log_kv(Severity, category, {kv...}, correlation_id)` as primary surface; **align with A6-P10 capability-scoping**: every KV event carries `{correlation_id, layer_id, category, severity, logical_system_id}` so capability-scoped sinks (A6-P10) can filter without re-parsing the message body. | A6-P10 introduces per-sink capability filtering keyed on `logical_system_id` and `domains`. A KV-first log shape makes that filter a branch, not a parse — fusing O2 with S2. |
| A8-P11 | **defend** | — | HAL contract test suite is CI plumbing, not a runtime artifact; no hot-path cost; peer feedback uniformly supportive (reinforced by A7-P3/P7 which make `core/` buildable without `hal/`, so the contract tests can isolate the HAL). |
| A8-P12 | **defend** | — | Stable `PhaseId`s require A1-P7 (per-thread sequence) to maintain the `< 100 ns` target under multi-socket load. Defend on that basis; A1-P7 is already in A1's proposal set and A1 is expected to carry it. If A1-P7 is rejected, A8-P12 would amend to Level-≥2-only (already the case — stripped at L1). |

## 6. Votes on peer proposals

Vote table covering all 98 peer proposals (110 total − A8's own 12). Rules: `blocking=true` only if objection rests on a hard rule from `04-agent-rules.md`. `override_request` is A1-veto override; N/A for non-A1 vetoes here (none observed).

### 6.1 Votes on A1 (14)

| proposal_id | vote | rationale (cite rule id) | blocking | override_request |
|-------------|------|--------------------------|----------|-------------------|
| A1-P1 | agree | X2; pre-sizing `producer_index` removes admission-path allocation — also benefits A8 because deterministic sizing lets tests predict when `ResourceExhausted` fires | false | false |
| A1-P2 | agree | X3; bounding DATA-mode lock hold time makes `FakeClock`-driven watchdog tests (A8-P1) deterministic | false | false |
| A1-P3 | agree | P2; Function Cache LRU + HEARTBEAT presence gives A8 per-peer cache observability (drop counter + cache-miss metric) | false | false |
| A1-P4 | agree | X4; enumerating Task hot/cold fields makes `dump_state()` (A8-P4) safe to snapshot without tearing a cache line | false | false |
| A1-P5 | agree | X9; explicit SPMD / event-loop-stage budgets are directly measurable via A8-P12 `PhaseId`s | false | false |
| A1-P6 | agree | P6; staging binaries and descriptor-template dedup reduces REMOTE_SUBMIT bytes — dedup counters feed O3 | false | false |
| A1-P7 | agree | X3; per-thread local sequence removes cross-socket ping-pong that currently breaks the `< 100 ns` per-Phase target — **structurally required** for A8-P12 at L2 | false | false |
| A1-P8 | agree | X2; same rationale as A1-P1; `WouldBlock` surface becomes a first-class observable in `LayerStats` | false | false |
| A1-P9 | agree | X4; `select_workers` under 10 ns accelerates Chip→Core phase measurement on A8-P5 budget | false | false |
| A1-P10 | agree | P1; **A8 is co-owner per synthesis**; enforces NFR-1 ≤ 1% at L1 and ≤ 5% at L2 | false | false |
| A1-P11 | agree | X9; per-arg marshaling budget lets A8-P12 emit a `ARG_MARSHAL` phase with measurable per-arg timing | false | false |
| A1-P12 | agree | X9, P5; exposing `max_batch` as a tunable makes tail latency a first-class SLO (A8-P5 alert rule target) | false | false |
| A1-P13 | agree | P6, X9; bounding `args_blob` copy fits the 5 μs stage; observable via A8-P12 `ARGS_COPY` PhaseId | false | false |
| A1-P14 | agree | X4; documenting placement + shard default closes the X4 evidence gap | false | false |

### 6.2 Votes on A2 (9)

| proposal_id | vote | rationale (cite rule id) | blocking | override_request |
|-------------|------|--------------------------|----------|-------------------|
| A2-P1 | agree | E6; version fields support REPLAY (A6-P12) and distributed trace correlation (A8-P6); validation stays at handshake — zero hot-path cost | false | false |
| A2-P2 | agree | E4; schema-registered `LevelOverrides` lets per-level profiling and tracing knobs (A8-P5 alert-rule overrides, A8-P3 histogram bucket counts) extend without editing `runtime/` | false | false |
| A2-P3 | agree | E4; opening init-time enums (FailurePolicy/TransportBackend/SimulationMode/NodeRole) while keeping `DepMode` closed aligns with A8's need for pluggable test backends without touching hot path | false | false |
| A2-P4 | agree | E5; migration plan is orthogonal to A8 but improves testability of the transition by naming feature flags that gate the rollout | false | false |
| A2-P5 | agree | E2; BC policy table lets A8-P11 HAL contract-test suite enumerate stable vs. evolving interfaces | false | false |
| A2-P6 | agree | E4; registry-dispatched `IDistributedProtocolHandler` makes it possible to plug an A8-P7 fault-injection handler (drop/reorder/duplicate) without editing transport switches | false | false |
| A2-P7 | abstain | G2/E1 tension; reserving an unused async-policy sub-interface is orthogonal to A8's rubrics. Abstain: no direct testability impact either way. | false | false |
| A2-P8 | agree | G3; recording intentional closures as known deviations improves design-level traceability, which helps A8-P11 delineate what the HAL contract-test suite must cover | false | false |
| A2-P9 | agree | E6; versioned trace schema is a **prerequisite** for A6-P12 (signed REPLAY trace) and A8-P6 (distributed trace alignment) | false | false |

### 6.3 Votes on A3 (15)

| proposal_id | vote | rationale (cite rule id) | blocking | override_request |
|-------------|------|--------------------------|----------|-------------------|
| A3-P1 | agree | LSP, D7, V5; `ERROR` state makes `TraceEvent.task_state` transitions complete — the normative state machine must match the narrative. A8 rubric #2 (trace IDs on every critical op) depends on a total state taxonomy | false | false |
| A3-P2 | agree | LSP; deterministic `submit()` return contract is directly measurable (A8-P12 `SUBMISSION_ADMIT` PhaseId carries `AdmissionStatus`) | false | false |
| A3-P3 | agree | V4; admission-path failure scenarios close a rubric-4 V4 gap that would otherwise leave A8-P7 fault-injection matrix incomplete | false | false |
| A3-P4 | agree | O1, DS3; `DEP_FAILED` event replaces the "eventually timeout" hand-wave and becomes a traceable event with existing `correlation_id`/`task_key` — no A8 data model change needed | false | false |
| A3-P5 | agree | DS4, G3; sibling cancellation policy produces deterministic `ErrorContext.remote_chain` fan-in — observability surface (A8-P10 KV logging) already threads `correlation_id` | false | false |
| A3-P6 | agree | G1, V3; requirement ↔ scenario traceability matrix is the missing table that A8-P11 HAL contract-test suite cross-references | false | false |
| A3-P7 | agree | G3, S3; submission precondition validation codifies inputs that A8-P7 fault-injection and A8-P3 admission-error histograms can count distinct codes for | false | false |
| A3-P8 | agree | G3, X9; cyclic `intra_edges` DFS with a bounded cost is measurable via a new `DATA_DEP_SCAN` PhaseId (already named in A8-P12) | false | false |
| A3-P9 | agree | G3; SPMD index delivery contract is testable — A8-P11 HAL contract tests can assert ABI conformance | false | false |
| A3-P10 | agree | G1, D7; complete Python exception mapping is directly testable by `python_mapping_completeness` (`modules/error.md:381`) which A8 already tracks | false | false |
| A3-P11 | agree | G3; `[ASSUMPTION]` marker at `complete_in_future` matches A8's observability discipline — unresolved questions must be explicit at the point of use | false | false |
| A3-P12 | agree | G3, R4; `drain()`/`submit()` concurrency contract produces a deterministic state transition observable via A8-P4 `dump_state()` | false | false |
| A3-P13 | agree | DS3, G3; **prerequisite for A8-P6** — declared per-chain FIFO transport ordering is the premise of distributed-trace causal-order guarantee | false | false |
| A3-P14 | agree | LSP, V5; documented `COMPLETING` skip at leaf Workers closes a state-machine inconsistency that would otherwise confuse A8-P12 PhaseId boundaries | false | false |
| A3-P15 | agree | G3, X6; debug-mode `NONE`-dep-mode cross-check is a textbook X6 "production debuggability" hook — exactly the kind of low-cost testability lever A8 endorses | false | false |

### 6.4 Votes on A4 (9)

| proposal_id | vote | rationale (cite rule id) | blocking | override_request |
|-------------|------|--------------------------|----------|-------------------|
| A4-P1 | agree | D7, V5; canonical enum casing avoids mismatched trace-string comparisons across docs | false | false |
| A4-P2 | agree | D7, V5; broken anchor in task-model cross-reference is a pure correctness fix | false | false |
| A4-P3 | agree | D7, G4; "Two" → "Three" invariants is a pure accuracy fix | false | false |
| A4-P4 | agree | D7, V5; numeric ordering of open questions is a pure housekeeping fix | false | false |
| A4-P5 | agree | D7; glossary entries for `EventLoopDeploymentConfig`, `SourceCollectionConfig`, `IEventCollectionPolicy` are needed by A8-P2's test-only seam (see revision). If A9-P2/A9-P7 land, `SourceCollectionConfig` may fold; A4-P5 should drop that entry in that case | false | false |
| A4-P6 | agree | V2, G5; ADR back-references from top-level views make A8-P12 PhaseId design (ADR-010-aligned) discoverable | false | false |
| A4-P7 | agree | V5, D7; one unambiguous label for the L0 `ISchedulerLayer` helps A8-P11 contract-test suite targeting | false | false |
| A4-P8 | agree | D7; Appendix-B state-count sync with `modules/core.md` fixes drift against A3-P1's new `ERROR` state | false | false |
| A4-P9 | agree | D7; expanded `Task State` glossary entry enumerates states used by A8-P12 PhaseIds | false | false |

### 6.5 Votes on A5 (10)

| proposal_id | vote | rationale (cite rule id) | blocking | override_request |
|-------------|------|--------------------------|----------|-------------------|
| A5-P1 | agree | R2; exponential backoff + jitter is also A8-observable — retry counter histograms fit A8-P3 | false | false |
| A5-P2 | agree | R3; per-peer circuit breaker introduces a peer-state metric A8-P3 can surface directly | false | false |
| A5-P3 | agree | R5; committing to failover or deterministic fail-fast closes a design-level ambiguity that A8-P6 trace merge depends on. Merge-register pair with A10-P2 accepted. | false | false |
| A5-P4 | agree | DS4; `idempotent: bool` on `TaskDescriptor` gates retry decision deterministically and becomes an observable attribute in trace events | false | false |
| A5-P5 | agree | R6; **jointly owned with A8-P7** per synthesis. Chaos matrix is the *what*; A8-P7 `IFaultInjector` is the *how* | false | false |
| A5-P6 | agree | R5, O5; **paired with A8-P4** per Merge Register. Watchdog detects the silent hang; `dump_state()` supplies evidence. Accept pairing. | false | false |
| A5-P7 | agree | R1; `Timeout` parameter on `IMemoryOps` async closes a rubric-1 A5 gap and uses `IClock` (A8-P1) under the hood | false | false |
| A5-P8 | agree | R4; degradation specs for admission saturation + partial WorkerGroup loss become A8-observable state transitions | false | false |
| A5-P9 | agree | R3; QUARANTINED Worker state fits A8-P12 PhaseId taxonomy and `dump_state()` worker-state matrix | false | false |
| A5-P10 | agree | DS4; per-handler idempotency contract is a CI-checkable assertion — aligns with A8-P11's contract-test discipline | false | false |

### 6.6 Votes on A6 (14)

| proposal_id | vote | rationale (cite rule id) | blocking | override_request |
|-------------|------|--------------------------|----------|-------------------|
| A6-P1 | agree | S1; complete STRIDE table — observability for trust-boundary crossings requires enumerated boundaries first | false | false |
| A6-P2 | agree | S1, S2; node authentication in HANDSHAKE produces auditable handshake outcomes (fits A6-P6 audit sink on top of A8 sink plumbing) | false | false |
| A6-P3 | agree | S3; bounded variable-length payload parsing adds a `corrupted_frames` metric — A8-P3/A8-P9 alert targets | false | false |
| A6-P4 | agree | S4, S5; fail-closed TLS default is a startup check, not a hot-path cost | false | false |
| A6-P5 | agree | S2, S3; rkey lifecycle is observable as a `rkey_inflight` counter feeding A8-P3 | false | false |
| A6-P6 | agree | S6; `IAuditSink` as a sibling of `ILogSink` uses the exact sink plug-in surface A8 already specifies — no new subsystem | false | false |
| A6-P7 | agree | S1, S3; attestation failures produce a new audit/error class observable through A8 trace events | false | false |
| A6-P8 | agree | S3; DLPack boundary validation is a Python→C stage check that A8-P12 `ARG_MARSHAL` phase will measure | false | false |
| A6-P9 | agree | S1, S2, S3; `logical_system_id` on `MessageHeader` provides the capability key that A6-P10 and A8-P10 amended KV sinks filter on | false | false |
| A6-P10 | agree | S2; **aligns with A8-P10 amendment**: capability-scoped KV sinks enforce tenant isolation at the sink filter rather than at call sites | false | false |
| A6-P11 | agree | S2; gated `register_factory` is a pure init-time guard; no hot-path cost | false | false |
| A6-P12 | agree | S1, S3; signed REPLAY trace format complements A2-P9 versioned trace schema and A8-P6 trace alignment | false | false |
| A6-P13 | abstain | S1 (DoS); per-tenant rate-limit is primarily a security control. A8 has no direct testability stake beyond the observable rejection counter; abstain. | false | false |
| A6-P14 | agree | G5, S5; key-material lifecycle ADR is pure documentation; improves audit-trail coverage | false | false |

### 6.7 Votes on A7 (9)

| proposal_id | vote | rationale (cite rule id) | blocking | override_request |
|-------------|------|--------------------------|----------|-------------------|
| A7-P1 | agree | D6; breaking the `scheduler/` ↔ `distributed/` cycle is a hard-rule fix. Testability win: `scheduler/` unit tests can mock `distributed/` via events | false | false |
| A7-P2 | agree | ISP, D4; splitting `ISchedulerLayer` narrows test-double surface — a `bindings/` test needs only `ISchedulerSubmit`+`ISchedulerLifecycle` | false | false |
| A7-P3 | agree | D2; moving `DeviceAddress`/`NodeId` to `core/types.h` lets A8-P11 HAL contract tests and A8-P2 `RecordedEventSource` compile without pulling in any `hal/` target | false | false |
| A7-P4 | agree | D3, D5; moving distributed payloads to `distributed/` narrows transport to framing — A8-P7 fault injector on `IHorizontalChannel` becomes backend-agnostic | false | false |
| A7-P5 | agree | D2; `distributed_scheduler` depending only on `ISchedulerLayer` makes a mock `ISchedulerLayer` sufficient to test `distributed/` in isolation | false | false |
| A7-P6 | agree | D3, D5; extracting `composition/` (registry + deployment parser) isolates the code A8-P5 externalized-alerts parsing extends | false | false |
| A7-P7 | agree | D2, D6; documented forward-decl contract removes hidden `core → memory/transport` edges that would otherwise sneak into test builds | false | false |
| A7-P8 | agree | D7; consolidating `ScopeHandle` ownership is glossary hygiene A8-P11 contract tests benefit from | false | false |
| A7-P9 | agree | D7; deduplicating `MemoryError` class names is a pure Python-surface fix | false | false |

### 6.8 Votes on A9 (8)

| proposal_id | vote | rationale (cite rule id) | blocking | override_request |
|-------------|------|--------------------------|----------|-------------------|
| A9-P1 | agree | G2; dropping `submit_group`/`submit_spmd` overloads leaves a single admission entry — narrower test surface for A8-P2 `RecordedEventSource` | false | false |
| A9-P2 | **disagree** | X5, §8.2 DfT; deleting *all* event-loop deployment plurality is acceptable only if the single `IEventLoopDriver` test seam survives (synthesis bridge). Without that seam, A8-P2 becomes unimplementable. Not blocking — A9's proposal conditioned on the bridge is acceptable; blanket deletion is not. Requesting amendment to keep the single test-only driver seam | false | false |
| A9-P3 | abstain | G2; removing collectives from `IHorizontalChannel` is orthogonal to A8 (affects A2/A1 more) | false | false |
| A9-P4 | agree | G2; dropping `SubmissionDescriptor::Kind` simplifies A8-P12 PhaseId attribution (one code path) | false | false |
| A9-P5 | agree | G2, DRY; unifying the two admission enums gives A8-P3 histograms a single status dimension | false | false |
| A9-P6 | **disagree** | X5, §8.2 DfT ("Reproducible failures"). Deferring PERFORMANCE/REPLAY cancels A8's REPLAY-based reproducible-failure tactic (`modules/hal.md:117, 346-349`, `07-cross-cutting-concerns.md:131`), invalidates A6-P12 (signed REPLAY trace) and weakens A2-P9 (versioned trace schema — REPLAY is its primary consumer). Not blocking under G2 rule; A9's justification is YAGNI, not a hard rule — but A8 strongly objects on X5 grounds. Recommend keeping REPLAY in v1 (FUNCTIONAL + REPLAY) and deferring only PERFORMANCE (which has no A8 dependency). | false | false |
| A9-P7 | abstain | G2; folding `SourceCollectionConfig` into `EventHandlingConfig` is a config-struct refactor. A8-P2 amendment references the merged struct name trivially. Abstain. | false | false |
| A9-P8 | agree | G2, D1; moving companion-artifacts obligation out of runtime design reduces cross-domain coupling; A8 doesn't depend on artifact format being in-scope | false | false |

### 6.9 Votes on A10 (10)

| proposal_id | vote | rationale (cite rule id) | blocking | override_request |
|-------------|------|--------------------------|----------|-------------------|
| A10-P1 | agree | P4, X3; default `producer_index` sharding avoids the single-lock contention peak — observable via A8-P3 per-shard histograms | false | false |
| A10-P2 | agree | P4, R5; decentralizing Pod coordinator. **A8-P6 amendment adds any-owner-deterministic trace merge** to stay forward-compatible with this | false | false |
| A10-P3 | agree | DS6; per-data-element consistency table is a direct O3/O4 documentation win — readers can audit state without rediscovering from six files | false | false |
| A10-P4 | agree | DS1; stateful/stateless classification aligns with A8-P4 `dump_state()` surface design | false | false |
| A10-P5 | agree | P6; merged with A1-P6 per register; A8 agrees — per-peer REMOTE_SUBMIT projection is observable via `bytes_per_peer` metric | false | false |
| A10-P6 | agree | R5, O5; faster peer-failure detection makes `NodeLost` observable within SLO — closes a current A8 rubric-4 gap | false | false |
| A10-P7 | agree | P4, DS6; sharded TaskManager path preserves earliest-first completion bias and opens admission-throughput observability | false | false |
| A10-P8 | agree | DS7, P6; aggregated data-flow diagram is pure O4 win | false | false |
| A10-P9 | agree | DS6; gating `WorkStealing` against `RETRY_ELSEWHERE` plus assignment log is auditable by A8-P11 contract tests | false | false |
| A10-P10 | agree | X4; producer_index cap + cache-line layout guidance is foundational for A8-P3 histogram cache-friendliness | false | false |

### Vote tally

- Total votes cast: **98** (14+9+15+9+10+14+9+8+10).
- Agree: **92**.
- Disagree: **2** (A9-P2, A9-P6; both non-blocking; both request amendments rather than outright rejection).
- Abstain: **4** (A2-P7, A6-P13, A9-P3, A9-P7).
- `blocking=true`: **0** (no hard-rule violations stand after the amendments above).
- `override_request=true`: **0** (no A1 veto applied to any A8-relevant proposal).

## 7. Cross-aspect tensions (new in round 2)

| pair | with_proposal | preferred_resolution |
|------|---------------|----------------------|
| A8 vs A9 | A8-P2 vs A9-P2 | Keep a single `IEventLoopDriver` test-only seam (`enable_test_driver` compile flag). A9 retains closed enums for deployment modes; A8 keeps the driveability seam. No virtual dispatch in release; both rubrics satisfied. |
| A8 vs A9 | A8-P5 vs A9 (YAGNI) | Amend A8-P5: alert rules live in an external file referenced by `DeploymentConfig.alerting_rules_path` (4-field schema); Prometheus/OTEL sink is recorded in `10-known-deviations.md` as an opt-in deferred item, not shipped by default. Format choice only. |
| A8 vs A9 | A8-P6 / A8-P8 / A8-P10 / A8-P12 vs A9-P6 | A9-P6 defers PERFORMANCE/REPLAY entirely. A8 strongly prefers a split: defer **PERFORMANCE** (no A8 dependency); **keep REPLAY** in v1 to preserve §8.2 DfT "Reproducible failures". A6-P12 and A2-P9 align. |
| A8 vs A2 | A8-P6 with A2-P9 | Versioned trace schema (A2-P9) is the header surface A8-P6 merges against. Fuse: `trace_schema_version` + `capture_node_id` + clock-sync offset in a single header. |
| A8 vs A6 | A8-P10 with A6-P10 | Capability-scoped sinks + structured KV = tenant-aware observability without call-site changes. Every KV event carries `logical_system_id`; sink filter is a branch. |
| A8 vs A5 | A8-P7 with A5-P5 | Joint co-ownership: A5-P5 enumerates the scenario matrix (what); A8-P7 defines the `IFaultInjector` seam (how). Sim-only compile target; zero release-build cost. |
| A8 vs A5 | A8-P4 with A5-P6 | Paired per Merge Register: A5-P6 detects the silent-hang; A8-P4 provides the evidence surface. One incident-response workflow, two proposals, fast-track together. |
| A8 vs A10 | A8-P6 with A10-P2 | Decentralized coordination needs a trace-merge owner. A8-P6 amendment declares merge owner = coordinator when centralized (v1) / any-owner-deterministic when A10-P2 lands. |
| A8 vs A7 | A8-P1 / A8-P2 / A8-P11 with A7-P3 / A7-P7 | `core/types.h` relocation (A7-P3) + forward-decl contract (A7-P7) make A8's test doubles buildable without `hal/`. No conflict; reinforcing. |
| A8 vs A1 | A8-P3 / A8-P8 / A8-P12 (hot-path-touching) | Two-tier paths already documented in round 1: branchless histogram increment (P3), Level-2 opt-in strip (P8), PhaseId compile-strip at L1 (P12). A1's veto authority retained over exact instruction sequences. |

## 8. Stress-attack on emerging consensus

*Not applicable — round 3 only.*

## 9. Status

- **Satisfied with current design?** **partially** (unchanged from round 1, but improved).
  - A8's 12 proposals converged without blocking objections after the round-2 amendments.
  - Key convergence wins: A8-P1 (IClock) trivially compatible with A7-P3/P7; A8-P7 jointly owned with A5-P5; A8-P4 paired with A5-P6; A8-P12 prerequisite (A1-P7) already in A1's set; A8-P10 aligned with A6-P10; A8-P6 forward-compatible with A10-P2 via amendment.
  - Outstanding concerns: A9-P2 and A9-P6 require amendment before A8 can accept (both non-blocking in the convergence rule). A9-P2 must preserve a single `IEventLoopDriver` test seam (synthesis bridge); A9-P6 should ship FUNCTIONAL + REPLAY in v1 and defer only PERFORMANCE.
- **Open items expected in round 3 (stress-attack targets):**
  - A8-P1 (IClock) vs A1's expected stress on any residual indirection under L2 profiling.
  - A8-P3 (stats + histograms) vs A1's stress on histogram cache-line footprint for N = 64-bucket arrays.
  - A8-P4 / A5-P6 paired bundle vs A9 stress on "two proposals for one concern" (defensible: detect vs diagnose are distinct roles).
  - A8-P5 vs A9's stress on external-file format discipline (defensible: single 4-field schema).
  - A8-P6 vs A9-P6 stress on REPLAY scope (A8 defends split: keep REPLAY, defer PERFORMANCE).
  - A9-P2 amendment tracking: confirm the single `IEventLoopDriver` test seam is preserved in round 3.

## Appendix — Merge Register response

| Merge target | Absorbed | A8 position |
|--------------|----------|-------------|
| A1-P6 | A10-P5 | **Accept.** Not A8-owned; no objection. |
| A1-P14 | A10-P10 | **Accept.** Not A8-owned; no objection. |
| A5-P3 | A10-P2 | **Accept.** A8-P6 amendment explicitly tracks this merge — trace-merge owner definition adapts to decentralized coordination. |
| A5-P6 | A8-P4 | **Accept as pairing, not absorption.** A5-P6 (detect) and A8-P4 (diagnose) are distinct but same-workflow. Keep both IDs; route as a fast-track bundle. If the synthesis requires formal absorption, A8 prefers absorbing A5-P6 *into* A8-P4 to keep the `dump_state()` ID intact (A8-P4's JSON snapshot is the artifact the watchdog references); either ordering is acceptable. |
| A7-P2 | A9-P1 | **Accept.** Not A8-owned; no objection (narrower `ISchedulerLayer` helps A8 test doubles). |
| A6-P8 | A1-P11 (partial) | **Accept.** Not A8-owned; no objection. |

WROTE /data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/reviews/2026-04-18-171357/round-2/reviews/A8-testability.md
