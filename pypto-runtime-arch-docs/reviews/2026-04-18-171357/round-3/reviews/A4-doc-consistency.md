# Aspect A4: Document Consistency — Round 3 (Stress Round)

## Metadata

- **Reviewer:** A4
- **Round:** 3 (MANDATORY STRESS ROUND)
- **Target:** `docs/pypto-runtime-design/` (same package; no target changes)
- **Runtime Mode:** yes (A1 ×2, A1 hot-path veto; A4 = non-A1, `override_request` available)
- **Timestamp (UTC):** 2026-04-19T00:00:00Z

---

## 1. Rubric Checks

Rubric evidence is unchanged from R1/R2 — the source docs have not yet been edited. The stress round is verification against the amended R2 proposal set, not a re-scan. Carry-forward table (same rows as R2 §1):

| # | Check | R1 Result | R3 Stress-Round Verdict |
|---|-------|-----------|--------------------------|
| 1 | Same concept named identically across views, diagrams, module docs, ADRs, glossary (D7, V5) | Fail | **holds under R2 amendments** for A4-P1/P7, A7-P8/P9, A9-P5 — provided every amendment is landed in **one PR** with glossary + Appendix-B in the same commit (§8 row R3-X1). |
| 2 | Logical ↔ Development ↔ Process ↔ Physical mapping (V2, V5, D7) | Weak | **holds** once A4-P7 + A7-P2 + A10-P8 land together; scenario 6.1.1 step 5 (`06-scenario-view.md:23`) and 6.1.2 step 4 (`06-scenario-view.md:48`) are the two load-bearing mappings — both survive the amended proposals (§8 rows R3-X3, R3-X11). |
| 3 | All required views present (V1) | Pass | unchanged. |
| 4 | ADRs referenced from the views they constrain (V2, G5) | Fail | **holds** under A4-P6 + A2-P5 (appendix-c); 6.1.2 RDMA step (`06-scenario-view.md:51`) and REMOTE_SUBMIT step (`06-scenario-view.md:48`) now each hit an ADR citation via the amended `04-process-view.md` / `06-scenario-view.md` back-links (§8 row R3-X8). |
| 5 | Glossary covers every formal term (D7) | Weak | **holds conditional** — A4-P5 is split and the branches are now determined: A9-P7 agreed (drop `SourceCollectionConfig` row) and A9-P2 is still disputed (if agreed, `IEventCollectionPolicy` becomes a one-liner). See §5. |
| 6 | `appendix-b-codebase-mapping.md` matches Development View (D7) | Weak | **holds** once A4-P8 ∧ A3-P1 both land in one PR. Appendix-B must enumerate `ERROR` (+`CANCELLED` if adopted) in the `TaskState` annotation. A5-P9's `QUARANTINED` requires a parallel row for `WorkerState` — **newly surfaced stress-finding**, captured as **R3-P1** in §4. |

Two **new** consistency findings surfaced in §8 stress-replay that did not exist in R1/R2 and are captured as new proposals in §4:

- **R3-P1 (medium):** `WorkerState` has no Appendix-B row or glossary entry, yet A5-P9 agrees to add `QUARANTINED`. This is the same class of drift that A4-P8 closes for `TaskState` — we must close it symmetrically.
- **R3-P2 (medium):** Scenarios 6.1.1 / 6.1.2 reference `parent Orchestration Task transitions COMPLETING → COMPLETED` (`06-scenario-view.md:28`) and `COMPLETING → ERROR` (`06-scenario-view.md:93`) — after A3-P1 lands with `ERROR`, the state diagram in `04-process-view.md` §4.3 and the Scenario View prose need a single pass to align wording ("COMPLETED(error)" vs. "ERROR" vs. "FAILED"). Currently three spellings for one terminal state.

---

## 2. Pros

Carry-forward from R1/R2 with two stress-round reinforcements:

- **R2 §5 amendments defuse A4's conditional-glossary risk.** A9-P7 is now unanimously agreed (8 agree / 0 disagree / 1 abstain per `round-2/tally.md:121`), so A4-P5's conditional branch for `SourceCollectionConfig` resolves to "drop the row" — this removes one moving part from §5.
- **A6 / A5 / A8 new-type inventories were absorbed cleanly.** R2 votes land `HandshakePayload`, `AuditEvent`, `logical_system_id`, `RegistrationToken`, `RetryPolicy`, `CircuitBreaker`, `DrainInProgress`, `IClock`, `LatencyHistogram`, `AlertRule`, `IClockSync`, `PhaseId`, `IFaultInjector`. A4 verified that each of these has a single deterministic insertion site in `appendix-a-glossary.md` (the table is alphabetical by term). No cross-type naming collisions surfaced in the stress replay.

