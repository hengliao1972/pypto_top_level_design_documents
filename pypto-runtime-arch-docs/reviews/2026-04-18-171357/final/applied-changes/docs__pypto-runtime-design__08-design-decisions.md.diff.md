# Edit: docs/pypto-runtime-design/08-design-decisions.md

- **Driven by proposal(s):** A9-P6, A6-P12 (ADR-011-R2); A7-P6 (ADR-014); A7-P4, A2-P6 (ADR-015); A3-P1, A4-R3-P2 (ADR-016); A2-P3, A9-P4 (ADR-017); A5-P11 (ADR-018); A10-P1, A10-P7 (ADR-019); A6-P2 (ADR-020)
- **ADR link (if any):** ADR-011-R2, ADR-014, ADR-015, ADR-016, ADR-017, ADR-018, ADR-019, ADR-020
- **Timestamp (UTC):** 2026-04-19
- **Edit kind:** additive (one in-place amendment to ADR-011, seven new top-level ADRs appended)

## Location

- Path: `docs/pypto-runtime-design/08-design-decisions.md`
- Section / heading:
  - ADR-011 Related sub-section — insert new "Revision R2 (2026-04-18)" subsection before the trailing `---`.
  - End of file — append ADR-014 through ADR-020.
- Line range before edit:
  - ADR-011 anchor: original lines 497–502 (Related bullets → `---` → ADR-012 header).
  - End-of-file anchor: original line 628 (last bullet of ADR-013 Related).

## Before

Before (ADR-011 trailing block):

```496:500:docs/pypto-runtime-design/08-design-decisions.md
- [02-logical-view §2.8.1](02-logical-view/10-platform.md#281-simulation-modes) — Simulation Mode definition
- [07-cross-cutting-concerns.md §7.2](07-cross-cutting-concerns.md#72-observability) — trace event schema shared with simulation and replay

---

## ADR-012: Submission Model with Dependency Modes, Outstanding Window, and Group Workspace
```

Before (end of file for ADR-014..ADR-020 append):

```627:628:docs/pypto-runtime-design/08-design-decisions.md
- [`tensor-dependency.md`](../../tensor-dependency.md) — frontend dep-analysis specification (workspace root).
- [09-open-questions.md](09-open-questions.md) — deferred: runtime WAR/WAW, sub-range overlap on `TaskArgs`, token adoption.
```

(end-of-file)

## After

After (ADR-011 trailing block — Revision R2 subsection inserted; marker `[UPDATED: A9-P6, A6-P12: new ADR added]`):

```markdown
- [07-cross-cutting-concerns.md §7.2](07-cross-cutting-concerns.md#72-observability) — trace event schema shared with simulation and replay

<!-- [UPDATED: A9-P6, A6-P12: new ADR added] -->

### Revision R2 (2026-04-18) — SimulationMode option (iii) + REPLAY deferral

**Status:** Accepted
**Date:** 2026-04-18
**Deciders:** Architecture team
**Driven by:** A9-P6 (defer PERFORMANCE/REPLAY simulation), A6-P12 (signed/schema-validated REPLAY trace)

#### Context

Review round 3 (Aspects A5, A7, A8, A9) raised that implementing all three SimulationMode variants in v1 exceeds the reviewed scope and creates risk of silent fallback ...

#### Decision (amends original Decision)

Adopt **option (iii)** — v1 ships `FUNCTIONAL` leaf-engine only; `SimulationMode` enum stays **open**; REPLAY scaffolding **not factory-registered** in v1; REPLAY trace envelope frozen; engine deferred to v2 (Q17) ...

---

## ADR-012: Submission Model with Dependency Modes, Outstanding Window, and Group Workspace
```

After (end of file — seven new ADRs appended, each with `[UPDATED: <driver-ids>: new ADR added]` marker above the ADR header):

