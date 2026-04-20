You are reviewer A5 (Reliability & Fault Tolerance), round 3 — the MANDATORY STRESS ROUND.

## Context

107 of 110 proposals were agreed in round 2; 3 remain disputed (A2-P7, A9-P2, A9-P6). Your job this round is to attack the emerging consensus from your aspect's strongest position and to replay at least one scenario from `06-scenario-view.md` against each accepted proposal.

## Required reading

1. Your rubric: `/data/linjiashu/Code/pto-dev-folder/.cursor/agent/architecture-review-enhancement/aspects.md` → section "A5. Reliability & Fault Tolerance".
2. Hard rules: `/data/linjiashu/Code/pto-dev-folder/docs/architecture-design-guide/04-agent-rules.md`.
3. Principles: `/data/linjiashu/Code/pto-dev-folder/docs/architecture-design-guide/01-design-principles.md`.
4. Round-2 synthesis: `/data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/reviews/2026-04-18-171357/round-2/synthesis.md`
5. Round-2 debate log: `/data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/reviews/2026-04-18-171357/round-2/debate-log.md`
6. Round-2 tally (classification): `/data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/reviews/2026-04-18-171357/round-2/tally.md`
7. Your own round-1 and round-2 reviews: `/data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/reviews/2026-04-18-171357/round-1/reviews/A5-reliability.md`, `/data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/reviews/2026-04-18-171357/round-2/reviews/A5-reliability.md`.
8. ALL peer reviews from round 2:
- `/data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/reviews/2026-04-18-171357/round-2/reviews/A1-performance.md`
- `/data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/reviews/2026-04-18-171357/round-2/reviews/A2-extensibility.md`
- `/data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/reviews/2026-04-18-171357/round-2/reviews/A3-functional-sanity.md`
- `/data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/reviews/2026-04-18-171357/round-2/reviews/A4-doc-consistency.md`
- `/data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/reviews/2026-04-18-171357/round-2/reviews/A6-security.md`
- `/data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/reviews/2026-04-18-171357/round-2/reviews/A7-modularity.md`
- `/data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/reviews/2026-04-18-171357/round-2/reviews/A8-testability.md`
- `/data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/reviews/2026-04-18-171357/round-2/reviews/A9-simplicity.md`
- `/data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/reviews/2026-04-18-171357/round-2/reviews/A10-scalability.md`
9. Scenarios for replay (you MUST attach at least one to each stress-attack): focus on 6.2.1 (AICore Hang), 6.2.2 (Remote Node Crash), 6.2.4 (Network Timeout) — one per retry/watchdog/circuit-breaker proposal. Scenario doc: `/data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/06-scenario-view.md`.
10. Target design docs (re-skim as needed):
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

- Performance (A1) carries double weight and a hot-path veto.
- All proposals, including any new ones you raise, must carry a `hot_path_impact` label.
- If you are A1, this is your last chance to veto a hot-path-touching proposal. Revoke blanket `agree` if the amended text still injects hot-path cost.

## Task — STRESS ATTACK

1. **Stress-attack every tentatively agreed proposal** that touches your aspect. For each proposal with status `agreed` in the round-2 synthesis:
   - Attempt the strongest attack your aspect can mount against the AMENDED text (round-2 §5 revisions).
   - Replay the relevant scenario from `06-scenario-view.md` step-by-step under the amended proposal. Does any step violate your rubric? Does the two-tier bridge actually hold end-to-end?
   - Verdict: `holds` | `breaks` | `uncertain`. If `breaks` or `uncertain`, propose a concrete amendment for the owner.
   - You do not need to attack every single proposal — focus on the ones where your aspect has standing. Use the round-1 + round-2 "cross-aspect tensions" sections to prioritize.
2. **Revise your own proposals** one more time if the stress attack of a peer breaks their proposal (propagate the change to yours as needed).
3. **Vote one more time** on the three disputed proposals (A2-P7, A9-P2, A9-P6). Use the amended proposal text from round-2 §5, and consider the synthesis's recommended landing zones:
   - A2-P7: recommendation is to reframe as Q-record in `09-open-questions.md` (no interface in v1)
   - A9-P2: recommendation is to keep single `IEventLoopDriver` test seam + closed enums + appendix of v2 extensions
   - A9-P6: synthesis recommends option (iii) — ship FUNCTIONAL-only implementation but keep REPLAY enum + scaffolding so A6-P12 and A8 can land day 2
4. **Record your scenario replay** in §8 (stress-attack table). Every row must cite the scenario line (e.g. `06-scenario-view.md:35-58`).

## Convergence notes

- The parent uses: **converged** iff (all proposals classified as agreed or rejected) AND (round ≥ 3) AND (stress round passed with no new disputes introduced or at most a handful of minor amendments).
- A1 hot-path veto (Runtime Mode only): if you are A1 and the stress attack reveals a proposal that DOES introduce hot-path cost, set `blocking=true` on it and the parent will route back to the owner in round 4.
- Override: if you believe a prior blocking objection was in error, mark `override_request=true` with rationale.

## Output contract

- Write your report to EXACTLY this path: `/data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/reviews/2026-04-18-171357/round-3/reviews/A5-reliability.md`
- Use the `review-round.md` template; fill §8 (Stress-Attack) thoroughly — this is the primary output of round 3.
- Do not modify any file other than the output path.
- Return a one-line confirmation: `WROTE /data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/reviews/2026-04-18-171357/round-3/reviews/A5-reliability.md`.
