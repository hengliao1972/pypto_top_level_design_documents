# Architecture Review Run — `pypto-runtime-design`

Chronological, human-readable index of every artifact produced during this review run. Follow the links in order to replay exactly what happened.

## Run Metadata

- **Run ID:** 2026-04-18-171357
- **Target:** `docs/pypto-runtime-design/`
- **Runtime Mode:** yes (A1 ×2; hot-path veto; mandatory stress round; scenario replay)
- **Reviewer Roster:** A1 (Performance), A2 (Extensibility), A3 (Functional-sanity), A4 (Doc-consistency), A5 (Reliability), A6 (Security), A7 (Modularity), A8 (Testability), A9 (Simplicity), A10 (Scalability)
- **Rounds Run:** 3
- **Convergence Status:** converged at Round 3
- **Manifest:** [run-manifest.md](run-manifest.md)
- **Final Proposal:** [final/final-proposal.md](final/final-proposal.md)
- **Residual Risks:** [final/residual-risks.md](final/residual-risks.md)

## Summary Counts

| Metric | Value |
|---|---|
| Proposals catalogued (R1) | 110 |
| Proposals merged / withdrawn during R1 → R2 | 3 |
| Proposals agreed (R2) | 107 |
| Proposals disputed (R2) | 3 |
| Proposals newly introduced (R3) | 6 |
| **Total agreed proposals** | **116** |
| Rejected proposals | 0 |
| Deferred proposals | 0 |
| Applied edits (target files modified) | 29 |
| New ADRs | 8 (ADR-011-R2 amendment + ADR-014..ADR-020) |
| New open questions | 4 (Q15-Q18) |
| New known-deviations | 1 |
| New compatibility appendix | 1 (`appendix-c-compatibility.md`) |
| Runtime-mode A1 vetos applied | 0 |
| Non-A1 overrides succeeded | 0 |
| Blocking objections filed | 0 |

## Chronological Walkthrough

### Phase 0 — Setup

- [run-manifest.md](run-manifest.md)

### Round 1 — Independent Reviews

- Prompts:
  - [round-1/prompts/A1-performance.prompt.md](round-1/prompts/A1-performance.prompt.md)
  - [round-1/prompts/A2-extensibility.prompt.md](round-1/prompts/A2-extensibility.prompt.md)
  - [round-1/prompts/A3-functional-sanity.prompt.md](round-1/prompts/A3-functional-sanity.prompt.md)
  - [round-1/prompts/A4-doc-consistency.prompt.md](round-1/prompts/A4-doc-consistency.prompt.md)
  - [round-1/prompts/A5-reliability.prompt.md](round-1/prompts/A5-reliability.prompt.md)
  - [round-1/prompts/A6-security.prompt.md](round-1/prompts/A6-security.prompt.md)
  - [round-1/prompts/A7-modularity.prompt.md](round-1/prompts/A7-modularity.prompt.md)
  - [round-1/prompts/A8-testability.prompt.md](round-1/prompts/A8-testability.prompt.md)
  - [round-1/prompts/A9-simplicity.prompt.md](round-1/prompts/A9-simplicity.prompt.md)
  - [round-1/prompts/A10-scalability.prompt.md](round-1/prompts/A10-scalability.prompt.md)
- Reviews:
  - [round-1/reviews/A1-performance.md](round-1/reviews/A1-performance.md)
  - [round-1/reviews/A2-extensibility.md](round-1/reviews/A2-extensibility.md)
  - [round-1/reviews/A3-functional-sanity.md](round-1/reviews/A3-functional-sanity.md)
  - [round-1/reviews/A4-doc-consistency.md](round-1/reviews/A4-doc-consistency.md)
  - [round-1/reviews/A5-reliability.md](round-1/reviews/A5-reliability.md)
  - [round-1/reviews/A6-security.md](round-1/reviews/A6-security.md)
  - [round-1/reviews/A7-modularity.md](round-1/reviews/A7-modularity.md)
  - [round-1/reviews/A8-testability.md](round-1/reviews/A8-testability.md)
  - [round-1/reviews/A9-simplicity.md](round-1/reviews/A9-simplicity.md)
  - [round-1/reviews/A10-scalability.md](round-1/reviews/A10-scalability.md)
