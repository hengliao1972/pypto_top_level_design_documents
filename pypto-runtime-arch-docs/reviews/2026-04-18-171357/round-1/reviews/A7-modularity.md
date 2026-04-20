# Aspect A7: Modularity & SOLID — Round 1

## Metadata

- **Reviewer:** A7
- **Round:** 1
- **Target:** `/data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/`
- **Runtime Mode:** yes
- **Timestamp (UTC):** 2026-04-19T00:00:00Z

## 1. Rubric Checks

| # | Check | Result | Evidence (file:line) | Rule(s) |
|---|-------|--------|----------------------|---------|
| 1 | SRP — each module has a single reason to change | Weak | `docs/pypto-runtime-design/03-development-view.md:107-119` lists responsibilities, but `runtime/` bundles Runtime+DistributedRuntime+Registry+deployment_parser+layer_factory+wiring+elision+version (`docs/pypto-runtime-design/modules/runtime.md:215-234`); `transport/` also owns distributed protocol payload structs (`docs/pypto-runtime-design/modules/transport.md:103-150`), which are protocol semantics, not framing. | D1, D3, D5 |
| 2 | DIP — high-level modules depend on abstractions only | Weak | `core/` depends on `hal/` for opaque handle types (`docs/pypto-runtime-design/modules/core.md:15-17`); cascades `hal/` into every consumer of `core/`. `distributed/` explicitly depends on `scheduler/` **base managers** (`docs/pypto-runtime-design/modules/distributed.md:15`), i.e. concrete `TaskManager/WorkerManager/ResourceManager`, not `ISchedulerLayer`. | D2 |
| 3 | DAG — no cycles | **Fail** | `docs/pypto-runtime-design/03-development-view.md:200-213` states `distributed/` is *depended on by* `scheduler/` ("for cross-node coordination") AND `scheduler/` is *depended on by* `distributed/` (via `scheduler/` base managers cited in `docs/pypto-runtime-design/modules/distributed.md:15`). Two directed edges between the same two nodes = cycle candidate. The graphic on `docs/pypto-runtime-design/03-development-view.md:177-196` also puts both modules at the same level with a `runtime/` arrow that obscures the direction. | D6 |
| 4 | ISP — role-focused interfaces | Weak | `ISchedulerLayer` is a fat umbrella of 18+ methods spanning identity, wiring setters, submit overloads, dep/child notifications, scope, lifecycle, status (`docs/pypto-runtime-design/02-logical-view/09-interfaces.md:10-62`, `docs/pypto-runtime-design/modules/core.md:31-78`). Every consumer compiles against the full surface even when it only calls `notify_child_complete` or only `submit`. | ISP, D4 |
| 5 | Cohesion — all parts serve the same purpose | Weak | `runtime/` mixes composition, registry, deployment parsing, elision routing, handshake, and version — seven `src/*.cpp` siblings (`docs/pypto-runtime-design/modules/runtime.md:225-234`). `transport/` mixes reliable-framed-byte transport with protocol payload schema (`docs/pypto-runtime-design/modules/transport.md:102-150`). | D5 |
| 6 | Coupling — narrow, stable interfaces | Weak | `ISchedulerLayer` coupling to `memory/` and `transport/` via the 5 `set_*` wiring methods leaks those modules' interface pointers into `core/` (`docs/pypto-runtime-design/modules/core.md:40-45`); the code-level DAG in `docs/pypto-runtime-design/03-development-view.md:202-211` does not list `memory/` or `transport/` as dependencies of `core/`. Either forward-decl is used and the dependency is invisible, or a latent DAG violation. | D2, D4, D6 |

## 2. Pros

