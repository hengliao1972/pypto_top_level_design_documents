# Aspect A9: Simplicity (KISS/YAGNI) — Round 1

## Metadata

- **Reviewer:** A9
- **Round:** 1
- **Target:** `/data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/`
- **Runtime Mode:** yes
- **Timestamp (UTC):** 2026-04-19

## 1. Rubric Checks

| # | Check | Result | Evidence (file:line) | Rule(s) |
|---|-------|--------|----------------------|---------|
| 1 | Every abstraction earns its keep — current requirement demands it | Weak | `docs/pypto-runtime-design/02-logical-view/05-machine-level-registry.md:15-30` (12 `I*Factory` slots on `MachineLevelDescriptor`; many have exactly one registered implementation today) | G2, G4 |
| 2 | Speculative future-proofing features present | Fail | `docs/pypto-runtime-design/02-logical-view/09-interfaces.md:436-442` (`FULLY_SPLIT`/`SPLIT_DEFERRED` deployment modes with no workload driver); `docs/pypto-runtime-design/02-logical-view/05-machine-level-registry.md:144-145` (`IHorizontalChannel::all_reduce`/`barrier` added while `09-open-questions.md:47-57` still debates whether collectives belong there at all); `docs/pypto-runtime-design/08-design-decisions.md:455-461` (`PERFORMANCE` + `REPLAY` sim modes each require per-Function companion artifacts with no v1 workload) | G2 (YAGNI) |
| 3 | Patterns/frameworks chosen for clarity, not cleverness | Weak | `docs/pypto-runtime-design/02-logical-view/02-scheduler.md:266-358` (3-stage A/B/C event loop × 4 deployment modes × pluggable `IEventCollectionPolicy`/`IExecutionPolicy` — conceptual surface is large for a loop whose default is "run A→B→C on one thread") | G4 |
| 4 | No single-implementation wrappers / interfaces with no planned variability | Weak | `docs/pypto-runtime-design/02-logical-view/09-interfaces.md:33-38` (`submit`/`submit_group`/`submit_spmd` overloads all build the same `SubmissionDescriptor` — three entry points for one operation); `docs/pypto-runtime-design/02-logical-view/09-interfaces.md:398-427` (`IEventCollectionPolicy` interface whose four documented variants are all config-parameter variations of weighted-round-robin); `docs/pypto-runtime-design/02-logical-view/09-interfaces.md:98-120` (`SubmissionDescriptor::Kind {SINGLE, GROUP, SPMD}` tag that the runtime treats uniformly per `07-task-model.md:69` "Group is the general form; Single and SPMD are special cases") | G2 |
| 5 | DRY applied to knowledge, not forcing unrelated concepts together | Weak | `docs/pypto-runtime-design/02-logical-view/09-interfaces.md:242-243` (`AdmissionDecision {ADMIT, WAIT, REJECT}`) vs `docs/pypto-runtime-design/02-logical-view/09-interfaces.md:293-297` (`ResourceAllocationResult::Status {ALLOCATED, WAIT, REJECTED}`) — two near-identical enums on the same policy hook; `docs/pypto-runtime-design/02-logical-view/05-machine-level-registry.md:21` vs `docs/pypto-runtime-design/02-logical-view/09-interfaces.md:358-396` (`EventHandlingConfig` + `SourceCollectionConfig` are two config structs that together describe one behavior: "how should Stage A/B process events?") | G2, DRY |

## 2. Pros

