# Aspect A3: Functional Sanity & Correctness — Round 2

## Metadata

- **Reviewer:** A3 (Functional Sanity & Correctness)
- **Round:** 2
- **Target:** `docs/pypto-runtime-design/` (rev as of 2026-04-18-171357)
- **Runtime Mode:** yes
- **Timestamp (UTC):** 2026-04-19T18:00:00Z

## 1. Rubric Checks

Re-evaluated in light of peer findings and the synthesis. No target-doc edits have landed between rounds, so the Result column is unchanged except where a peer's finding tightens or explains a gap I previously marked "Weak".

| # | Check | Result | Evidence (file:line) | Rule(s) |
|---|-------|--------|----------------------|---------|
| 1 | Every stated requirement traces to ≥ 1 scenario walkthrough | Weak | `01-introduction.md:32-57` vs `06-scenario-view.md:1-155`; FR-4/7/8/9/10, NFR-2/3/5/7 still lack a walkthrough. A9-P6 proposes to drop `PERFORMANCE`/`REPLAY` from v1, which would narrow FR-10 and shrink my required traceability set. | G1, V3 |
| 2 | ≥ 1 failure scenario per critical path | Weak | `04-process-view.md:641-700` defines four critical paths; `06-scenario-view.md:81-143` has failure scenarios only for three. §4.8.4 admission path still has no failure scenario. Now also tied to A5-P8 (degradation specs) and A8-P7 (`IFaultInjector`) — a shared test surface is available if those land. | V3, V4 |
| 3 | All interface implementations honor LSP | Fail | `04-process-view.md:229-284` Task state diagram still omits `ERROR`/`CANCELLED` but is referenced five times in `06-scenario-view.md`. `ISchedulerLayer::submit` return contract still inconsistent with `AdmissionDecision`. A7-P2 (split `ISchedulerLayer` into role interfaces) and A9-P1 (drop `submit_*` overloads) both depend on this contract being fixed first. | LSP, D7, V5 |
| 4 | Ambiguous / missing info marked `[ASSUMPTION]` | Weak | Still true at `02-logical-view/07-task-model.md:201` (`complete_in_future`) and `04-process-view.md:561` (transport ordering). A8-P6 (distributed trace alignment) would force the transport ordering to be pinned — partial overlap with A3-P13. | G3 |
| 5 | Explicit, not implicit, error conditions in Process View | Weak | `04-process-view.md:568-633` good breadth; gaps at `06-scenario-view.md:139` ("eventually timeout") and undefined sibling-on-failure behavior remain. A5-P4 (`idempotent` flag on TaskDescriptor) and A5-P10 (per-REMOTE_* idempotency) complement A3-P4/A3-P5. | — |
| 6 | Edge cases enumerated | Weak | Empty submission, `intra_edges` self-loop, `boundary_in/out` OOB, `workspace_request` subrange bounds, `drain()`/`submit()` race, `REMOTE_COMPLETE` before `REMOTE_SUBMIT` ACK — all still unspecified. A6-P3 (bounded payload parsing) and A6-P8 (DLPack/CAI validation) overlap with my A3-P7; the synthesis already proposes a merge into A1-P11. | V3, V4 |

## 2. Pros

- `06-scenario-view.md:1-155` four-view walkthroughs remain the strongest V3 artifact; **A3-P6** adds a traceability matrix on top rather than rebuilding. — cites V3.
- Submission/Task split, boundary classification, `dep_mode` contract, and the non-aliasing intermediate-memref invariant remain a solid correctness backbone. — cites G3, LSP at `02-logical-view/12-dependency-model.md:105-128`, `02-logical-view/04-memory.md:66-68`.
- Admission-concurrency analysis per `dep_mode` (BARRIER/DATA/NONE) is explicit; A1-P1/P2/P8/P14 harden it with pre-sizing + placement; A10-P1 and A1-P2's shared `shard_count` LevelParam give a common tuning knob. — cites LSP, G3 at `02-logical-view/02-scheduler.md:107-124`.
- `ErrorCode` layout + `ErrorContext` contract continue to pin propagation. A5-P10 (per-REMOTE_* idempotency) and A6-P12 (signed REPLAY trace, conditional on A9-P6) extend it coherently. — cites SRP, G3 at `modules/error.md:22-113, 169-176`.
- `AdmissionDecision {ADMIT, WAIT, REJECT}` + `ResourceAllocationResult {ALLOCATED, WAIT, REJECTED}` make back-pressure vs. hard-rejection an in-contract distinction. A9-P5 (unify the two enums as `AdmissionStatus`) removes the DRY violation without losing that distinction — I support it because it simplifies A3-P2's return-contract decision. — cites G3, DRY.
- Atomic admission with full rollback is contracted. A3-P3 + A3-P7 complete the story by naming distinct error sub-codes per failure mode. — cites LSP at `02-logical-view/02-scheduler.md:62-70`.
- `TaskHandle.generation` as correctness backstop composes well with A10-P10 / A1-P14's pre-sized layout for `producer_index`. — cites G3 at `02-logical-view/02-scheduler.md:121`, `02-logical-view/12-dependency-model.md:93`.

