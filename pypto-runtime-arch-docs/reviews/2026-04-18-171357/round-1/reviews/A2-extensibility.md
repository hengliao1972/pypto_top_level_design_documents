# Aspect A2: Extensibility & Evolvability — Round 1

## Metadata

- **Reviewer:** A2
- **Round:** 1
- **Target:** `docs/pypto-runtime-design/` (full doc set, incl. `modules/*.md`, ADRs, open questions)
- **Runtime Mode:** yes (A1 has double weight and hot-path veto; every proposal below carries a `hot_path_impact` label)
- **Timestamp (UTC):** 2026-04-19

## 1. Rubric Checks

| # | Check | Result | Evidence (file:line) | Rule(s) |
|---|-------|--------|----------------------|---------|
| 1 | Boundaries where change is anticipated are protected by interfaces | Pass | `02-logical-view.md:203-217` (Interfaces at a Glance); `modules/hal.md:14-19`; `modules/scheduler.md:101-142` (policy interfaces); `02-logical-view/05-machine-level-registry.md:32-56` (six factory kinds) | E1, D2 (DIP), §1.5 |
| 2 | New functionality added by new code (not modifying existing) | Weak | Positive: `02-logical-view/05-machine-level-registry.md:164-171` (topology as config); `modules/hal.md:438-443`; `modules/runtime.md:518-524`. Negative: `modules/runtime.md:134-143` (`LevelOverrides` is a closed struct — adding a new override edits this type, per Q6 `09-open-questions.md:74-84`); closed enums in `02-logical-view/09-interfaces.md:79-83` (`DepMode`), `modules/runtime.md:139` (`FailurePolicy`), `modules/hal.md:163-164` (`SimulationMode`), `modules/runtime.md:108-112` (`TransportBackend`, `NodeRole`, `Variant`). | E4, OCP, G2 |
| 3 | All APIs, schemas, protocols, configs versioned | Weak | Present: `modules/transport.md:87-98,152` (`MessageHeader::version`); `modules/runtime.md:340-349` (`PeerHandshakeRecord.protocol_version`); `modules/error.md:110` ("stable across minor versions"); `modules/profiling.md:104` (layout stability). Missing: `02-logical-view/09-interfaces.md:98-120` (`SubmissionDescriptor` no schema version); `modules/runtime.md:164-198` (`MachineLevelDescriptor` no version); `modules/runtime.md:97-152` (`DeploymentConfig` no version); `02-logical-view/10-platform.md:1-22` (`PlatformCaps` versioning implicit-only via "append bits"); `07-cross-cutting-concerns.md:67-79` (trace `Event` schema has no explicit `version` field at event level); `01-introduction.md:43` (FR-8 "stable C API" — no ABI version story documented). | E6 |
| 4 | Externalizable values live in config rather than hardcoded | Pass | `modules/runtime.md:92-152` (`DeploymentConfig`/`Timeouts`/`LevelOverrides`); `02-logical-view/05-machine-level-registry.md:21` (`LevelParams` bundle); `modules/hal.md:390-400` (HAL config table). Minor: `modules/hal.md:85` (`align ≥ 512` hardcoded); `modules/error.md` `max_cause_depth` default hardcoded in doc prose. | E3, X8 |
| 5 | Incremental migration plan from current architecture | Weak | `00-index.md:25-33` traces source material into views but there is no step-by-step reversible migration plan from the existing monolithic runtime (`DistScheduler`/`PTO2Runtime`/`AicpuExecutor`, ADR-003:84-87) to the new hierarchy. `appendix-b-codebase-mapping.md` is cited but is a static mapping, not a phased transition. `10-known-deviations.md` lists no E5 deviation or exception. `09-open-questions.md:86-96` (Q7) leaves DSL backward-compat placement unresolved. | E5, NFR-5 |
| 6 | Backward-compat guarantees documented for interface evolution | Weak | Partial: `modules/error.md:110,172`, `modules/transport.md:152`, `modules/hal.md:291-293` (DeviceCaps "append-only"). Gaps: no BC statement for `ISchedulerLayer` (core contract, `02-logical-view/09-interfaces.md:9-62`); no BC statement for `MachineLevelDescriptor`, `SubmissionDescriptor`, `TaskDescriptor`, `LevelParams`, `DeploymentConfig`; Python class set in `modules/bindings.md:28-54` has no deprecation policy. | E2 |

