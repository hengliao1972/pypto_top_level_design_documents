# Aspect A9: Simplicity (KISS / YAGNI) — Round 2

## Metadata

- **Reviewer:** A9
- **Round:** 2
- **Target:** `/data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/`
- **Runtime Mode:** yes
- **Timestamp (UTC):** 2026-04-19

## 1. Rubric Checks

These remain substantively unchanged from Round 1; brief restatement in light of peer feedback:

| # | Check | Result | Evidence (file:line) | Rule(s) |
|---|-------|--------|----------------------|---------|
| 1 | Every abstraction earns its keep | Weak → **Weak (unchanged)** | `02-logical-view/05-machine-level-registry.md:15-30` (factory inflation); A7-P2 reinforces that the interface *split* should come with a *collapse* of the overload set — i.e., my P1 is compatible with A7-P2. | G2, G4 |
| 2 | Speculative future-proofing absent | Fail → **Fail (unchanged)** | `09-interfaces.md:436-442` (4 deployment modes); `08-design-decisions.md:455-461` (3 sim modes); `05-machine-level-registry.md:144-145` (collectives before Q4). A2's proposals (P6, P7) *extend* the speculative surface; A6-P7, A6-P12, A6-P13 add more. | G2 (YAGNI) |
| 3 | Clarity over cleverness | Weak → **Weak (unchanged)** | 3-stage A/B/C loop × 4 deployment modes × pluggable policies. A8-P2 (driveable loop) is compatible with a *single test seam* rather than the current pluggable-policy machinery. | G4 |
| 4 | No single-implementation wrappers | Weak → **Weak (unchanged)** | Reinforced by peers: A2-P6 (single-backend pluggable protocol handler) and A2-P7 (async-policy seam with no named extender) are textbook "interfaces with one implementation and no planned variability." | G2 |
| 5 | DRY on knowledge | Weak → **Weak (unchanged)** | Admission-enum duplication persists. A5-P9 (`QUARANTINED` state between FAILED/UNAVAILABLE) is a new DRY-on-knowledge violation introduced in R1. | G2, DRY |

## 2. Pros

(Pros from Round 1 stand unchanged. New pros observed while reading the peer reviews:)

- **Several peer proposals reinforce A9's direction.** A7-P2 (split `ISchedulerLayer` into role interfaces) pairs cleanly with A9-P1; the combined edit is net-simpler than either in isolation. A1's pre-sizing / placement proposals (A1-P1, A1-P8) remove allocations on the hot path *without* adding abstraction — model KISS + performance fusion.
- **The conflict register's bridge patterns are simplicity-friendly.** The synthesiser's "two-tier path (fast default / slow opt-in)" formulation for A1-P12, A6-P4, A10-P7 keeps the fast path lean and contains optional complexity under a single switch — aligns with G4 (clarity over cleverness).
- **A5-P4 (`idempotent: bool` flag on `TaskDescriptor`)** is a one-bit change that unlocks safe retry; minimal surface area for maximal reliability gain. KISS-aligned (G2) and DS4-aligned.

## 3. Cons

(Cons from Round 1 stand unchanged. New cons observed from the peer-review package:)

- **A2's proposal set, taken as a whole, re-opens the very surface A9 is trying to close.** A2-P1 (version every contract), A2-P2 (schema-registered `LevelOverrides`), A2-P6 (pluggable `IDistributedProtocolHandler`), A2-P7 (async-policy extension seam) each add a versioning/registry/interface layer that has no second implementation in v1 and no named future extender. Cite G2 (YAGNI) and §2.7 YAGNI.
- **A6's `PERFORMANCE`/`REPLAY` dependency (A6-P12) only exists because the `SIM` modes it defends exist (A9-P6).** If A9-P6 ships, A6-P12 falls away for free. Cite G2.
- **A5-P9 (`QUARANTINED` Worker state) is FSM inflation.** `FAILED` and `UNAVAILABLE` already bracket the transition; `QUARANTINED` is a time-windowed sub-state of `UNAVAILABLE` expressible with a timestamp field. Cite G2, DRY.
- **A8-P5 (externalize alert rules + Prom/OTEL sink) ships two systems at once** (rule format *and* a wire sink) for a v1 that has no declared operator story. The synthesiser's amendment (alert-rule *file location* only, OTEL sink as known-deviation) is the KISS form. Cite G2.

