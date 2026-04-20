# 10. Known Deviations

This section documents rule violations in the current architecture design, with justification and mitigation for each. Rule IDs reference [04-agent-rules.md](../../.cursor/guides/architecture-design-guide/04-agent-rules.md).

---

## Deviation 1: Intra-Node Channels Not Encrypted (Rule S5)

**Rule:** S5 — Encrypt in transit and at rest. All network communication uses TLS or equivalent.

**Violation:** Intra-node communication channels (shared memory, register bank, DMA) do not use encryption.

**Justification:** Intra-node components (Host, Device, Chip, Core) share a single trust domain within one physical machine. Encryption on these paths would introduce latency on nanosecond-scale critical paths (Rule X1, X3), negating the performance architecture. The threat model (§7.1.1) classifies intra-node paths as trusted.

**Mitigation:** The trust boundary is explicitly documented in the security section (§7.1). If future requirements demand intra-node encryption (e.g., multi-tenant hardware sharing), the `IVerticalChannel` and `IHorizontalChannel` interfaces support wrapping with encryption-enabled implementations without modifying core logic.

---

## Deviation 2: No Persistent State Recovery (Rule R5)

**Rule:** R5 — No single point of failure. Every component on the critical path must have a redundancy or fallback strategy.

**Violation:** The runtime does not persist task state across process restarts. A host process crash loses all in-flight tasks.

**Justification:** The Simpler runtime is a computation engine, not a stateful service. Computations are deterministic and re-submittable. Adding checkpointing or persistent task queues would add complexity and latency to every task submission (violating Rules G2, X1) for a recovery scenario that is better handled at the application layer (Python re-submission).

**Mitigation:** The Python-level API supports idempotent re-submission. External process monitors can detect crashes and restart. For distributed mode, the `failure_policy` configuration (§6.2.2) provides ABORT_ALL, CONTINUE_REDUCED, and RETRY_ALTERNATE strategies.

---

## Deviation 3: Network Encryption Deferred to Backend (Rule S5)

**Rule:** S5 — Encrypt in transit and at rest.

**Violation:** The runtime framework does not enforce TLS/mTLS on inter-node channels. Encryption is left to the network backend implementation.

**Justification:** Network backends (RDMA, TCP) have different encryption mechanisms (IPsec for RDMA, TLS for TCP). Enforcing encryption at the framework level would either restrict backend choices or add an unnecessary abstraction layer. The backend selection already determines the encryption approach.

**Mitigation:** The `IHorizontalChannel` interface supports encrypted backends (e.g., `transport_tcp` with TLS). Deployment configuration can require encrypted backends. The security section (§7.1.2) documents this as a known limitation with [ASSUMPTION] tag.

---

## Deviation 4: ~~Module Detailed Designs Not Yet Produced~~ (Resolved — Rule V1, Guide Phase 3)

**Rule:** V1 — All five views required (Development View requires per-module detailed designs per the guide).

**Previous violation:** Module Detailed Design documents (template Section 8) had not been produced for the ten modules.

**Resolution:** Per-module detailed design drafts now exist under [`modules/`](modules/) (one document per module, Section 8 template). They are **Draft** status pending review alongside the rest of the architecture package.

