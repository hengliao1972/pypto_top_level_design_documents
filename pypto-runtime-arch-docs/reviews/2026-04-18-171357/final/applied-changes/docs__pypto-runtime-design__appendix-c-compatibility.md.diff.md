# Edit: docs/pypto-runtime-design/appendix-c-compatibility.md

- **Driven by proposal(s):** A2-P1 (R3-narrowed scope), A2-P5 (central BC policy index), A4-P6 (ADR citation style), A2-P2 (closed-v1 / open extension_map), A2-P9 (trace schema header), A6-P12 (signed envelope + frozen schema v1; REPLAY engine v2), A8-P4 (`dump_state()` schema), A8-P5 (`AlertRule` schema)
- **ADR link (if any):** ADR-011-R2 (SimulationMode + signed envelope), ADR-015 (Invariant I-DIST-1), ADR-017 (closed-enum policy), ADR-020 (coordinator_generation)
- **Timestamp (UTC):** 2026-04-19
- **Edit kind:** structural (new file)

## Location

- Path: `docs/pypto-runtime-design/appendix-c-compatibility.md` (new)
- Section / heading: whole-file creation
- Line range before edit: — (file did not exist); after edit: 1–77

## Before

*(file did not exist prior to this edit)*

## After

New file structure (section headings; see the file itself for the full content):

```markdown
# Appendix C — Backward Compatibility Policy

<intro paragraph referencing A2-P5 and A4-P6; cross-links to 02-logical-view/09-interfaces.md, modules/runtime.md, modules/distributed.md, modules/transport.md, modules/profiling.md, appendix-a-glossary.md>

## C.1 Versioning scope
<A2-P1 R3 narrowing: multi-node wire messages + persistent artifacts only; non-wire in-process types rely on the A2-P5 checklist + A7-P4/A8-P11 lints; validation runs at bindings + handshake/MessageHeader boundaries>

## C.2 Per-interface BC table
| Interface | Versioning style | First-added-in | BC guarantees | Owner |
|-----------|------------------|----------------|---------------|-------|
| MessageHeader | stable-frozen | v1 (ADR-015) | … | transport/ |
| TaskDescriptor | stable-additive | v1 | … | distributed/ / core/ |
| FunctionDescriptor | stable-additive | v1 | … | distributed/ / core/ |
| HandshakePayload | stable-additive | v1 (ADR-020) | … | distributed/ |
| HeartbeatPayload | stable-additive | v1 | … | distributed/ |
| Trace schema (trace-file header) | stable-frozen (v1) | v1 (A2-P9, ADR-011-R2) | … | profiling/ + hal/ |
| dump_state() schema | versioned-subinterface | v1 (A8-P4) | … | runtime/ + bindings/ |
| AlertRule schema | stable-additive | v1 (A8-P5) | … | runtime/ + profiling/ |
| (reserved) | — | — | … | — |

## C.3 Versioning checklist for new fields
<8-step checklist per A2-P5 cross-reference: class the interface, bump plan, enumerate consumers, unknown-tag tolerance, deprecation procedure, header-independence lint (A7-P4/A8-P11), handshake compat, appendix-C row update>

## C.4 Unknown-tag tolerance
<A2-P2 closed-enum rule (DepMode, AdmissionStatus, MessageType closed; unknown values rejected with ProtocolVersionMismatch / InvalidArgument); open extension_map rule for stable-additive interfaces (ignore+forward); bounded parsing via A6-P3>

## C.5 Signed schema policy
<A6-P12 reframe: v1 signed envelope + frozen schema; REPLAY engine deferred to v2 per A9-P6 option (iii) and Q17; signing keys externally provisioned per A6-P14; rotate by restart in v1>

## C.6 Change log
| Date | Change | Driven by |
| 2026-04-18 | Initial appendix … | A2-P5, A2-P1, A4-P6 |
```

## Rationale

- **A2-P5** calls out a new `appendix-c-compatibility.md` as the central BC policy index; without it, the per-interface stability classes asked for by §9 of multiple `modules/*.md` files have nowhere to land. The new appendix satisfies §4's Applied Changes Index row for this doc (driven by `A2-P1, A2-P5`).
- **A2-P1 R3 narrowing** is applied up-front in §C.1 so readers understand that only multi-node wire messages and persistent artifacts carry the `uint16_t schema_version` byte; non-wire in-process types rely on the A2-P5 checklist. This keeps the blanket E6 obligation bounded and consistent with the amended A2-P1 text.
- **A4-P6** is reflected in the cross-link paragraph up front and by citing ADRs inline in the per-interface "First-added-in" column. Views `03–06` will link back to this appendix per their "Related ADRs" paragraphs.
- The per-interface rows cover exactly the set requested in the task (MessageHeader, TaskDescriptor, FunctionDescriptor, HandshakePayload, HeartbeatPayload, trace schema (A2-P9), `dump_state()` schema (A8-P4), `AlertRule` (A8-P5), plus a "(reserved)" stub for future entries).
- **§C.4** encodes the A2-P2 closed-for-v1 / open-extension_map policy: closed enums reject unknown values; stable-additive interfaces carry through an `extension_map` per the unknown-tag tolerance rule, and proxies must preserve-and-forward. This aligns with ADR-017 (closed-enum-in-hot-path policy).
- **§C.5** captures the A6-P12 R3 reframe: v1 ships a signed envelope and a frozen schema, but **no** REPLAY engine — matching A9-P6 option (iii) and Q17's deferral trigger.
- **§C.6 change log** is a minimal append-log so future edits to this appendix can land with one-line rows rather than free-form prose drift.

## Verification steps

1. Grep confirms the new path `docs/pypto-runtime-design/appendix-c-compatibility.md` matches the path listed in §4 Applied Changes Index of `reviews/2026-04-18-171357/final/final-proposal.md` and in §1 A2-P1 / A2-P5 "Planned diff file" entries.
2. All required sections are present in the exact shape specified by the task: "Versioning scope", "Per-interface BC table", "Versioning checklist for new fields", "Unknown-tag tolerance", "Signed schema policy".
3. The BC table's columns match the spec: `Interface | Versioning style | First-added-in | BC guarantees | Owner`.
4. All named interfaces are present as rows: `MessageHeader`, `TaskDescriptor`, `FunctionDescriptor`, `HandshakePayload`, `HeartbeatPayload`, trace schema, `dump_state()`, `AlertRule`, plus the `(reserved)` stub.
5. Signed-schema §C.5 explicitly reflects A6-P12 R3 reframe wording — "signed envelope + frozen schema v1; REPLAY engine v2".
6. §C.4 explicitly references closed `DepMode`/`AdmissionStatus`/`MessageType` (ADR-017) and the open-extension_map preserve-and-forward rule (A2-P2).
7. Cross-appendix refs resolve: `appendix-a-glossary.md` (for the new v1 canonical terms) is reachable from the intro paragraph; `modules/*.md` links resolve once their own diffs land.