```markdown
- [09-open-questions.md](09-open-questions.md) — deferred: runtime WAR/WAW, sub-range overlap on `TaskArgs`, token adoption.

---

<!-- [UPDATED: A7-P6: new ADR added] -->

## ADR-014: Freeze `runtime::composition` Sub-Namespace + Promotion Triggers

**Status:** Accepted / **Date:** 2026-04-18 / **Driven by:** A7-P6
... Context / Decision / Consequences / Related ...

---

<!-- [UPDATED: A7-P4, A2-P6: new ADR added] -->

## ADR-015: Distributed Header Registration Policy (Invariant I-DIST-1)
... I-DIST-1 enforces `distributed/` headers non-includable from `transport/`; IWYU-CI ...

---

<!-- [UPDATED: A3-P1, A4-R3-P2: new ADR added] -->

## ADR-016: TaskState Canonical Spelling + Transitions
... canonical `COMPLETING → ERROR` (Task) + `FAILED` (Worker); optional `CANCELLED` ...

---

<!-- [UPDATED: A2-P3, A9-P4: new ADR added] -->

## ADR-017: Closed-Enum-in-Hot-Path Policy
... default = open via string-keyed registry; closed enum exception requires benchmark-or-veto evidence; `DepMode` closed; `Kind` removed (A9-P4) ...

---

<!-- [UPDATED: A5-P11: new ADR added] -->

## ADR-018: Single Unified Peer-Health FSM
... states `{HEALTHY, SUSPECT, OPEN, HALF_OPEN, UNAVAILABLE(Quarantine), LOST, AUTH_REVOKED}`; co-owned by breaker / heartbeat / auth ...

---

<!-- [UPDATED: A10-P1, A10-P7: new ADR added] -->

## ADR-019: Admission Shard Default + Deployment Cue
... default `shards=1`; opt-in when `concurrent_submitters × cluster_nodes ≥ 64`; global outstanding-submission atomic preserved ...

---

<!-- [UPDATED: A6-P2: new ADR added] -->

## ADR-020: Coordinator Generation + Stale-Coordinator Reject
... add `coordinator_generation: uint64_t` to HandshakePayload and ClusterView; `StaleCoordinatorClaim` reject rule; `verified: bool` in ClusterView ...
```

(Summarized above — each appended ADR in the actual file contains full Context, Decision, Consequences, and Related sections per the existing style.)

## Rationale

This edit lands seven new ADRs and one ADR amendment that were agreed during the R3 review (final-proposal.md §5). Each ADR is traceable back to its driver proposals in the agreed set of 116:

- **ADR-011-R2** narrows ADR-011 to `FUNCTIONAL`-only in v1 and locks the REPLAY envelope format, preventing silent-fallback risk (A9-P6) while preserving the A6-P12 signing/schema obligation via a frozen envelope.
- **ADR-014** freezes `runtime::composition` as a sub-namespace with explicit promotion triggers (A7-P6), avoiding premature module split.
- **ADR-015** codifies Invariant I-DIST-1 (A7-P4 + A2-P6), cleanly separating transport framing from distributed payloads via IWYU-CI.
- **ADR-016** canonicalizes TaskState spelling per A3-P1 + A4-R3-P2, eliminating scenario-view drift.
- **ADR-017** resolves the "open vs closed enum" debate raised by A2-P3 / A9-P4 with a benchmark-or-veto evidence rule.
- **ADR-018** unifies peer-health state across breaker / heartbeat / auth (A5-P11), fixing the R3 oscillation regression.
- **ADR-019** defaults admission-sharding to 1 and documents the 64-peer opt-in cue (A10-P1 + A10-P7).
- **ADR-020** adds `coordinator_generation` to defeat demoted-coordinator replay (A6-P2).

All edits are additive; ADR-011-R2 is a subsection of the existing ADR-011 and preserves the prior Decision verbatim.

## Verification steps

1. `grep -n "^## ADR-" docs/pypto-runtime-design/08-design-decisions.md` shows 20 ADR headers (ADR-001 .. ADR-020) in order.
2. `grep -n "^### Revision R2" docs/pypto-runtime-design/08-design-decisions.md` shows exactly one match inside ADR-011.
3. `grep -n "\[UPDATED:" docs/pypto-runtime-design/08-design-decisions.md` lists eight `[UPDATED: ...]` markers (one per new ADR / amendment) and no others.
4. Cross-doc consistency: ADR-011-R2 and ADR-020 are referenced by Q17 and ADR-018 respectively in `09-open-questions.md` and ADR-018's Related list.
5. Re-render `docs/pypto-runtime-design/` with a TOC generator; confirm ADR section count = 20 and all anchors are reachable.