**Remaining gap:** Internal architecture details may still be refined during implementation provided public interface contracts are preserved ([Module Design Conventions in the Architecture Design Guide](../../.cursor/guides/architecture-design-guide/05-output-templates.md#module-design-conventions)).

---

<!-- [UPDATED: A5-P10: future-type annotation] -->

## Deviation 5: `REMOTE_COLLECTIVE_FLUSH` Dedup-Window Annotation Placeholder (Rule DS4)

**Rule:** DS4 — Every `REMOTE_*` handler MUST declare `idempotency ∈ {safe, at-most-once, at-least-once}` with a link to its dedup key; doc-lint CI enforces the annotation.

**Violation:** `REMOTE_COLLECTIVE_FLUSH` is reserved as a future `MessageType` (see Q16 and A9-P3 orchestration-composed collectives path) but is not implemented in v1 and therefore carries no concrete dedup-key binding.

**Justification:** Reserving the type now prevents future type-tag churn; emitting a placeholder idempotency annotation today would be misleading. Per A5-P10, the type is recorded as a future-type annotation — doc-lint CI treats `REMOTE_COLLECTIVE_FLUSH` as an allowlisted "reserved, no v1 handler" entry until the engine lands.

**Mitigation:** When the orchestration-composed collective handler is implemented (Q16), the annotation becomes normative: declared idempotency class + dedup-key reference checked in CI. Until then, any attempt to send `REMOTE_COLLECTIVE_FLUSH` returns `ErrorCode::MessageTypeNotImplemented`.

---

<!-- [UPDATED: 2026-04-19-P11.3: Phase-10 bindings scope cut] -->

## Deviation 6: DLPack / tensor interop not exposed in bindings v1 (Rule V1, modules/bindings.md §6)

**Rule:** V1 — Module detailed designs must be implemented in full, preserving the declared public interface. `modules/bindings.md` §6 lists DLPack zero-copy tensor interop as a v1 binding surface.

**Violation:** The `pypto-runtime` Phase-10 MVP (slices 10.1-10.5) did not bind any tensor / DLPack surface. Python side-effects that cross the tensor boundary (e.g. `numpy → IMemoryManager::allocate`) are inaccessible from `simpler.Runtime` today.

**Justification:** The `memory/` layer concrete surface (`IMemoryManager::allocate` / `mark_tensor_free`, `TensorMap::insert / resolve`) is stable, but the tensor-lifecycle protocol that DLPack must marshal — fan-out ref-count deliveries, cross-Submission WAR/WAW deps (deferred separately per Q12), workspace vs. data placement — has not yet been exercised end-to-end from Python. Binding DLPack before an end-to-end Python-side allocator-consumer scenario exists risks leaking a surface that would then have to be re-shaped, violating `V1` *stability* more severely than v1 *completeness*. The decision to cut was explicit, documented in `pypto-runtime/docs/plans/2026-04-19-phase-10-checkpoint.md`, and the deferral is visible in the master-plan `Status` line.

**Mitigation:** No DLPack code was merged, so no mis-aligned promise exists in `simpler/__init__.py` or `_core`. A follow-up slice (P12+) will land `simpler.Tensor` + DLPack protocol once the first Python-level scenario that exercises cross-Submission tensor deps is specified. Until then Python consumers of `pypto-runtime` drive the Runtime through `submit_task` / `submit` / `submit_spmd` with opaque `TaskArgs`.

---

<!-- [UPDATED: 2026-04-19-P11.3: wheel packaging deferred] -->

## Deviation 7: scikit-build-core wheel packaging not smoke-tested (Rule G6)

**Rule:** G6 — The runtime must ship as a `pip install`-able wheel with no out-of-tree build step.

**Violation:** The `pypto-runtime` build system is exclusively `cmake --preset a2a3sim` / `ctest --preset a2a3sim`. No scikit-build-core `pyproject.toml` entry, no wheel build, no `pip install pypto-runtime` smoke test has been exercised.

**Justification:** The Phase-0 decision (master-plan Phase-1 Q6) committed to `CMake + scikit-build-core + nanobind` as the delivery path. The Phase-10 MVP wires nanobind into the existing CMake preset, which is sufficient to exercise the full Python surface (`simpler._core`, `simpler.errors`, `simpler.Runtime`, `simpler.DistributedRuntime`, submission handles, `simpler.Scope`) via `ctest`. scikit-build-core layering + wheel metadata + CI smoke are pure *packaging* work that does not change the runtime surface and is only value-added for an external Python consumer. No such consumer exists at the close of P10.

**Mitigation:** The nanobind + CMake integration already follows scikit-build-core idioms (`nanobind_add_module`, no hand-rolled `setup.py`). Adding a `pyproject.toml` + `python -m build` smoke is a 1-slice follow-up. The deferral is documented in `pypto-runtime/docs/plans/2026-04-19-phase-10-checkpoint.md` and in the master-plan Phase-10 entry.

---

<!-- [UPDATED: 2026-04-19-P11.3: chaos-matrix pending-upstream tags] -->

## Deviation 8: §7.4 chaos-matrix rows SKIP with upstream-labelled ctest tags (Rule IR-T10)

**Rule:** IR-T10 — Every external-dependency touch has at least one failure-injection test. §7.4 normative matrix mandates all 14 rows (9 core + 5 security) be exercised end-to-end.

**Violation:** 12 of the 14 chaos-matrix rows are `GTEST_SKIP` today. Only `Chaos.Core.DmaBitFlip` (row 6) and `Chaos.Core.MemoryWatermarkBreach` (row 9) execute the full seam-to-observable path; every other row still schedules its `FaultKind` on the device injector (the §7.4.2 seam contract *is* exercised) but the downstream consumer that would turn the fault into an observable outcome does not yet exist:

- `pending-transport` (3 rows: NetworkPacketDrop, ByzantineSlowPeer, RdmaCrcCorrupt) — transport/ has `RetryPolicy` (slice 6.6) + chaos harness but no e2e scenario.
- `pending-distributed` (2 rows: NodeKill, CoordinatorIsolation) — distributed/ skeleton (P8) covers the protocol but not the peer-kill harness.
- `pending-observability` (1 row: ClockStep) — observability/ layer not yet standing on its own.
- `pending-admission` (2 rows: SlotPoolExhausted core + S4_QuotaSkew security) — scheduler admission (P7) landed but the HAL-fault → admission feedback loop is not wired.
- `pending-security` (4 rows: S1 HandshakeDowngrade, S2 CertRotation, S3 RdmaRkeyRace, S5 AuditSinkDrop) — `security/` module not yet in phase.

**Justification:** Each skip has a named upstream owner. Attempting to close any single row would require a vertical integration across 2-3 currently-deferred modules (e.g. NodeKill needs distributed/peer-health + runtime/progress-thread + hal/fault-injector already present). Aligning this work with the module phases (rather than opportunistic bring-up) keeps the iron-law "root cause before fix" discipline intact.

**Mitigation:**

1. Every SKIP row **still** schedules its `FaultKind` via `IFaultInjector::schedule_fault` so the §7.4.2 seam contract is exercised on every run (verified by `EXPECT_EQ(inj->pending(spec.kind), 1U)` before the skip).
2. Per P11.2, each row now carries a ctest label mirroring its C++ `skip_reason`; `ctest --preset a2a3sim -L pending-<upstream>` enumerates exactly the rows waiting on each module. The audit surface is executable, not prose-only.
3. Each row's restoration to "runs e2e" is tracked by the owning module's phase plan. Closing the last row retires this deviation.
