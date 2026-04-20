# Edit: docs/pypto-runtime-design/modules/distributed.md

- **Driven by proposal(s):** A1-P3, A1-P6, A2-P3, A2-P6, A5-P1, A5-P2, A5-P3, A5-P10, A5-P11, A5-P13, A6-P2, A6-P11, A7-P1, A7-P4, A7-P5, A8-P3, A8-P6, A8-P7, A10-P2, A10-P6, A10-P9
- **ADR link(s):** ADR-015 (I-DIST-1); ADR-017 (closed `DepMode`); ADR-018 (unified peer-health FSM); ADR-020 (coordinator_generation)
- **Timestamp (UTC):** 2026-04-19
- **Edit kind:** structural (new §3.5 Unified peer-health FSM inserted) + additive (consolidated `## 13. Review Amendments (R3)` section appended before Document status footer)

## Location

- Path: `docs/pypto-runtime-design/modules/distributed.md`
- New section: `### 3.5 Unified peer-health FSM` (inserted between §3.3 and §4)
- New section: `## 13. Review Amendments (R3)` (inserted before footer)
- Line range before edit (structural insertion point): after §3.3 at ~line 236; footer at 472–473

## Before

```229:236:docs/pypto-runtime-design/modules/distributed.md
### 3.3 Key Design Decisions (Module-Level)

- **Remote dependencies use the local task state machine.** Remote proxies allocate real local slots and drive the same `TaskState` transitions, so `scheduler/` does not need special cases for remote deps.
- **Partitioner plugs in as a policy** — identical lifecycle rules to `ITaskSchedulePolicy` (synchronous, scheduler-thread only, no locks).
- **Protocol handler is a thin adapter** — it translates messages into `SchedulerEvent`s. All state mutation lives in `distributed_scheduler` proper, preserving the Stage-B mutation invariant (ADR-010).
- **Failure policy pluggable** — the three v1 modes (`ABORT_ALL`, `CONTINUE_REDUCED`, `RETRY_ELSEWHERE`) cover current deployments; additional policies are added as plug-ins.
- **Idempotent message handling.** Every inbound message carries `correlation_id` + `remote_task_key`; duplicate delivery is tolerated via a sliding dedup window.
```

```472:473:docs/pypto-runtime-design/modules/distributed.md
**Document status:** Draft — ready for review.
```

## After

Two structural changes plus a consolidated §13:

1. **New §3.5 "Unified peer-health FSM"** (A5-P11) inserted between §3.3 and §4. Defines `PeerHealthState` with 7 states `{HEALTHY, SUSPECT, OPEN, HALF_OPEN, UNAVAILABLE(Quarantine), LOST, AUTH_REVOKED}`, co-owned across breaker (A5-P2), heartbeat (A10-P6), auth (A6-P2). Authoritative per ADR-018.
2. **New §13 Review Amendments (R3)** consolidates the 21 remaining callouts.

## Per-proposal edits

### A5-P11 — Unified peer-health FSM (new §3.5; structural)

- **Before:** Peer health was tracked implicitly by breaker + heartbeat + auth with overlapping / conflicting views.
- **After (new §3.5):**

> [UPDATED: A5-P11: new section — unified peer-health FSM] Single FSM with states `{HEALTHY, SUSPECT, OPEN, HALF_OPEN, UNAVAILABLE(Quarantine), LOST, AUTH_REVOKED}`. Breaker, heartbeat, auth are the only writers (`on_breaker_event / on_heartbeat_event / on_auth_event`). Readers query `get_peer_health(NodeId)`. Invariants: one state per peer (seqlock-serialized); `SUSPECT ↔ HEALTHY` rate-limited to prevent livelock; `UNAVAILABLE(Quarantine)` and `AUTH_REVOKED` require affirmative probe / re-handshake.

- **Rationale:** R3 / R5 — closes the 6.2.2 livelock gap and removes shadow state across modules.

### A1-P3 — HEARTBEAT `function_bloom` (distributed side)

> [UPDATED: A1-P3: function_bloom on HeartbeatPayload] Each peer publishes `function_bloom[4]`; coordinator Bloom-checks before inlining binaries in `REMOTE_SUBMIT`.

### A1-P6 — REMOTE_BINARY_PUSH + descriptor-template registry (distributed consumer)

> [UPDATED: A1-P6: per-peer template registry + descriptor_template_id] Binaries staged via REMOTE_BINARY_PUSH; per-peer template registry; partitioner projects per-peer (absorbs A10-P5).

### A2-P3 — Open string-keyed factories; closed `DepMode`

> [UPDATED: A2-P3: open enums via string IDs; DepMode closed] `FailurePolicy`, `TransportBackend`, `NodeRole` become string IDs at `Runtime::init`. `DepMode` stays closed (ADR-017). `SimulationMode` stays open per A9-P6.

### A2-P6 — `DistributedMessageHandler` free function v1

> [UPDATED: A2-P6: free-function dispatch] `distributed/protocol.hpp` dispatches `MessageType` via an O(1) table. Abstract `IDistributedProtocolHandler` class demoted until a second backend exists. I-DIST-1 preserved.

### A5-P1 — `RetryPolicy`

> [UPDATED: A5-P1: RetryPolicy] `{base_ms=50, max_ms=2000, jitter=0.3, max_retries=5}`; backoff `min(max_ms, base_ms·2^n)·(1+U[-jitter,+jitter])`.