- **10-module decomposition with explicit per-module SRP statements** — every module has a "Single responsibility" line and a responsibilities table, e.g. `docs/pypto-runtime-design/03-development-view.md:105-119`, `docs/pypto-runtime-design/modules/hal.md:9-11`, `docs/pypto-runtime-design/modules/core.md:9-11`. Satisfies D1/D3 at the claim level.
- **Strategy pattern for scheduling policies (ISP, OCP)** — three role-specific interfaces `ITaskSchedulePolicy / IWorkerSelectionPolicy / IResourceAllocationPolicy` (`docs/pypto-runtime-design/02-logical-view/09-interfaces.md:158-261`). Each is small and focused; consumers depend only on the policy they need. D4, E4.
- **Clean platform abstraction (HAL)** — `hal/` has zero upstream dependencies and `onboard/`/`sim/` confine vendor SDK symbols (`docs/pypto-runtime-design/modules/hal.md:238-247`). Enforces X7 and, by extension, D2/D4 at the platform boundary.
- **`error/` is a true leaf** — "Depends on: Standard library only. No other runtime modules." (`docs/pypto-runtime-design/modules/error.md:15-17`). Cross-cutting module with stable ABI codes; clean D1/D6.
- **Scheduler internal decomposition (TaskManager/WorkerManager/ResourceManager)** is documented as an internal concern behind a stable `ISchedulerLayer` (`docs/pypto-runtime-design/02-logical-view/02-scheduler.md:9-14`, ADR-003, ADR-008). Preserves external D4.
- **Dedicated `IPartitioner` and `IDistributedProtocolHandler` abstractions** for distributed scheduling (`docs/pypto-runtime-design/modules/distributed.md:36-84`). The `IDistributedProtocolHandler` in particular is a clean seam — protocol decode produces events only, no state mutation. D2, E1.
- **Dependency-Model §2.10 is a specification module, not a new code module** (`docs/pypto-runtime-design/02-logical-view/12-dependency-model.md:136-140`). Keeps `scheduler/` and `memory/` cohesive without introducing a spurious "dependency" module. D3, G2.
- **Explicit logical DAG diagrammed and claimed invariant** — `docs/pypto-runtime-design/02-logical-view.md:79-133`, with foundation/contract/assembler layering. D6 at the concept level.
- **Factories-in-descriptor (Machine Level Registry) is OCP-aligned** — new levels and transports are added by registering a descriptor; no framework code is modified (`docs/pypto-runtime-design/02-logical-view.md:217`). D2, E4.

## 3. Cons

- **Cycle candidate `scheduler/` ↔ `distributed/`.** `docs/pypto-runtime-design/03-development-view.md:207` lists `distributed/` as "Depended on by: `scheduler/` (for cross-node coordination), `runtime/`", while `docs/pypto-runtime-design/modules/distributed.md:15-17` and `docs/pypto-runtime-design/modules/distributed.md:215` show `distributed_scheduler --> schedBase` where `schedBase = scheduler/ TaskManager+WorkerManager+ResourceManager`. Hard rule D6 violated.
- **`core/` depends on `hal/`** (`docs/pypto-runtime-design/modules/core.md:15-17`). The concrete coupling is only the typedef of `DeviceAddress`/`NodeId`, but the effect is that every consumer of `core/` transitively depends on `hal/`, reversing the usual DIP direction and making `core/` unbuildable without a HAL target. D2.
- **`ISchedulerLayer` is a fat interface.** 18+ virtual methods over four unrelated concerns (`docs/pypto-runtime-design/02-logical-view/09-interfaces.md:10-62`). A Python `bindings/` submit path, an assembly-time `runtime/` wiring path, a parent-layer completion path, and a child-layer completion path each drag in the full vtable. ISP; D4.
- **`runtime/` has 7+ internal concerns** (registry, parser, factory, wiring, elision, version, runtime) in one module (`docs/pypto-runtime-design/modules/runtime.md:225-234`). Likely to change for many different reasons (new level topology, new handshake field, new elision rule, new distributed mode). D3, D5.
- **Transport owns distributed protocol semantic payloads.** `RemoteSubmitPayload`, `RemoteDepNotifyPayload`, `RemoteCompletePayload`, `RemoteDataReadyPayload`, `RemoteErrorPayload`, `HeartbeatPayload` are all specified in `docs/pypto-runtime-design/modules/transport.md:102-150`, yet their **semantics** are `distributed/`'s responsibility (`docs/pypto-runtime-design/modules/distributed.md:65-90`). Transport should be a dumb pipe. D3, D5.
- **`distributed_scheduler` reuses `scheduler/` base managers directly**, not via `ISchedulerLayer` — `docs/pypto-runtime-design/modules/distributed.md:215-219` shows `scheduler --> schedBase` where `schedBase = scheduler/ TaskManager+WorkerManager+ResourceManager`. This breaks ADR-008 ("The internal split ... is an implementation detail ... policies attach here, not at the `ISchedulerLayer` boundary") — if policies alone attach, so should cross-module consumers. D2.
- **DAG diagram vs. reality mismatch for `core/` ↔ `memory/` / `transport/`.** `ISchedulerLayer` in `core/` takes `IMemoryManager*`, `IMemoryOps*`, `IVerticalChannel*`, `IHorizontalChannel*` as setter parameters (`docs/pypto-runtime-design/modules/core.md:40-45`), but `core/`'s depends-on row in `docs/pypto-runtime-design/03-development-view.md:203` lists only `hal/`. Either these are forward-declared (should be documented) or there is an implicit undeclared dependency. D2, D6.
- **Duplicated concept `ScopeHandle`**: core/ owns it on `ISchedulerLayer` (`docs/pypto-runtime-design/modules/core.md:67-68`) and memory/ has `scope_enter`/`scope_exit` returning/accepting the same type on `IMemoryManager` (`docs/pypto-runtime-design/modules/memory.md:52-54`). No stated owner. Ubiquitous-language drift. D7.
- **Naming: `simpler.errors.MemoryError` both as subclass of `SimplerError` and as `simpler.errors.SimplerMemoryError(MemoryError)`** (`docs/pypto-runtime-design/modules/bindings.md:71-72`) — two distinct Python classes named `MemoryError` (one from stdlib, one `SimplerError`-based) cohabit the `simpler.errors` namespace. D7.
- **`IExecutionEngine` has a "simulation-mode specialization" comment on a non-sim interface** (`docs/pypto-runtime-design/modules/hal.md:111-119`). The leaf-only specialization is a tacit LSP note; the contract section should enumerate which invariants are upheld across all variants. LSP (D3 indirectly).