## 4. Proposals

No new A9 proposals this round. Revisions of existing proposals are in §5.

## 5. Revisions of own proposals

| id | action | amended_summary (if amend/split) | reason |
|----|--------|----------------------------------|--------|
| A9-P1 | **amend** | Combined with A7-P2 into one proposal: **"Single `submit(SubmissionDescriptor)` entry point on a role-split `ISchedulerLayer`"** — `ISchedulerLayer` decomposes into three role interfaces (`ISubmissionSink`, `ITaskLifecycleManager`, `ISchedulerQuery`), and the submission role carries exactly one virtual method `submit(SubmissionDescriptor)`. The `submit_group`/`submit_spmd`/`submit(TaskDescriptor)` overloads become free-function builders living alongside the submission role in `runtime/`. | The synthesiser proposed this merge in the Merge Register; A7 and A9 are orthogonal (ISP split vs overload collapse). Combined, the two edits reduce the vtable from 4+ entry points to one, while giving A7 the role-focused interfaces it wants. Net result is strictly simpler than either in isolation. Cite G2, ISP, D3. |
| A9-P2 | **amend** | Retain the core simplifications (cut `FULLY_SPLIT` + `SPLIT_DEFERRED`; collapse `IEventCollectionPolicy`/`IExecutionPolicy` to closed enums + config). **New amendment:** keep a *single* `IEventLoopDriver` interface as a test-only seam satisfying A8-P2 (driveable event loop). `IEventLoopDriver` has one production implementation (the real loop) and one test implementation (`RecordedEventSource` + `step()`); it is not a pluggable policy registry. | A8's driveable-loop requirement is legitimate (X5 — injectable test seams); a single test double satisfies it without reintroducing the pluggable-policy surface. A2's extensibility objection is addressed by documenting the future extension interfaces in an appendix as "extension points available when a second implementation exists" (deferred, not deleted). Cite G2, X5, E4 (interfaces introduced *when* needed). |
| A9-P3 | **defend** | (unchanged) | A9-P3 defense strengthened by synthesiser amendment: add an ADR committing to "collectives as Orchestration Functions in v1, composed from `transfer()`; hardware-native collective path re-enters as a separate `ICollectiveOps` extension interface when HCCL perf data demands it." This closes Q4 with the simpler option and keeps the door open via a *new* extension interface — cleaner than the current fat `IHorizontalChannel`. No peer produced a concrete counter-argument; every non-trivial channel backend still has to stub the current `barrier`/`all_reduce` for no runtime consumer. Cite G2, E4. |
| A9-P4 | **defend** | (unchanged) | A3 (the only cited potential dissenter in the synthesis) has no proposal that *requires* `Kind`; A3-P7 (submission precondition validation) can read `spmd.has_value()` and `tasks.size()` — same information. A1's admission path drops one branch. No remaining objection. Cite G2, DRY. |
| A9-P5 | **defend** | (unchanged) | Low-controversy fast-track per synthesis. No peer objected. Cite G2, DRY. |
| A9-P6 | **amend** | Ship v1 with `FUNCTIONAL` simulation only. **New amendment:** add a dedicated ADR ("ADR-011-R2") formally deferring `PERFORMANCE` and `REPLAY` to v2, with the explicit trigger conditions: `PERFORMANCE` returns when a concrete perf-modeling consumer (CI perf-gate workload) is named; `REPLAY` returns when a post-mortem debugging customer is named. Move both to `09-open-questions.md` as follow-up questions with these triggers. | Synthesiser suggests exact amendment. A8's debugging-replay concern is addressed by the deferral-with-trigger pattern (not a deletion). A6-P12 (signed REPLAY trace) automatically becomes moot. Cite G2, E5 (incremental migration — don't build two sim pipelines ahead of workloads). |
| A9-P7 | **amend** | Fold `SourceCollectionConfig` into `EventHandlingConfig`. **New amendment:** simultaneously coordinate with A4-P5 — the glossary update drops the `SourceCollectionConfig` entry and the `EventHandlingConfig` entry is expanded to list the absorbed fields; no glossary churn beyond one entry collapsed into another. | Synthesiser flagged the A4 coordination; addressing it up-front avoids a conflict. Cite G2 (config surface collapse), D7 (consistent naming — one struct, one glossary entry). |
| A9-P8 | **defend** | (unchanged) | Low-controversy fast-track per synthesis. No peer objected. The artifact-pipeline scope creep persists in ADR-011 and should move to the frontend doc. Cite G2, SRP (D3). |

