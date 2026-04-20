# 9. Open Questions

Unresolved questions flagged for review. Each question includes the context and the design options being considered.

---

## Q1: Multi-Die Level Placement

**Context:** a5-class multi-die Devices have multiple chip dies within a single device package.

**Question:** Should multi-die Devices register a separate `"ChipDie"` Machine Level between `"Device"` and `"Chip"`, or should the Device-level Scheduler internally manage die-level scheduling?

**Options:**
- **(A) Separate Machine Level:** Register `"ChipDie"` with its own Scheduler, Memory Manager, and Channels. Clean and consistent with the hierarchical model, but adds a level for every multi-die deployment.
- **(B) Internal to Device Scheduler:** The Device-level Scheduler handles die assignment as an internal concern. Simpler for single-die platforms, but hides scheduling decisions from the framework.

**Note:** The registry model supports both options — this is a configuration decision, not a framework design decision.

---

## Q2: Level Elision Strategy

**Context:** When a level is elided (`instance_count = 0`), the parent level must connect to the grandchild level.

**Question:** Should the framework provide an automatic pass-through adapter that bridges the parent's Vertical Channel to the grandchild, or should the parent level's `IVerticalChannelFactory` be responsible for detecting elision and creating a direct channel?

**Options:**
- **(A) Framework adapter:** The runtime inserts a transparent forwarding layer. Zero implementation burden on level authors, but adds a layer of indirection.
- **(B) Factory responsibility:** Each `IVerticalChannelFactory` checks whether its child is elided and adapts. More explicit, but each factory must handle the case.
- **(C) Hybrid:** The framework provides a `PassThroughVerticalChannel` adapter as a utility; factories can use it or implement their own bypass logic.

---

## Q3: Async Queue Sizing

**Context:** Per-level processing queues (ready queues, completion queues) must be sized appropriately.

**Question:** Should per-level processing queues be statically sized (configured in `LevelParams`) or dynamically growable?

**Options:**
- **(A) Static (pre-allocated):** Fixed size from `LevelParams`. Predictable memory usage, no allocation in hot path, but may waste memory or overflow.
- **(B) Dynamic (growable):** Start small, grow as needed. Better memory utilization, but allocation in hot path risks latency spikes.
- **(C) Static with overflow to dynamic:** Pre-allocate a fixed primary queue; if full, spill to a dynamically allocated overflow. Best of both, but more complex.

---

## Q4: Collective Operations Placement

**Context:** Collective operations (all-reduce, all-gather) are essential for distributed machine learning workloads.

**Question:** Should collective operations be expressed as compositions of `IHorizontalChannel.transfer()` calls at a distributed level, or should the Horizontal Channel itself expose native collective primitives?

**Options:**
- **(A) Composition:** Collectives are implemented as Orchestration Functions that compose point-to-point transfers. Maximum flexibility and portability, but may not leverage hardware-optimized collectives.
- **(B) Native primitives:** `IHorizontalChannel` exposes `barrier()`, `all_reduce()`, `all_gather()` natively. Can leverage hardware collectives (e.g., HCCL), but makes the Channel interface larger and harder to implement for simple backends.
- **(C) Layered:** Basic `IHorizontalChannel` has point-to-point only; an `ICollectiveOps` extension interface provides native collectives when available, with a fallback implementation built on point-to-point.

---

## Q5: Machine Level Hot-Registration

**Context:** The current design freezes the Machine Level Registry at initialization time (`registry.freeze()`).

**Question:** Should the framework support adding or removing levels at runtime (e.g., for dynamic cluster membership)?

**Options:**
- **(A) No (freeze at init):** Simplest and most predictable. Dynamic cluster membership is handled by updating node counts within existing levels, not by adding/removing levels.
- **(B) Yes (hot-registration):** Supports truly dynamic topologies. Significantly more complex: requires live rewiring of channels, draining tasks from removed levels, and synchronizing across nodes.

**Current recommendation:** Option A for the initial design. Dynamic node membership within a level (adding/removing Pod workers) can be supported without hot-registration.

---