## 4. Proposals

| id | severity | summary | affected_docs | hot_path_impact | trade_offs | sanity_test |
|----|----------|---------|---------------|-----------------|------------|-------------|
| A7-P1 | blocker | Break the `scheduler/` ↔ `distributed/` dependency cycle by fixing its single direction (distributed depends on scheduler; scheduler depends only on core/abstractions for cross-node notifications) | `03-development-view.md`, `modules/distributed.md`, `modules/scheduler.md` | none | Gain: DAG invariant holds (D6). Give up: scheduler callbacks to remote tasks now route via the runtime-owned event loop / `ISchedulerLayer.notify_dep_satisfied`, not a direct call into `distributed/`. | Reviewer verifies by running a build-time DAG check script against the updated dependency table and by grepping `scheduler/` for `#include "distributed/..."` (must be zero). |
| A7-P2 | high | Split `ISchedulerLayer` into four role interfaces (Wiring, Submit, Completion, Lifecycle) plus an aggregator | `modules/core.md`, `02-logical-view/09-interfaces.md` | none | Gain: ISP — `bindings/` depends only on Submit; child layer depends only on Completion; `runtime/` depends on Wiring+Lifecycle. Give up: one aggregator class instead of one flat class; minor test plumbing. | Verify each module's header only `#include`s the role interface it consumes (static assert with `grep`/`clangd` refs count per role). |
| A7-P3 | high | Move `DeviceAddress` / `NodeId` / generic opaque id typedefs from `hal/` into `core/types.h`; invert `core/ → hal/` dependency into `hal/ → core/` | `modules/core.md`, `modules/hal.md`, `03-development-view.md` | none | Gain: DIP holds (D2); `core/` becomes a true leaf consumer only of `error/`. Give up: HAL now includes a core header; a small refactor. | Verify `core/` builds standalone against a stub HAL = empty target; `include/core/types.h` contains `DeviceAddress`. |
| A7-P4 | medium | Relocate distributed protocol payload structs (`RemoteSubmitPayload`, `RemoteDepNotifyPayload`, `RemoteCompletePayload`, `RemoteDataReadyPayload`, `RemoteErrorPayload`, `HeartbeatPayload`) from `transport/` into `distributed/` | `modules/transport.md`, `modules/distributed.md` | none | Gain: SRP — transport/ is framing + reliability only; distributed/ owns protocol semantics (D3, D5). Give up: an `IHorizontalChannel::send` now takes an opaque byte-span instead of a typed payload (already true in practice). | Verify `transport/` has no `#include "distributed/..."`; `distributed/` defines all payloads; `transport/messages.h` contains only `MessageHeader` + framing + `MessageType` tags. |
| A7-P5 | high | Require `distributed_scheduler` to depend only on `ISchedulerLayer` (+ split role interfaces from A7-P2), not on concrete `scheduler/` `TaskManager`/`WorkerManager`/`ResourceManager` | `modules/distributed.md`, `modules/scheduler.md`, ADR-008 text in `08-design-decisions.md` | none | Gain: D2 DIP at the scheduler boundary; ADR-008 claim becomes enforced. Give up: `distributed_scheduler` either subclasses an abstract `AbstractSchedulerLayer` in `scheduler/` *or* rebuilds TaskManager-equivalent state machines — code duplication vs. coupling trade. | Verify `distributed/` `#include` list contains `core/` + `scheduler/i_scheduler_layer.h` (or equivalent), not `scheduler/task_manager.h` / `worker_manager.h` / `resource_manager.h`. |
| A7-P6 | medium | Extract Machine-Level-Registry + deployment parser into a new `composition/` (or `registry/`) module between `runtime/` and its callers | `modules/runtime.md`, `03-development-view.md`, `02-logical-view/05-machine-level-registry.md` | none | Gain: `runtime/` narrows to Runtime+DistributedRuntime+wiring+elision; registry/descriptor/parser evolves independently (D3, D5). Give up: one more compiled module; minor namespace churn. | Verify `runtime/src/` no longer contains `machine_level_registry.cpp`, `deployment_parser.cpp`, `layer_factory.cpp`; the new module's `include/` is consumed by both `runtime/` and `bindings/`. |
| A7-P7 | medium | Make the forward-declaration contract of `core/i_scheduler_layer.h` explicit (lists `IMemoryManager`, `IMemoryOps`, `IVerticalChannel`, `IHorizontalChannel` as forward-declared only; pointer-typed setters) and add the corresponding *interface dependency* edges to the logical DAG | `modules/core.md`, `02-logical-view.md` (DAG), `03-development-view.md` (DAG) | none | Gain: DAG truthfulness (D6 consistency); readers understand why `core/` compiles without `memory/` / `transport/`. Give up: a small header-discipline rule to maintain. | Verify `core/include/core/i_scheduler_layer.h` contains only forward declarations for the four pointer-typed interfaces; logical DAG includes explicit `ISchedulerLayer → IMemoryManager` / `IMemoryOps` / `IVerticalChannel` / `IHorizontalChannel` edges annotated "interface reference only". |
| A7-P8 | low | Consolidate `ScopeHandle` ownership in `core/` and have `memory/` consume it (or rename the memory-side concept to `MemoryScopeHandle`) | `modules/core.md`, `modules/memory.md`, `appendix-a-glossary.md` | none | Gain: D7 ubiquitous language; eliminates implicit coupling between two modules defining the same concept. Give up: one rename + glossary entry update. | Verify glossary lists exactly one `ScopeHandle` (or two clearly distinct handles), and `memory/include/memory/i_memory_manager.h` uses `core::ScopeHandle` if unified. |
| A7-P9 | low | Deduplicate the two `MemoryError` classes in `simpler.errors` — one bases on `SimplerError`, the other bases on Python `MemoryError` | `modules/bindings.md` | none | Gain: naming clarity (D7). Give up: a single decision (keep one name; e.g. rename the stdlib-derived one to `SimplerOutOfMemory`). | Verify `dir(simpler.errors)` lists no two classes named `MemoryError`; `test_exception_hierarchy.py` passes. |