## 2. Pros

- The design is, at its spine, an OCP/DIP-first architecture. Rule NFR-2 is lifted to a first-class requirement ("Adding a new hardware level requires only implementing six component interfaces and registering — no existing code modified", `01-introduction.md:52`), and the Machine Level Registry + factory pattern is the concrete realization (`02-logical-view/05-machine-level-registry.md:164-171`; ADR-002 `08-design-decisions.md:42-73`). Cites E4, D2.
- Strategy-pattern policy seams (`ITaskSchedulePolicy`, `IWorkerSelectionPolicy`, `IResourceAllocationPolicy`) are optional with defaults — a classic OCP-friendly shape where the critical path is unchanged when no policy is supplied, and new behaviour lands in new `.cpp` files (`modules/scheduler.md:547-553`; ADR-008 `08-design-decisions.md:241-295`). Cites E4.
- Simulation is factored as a leaf-only `IExecutionEngine` factory, so a fourth Mode ("HYBRID") is *documented as* an additive registration without touching scheduler/memory/transport code (`02-logical-view/10-platform.md:41`; ADR-011 `08-design-decisions.md:481`). Cites E4, D2.
- Control-plane separation (Vertical vs Horizontal; ADR-004 `08-design-decisions.md:110-141`) gives two independently evolvable contracts where a new inter-node transport (RDMA, CXL) plugs in without touching intra-node paths (`modules/transport.md:442-447`). Cites E1, D4.
- Inter-node protocol is explicitly versioned with a handshake-based negotiation path (`modules/transport.md:87-98,152,340-344`; `modules/runtime.md:340-349`). Cites E6, E2.
- The `ErrorCode` taxonomy commits to numeric-code stability across minor versions and append-only evolution, with deprecated codes retaining their number (`modules/error.md:110,172`). Cites E2, E6.
- Deferred-work discipline: ADR-012/013 plus `09-open-questions.md:142-190` (Q12–Q14) document what was intentionally *not* built into v1 and the triggers that would re-open them — this is an incremental-evolution plan *within* the runtime boundary. Cites E5, G3.

## 3. Cons

- **No schema version field on core data contracts.** `SubmissionDescriptor` (`02-logical-view/09-interfaces.md:98-120`), `MachineLevelDescriptor` (`modules/runtime.md:164-198`), `DeploymentConfig`/`PeerSpec`/`LevelOverrides` (`modules/runtime.md:97-152`), `LevelParams`, and the Chrome-Trace `Event` schema (`07-cross-cutting-concerns.md:67-79`) are all ABI / on-disk surfaces shared between compiled `bindings/`, user C++ code, traces, and (for DeploymentConfig) deployment descriptors, yet none carry a `schema_version`/`abi_version` field. Only the wire protocol between peers has one. Violates E6.
- **`LevelOverrides` is a closed typed struct** (`modules/runtime.md:134-143`) with hardcoded fields (`instance_count`, `slot_pool_size`, `max_outstanding_submissions`, `partitioner`, `failure_policy`, `event_loop`). Adding a new override requires editing the struct and recompiling `runtime/` — the very anti-pattern Q6 (`09-open-questions.md:74-84`) is trying to avoid for `LevelParams`. The `extra: map<string,string>` field is an escape hatch but is stringly-typed, defeating validation. Violates E4 / OCP.
- **Closed enums at semantically-load-bearing extension points.** `DepMode` (BARRIER/DATA/NONE; `02-logical-view/09-interfaces.md:79-83`), `FailurePolicy` (`modules/runtime.md:139`), `SimulationMode` (`modules/hal.md:163-164`), `Variant`, `NodeRole`, `TransportBackend` (`modules/runtime.md:100-112`). Each enum is hot-coded into a central `switch` in TaskManager / IPlatformFactory / runtime composition. A new DepMode (e.g., `RAW_CROSS_SUBMISSION` per Q12 option B) or a new FailurePolicy requires editing the enum and every switch that consumes it. Violates E4 / OCP. (Note: `DepMode` is hot-path; see tensions §7.)
- **No interface-evolution policy statement.** No document in the set enumerates, for each public interface (`ISchedulerLayer`, `IMemoryManager`, `IMemoryOps`, `IVerticalChannel`, `IHorizontalChannel`, the six factory interfaces, Python surface in `modules/bindings.md:28-54`), whether it is (a) frozen for v1, (b) open for additive evolution, or (c) evolved via versioned sub-interface. Violates E2, weakens E6.
- **No migration plan from the current runtime.** ADR-001/002/003/008/010/012 each describe the destination but none describes the reversible step sequence from the current `DistScheduler`+`PTO2Runtime`+`AicpuExecutor` (cited in ADR-003 context `08-design-decisions.md:84-87`) to the new hierarchy. No feature flags, no interim shims, no rollback. `10-known-deviations.md` records none of this. Violates E5; contradicts Rule E5 "Never plan a big-bang rewrite without a fallback."
- **DSL backward-compat is an unresolved open question,** not a plan (`09-open-questions.md:86-96`, Q7). NFR-5 commits the runtime to back-compat but does not specify where the aliasing lives or how deprecation proceeds. Violates E2/E5.
- **MessageType extension requires central edits and a version bump** (`modules/distributed.md:460`, `modules/transport.md:445`). The doc describes the path but there is no pluggable `IDistributedProtocolHandler` registry — new message types edit the dispatch switch. Weakly violates E4.
- **Policy interfaces are synchronous-only by contract with no negotiated upgrade path** (`09-open-questions.md:127-138`, Q11). An external cluster-manager-driven policy cannot plug in without the runtime gaining an async-policy mode. This is acknowledged as open but there is no extension seam reserved (e.g., a future `IAsyncTaskSchedulePolicy`). Weakens E1 at the most likely evolution point.
- **Trace event schema has no explicit version field** (`07-cross-cutting-concerns.md:67-79`); `modules/profiling.md:104` promises "layout stability across patch releases" and bumps recorded in header metadata, but the per-event `Event` struct has no field that a `REPLAY`-mode consumer (ADR-011) can key off for forward/backward compatibility. Violates E6.
- **Registry is frozen at init by design** (Q5 `09-open-questions.md:60-70`, ADR-002 `08-design-decisions.md:72-73`). This is an intentional E5 scope limit, not a defect, but it means the "evolvability at runtime" ceiling is explicit — worth recording as a known deviation rather than an open question.