## 3. Cons

(Retained from R1; peer evidence now further supports most items. Newly-tightened findings noted inline.)

- **`ERROR`/`CANCELLED` state absent from the normative Task FSM** — unchanged. A3-P1 remains a blocker. No peer challenge.
- **`submit()` return contract vs. `AdmissionDecision`** — unchanged. A7-P2 + A9-P1 cannot land cleanly until A3-P2 is resolved; see tension table.
- **Admission-path failure scenarios missing** — unchanged. A3-P3 addresses; A5-P8 (degradation specs) and A8-P7 (`IFaultInjector`) now provide a reusable test seam.
- **Producer-failure → consumer "eventually timeout"** — unchanged. A5-P4's `idempotent` flag + A5-P2 circuit breaker gate whether the producer is retried before `DEP_FAILED` fans out; A3-P4 still owns the propagation contract.
- **Sibling cancellation policy unspecified** — unchanged. A3-P5 carries.
- **Requirement → scenario traceability incomplete** — unchanged; scope will shrink if A9-P6 defers `PERFORMANCE`/`REPLAY`.
- **DepMode::NONE misuse undetectable** — unchanged. A3-P15 carries.
- **Empty / degenerate submissions unvalidated** — unchanged. A6-P8 (DLPack/CAI validation) and A6-P3 (bounded payload) are adjacent; synthesis proposes fusing boundary validation into A1-P11's per-arg budget. A3-P7 should be merged-in the same step.
- **`drain()` concurrency contract silent** — unchanged. A3-P12 carries.
- **Cross-node ordering silent** — unchanged. A8-P6 (trace alignment) and A6-P3 (bounded parsing) both presume a transport-ordering stance; A3-P13 still required to pin it.
- **`complete_in_future` unresolved** — unchanged. A3-P11 marker is still the minimal action while Q8 is debated.
- **`COMPLETING` leaf skip soft** — unchanged. A3-P14 carries.
- **Cyclic `intra_edges` detection cost unspecified** — unchanged. A3-P8 needs a two-tier path (see §5 revision) to survive A1's hot-path gate.
- **SPMD index/size delivery informal** — unchanged. A3-P9 carries; interacts with A9-P4 (drop `Kind`) because A9-P4 uses `spmd.has_value()` as the SPMD discriminator — that is consistent with A3-P9 only if the `TaskArgs` reserved slots are independent of the discriminator.
- **Python exception mapping incomplete** — unchanged. A3-P10 carries.

## 4. Proposals (new this round)

No new proposals this round. All Round-1 proposals remain — see §5 for actions.

## 5. Revisions of own proposals