### Proposal detail

#### A7-P1: Break `scheduler/` ↔ `distributed/` cycle

- **Rationale:** D6 (hard rule: module dependency graph must be a DAG). `03-development-view.md:207` says `distributed/` is depended on by `scheduler/` "for cross-node coordination"; `modules/distributed.md:15-17` says `distributed/` depends on `scheduler/` (base managers). Both cannot hold.
- **Edit sketch:**
  - File: `03-development-view.md` §3.2 dependency-rules table, row for `distributed/`.
  - Location: `Depended on by` column for `distributed/`.
  - Delta: replace `scheduler/ (for cross-node coordination), runtime/` with `runtime/`. Move cross-node coordination to run through `ISchedulerLayer.notify_dep_satisfied` dispatched by `runtime/` (the node glue). Add an ADR note that `scheduler/` must not `#include` anything from `distributed/`.
- **Trade-offs:**
  - Gain: D6 holds; modular build order is strict leaf-to-root.
  - Give up: `scheduler/` cannot call `distributed/` helpers directly; remote notifications must surface as `SchedulerEvent`s produced by `distributed/` and consumed by `scheduler/` through `core/` types.
- **Sanity test:** `rg -l 'include\s+"distributed/' scheduler/include scheduler/src` returns empty; build-time topological sort in CMake succeeds with `scheduler/` preceding `distributed/`.