## 4. Proposals

| id | severity | summary | affected_docs | hot_path_impact | trade_offs | sanity_test |
|----|----------|---------|---------------|-----------------|------------|-------------|
| A2-P1 | high | Add `schema_version: uint16_t` to every cross-module data contract (`SubmissionDescriptor`, `MachineLevelDescriptor`, `DeploymentConfig`, `LevelParams`, trace `Event`, `ErrorContextPod`). Declare a compatibility policy (minor = additive, major = breaking + handshake). | `02-logical-view/09-interfaces.md`, `modules/runtime.md`, `07-cross-cutting-concerns.md`, `modules/profiling.md`, `modules/error.md`, new `appendix-c-compatibility.md` | none | Gain E6 compliance, tool-readable trace/ABI boundaries. Give up ~2–4 B per descriptor; policy maintenance. | Grep the doc set for each listed type; every definition has a `schema_version` field; new `appendix-c-compatibility.md` enumerates the BC policy per type. |
| A2-P2 | high | Replace `LevelOverrides` typed fields with a **schema-registered key/value surface** (per Q6 option C). Each level registers its schema via the Machine Level Registry; `runtime/deployment_parser` validates `LevelOverrides` against the schema at init. | `modules/runtime.md:134-143`, `02-logical-view/05-machine-level-registry.md:7-31`, `09-open-questions.md:74-84` (mark Q6 as resolved-to-C) | none (init-time only) | Gain OCP: new per-level knobs ship with the level's own factory package. Give up compile-time typed access in `LevelOverrides`; adds a small validation step at init. | Add a fictional new level with a new knob in a test; existing `runtime/` source compiles unchanged; the test exercises the knob end-to-end. |
| A2-P3 | medium | Convert closed enums (`FailurePolicy`, `TransportBackend`, `SimulationMode`, `NodeRole`) into **string IDs resolved through a registry** at init (same pattern as partitioner, already string-typed in `LevelOverrides.partitioner`). Keep `DepMode` and `Variant` closed and document them as intentionally closed in the new BC policy (`DepMode` is hot-path and must be a small dispatch). | `modules/runtime.md:108-143`, `modules/hal.md:163-164`, `modules/distributed.md` (FailurePolicy), `08-design-decisions.md` (new ADR or ADR-011 extension) | none for the ones opened (init-time resolution → static fn ptr table); DepMode stays closed ⇒ `none` on the hot path | Gain OCP on deployment-time extension points. Give up compile-time exhaustiveness checking for the opened enums. | Register a fourth `SimulationMode` (e.g. `HYBRID`) in a test that lives entirely under `hal/a2a3/sim/`; runtime init picks it up without any edit to `modules/runtime.md` / `modules/hal.md` sources. |
| A2-P4 | high | Add **§3.5 Migration & Transition Plan** to `03-development-view.md` (or promote `appendix-b-codebase-mapping.md` to include it). Must: (a) map current classes → new modules; (b) define at least three reversible phases with feature flags (e.g., `PTO_USE_NEW_SCHEDULER`, default off → canary → on); (c) specify rollback criteria; (d) enumerate interim shims to keep current callers working. | `03-development-view.md`, `appendix-b-codebase-mapping.md`, `10-known-deviations.md` (drop any "big-bang" implicit stance) | none | Gain E5 compliance and de-risked rollout. Give up speed of a clean-room replace. | A reviewer can trace from each current class (`DistScheduler`, `PTO2Runtime`, `AicpuExecutor`) to a specific phase and a feature flag; the plan names a rollback gate for each phase. |
| A2-P5 | high | Add a **§9 Interface Evolution & Backward-Compat Policy** table to the new `appendix-c-compatibility.md` (or as a section in `03-development-view.md`) listing every public interface with a BC class: `stable-frozen`, `stable-additive`, `versioned-subinterface`, `unstable`. Include deprecation procedure (e.g., "mark, ship warning for one minor release, remove"). | new `appendix-c-compatibility.md`, `03-development-view.md`, `modules/*.md` (add a pointer from each `## 2. Public Interface` section) | none | Gain E2 compliance; implementers of custom Machine Levels get a binding contract. Give up flexibility to mutate interfaces ad hoc between versions. | Cross-reference every interface listed in `02-logical-view.md` Interfaces-at-a-Glance with the new policy table; no interface is absent. |
| A2-P6 | medium | Promote `IDistributedProtocolHandler` to a **first-class registry-dispatched interface**: new `MessageType` values register a handler at init; the transport hot path performs a single table lookup keyed by `MessageType` rather than editing a central switch. | `modules/distributed.md`, `modules/transport.md:77-98,460`, `02-logical-view/09-interfaces.md:133-152` | none on the hot path (O(1) function-pointer table keyed by `MessageType`; table is init-only, no virtual dispatch, no allocation) | Gain OCP on inter-node protocol evolution. Give up a single `switch` that the optimizer can inline across types. Two-tier fallback: table lookup with a hardcoded short-circuit for the top-N builtin types keeps the common-path latency identical. | Register a new `MessageType::REMOTE_TOPOLOGY_UPDATE` in a test without editing `modules/distributed.md` source; peer-to-peer smoke test exchanges the new message and dispatches to the registered handler. |
| A2-P7 | medium | Reserve an **async-policy extension seam** now: define (but do not yet implement) `IAsyncTaskSchedulePolicy` / `IAsyncWorkerSelectionPolicy` as a sub-interface of the sync ones, and document the upgrade path from Q11. | `02-logical-view/09-interfaces.md:154-260`, `modules/scheduler.md:101-148`, `09-open-questions.md:127-138` | none today (interface reservation only); if adopted later, behind a two-tier path (sync fast path remains default, async-aware wrapper opts in per Layer) | Gain E1/E5 compliance at the most likely evolution point. Give up simplicity of "one sync interface"; the sub-interface must be kept dead-code until needed. | Future reviewer can trace Q11 option (B) or (C) to an already-existing interface shape; no runtime/scheduler edits are needed to introduce the first async policy. |
| A2-P8 | medium | Record, in `10-known-deviations.md`, the **intentional closures** revealed by this review — Registry frozen at init (Q5), policies synchronous-only (Q11), `DepMode` closed enum — each with rationale and a revisit trigger. Preserves E1/E5 while being explicit about scope. | `10-known-deviations.md`, `09-open-questions.md:60-70,127-138,142-155` | none | Gain explicit E-rule exception bookkeeping (per Rule Exceptions procedure in `04-agent-rules.md`). Give up the optimistic read of "nothing violates E1". | Each listed deviation has the four parts required by `04-agent-rules.md` §Rule Exceptions (which rule, why, mitigation, where). |
| A2-P9 | low | Add an explicit **`trace_schema_version: uint16_t`** in the `Event` schema header (not per event) and require the profiling collector to emit it; consumers (including `REPLAY` mode per ADR-011) must reject unknown major versions. | `07-cross-cutting-concerns.md:64-79`, `modules/profiling.md:104`, `modules/hal.md:117` (REPLAY engine must verify) | none (header-only, not on the per-event hot path) | Gain safe REPLAY forward/back compat. Give up ~2 B in the trace file header. | A hand-crafted trace file with a future `trace_schema_version+1` is rejected by the `REPLAY` leaf engine with `ProtocolVersionMismatch`. |