## Q6: LevelParams Schema

**Context:** Different Machine Levels have different configuration needs (core counts, memory sizes, network settings).

**Question:** Should `LevelParams` be a generic key-value map, a union of per-level structs, or should each Machine Level define its own params type?

**Options:**
- **(A) Generic key-value map:** Maximum flexibility. No type safety; easy to misconfigure.
- **(B) Union of per-level structs:** Type-safe for known levels. Adding new levels requires modifying the union definition (violates OCP).
- **(C) Type-erased with schema validation:** Each level registers a schema for its params. Configuration is validated against the schema at registration time. Type-safe at runtime with OCP compliance.

---

## Q7: DSL Backward Compatibility Mapping

**Context:** Existing DSL constructs (`pl.incore`, `pl.auto_incore`, `FunctionType.*`) must map to the unified grammar.

**Question:** Should backward compatibility be handled at the compiler level (syntactic sugar) or at the runtime level (runtime alias resolution)?

**Options:**
- **(A) Compiler level:** The compiler translates old syntax to new `pl.at()` / `pl.function()` syntax during compilation. Clean runtime; old syntax eventually deprecated.
- **(B) Runtime level:** The runtime accepts both old and new function declarations and resolves them. More flexible but adds complexity to the runtime's function resolution path.

---

## Q8: Deferred Completion Tracking

**Context:** Tasks marked `complete_in_future` (SDMA, RoCE, etc.) need hardware completion tracking.

**Question:** How should the Scheduler discover that a deferred completion has occurred? Polling? Interrupt-driven? Completion queue?

**Options:**
- **(A) Polling:** Scheduler periodically polls hardware completion flags. Simple, portable, but wastes CPU cycles.
- **(B) Completion queue:** Hardware posts completion events to a queue; Scheduler drains the queue. Efficient, but requires hardware support.
- **(C) Hybrid:** Use completion queues where available (RDMA, SDMA); fall back to polling for simple hardware interfaces.

---

## Q10: Latency Budget Validation Method

**Context:** The Process View (§4.8) defines latency budgets for three critical paths. These budgets are based on hardware specifications and estimates (marked [ASSUMPTION]).

**Question:** How should latency budgets be validated during development — should there be a dedicated benchmark harness, or should profiling at Level 2 (phase-level timing) be sufficient?

**Options:**
- **(A) Dedicated benchmark harness:** A standalone test that submits synthetic tasks and measures per-stage latency against budgets. Repeatable, automatable, but adds development and maintenance cost.
- **(B) Profiling at Level 2:** Use the existing profiling infrastructure (§7.2) at Level 2 to capture per-stage timestamps. Requires real or simulated workloads; less isolated but tests real behavior.
- **(C) Both:** Benchmark harness for CI regression detection; profiling for production validation. Most thorough but highest effort.

**Note:** The budgets themselves may need revision once initial profiling data is available. The validation method should support iterative refinement.

---

## Q11: Async Policy Hook Invocation

**Context:** The schedule policy interfaces (`ITaskSchedulePolicy`, `IWorkerSelectionPolicy`, `IResourceAllocationPolicy`) are invoked synchronously on the scheduler's critical path. Some advanced policies may need to query external systems (e.g., a cluster manager for global load information, or a remote metrics service for data locality decisions).

**Question:** Should policy hooks support asynchronous invocation, or must all policies complete synchronously within the scheduler's thread?

**Options:**
- **(A) Synchronous only:** All policy methods must return immediately. Policies that need external data must pre-fetch and cache it. Simplest and safest for critical-path latency (Rule X3).
- **(B) Async with timeout:** Policy methods may return a "pending" status. The scheduler defers the decision and re-evaluates when the policy signals completion (or timeout). More flexible but adds complexity to the scheduler's event loop.
- **(C) Split interface:** Observation hooks (`on_*_state_change`) are synchronous (fire-and-forget). Decision hooks (`rank_ready_tasks`, `select_worker`, `allocate_resources`) may be async with a bounded timeout. Balances flexibility with critical-path safety.

