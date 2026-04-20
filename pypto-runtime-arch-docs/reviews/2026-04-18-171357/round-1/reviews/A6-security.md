# Aspect A6: Security & Trust Boundaries — Round 1

## Metadata

- **Reviewer:** A6
- **Round:** 1
- **Target:** docs/pypto-runtime-design/
- **Runtime Mode:** yes
- **Timestamp (UTC):** 2026-04-18T17:18:41Z

## 1. Rubric Checks

| # | Check | Result | Evidence (file:line) | Rule(s) |
|---|-------|--------|----------------------|---------|
| 1 | Every trust boundary identified and threat-modeled | Weak | `07-cross-cutting-concerns.md:11-34` — four boundaries enumerated with STRIDE; but several boundaries present in the design are not modeled: (a) untrusted Python → runtime C++ factory plug-in (`bindings.md:51`, `runtime.md:72-85` via `register_factory`); (b) REPLAY simulation trace-file input (`02-logical-view/10-platform.md:32`); (c) log / trace sinks that accept user-supplied Python callables (`bindings.md:321`, `07-cross-cutting-concerns.md:130`); (d) user-supplied AICore binary / function blob path (`modules/core.md:250-255`, `04-process-view.md:208-225`); (e) external data services / DFS / KV store (only the row in `07-cross-cutting-concerns.md:33` exists — no STRIDE enumeration for per-service endpoints); (f) multi-tenant Logical System isolation (`05-physical-view.md:237-246`, `01-introduction.md:44`) — no threats listed for tenant-to-tenant leakage through shared Machine Level instances | S1 |
| 2 | Least privilege for every component / key / service account | Weak | `07-cross-cutting-concerns.md:36-41` — only "each component accesses only its own level's Memory Scope" and "Logical System namespaces prevent cross-tenant access" are mentioned. There is no concept of per-tenant API key, per-peer credential, per-factory capability, or a principals-and-permissions model. `register_factory` is globally privileged with no check (`modules/runtime.md:72-85`). The RDMA `rkey` in `REMOTE_DATA_READY` is accepted from any peer with no scope restriction (`modules/transport.md:130-136`) and grants the reader direct RDMA READ into peer memory at `global_address` (`modules/distributed.md:346-363`). Peer "authentication" is only sketched: "reject unknown node IDs" (`07-cross-cutting-concerns.md:29`) without specifying *how* identity is bound to a key. | S2 |
| 3 | Inputs validated at trust boundaries (format, type, range, business rules) | Weak | Python ↔ C API: `07-cross-cutting-concerns.md:24` states "bounds-check all shape dimensions, reject negative sizes, validate alignment" but the binding module's validation surface is vague (`modules/bindings.md:55, 289, 370-371`) — only `DeploymentConfig` bad fields → `ValueError` is stated. `TaskDescriptor`, `SubmissionDescriptor.intra_edges` (cycle check: `modules/core.md:538`), DLPack stride/device-type validation (`modules/bindings.md:109-114`) are not documented as bounded-range-checked at the boundary. Distributed protocol: `MessageHeader::crc32` + `magic` + `version` provide integrity/framing (`modules/transport.md:76-98`), but `source_node`/`target_node` are **trusted from the wire** without cryptographic binding; `RemoteSubmitPayload::task_count` / `edge_count` (`modules/transport.md:104-112`) drive variable-length array parses with no documented upper bound; `RemoteErrorPayload::message_bytes` is "truncated" but no explicit `max` is defined. Function name path-traversal is marked Low with only verbal mitigation (`07-cross-cutting-concerns.md:26`). | S3 |
| 4 | Secure defaults (no relaxation without explicit opt-in) | Weak | Good: `07-cross-cutting-concerns.md:41` — "default configuration disables external network access; distributed mode requires explicit opt-in via `DistributedConfig`"; `bindings.md:348` — `debug_import=false`. Bad: `modules/transport.md:391-393` — `TRANSPORT_BACKEND` **defaults to `tcp` (unencrypted)** with no matching requirement that TLS/mTLS be enabled before cross-host use; `modules/transport.md:394` — `remote_timeout_ms` default 30 s (fine). Bad: simulation REPLAY (`02-logical-view/10-platform.md:32`) reads a trace artifact with no signed-input default. Bad: Python log sink registration (`modules/bindings.md:321`) has no opt-in flag for privilege-elevated sink contexts. | S4 |
| 5 | Sensitive data encrypted in transit and at rest | Fail | Two documented deviations (`10-known-deviations.md:5-40`) concede intra-node plaintext (Deviation 1) and *defer* inter-node encryption to the backend (Deviation 3). Neither the TCP backend (`modules/transport.md:184`) nor the RDMA backend (`modules/transport.md:185`) declares a concrete TLS / IPsec configuration schema; `EndpointSpec.credentials?` (`modules/transport.md:160`) is an optional, undefined field; there is no enforcement that a *multi-host* deployment must refuse to start on plaintext TCP. At rest: function binaries ferried cross-node (`04-process-view.md:221-225`) are not encrypted, only content-hashed for idempotency. Trace export (§7.2.6, `07-cross-cutting-concerns.md:125-130`) and profiling sinks write unencrypted JSON/binary dumps with no at-rest encryption clause. | S5 |
| 6 | Tamper-evident audit trail for security-relevant actions | Fail | No audit-trail artifact defined anywhere. The logging framework (`07-cross-cutting-concerns.md:108-114`) is general-purpose and explicitly *structured-log* only — no append-only, no integrity chain, no separate security-relevance classification. Security-relevant actions with no documented audit: node join (`07-cross-cutting-concerns.md:29`), `register_factory` plug-in registration (`modules/runtime.md:72-85`), Function binary registration (`modules/core.md:257-272`), `DeploymentConfig` changes, peer disconnect / `NodeLost` (`modules/transport.md:398-399`), shutdown, failure-policy overrides, log-sink registration (`modules/bindings.md:321`). | S6 |