#### A7-P2: Split `ISchedulerLayer` into role interfaces

- **Rationale:** ISP, D4. `docs/pypto-runtime-design/02-logical-view/09-interfaces.md:10-62` defines 18+ methods across unrelated concerns; `bindings/` (`docs/pypto-runtime-design/modules/bindings.md:42-44`) only calls `submit*` and `submission_tasks`, yet compiles against the full vtable; a parent layer only calls `notify_child_complete`; `runtime/` only calls `set_*`, `init`, `drain`, `shutdown`. Each should depend on the narrowest surface.
- **Edit sketch:**
  - File: `modules/core.md`, `02-logical-view/09-interfaces.md`.
  - Location: `ISchedulerLayer` definition.
  - Delta: define `ISchedulerWiring` (5 `set_*` methods + `layer_id`/`level_name`), `ISchedulerSubmit` (4 `submit*` + `submission_tasks`), `ISchedulerCompletion` (`notify_dep_satisfied`, `notify_child_complete`), `ISchedulerLifecycle` (`init`, `drain`, `shutdown`, `is_idle`, `get_stats`), `ISchedulerScope` (`scope_begin`, `scope_end`). `ISchedulerLayer` becomes `: public ISchedulerWiring, public ISchedulerSubmit, public ISchedulerCompletion, public ISchedulerLifecycle, public ISchedulerScope` — unchanged at the implementer level.
- **Trade-offs:**
  - Gain: ISP — each consumer compiles against its role only; internal refactors of one role don't force rebuild of all consumers. Improves DfB (compile-time isolation).
  - Give up: one aggregator + five smaller headers; minor ceremony; A9 (KISS) tension.
- **Sanity test:** `rg -l 'i_scheduler_layer\.h' bindings/` returns no hits after refactor — bindings includes only `i_scheduler_submit.h` and `i_scheduler_lifecycle.h`. Unit test per role interface with a mock.

#### A7-P3: Invert `core/` ↔ `hal/` dependency for handle types

- **Rationale:** D2 DIP; currently `core/` depends on `hal/` only for `DeviceAddress` / `NodeId` typedefs (`docs/pypto-runtime-design/modules/core.md:15-17`). This forces every consumer of `core/` to transitively depend on `hal/`, including host-only tools and tests.
- **Edit sketch:**
  - File: `modules/core.md` §1.3 + `include/core/types.h`; `modules/hal.md` §1.3 + `include/hal/types.h`.
  - Location: handle-type declarations.
  - Delta: move `using DeviceAddress = std::uint64_t;` and `using NodeId = std::uint64_t;` to `core/types.h`. `hal/` re-exports from its own `types.h` for compatibility or includes core/types.h. Update `03-development-view.md` depends-on rows.
- **Trade-offs:**
  - Gain: `core/` becomes buildable without a HAL target; DIP holds.
  - Give up: HAL builds must include core's types header; one transitive edge added in the other direction.
- **Sanity test:** `cmake --build core` with `hal/` excluded from targets succeeds; `nm core.a | rg 'hal::'` returns empty.

#### A7-P4: Move distributed payload structs to `distributed/`

- **Rationale:** D3/D5. `transport/` is described as "protocol-agnostic transport surface" (`docs/pypto-runtime-design/modules/transport.md:6-11`), yet `modules/transport.md:102-150` defines `RemoteSubmitPayload` etc. whose semantics belong to `distributed/` (`modules/distributed.md:65-90`). Two reasons to change `transport/`: wire framing changes AND distributed protocol evolution.
- **Edit sketch:**
  - File: `modules/transport.md` §2.3, `modules/distributed.md`, `02-logical-view/08-communication.md`.
  - Location: payload struct definitions.
  - Delta: remove all `RemoteXxxPayload` / `HeartbeatPayload` from `transport/messages.h`; keep only `MessageHeader`, `MessageType` enum, framing. Move payloads to `distributed/include/distributed/protocol_payloads.h`. `transport/` API gains `send(peer, MessageType, span<const std::byte>, Timeout)`.