### Proposal detail

#### A2-P1: Version every public data contract

- **Rationale:** E6 requires APIs, schemas, protocols, and configs to be versioned. The design versions only the inter-node wire protocol (`modules/transport.md:87-98,152`) and the `PeerHandshakeRecord` (`modules/runtime.md:340-349`). Everything else that crosses a stability boundary — `SubmissionDescriptor` (constructed in Python, serialized in `RemoteSubmitPayload`, `modules/transport.md:104-112`), `MachineLevelDescriptor` (shared between compiled-in factories and user-supplied factories), `DeploymentConfig` (parsed from configuration), `Event` (written to disk, consumed by `REPLAY`), `ErrorContextPod` (device-side POD crossing the host boundary, `modules/error.md:164`) — has no version field. NFR-1 "zero-overhead profiling" and X9 latency budgets are unaffected because version validation is done only at boundary handshakes (bindings entry, init, trace open), not on the per-submit hot path.
- **Edit sketch:**
  - File: `02-logical-view/09-interfaces.md`
  - Location: `struct SubmissionDescriptor` definition (~line 98)
  - Delta: add `uint16_t schema_version = SUBMISSION_SCHEMA_V1;` and note that the field is validated only at the Python↔C and inter-node boundaries, never on the in-process hot path.
  - Add analogous fields in `modules/runtime.md:164-198` and `modules/runtime.md:97-152`, and `version` on `Event` header in `07-cross-cutting-concerns.md`.
  - Add a new file `appendix-c-compatibility.md` with a 2-column table (type, version policy).