- **Uniform `ISchedulerLayer` contract at every level** (`docs/pypto-runtime-design/08-design-decisions.md:77-108`, ADR-003) keeps one mental model across the stack and is a textbook KISS/G2 win; worth preserving.
- **Runtime dependency model deliberately limited to cross-Submission RAW in v1** (`docs/pypto-runtime-design/02-logical-view/12-dependency-model.md:115-128`) — explicit YAGNI discipline: WAR/WAW, version lists, sub-range tracking, and token tensors are all **Deferred/Rejected** for v1 and routed to `09-open-questions.md`. Exactly how a runtime design should manage speculative complexity. Cites G2/YAGNI.
- **Token-tensor dep representation rejected for the runtime** (`docs/pypto-runtime-design/09-open-questions.md:176-189` Q14) with "runtime already has the 'one unified DAG' property via explicit edges; tokens would double object count for no expressive power". This is a clean G2/DRY application: don't bolt on an alternative encoding for the same information.
- **Homogeneous levels as degenerate case** (`docs/pypto-runtime-design/02-logical-view/03-worker.md:153-154`) — worker types/groups are optional, the default path is a flat pool. Good KISS default for the 99% case.
- **Known-deviations document acknowledges deferred complexity honestly** (`docs/pypto-runtime-design/10-known-deviations.md:19-27` — no persistent state recovery, justified by G2/X1). Rather than quietly doing it, the design says "we are not doing this; here is why". Cites G2.
- **ADR-004** (`docs/pypto-runtime-design/08-design-decisions.md:110-143`) explicitly rejects the three-separate-paths channel design with "Violates KISS (Rule G2); excessive complexity". The design shows awareness of KISS as a decision criterion.
- **Function Caching defaults and scope bounds documented** (`docs/pypto-runtime-design/08-design-decisions.md:214-240`, ADR-007) constrain cache scope to a single process with explicit invalidation — avoided the "distributed function cache" rabbit hole.
- **Strict DAG module dependency** (`docs/pypto-runtime-design/02-logical-view.md`, `03-development-view.md`) — D6 satisfied — makes the runtime easier to reason about than a tangled call graph.

## 3. Cons

- **Factory inflation on `MachineLevelDescriptor`** (`docs/pypto-runtime-design/02-logical-view/05-machine-level-registry.md:15-30`): 12 `I*Factory*` fields, several of which (`IEventCollectionPolicyFactory`, `IExecutionPolicyFactory`, `IResourceAllocationPolicyFactory`, `IWorkerSelectionPolicyFactory`) have exactly one concrete implementation today and no concrete second one on the roadmap. Violates G2 ("avoid over-engineering… simplest design that meets current requirements").
- **Three entry points for one operation** (`docs/pypto-runtime-design/02-logical-view/09-interfaces.md:28-38`): `submit(SubmissionDescriptor)` is the canonical call, yet `submit(TaskDescriptor)`, `submit_group(...)`, and `submit_spmd(...)` are all added as "convenience wrappers" that internally build the same `SubmissionDescriptor`. Convenience wrappers belong outside the interface, not as virtual methods every implementation must override (G2, G4, ISP tension with A7 noted below).
- **Redundant `SubmissionDescriptor::Kind` tag** (`docs/pypto-runtime-design/02-logical-view/09-interfaces.md:99-119`, `02-logical-view/07-task-model.md:26`): the docs themselves state SINGLE and SPMD are **specializations of GROUP**, and TaskManager treats them uniformly (`02-logical-view/02-scheduler.md:28`). Carrying `Kind` adds a discriminant the runtime never branches on correctness-critically. Violates DRY on knowledge.
- **Speculative event-loop deployment modes** (`docs/pypto-runtime-design/02-logical-view/09-interfaces.md:434-454` / `02-scheduler.md:310-324`): four modes — `SINGLE_THREADED`, `SPLIT_COLLECTION`, `SPLIT_DEFERRED`, `FULLY_SPLIT` — introduced in one pass. Only `SINGLE_THREADED` has an articulated current workload ("default, Core on AICPU"). `FULLY_SPLIT` is justified only by "Distributed/root levels where collection, decision-making, and batched policy work have different SLOs" — a hypothetical. YAGNI (G2).
- **`IEventCollectionPolicy` is a config-in-interface-clothing** (`docs/pypto-runtime-design/02-logical-view/09-interfaces.md:398-427`): the interface exists so four default implementations (fixed-priority, round-robin, weighted-round-robin, custom) can be selected per level. The four are all parameterizations of the same policy and would be cleaner as an enum + `SourceCollectionConfig` struct, not a virtual interface. Violates G2 ("wrappers/adapters/interfaces with only one implementation and no planned variability").
- **Two near-duplicate admission enums** (`docs/pypto-runtime-design/02-logical-view/09-interfaces.md:242-243` vs `:293-297`): `AdmissionDecision {ADMIT, WAIT, REJECT}` and `ResourceAllocationResult::Status {ALLOCATED, WAIT, REJECTED}` describe the same admission outcome at two different scopes of the same policy. The caller must translate between them. DRY (§2.5), G2.
- **`all_reduce` and `barrier` on `IHorizontalChannel`** (`docs/pypto-runtime-design/02-logical-view/05-machine-level-registry.md:144-145`) are on the interface *before* the open question Q4 ("should collectives be composed or native?", `09-open-questions.md:47-57`) is resolved. Every non-trivial `IHorizontalChannel` implementation must now implement (or stub) both methods — cost that every plug-in pays for a decision not yet made. YAGNI (G2).
- **Simulation facility introduces three distinct execution pipelines in v1** (`docs/pypto-runtime-design/08-design-decisions.md:451-472`, ADR-011): `PERFORMANCE` (requires per-Function timing models), `FUNCTIONAL` (CPU backend per AICore Function), and `REPLAY` (trace-file deserializer + state-machine driver). The ADR itself flags "Each AICore Function now has up to three companion artifacts" (`:487`). A v1 runtime design spec that commits to three simulator modes and their artifact pipelines before CI requirements surface. YAGNI (G2).
- **`EventHandlingConfig` + `SourceCollectionConfig` + `EventLoopDeploymentConfig` fan-out** (`docs/pypto-runtime-design/02-logical-view/09-interfaces.md:358-456`): three config structs per Machine Level for one event loop's behavior, with a documented runtime validation pass (`:456`) for mutual consistency. Configuration surface area that users must learn and that implementations must validate. G2.
- **Per-Function "up to three companion artifacts" design obligation** (`docs/pypto-runtime-design/08-design-decisions.md:487`) is stated as a runtime-design-level consequence, yet the artifacts (timing model + CPU backend) live in compiler/kernel tooling. The design doc takes on scope it doesn't need to. G2.

