# Appendix C — Backward Compatibility Policy

This appendix is the central backward-compatibility (BC) policy index introduced by proposal **A2-P5** and the ADR citation convention introduced by proposal **A4-P6**. It consolidates versioning scope, per-interface stability classes, versioning checklists, unknown-tag tolerance, and signed-schema policy. Where an interface pins a concrete ADR, that ADR is cited inline in the column "First-added-in"; views `03–06` link to this appendix from their "Related ADRs" paragraphs (A4-P6).

Cross-links: [`02-logical-view/09-interfaces.md`](02-logical-view/09-interfaces.md), [`modules/runtime.md`](modules/runtime.md), [`modules/distributed.md`](modules/distributed.md), [`modules/transport.md`](modules/transport.md), [`modules/profiling.md`](modules/profiling.md), [`appendix-a-glossary.md`](appendix-a-glossary.md).

---

## C.1 Versioning scope

Following the R3 amendment to **A2-P1**, the blanket `schema_version` obligation is narrowed to:

1. **Multi-node wire messages** — every message whose wire format crosses a node boundary: `MessageHeader`, `TaskDescriptor`, `FunctionDescriptor`, `HandshakePayload`, `HeartbeatPayload`, and the distributed stats messages.
2. **Persistent artifacts** — on-disk artifacts read back across binary versions: the trace-file header (A2-P9) and any dump/replay file produced by the runtime.

Non-wire in-process types (e.g. in-process structs passed through role-interface seams such as `ISchedulerSubmit`) are **not** versioned with a byte; they rely on the A2-P5 cross-reference checklist (§C.3 below) and on header-independence CI (A7-P4, A8-P11) to catch compatibility regressions at build time.

Validation of `schema_version` runs at two well-defined boundaries only:

- **`bindings/`** — Python↔C boundary (A1-P11).
- **Distributed handshake / message-entry** — at `HANDSHAKE` time for peer-pair negotiation, and at `MessageHeader` parse time for per-message rejection (`ProtocolVersionMismatch`).

---

## C.2 Per-interface BC table

Stability classes (per A2-P5):

- **stable-frozen** — wire format is frozen at v1; any change requires a new type name or a major schema bump.
- **stable-additive** — additive evolution only (append fields, never reorder/rename); `uint16_t schema_version` byte plus unknown-tag tolerance (§C.4).
- **versioned-subinterface** — the interface has explicit sub-versioning via a version field and may expose v2 methods alongside v1.
- **unstable** — pre-v1; breaking changes allowed between minor runtime releases.

| Interface | Versioning style | First-added-in | BC guarantees | Owner |
|-----------|------------------|----------------|---------------|-------|
| `MessageHeader` | stable-frozen | v1 (ADR-015) | Wire shape frozen at v1; `schema_version` byte present; framing bytes and `MessageType` tag set never reordered; new tags appended only. | `transport/` |
| `TaskDescriptor` | stable-additive | v1 | `uint16_t schema_version` + unknown-tag tolerance (receivers ignore unknown additive fields); `descriptor_template_id` (A1-P6) enables delta encoding. Breaking changes require wire-message rev and peer handshake rejection. | `distributed/` (payload); `core/` (in-proc analogue) |
| `FunctionDescriptor` | stable-additive | v1 | `uint16_t schema_version`; new fields appended only; binary-hash identity preserved across additive evolutions. | `distributed/` / `core/` |
| `HandshakePayload` | stable-additive | v1 (ADR-020) | Carries `schema_version`, `coordinator_generation` (A6-P2), `cluster_view` generation; stale-coordinator mismatches rejected with `StaleCoordinatorClaim`. | `distributed/` |
| `HeartbeatPayload` | stable-additive | v1 | Carries `schema_version`, `function_bloom` (A1-P3), peer-health counters; additive-only evolution; dropped messages tolerated. | `distributed/` |
| Trace schema (trace-file header) | stable-frozen (v1) | v1 (A2-P9, ADR-011-R2) | One-off header `{magic, trace_schema_version: uint16_t, platform, mode}`; `replay_engine` rejects mismatched major version with `ProtocolVersionMismatch`; header-only, not per-event. | `profiling/`, `hal/` |
| `dump_state()` schema | versioned-subinterface | v1 (A8-P4) | `Runtime::dump_state(DumpOptions) → StateDump` returns structured JSON with a top-level `schema_version`; additive fields are stable; non-additive changes require a new sub-schema keyed by `schema_version`. Default scope is the caller's `logical_system_id`; cross-tenant reads require `diagnostic_admin_token`. | `runtime/`, `bindings/` |
| `AlertRule` schema | stable-additive | v1 (A8-P5) | `{metric, op, threshold, window_ms, severity, logical_system_id}`; externalized; additive-only evolution; unknown keys ignored by the rule parser (warn-and-skip). | `runtime/`, `profiling/` |
| *(reserved)* | — | — | Reserved row for future public wire / persistent contracts; entries land here before their first multi-node ship. | — |