- **Trade-offs:**
  - Gain: E6 compliance; `REPLAY`, Python bindings, and peer nodes can reject incompatible formats with a specific error, not an undefined-behavior crash.
  - Give up: a small amount of bytes per descriptor; ongoing discipline to bump versions.
- **Sanity test:** Grep the doc set for `struct SubmissionDescriptor` / `MachineLevelDescriptor` / `DeploymentConfig` / `Event` / `ErrorContextPod` — each definition has a version field; `appendix-c-compatibility.md` lists each with a policy class.

#### A2-P2: Schema-registered `LevelOverrides`

- **Rationale:** E4 / OCP require extension without modification. `modules/runtime.md:134-143` defines `LevelOverrides` as a typed struct with hardcoded knobs; adding (e.g.) a `pipeline_depth` override for a new Machine Level requires editing this struct. Q6 (`09-open-questions.md:74-84`) acknowledges this for `LevelParams`; the same resolution (option C: type-erased with schema validation) applies to `LevelOverrides`. The `extra: map<string,string>` escape hatch is stringly-typed and unvalidated, which defeats E3 ("schema-validated at startup") and X8.
- **Edit sketch:**
  - File: `modules/runtime.md`
  - Location: `struct LevelOverrides` (~line 134-143)
  - Delta: replace typed fields with `std::unordered_map<std::string, ValueWithSchemaRef> overrides;` and add to `MachineLevelDescriptor` a `const LevelOverridesSchema* overrides_schema;`. Document that validation runs inside `deployment_parser.cpp` against the level's registered schema before any Layer is constructed.
- **Trade-offs:**
  - Gain: a new Machine Level that ships in its own translation unit (per ADR-002) can carry its own knobs without editing `runtime/`.
  - Give up: compile-time typed field access in `LevelOverrides`; consumers do a schema-keyed lookup. Validation cost at init is negligible.