## 4. Proposals

| id | severity | summary | affected_docs | hot_path_impact | trade_offs | sanity_test |
|----|----------|---------|---------------|-----------------|------------|-------------|
| A9-P1 | medium | Remove `submit_group` / `submit_spmd` / overloaded `submit(TaskDescriptor)` from `ISchedulerLayer`; keep only `submit(SubmissionDescriptor)` | `docs/pypto-runtime-design/02-logical-view/09-interfaces.md`, `docs/pypto-runtime-design/02-logical-view/07-task-model.md` | none | Gain: one entry point, one admission contract, smaller vtable. Give up: a couple more lines at call sites (or a free helper). | Grep for callers; confirm each rewrites to a 3-line `SubmissionDescriptor` builder; check vtable / ABI footprint shrinks. |
| A9-P2 | high | Defer `FULLY_SPLIT` and `SPLIT_DEFERRED` event-loop deployments and delete `IEventCollectionPolicy` / `IExecutionPolicy` as pluggable interfaces for v1; express the remaining two modes and the collection behavior as enums + config structs | `docs/pypto-runtime-design/02-logical-view/02-scheduler.md`, `docs/pypto-runtime-design/02-logical-view/09-interfaces.md`, `docs/pypto-runtime-design/02-logical-view/05-machine-level-registry.md`, `docs/pypto-runtime-design/08-design-decisions.md` (ADR-010) | none (slightly reduces virtual dispatch on the event-loop hot path) | Gain: one interface removed, two deployment modes cut until justified by profile data, scheduler loop becomes a switch + config. Give up: reintroduction cost if/when a real workload demands per-stage threading. | Confirm every concrete `IEventCollectionPolicy`/`IExecutionPolicy` in the current repo collapses to an enum + parameters; confirm no ADR references the removed modes as a critical-path requirement. |
| A9-P3 | high | Remove `barrier()` / `all_reduce()` from `IHorizontalChannel` for v1; commit to Q4 option (A) — collectives are Orchestration Functions on top of `transfer()` | `docs/pypto-runtime-design/02-logical-view/05-machine-level-registry.md` (lines 133-150), `docs/pypto-runtime-design/09-open-questions.md` (Q4), `docs/pypto-runtime-design/modules/transport.md` | none | Gain: every `IHorizontalChannel` implementer saves 2 methods; collectives question closed with "Orchestration function composition" until hardware collective perf data arrives. Give up: near-term HCCL native hook must come back as a channel extension interface later. | Enumerate current channel backends (shared-mem, RegisterBank, TCP, RDMA, TPUSH/TPOP); verify each keeps only `send`/`recv`/`poll`/`transfer`; verify distributed scenarios in `06-scenario-view.md:35-58` still compose. |
| A9-P4 | medium | Drop `SubmissionDescriptor::Kind`; runtime treats every Submission as a Group (SINGLE and SPMD are the special cases already described in `07-task-model.md:69` and `:205-223`) | `docs/pypto-runtime-design/02-logical-view/09-interfaces.md:98-119`, `docs/pypto-runtime-design/02-logical-view/07-task-model.md` | none | Gain: one discriminant removed from the hot admission path; one less invariant to document. Give up: SPMD-specific tuning must re-derive shape from `tasks.size()` + SPMD descriptor presence. | Run every admission code path mentally with `Kind` removed; confirm the branch on `Kind` in TaskManager is either absent or a pure optimization hint that can be recovered from `spmd.has_value()`. |
| A9-P5 | medium | Unify `IResourceAllocationPolicy::AdmissionDecision` and `ResourceAllocationResult::Status` on one enum, e.g., `AdmissionStatus {Admitted, WaitForWindow, Rejected}` | `docs/pypto-runtime-design/02-logical-view/09-interfaces.md:242-297` | none | Gain: DRY on admission outcome; callers stop translating between the two. Give up: tiny naming churn in `modules/scheduler.md`. | Grep for both enums' symbols; confirm each callsite uses the unified enum and no information is lost. |
| A9-P6 | high | Defer `PERFORMANCE` and `REPLAY` simulation modes to a post-v1 scope; ship v1 with `FUNCTIONAL` only | `docs/pypto-runtime-design/08-design-decisions.md` (ADR-011), `docs/pypto-runtime-design/02-logical-view/10-platform.md` (§2.8.1), `docs/pypto-runtime-design/07-cross-cutting-concerns.md` (§7.2) | none | Gain: no per-Function timing-model or trace-replay artifact pipeline in v1; ADR-011 "Negative #1" (three companion artifacts per AICore Function) eliminated. Give up: perf-modeling-without-hardware workflow and post-mortem replay workflow deferred until a concrete consumer asks for them. | Confirm `06-scenario-view.md` has no scenario requiring `PERFORMANCE` or `REPLAY`; confirm `01-introduction.md` FR-10 can be met by `FUNCTIONAL` + `ONBOARD` in v1; enumerate the test-plan entries that would need `PERFORMANCE` and verify none is blocking. |
| A9-P7 | medium | Fold `SourceCollectionConfig` into `EventHandlingConfig`; delete the separate struct. The merged `EventHandlingConfig` captures both per-event inline/deferred mode and per-source collection parameters | `docs/pypto-runtime-design/02-logical-view/09-interfaces.md:358-427`, `docs/pypto-runtime-design/02-logical-view/05-machine-level-registry.md:21` | none | Gain: one config struct per level instead of three (with `EventLoopDeploymentConfig`); one mutual-consistency pass removed from startup. Give up: slight renaming in `LevelParams`. | Attempt the merge on paper; confirm every field in `SourceCollectionConfig` has a clean home in `EventHandlingConfig`; verify startup validation in `09-interfaces.md:456` collapses to a single pass. |
| A9-P8 | low | Move the "up to three companion artifacts per AICore Function" statement out of the runtime design doc into a compiler/kernel-tooling design doc (or an explicit out-of-scope note) | `docs/pypto-runtime-design/08-design-decisions.md:487` (ADR-011 Negative) | none | Gain: runtime design doc scope stays runtime-only; artifact pipeline becomes a frontend concern, not an architectural obligation. Give up: the ADR must cross-reference the frontend doc instead of stating the obligation inline. | After edit, confirm no section of `docs/pypto-runtime-design/` requires a specific compiler-artifact format; confirm `ADR-011` still reads as a complete decision when the obligation is linked out. |