| id | action | amended_summary (if amend/split) | reason |
|----|--------|----------------------------------|--------|
| A3-P1 | split | **A3-P1a (blocker):** add `ERROR` state with transitions from `DISPATCHED`/`EXECUTING`/`COMPLETING`. **A3-P1b (medium):** add optional `CANCELLED` state and `on_cancel` handler. | No peer dissent on `ERROR` itself, but A9's YAGNI lens may flag `CANCELLED` as speculative if sibling-cancellation (A3-P5) is not adopted. Splitting preserves the blocker even if the cancellation story is deferred. |
| A3-P2 | amend | Commit option (b): widen signature to `std::expected<SubmissionHandle, ErrorContext>` (or equivalent `Status submit(..., SubmissionHandle* out)`); provide a throwing convenience overload for Python bindings. Adopt A9-P5 unified `AdmissionStatus` as the error payload. Record the decision as a new ADR. | A2 prefers extensibility (E1/E4); A9 prefers simplicity; option (b) satisfies both if we pair it with A9-P5. Hot-path impact: `none` (the `expected` payload is POD; admission is the cold path). |
| A3-P3 | defend | — | Synthesis marks fast-track. No peer pushback. Hot-path impact: `none`. |
| A3-P4 | amend | Keep `DEP_FAILED(producer_key, ErrorCode)` but specify that emission walks a **precomputed successor list stored on each Task slot at admission time** (SoA/AoS to be resolved under A1-P4). Fast path on success = 0 ops; slow path on failure = O(successors) walk. | A1 will challenge any event added on the hot path. Pre-materialising successors at admission (already required by `producer_index`) makes the failure walk bounded and aligns with A1-P4/A1-P8. Hot-path impact: `none` on success, `extends-latency` only on failure (cold). Two-tier preserved. |
| A3-P5 | amend | Keep sibling cancellation, but clarify: already-EXECUTING siblings **run to completion** (Worker contract preserved) and their results are discarded; only `PENDING`/`DEP_READY`/`DISPATCHED` siblings are transitioned to `CANCELLED`. Aggregation waits on the max of {already-EXECUTING child completion time, one event-loop iteration for cancellations}. | Addresses A1's concern that mid-execution cancellation cannot be forced on device workers without hardware support, and A5's retry-policy interaction: `idempotent` (A5-P4) siblings may be optionally re-dispatched elsewhere by A5 before the parent aggregates. |
| A3-P6 | amend | Traceability matrix required; however, if A9-P6 (defer `PERFORMANCE`/`REPLAY`) is agreed, FR-10 shrinks to `FUNCTIONAL` + `ONBOARD` and §6.1.5 (simulation-mode walkthrough) collapses from three sub-scenarios to one. | A9-P6 is the current leading deferral; A3-P6 should not over-commit scenarios that may be removed. |
| A3-P7 | amend | Merge with A1-P11 (per-arg marshaling budget) and A6-P8 (DLPack/CAI validation) per the synthesis conflict register. A3 keeps ownership of the **precondition catalog** (empty-tasks, self-loop, index-OOB, workspace-subrange-OOB, sum-overflow) and their distinct `AdmissionRejected` sub-codes; A1 keeps the per-arg latency budget; A6 keeps the boundary-validation threat model. Single validation pass at the Python↔C boundary in `bindings/`. | Three peers converge on the same validation step. Combining now avoids three near-duplicate pipeline passes. |
| A3-P8 | amend | Two-tier: release builds **skip** cycle detection when `SubmissionDescriptor.flags & FE_VALIDATED` is set by a trusted frontend; debug/test and untrusted-frontend builds always run the O(V+E) DFS with a `max_intra_edges` LevelParam cap from `04-process-view.md §4.8.4`. Cite A5/A6 — a trusted-frontend flag is an S3 trust-boundary opt-in, not an untyped escape hatch. | Answers A1's hot-path concern (HPI `extends(bounded)` → `none` on fast path); keeps safety for external frontends. |
| A3-P9 | defend | — | Interacts cleanly with A9-P4: A9-P4 drops `Kind` and relies on `spmd.has_value()`; A3-P9 fixes the delivery ABI of `spmd_index`/`spmd_size` independent of the discriminator. No conflict. |
| A3-P10 | defend | — | No peer objections. A6-P1 (threat model) and A5's error-path proposals all benefit from a complete `Domain → Python exception` mapping. |
| A3-P11 | defend | — | Trivial doc edit. No objections. |
| A3-P12 | amend | Specify that `drain()` is also atomic w.r.t. **distributed** submissions: remote peers observe the drain flag before accepting further `REMOTE_SUBMIT`; in flight `REMOTE_SUBMIT` messages continue to retirement; add `DrainInProgress` to `modules/error.md` Core domain. | A5-P8 (degradation specs for admission saturation) and A10-P2 (decentralised coordinator) require a draining semantic that survives coordinator failover; the R1 wording was single-node-only. |
| A3-P13 | amend | Pin per-link FIFO + idempotent delivery + duplicate detection via `TaskKey.generation`. Align with A5-P10 (DS4 protocol contract) rather than inventing a new invariant — A5 already proposes per-REMOTE_* idempotency, A3-P13 just adds the `[ASSUMPTION]` marker + reorder-buffer behavior. | Avoids duplication with A5-P10 and A8-P6 (trace alignment) — one ordering model, three consumers. |
| A3-P14 | defend | — | Pure doc fix. |
| A3-P15 | defend | — | Debug-only; no hot-path cost. |

