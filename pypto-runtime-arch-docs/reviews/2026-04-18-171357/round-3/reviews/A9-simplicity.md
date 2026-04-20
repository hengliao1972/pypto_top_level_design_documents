# Aspect A9: Simplicity (KISS / YAGNI) — Round 3 (Stress)

## Metadata

- **Reviewer:** A9
- **Round:** 3 (mandatory stress round)
- **Target:** `/data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/`
- **Runtime Mode:** yes
- **Timestamp (UTC):** 2026-04-19

## 1. Rubric Checks (post-R2-amendments)

Re-run against the **amended** proposal set from round-2 §5 (not the R1 wording), restricted to what actually lands if all 107 agreed proposals are accepted.

| # | Check | R1 | R2 | R3 (post-amend) | Evidence | Rule(s) |
|---|-------|----|----|-----------------|----------|---------|
| 1 | Every abstraction earns its keep | Weak | Weak | **Weak → Pass-with-caveats** | Of the net new interfaces introduced by the agreed set — `IClock` (A8-P1), `IEventLoopDriver` (A8-P2), `IFaultInjector` (A8-P7), `IDistributedProtocolHandler` (A2-P6), the three role interfaces from `ISchedulerLayer` split (A7-P2) — **three are sim/test-only seams**, **one is role-split (subtractive)**, and **one (A2-P6) still carries a single v1 impl** but is scoped-internal-to-`distributed_scheduler/` with CRTP/`final` devirt (per synth amendment). Residual concern: A2-P6. | G2, G4 |
| 2 | Speculative future-proofing absent | Fail | Fail | **Fail → Mixed** | Post-R2 amendments defer `PERFORMANCE`+`REPLAY` (A9-P6 option iii), gate `trust_boundary.multi_tenant` defaults off (A6-P7/P13), cut collectives from `IHorizontalChannel` (A9-P3), close `LevelOverrides` for v1 (A2-P2), remove `SubmissionDescriptor::Kind` (A9-P4). Residual `Fail` items: A2-P1 version-every-contract still blanket-applied (see stress §8 row 1); A5-P9 `QUARANTINED` still unjustified (see §8 row 5); A6-P12 contingent on A9-P6 option. | G2 (YAGNI) |
| 3 | Clarity over cleverness | Weak | Weak | **Weak → Pass** | The two-tier bridge pattern (fast default / slow opt-in) that A1/A5/A6/A10 adopted per synth is clarity-forward: each hot path has one documented default, and the opt-in surface is behind a `LevelParam`. `06-scenario-view.md:15-30` (6.1.1 steps 1-11) still reads as a plain A→B→C dispatch flow; no policy engine sits on top. Event loop retains the single-threaded default and a closed `ExecutionMode` enum (A9-P2 amended). | G4 |
| 4 | No single-implementation wrappers | Weak | Weak | **Weak (unchanged)** | A2-P6 `IDistributedProtocolHandler` and the **appendix of v2 extensions** attached to A9-P2's amendment both nominate interfaces with no current second implementer. Amendment mitigates (devirt + scoped internal) but does not eliminate. | G2 |
| 5 | DRY on knowledge | Weak | Weak | **Weak → Pass** | A9-P5 (unified `AdmissionStatus`), A9-P7 (fold `SourceCollectionConfig` into `EventHandlingConfig`), A9-P1⊕A7-P2 (collapse submit overloads into one virtual + free builders), A1-P11⊇A6-P8a (single per-arg budget doubles as validation) all land. Residual: A5-P9 `QUARANTINED` still encodes as a state what can live as a timestamp field on `UNAVAILABLE`. | G2, DRY |

**Net:** the amended proposal set is net-simpler than today's design on three of five checks (3/4/5 improve), neutral on 1, and partially unresolved on 2. The three unresolved items (A2-P1 blanket versioning, A5-P9 `QUARANTINED`, A2-P6 single-impl abstract class) are all **non-blocking from A9** but carry concrete amendments below.

## 2. Pros (new this round)

- **Stress-round amendments held the line on the hot path.** Every proposal with `hot_path_impact ∈ {allocates, blocks, relayouts, extends-latency}` in the round-2 catalog carries a two-tier bridge in its §5 amendment (fast-path default preserved; slow-path gated by `LevelParam`). Scenario 6.1.1 steps 1-11 execute without touching any slow-path branch under default config. This is exactly the A1-vs-A9 tension resolution the aspect-pair table anticipates. Cites G2, G4.
- **Role-split `ISchedulerLayer` + single `submit(SubmissionDescriptor)` combo (A7-P2 ⊕ A9-P1)** is a genuine net reduction: vtable shrinks from 4+ slots to 3 narrow role interfaces × 1 entry point each, and the three role interfaces are addressable separately by callers (ISP). Rare case where a modularity proposal *reduces* surface. Cites G2, ISP, D3.
- **A9-P5 (unified `AdmissionStatus`) + A9-P4 (drop `Kind`) + A9-P7 (fold config) + A9-P1 (drop overloads) all converged to `agreed`** with no dissenter remaining — the five low-controversy KISS items converge cleanly. Cites G2, DRY.
- **Synth's "option (iii)" landing zone for A9-P6** (FUNCTIONAL-only impl + keep `SimulationMode` enum open + REPLAY scaffolding) is a KISS-compatible compromise: no second sim pipeline ships in v1 (A9 preserved), and enum stays open so A2/A6/A8 forward paths are preserved with zero v1 implementation cost. Cites G2, E5.

## 3. Cons (new this round)