## 3. Cons

Two new, stress-round-only cons:

- **Terminal-state spelling triad** — `COMPLETED(error)` (`06-scenario-view.md:95`), `ERROR` (`06-scenario-view.md:93,111`), and `FAILED` (`06-scenario-view.md:91,108`). With A3-P1 agreed, this is guaranteed to cause reader confusion at the Task-vs-Worker-vs-AICore state layer. Captured as **R3-P2**. Cites **D7, V5**.
- **Worker-state drift** — `appendix-b-codebase-mapping.md:42-43` carries a `TaskState` row but no `WorkerState` row, even though `modules/scheduler.md §3.2` + `03-worker.md §2.1.4.1` both define it and A5-P9 adds `QUARANTINED`. Captured as **R3-P1**. Cites **D7**.

---

## 4. Proposals (new this round)

Two new low-impact proposals surfaced by the stress replay. Both are pure prose/anchor edits; `hot_path_impact: none`.

| id | severity | summary | affected_docs | hot_path_impact | trade_offs | sanity_test |
|----|----------|---------|---------------|-----------------|------------|-------------|
| A4-R3-P1 | medium | Add `WorkerState` row to `appendix-b-codebase-mapping.md` and glossary entry in `appendix-a-glossary.md`; enumerate `READY, BUSY, DRAINING, RETIRED, FAILED, QUARANTINED` (the last conditional on A5-P9 landing, which it has) | `appendix-b-codebase-mapping.md`, `appendix-a-glossary.md` | none | Gain: symmetry with `TaskState` (A4-P8/P9), A4 rubric #6 closes; lose: one new table row + one glossary row | `rg -n "QUARANTINED" docs/pypto-runtime-design/` finds `modules/scheduler.md §3.2`, `03-worker.md §2.1.4.1`, Appendix-B, and glossary. |
| A4-R3-P2 | low | Unify the terminal-failed-Task spelling. After A3-P1 lands, `06-scenario-view.md` lines 91/93/95/108/111 use three spellings (`FAILED` / `ERROR` / `COMPLETED(error)`). Rewrite all three to the single canonical pair `COMPLETING → ERROR` (and for the Worker-level object `FAILED`) per A3-P1's normative state machine. | `06-scenario-view.md`, `04-process-view.md` §4.3 prose | none | Gain: D7; lose: one search-replace pass | `rg -n "COMPLETED\(error\)" docs/pypto-runtime-design/` returns zero hits; scenarios 6.2.1 / 6.2.2 prose match `modules/core.md §2.3` TaskState enum verbatim. |

### Proposal detail

#### A4-R3-P1: Worker-state inventory parity with Task-state

- **Rationale:** D7 + A4 rubric #6. After A5-P9 adds `QUARANTINED`, the set of Worker states is `{READY, BUSY, DRAINING, RETIRED, FAILED, QUARANTINED}`. Currently neither Appendix-B nor the glossary enumerates them. A4-P8 closes this gap for `TaskState`; the same discipline must apply to `WorkerState` or A4 rubric #6 regresses the moment A5-P9 lands.
- **Edit sketch:**
  - Files: `appendix-b-codebase-mapping.md` (after line 43), `appendix-a-glossary.md` (alphabetical; before `Worker`)
  - Delta: one Appendix-B row `WorkerState (Host/Device) | core/worker.h | 6 states (READY, BUSY, DRAINING, RETIRED, FAILED, QUARANTINED); see modules/scheduler.md §3.2` + one glossary row `WorkerState → lifecycle of a Worker; see 03-worker.md §2.1.4.1`.
- **Trade-offs:** minimal; one-time doc addition.
- **Sanity test:** all six state names resolve to the same enum in `03-worker.md §2.1.4.1` + `modules/scheduler.md §3.2`.

#### A4-R3-P2: Unify terminal-failed-Task spelling across Scenario View

- **Rationale:** D7 + V5. Scenarios 6.2.1 and 6.2.2 mix spellings: `06-scenario-view.md:91` "marks AICore₃ as FAILED" (Worker-object FAILED — correct), `06-scenario-view.md:93` "parent Orchestration Task transitions COMPLETING → ERROR" (Task-state ERROR — correct post-A3-P1), and `06-scenario-view.md:95` "Host Scheduler: Task COMPLETED(error)" (a third spelling that conflates the state with the error payload). After A3-P1 agrees, `06-scenario-view.md:95` must read `Task → ERROR; error_context propagated via on_retire(err=...)` and must not introduce a synthetic `COMPLETED(error)` that does not exist in `modules/core.md §2.3`.
- **Edit sketch:**
  - File: `docs/pypto-runtime-design/06-scenario-view.md` (lines 95, 111)
  - Delta: Rewrite "Task COMPLETED(error); `notify_parent_complete` with error context" → "Task → ERROR; `notify_parent_error(parent, ErrorContext)` propagates upward" (line 95). Similarly align the `CONTINUE_REDUCED` branch on line 112 to match the canonical state spelling.