## 6. Votes on peer proposals

Vote abbreviations: **agree** / **disagree** / **abstain**. `blocking=true` reserved for objections grounded in a hard rule; A9's primary rules (G2, G4) are design-discipline soft rules, so A9 files no blocking objections this round. `override_request=false` throughout — A9 does not seek to override any A1 hot-path veto.

### A1 — Performance

| proposal_id | vote | rationale (cite rule id) | blocking | override_request |
|-------------|------|--------------------------|----------|-------------------|
| A1-P1 | agree | Pre-sizing `producer_index` removes hot-path allocation (X2) with one startup constant; KISS-aligned (G2). A fixed cap is simpler than a resizable map. | false | false |
| A1-P2 | agree | Capping DATA-mode lock hold time and making shard count a `LevelParam` bounds worst-case tail latency (X2, X9) without introducing new abstractions. | false | false |
| A1-P3 | agree | LRU + capacity + HEARTBEAT presence on the Function Cache makes the ADR-007 policy concrete; no new surface (P2). | false | false |
| A1-P4 | **disagree** | HPI=`relayouts`; hot/cold field split *prescribed* in the Logical View before any benchmark. G2 / P3 (no premature optimization). Acceptable only if the prescription lives in `modules/core.md §8` (as the synthesiser suggests) and the Logical View cites the module doc without enumerating fields. Amend and I switch to agree. | false | false |
| A1-P5 | agree | Latency budgets decompose hot paths into measurable targets (X9, P5); documentation, not new machinery. | false | false |
| A1-P6 | agree | Payload hygiene merges cleanly with A10-P5; reduces wire bytes for the common case. KISS-friendly; merger also means one proposal instead of two. | false | false |
| A1-P7 | agree | Per-thread local sequence + offline merge is a well-understood pattern (G4 clarity); removes hot-path contention. | false | false |
| A1-P8 | agree | Pre-size + placement parallels A1-P1; same KISS rationale. | false | false |
| A1-P9 | **disagree** | Premature layout (P3) for a ≤64-group cap with no workload evidence. Either the cap is speculative and we're not there yet (G2), or the cap is real and the extension path (2×64 bitmap) needs to be specified up front. The current proposal does neither. Prefer: defer until a workload demonstrates the hot-path win. | false | false |
| A1-P10 | agree | Workload-level profiling-overhead CI gate is low-cost and caught-early; aligns with X5/O3. | false | false |
| A1-P11 | agree | Per-arg marshaling budget is a single doc constraint at the Python↔C boundary; also absorbs A6-P8 per merge register — simpler overall. | false | false |
| A1-P12 | **disagree** | Adds `max_batch` tunable + tail budget. Synthesiser's suggested amendment ("fast-path default=Dedicated; `max_batch` applies only when `Batched` explicitly selected") would flip me to agree — this is the two-tier pattern the runtime-mode guidance expects. As currently written, it is tunable proliferation (G2). Amend and I agree. | false | false |
| A1-P13 | agree | Tightening the HAL contract around `args_blob ≤ 4 KiB` (per synth suggested amendment) removes a hot-path copy and is a net-simpler HAL invariant (G2, X2). | false | false |
| A1-P14 | agree | Document-only placement guidance for `producer_index`; co-owned with A10-P10 (accepted merge). | false | false |

### A2 — Extensibility