- Synthesis:
  - [round-1/synthesis.md](round-1/synthesis.md)

### Round 2 — Cross-Critique & Vote

- Prompts (one per reviewer; see `round-2/prompts/A<k>-<slug>.prompt.md`):
  - [round-2/prompts/A1-performance.prompt.md](round-2/prompts/A1-performance.prompt.md)
  - [round-2/prompts/A2-extensibility.prompt.md](round-2/prompts/A2-extensibility.prompt.md)
  - [round-2/prompts/A3-functional-sanity.prompt.md](round-2/prompts/A3-functional-sanity.prompt.md)
  - [round-2/prompts/A4-doc-consistency.prompt.md](round-2/prompts/A4-doc-consistency.prompt.md)
  - [round-2/prompts/A5-reliability.prompt.md](round-2/prompts/A5-reliability.prompt.md)
  - [round-2/prompts/A6-security.prompt.md](round-2/prompts/A6-security.prompt.md)
  - [round-2/prompts/A7-modularity.prompt.md](round-2/prompts/A7-modularity.prompt.md)
  - [round-2/prompts/A8-testability.prompt.md](round-2/prompts/A8-testability.prompt.md)
  - [round-2/prompts/A9-simplicity.prompt.md](round-2/prompts/A9-simplicity.prompt.md)
  - [round-2/prompts/A10-scalability.prompt.md](round-2/prompts/A10-scalability.prompt.md)
- Reviews:
  - [round-2/reviews/A1-performance.md](round-2/reviews/A1-performance.md)
  - [round-2/reviews/A2-extensibility.md](round-2/reviews/A2-extensibility.md)
  - [round-2/reviews/A3-functional-sanity.md](round-2/reviews/A3-functional-sanity.md)
  - [round-2/reviews/A4-doc-consistency.md](round-2/reviews/A4-doc-consistency.md)
  - [round-2/reviews/A5-reliability.md](round-2/reviews/A5-reliability.md)
  - [round-2/reviews/A6-security.md](round-2/reviews/A6-security.md)
  - [round-2/reviews/A7-modularity.md](round-2/reviews/A7-modularity.md)
  - [round-2/reviews/A8-testability.md](round-2/reviews/A8-testability.md)
  - [round-2/reviews/A9-simplicity.md](round-2/reviews/A9-simplicity.md)
  - [round-2/reviews/A10-scalability.md](round-2/reviews/A10-scalability.md)
- Extract / Tally:
  - [round-2/extract.md](round-2/extract.md)
  - [round-2/tally.md](round-2/tally.md)
- [round-2/debate-log.md](round-2/debate-log.md)
- [round-2/synthesis.md](round-2/synthesis.md)

### Round 3 — Stress Round (mandatory)

- Prompts:
  - [round-3/prompts/A1-performance.prompt.md](round-3/prompts/A1-performance.prompt.md)
  - [round-3/prompts/A2-extensibility.prompt.md](round-3/prompts/A2-extensibility.prompt.md)
  - [round-3/prompts/A3-functional-sanity.prompt.md](round-3/prompts/A3-functional-sanity.prompt.md)
  - [round-3/prompts/A4-doc-consistency.prompt.md](round-3/prompts/A4-doc-consistency.prompt.md)
  - [round-3/prompts/A5-reliability.prompt.md](round-3/prompts/A5-reliability.prompt.md)
  - [round-3/prompts/A6-security.prompt.md](round-3/prompts/A6-security.prompt.md)
  - [round-3/prompts/A7-modularity.prompt.md](round-3/prompts/A7-modularity.prompt.md)
  - [round-3/prompts/A8-testability.prompt.md](round-3/prompts/A8-testability.prompt.md)
  - [round-3/prompts/A9-simplicity.prompt.md](round-3/prompts/A9-simplicity.prompt.md)
  - [round-3/prompts/A10-scalability.prompt.md](round-3/prompts/A10-scalability.prompt.md)