## 2. Pros

- **Explicit trust-boundary table with STRIDE threats for the four primary boundaries** — `07-cross-cutting-concerns.md:11-34` satisfies the core of S1 for Python↔C, Host↔Device, Node↔Node, Runtime↔External-Data and maps each to a mitigation stub. [S1]
- **Secure-by-default network posture** — distributed mode requires explicit opt-in (`07-cross-cutting-concerns.md:41`); debug flags default off (`modules/bindings.md:348`). [S4, G6]
- **Documented deviations with justification and mitigation** — `10-known-deviations.md:5-40` is transparent about S5 gaps rather than silent, satisfying the `Rule Exceptions` protocol at the bottom of `04-agent-rules.md`. [S5 exception form]
- **Protocol framing has magic + version + CRC** — `modules/transport.md:76-98` provides wire-level integrity and version negotiation (`modules/transport.md:340-345`) that rejects `ProtocolVersionMismatch` on handshake. [S3 framing hygiene]
- **Idempotent / dedup-windowed message handling** — `modules/distributed.md:235, 272-275` reduces replay-injection risk at the protocol layer. [S1 replay mitigation]
- **TaskSlot generation counter** protects against stale-handle injection after `free()` (`modules/core.md:296-297`). [S3]
- **HAL register-bank handed only to authorized components** (`modules/hal.md:136`), and CANN/HCCL symbol scoping enforced by lint (`modules/hal.md:245`). [S2, X7]
- **Logical System partitioning described at the physical view** (`05-physical-view.md:237-246`) — at least names the multi-tenant surface. [S1, FR-9]

## 3. Cons

- **Trust-boundary roster is incomplete** — missing (a) untrusted-plug-in factory entry via `register_factory`, (b) REPLAY trace-file input, (c) Python-callable log/trace sinks, (d) multi-tenant Logical-System isolation as a boundary, (e) per-function binary provenance. [S1, `07-cross-cutting-concerns.md:11-34`]
- **No authentication mechanism is specified** — §7.1 mentions "node identity verification" and "authenticated node IDs" but never defines the primitive (pre-shared key? mTLS cert? SPIFFE ID?). The `HANDSHAKE` message (`modules/transport.md:83-85, 340-345`) carries no credential field. [S1/S2, `modules/transport.md:83-98`]
- **`EndpointSpec.credentials?` is an undefined optional** — a TLS/mTLS story is asserted but no schema, no key-material lifecycle, no rotation, no trust-root definition. [S5, `modules/transport.md:156-160`]
- **TCP backend defaults to plaintext** with no `require_tls_for_multi_host=true` guardrail. [S4/S5, `modules/transport.md:393`]
- **RDMA `rkey` exposure model is under-specified** — `REMOTE_DATA_READY` passes `global_address, rkey` (`modules/transport.md:130-136, modules/distributed.md:354-357`) and the consumer issues an RDMA READ. There is no *scope* on the `rkey` (per-buffer? per-task?), no expiry, no revocation. A compromised/captured message can convert into a sustained read into the producer's memory. [S2/S3, `modules/transport.md:130-136`]
- **Variable-length payload parses have no declared bounds** — `RemoteSubmitPayload::task_count` / `edge_count` drive trailing-array decodes. A message with `task_count = 2^32 − 1` is not rejected at the framing layer (CRC passes over any byte sequence). [S3, `modules/transport.md:104-112`]
- **`REMOTE_ERROR` message payload is opaque** — the handler reconstructs `ErrorContext` from `message_bytes` (`modules/distributed.md:394`) with no input validation documented — a remote peer can inject arbitrary UTF-8 into local logs / Python exception strings. [S3, `modules/transport.md:138-143`]
- **No input validation is attached to DLPack import** — device-type mismatch, negative / overflowing stride products, and untrusted `__dlpack_stream__` handles are not rejected at the binding boundary. [S3, `modules/bindings.md:109-114, 351`]
- **No audit trail** for node join / factory registration / function registration / shutdown / sink registration — S6 is effectively not addressed. [S6]
- **Function-binary provenance is not established** — `FunctionRegistry` keys by content hash for *idempotency*, not for trust (`modules/core.md:247-272`). Nothing in the framework verifies that the supplied blob is a legitimate, non-malicious AICore binary before the AICore `IExecutionEngine` dispatches it. [S1/S3]
- **Multi-tenant isolation is asserted but not enforced** — "Scheduler dispatches only within the same Logical System" and "Horizontal Channel messages filtered by `logical_system_name`" (`05-physical-view.md:242-243`) are unilateral statements with no concrete enforcement point (no check in `distributed_scheduler`, no label in `MessageHeader`). [S1/S2/S3]
- **Log sinks can exfiltrate** — a Python `ILogSink` (`modules/bindings.md:53, 321`) sees every log event globally. No capability scoping (sink-per-tenant, sink-per-severity minimum) is defined. [S2]
- **No threat model for REPLAY mode** — the runtime reconstructs FSM state from a "trace source" (`02-logical-view/10-platform.md:32`); untrusted traces can drive the state machines into unexpected states. No signed-trace requirement, no schema validation guarantee beyond a §7.2.2 shape. [S1/S3]