### A5-P2 — Per-peer circuit breaker (now subsumed into §3.5)

> [UPDATED: A5-P2: breaker subsumed into §3.5] `{fail_threshold=5, cooldown_ms=10000, half_open_max_in_flight=1}` states map into `PeerHealthState`. Steady-state fast-path via thread_local TSC short-circuit.

### A5-P3 — v1 coordinator fail-fast + liveness timeout

> [UPDATED: A5-P3: fail-fast + coordinator_liveness_timeout_ms] Every surviving peer surfaces `CoordinatorLost` within `heartbeat_timeout_ms`. Add `coordinator_liveness_timeout_ms < heartbeat_timeout_ms`, default `3 × heartbeat_interval_ms`. Scope pinned to failed Pod; `cluster_view` generation bumps.

### A5-P10 — DS4 per-`REMOTE_*` idempotency

> [UPDATED: A5-P10: idempotency annotation CI] Every `REMOTE_*` handler declares `idempotency ∈ {safe, at-most-once, at-least-once}` with a dedup-key link. Doc-lint CI enforces.

### A5-P13 — `breaker_auth_fail_weight=10`

> [UPDATED: A5-P13: auth-fail weight 10×] `AuthenticationFailed` counts 10× in breaker accumulator so flapping credentials trip OPEN before `fail_threshold=5` is exhausted by benign retries.

### A6-P2 — HANDSHAKE + `coordinator_generation` (distributed side)

> [UPDATED: A6-P2: coordinator_generation + StaleCoordinatorClaim] `HandshakePayload.coordinator_generation`; mismatched → `StaleCoordinatorClaim` (ADR-020). `ClusterView.verified` added.

### A6-P11 — Factory-registration gate

> [UPDATED: A6-P11: register_factory freeze() gate] All `distributed_scheduler` factory registrations flow through the single `runtime/` guard; after `freeze()` → `RegistrationClosed`; emits audit event.

### A7-P1 — Strictly descending module graph

> [UPDATED: A7-P1: bindings > runtime > scheduler > distributed > transport > hal > core] `scheduler/` may not `#include` `distributed/`; remote notifications cross via `SchedulerEvent` + `core/` types.

### A7-P4 — Payload structs owned by `distributed/`

> [UPDATED: A7-P4: distributed/protocol_payloads.h + I-DIST-1] `RemoteXxxPayload` / `HeartbeatPayload` live here; transport keeps only `MessageHeader`, framing, `MessageType` tags. IWYU-CI enforces I-DIST-1 (ADR-015).

### A7-P5 — `distributed_scheduler` depends only on `ISchedulerLayer`

> [UPDATED: A7-P5: depend on core/ISchedulerLayer] Replace `scheduler/ TaskManager+WorkerManager+ResourceManager` with `core/ ISchedulerLayer` (+ A7-P2 role interfaces). Stress-link only against `ISchedulerLayer.h`.

### A8-P3 — `DistributedStats` schema

> [UPDATED: A8-P3: DistributedStats schema] Concrete fields + units (messages by type, retries, dup-detect hits, peer-health transitions). Latency histograms share the A8-P3 `LatencyHistogram` primitive.

### A8-P6 — Distributed trace merge rule

> [UPDATED: A8-P6: merge rule + tie-breaker] Primary sort by `(sequence, correlation_id, happens_before)`; skew-windowed reorder; tie-breaker `min(node_id)` in youngest all-online epoch.

### A8-P7 — `IFaultInjector` hooks

> [UPDATED: A8-P7: IFaultInjector hooks in dedup path] Sim-only; covers RDMA-loss, heartbeat-miss, coordinator isolation.

### A10-P2 — v1 coordinator fail-fast (absorbed into A5-P3)

> [UPDATED: A10-P2: absorbed into A5-P3; v2 roadmap] v1 = fail-fast; v2 = decentralize via sticky routing + quorum `cluster_view` generation (roadmap-only ADR extension).

### A10-P6 — Heartbeat shard per 32 peers (pool cap=4)

> [UPDATED: A10-P6: heartbeat sharding + fast miss threshold] `heartbeat_miss_threshold_fast=2` with hysteresis; sub-second `NodeLost` under hard failure. Shard per 32 peers; pool cap=4 (assumes ≤128 peers; Q18 revisit trigger).

### A10-P9 — Gate `WorkStealing × RETRY_ELSEWHERE`

> [UPDATED: A10-P9: incompatibility unless assignment log] Incompatible unless a live `(submission_id, task_index) → final_node_id` log is written before dispatch.

## Verification steps

1. `rg -n "### 3\.5 Unified peer-health FSM" docs/pypto-runtime-design/modules/distributed.md` returns exactly one hit.
2. `rg -n "\[UPDATED: " docs/pypto-runtime-design/modules/distributed.md` prints 21+ entries (one per proposal in §13 + the §3.5 preamble marker).
3. `rg -n "PeerHealthState" docs/pypto-runtime-design/modules/distributed.md` surfaces the new enum and cross-refs.
4. `rg -n "I-DIST-1|IWYU" docs/pypto-runtime-design/modules/distributed.md` confirms the A7-P4 invariant is stated.
5. Cross-view: `transport.md §13` mirrors A1-P3 / A1-P6 / A2-P1 / A6-P2 / A6-P3 / A6-P4 / A6-P9 / A7-P4.
