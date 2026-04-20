# Edit: docs/pypto-runtime-design/modules/transport.md

- **Driven by proposal(s):** A1-P3, A1-P6, A2-P1, A6-P2, A6-P3, A6-P4, A6-P9, A7-P4, A8-P3, A10-P5
- **ADR link(s):** ADR-015 (I-DIST-1); ADR-020 (coordinator generation)
- **Timestamp (UTC):** 2026-04-19
- **Edit kind:** additive (consolidated `## 13. Review Amendments (R3)` section appended before Document status footer)

## Location

- Path: `docs/pypto-runtime-design/modules/transport.md`
- Section / heading: new `## 13. Review Amendments (R3)`; references §2.3, §2.4, §3.1, §4.3, §5.1, §5.3, §8
- Line range before edit: 457–458 (footer only)

## Before

```457:458:docs/pypto-runtime-design/modules/transport.md
**Document status:** Draft — ready for review.
```

## After

A new `## 13. Review Amendments (R3)` heading is inserted before the footer, with one `[UPDATED: <id>: ...]` quoted callout per assigned proposal. Per-proposal edits follow.

## Per-proposal edits

### A1-P3 — HEARTBEAT `function_bloom[4]`

- **Before:** Heartbeat payload had no function-cache-presence field.
- **After (callout in §13):**

> [UPDATED: A1-P3: function_bloom in HeartbeatPayload] Four `uint64_t` Bloom-filter advertising each peer's Function Cache; coordinator checks before inlining in REMOTE_SUBMIT.

- **Rationale:** P2 — coordinator avoids redundant binary shipping.

### A1-P6 — `REMOTE_BINARY_PUSH` MessageType + descriptor template registry

- **Before:** Function binaries could inflate `REMOTE_SUBMIT` size.
- **After (callout in §13):**

> [UPDATED: A1-P6: REMOTE_BINARY_PUSH + descriptor_template_id] New MessageType for staging binaries pre-submit (gated by function_bloom). `RemoteSubmitPayload.descriptor_template_id` + delta-encoded TaskDescriptor[]; per-peer registry; first-use template-miss ≤ 15 μs.

- **Rationale:** P6 / X9 — bounds REMOTE_SUBMIT size and cold-start cost.

### A2-P1 — `schema_version` on `MessageHeader`

- **Before:** Wire headers lacked explicit schema version.
- **After (callout in §13):**

> [UPDATED: A2-P1: schema_version on wire headers] `uint16_t schema_version` added to `MessageHeader` (covers TaskDescriptor, FunctionDescriptor, stats). Unknown additive fields tolerated by v1 readers.

- **Rationale:** E6 / E2.

### A6-P2 — HANDSHAKE authentication + `coordinator_generation`

- **Before:** `HandshakePayload` lacked authentication and generation fields.
- **After (callout in §13):**

> [UPDATED: A6-P2: authenticated HANDSHAKE + coordinator_generation] `HandshakePayload { node_id; nonce; signature_over_nonce_and_version; credential_id; coordinator_generation }`. v1 mTLS cert-pinned; failure → `AuthenticationFailed`; `ClusterView.verified`; `StaleCoordinatorClaim` rejects mismatched generation (ADR-020).

- **Rationale:** S1 / S2 / S5.

### A6-P3 — Bounded payload parsing

- **Before:** Parsing lacked a length-prefix entry guard.
- **After (callout in §13):**

> [UPDATED: A6-P3: length-prefix guard + caps] `max_task_count_per_remote_submit` (1024); `max_edge_count_per_remote_submit` (4096); `max_remote_error_message_bytes` (4 KiB); `max_tensor_meta_bytes` (16 KiB). Overflow drops the frame with `TransportCorrupted`; `corrupted_frames` metric increments.

- **Rationale:** S3 / X2.

### A6-P4 — Default-encrypted TCP + fail-closed

- **Before:** Plaintext TCP was possible without an init-time check.
- **After (callout in §13):**

> [UPDATED: A6-P4: require_encrypted_transport=true] Multi-host plaintext TCP fails init with `ErrorCode::InsecureTransport`. RDMA uses RC + rkey scoping (A6-P5) — explicit exemption. Loopback unaffected.

- **Rationale:** S4 / S5.

### A6-P9 — `logical_system_id` on `MessageHeader`

- **Before:** MessageHeader had no tenant scoping; cross-tenant inbound was silently routed.
- **After (callout in §13):**

> [UPDATED: A6-P9: logical_system_id (off hot 64-B line)] Add `uint32_t logical_system_id` outside the 64-B hot cache line; bump header version. Protocol handler drops cross-tenant inbound → `cross_tenant_rejected` metric + `CrossTenantDenied` error. ≤ 1 compare on slow framing path.

- **Rationale:** S1 / S2 / S3 — tenant isolation at wire level.

### A7-P4 — Move payload structs to `distributed/`

- **Before:** `transport/messages.h` owned all `RemoteXxxPayload` / `HeartbeatPayload`.
- **After (callout in §13):**

> [UPDATED: A7-P4: payload structs → distributed/protocol_payloads.h] Transport retains only `MessageHeader`, framing, `MessageType` tags. Public API narrows to `send(peer, MessageType, span<const std::byte>, Timeout)`. Invariant I-DIST-1 enforced via IWYU-CI (ADR-015).

- **Rationale:** D3 / D5 / SRP.

### A8-P3 — `ChannelStats` schema

- **Before:** Channel stats surface not enumerated.
- **After (callout in §13):**

> [UPDATED: A8-P3: ChannelStats schema] Enumerate `ChannelStats` with concrete fields + units (bytes-in/out, frames dropped, corrupted_frames, queue occupancy). Latency histograms share A8-P3 primitive.

- **Rationale:** O3 / O5.

### A10-P5 — Per-peer REMOTE_SUBMIT projection (absorbed into A1-P6)

- **Before:** Partitioner shipped the full submission everywhere.
- **After (callout in §13):**

> [UPDATED: A10-P5: per-peer projection] Partitioner emits per-peer sub-submissions covering only tasks/edges/boundaries touching that peer subset; shared `correlation_id` keeps `REMOTE_DEP_NOTIFY` joinable.

- **Rationale:** P6 — minimizes per-peer payload.

## Verification steps

1. `rg -n "\[UPDATED: " docs/pypto-runtime-design/modules/transport.md` prints 10 entries in §13.
2. `rg -n "REMOTE_BINARY_PUSH|descriptor_template_id" docs/pypto-runtime-design/modules/transport.md` surfaces A1-P6.
3. `rg -n "logical_system_id" docs/pypto-runtime-design/modules/transport.md` surfaces A6-P9.
4. `rg -n "schema_version" docs/pypto-runtime-design/modules/transport.md` surfaces A2-P1.
5. Cross-view: `distributed.md` owns `protocol_payloads.h` after A7-P4 move.