- **Trade-offs:**
  - Gain: SRP/cohesion. Distributed protocol evolves without `transport/` churn; transport focuses on reliable bounded-latency framing.
  - Give up: transport APIs trade a typed-struct convenience for typed-tag + byte-span; distributed protocol owns its own codec helpers.
- **Sanity test:** `rg 'RemoteSubmitPayload' transport/` returns empty; transport unit tests continue to pass without reference to distributed types.

#### A7-P5: `distributed_scheduler` must depend on `ISchedulerLayer` only, not on `TaskManager`/`WorkerManager`/`ResourceManager`

- **Rationale:** D2; ADR-008 explicitly states "The internal split ... is an implementation detail (ADR-008) — policies attach here, not at the `ISchedulerLayer` boundary" — i.e. internal managers are not a public API. `modules/distributed.md:15-17` and `modules/distributed.md:215-219` both couple `distributed_scheduler` to these internals.
- **Edit sketch:**
  - File: `modules/distributed.md` §1.3, §3.2 diagram, §3.3.
  - Location: depends-on list and internal dep diagram.
  - Delta: replace `schedBase = scheduler/ TaskManager+WorkerManager+ResourceManager` with `ischedLayer = core/ ISchedulerLayer`. If code reuse is genuinely required, promote the shared machinery into a new `scheduler/core/` sub-target (abstract base `AbstractISchedulerLayer`) that both `scheduler/` concrete levels and `distributed_scheduler` can consume.
- **Trade-offs:**
  - Gain: ADR-008 is enforced; distributed can be built and tested without the full `scheduler/` library.
  - Give up: If rewritten purely against `ISchedulerLayer`, `distributed_scheduler` duplicates state-machine code; alternative is a formal abstract base, slightly more complex than today.
- **Sanity test:** `rg -l 'task_manager\.h|worker_manager\.h|resource_manager\.h' distributed/` returns empty.

#### A7-P6: Extract Machine-Level-Registry + deployment parser

- **Rationale:** D3, D5. `runtime/src/` currently houses `runtime.cpp`, `distributed_runtime.cpp`, `machine_level_registry.cpp`, `layer_factory.cpp`, `deployment_parser.cpp`, `wiring.cpp`, `elision.cpp`, `version.cpp` (`docs/pypto-runtime-design/modules/runtime.md:225-234`) — at least three distinct change drivers (composition, registry schema, validation grammar).
- **Edit sketch:**
  - File: `03-development-view.md` §3.1 module tree, `modules/runtime.md`, new `modules/composition.md` (or `modules/registry.md`).
  - Location: module listing + dependency graph.
  - Delta: create `composition/` (or `registry/`) owning `MachineLevelRegistry`, `MachineLevelDescriptor`, `DeploymentConfig`, `deployment_parser`. `runtime/` depends on `composition/`. Re-parent `layer_factory.cpp` either to `composition/` (if descriptor-only) or keep in `runtime/` (if instantiation-heavy).
- **Trade-offs:**
  - Gain: cleaner SRP; registry/schema evolves without rebuilding runtime composition; also helps X5 (testability) — composition can be unit-tested without the full runtime.
  - Give up: one more compiled library; a small amount of namespace churn.
- **Sanity test:** `composition/` builds and `test_deployment_parser` runs without linking against `runtime/`.

#### A7-P7: Explicit forward-decl contract for `core/i_scheduler_layer.h`

- **Rationale:** D2, D6 consistency. `ISchedulerLayer` setters reference `IMemoryManager*`, `IMemoryOps*`, `IVerticalChannel*`, `IHorizontalChannel*` (`docs/pypto-runtime-design/modules/core.md:40-45`) but the DAG says `core/` depends only on `hal/` and `error/` (`docs/pypto-runtime-design/03-development-view.md:203`). The implicit assumption is forward-declaration only — it should be documented.
- **Edit sketch:**
  - File: `modules/core.md` §3.3 Key Design Decisions, `02-logical-view.md` §2.1.1 graph.
  - Location: wiring-interface section and DAG diagram.
  - Delta: add a bullet "`core/i_scheduler_layer.h` forward-declares `memory::IMemoryManager`, `memory::IMemoryOps`, `transport::IVerticalChannel`, `transport::IHorizontalChannel`; full definitions are included only by implementers, never by `core/` consumers." Add interface-reference edges to the logical DAG annotated as such.