- **The synth amendment for A2-P6 ("abstract-only-in-`distributed_scheduler` + single concrete backend with CRTP/`final` devirt") is a rename, not a retraction.** Every implementer of `IDistributedProtocolHandler` in v1 has to learn the abstract contract *and* the concrete backend *and* the devirt mechanism. The honest v1 framing is "one concrete `TcpDistributedProtocolHandler` class; add an abstract base the day a second backend exists." Cites G2 rubric check 4. Amendment proposed below in §8 row A2-P6.
- **A2-P1 as amended still reads as "every public data contract carries a version byte now."** The evolution policy in A2-P5 already captures the discipline; versioning should be **pay-as-you-evolve** (first forward-incompatible change adds the byte) rather than prophylactic. The synth folded some of A2-P1 into A2-P5 but the proposal's §5 revision kept the blanket versioning obligation. Still a YAGNI concern (G2). Amendment proposed in §8 row A2-P1.
- **A5-P9 (`QUARANTINED` Worker state) made it through R2 with A9 as the sole dissenter.** Five reviewers abstained because it does not affect their rubrics; the state nonetheless adds an FSM node that `UNAVAILABLE + timestamp_last_failure` already expresses. Round-3 stress raises this to "breaks under DRY" (rubric 5): two representations of one fact. Amendment proposed in §8 row A5-P9.
- **A6-P12 ("signed / schema-validated REPLAY trace") is agreed *contingent* on A9-P6 option choice.** Under synth-recommended option (iii) (my preference), A6-P12 becomes "format frozen at ADR-011-R2 time; no v1 implementation". If the parent locks in option (iii), A6-P12 must be explicitly re-scoped to "schema definition only" or it re-injects the trace-validator pipeline that A9-P6 sought to defer. Cites G2.

## 4. Proposals

No new A9 proposals this round. Three concrete amendments to peer proposals under §8 stress attacks (A2-P1, A2-P6, A5-P9). My own proposals' final-state revisions are in §5.

## 5. Revisions of own proposals (final state)

| id | action | amended_summary | reason |
|----|--------|-----------------|--------|
| A9-P1 | **defend (final)** | (R2 amendment stands: combined with A7-P2 into role-split `ISchedulerLayer` with exactly one virtual `submit(SubmissionDescriptor)` on `ISubmissionSink`.) | Unanimous agreed in R2; no dissenters to address in stress round. Merger clean. Cites G2, ISP, D3. |
| A9-P2 | **defend (final)** | (R2 amendment stands: cut `FULLY_SPLIT`/`SPLIT_DEFERRED`; collapse `IEventCollectionPolicy`/`IExecutionPolicy` to closed enums + config; **retain single `IEventLoopDriver` test seam** satisfying A8-P2 (`step()` + `RecordedEventSource` test double); appendix in `08-design-decisions.md` lists future extension interfaces as "available when a second implementation exists".) **Stress-round addition:** appendix entries must carry explicit **trigger conditions** (e.g., "reintroduce `IEventCollectionPolicy` when a workload demonstrates >1 collection policy would win; benchmark criteria: ..."). | Stress-round replay of 6.1.1 step 1-11 with `IEventLoopDriver` single seam: the production impl is the existing A→B→C loop; the test double drives it via `step()`. A8's X5 DfT rubric is satisfied. A2's OCP concern is partially addressed by the appendix; the **trigger condition** addition closes the last gap by making "when to add the interface" a documented, not judgmental, call. Cites G2, X5, E4. |
| A9-P3 | **defend (final)** | (R2 amendment stands: remove `barrier`/`all_reduce` from `IHorizontalChannel`; commit Q4 to option A (composition); ADR committing "hardware-native collective path re-enters as separate `ICollectiveOps` extension interface when HCCL perf data demands it.") | Unanimous agreed in R2. Stress replay of 6.1.2 step 4-7 confirms cross-node data movement uses `send`/`recv`/`poll`/`transfer` only; collectives are expressible as Orchestration Functions. Cites G2, E4. |
| A9-P4 | **defend (final)** | (unchanged: drop `SubmissionDescriptor::Kind`.) | Unanimous agreed in R2. Cites G2, DRY. |
| A9-P5 | **defend (final)** | (unchanged: unify admission enums on `AdmissionStatus`.) | Unanimous agreed in R2. Cites G2, DRY. |
| A9-P6 | **amend → accept option (iii)** | **Final form:** ship v1 with `FUNCTIONAL` simulation only; **keep `SimulationMode` enum open** with `PERFORMANCE` and `REPLAY` declared but unimplemented; preserve the minimal REPLAY scaffolding (enum value + `ITraceReader`/`ITraceWriter` placeholder interfaces declared but not registered in `MachineLevelDescriptor` factory slots for v1); ADR-011-R2 records named triggers (`PERFORMANCE` returns when a concrete perf-modeling consumer is named; `REPLAY` returns when a post-mortem debugging consumer is named); A6-P12 becomes "schema frozen at ADR-011-R2 time; no v1 implementation." | Synthesizer-recommended landing zone; also my preferred compromise. A9's core concern (no second sim pipeline ships in v1, no per-Function timing-model/replay artifact pipeline in v1) is preserved; A2's E5 migration-plan concern is addressed by the named triggers; A6/A8's forensics/DfT forward path is preserved via the scaffolding. Cites G2, E5. See §6 for final vote. |
| A9-P7 | **defend (final)** | (unchanged: fold `SourceCollectionConfig` into `EventHandlingConfig`; coordinate with A4-P5 glossary update.) | Unanimous agreed in R2. Cites G2, D7. |
| A9-P8 | **defend (final)** | (unchanged: move "up to three companion artifacts" obligation out of the runtime design doc.) | Unanimous agreed in R2. Cites G2, SRP. |

## 6. Votes on peer proposals (final vote on disputed proposals only)