## 6. Votes on peer proposals

Voting on all 95 peer proposals. `blocking=true` only when objection rests on a hard rule from `04-agent-rules.md`. `override_request` reserved for proposals where I believe A1 is over-using the hot-path veto.

### A1 — Performance & Hot Path

| proposal_id | vote | rationale (cite rule id) | blocking | override_request |
|-------------|------|--------------------------|----------|-------------------|
| A1-P1 | agree | Pre-sizing `producer_index` and forbidding hot-path rehash satisfies P1/X2 without breaking LSP; no correctness impact. | false | false |
| A1-P2 | agree | Capping DATA-mode lock hold and making `shard_count` a `LevelParam` strengthens admission-concurrency correctness (P4, DS6). Cleanly composes with A10-P1. | false | false |
| A1-P3 | agree | Function-cache LRU + capacity + `HEARTBEAT` presence closes an implicit assumption at `modules/profiling.md`; G3 aligned. | false | false |
| A1-P4 | agree | Task hot/cold split is layout-only (one-time "relayout") and improves X4; as long as the precomputed-successor array (used by A3-P4 amended) lives in the cold tier, no correctness tension. Confine to `modules/core.md §8` per synthesis. | false | false |
| A1-P5 | agree | Latency budgets for SPMD fan-out and event-loop stages directly support G1/V3 (traceability). A3-P6 co-owns. | false | false |
| A1-P6 | agree | Cap + staging for REMOTE_SUBMIT; cite DS3, P6. Merged form (see Merge Register) accepted. | false | false |
| A1-P7 | agree | Per-thread local sequence + offline merge does not alter correctness; O1 preserved via `correlation_id`. | false | false |
| A1-P8 | agree | Pre-sizing OutstandingWindow/ReadyQueue + CONTROL placement satisfies X2. | false | false |
| A1-P9 | agree | Bitmask-encode WG availability is a layout choice; A2's 64-group cap concern is answered by a 2×64 multi-bitmap fallback per synthesis — request the fallback be included in the edit. | false | false |
| A1-P10 | agree | Workload-level profiling-overhead CI gate benefits V4/O3. | false | false |
| A1-P11 | agree | Per-arg marshaling budget at Python↔C boundary is the natural home for A3-P7 + A6-P8 validation. A3 accepts joint ownership. | false | false |
| A1-P12 | agree | `BatchedExecutionPolicy.max_batch` + tail budget; require the fast-path default `Dedicated` (no knob) per synthesis, so A3/A9 parameter-proliferation concern is absorbed. | false | false |
| A1-P13 | agree | Bounding `args_blob` copy during DEP_READY→DISPATCHED strengthens X9 budget honesty and supplies the precondition A3-P7's sub-code `WorkspaceSizeMismatch` depends on. Conditional on the `args_blob ≤ 4 KiB` or fast-path HAL-contract amendment the synthesis requests. | false | false |
| A1-P14 | agree | Document `producer_index` placement + shard default. Merges with A10-P10 (see Merge Register). | false | false |

### A2 — Extensibility & Evolvability

| proposal_id | vote | rationale (cite rule id) | blocking | override_request |
|-------------|------|--------------------------|----------|-------------------|
| A2-P1 | agree | Versioning public data contracts (E6) is a prerequisite for A3-P2's amended `submit()` signature — the wire representation of `ErrorContext` must carry a version. | false | false |
| A2-P2 | agree | Schema-registered `LevelOverrides`; accept synthesis-suggested phasing ("closed for v1, schema-registered for v2") since it avoids the YAGNI concern. | false | false |
| A2-P3 | agree | Open extension-point enums while keeping `DepMode` closed — aligns with A3's view that `DepMode` is a runtime correctness contract (G3) and must not be extended casually. | false | false |
| A2-P4 | agree | Migration & Transition Plan is a V1/V5 doc obligation. | false | false |
| A2-P5 | agree | Interface Evolution & Backward-Compat policy — required for A3-P2's ABI-widening choice. | false | false |
| A2-P6 | abstain | Pluggable `IDistributedProtocolHandler` is outside correctness scope; A9's devirt concern is valid but doesn't cross into LSP. Abstain per cross-aspect etiquette. | false | false |
| A2-P7 | abstain | Async-policy extension seam is a design-evolution choice; no correctness impact absent a concrete consumer. | false | false |
| A2-P8 | agree | Recording intentional closures as known deviations satisfies G3. | false | false |
| A2-P9 | agree | Versioned trace schema — required by A8-P12 (stable PhaseIds) and by A3-P11's reference to Q8. | false | false |