---

## C.3 Versioning checklist for new fields

Every new public wire or persistent field must satisfy the following (per A2-P5 cross-reference checklist). This checklist is surfaced in review of any PR that touches a versioned header.

1. **Identify the interface class.** Pick one of `{stable-frozen, stable-additive, versioned-subinterface, unstable}`; record in this appendix.
2. **Reserve a schema-version bump (if needed).** Additive fields on `stable-additive` interfaces require **no** bump; the field is appended after the current last field with a known default. Non-additive changes require a bump, and this appendix's C.2 row MUST be updated in the same PR.
3. **Update `schema_version` consumers.** Enumerate every reader of the interface (cross-reference every `modules/*.md` header that `#include`s the type) and confirm either (a) additive-only read path or (b) explicit version-gated branch.
4. **Unknown-tag tolerance.** Confirm the receiver applies the rules in §C.4 for any new optional fields placed into `extension_map`.
5. **Deprecation procedure.** For removal or renames: mark the field deprecated, ship a warning for one minor runtime release, then remove. Cross-reference the ADR or known-deviation entry that authorizes the removal.
6. **Header-independence lint.** Run the A8-P11 IWYU/include-graph lint to confirm the header-independence invariant (A7-P4, Invariant I-DIST-1) has not been broken by the new field's include fan-out — specifically, that `distributed/` headers remain non-includable from `transport/`.
7. **Handshake compat.** If the field appears in `HandshakePayload` or any other message inspected at handshake time, add an explicit negotiation step in `modules/distributed.md` and confirm `ProtocolVersionMismatch` / `StaleCoordinatorClaim` paths still close cleanly.
8. **Appendix-C update.** Add or amend the row in §C.2 with the new First-added-in version and any change to BC guarantees; record the driving ADR / proposal id.

---

## C.4 Unknown-tag tolerance

Per A2-P2 (closed-for-v1 `LevelOverrides`, schema-registered v2) and A2-P1 R3 narrowing:

- **Closed v1 enums.** All hot-path closed enums — e.g. `DepMode`, `AdmissionStatus`, `MessageType` (for tags declared at v1) — are **closed `enum class`** values. Readers MUST reject unknown values with a typed error (`ProtocolVersionMismatch` on the wire; `InvalidArgument` on the in-process boundary). These enums are enumerated in §C.2 as `stable-frozen`.
- **Open extension maps.** For `stable-additive` interfaces, unknown additive fields are carried through an `extension_map` (or equivalent versioned tail). v1 readers MUST:
  - Ignore unknown `extension_map` keys silently when parsing.
  - Preserve and forward unknown keys when the message is relayed (e.g. by `distributed/` proxies) so that a peer at a higher schema version can still interpret them end-to-end.
  - NOT use unknown fields to drive admission, dispatch, or dependency decisions.
- **Bounded parsing.** All receive paths apply the A6-P3 entry-gate guard on payload size before parsing; unknown fields do not lift those bounds.

This rule-set keeps the hot-path closed-enum policy (ADR-017) intact while still allowing additive wire evolution.

---

## C.5 Signed schema policy

Per the A6-P12 R3 reframe and ADR-011-R2:

- **v1 ships a signed envelope + a frozen schema.** Trace artifacts (A2-P9) and any other persistent artifact whose consumers live outside the producing binary MUST be wrapped in a signed envelope `{schema_version, capture_node_id, content_hash, detached_signature}`. The inner payload's wire format is frozen at ADR-011-R2 time.
- **v1 does NOT ship a REPLAY engine.** The REPLAY `IExecutionEngine` is deferred to v2 per A9-P6 option (iii) and Q17. v1 binaries still produce and sign the envelope so that artifacts captured today are verifiable by the v2 REPLAY engine without format migration.
- **Schema rotation.** Any schema change (additive or otherwise) bumps the envelope's `schema_version` and is recorded as a row update in §C.2. Signing keys are externally provisioned per A6-P14 and rotate by restart in v1.
- **Verification contract.** Consumers MUST verify the envelope signature and match `schema_version` against their supported range before trusting any inner field. Mismatches raise `ProtocolVersionMismatch`.

---

## C.6 Change log

| Date | Change | Driven by |
|------|--------|-----------|
| 2026-04-18 | Initial appendix introduced as central BC policy index (versioning scope, per-interface table, checklists, unknown-tag tolerance, signed schema policy). | A2-P5, A2-P1, A4-P6 |
