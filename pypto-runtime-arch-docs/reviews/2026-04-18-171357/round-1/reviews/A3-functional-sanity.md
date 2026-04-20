# Aspect A3: Functional Sanity & Correctness — Round 1

## Metadata

- **Reviewer:** A3 (Functional Sanity & Correctness)
- **Round:** 1
- **Target:** `docs/pypto-runtime-design/` (full package, rev as of 2026-04-18-171357)
- **Runtime Mode:** yes
- **Timestamp (UTC):** 2026-04-19T00:00:00Z

## 1. Rubric Checks

| # | Check | Result | Evidence (file:line) | Rule(s) |
|---|-------|--------|----------------------|---------|
| 1 | Every stated requirement traces to ≥ 1 scenario walkthrough | Weak | `01-introduction.md:32-57` (10 FR + 7 NFR) vs `06-scenario-view.md:1-155` (3 happy-path scenarios); FR-4 (four function types), FR-7 (content-hash function cache), FR-8 (stable C API), FR-9 (Logical System multi-tenancy), FR-10 (three simulation modes), NFR-2 (add-a-level OCP flow), NFR-3 (ONBOARD ↔ SIM equivalence), NFR-5 (backward DSL compat), NFR-7 (distributed trace correlation) have no scenario walkthrough; `04-process-view.md:206-225` "Function Registration and Caching Flow" is a diagram, not a traced scenario | G1, V3 |
| 2 | ≥ 1 failure scenario per critical path | Weak | `06-scenario-view.md:81-143` covers AICore hang, remote-node crash, task-slot exhaustion, RDMA timeout. But `04-process-view.md:641-694` defines **four** critical paths (§4.8.1 Single-Node Kernel Dispatch, §4.8.2 Cross-Node Submission, §4.8.3 Task Completion Path, §4.8.4 Submission Admission Budget); no dedicated failure scenario exists for §4.8.3 (completion → Python) or §4.8.4 (admission-time failures — cyclic `intra_edges`, workspace-alloc reject, `NONE`-mode misuse, admission-queue starvation) | V3, V4 |
| 3 | All interface implementations honor LSP (preconditions, postconditions, invariants) | Fail | `04-process-view.md:229-284` Task state diagram lists only `FREE → … → RETIRED`; no `ERROR` (or `CANCELLED`) state. But `04-process-view.md:591` ("error: ErrorContext?" on Task), `06-scenario-view.md:93` ("parent Orchestration Task transitions COMPLETING → ERROR"), `06-scenario-view.md:111` ("parent Task → ERROR"), `06-scenario-view.md:126` ("parent Task → ERROR"), and `06-scenario-view.md:138` ("Data Movement Task → ERROR") all reference a state that the state machine does not define. `02-logical-view/07-task-model.md:172-186` "Task State (Summary)" confirms the omission. Also `02-logical-view/09-interfaces.md:28-38` `submit(...)` returns `SubmissionHandle` by value, but `02-logical-view/09-interfaces.md:65` says it "may block or return a back-pressure status" — the signature has no error-return slot, so LSP contract is inconsistent with `IResourceAllocationPolicy::AdmissionDecision::{WAIT,REJECT}` in `02-logical-view/09-interfaces.md:242-245`. Worker state machine `04-process-view.md:321-361` adds `FAILED/RECOVERING/UNAVAILABLE` branches that do not exist for all level types (`04-process-view.md:384-391` notes AICore has no `COMPLETING`) — acceptable if documented, but the "uniform across all layers" claim in `04-process-view.md:229-231` and `04-process-view.md:321-323` is not literally true | LSP, D7, V5 |
| 4 | Ambiguous / missing info marked `[ASSUMPTION]` | Weak | Explicit `[ASSUMPTION]` tags present in `01-introduction.md:71-76`, `04-process-view.md:701`, `05-physical-view.md:169, 187`, `07-cross-cutting-concerns.md:31, 46` — good. Implicit-but-unmarked assumptions: (a) `04-process-view.md:105` "each thread runs an independent partition of the event space" — partition scheme unspecified; (b) `02-logical-view/07-task-model.md:201` "complete_in_future" completion-discovery protocol is unspecified (acknowledged as Q8, but no `[ASSUMPTION]` tag at the point of use); (c) `02-logical-view/07-task-model.md:220` "SPMD sub-tasks are mutually independent" is a caller assertion mirroring `dep_mode=NONE` but not tagged; (d) `04-process-view.md:561` "causal consistency per task chain" — network ordering assumptions (FIFO per-link, idempotent delivery) are not explicit | G3 |
| 5 | Explicit, not implicit, error conditions in Process View | Weak | `04-process-view.md:568-633` defines error taxonomy, code system, context, propagation paths, fatal vs recoverable, Python mapping — good breadth. Gaps: (a) no `ERROR` state in the normative state machine (cross-ref to rubric #3); (b) `06-scenario-view.md:139` "Consumer tasks' dependency counter is not decremented; they eventually timeout" — timeout owner, duration, and poison-pill propagation unspecified; (c) `02-logical-view/02-scheduler.md:70` "If any of steps 1–4 fails … the admission is rolled back as a whole" does not specify which `ErrorCode` surfaces to the caller (`modules/error.md:73` defines `AdmissionRejected`, `CyclicDependency`, `WorkspaceExhausted`, but the mapping from admission-rollback step → code is not given); (d) parent sibling behavior on one child's failure is undefined — `04-process-view.md:613` says "aggregated" but does not specify whether remaining siblings are cancelled, allowed to run to completion, or both | — |
| 6 | Edge cases (empty / max / concurrent / out-of-order) enumerated | Weak | Empty: empty-submission (`SubmissionDescriptor.tasks.size() == 0`) behavior not specified (`02-logical-view/07-task-model.md:30-38` says "one or more Tasks" but no admission-time rejection is documented); empty `boundary_in_tasks` under `DATA` mode is implicitly a no-op but not stated. Max: no bound on `intra_edges.size()` or `spmd_count`; unbounded RAW fan-in under `DATA` is limited by window depth × arg count (§4.8.4) but max `producer_index` size during steady-state is not stated. Concurrent: (a) `02-logical-view/02-scheduler.md:107-124` admission/eviction concurrency is covered; (b) `drain()` vs concurrent `submit()` is undefined (`02-logical-view/09-interfaces.md:55, 68` — "blocks until all submitted Tasks … RETIRED" — does submit() after drain() start succeed, block, or fail?); (c) concurrent admission from `scheduler_thread_count > 1` assigning `submission_id` — monotonicity contract under multi-thread admission not spelled out. Out-of-order: (a) `REMOTE_COMPLETE` arriving before its `REMOTE_SUBMIT` has been ACKed (network reordering over UDP/RDMA with multi-QP) not discussed; (b) `notify_child_complete` arriving before `notify_dep_satisfied` for the same parent — handler ordering at the Scheduler event loop not shown | V3, V4 |

## 2. Pros

- `06-scenario-view.md:1-155` provides genuine four-view scenario walkthroughs with per-step Logical / Development / Process / Physical columns — this is the strongest V3 artifact in the package. — cites V3, found at `06-scenario-view.md:15-29`.
- The Submission / Task split, boundary classification, and `dep_mode` contract are explicit, normative, and accompanied by an invariant section (non-aliasing intermediate memrefs) that is traceable to a frontend specification. Correctness preconditions are stated, not implied. — cites G3, LSP, found at `02-logical-view/12-dependency-model.md:105-128` and `02-logical-view/04-memory.md:66-68`.
- Admission concurrency is analyzed per `dep_mode` with per-mode shared-state lists and lock obligations (`BARRIER`, `DATA`, `NONE`). This pre-empts a typical correctness hole where the runtime silently races admission against retirement. — cites LSP, G3, found at `02-logical-view/02-scheduler.md:107-124`.
- `ErrorCode` packed layout, per-domain partition, and stability contract in `modules/error.md:22-113` give explicit error semantics with severity routing, and `ErrorContext` has an immutable contract, bounded cause depth, and a device-POD downgrade path — correctness of propagation is pinned down. — cites SRP, G3, found at `modules/error.md:169-176, 349-355`.
- `ResourceAllocationResult` and `AdmissionDecision` enumerate `ALLOCATED / WAIT / REJECTED` and `ADMIT / WAIT / REJECT` explicitly, making back-pressure vs. hard-rejection an in-contract distinction rather than implicit behavior. — cites G3, found at `02-logical-view/09-interfaces.md:242-245, 294-299`.
- Known Deviations and Open Questions are maintained as first-class artifacts with rule IDs (`10-known-deviations.md:1-52`), satisfying the "state assumptions explicitly" rule at the package level. — cites G3, found at `10-known-deviations.md:1-52`.
- Atomic admission with full rollback on any sub-step failure is explicitly contracted (slots + edges + workspace are either all installed or all released). — cites LSP, found at `02-logical-view/02-scheduler.md:62-70`.
- `TaskHandle.generation` is declared as the correctness backstop for stale lookups ("independent mechanism from the dep-metadata lock"). The runtime does not rely on a single synchronization mechanism for handle validity. — cites G3, found at `02-logical-view/02-scheduler.md:121` and `02-logical-view/12-dependency-model.md:93`.

## 3. Cons

- **`ERROR` (and implicit `CANCELLED`) state is absent from the normative Task state diagram** but is referenced in five places in the Scenario View and in the Task `error: ErrorContext?` field. This is an LSP + D7 violation: the uniform "Task state machine" cannot be uniformly implemented if the diagram is incomplete. — cites LSP, D7, V5, at `04-process-view.md:229-284` vs `06-scenario-view.md:93, 111, 126, 138`.
- **Requirement → scenario traceability is incomplete** for FR-4, FR-7, FR-8, FR-9, FR-10, NFR-2, NFR-3, NFR-5, NFR-7. In particular FR-10 introduces three simulation modes (`PERFORMANCE`, `FUNCTIONAL`, `REPLAY`) and NFR-3 requires ONBOARD↔SIM equivalence, but no scenario walks a kernel through `SIM`. — cites G1, V3, at `01-introduction.md:45, 53` vs `06-scenario-view.md:1-155`.
- **Failure scenarios do not cover the admission critical path (§4.8.4)**: no walk-through for cyclic `intra_edges` rollback, workspace-allocation rejection, `NONE`-mode misuse, empty-submission rejection, admission-queue starvation (long-running early Submission blocking the window), or producer-index eviction-before-lookup generation-guard firing. — cites V4, R6, at `04-process-view.md:681-694`.
- **`ISchedulerLayer::submit(...)` return-type vs. admission semantics mismatch.** `SubmissionHandle submit(const SubmissionDescriptor&)` has no error-return slot, yet `IResourceAllocationPolicy::AdmissionDecision` can `REJECT`, and the narrative says submit() "may … return a back-pressure status". Either submit() throws on `REJECT` / blocks on `WAIT`, or the return contract must be widened to something like `std::variant<SubmissionHandle, AdmissionDecision>`. Current text is ambiguous and leaves implementers free to violate LSP across layers. — cites LSP, D7, at `02-logical-view/09-interfaces.md:28, 65` vs `02-logical-view/09-interfaces.md:242-245`.
- **Sibling task behavior on partial failure is unspecified.** `04-process-view.md:613` says multiple child errors are "aggregated into the parent's error context" but does not state whether still-running siblings are cancelled (so `aggregate()` can produce a deterministic fan-in) or allowed to complete (so aggregation must wait). This affects idempotency (DS4) and determinism of `ErrorContext.remote_chain`. — cites DS4, G3, at `04-process-view.md:605-615`.
- **Consumer-of-failed-producer is under-specified.** `06-scenario-view.md:139-140`: "Consumer tasks' dependency counter is not decremented; they eventually timeout." No contract states who owns the timeout, how long it is, or whether a poison-pill `DEP_SATISFIED_WITH_ERROR` is emitted. The dependency model (`02-logical-view/12-dependency-model.md:74-103`) describes insertion and eviction of producer-index entries but does not define producer-failure propagation to consumers. — cites R1, DS3, at `06-scenario-view.md:130-142`, `02-logical-view/12-dependency-model.md:74-103`.
- **`DepMode::NONE` is a caller-asserted correctness contract with no runtime-detectable misuse path.** `02-logical-view/07-task-model.md:57` explicitly says "Using NONE when a real dependency exists is a correctness bug, not a runtime-detected error." That is a reasonable design choice, but there is no scenario, no debug-mode cross-check, and no diagnostic hook that would catch this in testing. — cites G3, X6, at `02-logical-view/07-task-model.md:57`.
- **Empty / degenerate submissions are not addressed.** SubmissionDescriptor has no stated preconditions on `tasks.size() >= 1`, `boundary_in_tasks ⊆ [0,tasks.size())`, `intra_edges` self-loop rejection, `boundary_in ∩ boundary_out` constraints, or `workspace_request` sub-range vs. task index bounds. A malformed descriptor from `bindings/` is a trust-boundary input (S3 area) but also a correctness input that must be validated before admission rollback can even be defined. — cites G3, S3, at `02-logical-view/09-interfaces.md:98-120`.
- **`drain()` concurrency contract is silent.** `drain()` is specified to block until all submitted tasks reach `RETIRED`, but `submit()` during drain, or drain during drain, is undefined. This matters for shutdown correctness and for multi-tenant `LogicalSystem` termination. — cites G3, R4, at `02-logical-view/09-interfaces.md:55, 68`.
- **Out-of-order cross-node delivery is not bounded.** `04-process-view.md:542-561` lists six message types and a "causal consistency per task chain" guarantee, but does not specify the underlying transport ordering requirement (per-link FIFO? per-{src,task} FIFO?) that makes the causal guarantee hold. Under multi-path RDMA, `REMOTE_COMPLETE` can arrive before `REMOTE_SUBMIT` is ACKed. — cites DS3, G3, at `04-process-view.md:540-561`.
- **`complete_in_future` completion-discovery is deferred (Q8) but used in `§2.4.5`** without an `[ASSUMPTION]` marker at the point of use. Every Data Movement Task on the distributed path depends on this, so it is a correctness-critical hole, not just an open question. — cites G3, at `02-logical-view/07-task-model.md:201`, `09-open-questions.md:99-109`.
- **`COMPLETING` state applicability across levels is soft.** The state machine is "uniform across all layers" but `04-process-view.md:384-391` notes `COMPLETING` "not applicable" at Core level. Either the diagram must show a skip-transition, or the uniformity claim must be qualified. — cites LSP, V5, at `04-process-view.md:229-231, 384-391`.
- **Cyclic `intra_edges` detection is promised at admission but the mechanism and complexity are not documented.** A linear-time topological check vs. DFS with white/grey/black is a correctness-relevant admission cost (§4.8.4 budget 100 ns — a topological check over an N-task Group can't fit there for large N without specification). — cites G3, X9, at `02-logical-view/02-scheduler.md:70`, `04-process-view.md:681-694`.
- **SPMD `spmd_index` / `spmd_size` delivery mechanism is informally described** ("each sub-task receives implicit `spmd_index` and `spmd_size`") without a declared conduit (scalar args, register file entry, well-known memory slot). Correctness of any SPMD kernel depends on this mechanism being contracted. — cites G3, LSP, at `02-logical-view/07-task-model.md:217, 205-223`.
- **`bindings/` Python exception mapping is not exhaustive for `ErrorCode` domains.** `04-process-view.md:625-633` lists 5 Python exception classes, but `modules/error.md:35-45` defines 9 `Domain` values (Hal, Core, Scheduler, Memory, Transport, Distributed, Profiling, Runtime, Bindings) plus `Generic`. The `python_mapping_completeness` test (`modules/error.md:381`) is promised but no mapping table exists for Memory/Transport/Profiling domains. — cites G1, D7, at `04-process-view.md:625-633` vs `modules/error.md:381`.

## 4. Proposals

| id | severity | summary | affected_docs | hot_path_impact | trade_offs | sanity_test |
|----|----------|---------|---------------|-----------------|------------|-------------|
| A3-P1 | blocker | Add `ERROR` (and optional `CANCELLED`) state to normative Task state machine; cover transitions from every state | `04-process-view.md` §4.3.1, §4.3.2; `02-logical-view/07-task-model.md` §2.4.4 | none | Gain: diagrams match narrative; implementers cannot diverge; error propagation becomes testable. Give up: ~8 new transitions to document | Grep docs for `→ ERROR` / `ERROR →`; every occurrence must match a transition in the updated diagram; unit test exercises `on_fail()` from every EXECUTING / COMPLETING / DISPATCHED state |
| A3-P2 | high | Clarify `ISchedulerLayer::submit(...)` return contract: either (a) block on `WAIT` and throw on `REJECT`, or (b) widen return to `Expected<SubmissionHandle, AdmissionDecision>` | `02-logical-view/09-interfaces.md` §2.6.1, §2.6.3; `04-process-view.md` §4.5.6 | none (decision only; no hot-path change if (a)); may allocate if (b) uses `std::expected` with non-trivial error | Gain: LSP compliance; callers know how to handle back-pressure. Give up: commits to one style | Author two-line pseudocode snippets for `submit() → ADMIT`, `submit() → WAIT`, `submit() → REJECT` — each must compile against the chosen signature without ambiguity |
| A3-P3 | high | Add an **Admission-Path Failure scenario** to §6.2: cyclic `intra_edges`, workspace-alloc reject, `NONE`-mode caller bug with debug assertion, empty-submission rejection | `06-scenario-view.md` §6.2 (new §6.2.5–§6.2.7) | none | Gain: one failure scenario per §4.8 critical path (closes V4 gap for §4.8.4). Give up: +1 page of scenario text | Cross-check: for each row in `04-process-view.md` §4.8 critical-path table, at least one `06-scenario-view.md` §6.2 sub-section names that path in its "Context" field |
| A3-P4 | high | Specify **consumer-of-failed-producer** propagation: emit `DEP_FAILED(producer_key, ErrorCode)` event; consumer transitions `PENDING → ERROR` (not timeout); add to `SchedulerEvent::Type` enum | `02-logical-view/02-scheduler.md` §2.1.3.5 `SchedulerEvent`; `04-process-view.md` §4.5, §4.7.4; `06-scenario-view.md` §6.2.4 | extends-latency (one extra event per failed producer, bounded by fan-out) | Gain: removes "eventually timeout" hand-wave; deterministic error propagation. Give up: extra event type; slow-path latency increase only on error | Injection test: fail a producer task; assert every downstream consumer reaches `ERROR` within one event-loop iteration after the producer's `ERROR` event, without relying on a wall-clock timeout |
| A3-P5 | high | Specify **sibling cancellation policy** on parent-sees-child-failure: document default = "cancel still-running siblings, aggregate errors from completed+cancelled+failed"; expose policy hook | `04-process-view.md` §4.7.4; `02-logical-view/02-scheduler.md` §2.1.3.1 (TaskManager responsibilities); `06-scenario-view.md` §6.2.1, §6.2.2 | extends-latency on error (cancellation traversal only) | Gain: deterministic `ErrorContext.remote_chain` fan-in; no silent lingering siblings. Give up: must define idempotent cancellation of in-flight siblings | Inject failure in one child of a 3-child Orchestration Task; assert parent `ErrorContext` contains exactly one `cause` + two `CANCELLED` entries in `remote_chain`; no sibling Worker remains `EXECUTING` after propagation completes |
| A3-P6 | high | Add **Requirement ↔ Scenario traceability matrix** to §6.1; for each FR/NFR not covered by §6.1.1–6.1.3, add a scenario or mark `[not scenario-tested: covered by ADR-00X / module test]` | `06-scenario-view.md` (new §6.0 table); new §6.1.4 Function Registration & Content-Hash Cache (FR-7); §6.1.5 Simulation Mode Walkthrough (FR-10, NFR-3); §6.1.6 Add-a-Level OCP Flow (NFR-2); §6.1.7 Multi-Tenancy Isolation (FR-9) | none | Gain: every requirement has a visible verification path; rubric #1 Pass. Give up: ~3–4 new scenario sub-sections | Run a verification script: for each FR-N / NFR-N in §1.3, grep §6 for explicit mention; zero misses |
| A3-P7 | medium | Specify **empty / degenerate submission** preconditions and reject paths in Submission contract | `02-logical-view/09-interfaces.md` §2.6.1.A; `02-logical-view/02-scheduler.md` §2.1.3.1.A (admit step 0) | none (adds one branch before slot allocation on admission path, on the cold pre-admission path not the steady hot path) | Gain: closes "empty submission / self-loop / boundary-index OOB / workspace subrange OOB" correctness holes. Give up: admission has a small fixed-cost validation step | Submit (a) empty `tasks`, (b) `intra_edges` with self-loop, (c) `boundary_in_tasks` index OOB, (d) `workspace_request.subranges[i].size > total_bytes`; each must return `AdmissionRejected` with a distinct sub-code |
| A3-P8 | medium | Document **cyclic `intra_edges` detection algorithm and cost** in the admission path; bound N-task Group validation time | `02-logical-view/02-scheduler.md` §2.1.3.1.A step 2; `04-process-view.md` §4.8.4 (admission budget) | extends-latency (O(V+E) DFS on admit; bounds must be added to §4.8.4) | Gain: X9 latency budget becomes honest; implementers have a concrete target. Give up: admission budget grows for large Groups | Measure `admit()` time vs. `|tasks|` on synthetic Groups (N = 1, 10, 100, 1000); assert O(V+E) scaling and a documented per-edge cost |
| A3-P9 | medium | Define **SPMD `spmd_index` / `spmd_size` delivery mechanism** explicitly (e.g., carried as scalar args in the generated `TaskArgs`); add to `06-function-types.md` AICore Function contract | `02-logical-view/06-function-types.md`; `02-logical-view/07-task-model.md` §2.4.6 | none (compile-time slot in `TaskArgs`) | Gain: SPMD kernels have a contracted ABI; frontend and runtime agree. Give up: TaskArgs gains two well-known scalar slots | Golden test: frontend emits SPMD with `spmd_count = 24`; each sub-task reads its `spmd_index` and writes it to an output tensor; result must be `[0, 1, ..., 23]` without runtime reassignment |
| A3-P10 | medium | Complete **Python exception mapping** for all `Domain` values; add mapping table to `bindings.md` and reference from `04-process-view.md §4.7.6` | `04-process-view.md` §4.7.6; `modules/bindings.md` §2.2 (exception mapping) | none | Gain: every error reaches Python deterministically; `python_mapping_completeness` test from `modules/error.md:381` is satisfiable. Give up: a few new exception classes to document | Static assertion: iterate over `Domain::*` enum at test time; assert each maps to a declared `simpler.*Error` subclass |
| A3-P11 | medium | Mark `complete_in_future` completion-discovery as `[ASSUMPTION]` at point of use; cross-reference Q8 explicitly | `02-logical-view/07-task-model.md` §2.4.5; `09-open-questions.md` Q8 | none | Gain: G3 satisfied; reader is warned that this path is unresolved. Give up: one marker + one cross-ref | Grep: `[ASSUMPTION]` appears next to `complete_in_future` in `07-task-model.md` and points at `09-open-questions.md#q8` |
| A3-P12 | medium | Specify **`drain()` / `submit()` concurrency semantics**: after `drain()` is entered, subsequent `submit()` rejects with `DrainInProgress` error; document in interface contract | `02-logical-view/09-interfaces.md` §2.6.1; `modules/error.md` §2.1 (new `DrainInProgress` code in Core domain) | none | Gain: shutdown / tenant termination has a deterministic contract. Give up: one new error code + one atomic flag read on admission | Race test: start `drain()`; concurrently call `submit()` 1000×; assert every post-drain call returns `DrainInProgress` and every in-flight task reaches `RETIRED` |
| A3-P13 | medium | Pin **cross-node message ordering assumption** (per-link FIFO, idempotent delivery) in §4.6 and add `[ASSUMPTION]` tags; define behavior under out-of-order `REMOTE_COMPLETE` before ACK of `REMOTE_SUBMIT` | `04-process-view.md` §4.6.1–§4.6.3 | none on hot path; slow-path reorder buffer discussed | Gain: causal-consistency guarantee is defensible; transport replacement (e.g., multi-path RDMA) doesn't silently break correctness. Give up: small reorder buffer cost in distributed code | Review `04-process-view.md` §4.6 mentions of "FIFO" / "idempotent" vs `[ASSUMPTION]` markers; assert one per claim |
| A3-P14 | low | Qualify the "uniform state machine" claim and document **`COMPLETING` skip at leaf Workers** as an explicit transition in §4.3.5 | `04-process-view.md` §4.3.5 | none | Gain: LSP consistency; the uniformity language matches the table. Give up: one row / one note | Grep: every level in `04-process-view.md:384-391` has a consistent "Skip COMPLETING?" column matching the diagram |
| A3-P15 | low | Add a **debug-mode `NONE`-dep-mode cross-check** hook: when `dep_mode = NONE` and debug build, run a `DATA`-mode dry-run and log any edge the user asserted away | `02-logical-view/02-scheduler.md` §2.1.3.1.B; `07-cross-cutting-concerns.md` §7.2 | none in release; cold path in debug | Gain: the caller-asserted contract is now testable in CI. Give up: debug-only overhead + one config knob | CI test: intentionally mis-declare `NONE` for a RAW-dependent submission under debug build; assert a WARN log line is emitted and correctness still holds (RAW edge is still installed in debug) |

### Proposal detail

#### A3-P1: Add `ERROR` (and optional `CANCELLED`) state to Task state machine

- **Rationale:** LSP, D7, V5. The Task state machine claims uniformity (`04-process-view.md:229-231`) but the normative state diagram (§4.3.1) omits states that are referenced five times in the Scenario View and carried as `Task.error` on `Task`. Without the state, two implementations of `ISchedulerLayer` can legally diverge in how they surface "Task failed" — one may use a boolean flag + `COMPLETED`, another may reuse `RETIRED`, a third may invent a local `ERROR` state. That violates LSP for the uniform contract.
- **Edit sketch:**
  - File: `04-process-view.md`
  - Location: §4.3.1 state diagram; §4.3.2 handler table
  - Delta: Add `ERROR` state reachable from `DISPATCHED`, `EXECUTING`, `COMPLETING`; add optional `CANCELLED` state reachable from `PENDING`, `DEP_READY`, `RESOURCE_READY`, `DISPATCHED`, `EXECUTING`; add `on_fail(task, error)` and `on_cancel(task, reason)` handler rows. Mirror in `02-logical-view/07-task-model.md:172-186`.
- **Trade-offs:**
  - Gain: diagrams and narrative agree; implementers have a single source of truth; tests can assert on `state == ERROR`.
  - Give up: ~8 new transition arrows to draw; one handler row per new state.
- **Sanity test:** A grep-based lint over `docs/pypto-runtime-design/` finds zero `→ ERROR` / `→ CANCELLED` references that are not covered by an edge in the updated diagram. A unit test drives a Task through `EXECUTING → on_fail` and asserts reaching `ERROR` (not `COMPLETED`).

#### A3-P2: Clarify `submit()` return contract against `AdmissionDecision`

- **Rationale:** LSP, D7. `SubmissionHandle submit(const SubmissionDescriptor&)` has no error channel in its signature (`02-logical-view/09-interfaces.md:28`), yet the narrative at `02-logical-view/09-interfaces.md:65` allows it to "return a back-pressure status" and `AdmissionDecision` has three values including `REJECT`. Either the signature needs an error path or the narrative is wrong.
- **Edit sketch:**
  - File: `02-logical-view/09-interfaces.md`
  - Location: §2.6.1 `ISchedulerLayer::submit`; §2.6.3 `IResourceAllocationPolicy::should_admit`
  - Delta: Pick one of (a) `submit(...)` blocks on `WAIT`, throws `simpler::RuntimeError(AdmissionRejected)` on `REJECT`, returns `SubmissionHandle` only on `ADMIT`; or (b) change signature to `std::expected<SubmissionHandle, ErrorContext>` / `Status submit(const SubmissionDescriptor&, SubmissionHandle* out)`. Document the choice in `08-design-decisions.md` as an ADR.
- **Trade-offs:**
  - Gain: one normative answer; LSP compliant; hot-path layers can opt into the non-blocking form.
  - Give up: option (a) commits callers to exception-style error handling; option (b) widens the ABI.
- **Sanity test:** Write three mock caller snippets — ADMIT / WAIT / REJECT — each must compile and type-check against the chosen signature without branching on side channels. A conformance test exercises all three paths against a mock `IResourceAllocationPolicy`.

#### A3-P3: Add admission-path failure scenario

- **Rationale:** V4, R6. `04-process-view.md §4.8.4` declares admission a critical path, but `06-scenario-view.md §6.2` has no failure scenario for it. §6.2 failures cover compute and network only.
- **Edit sketch:**
  - File: `06-scenario-view.md`
  - Location: new §6.2.5 "Admission Rollback on Cyclic `intra_edges`", §6.2.6 "Admission Rejection on Workspace Exhaustion", §6.2.7 "`NONE` Mode Misuse (Debug-Mode Detection)"
  - Delta: 3–5 row step tables each, mirroring the existing §6.2.1 structure; cite `AdmissionRejected` / `WorkspaceExhausted` / `CyclicDependency` error codes from `modules/error.md:70-79`.
- **Trade-offs:**
  - Gain: closes V4 coverage gap on §4.8.4; each error code defined in `modules/error.md` has at least one scenario demonstrating it.
  - Give up: page length grows by ~1 page.
- **Sanity test:** For each row in `04-process-view.md §4.8` (four critical-path tables), at least one §6.2 sub-section names that path in its "Context" field. Zero misses.

#### A3-P4: Specify producer-failure → consumer propagation

- **Rationale:** R1, DS3, G3. `06-scenario-view.md:139-142` says consumers "eventually timeout" when their producer fails. That is neither R1-compliant (timeout owner / duration not specified) nor deterministic. The dependency model defines producer-index eviction on retirement but not on failure.
- **Edit sketch:**
  - File: `02-logical-view/02-scheduler.md`
  - Location: §2.1.3.5 `SchedulerEvent::Type` enum; add `DEP_FAILED`
  - Delta: Add `DEP_FAILED(producer_key, error_code)` event type. TaskManager on `TASK_FAILED(producer)` walks producer's successors list and emits `DEP_FAILED` to each. Consumer on `DEP_FAILED` transitions `PENDING → ERROR` (not `DEP_READY`) and propagates recursively. Cross-reference from `04-process-view.md §4.7.4` propagation paths and §4.5 dependency model; update `06-scenario-view.md §6.2.4` steps 4–5.
- **Trade-offs:**
  - Gain: deterministic error propagation across the dependency DAG; no wall-clock timeout on the cold error path.
  - Give up: one new event type; slow-path fan-out walk on failure.
- **Sanity test:** Fail an AICore kernel that has 3 downstream consumers in the same Submission. Assert all 3 consumers reach `ERROR` within one event-loop iteration of the failure event, with `ErrorCode` equal to the producer's code wrapped in `DependencyFailed`.

#### A3-P5: Specify sibling cancellation policy

- **Rationale:** DS4, G3. Cross-layer error propagation says siblings are "aggregated" without specifying whether still-running siblings are cancelled. If they are not cancelled, `aggregate()` must wait for their completion before the parent can enter `ERROR` — which blocks the parent's own error propagation for an unbounded time.
- **Edit sketch:**
  - File: `04-process-view.md`
  - Location: §4.7.4 cross-layer propagation; §4.3.2 `on_completion` handler
  - Delta: Add paragraph: "On first child failure, TaskManager sets parent's `child_error_detected` flag; subsequent sibling dispatches of the same parent are cancelled (`CANCELLED` state, A3-P1) before dispatch; siblings already EXECUTING run to completion (so Worker contract is preserved) but their outputs are discarded and they contribute `CANCELLED` entries to `ErrorContext.remote_chain`. Expose policy via `IResourceAllocationPolicy::on_child_error()` hook." Cross-reference `06-scenario-view.md §6.2.1` step 5.
- **Trade-offs:**
  - Gain: deterministic error aggregation time; no unbounded wait.
  - Give up: `IResourceAllocationPolicy` gains one hook; cancellation semantics must be implemented per worker type.
- **Sanity test:** Parent with 3 children where child A fails instantly, child B takes 10 ms, child C is still pending. Assert parent `ErrorContext.remote_chain` has exactly 3 entries (A: fail, B: complete or cancel, C: cancel) within a bounded time (one event-loop iteration after A fails, for the pending child).

#### A3-P6: Requirement ↔ Scenario traceability matrix

- **Rationale:** G1, V3. Nine requirements (FR-4/7/8/9/10, NFR-2/3/5/7) have no scenario walkthrough. Rubric #1 cannot be Passed without this.
- **Edit sketch:**
  - File: `06-scenario-view.md`
  - Location: new §6.0 "Requirement Traceability"
  - Delta: a single table mapping each FR-N / NFR-N to §6.1.x (scenario) or to an ADR / module test that verifies it. Add new happy-path scenarios §6.1.4 (Function Registration + Content Hash), §6.1.5 (Simulation Mode: PERFORMANCE / FUNCTIONAL / REPLAY swap), §6.1.6 (Register a new Machine Level — OCP flow for NFR-2), §6.1.7 (Multi-Tenancy Isolation — FR-9).
- **Trade-offs:**
  - Gain: every requirement has a visible verification path.
  - Give up: ~3–4 new scenario sub-sections.
- **Sanity test:** Automated: a short script iterates §1.3's FR / NFR IDs and greps §6 for each; zero misses.

#### A3-P7: Submission precondition validation

- **Rationale:** G3, S3. `SubmissionDescriptor` has no stated preconditions. A malformed descriptor is both a C-API trust-boundary input and a correctness input; without preconditions, admission rollback is undefined.
- **Edit sketch:**
  - File: `02-logical-view/09-interfaces.md`
  - Location: §2.6.1.A `SubmissionDescriptor`; add "Preconditions" subsection
  - Delta: List preconditions: `tasks.size() >= 1`; `intra_edges[i].producer_index != intra_edges[i].consumer_index`; `intra_edges[i].{producer,consumer}_index < tasks.size()`; `boundary_in_tasks ⊆ [0, tasks.size())`; `boundary_out_tasks ⊆ [0, tasks.size())`; `workspace_request.subranges[i].task_index < tasks.size()`; `sum(subranges[i].size_aligned) <= total_bytes`. `ISchedulerLayer::submit()` returns `AdmissionRejected` with sub-codes `EmptyTasks / SelfLoopEdge / EdgeIndexOOB / BoundaryIndexOOB / WorkspaceSubrangeOOB / WorkspaceSizeMismatch` when violated. Add to `modules/error.md` §2.1 Core domain.
- **Trade-offs:**
  - Gain: each failure mode has a distinct error code; admission rollback is well-defined.
  - Give up: small fixed-cost validation step on admission (before slot allocation — not on the per-task hot path once admitted).
- **Sanity test:** Send each degenerate descriptor listed above; assert `submit()` returns the corresponding distinct error sub-code. Fuzz test with `libFuzzer` over `SubmissionDescriptor` bytes; zero crashes.

#### A3-P8: Cyclic `intra_edges` detection algorithm + cost

- **Rationale:** G3, X9. `02-logical-view/02-scheduler.md:70` mandates rollback on "cyclic `intra_edges`" but specifies neither the algorithm nor the cost. `04-process-view.md §4.8.4` budgets "< 100 ns amortized" for window check + slot reservation; cycle detection over a 1000-task Group cannot fit.
- **Edit sketch:**
  - File: `02-logical-view/02-scheduler.md`
  - Location: §2.1.3.1.A admit() step 2
  - Delta: Specify DFS with white/grey/black coloring; cost is `O(|tasks| + |intra_edges|)`; add row to `04-process-view.md §4.8.4` table: "Cyclic intra_edges check | `< 50 ns` per edge amortized | DFS over |intra_edges|".
- **Trade-offs:**
  - Gain: X9 budget honest; implementers have a target.
  - Give up: admission budget grows with `|intra_edges|`.
- **Sanity test:** Benchmark `admit()` for synthetic Groups with N tasks, 2N edges, for N in {1, 10, 100, 1000}; assert O(V+E) scaling and documented per-edge cost.

#### A3-P9: SPMD index/size delivery contract

- **Rationale:** G3, LSP. "Each sub-task receives implicit `spmd_index` and `spmd_size`" is narrative. AICore kernels are compiled against a specific ABI; frontend and runtime must agree on how these scalars are delivered.
- **Edit sketch:**
  - File: `02-logical-view/07-task-model.md`
  - Location: §2.4.6 SPMD Submission
  - Delta: Specify that `spmd_index` and `spmd_size` are carried in `TaskArgs.scalar_args` at reserved indices 0 and 1; document in `06-function-types.md` AICore Function ABI.
- **Trade-offs:**
  - Gain: deterministic SPMD ABI; frontend-runtime contract.
  - Give up: two reserved scalar slots.
- **Sanity test:** Frontend emits SPMD(N=24) that writes `spmd_index` to per-instance output; runtime dispatches; output tensor must equal `[0..23]`.

#### A3-P10: Python exception mapping completeness

- **Rationale:** G1, D7. `modules/error.md:381` promises a `python_mapping_completeness` test, but `04-process-view.md §4.7.6` maps only 5 `Domain` values. Memory / Transport / Profiling / Bindings domains have no Python exception.
- **Edit sketch:**
  - File: `modules/bindings.md`
  - Location: §2.2 exception mapping
  - Delta: Add a table with one row per `Domain` mapping to a `simpler.*Error` subclass; update `04-process-view.md §4.7.6` to reference it.
- **Trade-offs:**
  - Gain: every error domain reaches Python deterministically.
  - Give up: a few new exception class names.
- **Sanity test:** Static assertion at `bindings/` test time iterates over `Domain::*` enum; every value maps to a declared exception class. Zero unmapped domains.

#### A3-P11: `[ASSUMPTION]` marker at `complete_in_future` point of use

- **Rationale:** G3. `02-logical-view/07-task-model.md:201` uses `complete_in_future` as if its semantics were settled, but `09-open-questions.md Q8` marks the completion-discovery mechanism unresolved.
- **Edit sketch:**
  - File: `02-logical-view/07-task-model.md`
  - Location: §2.4.5 "Deferred Completion" paragraph
  - Delta: Prepend `[ASSUMPTION]` tag; add cross-reference to `09-open-questions.md#q8-deferred-completion-tracking`.
- **Trade-offs:** Gain: reader is warned at point of use. Give up: none.
- **Sanity test:** grep `complete_in_future` in target design; every mention is within 3 lines of an `[ASSUMPTION]` marker or a pointer to Q8.

#### A3-P12: `drain()` / `submit()` concurrency contract

- **Rationale:** G3, R4. `drain()` says it blocks until all Tasks are `RETIRED`, but the fate of a concurrent `submit()` (e.g., from a different binding thread during tenant shutdown) is unspecified. This is a silent correctness hole.
- **Edit sketch:**
  - File: `02-logical-view/09-interfaces.md`
  - Location: §2.6.1 behavioral contracts
  - Delta: Add "After `drain()` is entered, `submit()` returns `AdmissionRejected(DrainInProgress)`; multiple concurrent `drain()` calls are idempotent; `shutdown()` after `drain()` completes is guaranteed safe." Add `DrainInProgress` to `modules/error.md` Core domain.
- **Trade-offs:** Gain: deterministic shutdown. Give up: one error code + atomic flag read.
- **Sanity test:** Race test: call `drain()` and `submit()` 1000× concurrently; assert every post-drain `submit()` returns `DrainInProgress` and every prior-drain `submit()` reaches `RETIRED`.

#### A3-P13: Cross-node ordering assumptions

- **Rationale:** DS3, G3. Causal consistency is claimed (`04-process-view.md:561`) but the underlying transport ordering assumption is silent. Multi-path RDMA or IP ECMP can reorder messages; RDMA WRITE_WITH_IMM and two-sided receive may not share ordering.
- **Edit sketch:**
  - File: `04-process-view.md`
  - Location: §4.6.1 message table; §4.6.3 consistency guarantee
  - Delta: Add `[ASSUMPTION] Per-(source, destination, task-chain) FIFO delivery; idempotent delivery semantics at the transport layer; duplicate detection via TaskKey.generation.` Define runtime behavior if a `REMOTE_COMPLETE` arrives without a prior `REMOTE_SUBMIT` ACK: buffered in a reorder window of size `remote_reorder_window` (config, default 64), dropped with `ProtocolViolation` on overflow.
- **Trade-offs:** Gain: defensible causal guarantee. Give up: small reorder buffer in distributed/.
- **Sanity test:** Fault-injection test reorders `REMOTE_SUBMIT` / `REMOTE_COMPLETE` arrivals at the coordinator; assert correctness preserved up to `remote_reorder_window` and `ProtocolViolation` beyond.

#### A3-P14: `COMPLETING`-skip transition at leaf

- **Rationale:** LSP, V5. State machine is uniform per `04-process-view.md:229-231` but `04-process-view.md:384-391` explicitly states `COMPLETING` is not applicable at Core level. Either the diagram shows a skip edge or the uniformity is a lie.
- **Edit sketch:**
  - File: `04-process-view.md`
  - Location: §4.3.5 Worker state machine notes; §4.3.1 Task state diagram
  - Delta: Add an edge `EXECUTING → COMPLETED` labelled "leaf worker, no children" alongside the existing `EXECUTING → COMPLETING` edge; note in §4.3.5 table that AICore / leaf workers always take the skip edge.
- **Trade-offs:** Gain: LSP restored. Give up: one extra edge on the diagram.
- **Sanity test:** Assert in a Core-level conformance test that `Task.state` never equals `COMPLETING`; assert at Device/Host level it can equal `COMPLETING` for orchestration tasks.

#### A3-P15: Debug-mode `NONE`-dep-mode cross-check

- **Rationale:** G3, X6. `DepMode::NONE` is caller-asserted and runtime-silent on misuse. CI can catch this cheaply in debug builds by running a `DATA`-mode dry-run and comparing the installed external-edge set to the declared `NONE`.
- **Edit sketch:**
  - File: `02-logical-view/02-scheduler.md`
  - Location: §2.1.3.1.B DATA/BARRIER/NONE table
  - Delta: Add "In debug builds, `NONE` admission performs a dry-run `DATA` lookup against `producer_index`; any hit is logged `WARN NoneModeHiddenDependency producer=... consumer=... buffer=...` but not enforced." Reference in `07-cross-cutting-concerns.md §7.2`.
- **Trade-offs:** Gain: `NONE` misuse surfaces in CI. Give up: debug-only cost; one config knob.
- **Sanity test:** Intentionally mis-declare `NONE` on a RAW-dependent Submission under debug build; assert the WARN is emitted and (since debug mode also installs the edge) correctness holds.

## 5. Revisions of own proposals  (round ≥ 2 only)

Not applicable in round 1.

## 6. Votes on peer proposals  (round ≥ 2 only)

Not applicable in round 1.

## 7. Cross-aspect tensions

| pair | with_proposal | preferred_resolution |
|------|---------------|----------------------|
| A1 vs A3 | A3-P4 (`DEP_FAILED` event on producer failure) | Failure event fires only on the error cold path; zero cost on success — hot-path impact is `none`. A1 should accept; tag remains `extends-latency` only because the walk of fan-out under failure is O(successors). If A1 proposes a fast-path SoA `successors[]` array per Task slot, that also accelerates P4's walk. |
| A1 vs A3 | A3-P7 (empty / degenerate submission preconditions) | Validation runs once at admission in `submit()`, not per-task on the dispatch hot path. Fast path (pre-checked by frontend) can be gated by a `SubmissionDescriptor.flags & FE_VALIDATED` bit that skips repeat-validation in release builds; debug builds always validate. Two-tier path. |
| A1 vs A3 | A3-P8 (cyclic `intra_edges` DFS) | The DFS is O(V+E) on admission; for Single Submissions with `intra_edges.empty()` the check is a single size-zero branch. A1 should accept for small groups; for very large Groups (>1k tasks) a frontend-pre-validated flag (same FE_VALIDATED bit) can elide the check in release. Two-tier path. |
| A6 vs A3 | A3-P7 (C-API input validation) | Input validation at `bindings/` trust boundary (S3) is required anyway; A3-P7 aligns with A6's S3 requirement rather than conflicting. Combine: one validation step, satisfying both rubrics. |
| A9 vs A3 | A3-P1 (`ERROR` / `CANCELLED` state) | A9 may flag the extra state as YAGNI, but the state is already implicit in narrative — the proposal codifies existing behavior rather than inventing new machinery. `CANCELLED` can be deferred (A3 lists it as optional) if A9 objects. |
| A2 vs A3 | A3-P2 option (a) vs (b) | Option (a) (throw on REJECT) is simpler; option (b) (`expected<>`-style) is more extensible (E1/E4) and allows future `AdmissionDecision` values without ABI breakage. A2 likely prefers (b); A3 accepts either provided it is chosen and documented. Preferred: (b) with a `throw`ing convenience overload for callers who want exception semantics. |
| A5 vs A3 | A3-P4 + A3-P5 (failure propagation + sibling cancellation) | A5 will want retries on `DEP_FAILED`; A3 and A5 agree that error propagation should be deterministic first, retries layered above. Preferred resolution: `DEP_FAILED` carries the `ErrorCode`; A5's retry policy decides whether to resurrect the failed producer before `DEP_FAILED` fans out. |

## 8. Stress-attack on emerging consensus  (round 3+ only)

Not applicable in round 1.

## 9. Status

- **Satisfied with current design?** No.
- **Open items expected in next round:** A3-P1 (blocker — missing `ERROR` state), A3-P2 (high — `submit()` return contract), A3-P3 (high — admission failure scenarios), A3-P4 (high — producer failure → consumer propagation), A3-P5 (high — sibling cancellation policy), A3-P6 (high — requirement traceability), A3-P7 (medium — submission precondition validation), and tensions with A1 on P4/P7/P8 (two-tier path resolutions).