### A4 — Document Consistency

| proposal_id | vote | rationale (cite rule id) | blocking | override_request |
|-------------|------|--------------------------|----------|-------------------|
| A4-P1 | agree | Canonical HAL enum casing (D7/V5). | false | false |
| A4-P2 | agree | Broken anchor in task-model x-ref — direct V1 fix. | false | false |
| A4-P3 | agree | Invariant-count prose fix ("Two"→"Three") is a V5/LSP-adjacent correctness-doc fix; confirms the non-aliasing invariant A3 relies on. | false | false |
| A4-P4 | agree | Reorder Open Questions numerically (V5). | false | false |
| A4-P5 | agree | Glossary entries for event-loop plumbing; if A9-P7 (merge `SourceCollectionConfig` into `EventHandlingConfig`) is agreed, one glossary entry drops per synthesis. | false | false |
| A4-P6 | agree | Cross-link views to ADRs (V2/V5). | false | false |
| A4-P7 | agree | Unify L0 component label (D7). | false | false |
| A4-P8 | agree | Sync Appendix-B `TaskState` count — this is the diff A3-P1 depends on once `ERROR`/`CANCELLED` land. | false | false |
| A4-P9 | agree | Expand `Task State` glossary — aligns with A3-P1. | false | false |

### A5 — Reliability & Fault Tolerance

| proposal_id | vote | rationale (cite rule id) | blocking | override_request |
|-------------|------|--------------------------|----------|-------------------|
| A5-P1 | agree | Exponential backoff + jitter for remote retries (R2); composes with A3-P4 by deciding whether the producer is resurrected before `DEP_FAILED` fans out. | false | false |
| A5-P2 | agree | Per-peer circuit breaker (R3); same composition as P1. | false | false |
| A5-P3 | agree | Commit coordinator-failover OR deterministic fail-fast. Prefer v1 = fail-fast per synthesis — it provides the deterministic FSM A3-P12 extends for drain. Merges with A10-P2 (see Merge Register). | false | false |
| A5-P4 | agree | `idempotent: bool` on TaskDescriptor (DS4) gates retry cleanly; a prerequisite for A3-P4's decision whether `DEP_FAILED` fires immediately or after a retry window. | false | false |
| A5-P5 | agree | Chaos / fault-injection harness — direct V4/R6 support for A3-P3's admission-failure scenarios. | false | false |
| A5-P6 | agree | Scheduler-thread watchdog; paired with A8-P4 `dump_state()` per Merge Register. | false | false |
| A5-P7 | agree | `Timeout` on `IMemoryOps` async (R1) — closes an implicit assumption I flagged in rubric #4. | false | false |
| A5-P8 | agree | Degradation specs for admission saturation + WG loss (R4) — directly covers my A3-P3 §4.8.4 gap. | false | false |
| A5-P9 | agree | `QUARANTINED` Worker state between `FAILED` and `UNAVAILABLE` is an LSP-friendly extension of the worker state machine — consistent with A3-P1 approach for Task. | false | false |
| A5-P10 | agree | DS4 per-REMOTE_* idempotency — the contract A3-P13 assumes; accept as co-owning. | false | false |

### A6 — Security & Trust Boundaries