Only the three disputed proposals require a final-round vote per the prompt. Votes on the 104 other agreed proposals from R2 stand unchanged (see R2 §6 of this reviewer's file).

| proposal_id | vote (final) | rationale (cite rule id) | blocking | override_request |
|-------------|--------------|--------------------------|----------|-------------------|
| A2-P7 (async-policy extension seam) | **agree (flip from R2 disagree)** | Conditioned on A2 adopting synth's recommended reframe: **no interface in v1, Q-record in `09-open-questions.md`** naming the two future policy axes + trigger conditions. This matches A9's cross-aspect resolution from R2 tensions (convert to Q-record). If A2's final amendment adds any class/interface declaration to the design docs for v1, flip back to disagree. Cites G2, E4 (interface introduced when needed). | false | false |
| A9-P2 (collapse pluggables) | **agree (self-vote, final)** | Amendment as in §5 above (single `IEventLoopDriver` seam + closed `ExecutionMode` enum + appendix with trigger conditions). A8's X5 concern satisfied by the test seam; A2's OCP concern substantially addressed by appendix-with-triggers. A2's `override_request=true` from R2 can be withdrawn if the trigger-condition wording lands in the appendix. Cites G2, X5, E4. | false | false |
| A9-P6 (defer PERFORMANCE/REPLAY) | **agree to option (iii)** | Synth's recommended landing zone. FUNCTIONAL-only impl in v1; enum stays open; REPLAY scaffolding (placeholder interfaces declared; no factory registrations; no per-Function replay artifact pipeline). ADR-011-R2 records named triggers. A6-P12 is re-scoped to "schema frozen; no v1 implementation." This keeps v1 obligation-free while preserving day-2 forward paths for A2 (E5), A6 (S6 audit), A8 (X5 DfT). A2's `override_request=true` from R2 addressed by the named triggers; A6/A8 dissent partially addressed by scaffolding retention. Cites G2, E5. | false | false |

A9's primary rules (G2, G4) are design-discipline soft rules; A9 files no blocking objections and no override requests this round.

## 7. Cross-aspect tensions (updated)

| pair | with_proposal | resolution state |
|------|---------------|-------------------|
| A2 vs A9 | A2-P7 | **Resolved (conditional)** — synth reframe as Q-record; A9 flips to agree. If A2's final edit adds any interface declaration, tension reopens. |
| A2 vs A9 | A9-P2 | **Resolved (conditional)** — trigger-condition addition to the appendix closes the last OCP gap. Pending A2 override withdrawal. |
| A2 vs A9 | A9-P6 | **Resolved** — option (iii) preserves enum + scaffolding; A2's E5 migration plan preserved via named triggers. |
| A2 vs A9 | A2-P1 | **Outstanding** — blanket versioning still reads as YAGNI. Amendment in §8. |
| A2 vs A9 | A2-P6 | **Outstanding** — single-impl abstract class with CRTP devirt is a rename, not a retraction. Amendment in §8. |
| A5 vs A9 | A5-P9 | **Outstanding** — `QUARANTINED` state still expressible as timestamp field. Amendment in §8. |
| A6 vs A9 | A6-P12 | **Resolved via A9-P6 option (iii)** — re-scope to "schema frozen; no v1 implementation." |
| A8 vs A9 | A8-P2 | **Resolved** — single `IEventLoopDriver` seam approved by both sides. |

## 8. Stress-Attack on emerging consensus

Per the round-3 prompt: **every agreed proposal** with A9 standing is stress-tested against its amended text, with at least one scenario from `06-scenario-view.md` replayed against each. Priority was given to items A9 voted `disagree` in R2 that nevertheless crossed the agreed threshold, plus items where the two-tier bridge's end-to-end integrity matters.

Scenarios in primary rotation:
- **6.1.1** (single-node hierarchical kernel execution, `06-scenario-view.md:15-30`, 11 steps)
- **6.2.3** (task slot pool exhaustion, `06-scenario-view.md:116-128`, 5 steps)

Additional scenario touches where the proposal is distributed/failure-mode-specific:
- **6.1.2** (distributed multi-node, `:35-58`) — for distributed-path proposals
- **6.2.1** (AICore hang, `:83-98`) — for worker-state proposals
- **6.2.2** (remote node crash, `:100-114`) — for failure-policy proposals
- **6.2.4** (RDMA timeout, `:130-142`) — for timeout/remote-error proposals

### 8.1 Stress-attack table (agreed proposals with A9 standing)

| proposal_id | stress_attempt | scenario replay | verdict |
|-------------|----------------|-----------------|---------|
| **A1-P4** (hot/cold `Task` split) | A9 R2 concern was "Logical View prescribes layout before benchmark." Amendment moved the prescription to `modules/core.md §8`. Attack: does the Logical View still carry enough constraint to mislead an implementer? Replay **6.1.1 steps 2-6**: Task descriptor moves Host→Device (step 2), AICPU reads it (step 3), dispatches child tasks (step 3-4). Under amended A1-P4, Logical View says "Task fields are partitioned hot/cold per `modules/core.md §8`"; the module doc owns the layout. Scenario executes without touching layout details in the Logical View narrative. | `06-scenario-view.md:17-24` (steps 1-6) | **holds** |
| **A1-P9** (bitmask-encode slot availability) | A9 R2 concern: premature layout (P3) with speculative ≤64-group cap + no extension path. Amendment adds the 2×64 fallback. Attack: with 2×64 bitmask + cap documented up front, is the complexity justified by 6.1.1's single-node flow? Replay **6.1.1 step 2** (Task SUBMITTED→PENDING→DEP_READY→RESOURCE_READY→DISPATCHED): resource-readiness checks traverse the slot-type availability bitmask once per transition. For a 4-level (Host/Device/Chip/Core) single-node deploy, slot-type count ≪ 64; bitmask fits, no fallback triggered. | `06-scenario-view.md:19` (step 2) | **holds** |
| **A1-P12** (`BatchedExecutionPolicy.max_batch` + tail budget) | A9 R2 concern: tunable proliferation (G2). Amendment: fast-path default = Dedicated; `max_batch` applies only when `Batched` explicitly selected. Attack: does the fast-path default preserve 6.1.1's happy path? Replay **6.1.1 step 1-3**: `submit()` → Host Scheduler → Device Worker. Default `ExecutionMode=Dedicated` means no batching branch on hot path; `max_batch` parameter reachable only when `LevelParam.execution_mode=Batched` is set. Fast-path unchanged. | `06-scenario-view.md:17-21` (steps 1-3) | **holds** |
| **A1-P13** (bound `args_blob`; 1 KiB ring-slot fast path) | Attack: does the two-tier (fast ≤1 KiB / slow >1 KiB slow-path) introduce policy branching on the hot admission path? Replay **6.1.1 step 1** (Python→bindings→runtime): args_blob for typical tensor kernels fits in 1 KiB (handles + scalars). Fast path taken without branch visible in Logical View. Slow-path (≥1 KiB) reserved for large-arg SPMD launches which 6.1.3 depends on. | `06-scenario-view.md:17` (step 1); `:63-77` (6.1.3) | **holds** |
| **A2-P1** (version every public data contract) | Attack: blanket versioning adds a byte + serialization obligation to every wire/persistent contract for a v1 runtime whose only wire protocols are handshake, REMOTE_SUBMIT, REMOTE_DATA_READY, and HEARTBEAT (4 messages). The proposal's §5 revision still reads as "all public contracts carry a version field." Replay **6.1.2 steps 4-8** (REMOTE_SUBMIT, REMOTE_DATA_READY, REMOTE_COMPLETE over Horizontal Channel): each of these 4 wire messages grows by (at minimum) 1 version byte in v1 without a second version planned. A2-P5 (evolution policy) already captures the discipline. | `06-scenario-view.md:48-54` (steps 4-8) | **uncertain** — amendment below |
|  | **Proposed amendment:** narrow A2-P1 to "reserve one version byte on each **multi-node wire message** (REMOTE_SUBMIT, REMOTE_DATA_READY, REMOTE_COMPLETE, HEARTBEAT) and each **persistent artifact** (Function Cache entry, ADR-007). Do not blanket-require version bytes on in-process structs, trace records (covered by A2-P9), or Python-binding structs (never evolved without Python re-import)." This preserves the E6 discipline where wire/persistence actually exists and removes the YAGNI ceremony on in-process data contracts. A2's E6 concern fully preserved; A9's G2 concern resolved. | | |
| **A2-P2** (`LevelOverrides` closed-for-v1, schema-registered v2) | Attack: the "v2 schema registration" carrot keeps the enum closed in v1 (A9 satisfied) but imports a registration mechanism in v1 prose. Does this actually add v1 surface? Replay **6.1.1 step 1** (runtime init): `MachineLevelDescriptor` is constructed from config; v1 uses the closed enum only. Schema-registry is documented as v2 in ADR + Q-record, no v1 code. Surface = zero in v1. | `06-scenario-view.md:13-15` (preconditions) | **holds** |
| **A2-P3** (open extension-point enums; keep `DepMode` closed) | A9 R2 abstained pending "open" semantics. Attack: if "open" means the enums are declared extensible via TU-local registration, that's a registry; if "open" means documentation-only extension hooks, that's fine. Amendment clarifies: closed-in-C++-ABI, schema-extensible-in-docs. Replay **6.1.1**: closed C++ enums means the admission path in step 2 cannot receive unknown enum values at runtime; doc-extensibility is a v2+ concern. Net v1 surface = zero. | `06-scenario-view.md:19` (step 2) | **holds** |
| **A2-P4** (Migration & Transition Plan) | Attack: a migration plan document adds reading burden but no runtime surface. Already a doc-only obligation. Replay **6.1.1**: zero effect on runtime flow. | `06-scenario-view.md` (N/A — doc) | **holds** |
| **A2-P5** (Interface Evolution & BC Policy) | Attack: policy document. Replay: same as A2-P4. Preserves A9's preference for "discipline as policy, not as ceremony-per-contract." | `06-scenario-view.md` (N/A — doc) | **holds** |
| **A2-P6** (`IDistributedProtocolHandler` abstract boundary; one concrete v1, CRTP/`final` devirt) | Attack: A2-P6 is the textbook "one interface, one implementation" pattern (rubric check 4). Amendment ("abstract only in `distributed_scheduler`; single concrete backend with CRTP/`final` devirt") renames it as "internal abstraction" but the abstract class still exists, still has to be learned, still has to be maintained in ADR. Replay **6.1.2 step 4-5** (REMOTE_SUBMIT over Horizontal Channel): Node₀ constructs payload, invokes `IDistributedProtocolHandler::send_remote_submit()`; concrete `TcpDistributedProtocolHandler` handles it. Surface visible to v1 maintainers: 1 abstract class + 1 concrete class + 1 ADR on devirt pattern. Honest v1 framing: `TcpDistributedProtocolHandler` concrete class; add abstract base the day a second backend is on a roadmap. | `06-scenario-view.md:48-50` (steps 4-5) | **uncertain** — amendment below |
|  | **Proposed amendment:** demote the abstract class to a **marker**: a single `protocol_handler::send_remote_submit(...)` free function (or static member) in `distributed_scheduler/protocol.hpp` for v1. When a second backend becomes real, introduce `IDistributedProtocolHandler` via an ADR — the free-function signature becomes its single method, zero call-site churn. This preserves A2's "abstract boundary" intent (the seam exists; its name is `protocol.hpp`) with zero v1 abstract-class ceremony. If A2 finds the free-function route unacceptable, keep the abstract class but require the ADR to name a roadmap for the second backend (currently absent). | | |
| **A2-P7** (reserve async-policy extension seam) | Vote final in §6 above; disputed item. Scenario replay covered under §8.2. | see §8.2 | see §6 |
| **A2-P8** (record intentional closures as known deviations) | Attack: doc-only; no runtime effect. Replay: N/A. A9-aligned with the pattern (deferred-not-cancelled). | `06-scenario-view.md` (N/A — doc) | **holds** |
| **A2-P9** (versioned trace schema) | Attack: one byte per trace record + version field in trace header. Is the byte worth it for v1 with no second trace schema? Replay **6.1.1 step 11** (trace flush at task retirement, mentioned in postconditions `:31`): trace records carry 1 version byte; trace schema is the single evolvable artifact where version makes sense (traces outlive the runtime writing them). Accept as a non-blanket version use. | `06-scenario-view.md:31` (postconditions) | **holds** |
| **A3-P1** (add `ERROR` + optional `CANCELLED` Task states) | Attack: state explosion check. The R1 FSM had {SUBMITTED, PENDING, DEP_READY, RESOURCE_READY, DISPATCHED, EXECUTING, COMPLETING, COMPLETED, RETIRED}. Adding `ERROR` (blocker, correct) and optional `CANCELLED`(medium) brings the count to 10-11. Replay **6.2.1 step 5** (AICore hang): Parent Orchestration Task transitions COMPLETING→ERROR. Without `ERROR`, prior FSM quietly collapses this into COMPLETED-with-error-flag — a DRY-violating encoding. Correctness trumps count here. | `06-scenario-view.md:94` (step 5 of 6.2.1) | **holds** — correctness overrides G2 |
| **A3-P2** (`std::expected<SubmissionHandle, ErrorContext>`) | Attack: does `std::expected` add cognitive overhead on the common path? Replay **6.1.1 step 1** (submit returns): happy path = `expected.value()` once; error path = `expected.error()` with typed context. This is simpler than exception-vs-nullopt-vs-out-param ambiguity. | `06-scenario-view.md:17` (step 1) | **holds** |
| **A3-P8** (cyclic `intra_edges` detection) | A9 R2 concern: HPI=`extends(bounded)` on admission path. Amendment: debug-mode only; fast path (FE_VALIDATED) skips. Attack: does the fast path really skip for 6.1.1? Replay **6.1.1 step 1**: if FE_VALIDATED bit is set on the SubmissionDescriptor (front-end already validated the DAG), admission skips the detection. For SPMD in 6.1.3: FE_VALIDATED can be set on group-generated descriptors. For arbitrary user orchestration: debug-mode pays the cost; release-mode trusts FE_VALIDATED. | `06-scenario-view.md:17` (step 1); `:67-77` (6.1.3) | **holds** |
| **A5-P1** (exponential backoff + jitter) | Attack: does retry logic appear on hot path? Replay **6.1.1**: no retries on happy path. Replay **6.2.4 step 2** (RDMA timeout): retry logic lives in distributed-scheduler error-handling module, fires only on timeout; not on hot path. Textbook R2 application. | `06-scenario-view.md:136` (step 2 of 6.2.4) | **holds** |
| **A5-P2** (per-peer circuit breaker) | Attack: HPI=`none (thread-local TSC)` per amendment. Replay **6.1.2 step 4** (REMOTE_SUBMIT): pre-check reads thread-local TSC + circuit-breaker state (1 load, 1 compare); no lock, no atomic. Hot-path cost measurable and small. | `06-scenario-view.md:48` (step 4 of 6.1.2) | **holds** |
| **A5-P3** (v1 deterministic coordinator fail-fast) | A9 aligned with fail-fast v1 branch. Replay **6.2.2 step 3-6a** (remote node crash, ABORT_ALL policy): coordinator marks Node₁ FAILED, cancels in-flight tasks, raises `DistributedError`. Simpler than a quorum/decentralized coordinator. | `06-scenario-view.md:107-111` (steps 3-6a of 6.2.2) | **holds** |
| **A5-P4** (`idempotent: bool` on `TaskDescriptor`) | Attack: one bit, maximal correctness; hot path reads bit to gate retry-safe branch. Replay **6.1.1**: hot path reads `idempotent` only on retry, i.e., only if error occurs. No retry in 6.1.1 happy path → bit not read. | `06-scenario-view.md:17-31` (steps 1-11) | **holds** |
| **A5-P5** (chaos / fault-injection matrix) | Attack: sim-only; X5 seam. Replay: N/A for runtime; relevant under **6.2.x** fault matrix. | `06-scenario-view.md:81-142` (§6.2) | **holds** |
| **A5-P6 ↔ A8-P4** (scheduler watchdog + `dump_state()`) | Attack: watchdog adds a thread; `dump_state()` adds a diagnostic method. Does any of this leak onto hot path? Replay **6.1.1 step 2-10**: scheduler hot-path unchanged; watchdog runs on a separate thread with its own clock; `dump_state()` called only on operator demand. | `06-scenario-view.md:19-28` (steps 2-10) | **holds** |
| **A5-P7** (`Timeout` on `IMemoryOps` async) | Attack: one field on async-op struct. Replay **6.1.2 step 7** (RDMA write): timeout field drives poll-timeout on completion wait; without it, hangs are indistinguishable from slow paths. R1 required. Hot-path cost = 1 load + 1 compare (when polling). | `06-scenario-view.md:51` (step 7 of 6.1.2) | **holds** |
| **A5-P8** (degradation specs + alert rules) | Attack: documentation + alert rules. Replay: N/A for runtime; relevant on pool exhaustion, loss, etc. | `06-scenario-view.md:116-128` (6.2.3) | **holds** |
| **A5-P9** (`QUARANTINED` Worker state) | A9 R2 sole dissenter; attack retained in stress round. The R2 amendment did not respond to A9's core objection: `QUARANTINED` is a time-windowed sub-state of `UNAVAILABLE`. One state + one timestamp field (`unavailable_since`) expresses the same behavior. Replay **6.2.1 step 3-5** (AICore hang): Worker marked FAILED (permanent hw fault). No QUARANTINED state needed for 6.2.1. Where would QUARANTINED fire? Only on transient failures (e.g., remote node flapping). Replay **6.2.2 step 3** (remote crash): Node₁ marked FAILED. Again, permanent-until-explicit-unquarantine is a timestamp policy, not a state. | `06-scenario-view.md:90-94` (6.2.1); `:107-109` (6.2.2) | **breaks (DRY)** — amendment below |
|  | **Proposed amendment:** drop the `QUARANTINED` state; extend `UNAVAILABLE` with two fields: `unavailable_since: Timestamp` and `unavailable_policy: {Permanent, Quarantine(duration)}`. The two behaviors previously expressed by `FAILED` and `QUARANTINED` become policy variants of a single state. Observable behavior preserved; FSM shrinks by one node; DRY satisfied. A5 ownership respected. | | |
| **A5-P10** (DS4 per-REMOTE_* idempotency) | Attack: per-message idempotency keys add a wire field. Replay **6.1.2 step 4-8**: each REMOTE_* carries a `message_id`; receiver deduplicates. R1/R2 interlock. | `06-scenario-view.md:48-54` (steps 4-8 of 6.1.2) | **holds** |
| **A6-P1** (trust-boundary threat model) | Attack: doc-only. Replay: N/A for runtime. | `06-scenario-view.md` (N/A — doc) | **holds** |
| **A6-P2** (mTLS cert-pinned node auth) | Attack: HANDSHAKE gains auth. Replay **6.1.2 step 0** (precondition: 2 nodes connected): one-time handshake cost at startup. Not on hot path. | `06-scenario-view.md:39-41` (preconditions of 6.1.2) | **holds** |
| **A6-P3** (bounded payload parsing entry gate) | Attack: HPI=`extends (≤1 compare at entry; none on reject)`. Replay **6.1.2 step 5** (Node₁ deserialize REMOTE_SUBMIT): length check ≤1 compare; reject path short-circuits. Hot-path cost bounded. | `06-scenario-view.md:49` (step 5 of 6.1.2) | **holds** |
| **A6-P4** (default-encrypted TCP control; RDMA exempt) | Attack: TCP paths pay TLS cost; RDMA paths unchanged. Replay **6.1.2 step 7** (RDMA data move): unaffected. Replay **6.1.1**: single-node, no TCP, no TLS cost. Two-tier bridge intact. | `06-scenario-view.md:51` (step 7 of 6.1.2); `06-scenario-view.md:15-30` (6.1.1) | **holds** |
| **A6-P5** (scoped revocable RDMA `rkey`) | Attack: rotation on Submission retirement. Replay **6.1.2 step 10** (all children complete): rkey rotated as part of normal retirement flow; no extra step. | `06-scenario-view.md:54` (step 10 of 6.1.2) | **holds** |
| **A6-P6** (security audit trail) | Attack: structured logging integrates with A8-P10. Replay: audit records emitted on auth/key events; not on kernel dispatch hot path. | `06-scenario-view.md` (doc + non-hot-path) | **holds** |
| **A6-P7** (function-binary attestation, multi-tenant gated) | A9 R2 concern: YAGNI for single-tenant v1. Amendment: gated behind `trust_boundary.multi_tenant=true`; default off. Replay **6.1.1** (single-node, single-tenant default): `multi_tenant=false` → attestation code path absent. Replay **6.1.2** (distributed, single-tenant default): same. Multi-tenant deployments are a separate configuration with declared users. | `06-scenario-view.md:13-30` (6.1.1); `:37-58` (6.1.2) | **holds** |
| **A6-P8** (byte-cap on args) | Merged part into A1-P11. Remainder (A6-P8b byte-cap): single constant check at entry gate. Replay **6.1.1 step 1**: args_blob ≤ cap, passes; >cap, rejected. One compare. | `06-scenario-view.md:17` (step 1) | **holds** |
| **A6-P9** (Logical System isolation in wire + code) | Attack: isolation invariant already in design; P9 adds enforcement hooks at wire boundary. Replay **6.1.2 step 4** (REMOTE_SUBMIT): receiver verifies sender's Logical System ID matches; 1 compare. | `06-scenario-view.md:48` (step 4 of 6.1.2) | **holds** |
| **A6-P10** (capability-scoped log/trace sinks) | Attack: sink-capability check on log emit. Hot-path cost = 1 load + 1 compare (or inlined-away). Replay **6.1.1**: log emits in admission + completion paths; check inlined. | `06-scenario-view.md:17-30` | **holds** |
| **A6-P11** (gated `register_factory`) | Attack: secure-default policy on startup-only path. Replay **6.1.1 preconditions**: runtime init registers factories under policy; hot path unchanged. | `06-scenario-view.md:13-15` (preconditions) | **holds** |
| **A6-P12** (signed / schema-validated REPLAY trace) | Attack: **contingent on A9-P6 option**. Under synth option (iii) (A9 preferred): A6-P12 re-scoped to "schema frozen at ADR-011-R2 time; no v1 implementation." Under option (i): A6-P12 fully withdrawn. Under option (ii): A6-P12 retains v1 impl — this is where a stress-round re-attack would land. Replay: N/A (replay trace is sim-mode, not hot-path). | `06-scenario-view.md` (N/A — sim-only) | **holds (under option iii)** |
| **A6-P13** (per-tenant submit rate-limit) | Attack: multi-tenant-only; default off. Replay **6.1.1** (single-tenant): rate-limit absent. | `06-scenario-view.md:13-30` | **holds** |
| **A6-P14** (key-material lifecycle ADR) | Attack: doc-only. | `06-scenario-view.md` (N/A) | **holds** |
| **A7-P1** (break `scheduler/` ↔ `distributed/` cycle) | Attack: D6 hard rule; cycle removal is always simpler. Replay **6.1.2**: distributed path depends on scheduler interfaces; scheduler does not depend on distributed concretes. DAG restored. | `06-scenario-view.md:47-54` (6.1.2) | **holds** |
| **A7-P2** (role-split `ISchedulerLayer`) | Merged with A9-P1. Addressed in A9-P1. | `06-scenario-view.md:17` (step 1) | **holds** |
| **A7-P3** (invert `core/` ↔ `hal/` for handle types) | Attack: DIP fix; handle types move to `core/` where HAL consumes them. Replay **6.1.1 step 5-6**: register-bank + execution engine consume core handle types; core does not depend on hal concretes. | `06-scenario-view.md:22-23` (steps 5-6) | **holds** |
| **A7-P4** (move distributed payload structs to `distributed/`) | Attack: cohesion fix. Replay: structural. | `06-scenario-view.md` (doc layout) | **holds** |
| **A7-P5** (`distributed_scheduler` depends only on `ISchedulerLayer`) | Attack: narrow coupling. Replay **6.1.2 step 2**: Pod Scheduler consumes ISchedulerLayer role interfaces only. | `06-scenario-view.md:46` (step 2 of 6.1.2) | **holds** |
| **A7-P6** (extract MLR + deployment parser) | Attack: SRP. Replay: structural. | `06-scenario-view.md` (doc) | **holds** |
| **A7-P7** (forward-decl contract) | Maintenance. | N/A | **holds** |
| **A7-P8** (consolidate `ScopeHandle` ownership) | Attack: one owner per handle. Replay: structural. | `06-scenario-view.md` (doc) | **holds** |
| **A7-P9** (dedupe `MemoryError`) | DRY. | N/A | **holds** |
| **A8-P1** (injectable `IClock`) | Attack: single test seam; X5. Replay **6.1.1 step 2** (latency-budget timing): production Clock uses TSC; test double returns controlled values. One interface, two impls. KISS-compatible. | `06-scenario-view.md:19` (step 2) | **holds** |
| **A8-P2** (driveable event-loop, single `IEventLoopDriver`) | Merged with A9-P2 amendment. Single test seam; appendix lists future policy extensions with triggers. Replay **6.1.1 step 2-10**: production driver runs A→B→C loop; test driver uses `step()` + `RecordedEventSource` to advance deterministically. | `06-scenario-view.md:19-28` (steps 2-10) | **holds** |
| **A8-P3** (stats structs + latency histograms) | Attack: histogram update on hot path. Replay **6.1.1 step 2-10**: per-transition histogram bucket update = 1 atomic increment. Synth-amended to thread-local counters merged offline. Hot-path cost bounded. | `06-scenario-view.md:19-28` | **holds** |
| **A8-P4 ↔ A5-P6** | Addressed in A5-P6. | see A5-P6 | **holds** |
| **A8-P5** (external alert-rules file; Prom/OTEL as opt-in deviation) | A9 R2 concern: two systems at once. Amendment: alert-rule file location only; OTEL as known-deviation. Attack: is OTEL still in v1 prose? Confirmed: amendment puts OTEL in `10-known-deviations.md` ledger. Replay: N/A (config file, not hot path). | `06-scenario-view.md` (N/A — ops) | **holds** |
| **A8-P6** (distributed trace time-alignment contract) | Attack: doc. Replay **6.1.2**: trace IDs carry node-time offset; post-merge offline. | `06-scenario-view.md:56` (postconditions of 6.1.2) | **holds** |
| **A8-P7** (`IFaultInjector` sim-only seam) | Sim-only; does not touch hot path. | `06-scenario-view.md` (N/A — sim) | **holds** |
| **A8-P8** (AICore in-core trace upload protocol) | Attack: L2 opt-in. Replay: when enabled, AICore writes trace blob post-completion; off-hot-path by construction. | `06-scenario-view.md:25` (step 7) | **holds** |
| **A8-P9** (profiling drop/degraded as first-class alerts) | Alert rules; ops concern. | N/A | **holds** |
| **A8-P10** (structured KV logging primary surface) | Attack: log call-site cost = format → KV struct. Replay **6.1.1**: structured logs emit only at SUBMITTED/COMPLETED transitions (not per-step); cost amortized. | `06-scenario-view.md:17,28` | **holds** |
| **A8-P11** (HAL contract test suite) | Sim+onboard tests. | N/A | **holds** |
| **A8-P12** (stable `PhaseId`s for Submission lifecycle) | Attack: enum of phase IDs for trace correlation. Replay **6.1.1**: each transition emits a PhaseId tag; trace records carry PhaseId; offline consumer decodes. One enum, stable. | `06-scenario-view.md:19-28` | **holds** |
| **A10-P1** (default `producer_index` sharding policy) | Merged with A1-P14. | see A1-P14 | **holds** |
| **A10-P3** (per-data-element consistency model) | Doc. | N/A | **holds** |
| **A10-P4** (stateful/stateless classification) | Doc. | N/A | **holds** |
| **A10-P6** (faster peer-failure detection with hysteresis, dedicated heartbeat thread) | Attack: heartbeat on dedicated thread; hysteresis bounds false positives. Replay **6.2.2 step 1-2**: coordinator's dedicated heartbeat thread detects Node₁ loss; hysteresis prevents flapping from triggering premature failover. Not on hot path. | `06-scenario-view.md:105-106` (steps 1-2 of 6.2.2) | **holds** |
| **A10-P7** (two-tier sharded TaskManager) | A9 R2 concern: premature sharding. Amendment: `shards=1` default (unsharded fast path); sharded only when `LevelParam` overrides. Replay **6.1.1 step 1-11**: `shards=1` → single TaskManager instance, no shard hashing, no inter-shard routing. Fast-path unchanged. | `06-scenario-view.md:17-30` | **holds** |
| **A10-P8** (single "Data & State Reference" section in `07-cross-cutting-concerns.md`) | Doc consolidation. | N/A | **holds** |
| **A10-P9** (gate `WorkStealing` × `RETRY_ELSEWHERE`) | Attack: correctness guard. Replay: documentation constraint. | N/A | **holds** |

### 8.2 Disputed-proposal scenario replays (final-vote substantiation)

#### A2-P7 — async-policy extension seam

- **Amended text (R2 §5 of A2):** no interface in v1; `09-open-questions.md` Q-record naming two future policy axes + triggers.
- **Scenario replay (6.1.1 step 1-11 + 6.1.2 step 1-11):** under the Q-record framing, v1 runtime has **zero** async-policy surface. Admission path (6.1.1 step 1-2) is synchronous `std::expected` return. Distributed path (6.1.2) is synchronous handshake + REMOTE_SUBMIT + poll. No async-policy interface is referenced anywhere in the v1 flow.
- **Verdict:** **holds** under the Q-record amendment. A9 flips to agree (see §6).

#### A9-P2 — collapse policy-enum/interface pluggability

- **Amended text (R2 §5 of A9, augmented in §5 above):** cut `FULLY_SPLIT`/`SPLIT_DEFERRED`; collapse `IEventCollectionPolicy`/`IExecutionPolicy` to closed enums + config; retain single `IEventLoopDriver` test seam; appendix lists v2 extensions with trigger conditions.
- **Scenario replay (6.1.1 step 2-10 + 6.1.3 step 1-7):** production event loop is a plain A→B→C sequence parameterized by closed `ExecutionMode` enum; `SPLIT_COLLECTION` mode (Host/Pod with networked sources) runs collection on a separate thread. Test-harness replay: `IEventLoopDriver.step()` advances one cycle; `RecordedEventSource` injects events deterministically. A8-P2 DfT need satisfied.
- **A2's OCP concern:** closed `ExecutionMode` + `EventCollectionMode` enums are C++-ABI-closed; appendix entries for future extensions carry explicit trigger conditions (benchmark criteria, workload evidence), so "when to open" is documented, not judgmental.
- **Verdict:** **holds** under the amendment as augmented. A9 self-votes agree (see §6).

#### A9-P6 — defer PERFORMANCE / REPLAY simulation

- **Amended text (synth option iii):** ship v1 with `FUNCTIONAL` only; keep `SimulationMode` enum open; preserve REPLAY scaffolding (placeholder interfaces declared but not factory-registered); ADR-011-R2 records named triggers; A6-P12 re-scoped to "schema frozen; no v1 impl."
- **Scenario replay (6.1.1 step 1-11 under `SIM=FUNCTIONAL`):** Host-side CPU backend executes AICore Function as host code; no per-Function timing model required; no trace replay consumer. Steps 1-11 execute without invoking PERFORMANCE or REPLAY pipelines. A9's G2 concern (no second sim pipeline in v1) preserved.
- **A2/A6/A8 forward paths:** enum stays open (A2 E5 migration preserved); REPLAY scaffolding declared (A8 X5 DfT forward path preserved); schema frozen at ADR-011-R2 time (A6 S6 audit forward path preserved).
- **Verdict:** **holds** under option (iii). A9 votes agree (see §6).

### 8.3 Total incremental surface-area summary (scenarios 6.1.1 + 6.2.3)

Per the prompt's guidance to "measure total incremental surface area added by the agreed proposals" when replaying 6.1.1 + 6.2.3:

| Surface category | Incremental items added by 107 agreed proposals (post-amend) | A9 verdict |
|------------------|---------------------------------------------------------------|------------|
| New production interfaces | 1 (`IDistributedProtocolHandler`, A2-P6) — **contested**; amendment proposed to demote to free function | weak |
| New role interfaces (from split) | 3 (from role-splitting the existing `ISchedulerLayer`, A7-P2) — **subtractive** (fewer virtual methods than today) | pass |
| New test/sim-only seams | 3 (`IClock`, `IEventLoopDriver`, `IFaultInjector`) — one impl + one test double each | pass |
| New enum values on `TaskState` | +2 (`ERROR` blocker, `CANCELLED` optional, A3-P1) | pass (correctness) |
| New Worker state | +1 (`QUARANTINED`, A5-P9) — **contested**; amendment proposed to fold into `UNAVAILABLE + policy` | weak |
| Enums removed | −1 (`SubmissionDescriptor::Kind`, A9-P4); −1 (two admission enums → `AdmissionStatus`, A9-P5) | pass |
| Config structs removed | −1 (`SourceCollectionConfig` folded, A9-P7) | pass |
| Virtual methods removed | −3 (submit overloads, A9-P1); −2 (collectives off `IHorizontalChannel`, A9-P3) | pass |
| Deployment modes removed | −2 (`FULLY_SPLIT`, `SPLIT_DEFERRED`, A9-P2) | pass |
| Sim modes deferred | −2 (`PERFORMANCE`, `REPLAY` impl, A9-P6 option iii) | pass |
| Wire-message version bytes | +4 if A2-P1 kept blanket; +4 (multi-node only) if A2-P1 narrowed per §8 amendment — **contested** | weak |
| New `LevelParam` fields | `max_batch` (A1-P12, off-default), `shard_count` (A10-P7, default 1), `trust_boundary.multi_tenant` (A6-P7, default off), `unavailable_since`/`unavailable_policy` (A5-P9 if amended) | pass (two-tier: defaults = off) |
| New wire-message fields | `message_id` (A5-P10), `version` (A2-P9 trace; A2-P1 narrowed), `timeout` (A5-P7), `idempotent` bit (A5-P4), Logical System ID (A6-P9), signed payload (A6-P12 schema only) | pass-with-caveats |

**Net:** the 107 agreed proposals, after the amendments landed in R2 §5, produce a **net reduction** in framework-extension surface (virtual methods, deployment modes, sim impls, duplicate enums, config structs) and a **bounded increase** in correctness surface (error states, timeout fields, idempotency bits) and test seams. The three residual concerns (A2-P1 scope, A2-P6 framing, A5-P9 encoding) each have concrete A9 amendments recorded above; none is blocking.

### 8.4 No new disputes introduced

Per the prompt: "stress round passed with no new disputes introduced or at most a handful of minor amendments." A9's stress attack introduced **three minor amendments** (A2-P1 narrow-to-wire-and-persistence, A2-P6 demote-to-free-function, A5-P9 fold-into-`UNAVAILABLE+policy`) — all owner-routable, none blocking. A9 does **not** flip any previously-agreed proposal to disagree.

## 9. Status

- **Satisfied with current design?** **yes, with three minor amendments routed to owners.**
- **Open items:**
  - **A2-P1** (owner: A2) — narrow blanket versioning to multi-node wire messages + persistent artifacts; fold blanket obligation into A2-P5's evolution policy.
  - **A2-P6** (owner: A2) — demote `IDistributedProtocolHandler` abstract class to a `protocol.hpp` seam (free function / static) with single concrete v1 impl; introduce the abstract base via ADR the day a second backend roadmap appears.
  - **A5-P9** (owner: A5) — replace `QUARANTINED` state with `unavailable_since: Timestamp` + `unavailable_policy: {Permanent, Quarantine(duration)}` fields on the existing `UNAVAILABLE` state.
- **Final votes on disputed proposals (recorded §6):** A2-P7 = **agree** (flip, conditional on Q-record reframe). A9-P2 = **agree** (self-vote; amendment augmented with trigger conditions). A9-P6 = **agree to option (iii)** (synth-recommended).
- **Override requests:** 0. **Blocking objections:** 0. A9's primary rules (G2, G4) are design-discipline soft rules; A9 does not file blocking objections.
- **Convergence assessment from A9's vantage:** the stress round did not break consensus on any agreed proposal. The three disputed proposals all converge under the synth-recommended landing zones. Round 3 meets the parent's convergence criteria from A9's aspect.