### Proposal detail

#### A9-P1: Drop `submit_group` / `submit_spmd` overloads from `ISchedulerLayer`

- **Rationale:** Rule G2 / rubric check 4 — single-implementation wrappers with no variability. `07-task-model.md:69` states "Group Submission is the general form; Single and SPMD are special cases." Three virtual entry points where one suffices violates both G2 (simplicity) and G4 (clarity: caller must pick the "right" entry point when behavior is identical). The `SubmissionDescriptor` canonical path already exists at `09-interfaces.md:28`.
- **Edit sketch:**
  - File: `docs/pypto-runtime-design/02-logical-view/09-interfaces.md`
  - Location: lines 28-41 (method list of `ISchedulerLayer`)
  - Delta: delete the three convenience wrappers (lines 33-38); keep only `submit(const SubmissionDescriptor&)`. Add a one-paragraph note above the class naming free-helper builders (`build_single(...)`, `build_group(...)`, `build_spmd(...)`) living in `runtime/` as non-virtual factories.
- **Trade-offs:**
  - Gain: `ISchedulerLayer` vtable shrinks by three slots; every implementation writes one `submit(...)` rather than four; call sites describe intent via the descriptor they build.
  - Give up: call-site verbosity increases by ~3 lines per call; any existing prototype code using the convenience entry points must be updated.