**Current recommendation:** Option A for the initial design. Async policies can be added as a future extension if demand materializes.

---

## Q12: Runtime-Side WAR / WAW Tracking (Cross-Submission)

**Context:** Per [ADR-013](08-design-decisions.md#adr-013-two-stage-dependency-analysis--frontend-intra-submission-dag--runtime-cross-submission-raw), `DATA` mode installs only producer → consumer (RAW) edges across Submissions, backed by a single-valued `producer_index`. The frontend covers intra-Submission WAR / WAW via `intra_edges`, and the **non-aliasing intermediate-memref invariant** ([§2.1.6](02-logical-view/04-memory.md#216-memory-manager)) is what makes this sound across Submissions. Workloads that reuse a memref across Submissions (for example, an assemble-style operation spanning a Submission boundary, or a memory manager that deliberately aliases buffers) cannot express their hazards in the current model.

**Question:** When (and how) should the runtime's `DATA` mode be extended to track cross-Submission WAR / WAW via per-`BufferRef` version lists (as specified in [`tensor-dependency.md` §3](../../tensor-dependency.md))?

**Options:**
- **(A) Defer indefinitely; require the frontend to serialize aliasing cases to the same Submission.** Cheapest. Works as long as aliasing is tractable at trace time. Current default.
- **(B) Extend `producer_index` to a version list (producers + consumer set) and install WAR / WAW edges at admission.** Matches tensor-dependency.md §3 one-to-one; increases admission cost to O(boundary_in × avg_consumers). Requires a retirement-time GC pass that understands the consumer set.
- **(C) Conservative auto-`BARRIER` on detected aliasing.** The memory manager surfaces alias metadata; `TaskManager` promotes a `DATA`-mode Submission to `BARRIER` when a boundary-in `BufferRef` overlaps a live outstanding producer. Simplest upgrade path; loses parallelism at the alias point but avoids the version-list bookkeeping.

**Trigger for revisiting:** a first workload that requires a runtime-observable assemble or buffer-reuse across Submissions, or a measured performance regression from the auto-`BARRIER` fallback in option (C).

**Related:** ADR-013, [§2.10.6](02-logical-view/12-dependency-model.md#2106-scope-limits-v1), [`tensor-dependency.md` §3](../../tensor-dependency.md).

---

## Q13: Sub-Range `(offset, length)` on `TaskArgs` Tensor Arguments

**Context:** The frontend analyzer in [`tensor-dependency.md` §2.3 / §6](../../tensor-dependency.md) reasons over memref sub-ranges — notably for `assemble`-style ops that write a slice of a larger buffer. In the runtime, `ContinuousTensor` / `TaskArgs` carries a `BufferRef` + shapes but does not currently expose `(offset, length)` as a first-class overlap key. As long as sub-range analysis stays inside one Submission (frontend territory), this is fine: the frontend can serialize overlapping assembles via `intra_edges`. If sub-range analysis ever needs to happen at the runtime boundary (Q12 option B), the runtime would need to know which part of a `BufferRef` each Task touches.

**Question:** If / when cross-Submission sub-range overlap becomes a runtime concern, how should `(offset, length)` be carried on `TaskArgs` without breaking the existing `ContinuousTensor` descriptor or inflating hot-path storage?

**Options:**
- **(A) Add an optional `SubRange { offset, length }` field to the tensor-argument struct, sparsely populated.** Zero cost when absent; straightforward hash-map keying on `(BufferRef, offset, length)` when present. Widens the descriptor layout.
- **(B) Require the frontend to emit a synthetic child `BufferRef` per assembled sub-range.** Keeps `TaskArgs` unchanged; pushes the mechanism entirely into frontend + memory manager. Risks over-allocating `BufferRef` identities (one per sub-range write) and makes debugging noisier.
- **(C) Use an out-of-band "alias map" maintained by the memory manager and consulted by `TaskManager` on `DATA` mode.** Keeps `TaskArgs` flat; requires an additional query on every `DATA`-mode lookup. Potentially the cleanest if the memory manager is already tracking sub-range issuance.

**Trigger for revisiting:** Q12 is resolved toward option (B) [version-list runtime WAR/WAW], or a workload with trace-time-unresolvable sub-range aliasing emerges.

**Related:** Q12, [ADR-006 Tensor Lifecycle](08-design-decisions.md#adr-006-tensor-lifecycle-with-implicit-scope-and-explicit-free), [§2.4.1 `TaskArgs`](02-logical-view/07-task-model.md#241-task-state-model), [`tensor-dependency.md` §2.3](../../tensor-dependency.md).

---

## Q14: Token-Tensor Edge Representation in the Runtime

**Context:** [`tensor-dependency.md` §7](../../tensor-dependency.md) proposes representing WAR / WAW / resource dependencies as synthetic **token** tensors routed through the same RAW-edge machinery, yielding one unified DAG. The runtime currently carries dependencies as first-class edges (`IntraGroupEdge`, `DepList`, `producer_index`) — the "one unified DAG" property is already in place via explicit edges. Token adoption inside the runtime would turn every intra-Submission WAR / WAW edge into an extra `BufferRef` + Task dep, doubling the object count on the admission path for no new expressive power.

**Question:** Should tokens be adopted as an alternative dep-edge representation in the runtime (for example, as a debugging / visualization mode, or as a uniform representation alongside explicit edges)?

**Options:**
- **(A) Reject for v1 (current).** Keep explicit edges as the sole runtime representation; tokens, if used by the frontend, are lowered into `intra_edges` before `submit(...)`. Simplest; rejects duplicate mechanism.
- **(B) Optional frontend-visualization mode.** A flag in the frontend lowering can emit token `BufferRef`s inline with real tensors so visualizers can render one RAW-only DAG. The runtime's admission path remains unchanged; tokens become no-op data tensors.
- **(C) First-class runtime tokens.** Introduce `TokenBufferRef` as a synthetic `BufferRef` subtype; allow the frontend to ship token-based Submissions; runtime treats token `BufferRef`s as dep-only edges. Largest change; requires admission path to recognize and skip data-plane work for tokens.

**Current default:** Option (A). Option (B) is a likely near-term follow-up if debuggers / profilers benefit from a single visualized DAG.

**Related:** [ADR-013](08-design-decisions.md#adr-013-two-stage-dependency-analysis--frontend-intra-submission-dag--runtime-cross-submission-raw), [§2.10.2](02-logical-view/12-dependency-model.md#2102-frontend-contract), [`tensor-dependency.md` §7](../../tensor-dependency.md).

---

## Q9: Stages 1b–8 Design Document Status

**Context:** The current source material (`docs/architecture-redesign/`) contains a detailed Stage 1 document and plans for Stages 1b–8, but those stages have not yet been written as full design documents.

**Question:** Which stages should be prioritized for full design specification?

**Suggested priority (from master plan):**
1. **Stage 1b (DSL Design)** — needed to validate the abstract machine model against the user-facing programming model.
2. **Stage 2 (Task Model & State Machine)** — needed to finalize the state machine and dependency model before implementation.
3. **Stage 3 (HAL)** — needed to define the platform abstraction boundary.
4. **Stage 4 (Module Decomposition)** — needed to finalize module interfaces before coding begins.
5. **Stages 5–8** — can proceed in parallel once the core model and module boundaries are approved.

---

<!-- [UPDATED: A2-P7: new Q added] -->

## Q15: Async-Policy Extension Seam with Transport-Capability Axis

**Context:** Q11 already asks whether schedule-policy hooks (`ITaskSchedulePolicy`, `IWorkerSelectionPolicy`, `IResourceAllocationPolicy`) should support asynchronous invocation. Round 3 review of the distributed stack widened the question along two additional axes: (1) **transport-capability semantics** — policies that want to key decisions on capabilities such as "peer X has RDMA `rkey` scope Y" or "peer Y enforces TCP-TLS binding" need a queryable capability surface, not just a yes/no async flag; and (2) **async-submit return path** — if async policies are admitted, `submit(...)` may return before the scheduler commits a SubmissionHandle, which perturbs ADR-013's admission-ordering assumptions and the outstanding-window accounting (ADR-012, ADR-019). A2-P7 elected to **reserve the seam as a Q-record only**, shipping no v1 interface surface that would have to be migrated later.

**Question:** When (and how) should the async-policy extension seam be opened, and across which of the three axes (async-policy invocation, transport-capability semantics, async-submit return path)?

**Options:**
- **(A) Defer all three axes; keep v1 synchronous-only and transport-capability-opaque (current).** Matches Q11's current recommendation. Zero v1 interface surface.
- **(B) Open only the transport-capability axis in v1** (a read-only `PeerCapabilities` query on the `IHorizontalChannel`), leaving Q11's async decision for later. Gives policies the data they need without unblocking async dispatch.
- **(C) Open all three axes together with a bounded-timeout async hook and a versioned return-path contract.** Most flexible, highest v1 complexity; reviewers A3/A5/A9 flip back to disagree if Phase-5 final text reintroduces any v1 interface slot.

**Trigger for revisiting:** a concrete policy proposal that requires either cross-node capability queries (e.g., RDMA-aware worker selection) or defers admission on external signal (e.g., global load balancer).

**Related:** Q11 (async policy hook invocation), [A2-P7](reviews/2026-04-18-171357/final/final-proposal.md), [ADR-012](08-design-decisions.md#adr-012-submission-model-with-dependency-modes-outstanding-window-and-group-workspace), [ADR-019](08-design-decisions.md#adr-019-admission-shard-default--deployment-cue).

---

<!-- [UPDATED: A5-P14: new Q added] -->

## Q16: Orchestration-Composed Collective Partial-Failure → FailurePolicy Mapping

**Context:** A9-P3 resolves Q4 in favor of option (A): collective operations (all-reduce, all-gather) are implemented as Orchestration Functions composed from `IHorizontalChannel.transfer()` point-to-point primitives. This keeps `IHorizontalChannel` minimal but creates a gap A5-R3 flagged: when a composed collective experiences a **partial failure** (N of M peers fail mid-collective), there is no canonical rule for which `FailurePolicy` branch fires. A native all-reduce primitive could have defined "partial-failure = call policy with `partial_group_policy`"; a composed collective instead emits a cascade of per-peer transfer errors, and policy must be reconstructed from the cascade. A5-P14 recorded this as a v2 ADR trigger rather than pre-writing the mapping.

**Question:** How should the runtime map an orchestration-composed collective's partial-failure cascade onto `failure_policy` (§6.2.2) and `partial_group_policy`?

**Options:**
- **(A) Treat each point-to-point failure independently.** Each failed `transfer()` triggers its own retry / breaker / FailurePolicy path; the composed collective reports failure when any constituent fails past its budget. Simple; loses the "collective" atomicity that callers expect.
- **(B) Introduce a `CollectiveContext` that aggregates constituent transfer outcomes.** The orchestration function opens a context at start, collects per-peer results, and invokes `partial_group_policy` once with the aggregate at the end. Preserves callers' collective-level expectations; adds one new type.
- **(C) Promote partial-failure handling back into `IHorizontalChannel` via ICollectiveOps (Q4 option C).** Reverts the Q4/A9-P3 decision for the partial-failure case only. Heaviest option; re-opens a settled trade-off.

**Trigger for revisiting:** first workload that requires all-reduce partial-failure semantics (e.g., straggler-tolerant training), or a chaos-scenario run (A5-P5) where the composed-collective view disagrees with the native-primitive expectation.

**Related:** [A9-P3](reviews/2026-04-18-171357/final/final-proposal.md), [A5-P14](reviews/2026-04-18-171357/final/final-proposal.md), Q4, §6.2.2 (failure policy), `partial_group_policy` (glossary).

---

<!-- [UPDATED: A9-P6, A6-P12: new Q added] -->

## Q17: REPLAY Engine v2 Concrete Triggers and Schema Evolution

**Context:** Per ADR-011-R2, v1 ships `FUNCTIONAL` leaf-engine only; `PERFORMANCE` and `REPLAY` scaffolding is declared but not factory-registered, and the REPLAY trace envelope is frozen at the ADR-011-R2 schema (A6-P12 + A2-P9 — `{magic, trace_schema_version, platform, mode, capture_node_id, content_hash, detached_signature}`). The REPLAY engine implementation is deferred to v2. ADR-011-R2 names "Q17" as the tracking question for when v2 is triggered and how the schema may evolve.

**Question:** What concrete triggers promote REPLAY (and optionally PERFORMANCE) engine implementation from deferred to v2, and how does the frozen envelope schema evolve without invalidating v1-captured traces?

**Options for promotion triggers:**
- **(A) Post-mortem demand metric:** promote when ≥ N captured traces / month are submitted to human reviewers who would use replay. Needs a simple tallying infrastructure; captures real demand.
- **(B) Coverage-of-scenario metric:** promote when a specific A5-P5 chaos scenario cannot be reproduced without replay (e.g., a multi-node race that happens only under production load).
- **(C) Tooling-driven trigger:** promote alongside a committed investment in replay-based CI (deterministic regression for scheduler-ordering bugs).

**Options for schema evolution:**
- **(α) Versioned additive.** `trace_schema_version` minor bumps for additive fields; v2 engine reads v1 traces by ignoring unknown fields. Preferred.
- **(β) Major-bump + re-capture.** v2 engine refuses v1 traces; v1 trace emitters are frozen archives. Simplest on the engine side; wastes v1 traces.
- **(γ) Compatibility layer.** v2 engine ships a schema upgrader that rewrites v1 traces to v2 format offline. Most flexible; most engineering.

**Current recommendation:** combine **(B)** + **(α)** — promote when chaos-scenario demand appears; evolve schema additively. Confirm at v2 kickoff.

**Related:** [ADR-011-R2](08-design-decisions.md#adr-011-simulation-facility-with-three-leaf-execution-modes) (Revision R2), [A9-P6](reviews/2026-04-18-171357/final/final-proposal.md), [A6-P12](reviews/2026-04-18-171357/final/final-proposal.md), [A2-P9](reviews/2026-04-18-171357/final/final-proposal.md), [07-cross-cutting-concerns.md §7.2](07-cross-cutting-concerns.md#72-observability).

---

<!-- [UPDATED: A10-P6: new Q added] -->

## Q18: Heartbeat Shard Cap Beyond 128 Peers

**Context:** A10-P6 introduces faster peer-failure detection via a sharded heartbeat thread pool — one shard per 32 peers, thread-pool cap = 4. The cap-4 assumption bounds total heartbeat thread overhead on a single node and is sized for `≤ 128 peers per node` (4 shards × 32 peers). Topologies beyond that cap (large-cluster trainers, multi-tenant pods) would either need more shards or a different distribution strategy.

**Question:** When the topology grows past ≤ 128 peers per node, how should the heartbeat subsystem scale?

**Options:**
- **(A) Raise the thread-pool cap.** Shards = peers / 32, cap = max(4, ceil(peers / 32)). Straightforward; costs per-node CPU linearly in peer count.
- **(B) Share heartbeat across multiple nodes via a tree.** Each node heartbeats a subset of peers and gossips state; cap stays at 4. Reduces per-node cost; increases staleness and complicates the unified peer-health FSM (ADR-018) update path.
- **(C) Replace shard-per-32 with event-driven (transport-keepalive-only) detection.** No dedicated heartbeat thread; rely on TCP keepalive / RDMA completion errors exclusively. Eliminates the cap; loses the hysteresis budget A10-P6 relies on to tolerate 5% packet loss.

**Trigger for revisiting:** first deployment target that exceeds 128 peers per node, or a regression in `NodeLost` detection latency on a 64–128 peer cluster.

**Related:** [A10-P6](reviews/2026-04-18-171357/final/final-proposal.md), [ADR-018: Single Unified Peer-Health FSM](08-design-decisions.md#adr-018-single-unified-peer-health-fsm), [`modules/distributed.md`](modules/distributed.md) heartbeat-config section.