- **Trade-offs:** pure rewording; no semantic change; removes a synthetic spelling that would otherwise cause glossary drift in R4.
- **Sanity test:** no instance of `COMPLETED\(` in `06-scenario-view.md`; all terminal task states resolve to `{COMPLETED, ERROR, CANCELLED?}` per `modules/core.md §2.3` post-A3-P1.

---

## 5. Revisions of own proposals

All R2 revisions carry forward. A4-P5 branches now mostly resolved by R2 vote tallies; A4-P8 / A4-P9 amendments remain conditional only on A3-P1 (already agreed in R2).

| id | action | amended_summary (if amend/split) | reason |
|----|--------|----------------------------------|--------|
| A4-P1 | defend | — | HAL enum casing canonicalization. No R2 peer objection; R3 stress replay confirms `hot_path_impact: none`. Scenario 6.1.1 step 5-6 never cites `Variant`/`SimulationMode`, so the rename is local to `modules/hal.md` + ADR-011 index. |
| A4-P2 | defend | — | Broken anchor fix. Trivial; stress-replay confirms target still at `02-logical-view/02-scheduler.md §2.1.3.1.A` (unchanged by any R2 amendment). |
| A4-P3 | defend | — | Invariant count prose. Unchanged; no R2 proposal touches `02-logical-view/12-dependency-model.md §2.10.5`. |
| A4-P4 | defend | — | Q9 reorder. Unchanged by R2. |
| A4-P5 | **amend (finalize split)** | **Final text:** Add glossary entries for `EventLoopDeploymentConfig`. **DROP** the `SourceCollectionConfig` row (A9-P7 agreed — the config is folded into `EventHandlingConfig`). **DOWNGRADE** the `IEventCollectionPolicy` row to a one-line glossary pointer that reads *"interface name reserved for policy discussion in `09-open-questions.md`; no concrete interface in v1"* — iff A9-P2 is agreed in R3 (see §6 vote). If A9-P2 remains disputed/rejected, restore the full glossary entry pointing at `02-logical-view/09-interfaces.md §2.6.4`. | Both conditional branches are now pinned: A9-P7 resolved agree; A9-P2 resolved as part of §6 R3 re-vote. |
| A4-P6 | defend | — | ADR back-links in top-level views. Stress replay (scenario 6.1.1 + 6.1.2) finds these now directly actionable: 03 cites ADR-011 (at §3.3 cross-compile table); 04 cites ADR-010 at the event-loop section; 06 cites ADR-001 (4+1 views), ADR-008 (distributed), ADR-010 (event loop), ADR-012 (scheduler levels). |
| A4-P7 | defend | — | L0 label unification. Compatible with A7-P2 (role-interface split): the label becomes `Scheduler: AicoreDispatcher` (implementing the split role interfaces `ISchedulerSubmit` + `ISchedulerLifecycle` + ...). Stress replay of scenario 6.1.1 step 5 (`06-scenario-view.md:23`) confirms the narrative label is consumed in only two surface points (Physical View diagram + prose); the amendment is idempotent. |
| A4-P8 | defend | Amendment unchanged from R2. With A3-P1 agreed, the `TaskState` annotation becomes "10 lifecycle states + FAILED + ERROR [+ CANCELLED if adopted]". | |
| A4-P9 | defend | Amendment unchanged from R2. Enumerate the full state list plus `ERROR`. | |

---

## 6. Votes on peer proposals (R3 disputed-only re-vote)

A4's R2 vote tally on the 107 agreed proposals carries forward unchanged (all 92 agree / 9 abstain, 0 blocking, 0 override; see `round-2/reviews/A4-doc-consistency.md §6`). This section records **only** A4's R3 re-vote on the three disputed proposals, per the prompt.