- **Sanity test:** Author a fictitious Machine Level (e.g. `"Accelerator"`) with a per-level knob not anticipated by `modules/runtime.md`. Register it and its schema in a test. `DeploymentConfig` containing that knob parses cleanly; `runtime/` source compiles unchanged.

#### A2-P3: Open the extension-point enums, keep `DepMode` closed

- **Rationale:** E4 / OCP. `FailurePolicy`, `TransportBackend`, `SimulationMode`, `NodeRole` are all deployment-time properties whose lifetime is "init of one Runtime"; keeping them as C++ enums forces every future extension to edit the scheduler, runtime, or HAL. `partitioner` already uses the string-ID pattern (`modules/runtime.md:138`) — the same pattern generalizes. `DepMode`, by contrast, is hot-path (`modules/scheduler.md:264`, `02-logical-view/07-task-model.md:53-57`); widening it would collide with A1 (X3, X9). Preempt A1: keep `DepMode` closed and record the reason in the BC policy (A2-P5).
- **Edit sketch:**
  - Files: `modules/runtime.md:108-143`, `modules/hal.md:163-164`, `modules/distributed.md` (FailurePolicy section)
  - Delta: replace `FailurePolicy`, `TransportBackend`, `SimulationMode`, `NodeRole` with string-keyed factory registration; resolve strings to `IFoo*` during `Runtime::init`. Add a new line in ADR-011 noting `SimulationMode` is now an open factory, not an enum. Leave `DepMode` closed; document rationale.
- **Trade-offs:**
  - Gain: OCP on deployment-time extension points; a new transport backend or failure policy ships in its own translation unit.
  - Give up: the compiler no longer guarantees every `switch` is exhaustive over these values. Mitigated by requiring an explicit default arm that calls `unreachable()` with the offending string.
- **Sanity test:** Register a new failure-policy string ("`retry-alternate-with-quarantine`") in a test. Runtime honors it at init; no edits to any `modules/*.md` source file outside the test directory.

#### A2-P4: Migration & Transition Plan

- **Rationale:** E5 ("Never plan a big-bang rewrite without a fallback"). The design describes the destination but not the path. The current runtime has three live schedulers (ADR-003 context, `08-design-decisions.md:84-87`). `appendix-b-codebase-mapping.md` is a static table, not a phased plan.
- **Edit sketch:**
  - File: `03-development-view.md`
  - Location: new `§3.5 Migration & Transition Plan`
  - Delta: add three-to-five phase table (columns: phase, feature flag, current-class retired, new-class replacing, rollback gate, validation criterion). Cross-link from `appendix-b-codebase-mapping.md` and from `10-known-deviations.md` (remove any implicit big-bang stance).
- **Trade-offs:**
  - Gain: E5 compliance; predictable, reviewable rollout; explicit rollback guarantee.
  - Give up: some of the clean-room narrative in the new docs (needs to accept that the old runtime coexists during transition).
- **Sanity test:** Every named current class in ADR-003/ADR-005 context appears in the phase table with a retirement phase and a rollback criterion.

#### A2-P5: Interface Evolution & Backward-Compat Policy

- **Rationale:** E2 requires documented BC guarantees for every interface. Currently only `ErrorCode` numbers (`modules/error.md:110`), the inter-node protocol (`modules/transport.md:152`), and `DeviceCaps` bits (`modules/hal.md:291-293`) commit to a stability rule. The rest (notably `ISchedulerLayer`, the single most-implemented interface, `02-logical-view/09-interfaces.md:9-62`) has no statement.
- **Edit sketch:**
  - File: new `appendix-c-compatibility.md`
  - Location: add `## Interface Evolution Policy` table
  - Delta: row per public interface from `02-logical-view.md` Interfaces-at-a-Glance (≈ 25 rows); columns = {Interface, Stability Class (`stable-frozen` / `stable-additive` / `versioned-subinterface` / `unstable`), Deprecation Procedure}. Cross-link from each module doc's `## 2. Public Interface` section.
- **Trade-offs:**
  - Gain: E2 compliance; external Machine Level authors know which interfaces they can rely on across releases.
  - Give up: loses the freedom to quietly mutate an interface between releases; every revision becomes a policy-checked change.
- **Sanity test:** Grep `02-logical-view.md:203-217` Interfaces-at-a-Glance table entries; each appears in `appendix-c-compatibility.md`.