- **Sanity test:** Rebuild the interface in a scratch pad; show that every scenario in `06-scenario-view.md` still compiles against only `submit(SubmissionDescriptor)`. Confirm module `scheduler.md` loses no required functionality.

#### A9-P2: Cut speculative event-loop deployments and pluggable policies

- **Rationale:** Rule G2 / rubric check 2. `09-interfaces.md:436-442` introduces four deployment modes; `02-scheduler.md:314-319` justifies `FULLY_SPLIT` only with "Distributed/root levels where collection, decision-making, and batched policy work have different SLOs" — a hypothetical, not a workload. `IEventCollectionPolicy` (`09-interfaces.md:398-427`) exists to select between four documented variants that are all parameterizations of weighted-round-robin; that is configuration, not polymorphism (rubric check 4). `IExecutionPolicy` (`09-interfaces.md:458+`) has the same shape. ADR-010 (`08-design-decisions.md:400-420`) accepts "configuration complexity" as a known Negative; simplicity review recommends paying that cost only when a concrete workload demands it.
- **Edit sketch:**
  - File: `docs/pypto-runtime-design/02-logical-view/02-scheduler.md`
  - Location: §2.1.3.5 (lines 266-360) and §2.1.3.6 (lines 366+).
  - Delta: reduce the deployment table to `SINGLE_THREADED` (default) + `SPLIT_COLLECTION` (opt-in for Host/Pod with networked event sources). Replace `IEventCollectionPolicy` with a `SourceCollectionConfig` weighted-round-robin driver and `IExecutionPolicy` with an `ExecutionMode` enum (`Dedicated | Interleaved | Batched`) + small switch in the loop. Update `MachineLevelDescriptor` fields accordingly in `05-machine-level-registry.md:29-30`. Update ADR-010 Negative to "two interfaces collapsed to config; two modes cut until workload evidence".
- **Trade-offs:**
  - Gain: two interfaces removed from the framework ABI; two deployment modes cut until profile data demands them; scheduler event loop reads as a plain A→B→C loop with branches rather than a policy engine.
  - Give up: re-adding interfaces later is additive but not free; teams that *expected* to plug a custom collection policy in v1 must wait. The A2 (Extensibility) reviewer will push back — see tensions.