- Reviews:
  - [round-3/reviews/A1-performance.md](round-3/reviews/A1-performance.md)
  - [round-3/reviews/A2-extensibility.md](round-3/reviews/A2-extensibility.md)
  - [round-3/reviews/A3-functional-sanity.md](round-3/reviews/A3-functional-sanity.md)
  - [round-3/reviews/A4-doc-consistency.md](round-3/reviews/A4-doc-consistency.md)
  - [round-3/reviews/A5-reliability.md](round-3/reviews/A5-reliability.md)
  - [round-3/reviews/A6-security.md](round-3/reviews/A6-security.md)
  - [round-3/reviews/A7-modularity.md](round-3/reviews/A7-modularity.md)
  - [round-3/reviews/A8-testability.md](round-3/reviews/A8-testability.md)
  - [round-3/reviews/A9-simplicity.md](round-3/reviews/A9-simplicity.md)
  - [round-3/reviews/A10-scalability.md](round-3/reviews/A10-scalability.md)
- Extract / Tally / Logs:
  - [round-3/extract.md](round-3/extract.md)
  - [round-3/tally.md](round-3/tally.md)
  - [round-3/debate-log.md](round-3/debate-log.md)
  - [round-3/synthesis.md](round-3/synthesis.md)

### Phase 5 — Auto-Apply