## 4. Proposals

| id | severity | summary | affected_docs | hot_path_impact | trade_offs | sanity_test |
|----|----------|---------|---------------|-----------------|------------|-------------|
| A6-P1 | high | Extend §7.1 STRIDE table with 6 missing boundaries (plug-in factory, REPLAY trace, Python sinks, tenant isolation, function-binary provenance, external-data per-service) | `07-cross-cutting-concerns.md` | none | Gain: S1 completeness; Give up: doc size +~30 rows | Reviewer searches §7.1.2 for each new boundary name; every row has STRIDE letter + risk + mitigation reference |
| A6-P2 | blocker | Define a concrete node-authentication primitive in `HANDSHAKE` and `DeploymentConfig`; require it before cross-host peers become `ACTIVE` in `ClusterView` | `modules/transport.md` §2.3/§5.3, `modules/distributed.md` §2.4, `07-cross-cutting-concerns.md` §7.1.3, `10-known-deviations.md` Deviation 3 | none (handshake-time only, slow path) | Gain: S1/S2/S5 on spoofing & tampering; Give up: +1 RTT at `connect`, key-material lifecycle to manage | Run two-node sim with mismatched credentials → handshake rejected with `ErrorCode::AuthenticationFailed`; correct credentials → handshake succeeds and peer enters `participating_nodes` |
| A6-P3 | high | Add S3 input-validation checklist for every `RemotePayload` — declare `max_task_count`, `max_edge_count`, `max_message_bytes`, `max_remote_error_message_bytes`; reject at framing layer before CRC dispatch | `modules/transport.md` §2.3, §8 Configuration | none (receive-path, already a slow path by design budget) | Gain: S3 on DoS / malformed-input; Give up: 1 extra bounds check per frame (< 10 ns) | Fuzz the codec with `task_count = 2^32-1`; the frame must be dropped with `TransportCorrupted` *before* allocation; metric `corrupted_frames` increments |
| A6-P4 | high | Require TLS/mTLS for TCP backend when `peers` span > 1 host; add `DeploymentConfig.require_encrypted_transport: bool = true` default; fail-closed on startup if plaintext + multi-host | `modules/runtime.md` §2, `modules/transport.md` §8, `07-cross-cutting-concerns.md` §7.1.3, `10-known-deviations.md` Deviation 3 | none (startup/slow path) | Gain: S4/S5 secure default across hosts; Give up: deployments must provision cert material before first cluster boot | Start `DistributedRuntime` with two `peers` on different hostnames and `require_encrypted_transport=true`, no credentials → `RuntimeInitError` at `init`; with TLS credentials configured → init succeeds |
| A6-P5 | high | Specify `rkey` scope: per-buffer, single-use (consumed by the receiver's RDMA READ), and revoked on `SUBMISSION_RETIRED` via `IMemoryOps::deregister_peer_read`; document in `memory.md` and `transport.md` | `modules/memory.md`, `modules/transport.md` §2.3 (`RemoteDataReadyPayload`), `modules/distributed.md` §5.3 | none (scheduler slow path at retirement) | Gain: S2 blast-radius reduction for captured messages; Give up: one MR deregister per retired buffer (~1 µs; off hot path per `04-process-view.md:491`) | Instrument the RDMA backend: second RDMA READ using the same `rkey` after `SUBMISSION_RETIRED` must fail with `TransportDisconnected`/`RkeyRevoked`; first succeeds |
| A6-P6 | high | Introduce a security-audit log (`audit/`) distinct from the general log: append-only, structured, signed-per-boot with monotonic sequence numbers; events: node_join, factory_register, function_register, config_change, peer_lost, shutdown, sink_register, tenant_switch | `07-cross-cutting-concerns.md` §7.1 (add §7.1.4 "Audit Trail"), `modules/runtime.md` §5, `modules/profiling.md` §3 | none (security events are rare & non-critical-path; must not allocate on hot path — emit via pre-allocated audit ring) | Gain: S6 compliance; Give up: small RAM budget for audit ring + disk flush policy | Verify via a boot-to-shutdown run that every `register_factory`, `register_function`, and `HANDSHAKE` produces exactly one audit record with a monotonic sequence; gap detection raises `AuditIntegrityAlert` |
| A6-P7 | medium | Require function-binary provenance: `FunctionDesc` carries an `Attestation` (signer id + signature over `hash`); `FunctionRegistry::register_function` rejects unsigned blobs unless `DeploymentConfig.allow_unsigned_functions = false` (default **false** in multi-tenant mode, true in single-tenant / sim) | `modules/core.md` §2.6, `modules/runtime.md` §2.4 `DeploymentConfig`, `07-cross-cutting-concerns.md` §7.1 | none (registration is slow path) | Gain: S1/S3 trust anchor for code-on-NPU; Give up: build-pipeline change for kernel authors; key-management burden | Unit test: register blob without attestation in multi-tenant config → `ErrorCode::FunctionNotAttested`; with valid attestation → `FunctionId` returned |
| A6-P8 | medium | Validate DLPack / CAI imports at the binding boundary: reject negative / mismatched-device / overflowing-stride / unowned-capsule tensors with `ValueError` before reaching `IMemoryOps::register_external`; add per-import max tensor bytes from `DeploymentConfig` | `modules/bindings.md` §2.4, §7, `07-cross-cutting-concerns.md` §7.1.3 bullet 1 | none (Python → C call, already pre-critical-path) | Gain: S3 robustness at the top user boundary; Give up: a few hundred nanoseconds per tensor on submit fast path — absorbed in the `< 0.5 μs` Python→C stage of `04-process-view.md:646` | Pass a DLPack capsule with negative stride / mismatched device-type → `ValueError`; pass a valid capsule → no copy, no error |
| A6-P9 | medium | Enforce Logical System isolation in code: add `logical_system_id` to `MessageHeader`; `distributed_scheduler` rejects cross-tenant inbound messages at `protocol_handler` entry; `ISchedulerLayer::submit` checks submitter tenant against layer tenant | `modules/transport.md` §2.3, `modules/distributed.md` §2.2, `05-physical-view.md` §5.5.2 | none (check is one integer compare on the already-slow Stage-A admission path and framing path) | Gain: S1/S2/S3 tenant isolation actually enforced; Give up: 8 bytes in header; one compare per inbound message (< 5 ns) | Two-tenant sim: tenant-A submits a Task whose `logical_system_id` mismatches the target Layer → rejected with `ErrorCode::CrossTenantDenied`; same-tenant path works |
| A6-P10 | medium | Capability-scope log / trace sinks: `simpler.logging.add_sink(callable, *, severity_min, domains, logical_system_id)`; drop events outside the sink's capability silently (never leak cross-tenant events to a tenant-scoped sink) | `modules/bindings.md` §2.1, `modules/profiling.md` §2 (ISink), `07-cross-cutting-concerns.md` §7.2.4 | none (sink dispatch is on flush thread, not hot path — extra filter is ≤ 2 compares) | Gain: S2 least-privilege for sinks; Give up: API change to `add_sink`; existing callers need a migration default | Register a tenant-A sink with `logical_system_id=A` and emit a tenant-B event → sink does not receive it; a tenant-A event → sink receives it |
| A6-P11 | medium | Constrain `register_factory` to `init`-phase only *and* require the caller to hold a `RegistrationToken` obtained from `DeploymentConfig.factory_registration_token` (configured by the deployer, not present in Python surface by default) | `modules/runtime.md` §2.2 `register_factory`, `modules/bindings.md:51` | none (init-time) | Gain: S2 reduces attack surface from "any Python code" to "deployer-configured initialization hook"; Give up: library authors that currently register factories from plain Python need to read a token from config | Attempt `register_factory` after `Runtime.__init__` returns → `ErrorCode::RegistrationClosed`; before init with wrong token → `ErrorCode::Unauthorized`; with correct token → registered |
| A6-P12 | medium | Define a signed / schema-validated REPLAY trace format and gate `IExecutionEngine` REPLAY on successful validation; reject mismatched `schema_version` / `capture_node_id` / bad signature | `02-logical-view/10-platform.md` §2.8.1, `07-cross-cutting-concerns.md` §7.2.6 bullet "Replay-compatible trace" | none (REPLAY init is explicitly not a production hot path) | Gain: S1/S3 for REPLAY; Give up: build + sign step in capture pipeline | Flip a byte in a captured trace → REPLAY init fails `InvalidReplayTrace`; signed trace → succeeds |
| A6-P13 | low | Add rate-limit + per-tenant quota to C API `submit()` (counter + leaky bucket) — currently only "task slot pool limits per Logical System" is stated (`07-cross-cutting-concerns.md:25`). Budget: counter bump (≤ 1 ns on hot path) with back-pressure only in the slow path | `modules/bindings.md` §2.1, `modules/scheduler.md` (TaskManager admission), `07-cross-cutting-concerns.md` §7.1.2 | none (fast path = single relaxed counter increment; slow path only when quota breached) | Gain: S1 DoS-resistance; Give up: per-tenant config surface | Flood a single tenant; after the quota threshold the call returns `AdmissionDecision::REJECTED` with `ErrorCode::TenantQuotaExceeded`; other tenants unaffected |
| A6-P14 | low | Explicit key-material lifecycle note in `08-design-decisions.md` (new ADR-N): "Transport credentials are externally provisioned; runtime never generates or persists keys; rotation is restart-based in v1" — closes the gap implicit in Deviation 3 | `08-design-decisions.md`, `10-known-deviations.md` §Deviation 3 | none (doc only) | Gain: G5 (justify decisions) + S5 explicit scope; Give up: no functional change | The new ADR exists, cross-linked from Deviation 3 and `transport.md` §12 "TLS / encryption enforcement" |

### Proposal detail

#### A6-P1: Complete the trust-boundary threat model

- **Rationale:** S1 requires *every* trust boundary to be threat-modeled. The current §7.1 table misses (a) untrusted-plug-in factory registration, (b) REPLAY trace-file input, (c) Python-supplied log / trace sinks, (d) multi-tenant Logical-System isolation as a boundary, (e) per-function binary provenance, (f) per-service External-Data endpoints (DFS / KV / DSM / block storage — currently collapsed into a single row). Cites S1, G6.
- **Edit sketch:**
  - File: `docs/pypto-runtime-design/07-cross-cutting-concerns.md`
  - Location: §7.1.1 trust-boundary table and §7.1.2 STRIDE table
  - Delta: Add six rows with STRIDE categories & risk; link each to a mitigation bullet in §7.1.3. Keep prose otherwise unchanged.
- **Trade-offs:**
  - Gain: S1 rubric passes; reviewers can find every entry point in one place.
  - Give up: Doc grows by ~30 rows; minor maintenance each time a new boundary is added.
- **Sanity test:** Grep §7.1 for each new boundary name (e.g., "Plug-in Factory", "REPLAY Trace", "Log Sink", "Tenant Isolation", "Function Binary"); every term has both a row in §7.1.1 and a row in §7.1.2 with a STRIDE letter and a mitigation reference.

#### A6-P2: Concrete node authentication in `HANDSHAKE`

- **Rationale:** S1 (spoofing in the Node↔Node row of `07-cross-cutting-concerns.md:29`) is mitigated only by "reject unknown node IDs," which is not an authentication primitive. S2 (least privilege) requires every peer to carry verifiable identity bound to a credential. `modules/transport.md:83-98` defines `HANDSHAKE` but no credential field exists. Cites S1, S2.
- **Edit sketch:**
  - File: `docs/pypto-runtime-design/modules/transport.md` §2.3 and §5.3
  - Location: `MessageType::HANDSHAKE` payload and handshake flow
  - Delta: Add `HandshakePayload { uint64_t node_id; bytes nonce; bytes signature_over_nonce_and_version; uint32_t credential_id; }`; extend §5.3 to "verify signature against the credential pinned in `DeploymentConfig.peers[].public_key`; on fail → `ErrorCode::AuthenticationFailed`; peer is *never* added to `participating_nodes`."
  - Mirror in `modules/distributed.md` §2.4 (`ClusterView` gains a `verified: bool` flag) and `07-cross-cutting-concerns.md` §7.1.3 (add "Node authentication" bullet spelling the primitive).
- **Trade-offs:**
  - Gain: Hard S1 mitigation; S2 basis; keys the entire downstream trust model.
  - Give up: Deployers must distribute keys; +1 RTT at `connect` (handshake-time only, not on any hot path).
- **Sanity test:** Two-node sim with mismatched public keys → handshake fails with `ErrorCode::AuthenticationFailed` and the peer never appears in `cluster_view.participating_nodes`; with correct keys → peer appears with `verified=true`.

#### A6-P3: Bounded variable-length payload parsing

- **Rationale:** S3 requires validation of format / type / range at trust boundaries. `RemoteSubmitPayload::task_count` / `edge_count` (`modules/transport.md:104-112`), `tensor_meta_bytes`, and `message_bytes` (`modules/transport.md:138-143`) drive trailing-array parses without declared maxima. CRC32 passes on any byte sequence, so "framing is integrity-protected" does not replace bounds validation. Cites S3, X2 (DoS allocation).
- **Edit sketch:**
  - File: `modules/transport.md` §2.3 and §8 Configuration
  - Location: after the payload structs and in the configuration table
  - Delta: Add config keys `max_task_count_per_remote_submit` (default 1024), `max_edge_count_per_remote_submit` (default 4096), `max_remote_error_message_bytes` (default 4 KiB), `max_tensor_meta_bytes` (default 16 KiB). In §7 Error Handling, add row "payload-bound exceeded → `TransportCorrupted`, drop connection."
- **Trade-offs:**
  - Gain: S3 robustness; eliminates a full class of DoS / heap-blowup attacks from a compromised peer.
  - Give up: ≤ 10 ns per frame (≤ 4 bounds compares); config surface +4 keys.
- **Sanity test:** Fuzz the codec with `task_count = 2^31`; the frame is dropped before `message_factory` allocates anything; `ChannelStats.corrupted_frames` increments.

#### A6-P4: Secure-default TLS for multi-host TCP

- **Rationale:** S4 / S5 — current `TRANSPORT_BACKEND` default is `tcp` (`modules/transport.md:393`) with plaintext acceptance baked into Deviation 3 (`10-known-deviations.md:31-40`). Silent plaintext multi-host traffic is not a secure default. Cites S4, S5.
- **Edit sketch:**
  - File: `modules/runtime.md` §2.4 `DeploymentConfig` and `modules/transport.md` §8
  - Location: add config field and init-time check
  - Delta: Add `require_encrypted_transport: bool = true`; in `runtime/init`, if any two `peers` resolve to different hosts and the chosen backend is plaintext TCP, raise `ErrorCode::InsecureTransport`. Update `10-known-deviations.md` Deviation 3 to reflect "enforced at init unless explicitly disabled."
- **Trade-offs:**
  - Gain: S4/S5 — no accidental plaintext clusters.
  - Give up: Deployments must provision cert material; single-host loopback tests unaffected.
- **Sanity test:** Configure `DistributedRuntime` with two hostnames and no TLS credentials → `RuntimeInitError(InsecureTransport)` at `init`; pass TLS creds → init proceeds.

#### A6-P5: Scoped, revocable RDMA `rkey`

- **Rationale:** S2/S3 — `REMOTE_DATA_READY{global_address, rkey, size}` (`modules/transport.md:130-136`, `modules/distributed.md:354-357`) grants direct RDMA READ access to peer memory. No scope or revocation story exists. A replayed or captured message remains usable as long as the MR is live.
- **Edit sketch:**
  - File: `modules/memory.md` §2 (IMemoryOps) and `modules/transport.md` §2.3 `RemoteDataReadyPayload`
  - Location: `IMemoryOps` API and payload doc
  - Delta: Add `IMemoryOps::register_peer_read(BufferRef, NodeId peer, Scope scope) -> RkeyToken` returning a single-use or Submission-lifetime rkey; add `IMemoryOps::deregister_peer_read(RkeyToken)`; document that `distributed_scheduler` calls `deregister` on `SUBMISSION_RETIRED` (`04-process-view.md:491`, `modules/distributed.md` §5.3).
- **Trade-offs:**
  - Gain: Blast radius for a captured message ≤ one Submission window; S2 strengthens.
  - Give up: One MR deregister per retired buffer (~1 µs on the scheduler slow path). Off hot path.
- **Sanity test:** Instrument the RDMA backend to replay an old `REMOTE_DATA_READY` after `SUBMISSION_RETIRED` → RDMA READ fails with `TransportDisconnected` / `RkeyRevoked`; first read succeeds.

#### A6-P6: Security audit trail

- **Rationale:** S6 requires a tamper-evident audit trail. None exists — `07-cross-cutting-concerns.md` §7.2.4 is a general log only, with no integrity chain, no severity-class filter, no append-only discipline. Cites S6.
- **Edit sketch:**
  - File: `07-cross-cutting-concerns.md` add §7.1.4 "Audit Trail"; `modules/runtime.md` §5 add audit emit points; `modules/profiling.md` §3 add `IAuditSink` alongside `ILogSink`.
  - Location: new subsection + ~1 emit call per security event site (node_join, factory_register, function_register, config_change, peer_lost, shutdown, sink_register, tenant_switch)
  - Delta: Define `AuditEvent { timestamp_ns; sequence_no; principal_id; action; subject; outcome; prev_hash; }` — hash-chained per-boot, flushed by a dedicated audit sink (file + syslog supported). Pre-allocated audit ring; no hot-path allocation.
- **Trade-offs:**
  - Gain: S6 passes; regulatory / operational observability.
  - Give up: RAM budget for ring (+ a few KiB); disk-flush policy.
- **Sanity test:** Boot runtime, register a factory, register a function, start a cluster, shut down → audit log contains exactly those events in order, each with monotonic `sequence_no` and correct `prev_hash`; flipping a byte in the file detected by chain-check on reload.

#### A6-P7: Function-binary attestation

- **Rationale:** S1/S3 — the runtime loads arbitrary binaries via `FunctionRegistry::register_function` (`modules/core.md:257-272`) and executes them on the NPU. Content hash is for idempotency (`08-design-decisions.md` ADR-007, cited by `modules/core.md:247`), not for trust.
- **Edit sketch:**
  - File: `modules/core.md` §2.6 `FunctionDesc`; `modules/runtime.md` §2.4 `DeploymentConfig`; `07-cross-cutting-concerns.md` §7.1.3 mitigations
  - Location: add `Attestation { key_id; signature_over_hash; }` field to `FunctionDesc`; add `allow_unsigned_functions: bool = false` to `DeploymentConfig` (default `false` in multi-tenant mode; `true` default only in single-tenant / `SIM` variant).
  - Delta: `register_function` verifies against `DeploymentConfig.trusted_signers`; unsigned + `allow_unsigned=false` → `ErrorCode::FunctionNotAttested`.
- **Trade-offs:**
  - Gain: Trust anchor for AICore code; S1/S3.
  - Give up: Kernel-author / tooling must produce signed artifacts; key management.
- **Sanity test:** In multi-tenant config, register an unsigned blob → `FunctionNotAttested`; with a valid signature against a configured signer → returns `FunctionId`.

#### A6-P8: Boundary validation for DLPack / CAI imports

- **Rationale:** S3 — `bindings/` imports arbitrary DLPack / CAI tensors (`modules/bindings.md:109-114`) with no documented validation. Cites S3.
- **Edit sketch:**
  - File: `modules/bindings.md` §2.4 and §7
  - Location: DLPack interop subsection
  - Delta: Add a "Validation" bullet list: "Before calling `IMemoryOps::register_external` the binding rejects (a) dtype not in the runtime-known set, (b) device-type mismatched with target layer, (c) shape / stride with overflowing product or negative values, (d) zero-length dtype × stride, (e) capsule not owned by the caller. Maximum bytes per imported tensor clamped at `DeploymentConfig.max_import_bytes` (default 1 GiB).". Corresponding check-list entries in `07-cross-cutting-concerns.md` §7.1.3 bullet 1.
- **Trade-offs:**
  - Gain: S3 robustness at the top user boundary.
  - Give up: A few hundred ns per import, absorbed in the `< 0.5 μs` Python→C API stage (`04-process-view.md:646`).
- **Sanity test:** Pass a capsule with negative stride → `ValueError`; pass a capsule from a different device → `ValueError`; pass a valid contiguous tensor → no copy, no error.

#### A6-P9: Enforce Logical System isolation in wire + code

- **Rationale:** S1/S2/S3 — `05-physical-view.md:237-246` asserts isolation but the wire format (`modules/transport.md:76-98`) has no tenant field and no enforcement point is specified in `distributed_scheduler` or `ISchedulerLayer::submit`.
- **Edit sketch:**
  - File: `modules/transport.md` §2.3 `MessageHeader`; `modules/distributed.md` §2.2 `IDistributedProtocolHandler`; `05-physical-view.md` §5.5.2
  - Location: header layout + inbound dispatch + submit
  - Delta: Add `uint32_t logical_system_id;` to `MessageHeader` (bump `version`); protocol handler drops inbound messages whose `logical_system_id` does not match the receiving Layer (metric `cross_tenant_rejected` increments); `ISchedulerLayer::submit` checks caller tenant against Layer tenant → `ErrorCode::CrossTenantDenied`.
- **Trade-offs:**
  - Gain: S1/S2/S3 — isolation actually enforced.
  - Give up: 4 bytes per header; one integer compare per inbound message (≤ 5 ns, off hot path).
- **Sanity test:** Two-tenant sim: tenant-A submits to a tenant-B Layer → `CrossTenantDenied`; same-tenant submit works; forged inbound message with other tenant id → dropped silently with metric increment.

#### A6-P10: Capability-scoped log / trace sinks

- **Rationale:** S2 — a Python `ILogSink` currently sees every log event globally (`modules/bindings.md:321`, `07-cross-cutting-concerns.md` §7.2.4). In a multi-tenant deployment that is a cross-tenant information leak.
- **Edit sketch:**
  - File: `modules/bindings.md` §2.1 `simpler.logging.add_sink`; `modules/profiling.md` §2 `ISink`; `07-cross-cutting-concerns.md` §7.2.4
  - Location: `add_sink` signature
  - Delta: `add_sink(callable, *, severity_min='INFO', domains=None, logical_system_id=None) -> SinkHandle`; the dispatch loop filters events against the sink's capability set; callers without `logical_system_id=None` (the "admin" default) get only their tenant.
- **Trade-offs:**
  - Gain: S2 least privilege for sinks.
  - Give up: `add_sink` signature change; migration default = admin (backward-compatible for single-tenant).
- **Sanity test:** Register a tenant-A sink with `logical_system_id=A`; emit a tenant-B event → sink does not receive it; tenant-A event → sink receives it.

#### A6-P11: Gated `register_factory`

- **Rationale:** S2 — `register_factory` is described as "expert use; typically called by compiled C++ modules, not Python" (`modules/bindings.md:51`) but is exposed in Python and has no privilege check. An attacker with Python code execution can register arbitrary factories and subvert the runtime. Cites S2, S4.
- **Edit sketch:**
  - File: `modules/runtime.md` §2.2; `modules/bindings.md:51`
  - Location: `register_factory` signature
  - Delta: Extend signature to `register_factory(descriptor, *, registration_token)`. The token is set via `DeploymentConfig.factory_registration_token` by the deployer; absent token → `ErrorCode::Unauthorized`. Registration is rejected once `freeze()` has run: `ErrorCode::RegistrationClosed` (`modules/runtime.md:79`).
- **Trade-offs:**
  - Gain: S2 reduces attack surface from "any Python" to "deployer-provisioned init hook."
  - Give up: Library authors must read the token from config in init code.
- **Sanity test:** Post-`__init__` `register_factory` → `RegistrationClosed`; pre-init with wrong token → `Unauthorized`; with correct token → registered.

#### A6-P12: Signed / schema-validated REPLAY trace

- **Rationale:** S1/S3 — `PERFORMANCE` / `REPLAY` modes reconstruct FSM state from a trace file (`02-logical-view/10-platform.md:32`, `07-cross-cutting-concerns.md:130`). Malformed / malicious traces are an injection vector into the FSM. Cites S1, S3.
- **Edit sketch:**
  - File: `02-logical-view/10-platform.md` §2.8.1; `07-cross-cutting-concerns.md` §7.2.6 replay bullet
  - Location: REPLAY mode description
  - Delta: Add a "trace format integrity" paragraph: "REPLAY traces carry a header with `schema_version`, `capture_node_id`, `content_hash`, and a detached signature over the content hash. `IExecutionEngine` REPLAY init verifies the header against `DeploymentConfig.trusted_signers`; failure → `ErrorCode::InvalidReplayTrace`."
- **Trade-offs:**
  - Gain: S1/S3 for REPLAY.
  - Give up: Capture pipeline must sign artifacts.
- **Sanity test:** Flip a byte in a captured trace → `InvalidReplayTrace` at init; untouched → REPLAY starts.

#### A6-P13: Per-tenant submit rate-limit

- **Rationale:** S1 DoS — `07-cross-cutting-concerns.md:25` states "Task slot pool limits per Logical System" as the only DoS mitigation. A single tenant can still monopolize scheduling up to pool size; per-tenant rate-limit gives graceful back-pressure. Cites S1.
- **Edit sketch:**
  - File: `modules/bindings.md` §2.1; `modules/scheduler.md` TaskManager admission path; `07-cross-cutting-concerns.md` §7.1.2
  - Location: submit admission
  - Delta: Add `DeploymentConfig.per_tenant_submit_qps: uint32 | None`; admission installs a relaxed counter + periodic refill (leaky bucket) — increment in fast path (≤ 1 ns), branch-on-breach in slow path only. On breach → `AdmissionDecision::REJECTED` with `ErrorCode::TenantQuotaExceeded`.
- **Trade-offs:**
  - Gain: S1 DoS hardening.
  - Give up: Config surface; per-tenant counter (cache-aligned).
- **Sanity test:** Flood tenant A at 2 × quota → half of submits rejected with `TenantQuotaExceeded`; tenant B's submits unaffected.

#### A6-P14: Key-material lifecycle ADR

- **Rationale:** G5 / S5 — Deviation 3 (`10-known-deviations.md:31-40`) defers encryption to the backend but never states *how* keys are managed. An explicit ADR closes that gap.
- **Edit sketch:**
  - File: `08-design-decisions.md` new ADR-N "Transport Key Material Lifecycle"; cross-link from `10-known-deviations.md` Deviation 3 and `modules/transport.md` §12
  - Location: new ADR at end of file; cross-link bullets
  - Delta: State "Transport credentials (TLS certs, RDMA key material) are externally provisioned via `DeploymentConfig`; the runtime never generates or persists keys; rotation is restart-based in v1; multi-rotation without restart is a v2 extension."
- **Trade-offs:**
  - Gain: G5 + S5 explicit scope.
  - Give up: No functional change.
- **Sanity test:** The ADR exists, is linked from Deviation 3 and `transport.md` §12, and names v1 scope plus a v2 extension path.

## 5. Revisions of own proposals  (round ≥ 2 only)

_Not applicable in round 1._

## 6. Votes on peer proposals  (round ≥ 2 only)

_Not applicable in round 1._

## 7. Cross-aspect tensions

| pair | with_proposal | preferred_resolution |
|------|---------------|----------------------|
| A6 vs A1 | A6-P3 (payload bounds check) | Fast path / slow path: receive-path bounds check is already off the < 50 μs cross-node submit budget (§4.8.2) — bounds check executes only in the codec framing stage which is already a slow path relative to the hot-path RDMA transit. A1 should see no hot-path regression. |
| A6 vs A1 | A6-P8 (DLPack validation) | Validation lives in the Python → C stage (< 0.5 μs in §4.8.1). Additional ~200 ns per tensor fits the existing budget; keep the fast path single-function-call with inline bounds checks (no heap). |
| A6 vs A1 | A6-P9 (`logical_system_id` on inbound dispatch) | Single integer compare; off hot path. Add the field at a non-critical position in `MessageHeader` so the cache line containing `correlation_id` / `source_node` remains intact. |
| A6 vs A1 | A6-P13 (per-tenant QPS limit) | Two-tier: fast path = relaxed atomic counter increment (≤ 1 ns); slow path (on refill tick) = leaky-bucket recompute. A1 should find zero hot-path cost in the steady state. |
| A6 vs A1 | A6-P5 (rkey revocation) | Revocation is attached to `SUBMISSION_RETIRED` (`04-process-view.md:491`), which is explicitly the slow path; no hot-path cost. |
| A6 vs A2 | A6-P6 (audit trail) | Audit is a *new* sink type, added via the existing profiling sink plug-in interface (`modules/profiling.md` §2). It is an OCP extension, not a modification of existing sinks — aligns with A2 (E4). |
| A6 vs A9 | A6-P6 (audit trail) | A9 may object "YAGNI" — preempt with: the framework ships only the interface + an in-memory ring; file/syslog sinks ship as separate OCP extensions. The audit event list is finite and small; this is a single abstraction that earns its keep on S6. |
| A6 vs A9 | A6-P7 (function attestation) | A9 may flag the attestation field as speculative for single-tenant users. Preempt: default `allow_unsigned_functions=true` in single-tenant / `SIM`, so the cost is zero for those users; attestation only kicks in for multi-tenant production. |
| A6 vs A5 | A6-P2 (node auth) + A6-P4 (TLS default) | Both add init-time steps that can *fail*. A5 should require that the failure surface is classified `Fatal` per §4.7.5 (it is — both are `InitFailed` family). Graceful degradation is explicitly **not** offered for these — secure-by-default requires fail-closed. |

## 8. Stress-attack on emerging consensus  (round 3+ only)

_Not applicable in round 1._

## 9. Status

- **Satisfied with current design?** partially
- **Open items expected in next round:**
  - `A6-P2` (node authentication) — blocker: must converge on a single primitive (mTLS vs pre-shared key vs SPIFFE) before round 2 close.
  - `A6-P4` (TLS default) — high: acceptance depends on whether deployers are ready to provision keys pre-boot; may get pushback from A9 (KISS) and A5 (availability).
  - `A6-P6` (audit trail) — high: likely A9 pressure; expect to defend the minimal surface.
  - `A6-P7` (function attestation) — medium: requires kernel-author / tooling alignment; may need to split into "interface now, enforcement later."
  - `A6-P9` (tenant id on header) — medium: A4 (doc consistency) will want this mirrored in glossary + interfaces.
  - `A6-P11` (gated `register_factory`) — medium: depends on A2's view of plug-in registration as an OCP extension vs API break.