- **Hot-path impact:** `none` (arguably a small *reduction* in virtual-dispatch per event-loop iteration since two v-calls become one branch).
- **Sanity test:** Walk `06-scenario-view.md §6.1.1-§6.1.3` with only `SINGLE_THREADED` + `SPLIT_COLLECTION`; verify each scenario maps to one of the two modes. Grep the codebase for concrete `IEventCollectionPolicy`/`IExecutionPolicy` implementations; confirm all four variants collapse to enum + params.

#### A9-P3: Remove collective primitives from `IHorizontalChannel`

- **Rationale:** Rule G2 / rubric check 2. `05-machine-level-registry.md:144-145` commits `IHorizontalChannel` to `barrier()` + `all_reduce()` while `09-open-questions.md:47-57` (Q4) explicitly asks whether collectives should live on the channel interface, be composed from `transfer()`, or live on a layered `ICollectiveOps`. Shipping the methods on the core interface *before* Q4 is resolved forces every backend (shared-mem, RegisterBank, TPUSH/TPOP, TCP, RDMA) to implement or stub them and pre-empts the open question.
- **Edit sketch:**
  - File: `docs/pypto-runtime-design/02-logical-view/05-machine-level-registry.md`
  - Location: lines 134-149 (the `IHorizontalChannel` definition).
  - Delta: remove `barrier(...)` and `all_reduce(...)` methods; leave `send`/`recv`/`poll`/`transfer`. Add a one-line forward reference: "Collective primitives are addressed in Q4; v1 composes collectives at the Orchestration level."
  - Update `docs/pypto-runtime-design/09-open-questions.md` Q4 current default to "(A) composition".
- **Trade-offs:**
  - Gain: 5 backends drop 2 stubs each; `IHorizontalChannel` is honestly minimal; Q4 stays open without commitment.
  - Give up: when HCCL / hardware-native collective wins materialize, collectives must be re-added via a separate extension interface (`ICollectiveOps`), not by amending the base.
- **Hot-path impact:** `none` (these methods are not on the hot path today; the single-node scenario never hits them).
- **Sanity test:** Re-walk the multi-node scenario in `06-scenario-view.md:35-58`; verify collective-free flow still works via `transfer()` + Orchestration. Confirm no runtime module depends on the removed methods.

#### A9-P4: Drop `SubmissionDescriptor::Kind`

- **Rationale:** Rule G2 / DRY. `07-task-model.md:26,69,205-223` make explicit that SINGLE and SPMD are specializations of Group. The runtime admission path in `02-scheduler.md:28` treats all Submissions identically. A tag on the descriptor that the runtime does not branch on is vestigial information.
- **Edit sketch:**
  - File: `docs/pypto-runtime-design/02-logical-view/09-interfaces.md`
  - Location: lines 99-120 (`SubmissionDescriptor` struct).
  - Delta: delete the `enum class Kind` and the `kind` member; reuse `spmd.has_value()` as the SPMD discriminator; the Single case is `tasks.size() == 1 && !spmd.has_value()`.
- **Trade-offs:**
  - Gain: one enum removed; one field removed; no admission-path branch on `Kind`.
  - Give up: any caller that currently reads `desc.kind` for logging/introspection must derive it from the other fields.
- **Hot-path impact:** `none` (admission path already does not branch on `Kind` for correctness).
- **Sanity test:** Search every admission / workspace / SPMD code path in `02-scheduler.md`, `modules/scheduler.md`, `07-task-model.md`; verify behavior reconstructable from `tasks.size()` and `spmd.has_value()`.

#### A9-P5: Unify admission enums (DRY)

- **Rationale:** Rule G2, DRY (§2.5). `09-interfaces.md:242-243` and `:293-297` both describe the outcome of the same admission decision; carrying two near-identical enums forces callers to translate — exactly the "forcing unrelated concepts together" / DRY-on-knowledge anti-pattern from rubric check 5.
- **Edit sketch:**
  - File: `docs/pypto-runtime-design/02-logical-view/09-interfaces.md`
  - Location: lines 242-243 and 293-297.
  - Delta: introduce a single `enum class AdmissionStatus { Admitted, WaitForWindow, Rejected }`; both `IResourceAllocationPolicy::should_admit` and `ResourceAllocationResult::status` use it.
