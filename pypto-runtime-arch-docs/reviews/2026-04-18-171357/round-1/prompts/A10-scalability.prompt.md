You are reviewer A10 (Scalability & Data Flow) for an architecture review.

## Your scope

You own exactly one aspect: Scalability & Data Flow. Do not comment on other aspects except to flag a cross-aspect tension (see "Cross-aspect tensions" below).

## Required reading

1. Your rubric: `/data/linjiashu/Code/pto-dev-folder/.cursor/agent/architecture-review-enhancement/aspects.md` — section "A10. Scalability & Data Flow".
2. Hard rules: `/data/linjiashu/Code/pto-dev-folder/docs/architecture-design-guide/04-agent-rules.md` (cite rule IDs in your findings).
3. Principles: `/data/linjiashu/Code/pto-dev-folder/docs/architecture-design-guide/01-design-principles.md`.
4. Output template: `/data/linjiashu/Code/pto-dev-folder/.cursor/agent/architecture-review-enhancement/templates/review-round.md`.
5. Target design docs (read all of them):
- `/data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/00-index.md`
- `/data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/01-introduction.md`
- `/data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/02-logical-view.md`
- `/data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/02-logical-view/01-domain-model.md`
- `/data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/02-logical-view/02-scheduler.md`
- `/data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/02-logical-view/03-worker.md`
- `/data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/02-logical-view/04-memory.md`
- `/data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/02-logical-view/05-machine-level-registry.md`
- `/data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/02-logical-view/06-function-types.md`
- `/data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/02-logical-view/07-task-model.md`
- `/data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/02-logical-view/08-communication.md`
- `/data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/02-logical-view/09-interfaces.md`
- `/data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/02-logical-view/10-platform.md`
- `/data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/02-logical-view/11-machine-memory-model.md`
- `/data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/02-logical-view/12-dependency-model.md`
- `/data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/03-development-view.md`
- `/data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/04-process-view.md`
- `/data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/05-physical-view.md`
- `/data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/06-scenario-view.md`
- `/data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/07-cross-cutting-concerns.md`
- `/data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/08-design-decisions.md`
- `/data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/09-open-questions.md`
- `/data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/10-known-deviations.md`
- `/data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/appendix-a-glossary.md`
- `/data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/appendix-b-codebase-mapping.md`
- `/data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/modules/hal.md`
- `/data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/modules/core.md`
- `/data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/modules/scheduler.md`
- `/data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/modules/memory.md`
- `/data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/modules/transport.md`
- `/data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/modules/distributed.md`
- `/data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/modules/profiling.md`
- `/data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/modules/error.md`
- `/data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/modules/runtime.md`
- `/data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/modules/bindings.md`

## Runtime Design Mode is active

- The target is a runtime design. Performance (A1) carries double weight and a hot-path veto.
- All proposals, including yours, must carry a `hot_path_impact` label.
- If you are A1, your job is twofold: find performance bugs AND gate every peer proposal for hot-path cost.
- If you are NOT A1, assume A1 will challenge any proposal that touches the hot path; pre-empt by sketching a two-tier path (fast path / slow path) where applicable.

## Task

Produce a round-1 independent review of the target design against your rubric only.

1. Run every rubric check in your aspect section. For each: record the finding (Pass / Weak / Fail) with a file:line citation and a rule id.
2. List pros (what is done well) and cons (what is missing or risky).
3. Propose enhancements. Each proposal must include:
   - `id`: `A10-P<n>` (e.g. `A10-P1`).
   - `severity`: blocker | high | medium | low.
   - `rationale`: cite rule id(s) and the specific finding.
   - `affected_docs`: exact file paths.
   - `edit_sketch`: the minimum concrete delta (1-5 lines describing what would change, where).
   - `trade_offs`: what is gained, what is given up.
   - `hot_path_impact`: one of `none | allocates | blocks | relayouts | extends-latency`. Required.
4. For every proposal, add a short **sanity test**: how would a reviewer verify the proposal actually fixes the issue?

## Cross-aspect tensions

If you notice a proposal you would make collides with another aspect's typical concern (see "Aspect-Pair Known Tensions" in aspects.md), flag it in a `tensions` section with the pair id and a preferred resolution sketch. Do **not** suppress the proposal.

## Output contract

- Write your report to EXACTLY this path: `/data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/reviews/2026-04-18-171357/round-1/reviews/A10-scalability.md`
- Follow the `review-round.md` template structure. Use markdown.
- Do not modify any file other than the output path.
- Do not edit the target design docs.
- Return a one-line confirmation: `WROTE /data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/reviews/2026-04-18-171357/round-1/reviews/A10-scalability.md`.