#### A2-P6: Pluggable `IDistributedProtocolHandler` registry

- **Rationale:** E4. `modules/distributed.md:460` and `modules/transport.md:445` describe extension-by-central-edit ("add handler in `IDistributedProtocolHandler`"). Every new `MessageType` touches the same switch. A registry-dispatched handler (table indexed by `MessageType`) turns this into new-code-only addition.
- **Edit sketch:**
  - File: `02-logical-view/09-interfaces.md` (owner of distributed interfaces)
  - Location: new sub-section `§2.6.2.1 Distributed Message Handler Registry`
  - Delta: define `using DistributedMessageHandler = void(*)(const MessageHeader&, const uint8_t* payload, size_t n);`; registered in `DistributedProtocolDispatcher` keyed by `MessageType`. Peers advertise supported types in `HANDSHAKE`. Specify that the table is initialized at `Runtime::init`, then read-only.
- **Trade-offs:**
  - Gain: OCP; new `MessageType` ships alongside its handler, no central switch edit.
  - Give up: one indirect call per message. Mitigation: inline the hot 3–4 builtins via a two-tier short-circuit (A1-friendly fallback, measured against `< 10 μs` remote-submit budget in `07-cross-cutting-concerns.md:193`).
- **Sanity test:** New `REMOTE_TOPOLOGY_UPDATE` type compiles and dispatches via registration only; no edits to `modules/distributed.md` outside the new registration call site. Inter-node round-trip ≤ budget.

#### A2-P7: Reserve async-policy extension seam

- **Rationale:** E1 says "place abstraction boundaries where change is likely." Q11 (`09-open-questions.md:127-138`) is the most likely evolution point for the scheduler and today has no shape reserved — only a bullet in "future work." Declaring a sub-interface shape (even if dead-code) gives implementers of Q11 option (B)/(C) a starting edge.
- **Edit sketch:**
  - File: `02-logical-view/09-interfaces.md`
  - Location: new `§2.6.3.A Async Policy Extension (reserved)`
  - Delta: declare `class IAsyncTaskSchedulePolicy : public ITaskSchedulePolicy` with `Status rank_ready_tasks_async(TaskHandle[], size_t, Callback)` and a note that the scheduler uses `dynamic_cast` at Layer construction only.
- **Trade-offs:**
  - Gain: E1/E5 at the known growth point; Q11 becomes an implementation task, not a re-design.
  - Give up: interface surface enlargement today; risk of declaring an interface that ultimately needs different shape.
- **Sanity test:** An experimental async policy in a test compiles against the reserved interface without any edit to `modules/scheduler.md:101-148`.

#### A2-P8: Record intentional closures as known deviations

- **Rationale:** Rule E1 and the Rule Exceptions procedure in `04-agent-rules.md:140-148`. The design intentionally closes: Registry freeze (Q5), synchronous-only policies (Q11), closed `DepMode` (implied by hot-path considerations, `modules/scheduler.md:264`). These are reasoned choices but are not recorded under `10-known-deviations.md` or as ADRs; they show up only as open questions. Converting them into explicit deviations with revisit triggers is the least-invasive E1/E5 compliance path.
- **Edit sketch:**
  - File: `10-known-deviations.md`
  - Location: append three deviation entries (matching the existing four-part template: rule, violation, justification, mitigation).
  - Delta: entries for "Registry frozen at init", "Synchronous policies only", "`DepMode` closed enum (hot path)". Cross-link back from `09-open-questions.md` Q5, Q11, Q12.
- **Trade-offs:**
  - Gain: explicit E-rule exception bookkeeping; Rule Exceptions procedure satisfied.
  - Give up: must keep revisit triggers current when workloads evolve.
- **Sanity test:** Each of Q5, Q11, Q12 has a matching entry in `10-known-deviations.md`.

#### A2-P9: Versioned trace schema

- **Rationale:** E6. `REPLAY` mode (ADR-011) depends on traces captured from `ONBOARD`. If the trace schema evolves, a `REPLAY` engine reading an older trace cannot self-detect incompatibility without a version field. `modules/profiling.md:104` describes "version bumps for breaking changes recorded in header metadata written by sinks" but the `Event` / trace-file header definition in `07-cross-cutting-concerns.md:64-79` shows no field.
- **Edit sketch:**
  - File: `07-cross-cutting-concerns.md`
  - Location: `§7.2.2 Trace Event Model`
  - Delta: prepend a one-off trace-file header struct (`magic`, `trace_schema_version`, `platform`, `mode`) and reference it from `modules/profiling.md:104` and `modules/hal.md:117` (`replay_engine` now required to validate).
