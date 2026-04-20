# Edit: docs/pypto-runtime-design/modules/runtime.md

- **Driven by proposal(s):** A2-P2, A2-P3, A2-P5, A5-P1, A5-P2, A6-P4, A6-P7, A6-P11, A6-P13, A7-P6, A8-P1, A8-P4, A8-P5 (+ cross-refs A2-P4, A9-P6)
- **ADR link(s):** ADR-011-R2 (SimulationMode option iii); ADR-014 (`runtime::composition` freeze); ADR-017 (closed `DepMode`)
- **Timestamp (UTC):** 2026-04-19
- **Edit kind:** additive (consolidated `## 13. Review Amendments (R3)` section appended before Document status footer)

## Location

- Path: `docs/pypto-runtime-design/modules/runtime.md`
- Section / heading: new `## 13. Review Amendments (R3)`; references §2.1, §2.2, §2.4, §3.1, §4.1, §4.3, §5.1
- Line range before edit: 534–535 (footer only)

## Before

```534:535:docs/pypto-runtime-design/modules/runtime.md
**Document status:** Draft — ready for review.
```

## After

A new `## 13. Review Amendments (R3)` heading is inserted before the footer, with one `[UPDATED: <id>: ...]` quoted callout per assigned proposal and two cross-reference callouts (A2-P4, A9-P6). Per-proposal edits follow.

## Per-proposal edits

### A2-P2 — `LevelOverrides` closed-for-v1; v2 planned

> [UPDATED: A2-P2: closed v1 + Q6 v2 trigger] `LevelOverrides` typed and closed for v1. Schema-registered v2 recorded in Q6, trigger = "first concrete out-of-tree level". Validation at `deployment_parser` init only.

- **Rationale:** E4 / E5 / YAGNI.

### A2-P3 — String-keyed factories for open enums

> [UPDATED: A2-P3: string IDs for FailurePolicy/TransportBackend/NodeRole] Resolved via registry at `Runtime::init`. `SimulationMode` stays open (A9-P6). `DepMode` closed (ADR-017).

- **Rationale:** E4 / OCP / X3.

### A2-P5 — BC policy + stability classes

> [UPDATED: A2-P5: BC stability classes] Each public interface tagged `{stable-frozen, stable-additive, versioned-subinterface, unstable}`; "mark, ship warning for one minor release, remove". v1 readers tolerate unknown additive fields.

- **Rationale:** E2.

### A5-P1 — `RetryPolicy` on `DeploymentConfig`

> [UPDATED: A5-P1: RetryPolicy struct] `{base_ms=50, max_ms=2000, jitter=0.3, max_retries=5}`; wired through distributed retry paths; A8-P5 alert at n=4.

- **Rationale:** R2.

### A5-P2 — `CircuitBreaker` on `DeploymentConfig`

> [UPDATED: A5-P2: CircuitBreaker struct] `{fail_threshold=5, cooldown_ms=10000, half_open_max_in_flight=1}`; states map into `PeerHealthState` in `distributed.md §3.5`.

- **Rationale:** R3.

### A6-P4 — `require_encrypted_transport=true`

> [UPDATED: A6-P4: fail-closed init] Multi-host plaintext TCP → `ErrorCode::InsecureTransport`. RDMA exempt (A6-P5). Loopback unaffected.

- **Rationale:** S4 / S5.

### A6-P7 — `allow_unsigned_functions` default gated

> [UPDATED: A6-P7: allow_unsigned_functions default] Defaults false in multi-tenant, true in single-tenant / SIM. Unsigned + false → `FunctionNotAttested`. Gated by `trust_boundary.multi_tenant`.

- **Rationale:** S1 / S3.

### A6-P11 — Gated `register_factory`

> [UPDATED: A6-P11: registration_token + post-freeze gate] `register_factory(descriptor, *, registration_token)`. Missing/wrong → `Unauthorized`. Post-`freeze()` → `RegistrationClosed`. Single guard in `runtime/` emits audit event.

- **Rationale:** S2 / S4.

### A6-P13 — Per-tenant submit rate-limit wired through runtime

> [UPDATED: A6-P13: per_tenant_submit_qps] `DeploymentConfig.per_tenant_submit_qps: uint32 | None`; single-tenant default = no limit; scheduler-side enforcement returns `TenantQuotaExceeded`.

- **Rationale:** S1.

### A7-P6 — Extract `runtime::composition` sub-namespace

> [UPDATED: A7-P6: runtime::composition frozen per ADR-014] Move `MachineLevelRegistry`, `MachineLevelDescriptor`, `DeploymentConfig`, `deployment_parser` into `runtime::composition` with its own `include/`. IWYU forbids cross-includes. Promotion triggers per ADR-014.

- **Rationale:** D3 / D5.

### A8-P1 — `IClock` threaded through runtime

> [UPDATED: A8-P1: IClock on Runtime] `Runtime` constructs a single `IClock` threaded via `ProfilingConfig`, `worker_timeout_ms`, heartbeat, and Timeouts. Release: link-time specialized to `CLOCK_MONOTONIC_RAW`.

- **Rationale:** X5.

### A8-P4 — `Runtime::dump_state()` diagnostic endpoint

> [UPDATED: A8-P4: dump_state(DumpOptions) → StateDump] Structured JSON aggregated across `ISchedulerLayer::describe()`, `IMemoryManager::describe_state()`, `DistributedRuntime::describe_cluster_state()`. Default scope = caller's `logical_system_id`; unscoped requires `diagnostic_admin_token`.

- **Rationale:** O4 / X6 / DfD.

### A8-P5 — `Runtime::on_alert` + externalized alert rules

> [UPDATED: A8-P5: on_alert + externalized AlertRule] `AlertRule{metric, op, threshold, window_ms, severity, logical_system_id}`; Prom/OTEL sinks opt-in deviation.

- **Rationale:** O5 / E3 / X8.

### Cross-ref A2-P4 — Migration & transition plan

> [UPDATED: A2-P4-ref: migration plan in 03-development-view.md §3.5] Phase table drives `PTO_USE_NEW_SCHEDULER` feature-flag rollout; `runtime/` respects phase flags during init.

### Cross-ref A9-P6 — SimulationMode option iii

> [UPDATED: A9-P6-ref: v1 registers FUNCTIONAL only] `SimulationMode` stays open; REPLAY scaffolding declared but not factory-registered; explicit REPLAY request rejects with a clear error. ADR-011-R2 authoritative.

## Verification steps

1. `rg -n "\[UPDATED: " docs/pypto-runtime-design/modules/runtime.md` prints 15 entries in §13.
2. `rg -n "registration_token|factory_registration_token" docs/pypto-runtime-design/modules/runtime.md` surfaces A6-P11.
3. `rg -n "runtime::composition" docs/pypto-runtime-design/modules/runtime.md` surfaces A7-P6.
4. Cross-view: `bindings.md §13` (A6-P11, A8-P4, A8-P5) corroborates.