| proposal_id | vote | rationale (cite rule id) | blocking | override_request |
|-------------|------|--------------------------|----------|-------------------|
| A6-P1 | agree | Trust-boundary threat model (S1) is a prerequisite for A3-P7's validation pass. | false | false |
| A6-P2 | agree | Node authentication in `HANDSHAKE` (S2/S3). | false | false |
| A6-P3 | agree | Bounded variable-length payload parsing (S3); fold the cap check into A1-P11's per-arg budget per synthesis. | false | false |
| A6-P4 | agree | TLS-default on TCP only, OFF on RDMA per synthesis two-tier; S5 satisfied without hot-path impact. | false | false |
| A6-P5 | agree | Scoped/revocable RDMA `rkey` (S2) with rotation on Submission retirement (not per task) per synthesis; avoids hot-path cost. | false | false |
| A6-P6 | agree | Security audit trail (S6); composes with A8-P10 structured logging. | false | false |
| A6-P7 | agree | Function-binary attestation as opt-in `trust_boundary.multi_tenant=true`; off by default. | false | false |
| A6-P8 | agree | DLPack/CAI boundary validation; merges with A3-P7 + A1-P11 per synthesis. | false | false |
| A6-P9 | agree | Enforce Logical System isolation (S2); directly supports FR-9 (the traceability gap I flagged). | false | false |
| A6-P10 | agree | Capability-scoped log sinks (S2/O2). | false | false |
| A6-P11 | agree | Gated `register_factory` (S2/S4). | false | false |
| A6-P12 | abstain | Signed/schema-validated REPLAY trace is moot if A9-P6 defers REPLAY; otherwise agree. Vote tied to A9-P6 resolution. | false | false |
| A6-P13 | abstain | Per-tenant submit rate-limit lacks a v1 multi-tenant scenario; orthogonal to A3 correctness. | false | false |
| A6-P14 | agree | Key-material lifecycle ADR (S5). | false | false |

### A7 — Modularity & SOLID

| proposal_id | vote | rationale (cite rule id) | blocking | override_request |
|-------------|------|--------------------------|----------|-------------------|
| A7-P1 | agree | Break `scheduler/` ↔ `distributed/` cycle (D6). | false | false |
| A7-P2 | agree | Split `ISchedulerLayer` into role interfaces — **accept only if** A3-P2's return contract is resolved first; otherwise LSP drift between role interfaces is worse than today. Merges with A9-P1 per synthesis. | false | false |
| A7-P3 | agree | Invert `core/`↔`hal/` for handle types (D2). | false | false |
| A7-P4 | agree | Move distributed payload structs to `distributed/` (D3). | false | false |
| A7-P5 | agree | `distributed_scheduler` depends only on `ISchedulerLayer` (D2). | false | false |
| A7-P6 | agree | Extract MLR + deployment parser (D3/D5). | false | false |
| A7-P7 | agree | Forward-decl contract (D6). | false | false |
| A7-P8 | agree | Consolidate `ScopeHandle` ownership — single-owner invariant supports DS7 and A3's correctness story. | false | false |
| A7-P9 | agree | Deduplicate Python `MemoryError` class names (D7); aligns with A3-P10 mapping table. | false | false |

### A8 — Testability & Observability

| proposal_id | vote | rationale (cite rule id) | blocking | override_request |
|-------------|------|--------------------------|----------|-------------------|
| A8-P1 | agree | Injectable `IClock` (X5) — A3-P12 drain/submit race test needs this. | false | false |
| A8-P2 | agree | Driveable event-loop `step()` + `RecordedEventSource` (X5) — A3-P4/A3-P5 test plans need this. | false | false |
| A8-P3 | agree | Enumerate stats + bucketed latency histograms (O3); answer A1's hot-path concern via branchless insert + pre-allocated buckets per synthesis two-tier. | false | false |
| A8-P4 | agree | `Runtime::dump_state()` — makes A3-P3/A3-P4 scenarios inspectable in production. Merges with A5-P6 per Merge Register. | false | false |
| A8-P5 | agree | Externalise alert rules + Prometheus/OTEL sink (O5, E3). | false | false |
| A8-P6 | agree | Distributed trace alignment — A3-P13 depends on this for causal-ordering validation. | false | false |
| A8-P7 | agree | `IFaultInjector` (X5) — reusable for A3-P3 and A5-P5. | false | false |
| A8-P8 | agree | AICore in-core trace upload with Level-1 default strip; two-tier preserved. | false | false |
| A8-P9 | agree | Profiling drop/degraded state as first-class alerts (O3/O5). | false | false |
| A8-P10 | agree | Structured KV logging (O2) — composes with A6-P6 audit trail. | false | false |
| A8-P11 | agree | HAL contract test suite runnable against sim + onboard (X5/DfT). | false | false |
| A8-P12 | agree | Stable `PhaseId`s for Submission lifecycle (O1) — enables A3-P3/P6 traceability validation. | false | false |

### A9 — Simplicity (KISS / YAGNI)