- **Trade-offs:**
  - Gain: `REPLAY` is safely forward/backward-compatible.
  - Give up: a few bytes of header overhead per trace file.
- **Sanity test:** A trace with `trace_schema_version+1` is rejected by `replay_engine` with `ProtocolVersionMismatch`; a matching-version trace is accepted.

## 5. Revisions of own proposals

Not applicable in round 1.

## 6. Votes on peer proposals

Not applicable in round 1.

## 7. Cross-aspect tensions

| pair | with_proposal | preferred_resolution |
|------|---------------|----------------------|
| A2 vs A1 | A2-P1 (version field on `SubmissionDescriptor`) | Two-tier path: validate `schema_version` only at the bindings↔C boundary (Python → C++ entry) and at the inter-node handshake (`RemoteSubmitPayload`), never on the per-submit in-process hot path. A compile-time `static_assert(SUBMISSION_SCHEMA_VERSION == EXPECTED)` at the C++ call site completes the guarantee without a runtime branch. |
| A2 vs A1 | A2-P3 (opening `FailurePolicy`/`TransportBackend`/`SimulationMode`/`NodeRole`); closed-keep for `DepMode` | Only init-time extension points are opened (resolved to `IFoo*` once, stored in `Runtime`). `DepMode` stays a small closed `enum class` so the admission-time dispatch (`modules/scheduler.md:333-357`) remains a jump table with 3 arms. Recorded as an intentional deviation in A2-P8. |
| A2 vs A1 | A2-P6 (dispatcher registry for `MessageType`) | Fast-path short-circuit on the top-N builtin `MessageType`s (if-ladder at the call site); fallback table lookup for registered extensions. Keeps the common-path latency within the `< 10 μs` remote-submit budget (`07-cross-cutting-concerns.md:193`). |
| A2 vs A9 | A2-P7 (reserved `IAsyncTaskSchedulePolicy`) | Declare the sub-interface in the logical view but do not wire a dispatcher until Q11 resolves. This is minimum-surface reservation — A9 (YAGNI) objection is limited because the interface is documentation-only until an async-policy implementer arrives. If A9 still objects, compromise: publish the design under `09-open-questions.md` Q11 as a "reserved shape" rather than in `09-interfaces.md`. |
| A2 vs A9 | A2-P2 (schema-registered `LevelOverrides`) | The same pattern already exists for `LevelParams` per Q6 option C — A2-P2 generalizes a pattern the design *already converged on*, so A9 "new plugin infrastructure" objection does not apply. One mechanism, two consumers. |
| A2 vs A7 | A2-P6 (pluggable `IDistributedProtocolHandler`) | Both aspects want narrow surfaces; the registry adds **one** new seam (`DistributedMessageHandler`) rather than per-type interfaces, which A7 should prefer over a fat switch that grows with every new type. |

## 8. Stress-attack on emerging consensus

Not applicable in round 1.

## 9. Status

- **Satisfied with current design?** partially — strong at the structural extension points (interfaces, factories, HAL, policies); weak at versioning discipline, deployment-config extensibility, and migration planning.
- **Open items expected in next round:**
  - A2-P1 (schema versioning) — expect A1 scrutiny on the hot-path impact of any per-submit validation; resolve via the two-tier handshake-only stance sketched above.
  - A2-P2 (schema-registered `LevelOverrides`) — expect A9 pushback on "plugin infrastructure"; respond with "already converged via Q6".
  - A2-P3 (open enums) — expect A1 to veto any opening of `DepMode`; agreed in advance.
  - A2-P4 (migration plan) — expect A5 / A7 alignment (reliability and modular rollout); no tension.
  - A2-P5 (BC policy doc) — expect A4 (consistency) alignment; no tension.
  - A2-P6 (dispatcher registry) — expect A1 review on per-message dispatch cost; mitigation already scoped.
  - A2-P7 (reserved async-policy seam) — expect A9 YAGNI pushback; compromise path available.
  - A2-P8 (known-deviations bookkeeping) — expect no tension.
  - A2-P9 (trace schema version) — expect no tension.