- **Trade-offs:**
  - Gain: one enum; no translation step; consistent naming.
  - Give up: small mechanical rename in `modules/scheduler.md`.
- **Hot-path impact:** `none`.
- **Sanity test:** Replace both names with the unified enum in a scratch diff; confirm no semantic loss (the three states map 1:1).

#### A9-P6: Ship only `FUNCTIONAL` simulation in v1

- **Rationale:** Rule G2 / rubric check 2. ADR-011 (`08-design-decisions.md:433-498`) commits the v1 runtime design to three simulation modes, each requiring companion artifacts per AICore Function (`:487` — "up to three companion artifacts"). `FUNCTIONAL` covers host-side correctness (the concrete CI need). `PERFORMANCE` requires per-Function timing models — a compiler-tooling stream with no current owner; `REPLAY` requires a complete trace serializer/deserializer + state-machine driver — a debug tool without a v1 user. Deferring both to post-v1 is the KISS move.
- **Edit sketch:**
  - File: `docs/pypto-runtime-design/08-design-decisions.md` (ADR-011).
  - Location: §Decision (lines 451-463).
  - Delta: mark `PERFORMANCE` and `REPLAY` as "Deferred — opt-in v1.x"; make `FUNCTIONAL` the sole v1 `SIM` mode. Update `02-logical-view/10-platform.md:§2.8.1` and `07-cross-cutting-concerns.md:§7.2` accordingly. Move the deferred modes to `09-open-questions.md` as a follow-up.
- **Trade-offs:**
  - Gain: zero per-Function timing-model / trace-replay artifact obligation in v1; simpler `SIM` build; Fewer factory registrations; tooling scope contracts.
  - Give up: perf-modeling-without-hardware becomes a v1.x task; post-mortem replay is not available in v1.
- **Hot-path impact:** `none`.
- **Sanity test:** Confirm FR-10 in `01-introduction.md` is satisfied by `FUNCTIONAL` + `ONBOARD`. Confirm no scenario in `06-scenario-view.md` requires `PERFORMANCE` or `REPLAY`. Confirm the CI plan in `07-cross-cutting-concerns.md` covers only host-functional testing in v1.

#### A9-P7: Merge `SourceCollectionConfig` into `EventHandlingConfig`

- **Rationale:** Rule G2 / rubric check 5. Two config structs describing one behavior (event-loop delivery characteristics) + a third (`EventLoopDeploymentConfig`) fan out into a triple config surface with a documented "mutual consistency" validation pass (`09-interfaces.md:456`). Collapsing two of the three reduces the combinatorial space users must reason about.
- **Edit sketch:**
  - File: `docs/pypto-runtime-design/02-logical-view/09-interfaces.md`
  - Location: lines 358-427.
  - Delta: extend `EventHandlingConfig` with the `SourceCollectionConfig` fields (source list, per-source weight, max events per cycle); delete `SourceCollectionConfig`. Update `MachineLevelDescriptor.level_params` entry at `05-machine-level-registry.md:21`.
- **Trade-offs:**
  - Gain: one struct per Machine Level instead of two; the startup-time validation pass becomes trivial.
  - Give up: `EventHandlingConfig` grows in size — watch for accidental cache-line bloat (flag for A1).
- **Hot-path impact:** `none` (startup-time config, not hot path).
- **Sanity test:** Build a merged struct on paper; verify every current field fits with no lossy overlap; confirm startup validation is a type check rather than a cross-struct invariant.

#### A9-P8: Move per-Function simulation artifacts out of the runtime design