| proposal_id | vote | rationale (cite rule id) | blocking | override_request |
|-------------|------|--------------------------|----------|-------------------|
| A9-P1 | agree | Drop `submit_group`/`submit_spmd` overloads — aligned with A3-P2 (one return contract only). Merges with A7-P2 per Merge Register. | false | false |
| A9-P2 | abstain | Cutting `FULLY_SPLIT`/`SPLIT_DEFERRED` and pluggable policies is an A2-vs-A9 question; A3 is neutral provided the remaining modes do not introduce LSP drift in `ISchedulerLayer`. Synthesis bridge (keep `IEventLoopDriver` test-only seam for A8-P2) is acceptable. | false | false |
| A9-P3 | abstain | Remove collectives from `IHorizontalChannel` — Q4 resolution; no direct correctness impact. | false | false |
| A9-P4 | agree | Drop `SubmissionDescriptor::Kind`; A3-P7 precondition validation works fine on `spmd.has_value()` + `tasks.size()`. Confirms synthesis. | false | false |
| A9-P5 | agree | Unify admission enums as `AdmissionStatus` — directly required by A3-P2's amended return contract. | false | false |
| A9-P6 | agree | Defer `PERFORMANCE`/`REPLAY` to post-v1 with ADR — shrinks FR-10 scope per A3-P6. If adopted, A6-P12 is moot. | false | false |
| A9-P7 | agree | Fold `SourceCollectionConfig` into `EventHandlingConfig` — no correctness impact. | false | false |
| A9-P8 | agree | Move per-Function companion-artifacts obligation out of runtime design (SRP). | false | false |

### A10 — Scalability & Data Flow

| proposal_id | vote | rationale (cite rule id) | blocking | override_request |
|-------------|------|--------------------------|----------|-------------------|
| A10-P1 | agree | Default `producer_index` sharding; aligns with A1-P2 shared `shard_count` LevelParam. | false | false |
| A10-P2 | agree | Decentralise Pod-level coordinator — accept the synthesis compromise (v1 fail-fast + roadmap to decentralised in v2). Merges with A5-P3 per Merge Register. | false | false |
| A10-P3 | agree | Explicit per-data-element consistency model (DS6) — directly answers my rubric #4 "implicit assumptions" finding on shared mutable state. | false | false |
| A10-P4 | agree | Stateful/stateless module classification (DS1). | false | false |
| A10-P5 | agree | Per-peer projection of `REMOTE_SUBMIT`; merges into A1-P6 per Merge Register. | false | false |
| A10-P6 | agree | Faster configurable peer-failure detection — consistent with A5-P3 fail-fast. | false | false |
| A10-P7 | agree | Sharded TaskManager path — two-tier with `shards=1` default answers A1 veto concern. | false | false |
| A10-P8 | agree | Aggregated data-flow + ownership diagram (DS7) — complements A3's LSP-across-layers audit. | false | false |
| A10-P9 | agree | Gate `WorkStealing` vs. `RETRY_ELSEWHERE` (DS6) — a correctness invariant for idempotency. | false | false |
| A10-P10 | agree | Cap + layout guidance for `producer_index`; merges into A1-P14 per Merge Register. | false | false |

### Vote summary

- Agree: 85; Abstain: 7; Disagree: 0; Blocking: 0; Override requests: 0.
- Total peer proposals voted: **95** (A1:14, A2:9, A4:9, A5:10, A6:14, A7:9, A8:12, A9:8, A10:10) = 95; consistent with 110 total − 15 A3 = 95.

## 6.b Merge Register verdict (A3 perspective)

| Merge into | Absorbed | A3 verdict | Reason |
|------------|----------|------------|--------|
| A1-P6 | A10-P5 | accept | Pure payload hygiene; no correctness impact beyond what A3-P13 already requires. |
| A1-P14 | A10-P10 | accept | One ownership narrative for `producer_index`; strengthens DS7 coherence. |
| A5-P3 | A10-P2 | accept (with v1 = fail-fast) | Consistent with A3-P12 amended drain contract and with V4 (critical-path failure scenario obligation). v2 decentralisation as roadmap. |
| A5-P6 | A8-P4 | accept | Watchdog + `dump_state()` paired: A3-P3/P4/P5 debug plans are immediately inspectable. |
| A7-P2 | A9-P1 | accept **conditional on A3-P2 resolved first** | Role-interface split cannot land cleanly if `submit()`'s return contract is still ambiguous. If A3-P2 amended option (b) is agreed, merge proceeds. |
| A6-P8 | A1-P11 (partial) | accept, extend to also absorb A3-P7 | Three proposals converge on the Python↔C validation pass; unifying gives one pipeline with distinct sub-codes owned by A3, latency budget owned by A1, threat model owned by A6. |