| proposal_id | vote | rationale (cite rule id) | blocking | override_request |
|-------------|------|--------------------------|----------|-------------------|
| A2-P7 | **agree (R3 flip from abstain)** | A2's R2 amendment explicitly converts the proposal into a Q-record in `09-open-questions.md` naming the two future policy axes — this matches the synthesis recommendation (`round-2/synthesis.md:179`). Under the Q-record framing, A4 has a clear D7 win: one new Q-section with stable ID; no new interface → no glossary churn; no V5 diagram impact. Cites **D7, V5, G5**. Prefer the Q-record live at the end of `09-open-questions.md` (appended as Q15, not renumbered) to keep Q1–Q14 anchors stable after A4-P4 reorder. | false | false |
| A9-P2 | **agree (R3 flip from abstain)** | A9's R2 amendment commits to: (i) single `IEventLoopDriver` test-only seam with `step()` + `RecordedEventSource`, (ii) closed enums for deployment modes, (iii) appendix listing future extension interfaces. From A4's rubric: (i) is one new glossary row (`IEventLoopDriver`) — clean D7; (ii) closed enums are preferred for glossary stability (A4 R1 Con #5 specifically called out speculative pluggable entries); (iii) the appendix entry resolves A2's OCP concern as an A4-style doc artifact. A4-P5's branch for `IEventCollectionPolicy` collapses to a one-line glossary pointer (§5). Cites **D7, V5**. Synthesis recommendation (`round-2/synthesis.md:188`) is the same landing zone. | false | false |
| A9-P6 | **agree, option (iii) (R3 flip from abstain)** | A4 endorses the synthesis's recommended middle ground — ship FUNCTIONAL-only implementation, **keep the `SimulationMode` enum open**, **keep REPLAY scaffolding** so A6-P12 and A8 can land day 2. From A4's rubric: option (iii) has the minimum doc-consistency cost — `appendix-a-glossary.md:69,79` entries for `SimulationMode` and `Replay` stay as-is; ADR-011-R2 gets the named trigger list; `modules/hal.md` variant table keeps `REPLAY` row as "scaffold; see ADR-011-R2"; `09-open-questions.md` gets the trigger-condition record. Options (i) and (ii) would each force asymmetric doc edits (either remove the enum member and break the glossary/ADR/platform-doc trio, or commit to a full REPLAY implementation path that A9 has demonstrated is not required for v1). Cites **D7, E6, G5**. | false | false |

Net effect: all three disputed proposals flip to agree from A4's chair, consistent with the synthesis's predicted landing zone (`round-2/synthesis.md:205-219`).

---

## 7. Cross-aspect tensions (stress-round observations)

| pair | with_proposal | preferred_resolution |
|------|---------------|----------------------|
| A4 vs A3 | A4-P8 / A4-P9 / R3-P1 / R3-P2 vs A3-P1 (add ERROR/CANCELLED) | Land A3-P1 first (state-machine edit in `modules/core.md §2.3` + `04-process-view.md §4.3`), A4-P8/P9 in the same PR (appendix + glossary), and R3-P2 in the follow-up that syncs `06-scenario-view.md` prose. Zero structural conflict; pure ordering. |
| A4 vs A5 | R3-P1 vs A5-P9 (`QUARANTINED` Worker state) | R3-P1 **must** ride in the same PR as A5-P9. Otherwise Appendix-B + glossary miss `QUARANTINED` at the moment the state-machine edit lands in `03-worker.md §2.1.4.1`, reopening a drift A4 rubric #6 just closed. No blocking; coordination only. |
| A4 vs A9 | A4-P5 (conditional glossary) vs A9-P2 amendment (`IEventLoopDriver`, closed enums, extension appendix) | A4-P5's final branch collapses cleanly under A9-P2-amended: `SourceCollectionConfig` dropped (A9-P7), `IEventCollectionPolicy` becomes a one-liner pointer to `09-open-questions.md` (A9-P2). Net: 1 glossary row added (`EventLoopDeploymentConfig`), 0 removed (because the drop was never present in glossary), 1 one-liner added. Smallest possible final state. |
| A4 vs A2 | A4-P5 `IEventCollectionPolicy` pointer-row vs A2's override on A9-P2 (want ADR-listed extensions) | The A9-P2 amendment's "appendix of future extensions" is the A2 win; A4 writes the glossary pointer to that appendix section rather than to `02-logical-view/09-interfaces.md §2.6.4`. Single coherent link target; no divergence. |
| A4 vs A1 | A4-P1..P9 + R3-P1/R3-P2 vs A1 hot-path set | All A4 proposals remain `hot_path_impact: none`. R3-P1/R3-P2 are pure prose/anchor/table edits; R3 stress replay of the critical paths (`06-scenario-view.md:17-30`, `06-scenario-view.md:43-56`) introduces zero allocation, zero blocking, zero layout churn, zero latency change. |

No blocking cross-aspect tension observed from A4 this round.

---

## 8. Stress-attack on emerging consensus

**Method.** For every agreed proposal that meaningfully touches A4's rubric (D7/V1/V2/V5), I attempt the strongest attack an A4 reviewer can mount against the R2-amended text, then replay the relevant scenario step(s) from `06-scenario-view.md` to see whether vocab / component label / ADR cite survive. Proposals that do not intersect A4's rubric (e.g., pure algorithmic or HPI-internal amendments like A1-P1, A1-P2, A1-P7, A1-P9, A10-P7) are not attacked here because A4 has no standing — I confirm in one line per group that the R2 amendment introduces no new doc-level term that would drift.

Stress-attack verdicts: 22 `holds`, 0 `breaks`, 2 `uncertain (minor amendment)` — R3-P1 and R3-P2 cover both uncertainties with concrete amendments (§4). No proposal receives a `blocking=true` from A4.

### 8.1 A4-relevant agreed proposals — per-proposal stress attack table

| proposal_id | stress_attempt | scenario line(s) | verdict |
|-------------|----------------|-------------------|---------|
| R3-X1 / A4-P1 (HAL enum casing) | A9 could challenge: "mixing `ONBOARD/SIM` canonical with dev-column prose creates a third naming axis (`onboard`/`sim` lowercase in file paths)." | `06-scenario-view.md:24` (AICore hardware), `modules/hal.md:163-164` | **holds** — scenario prose uses the logical term "AICore hardware", never the enum spelling. The rename is wholly contained in `modules/hal.md` + ADR-011; sanity test `rg -n "Onboard|Performance,|Functional,|Replay " docs/pypto-runtime-design/modules/hal.md` returns 0 post-edit. |
| R3-X2 / A4-P2 (anchor fix) | Strongest attack: the fixed anchor target might be renumbered by some other R2 amendment. | `02-logical-view/02-scheduler.md §2.1.3.1.A` (no R2 amendment touches this section's heading) | **holds** — no R2 proposal renumbers §2.1.3.1.A. Anchor `#2131a-submission-admission--outstanding-window` remains stable. |
| R3-X3 / A4-P7 (L0 label unification) | A7's role-interface split (A7-P2) could leave the Physical View label awkward: `Scheduler: AicoreDispatcher (implementing ISchedulerSubmit+ISchedulerLifecycle+...)` — too long for the diagram box. | `06-scenario-view.md:23` step 5: "Chip Scheduler dispatches compute_A to AICore₀" — Chip Scheduler is L1; L0 label is in `05-physical-view.md:37` | **holds** — A4-P7's edit sketch is surgical: replace diagram box `│ Dispatcher: AICore dispatch (register poll)  │` with `│ Scheduler: AicoreDispatcher (register poll)  │` (same character budget). Role-interface lists go in the glossary row for `AicoreDispatcher`, not in the Physical View box. Scenario 6.1.1 step 5 is unaffected — it references "Chip Scheduler", not the L0 box. |
| R3-X4 / A4-P8 + A4-P9 (TaskState enumeration) | A3's amendment adds `ERROR` + optional `CANCELLED`; glossary and Appendix-B must list **exactly** the enum in `modules/core.md §2.3` with no extra / missing states. Attack: A3's amendment is conditional on `CANCELLED`, so the appendix enumerates a state that may not exist. | `06-scenario-view.md:20` (SUBMITTED → PENDING → DEP_READY → RESOURCE_READY → DISPATCHED), `06-scenario-view.md:28` (COMPLETING → COMPLETED), `06-scenario-view.md:93` (COMPLETING → ERROR) | **holds** — A4-P8/P9 amendments explicitly carry the conditional "`CANCELLED` if adopted" so the appendix tracks the enum's final shape; A4 commits to re-reading `modules/core.md §2.3` at PR merge time to confirm the state set before the appendix row is finalized. |
| R3-X5 / A3-P1 (ERROR/CANCELLED states) | Strongest attack: the scenario walkthroughs use a third spelling (`COMPLETED(error)`) on `06-scenario-view.md:95`, which is not in the new enum. After A3-P1 lands, this prose is inconsistent. | `06-scenario-view.md:91,93,95,108,111` | **uncertain (minor amendment)** — A3-P1 itself is a clean state-machine addition; the drift is in the *scenario prose*, not in A3-P1. **Captured as R3-P2 in §4.** Not blocking; A3-P1 still `holds` at the state-machine level. |
| R3-X6 / A5-P9 (QUARANTINED Worker state) | Adds `QUARANTINED` to `WorkerState` but nothing in R1/R2 adds the corresponding glossary / Appendix-B row. | `06-scenario-view.md:91` "marks AICore₃ as FAILED" (Worker-state FAILED referenced) | **uncertain (minor amendment)** — The scenario text references `FAILED` only; `QUARANTINED` is introduced in `modules/scheduler.md §3.2` + `03-worker.md §2.1.4.1` without a glossary/appendix touchpoint. **Captured as R3-P1 in §4.** Not blocking; A5-P9 still `holds` at the state-machine level. |
| R3-X7 / A7-P2 (split ISchedulerLayer) | Strongest attack: every doc that used "`ISchedulerLayer`" as a monolithic term (including glossary, interfaces doc, module docs, Appendix-B) must be updated in one atomic PR or an intermediate reader sees a name that no longer exists. | `02-logical-view/09-interfaces.md §2.6.2`, glossary entry `ISchedulerLayer` | **holds** — R2 vote unanimous; A7's R2 amendment explicitly enumerates the role interfaces (`ISchedulerSubmit`, `ISchedulerLifecycle`, ...). A4 commits to extending glossary + `02-logical-view.md §2.7 Interfaces-at-a-Glance` + Appendix-B in the same PR. Scenario 6.1.1 uses "Chip Scheduler", "Device Scheduler" etc. — narrative labels, not interface names — so the Scenario View survives unchanged. |
| R3-X8 / A4-P6 (ADR cross-links) | A9 could object that inline ADR cites clutter every view. | `06-scenario-view.md:48` REMOTE_SUBMIT step (ADR-008 distributed), `06-scenario-view.md:51` RDMA step (ADR-008 + ADR-012), `06-scenario-view.md:20` PENDING→DEP_READY (ADR-010) | **holds** — A4-P6 R2 amendment commits to inline single-citation only where the decision is local; no "Related ADRs" section. Stress replay: scenario 6.1.2 step 4 gets one `[ADR-008]` link; scenario 6.1.1 step 2 gets one `[ADR-010]` link; no view grows by more than ~4 citations. V2 rubric closes. |
| R3-X9 / A2-P1 (schema_version on data contracts) | Version field must appear in every cross-module contract; glossary must track the new `schema_version` row. | `06-scenario-view.md:48` REMOTE_SUBMIT payload (introduces `schema_version: u16`) | **holds** — A2-P1's R2 amendment is normative in `modules/transport.md §2.3` and `07-cross-cutting-concerns.md` (new Appendix-C). Scenario 6.1.2 step 4 gains a one-line annotation "REMOTE_SUBMIT message carries `schema_version=1`"; A4 extends glossary with the `schema_version` term. |
| R3-X10 / A2-P4 + A2-P5 (migration plan + BC policy in Appendix-C) | Strongest attack: a new appendix (Appendix-C) could drift from existing Appendix-A/B voice and structure. | all views reference ADRs (via A4-P6) → Appendix-C is the new back-link target | **holds** — A4 authors the Appendix-C skeleton (per R2 §7 tension row A4-vs-A2) with two sections: BC class per interface (A2) and ADR back-link index (A4). Single appendix, single voice. No scenario step references Appendix-C directly; its role is cross-cutting. |
| R3-X11 / A9-P5 (unify AdmissionStatus) | A4 rubric #1 highlight: one concept, one name. Attack vector: the rename must propagate from `modules/scheduler.md §2.1.3.1.A` through `02-logical-view/09-interfaces.md §2.6.2` and glossary. | `06-scenario-view.md:19` step 1 "state → SUBMITTED" — precedes admission, doesn't reference decision enum directly | **holds** — scenario prose doesn't mention the enum by name, so only definitional sites need updating. Clean textbook A4 rubric-#1 fix. |
| R3-X12 / A9-P7 (fold SourceCollectionConfig) | Attack: glossary entries for `SourceCollectionConfig` created anywhere else must be dropped; if `02-logical-view/09-interfaces.md §2.6.4` has a narrative reference, it must be folded. | `02-logical-view/09-interfaces.md §2.6.4`, `appendix-a-glossary.md` (no existing entry) | **holds** — A9-P7 edit sketch explicitly drops `SourceCollectionConfig`; A4-P5 final text no longer adds it (see §5). Zero ghost references. |
| R3-X13 / A7-P8 (consolidate ScopeHandle) | Attack: two existing definitions (in `core/` and `memory/` doc sections) must both be updated to point at the consolidated `core/types.h` location. | glossary `ScopeHandle`, `modules/core.md`, `modules/memory.md` | **holds** — A7-P8 edit sketch is a pure D7 textbook case; A4 supports the consolidation and extends glossary to one authoritative row. |
| R3-X14 / A7-P9 (dedupe Python MemoryError) | Attack: `simpler.errors` namespace must not have two `MemoryError` classes. | `modules/bindings.md §2.1`, glossary `simpler.MemoryError` | **holds** — trivial name collision removal; Python convention says we inherit the builtin or use a distinct name (`simpler.DeviceMemoryError`). A7-P9 edit sketch picks the latter. |
| R3-X15 / A6-P9 (logical_system_id on MessageHeader) | Adds a critical new field to wire format; Scenario 6.1.2 step 4 prose must mention it or step-6 cross-node execution becomes ambiguous. | `06-scenario-view.md:48` REMOTE_SUBMIT via IHorizontalChannel | **holds** — A6-P9's R2 amendment commits to `logical_system_id: u64` as the first field in `MessageHeader`. Scenario 6.1.2 step 4 gains a one-line annotation "header.logical_system_id set from local scope" (non-structural; pure narrative). Glossary gains `logical_system_id` row. |
| R3-X16 / A6-P2 (HandshakePayload + credential_id) | New formal types must land in glossary; scenario HANDSHAKE step must reference the field. | `06-scenario-view.md` does not have an explicit HANDSHAKE scenario — this is a gap for A3 (scenario coverage), not A4 | **holds** — A6-P2's doc sites are `modules/transport.md §8` and the (new) §7.1 trust-boundary table. A4 extends glossary with `HandshakePayload`, `credential_id`, `node_identity` rows. No V5 diagram impact. |
| R3-X17 / A6-P4 (TLS default for TCP control plane) | Updates `10-known-deviations.md` Deviation 3. Attack: the deviation's title must match the new reality ("RDMA plaintext is a scoped exception", not a blanket one). | `06-scenario-view.md:52` Node₁ → Node₀ REMOTE_COMPLETE (TCP-class control path) | **holds** — A6-P4's R2 amendment rewrites Deviation 3 with explicit RDMA-only scope. Scenario 6.1.2 step 8 is a control-path message — default TLS applies. Scenario 6.1.2 step 7 (RDMA_WRITE) stays explicitly exempt. Prose alignment is automatic. |
| R3-X18 / A8-P2 (single IEventLoopDriver seam) | Tied to A9-P2. After A9-P2 amendment, `step()` + `RecordedEventSource` is the ONLY test seam; glossary must track this single canonical name. | not scenario-touching (test-only seam) | **holds** — one glossary row (`IEventLoopDriver`), one module-doc row; A9-P2's amended appendix of future extensions lists the names that could attach to this seam later (doc-only). |
| R3-X19 / A8-P1 (injectable IClock) | Adds `hal::IClock`; determinism claim at `01-design-principles.md:389` must be back-linked. | not scenario-touching | **holds** — A4 extends glossary with `IClock`; `01-design-principles.md:389` updated to cite A8-P1 via ADR-N... (new ADR) or inline reference to `modules/hal.md §2.x`. |
| R3-X20 / A10-P3 + A10-P4 + A10-P8 (Data & State Reference) | Combining consistency table + stateful/stateless classification + ownership diagram into **one** `07-cross-cutting-concerns.md` section is an A4 follow-through; the section must not drift from `02-logical-view/11-machine-memory-model.md` and `02-logical-view/12-dependency-model.md`. | `06-scenario-view.md:146-154` cross-view consistency table (existing) — the new section is upstream of this | **holds** — A4 commits (per R2 §7 tension row A4-vs-A10) to authoring a single "Data & State Reference" section in `07-cross-cutting-concerns.md` that combines all three A10 proposals. One section, consistent voice. |
| R3-X21 / A5-P1 + A5-P2 + A5-P8 (RetryPolicy + CircuitBreaker + degradation enums) | Many new formal types in a short span; glossary must not split (one row per type) and the Appendix-C BC table must list them. | `06-scenario-view.md:137` Network Timeout scenario → RetryPolicy activates (`06-scenario-view.md:140` "eventually timeout") | **holds** — scenario 6.2.4 text is already descriptive ("returns no completion within timeout period (Rule R1)"); A5's R2 amendment commits `RetryPolicy` and `CircuitBreaker` to `modules/transport.md §6` / `modules/distributed.md`. A4 glossary extension is mechanical. |
| R3-X22 / A6-P12 (signed REPLAY trace; contingent on A9-P6) | After R3 §6 re-vote lands A9-P6 at option (iii) (keep scaffolding), A6-P12 collapses to "schema + signing format frozen at ADR-011-R2 time, implementation deferred." Glossary must still carry `REPLAY` and the `replay_trace_signature` header. | 6.1.1 step 6 kernel execution produces trace data | **holds** — under the R3 option (iii) landing, A6-P12 lands with the enum + scaffolding retained; glossary does not churn (REPLAY stays), one new row added for `replay_trace_signature`. Zero Scenario View impact. |
| R3-X23 / A7-P4 (move distributed payload structs to distributed/) | DAG edit; changes ownership column in Appendix-B. | `06-scenario-view.md:48` REMOTE_SUBMIT payload — ownership column must point at `distributed/` not `transport/` | **holds** — A7-P4's R2 amendment is unanimous; A4's rubric #6 requires the Appendix-B ownership column to update. Scenario 6.1.2 step 4 prose currently reads "`transport/` (RDMA)" for the *channel*, which is correct after A7-P4 (channel stays in transport; payload type moves). No scenario-prose edit needed. |
| R3-X24 / A1-P6 (REMOTE_BINARY_PUSH + descriptor_template_id; absorbs A10-P5) | New `MessageType` member + new header field. Glossary + interface doc must enumerate all `MessageType` values post-amendment. | `06-scenario-view.md:48` REMOTE_SUBMIT — after A1-P6, binary payload moves to a separate PUSH message | **holds** — A1-P6's R2 amendment introduces `REMOTE_BINARY_PUSH` as a new `MessageType`. A4 extends glossary; scenario 6.1.2 step 4 narrative gains one sentence "large binary payload (if any) is shipped separately via `REMOTE_BINARY_PUSH`; small descriptors travel in `REMOTE_SUBMIT`". No structural change. |

### 8.2 Proposals outside A4's standing (confirmed no-drift)

For completeness, the following R2-agreed proposals do not introduce new formal terms / component labels / ADR cites visible in scenarios 6.1.1 / 6.1.2, and A4's stress replay confirms no ripple into D7/V1/V2/V5:

- **Performance-internal (no new doc terms):** A1-P1, A1-P2, A1-P3, A1-P5, A1-P7, A1-P8, A1-P10, A1-P11, A1-P12 (amended), A1-P13 (amended), A1-P14, A10-P1, A10-P6, A10-P7 (amended), A10-P9, A10-P10.
- **Correctness clarifications that don't rename things:** A3-P2, A3-P3, A3-P4, A3-P5, A3-P6, A3-P7, A3-P8, A3-P9, A3-P10, A3-P11, A3-P12, A3-P13, A3-P14, A3-P15, A5-P3, A5-P4, A5-P5, A5-P6, A5-P7, A5-P10.
- **Security/testability additions absorbed into R3-X15..R3-X22 above:** A6-P1, A6-P3, A6-P5, A6-P6, A6-P7, A6-P8 (absorbed into A1-P11), A6-P10, A6-P11, A6-P13, A6-P14; A8-P3, A8-P4, A8-P5, A8-P6, A8-P7, A8-P8, A8-P9, A8-P10, A8-P11, A8-P12.
- **Modularity edits that touch the DAG but not the scenario vocab:** A7-P1 (cycle break), A7-P3 (core↔hal invert), A7-P5, A7-P6, A7-P7.
- **Simplicity deletions:** A9-P1 (absorbed in A7-P2), A9-P3 (collectives off IHorizontalChannel; Q4 option A + ADR), A9-P4 (drop SubmissionDescriptor::Kind), A9-P8 (move AICore companion artifacts out of design).
- **Disputed (voted in §6):** A2-P7, A9-P2, A9-P6.

A4 re-confirms **zero breaks** and **zero `blocking=true`** for all proposals above.

---

## 9. Status

- **Satisfied with current design?** Partially → **yes (subject to PR-level coordination).** The R2-amended proposal set closes every A4 rubric finding from R1 (rubric rows 1, 4, 5, 6 move from Fail/Weak to provisional Pass post-PR). R3 stress replay confirms the amended text survives scenarios 6.1.1 / 6.1.2 / 6.2.1 / 6.2.4 with exactly two minor amendments (R3-P1, R3-P2), both low-severity and both pure prose/table edits. No proposal `breaks` under A4's stress attack; no `blocking=true` filed; the three disputed proposals flip to agree under the synthesis's recommended landing zones.
- **Open items expected in next round (R4 — amendment-application only if parent routes back):**
  - Land **R3-P1** (`WorkerState` Appendix-B + glossary row) in the same PR as A5-P9.
  - Land **R3-P2** (terminal-failed-Task spelling unification in `06-scenario-view.md`) in the same PR as A3-P1.
  - Complete the Appendix-C composition task (coordinate with A2-P5 and A4-P6).
  - Complete the "Data & State Reference" section composition task (coordinate with A10-P3/P4/P8).
  - Absent these two, **no new disputes** introduced this round. A4 recommends the parent treat this as **converged** per the SKILL.md §Phase 4 rule ("stress round passed with no new disputes introduced or at most a handful of minor amendments") — the two R3 amendments are both low-severity minor amendments in A4's own column and do not require a return to the owners.
