# Round 3 (Stress Round) — Synthesis

- Run: `2026-04-18-171357`
- Target: `docs/pypto-runtime-design/`
- Mode: Runtime Design Mode (A1 ×2; hot-path veto enabled; mandatory stress round)
- Status at end of Round 3: **CONVERGED**

## 1. Inputs consumed

- 10 Round 3 reviews under `round-3/reviews/A*.md`
- `round-3/extract.md` (consolidated extraction of stress attacks, scenario replays, disputed-proposal votes, new proposals, and blocking-objection scan)
- `round-3/tally.md` (weighted vote outcomes)
- `round-3/debate-log.md` (vote matrix, resolution ledger, veto/override registers, convergence calc)

## 2. Key outcomes

### 2.1 All three R2 disputes resolved → agreed

| Proposal | Final landing zone | Weighted agree |
|---|---|---|
| A2-P7 | Q-record only in `09-open-questions.md`; no v1 interface; third axis = "transport-capability semantics" | 11 / 11 |
| A9-P2 | Single `IEventLoopDriver` test-only seam + closed `DeploymentMode`/`PolicyKind` enums + appendix of future-extension interfaces in `08-design-decisions.md` | 11 / 11 |
| A9-P6 (option iii) | v1 FUNCTIONAL-only; `SimulationMode` open enum; REPLAY scaffolding declared but not factory-registered; ADR-011-R2 names triggers; A6-P12 rescoped to "signed schema + frozen format v1; REPLAY engine v2" | 11 / 11 |

No A1 veto; no blocking; all R2-carry-forward override requests (A2's) are withdrawn.

### 2.2 Zero late blocking objections; zero new A1 vetoes

Every one of the 107 R2-agreed proposals survived stress attacks intact under scenario replay. 25 minor amendments were produced (all HPI=none) and are listed in `tally.md §3`.

### 2.3 Six new R3 proposals authored

All six have HPI=none and reached consensus within their aspect owners with no cross-aspect objection:

1. `A4-R3-P1` — Add `WorkerState` row to `appendix-b-codebase-mapping.md` + `appendix-a-glossary.md`.
2. `A4-R3-P2` — Unify Task terminal-failed spelling across `06-scenario-view.md:91,93,95,108,111`.
3. `A5-P11` — Unified peer-health FSM in `modules/distributed.md §3.5` (new).
4. `A5-P12` — Name `ErrorCode::ResourceExhausted` for `BufferRef` staging alloc failure.
5. `A5-P13` — Record `breaker_auth_fail_weight` (default 10) config field.
6. `A5-P14` — Q-record in `09-open-questions.md` for collective partial-failure FailurePolicy mapping.

### 2.4 Scenario replay summary

| Scenario | Replayed by | Net effect |
|---|---|---|
| 6.1.1 | A1, A2, A4, A7, A9 | Holds |
| 6.1.2 | A2, A4, A6, A7, A10 | Third-axis amendment to A2-P7; coordinator_generation amendment to A6-P2 |
| 6.1.3 | A1, A3, A8 | SPMD precondition and aggregation amendments (A3-P4/P5/P7) |
| 6.2.1 | A1, A4, A5, A8 | Holds |
| 6.2.2 | A5, A6, A10 | A5-P3 `coordinator_liveness_timeout_ms`; A5-P11 FSM; A10-P2a scope pin |
| 6.2.3 | A3, A5, A9 | A3-P7/P10 + A9-P5 scenario-text amendments |
| 6.2.4 | A4, A5 | Degradation rule externalization confirmed |

## 3. Convergence declaration

| Check | Result |
|---|---|
| Every proposal weighted-agree ≥ 2/3 | ✓ (100 %) |
| Open blocking objections | 0 |
| A1 hot-path vetoes applied | 0 |
| Stress round held | ✓ (this round) |
| Scenario replays recorded | ✓ (per §2.4 + debate-log §3) |

**Decision:** `CONVERGED` at Round 3. **No Round 4 required.** Proceed to Phase 5 (final proposal + apply edits).

## 4. Final-proposal inputs to Phase 5

Phase 5 must produce:

- `final/final-proposal.md` — canonical proposal list with stable IDs, amended text, owners, HPI, target doc + anchor, cross-references, and the three landing-zone refinements from §2.1.
- `final/applied-changes/` — per-edit diff files describing exactly what was written.
- `[UPDATED: R=3, ISO8601, by=run-id]` markers immediately adjacent to every changed block in the target docs.
- Append new ADRs to `08-design-decisions.md`:
  - ADR-011-R2 (amended) — `SimulationMode` option (iii) text, triggers for REPLAY engine.
  - ADR-A7-R3 — `runtime::composition` sub-namespace freeze + promotion triggers.
  - ADR — distributed header registration policy (Invariant I-DIST-1).
- Append new questions to `09-open-questions.md`:
  - Q — Transport-capability semantics axis for async-policy extension (A2-P7).
  - Q — Collective partial-failure FailurePolicy mapping (A5-P14).
- Update `appendix-a-glossary.md` and `appendix-b-codebase-mapping.md` per A4-R3-P1.
- Update `modules/*.md` per the 25 amendments in `tally.md §3` and the 6 new proposals.

## 5. Residual risks to record in `residual-risks.md`

- Conditional re-vote triggers on A9-P6 (A5, A7, A8) if Phase 5 final text drifts from option (iii).
- Conditional re-vote triggers on A2-P7 (A3, A5, A9) if Phase 5 final text reintroduces any v1 interface declaration.
- A5's 6.2.2 livelock analysis depends on A5-P11 FSM landing as a co-owned doc section; if ownership slips, re-open CB ↔ heartbeat ↔ auth interaction.
- A6-P12's "schema frozen v1" decision defers REPLAY forensic capture to v2; record triggers in ADR-011-R2.
- A10-P6 heartbeat sharding cap=4 assumes ≤128 peers per node; revisit if topology grows.

## 6. No Round 4

Per convergence rules and runtime-mode supplement, this synthesis closes debate. Phase 5 proceeds immediately.