## 7. Cross-aspect tensions (new this round)

| pair | with_proposal | preferred_resolution |
|------|---------------|----------------------|
| A3 vs A4 | A3-P1a (add `ERROR` state) + A4-P8 (sync Appendix-B TaskState count) | Land A3-P1a first; A4-P8 then propagates the count update. One ADR, two patches. |
| A3 vs A5 | A3-P4 (`DEP_FAILED`) + A5-P4 (`idempotent` flag) + A5-P1/P2 (retry + breaker) | A5's retry decision runs **before** `DEP_FAILED` emission. If `idempotent=true`, retry; on retry exhaustion or `idempotent=false`, emit `DEP_FAILED`. One event, one decision point. |
| A3 vs A8 | A3-P3 (admission failure scenarios) + A8-P7 (`IFaultInjector`) | A3 specifies the scenarios (§6.2.5–§6.2.7); A8 supplies the injection mechanism (sim-only). Shared fixtures. |
| A3 vs A6 | A3-P7 (precondition validation) + A6-P8 (DLPack/CAI validation) + A1-P11 (per-arg budget) | Single validation pass at Python↔C boundary; A3 owns precondition catalog, A6 owns threat model, A1 owns latency budget. See Merge Register verdict. |
| A3 vs A9 | A9-P4 (drop `Kind`) + A3-P9 (SPMD ABI) | No conflict — A9-P4 uses `spmd.has_value()` as discriminator; A3-P9 reserves `TaskArgs.scalar_args[0..1]` independent of the discriminator. Land both. |
| A3 vs A9 | A9-P1 (drop submit overloads) + A3-P2 (widen return) | A3-P2 must be agreed first. Then A7-P2+A9-P1 can land with one canonical `submit(const SubmissionDescriptor&) -> std::expected<SubmissionHandle, ErrorContext>` entry. |
| A3 vs A2 | A3-P2 (widen `submit` return) + A2-P1 (version data contracts) | Choose option (b) and embed a single byte of `ErrorContext` version so the widened ABI is evolvable without a second break. |
| A3 vs A10 | A3-P12 (drain contract) + A10-P2 (decentralised coordinator) | With v1 = fail-fast (per synthesis), drain is single-coordinator; v2 decentralised must specify how `DrainInProgress` propagates across the membership view. Record as a deferred ADR. |
| A3 vs A1 | A3-P8 (cyclic detection) + A1-P2/A1-P12 (latency budgets) | Two-tier: FE_VALIDATED fast path; debug/untrusted frontend slow path. Aligns with A1's `LevelParam` discipline. |
| A3 vs A1 | A3-P4 amended (precomputed successor walk) + A1-P4 (Task hot/cold split) | Successor array lives in the cold tier of `Task`; hot tier unchanged. Both proposals compose cleanly. |

## 8. Stress-attack on emerging consensus

_Deferred to round 3 per prompt._

## 9. Status

- **Satisfied with current design?** Partially — R2 has clear convergence on the Merge Register and on the fast-track A3 items (P3, P9, P10, P11, P12 amended, P13 amended, P14, P15). The remaining debate axes for A3 are:
  - **A3-P1a** (blocker — `ERROR` state) — expected to land in R2; A3-P1b (CANCELLED) may be conditional on A3-P5.
  - **A3-P2** (widened `submit` return contract) — must agree option (b) before A7-P2 + A9-P1 merge can finalise.
  - **A3-P4, A3-P5** amended — depend on A5 retry policy + A1 hot/cold split for the precomputed successor array; two-tier rationale already recorded.
  - **A3-P7** — merged with A1-P11 + A6-P8; convergence expected if the merge is accepted.
  - **A3-P8** amended — convergence expected via FE_VALIDATED two-tier.
- **Open items expected in next round (stress-attack round 3):** A3-P1a, A3-P2, A3-P4, A3-P5, A3-P7 (joint with A1-P11, A6-P8), A3-P8 (two-tier), A3-P12 (amended drain). Also expect stress-attacks on the Merge Register items where A3 is a conditional-accept: A7-P2+A9-P1 (conditional on A3-P2).