| proposal_id | vote | rationale (cite rule id) | blocking | override_request |
|-------------|------|--------------------------|----------|-------------------|
| A2-P1 | **disagree** | "Version every public data contract" before any contract has evolved is YAGNI (G2). A single discipline ("add a version byte when you evolve a header") captured in A2-P5's evolution policy is sufficient; pre-declaring version fields in every header adds bytes/ceremony with no current consumer. Prefer: close A2-P1 into A2-P5's policy doc. | false | false |
| A2-P2 | **disagree** | Schema-registered `LevelOverrides` for v1 is YAGNI (G2). The design already tracks Q6 as open; the right vehicle is the Q6 ADR when it lands, not a parallel registry now. Synth amendment ("closed for v1, schema-registered for v2") — if adopted, I flip to agree. | false | false |
| A2-P3 | **abstain** | Keeping `DepMode` closed is the right call (A2's own pre-compromise); opening the other enums for *schema-open / code-closed* is harmless if strictly documentation. If "open" means adding a registry mechanism, this is YAGNI (G2). Vote depends on concrete wording; abstain pending clarification. | false | false |
| A2-P4 | agree | A migration & transition plan is documentation that every real runtime needs (E5); no code surface added. | false | false |
| A2-P5 | agree | An evolution / backward-compat policy is one document; absorbs A2-P1's motivation without requiring a version byte on every struct today. (G2 satisfied if A2-P1 is withdrawn into A2-P5.) | false | false |
| A2-P6 | **disagree** | Pluggable `IDistributedProtocolHandler` registry with one v1 implementation is the canonical YAGNI violation (G2, rubric check 4). Synth amendment ("abstract only in `distributed_scheduler`; single concrete backend with inlined dispatch") — if the abstract class has exactly zero production polymorphism in v1, call it an internal abstraction, don't advertise it as an extension point. Still a disagree on the pluggable *registry* framing. | false | false |
| A2-P7 | **disagree** | An async-policy extension seam without a named future extender (Q that it resolves, concrete second implementation timelined) is speculative (G2). Convert to a Q-record entry if the concern is real — don't reserve interface slots for hypothetical extenders. | false | false |
| A2-P8 | agree | Recording intentional closures as known deviations is KISS-compatible documentation discipline (G2, G5). | false | false |
| A2-P9 | agree | A versioned trace schema (one byte or field) is cheap and traces *do* evolve; the cost/benefit is clearly on the side of agreement (E6). | false | false |

### A3 — Functional Sanity & Correctness

| proposal_id | vote | rationale (cite rule id) | blocking | override_request |
|-------------|------|--------------------------|----------|-------------------|
| A3-P1 | agree | Adding `ERROR` (and optional `CANCELLED`) to the Task FSM is a correctness fix, not complexity (G1 requirements-first overrides G2). A missing terminal state is a real bug; the FSM remains small. | false | false |
| A3-P2 | agree | Clarifying `submit()` return vs `AdmissionDecision` is documentation only; aligns with A9-P5 (unified enum). | false | false |
| A3-P3 | agree | Admission-path failure scenario fills a gap in the scenario view; no runtime surface added. | false | false |
| A3-P4 | agree | Producer-failure → consumer propagation is a correctness requirement (DS3); writing it down shrinks later rework. | false | false |
| A3-P5 | agree | Specifying sibling cancellation policy prevents undefined behavior; correctness trumps simplicity concerns here. | false | false |
| A3-P6 | agree | Requirements ↔ scenario traceability matrix is a single doc artifact; G5 (justify decisions) benefits. | false | false |
| A3-P7 | agree | Submission precondition validation compatible with A9-P4 (uses `spmd.has_value()` rather than `Kind`). | false | false |
| A3-P8 | **disagree** | Cyclic `intra_edges` detection on the admission path is `extends(bounded)` HPI — simplicity *and* A1 should push back unless it's debug-only. Synth amendment ("confine detection to debug mode; fast path skips") would flip me to agree. As written: disagree. | false | false |
| A3-P9 | agree | SPMD index/size delivery contract clarifies an existing ambiguity; doc-only. | false | false |
| A3-P10 | agree | Python exception mapping completeness prevents silent failures; lightweight. | false | false |
| A3-P11 | agree | `[ASSUMPTION]` marker at `complete_in_future` aligns with G3. | false | false |
| A3-P12 | agree | `drain()`/`submit()` concurrency contract is a correctness gap; lightweight. | false | false |
| A3-P13 | agree | Cross-node ordering assumptions are necessary for reasoning about distributed correctness (DS6). | false | false |
| A3-P14 | agree | `COMPLETING`-skip transition at leaf is a minor FSM clarification. | false | false |
| A3-P15 | agree | Debug-mode `NONE`-dep-mode cross-check is debug-only; no hot-path impact. | false | false |

### A4 — Documentation Consistency

All A4 proposals are documentation/naming fixes; none add runtime surface. A9 has no simplicity objection to any of them.

| proposal_id | vote | rationale (cite rule id) | blocking | override_request |
|-------------|------|--------------------------|----------|-------------------|
| A4-P1 | agree | Canonical casing enforces D7 (consistent naming). | false | false |
| A4-P2 | agree | Broken-anchor fix is pure maintenance. | false | false |
| A4-P3 | agree | Invariant-count prose fix. | false | false |
| A4-P4 | agree | Reordering open questions is cosmetic but cheap. | false | false |
| A4-P5 | agree | Glossary entries for event-loop plumbing — with the amendment that if A9-P7 passes, the `SourceCollectionConfig` entry collapses into `EventHandlingConfig`. Coordinated in A9-P7 revision. | false | false |
| A4-P6 | agree | Cross-linking views to ADRs improves G5 (justify decisions). | false | false |
| A4-P7 | agree | Unifying the L0 component label is D7 (consistent naming). | false | false |
| A4-P8 | agree | Appendix-B sync. | false | false |
| A4-P9 | agree | Glossary expansion for `Task State`. | false | false |

### A5 — Reliability

| proposal_id | vote | rationale (cite rule id) | blocking | override_request |
|-------------|------|--------------------------|----------|-------------------|
| A5-P1 | agree | Exponential backoff + jitter for remote retries is textbook (R2); no simplicity cost. | false | false |
| A5-P2 | agree | Per-peer circuit breaker in `distributed/` is R3; scoped to distributed layer, not hot path. | false | false |
| A5-P3 | agree | With the synthesiser amendment ("v1 = deterministic fail-fast + Q-record; v2 = decentralised coordinator"), this is the KISS option — A9 is explicitly aligned with fail-fast v1. Cite G2, DS3. | false | false |
| A5-P4 | agree | Single `idempotent: bool` on `TaskDescriptor` is minimum-cost, maximum-correctness (DS4). | false | false |
| A5-P5 | agree | Chaos harness is sim-only; does not add hot-path surface (R6, X5). | false | false |
| A5-P6 | agree | Scheduler-thread watchdog/deadman merges with A8-P4's `dump_state()` evidence surface; one paired addition (R4). | false | false |
| A5-P7 | agree | `Timeout` on `IMemoryOps` async is R1 (timeout all remote calls). | false | false |
| A5-P8 | agree | Degradation specs for admission saturation + WG loss are documentation of behavior that must exist anyway (R4). | false | false |
| A5-P9 | **disagree** | `QUARANTINED` between `FAILED` and `UNAVAILABLE` is FSM inflation (G2, rubric check 5). The time-window behavior can be expressed as a timestamp field on `UNAVAILABLE` without a new state. | false | false |
| A5-P10 | agree | DS4 per-REMOTE_* idempotency protocol contract is correctness documentation; lightweight. | false | false |

### A6 — Security & Trust Boundaries

| proposal_id | vote | rationale (cite rule id) | blocking | override_request |
|-------------|------|--------------------------|----------|-------------------|
| A6-P1 | agree | Trust-boundary threat model is foundational documentation (S1); keeps later security work scoped. | false | false |
| A6-P2 | agree | Concrete node authentication in `HANDSHAKE` is S2 (least privilege) on a boundary the design already crosses. | false | false |
| A6-P3 | agree | Bounded variable-length payload parsing with a single length-prefix guard at boundary entry (per synth amendment) is simple *and* sound (S3). | false | false |
| A6-P4 | agree | TLS on multi-host TCP *only* (RDMA untouched per synth amendment) is S5 with a clean two-tier path; KISS-compatible given the path separation. | false | false |
| A6-P5 | agree | Scoped, revocable RDMA `rkey` with rotation on Submission retirement (per synth amendment) is a lifecycle already implied by Submission completion — minimal extra ceremony. | false | false |
| A6-P6 | agree | Security audit trail (S6) is documentation of structured logging; integrates with A8-P10. | false | false |
| A6-P7 | **disagree** | Function-binary attestation is YAGNI (G2) for v1 without a named multi-tenant scenario. Synth amendment ("gate behind `trust_boundary.multi_tenant=true`; off by default") would flip me to agree — but only if the default is off and the check is absent in the single-tenant path. | false | false |
| A6-P8 | agree | Merged into A1-P11 per register; single per-arg budget covers both marshalling cost and boundary validation (DRY, S3). | false | false |
| A6-P9 | agree | Logical System isolation in wire and code is S2; the isolation invariant already exists in the design and needs enforcement hooks. | false | false |
| A6-P10 | agree | Capability-scoped log/trace sinks are S2 applied to observability. | false | false |
| A6-P11 | agree | Gating `register_factory` is S2/S4 (secure defaults) on a surface that's already present. | false | false |
| A6-P12 | **disagree** | Moot if A9-P6 passes (REPLAY deferred). Even if REPLAY is kept, signed/schema-validated trace is YAGNI without a named multi-tenant debugger. Route: decide A9-P6 first, then re-evaluate. | false | false |
| A6-P13 | **disagree** | No multi-tenant scenario in v1 → no per-tenant rate-limit needed (G2). Synth suggests deferral; A9 concurs. | false | false |
| A6-P14 | agree | Key-material lifecycle ADR is documentation; necessary once any key exists. | false | false |

### A7 — Modularity & SOLID

| proposal_id | vote | rationale (cite rule id) | blocking | override_request |
|-------------|------|--------------------------|----------|-------------------|
| A7-P1 | agree | Breaking the `scheduler/` ↔ `distributed/` cycle is D6 (DAG dependencies) — a hard rule. Simplicity agrees: a cycle is harder to reason about than a DAG. | false | false |
| A7-P2 | agree | **Merged with A9-P1** per register. Combined form: role-split `ISchedulerLayer` (A7-P2) + single `submit(SubmissionDescriptor)` (A9-P1). Net result fewer virtual methods than either alone; ISP + G2 both satisfied. | false | false |
| A7-P3 | agree | Inverting `core/` ↔ `hal/` for handle types is DIP (D2); aligns cohesion (D5). | false | false |
| A7-P4 | agree | Moving distributed payload structs to `distributed/` is cohesion (D5). | false | false |
| A7-P5 | agree | `distributed_scheduler` depending only on `ISchedulerLayer` is narrow coupling (D4). | false | false |
| A7-P6 | agree | Extracting MLR + deployment parser is SRP (D3). | false | false |
| A7-P7 | agree | Forward-decl contract is maintenance. | false | false |
| A7-P8 | agree | Consolidating `ScopeHandle` ownership reduces coupling (D4). | false | false |
| A7-P9 | agree | De-duplicating `MemoryError` is DRY (§2.5); trivial. | false | false |

### A8 — Testability & Observability

| proposal_id | vote | rationale (cite rule id) | blocking | override_request |
|-------------|------|--------------------------|----------|-------------------|
| A8-P1 | agree | `IClock` is a single test seam (X5); KISS-compatible — one interface, two impls (production + test double). | false | false |
| A8-P2 | agree | Driveable event-loop is compatible with A9-P2's amendment (single `IEventLoopDriver` seam, not a pluggable-policy registry). Bridge accepted. (X5) | false | false |
| A8-P3 | agree | Stats structs + bucketed histograms are a single data-surface addition (O3); no new runtime abstractions. | false | false |
| A8-P4 | agree | `dump_state()` diagnostic endpoint merged with A5-P6 watchdog; single paired fast-track. | false | false |
| A8-P5 | **disagree** | Externalising alert rules *and* adding an OTEL/Prom sink in one proposal ships two systems at once for a v1 with no operator story (G2). Synth amendment (alert-rule *file location* only; OTEL sink as known-deviation ledger entry) would flip me to agree. As written: disagree. | false | false |
| A8-P6 | agree | Distributed trace time-alignment is O1 (trace IDs / timestamps); doc + one contract. | false | false |
| A8-P7 | agree | `IFaultInjector` is sim-only (X5); does not touch hot path. | false | false |
| A8-P8 | agree | AICore in-core trace upload protocol is a boundary commitment, not new runtime surface. | false | false |
| A8-P9 | agree | Profiling drop/degraded alerts are O5 (SLO-driven alerts); tiny addition. | false | false |
| A8-P10 | agree | Structured KV logging as primary surface is O2; simpler than ad-hoc log strings once adopted. | false | false |
| A8-P11 | agree | HAL contract test suite (X5) aligns with A1-P13's HAL tightening; complementary. | false | false |
| A8-P12 | agree | Stable `PhaseId`s for Submission lifecycle is O1 (trace IDs); lightweight. | false | false |

### A10 — Scalability & Data Flow

| proposal_id | vote | rationale (cite rule id) | blocking | override_request |
|-------------|------|--------------------------|----------|-------------------|
| A10-P1 | agree | Default `producer_index` sharding merged with A1-P14; documentation-only. | false | false |
| A10-P2 | **disagree** | Merged with A5-P3 per register. A9 is aligned with the fail-fast v1 branch of that merger, not the decentralised-coordinator branch (G2, YAGNI for v1 scope). | false | false |
| A10-P3 | agree | Explicit per-data-element consistency model is DS6 (required). | false | false |
| A10-P4 | agree | Stateful/stateless module classification is documentation; helps reasoning about DS1. | false | false |
| A10-P5 | agree | Per-peer projection of `REMOTE_SUBMIT` merged with A1-P6; net-simpler combined proposal. | false | false |
| A10-P6 | agree | Configurable peer-failure detection is existing need; single tunable (E3). | false | false |
| A10-P7 | **disagree** | Sharded TaskManager with HPI=`relayouts` for a v1 with no measured single-thread bottleneck is premature (G2, P3). Synth amendment ("two-tier: default `shard_count=1` unsharded fast path; sharded when LevelParam overrides") would flip me to agree — but that essentially defers the sharded path to "available as config, not default." Accept the deferral; as-written proposal rejected. | false | false |
| A10-P8 | agree | Aggregated data-flow + ownership diagram is documentation (DS7). | false | false |
| A10-P9 | agree | Gating `WorkStealing` against `RETRY_ELSEWHERE` is a small correctness guard. | false | false |
| A10-P10 | agree | Cap + layout guidance for `producer_index` merged with A1-P14; documentation only. | false | false |

### Vote Summary

- **agree:** 76
- **disagree:** 18 (A1-P4, A1-P9, A1-P12, A2-P1, A2-P2, A2-P6, A2-P7, A3-P8, A5-P9, A6-P7, A6-P12, A6-P13, A8-P5, A10-P2, A10-P7; + implicit disagree-as-written on several more where a synth amendment would flip the vote)
- **abstain:** 1 (A2-P3 — pending concrete "open" wording)
- **blocking objections filed:** 0 (A9's primary rules are design-discipline soft rules; A9 does not hold hard-rule veto power and therefore reserves `blocking=true`)
- **override_requests:** 0

Pattern: A9 agrees with proposals that *simplify* (remove enums, pre-size allocations, collapse configs, cut dead interfaces) and with documentation/correctness proposals; A9 disagrees with proposals that *add surface for hypothetical future extenders* (A2-P1/P2/P6/P7, A6-P7/P12/P13, A8-P5, A10-P2/P7) or that *prescribe layout/tunables ahead of benchmarks* (A1-P4/P9/P12, A3-P8). In every "disagree" case there is a synth-suggested amendment (or a trivial one) that A9 would accept.

## 7. Cross-aspect tensions

| pair | with_proposal | preferred_resolution |
|------|---------------|----------------------|
| A2 vs A9 | A2-P1 (version every contract) | Fold A2-P1's motivation into A2-P5 (evolution policy as one document). Do not add version fields to headers that haven't evolved. One policy edit rather than N header edits. |
| A2 vs A9 | A2-P2 (schema-registered `LevelOverrides`) | Defer to Q6 ADR. For v1, the enum stays closed; the schema-registration mechanism ships with the Q6 ADR when a second override source is named. Phase as "closed for v1, schema-registered for v2." |
| A2 vs A9 | A2-P6 (pluggable `IDistributedProtocolHandler`) | Keep the abstract class *internal* to `distributed_scheduler` only; deliver exactly one concrete backend with inlined dispatch (devirt); do *not* advertise a registry. Flip from "extension point" framing to "internal abstraction we may open later" framing. |
| A2 vs A9 | A2-P7 (async-policy seam) | Convert to a Q-record entry rather than an interface seam. Name the trigger: "reserve interface when a concrete second async policy is on a roadmap." |
| A6 vs A9 | A6-P7 (function-binary attestation) | Gate behind `trust_boundary.multi_tenant=true`; default off. For v1 single-tenant deployments, the code path is absent — zero ceremony. |
| A6 vs A9 | A6-P12 (signed REPLAY trace) | Resolve A9-P6 first. If REPLAY deferred (A9-P6 agreed), A6-P12 is automatically withdrawn. If REPLAY kept, revisit in R3 with explicit threat. |
| A6 vs A9 | A6-P13 (per-tenant rate-limit) | Defer until multi-tenant deployment is a concrete scenario. Add to `09-open-questions.md` with the trigger condition. |
| A8 vs A9 | A8-P2 (driveable event loop) | Single `IEventLoopDriver` test seam, one production impl + one test double (`RecordedEventSource` + `step()`). This satisfies X5 without reintroducing `IEventCollectionPolicy`/`IExecutionPolicy` registries (which A9-P2 removes). Bridge agreed. |
| A8 vs A9 | A8-P5 (externalize alert rules + OTEL sink) | Two-step: (1) document the alert-rule file location as config (E3); (2) record "OTEL sink" as a known deviation until an operator story lands. Do not ship two systems in one proposal. |
| A1 vs A9 | A1-P4, A1-P9, A1-P12 | Two-tier or "move-prescription-to-module-doc" pattern. Hot-path prescriptions live in `modules/*.md` §8 (layout) or behind explicit fast-path defaults; the Logical View stays abstract. |
| A5 vs A9 | A5-P3 (coordinator failover) | Accept synth amendment: v1 = deterministic fail-fast; v2 = decentralized (or quorum) coordinator. A9 endorses the v1 KISS path. |
| A10 vs A9 | A10-P2 (decentralize Pod coordinator) | Merge into A5-P3 and take the fail-fast v1 path (same resolution as above). |
| A10 vs A9 | A10-P7 (sharded TaskManager) | Two-tier: `shard_count=1` default (no relayout); sharded path is a `LevelParam` override documented but not the default. A10 provides the config sketch; A9 accepts the deferral. |
| A2 vs A9 (meta) | A9-P2, A9-P3 (general A2 pushback) | Extension interfaces are **documented in an appendix as "available when a second implementation exists"** — not deleted, but not v1 obligations. Implementations that want to extend add the interface in a later ADR; until then, the built-in path is non-virtual. |

## 8. Stress-attack on emerging consensus

*N/A — round 2; stress attacks apply at round 3+ only.*

## 9. Status

- **Satisfied with current design?** **partially** — Round-1 concerns are largely preserved; the peer reviews validated A9's core tensions (four of A9's eight proposals have synth-proposed amendments that make them tighter, not weaker). The merger of A9-P1 with A7-P2 is a clear simplification win. Remaining headline debate is A9-P2 (event-loop deployment modes + pluggable policies) and A9-P6 (sim-mode deferral); both have concrete bridge proposals now.
- **Open items expected in next round:**
  - **Top debates:** A9-P2 (resolution hinges on A2's acceptance of the "extension points as appendix" pattern and A8's acceptance of single `IEventLoopDriver` seam); A9-P6 (resolution hinges on A8's acceptance of REPLAY deferral with named-trigger re-entry).
  - **Expected fast convergence:** A9-P1 (merged with A7-P2), A9-P3 (Q4 resolved to composition), A9-P4 (drop `Kind`), A9-P5 (unify admission enum), A9-P7 (fold config with A4-P5 coordination), A9-P8 (move artifact obligation out).
  - **Merger register items involving A9 — A9's position:** (1) **A7-P2 absorbed into A9-P1**: **accept** — treat as the new combined proposal described in §5. (2) No other merges name A9 as owner or absorbed.
  - **Expected A9 votes to flip if amendments land:** A1-P4, A1-P12, A2-P2, A3-P8, A6-P7, A8-P5 all flip to agree if the corresponding synth-suggested amendment is adopted. A10-P7 flips to agree if the two-tier default=1 phrasing is explicit.