- [final/final-proposal.md](final/final-proposal.md)
- Applied change diffs (one per modified target doc):
  - [final/applied-changes/docs__pypto-runtime-design__02-logical-view__02-scheduler.md.diff.md](final/applied-changes/docs__pypto-runtime-design__02-logical-view__02-scheduler.md.diff.md)
  - [final/applied-changes/docs__pypto-runtime-design__02-logical-view__03-worker.md.diff.md](final/applied-changes/docs__pypto-runtime-design__02-logical-view__03-worker.md.diff.md)
  - [final/applied-changes/docs__pypto-runtime-design__02-logical-view__04-memory.md.diff.md](final/applied-changes/docs__pypto-runtime-design__02-logical-view__04-memory.md.diff.md)
  - [final/applied-changes/docs__pypto-runtime-design__02-logical-view__05-machine-level-registry.md.diff.md](final/applied-changes/docs__pypto-runtime-design__02-logical-view__05-machine-level-registry.md.diff.md)
  - [final/applied-changes/docs__pypto-runtime-design__02-logical-view__06-function-types.md.diff.md](final/applied-changes/docs__pypto-runtime-design__02-logical-view__06-function-types.md.diff.md)
  - [final/applied-changes/docs__pypto-runtime-design__02-logical-view__07-task-model.md.diff.md](final/applied-changes/docs__pypto-runtime-design__02-logical-view__07-task-model.md.diff.md)
  - [final/applied-changes/docs__pypto-runtime-design__02-logical-view__09-interfaces.md.diff.md](final/applied-changes/docs__pypto-runtime-design__02-logical-view__09-interfaces.md.diff.md)
  - [final/applied-changes/docs__pypto-runtime-design__02-logical-view__10-platform.md.diff.md](final/applied-changes/docs__pypto-runtime-design__02-logical-view__10-platform.md.diff.md)
  - [final/applied-changes/docs__pypto-runtime-design__02-logical-view__12-dependency-model.md.diff.md](final/applied-changes/docs__pypto-runtime-design__02-logical-view__12-dependency-model.md.diff.md)
  - [final/applied-changes/docs__pypto-runtime-design__03-development-view.md.diff.md](final/applied-changes/docs__pypto-runtime-design__03-development-view.md.diff.md)
  - [final/applied-changes/docs__pypto-runtime-design__04-process-view.md.diff.md](final/applied-changes/docs__pypto-runtime-design__04-process-view.md.diff.md)
  - [final/applied-changes/docs__pypto-runtime-design__05-physical-view.md.diff.md](final/applied-changes/docs__pypto-runtime-design__05-physical-view.md.diff.md)
  - [final/applied-changes/docs__pypto-runtime-design__06-scenario-view.md.diff.md](final/applied-changes/docs__pypto-runtime-design__06-scenario-view.md.diff.md)
  - [final/applied-changes/docs__pypto-runtime-design__07-cross-cutting-concerns.md.diff.md](final/applied-changes/docs__pypto-runtime-design__07-cross-cutting-concerns.md.diff.md)
  - [final/applied-changes/docs__pypto-runtime-design__08-design-decisions.md.diff.md](final/applied-changes/docs__pypto-runtime-design__08-design-decisions.md.diff.md)
  - [final/applied-changes/docs__pypto-runtime-design__09-open-questions.md.diff.md](final/applied-changes/docs__pypto-runtime-design__09-open-questions.md.diff.md)
  - [final/applied-changes/docs__pypto-runtime-design__10-known-deviations.md.diff.md](final/applied-changes/docs__pypto-runtime-design__10-known-deviations.md.diff.md)
  - [final/applied-changes/docs__pypto-runtime-design__appendix-a-glossary.md.diff.md](final/applied-changes/docs__pypto-runtime-design__appendix-a-glossary.md.diff.md)
  - [final/applied-changes/docs__pypto-runtime-design__appendix-b-codebase-mapping.md.diff.md](final/applied-changes/docs__pypto-runtime-design__appendix-b-codebase-mapping.md.diff.md)
  - [final/applied-changes/docs__pypto-runtime-design__appendix-c-compatibility.md.diff.md](final/applied-changes/docs__pypto-runtime-design__appendix-c-compatibility.md.diff.md)
  - [final/applied-changes/docs__pypto-runtime-design__modules__bindings.md.diff.md](final/applied-changes/docs__pypto-runtime-design__modules__bindings.md.diff.md)
  - [final/applied-changes/docs__pypto-runtime-design__modules__core.md.diff.md](final/applied-changes/docs__pypto-runtime-design__modules__core.md.diff.md)
  - [final/applied-changes/docs__pypto-runtime-design__modules__distributed.md.diff.md](final/applied-changes/docs__pypto-runtime-design__modules__distributed.md.diff.md)
  - [final/applied-changes/docs__pypto-runtime-design__modules__hal.md.diff.md](final/applied-changes/docs__pypto-runtime-design__modules__hal.md.diff.md)
  - [final/applied-changes/docs__pypto-runtime-design__modules__memory.md.diff.md](final/applied-changes/docs__pypto-runtime-design__modules__memory.md.diff.md)
  - [final/applied-changes/docs__pypto-runtime-design__modules__profiling.md.diff.md](final/applied-changes/docs__pypto-runtime-design__modules__profiling.md.diff.md)
  - [final/applied-changes/docs__pypto-runtime-design__modules__runtime.md.diff.md](final/applied-changes/docs__pypto-runtime-design__modules__runtime.md.diff.md)
  - [final/applied-changes/docs__pypto-runtime-design__modules__scheduler.md.diff.md](final/applied-changes/docs__pypto-runtime-design__modules__scheduler.md.diff.md)
  - [final/applied-changes/docs__pypto-runtime-design__modules__transport.md.diff.md](final/applied-changes/docs__pypto-runtime-design__modules__transport.md.diff.md)
- [final/residual-risks.md](final/residual-risks.md)

## How to read this run

1. Start with `run-manifest.md` — it tells you scope, roster, and the timeline.
2. Read each round's `synthesis.md` for a top-down view of what each round concluded.
3. Dive into individual `reviews/A<k>-*.md` to see each aspect reviewer's raw output.
4. Use `debate-log.md` (rounds ≥ 2) to see every vote and the arithmetic behind each decision.
5. Read `final/final-proposal.md` to see the final agreed / rejected / deferred list.
6. Open the individual `applied-changes/*.diff.md` to see each concrete edit and its rationale.
7. Cross-check against `final/residual-risks.md` for anything the review deliberately left open.

## Proposal → Edit trace

See `final/final-proposal.md` §4 "Applied Changes Index" for the authoritative mapping of every proposal id to the file(s) it modifies and the diff file capturing each edit.