- **Rationale:** Rule G2 / SRP. ADR-011 Negative #1 (`08-design-decisions.md:487`) commits the runtime design to a specific per-Function artifact pipeline (device binary + timing model + CPU backend). Those artifacts are compiler/kernel-tooling deliverables; the runtime only needs to **consume** a binary + a leaf-engine factory. The design doc takes on scope it does not need, and that scope choice locks out compiler authors from iterating independently.
- **Edit sketch:**
  - File: `docs/pypto-runtime-design/08-design-decisions.md`
  - Location: ADR-011 Negative, line 487.
  - Delta: replace "Each AICore Function now has up to three companion artifacts…" with "Leaf `IExecutionEngine` implementations consume platform-specific artifacts defined in the compiler/kernel-tooling spec [link]." Cross-reference the frontend design for the concrete artifact formats.
- **Trade-offs:**
  - Gain: runtime design stays runtime-only; compiler/kernel authors own artifact schemas.
  - Give up: one extra cross-reference to maintain; if the compiler spec drifts, runtime integration spec must track it.
- **Hot-path impact:** `none`.
- **Sanity test:** After edit, run a structural review of `docs/pypto-runtime-design/` for any other section that specifies a per-Function artifact format; each should either be deleted or rephrased in consumption terms only.

## 5. Revisions of own proposals  (round ≥ 2 only)

*N/A in round 1.*

## 6. Votes on peer proposals  (round ≥ 2 only)

*N/A in round 1.*

## 7. Cross-aspect tensions

| pair | with_proposal | preferred_resolution |
|------|---------------|----------------------|
| A2 vs A9 | A9-P2, A9-P3 | A2 will argue removing `IEventCollectionPolicy` / `IExecutionPolicy` / collective-channel methods closes extension points (E4/OCP). Preferred resolution: **keep the extension points as additive opportunities, not as v1 obligations**. Concretely: document the interfaces as **"future extension hooks"** in an appendix, but do not require implementations to register them in v1. Factories default to the built-in path; a later ADR reintroduces the interface when a second implementation actually exists. This preserves YAGNI now and OCP later with a single edit. |
| A2 vs A9 | A9-P6 | A2 will flag deferring `PERFORMANCE`/`REPLAY` as "losing future capability". Preferred resolution: mark both as **deferred, not cancelled**; keep the section in `09-open-questions.md` with a trigger condition (first concrete consumer workload). This costs nothing now and leaves the door open. |
| A7 vs A9 | A9-P1, A9-P4 | A7 (Modularity/ISP) may argue `submit_group`/`submit_spmd` are role-focused interfaces. Preferred resolution: **role-focused ≠ virtual**. Keep the role-focused helpers as **free functions** above `runtime/` — they build a `SubmissionDescriptor` then call the single virtual entry. This satisfies ISP (narrow interface) and A9 (no three-overloads-for-one-operation). |
| A1 vs A9 | A9-P2 | A1 could argue that `IExecutionPolicy` is what enables the AICPU Interleaved mode's low-latency submit-and-execute cycle. Preferred resolution: **keep the Interleaved behavior but express it as a loop variant selected by an enum**, not a pluggable interface. The branch cost is identical (one enum check vs one vtable dispatch), and A1's hot-path budget is unchanged. |
| A1 vs A9 | A9-P7 | A1 may object that merging configs into `EventHandlingConfig` enlarges a hot-loop-reachable struct. Preferred resolution: place the merged struct in the Layer's cold-start config, not per-event; the hot path reads only the resolved dispatch function pointers, which are already cached. |

## 8. Stress-attack on emerging consensus  (round 3+ only)

*N/A in round 1.*

## 9. Status

- **Satisfied with current design?** partially — the runtime-boundary dependency model (§2.10) and the known-deviations discipline are excellent; the framework-extension surface (factories, policies, sim modes, collectives on the channel) over-indexes on future flexibility and violates G2/YAGNI in several places.
- **Open items expected in next round:** A9-P2 (deployment modes + policy interfaces) is the headline — expect A2 pushback; A9-P3 (collective methods on `IHorizontalChannel`) depends on Q4 resolution; A9-P6 (sim modes) depends on whether `PERFORMANCE`/`REPLAY` have concrete v1 consumers. A9-P1, A9-P4, A9-P5, A9-P7, A9-P8 are low-controversy cleanups and expected to converge quickly.
