# Edit: docs/pypto-runtime-design/09-open-questions.md

- **Driven by proposal(s):** A2-P7 (Q15); A5-P14 (Q16); A9-P6, A6-P12 (Q17); A10-P6 (Q18)
- **ADR link (if any):** —
- **Timestamp (UTC):** 2026-04-19
- **Edit kind:** additive (four new Open Questions appended at end-of-file)

## Location

- Path: `docs/pypto-runtime-design/09-open-questions.md`
- Section / heading: end of file — appended after the existing Q9 "Stages 1b–8 Design Document Status" entry.
- Line range before edit: 199–205 (end-of-file anchor).

## Before

```199:205:docs/pypto-runtime-design/09-open-questions.md
**Suggested priority (from master plan):**
1. **Stage 1b (DSL Design)** — needed to validate the abstract machine model against the user-facing programming model.
2. **Stage 2 (Task Model & State Machine)** — needed to finalize the state machine and dependency model before implementation.
3. **Stage 3 (HAL)** — needed to define the platform abstraction boundary.
4. **Stage 4 (Module Decomposition)** — needed to finalize module interfaces before coding begins.
5. **Stages 5–8** — can proceed in parallel once the core model and module boundaries are approved.
```

(end-of-file)

## After

Four new Questions (Q15 through Q18) appended at the end, each preceded by an `[UPDATED: <proposal-id>: new Q added]` marker. Outline shape (full content lives in the source file):

```markdown
5. **Stages 5–8** — can proceed in parallel once the core model and module boundaries are approved.

---

<!-- [UPDATED: A2-P7: new Q added] -->

## Q15: Async-Policy Extension Seam with Transport-Capability Axis

**Context:** Q11 already asks whether policy hooks should support async invocation. R3 widened the question along transport-capability semantics (RDMA rkey / TCP-TLS binding) and async-submit return path. A2-P7 elected to reserve the seam as a Q-record only; no v1 interface shipped.

**Question:** When and how should the async-policy seam be opened, and across which of the three axes?

**Options:** (A) Defer all three axes (current); (B) Open only transport-capability axis in v1; (C) Open all three with bounded-timeout async hook + versioned return-path contract.

**Trigger for revisiting:** concrete policy proposal requiring cross-node capability queries or external-signal deferral.

**Related:** Q11, A2-P7, ADR-012, ADR-019.

---

<!-- [UPDATED: A5-P14: new Q added] -->

## Q16: Orchestration-Composed Collective Partial-Failure → FailurePolicy Mapping

**Context:** A9-P3 resolved Q4 toward composed collectives. Partial-failure cascade has no canonical `FailurePolicy` / `partial_group_policy` mapping today. A5-P14 recorded as v2 ADR trigger.

**Question:** How should composed-collective partial failure map onto §6.2.2 failure policy?

**Options:** (A) Per-peer independent; (B) `CollectiveContext` aggregator; (C) Promote back into `ICollectiveOps` (Q4 option C).

**Trigger for revisiting:** straggler-tolerant training workload; chaos run showing composed-vs-native divergence.

**Related:** A9-P3, A5-P14, Q4, §6.2.2.

---

<!-- [UPDATED: A9-P6, A6-P12: new Q added] -->

## Q17: REPLAY Engine v2 Concrete Triggers and Schema Evolution

**Context:** Per ADR-011-R2, v1 ships `FUNCTIONAL` only; REPLAY scaffolding declared but not factory-registered; trace envelope frozen; engine deferred to v2. ADR-011-R2 names Q17 as the tracking question.

**Question:** What triggers promote REPLAY engine to v2, and how does the frozen envelope evolve?

**Options (promotion):** (A) post-mortem demand metric; (B) chaos-scenario coverage metric; (C) tooling-driven (CI replay).
**Options (schema):** (α) versioned additive; (β) major-bump + re-capture; (γ) compatibility layer.

**Current recommendation:** (B) + (α).

**Related:** ADR-011-R2, A9-P6, A6-P12, A2-P9, §7.2.

---

<!-- [UPDATED: A10-P6: new Q added] -->

## Q18: Heartbeat Shard Cap Beyond 128 Peers

**Context:** A10-P6 heartbeat sharding uses cap-4 thread pool (≤ 128 peers assumption). Larger topologies need a scaling plan.

**Question:** How should the heartbeat subsystem scale past 128 peers per node?

**Options:** (A) Raise cap linearly; (B) Tree / gossip; (C) Event-driven keepalive-only.

**Trigger for revisiting:** deployment target > 128 peers or `NodeLost` latency regression on 64–128 peer clusters.

**Related:** A10-P6, ADR-018, `modules/distributed.md`.
```

## Rationale

The four new Open Questions record deferral points explicitly raised during R3 but not landed as ADRs in v1:

- **Q15** reserves the async-policy extension seam without shipping any v1 interface slot (A2-P7). Keeps the OCP (E4) exit available without adding migration cost later.
- **Q16** tracks the partial-failure mapping gap A5 surfaced on the A9-P3 orchestration-composed path — a known R4 gap that will trigger a v2 ADR rather than hacking a v1 handler.
- **Q17** is the normative follow-up for ADR-011-R2's deferred REPLAY engine; concrete triggers let the v1 envelope freeze be respected.
- **Q18** scopes the cap-4 heartbeat thread-pool assumption and documents its revisit conditions.

All four are pure appends with no changes to existing Q1..Q14.

## Verification steps

1. `grep -n "^## Q[0-9]" docs/pypto-runtime-design/09-open-questions.md` returns Q1..Q8, Q10..Q14, Q15..Q18, Q9 in file order. Numbering is complete; Q9 remains at the end as in the original (reordered by A4-P4 elsewhere).
2. `grep -c "\[UPDATED:" docs/pypto-runtime-design/09-open-questions.md` returns `4`.
3. Q17 references ADR-011-R2, which is reachable in `08-design-decisions.md` (via the Revision R2 subsection inside ADR-011). Cross-link check passes.
4. Q16 references A9-P3 and §6.2.2; both exist in the agreed proposal set.