- **Trade-offs:**
  - Gain: DAG truthfulness; avoids surprise coupling; documents a real discipline.
  - Give up: modest header-discipline rule enforced by review.
- **Sanity test:** `clang -E core/include/core/i_scheduler_layer.h` shows forward declarations only; `rg 'include.*(memory|transport)/' core/include` returns empty.

#### A7-P8: Consolidate `ScopeHandle` ownership

- **Rationale:** D7. Both `core/` (`docs/pypto-runtime-design/modules/core.md:67-68`) and `memory/` (`docs/pypto-runtime-design/modules/memory.md:52-54`) define/use `ScopeHandle`. Either they are the same type (then one must own it) or they are different (then they must have distinct names).
- **Edit sketch:**
  - File: `modules/core.md`, `modules/memory.md`, `appendix-a-glossary.md`.
  - Location: `ScopeHandle` references.
  - Delta: declare `ScopeHandle` in `core/include/core/scope.h`; `memory/` consumes the type via `#include <core/scope.h>`. Glossary adds single definition.
- **Trade-offs:**
  - Gain: ubiquitous language (D7); clean ownership.
  - Give up: one header rename.
- **Sanity test:** `rg -w 'ScopeHandle' --type cpp` shows one declaration (in `core/`), multiple consumers.

#### A7-P9: Deduplicate Python `MemoryError` class names

- **Rationale:** D7. `modules/bindings.md:71-72` lists both `simpler.errors.MemoryError(SimplerError)` and `simpler.errors.SimplerMemoryError(MemoryError)`. `MemoryError` collides with stdlib's `builtins.MemoryError`; two `MemoryError` classes in `simpler.errors` namespace is ambiguous.
- **Edit sketch:**
  - File: `modules/bindings.md` §2.2.
  - Location: exception-mapping table.
  - Delta: rename the `Memory` domain (any `OutOfMemory`) mapping to `simpler.errors.SimplerOutOfMemory(MemoryError)` and the `Memory` domain (other) mapping to `simpler.errors.MemoryAccessError(SimplerError)`, or similar; ensure uniqueness within `simpler.errors`.
- **Trade-offs:**
  - Gain: naming clarity.
  - Give up: one rename; potentially a minor public-API adjustment.
- **Sanity test:** `python -c "import simpler.errors; names = [n for n in dir(simpler.errors) if n == 'MemoryError']; assert len(names) <= 1"` passes.

## 7. Cross-aspect tensions

| pair | with_proposal | preferred_resolution |
|------|---------------|----------------------|
| A7 vs A9 | A7-P2 (split `ISchedulerLayer`) | Keep the aggregator `ISchedulerLayer` for implementers; split only the headers so consumers include narrow role interfaces. Zero new classes at the implementation side, mitigating A9's ceremony objection. |
| A7 vs A9 | A7-P6 (extract `composition/` module) | If A9 objects to a new top-level module, accept the alternative of a strict sub-namespace `runtime::composition` with its own `include/` and forbid cross-includes — same isolation, no new CMake library. |
| A7 vs A1 | A7-P2 (role interfaces) | A1 may flag an extra vtable slot on hot paths (submit / notify). Resolution: role interfaces are pure decomposition; the aggregator has the same layout today — no new indirection introduced. `hot_path_impact: none` verified by codegen inspection (templates inlined at the implementation class). |
| A7 vs A1 | A7-P3 (core/types.h relocation) | No hot-path effect — typedefs only. A1 can trivially bless. |
| A7 vs A2 | A7-P5 (DIP at distributed/scheduler boundary) | A2 supports this (extensibility win). No tension. |

## 9. Status

- **Satisfied with current design?** no
- **Open items expected in next round:** A7-P1 (blocker; cycle removal), A7-P2 (ISP split), A7-P3 (core/hal inversion), A7-P5 (ADR-008 enforcement). A7-P4 and A7-P6 are candidate quick wins; A7-P7 / A7-P8 / A7-P9 are clean-ups.
