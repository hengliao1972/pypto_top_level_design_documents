# Aspect A4: Document Consistency — Round 1

## Metadata

- **Reviewer:** A4
- **Round:** 1
- **Target:** `docs/pypto-runtime-design/` (4+1 views, 12 logical sub-docs, 10 module docs, 2 appendices, ADR set, open questions, known deviations)
- **Runtime Mode:** yes (Performance A1 ×2 weight; all proposals carry `hot_path_impact`)
- **Timestamp (UTC):** 2026-04-18T17:21:59Z

---

## 1. Rubric Checks

| # | Check | Result | Evidence (file:line) | Rule(s) |
|---|-------|--------|----------------------|---------|
| 1 | Same concept named identically across views, diagrams, module docs, ADRs, glossary | **Fail** | `modules/hal.md:163-164` declares `enum class Variant { Onboard, Sim }` and `enum class SimulationMode { Performance, Functional, Replay }`, but `02-logical-view/10-platform.md:19-20,30-32`, `appendix-a-glossary.md:69,79`, `08-design-decisions.md` ADR-011 use `ONBOARD`/`SIM` and `PERFORMANCE`/`FUNCTIONAL`/`REPLAY` everywhere else (same members repeated in `modules/hal.md:394-395`) | D7, V5 |
| 2 | Logical component ↔ module ↔ process ↔ physical node mapping present and tight | **Weak** | Logical→Development mapping is complete (`02-logical-view.md:185-215`, `03-development-view.md:107-118`). But Physical View labels L0 as **`Dispatcher: AICore dispatch (register poll)`** (`05-physical-view.md:37`) while Development View treats it as a `scheduler/levels/aicore_dispatcher.cpp` implementing `ISchedulerLayer` (`03-development-view.md:49`) and Scenario walkthroughs refer to "Core Scheduler" events — three names for the same L0 `ISchedulerLayer` instance | V2, V5, D7 |
| 3 | All required views present and non-empty | **Pass** | `00-index.md:10-24` enumerates 01–10 + appendices + `modules/`; all files are populated (Draft status per `00-index.md:48-60`) | V1 |
| 4 | ADRs referenced from the views they constrain (and vice versa) | **Fail** | `08-design-decisions.md` contains ADR-001…ADR-013, but `01-introduction.md`, `02-logical-view.md` (overview), `03-development-view.md`, `04-process-view.md`, `05-physical-view.md`, `06-scenario-view.md` contain **zero** `ADR-NNN` cross-references (verified by grep). Only sub-docs (`02-logical-view/*.md`), `07-cross-cutting-concerns.md`, `09-open-questions.md`, and `modules/*.md` cite ADRs | V2, G5 |
| 5 | Glossary covers every formal term in the docs | **Weak** | `EventLoopDeploymentConfig` (used in `02-logical-view.md:191,198,213`, `02-logical-view/02-scheduler.md:312`, `02-logical-view/09-interfaces.md:429-434`, `modules/scheduler.md:162,551`), `SourceCollectionConfig` (same files), and `IEventCollectionPolicy` (`02-logical-view.md:191,198,213`) are **not** in `appendix-a-glossary.md`; yet sibling terms `EventHandlingConfig`, `IEventSource`, `IEventQueue`, `IExecutionPolicy` are defined there (`appendix-a-glossary.md:28,40-42`) | D7 |
| 6 | `appendix-b-codebase-mapping.md` matches current Development View and module docs | **Weak** | Appendix B (`appendix-b-codebase-mapping.md:42-43`) says the **target** `core/task.h` has "(10 states)"; `modules/core.md:156-167` defines the `TaskState` enum with 10 named values plus a `FAILED` error state (11), and `04-process-view.md:229-285` draws a diagram including the additional `FAILED` state. Count annotation is off-by-one against the actual target enum | D7 |

Internal consistency defects surfaced while running the above checks (all cited under the Cons list below):

- `02-logical-view/12-dependency-model.md:107` prose says "**Two** invariants hold across the pipeline" but then enumerates **three** invariants I1/I2/I3 (`:109,111,113`). This is an arithmetic-in-prose inconsistency a reader will notice immediately.
- `02-logical-view/07-task-model.md:45` hyperlinks the Outstanding Submission Window anchor as `02-scheduler.md#21_3_1_a-submission-admission--outstanding-window` (underscored digit segment), whereas **every other reference** to the same heading in the repo uses `#2131a-submission-admission--outstanding-window` (`appendix-a-glossary.md:11,65`, `08-design-decisions.md:524,562`, `modules/scheduler.md:26,478`, `02-logical-view/12-dependency-model.md:80`, `02-logical-view/09-interfaces.md:65`). The target heading is `## 2.1.3.1.A Submission Admission & Outstanding Window` (`02-logical-view/02-scheduler.md:37`), which GitHub-flavored markdown renders as `#2131a-submission-admission--outstanding-window`; the underscored form is a broken fragment.
- `09-open-questions.md` numbers questions `Q1…Q8, Q10, Q11, Q12, Q13, Q14, Q9` — `Q9` is the **last** section rather than between Q8 and Q10 (`09-open-questions.md:99,112,142,159,176,193`).
- `appendix-a-glossary.md:88` defines `Task State` as "Lifecycle phase (FREE → SUBMITTED → ... → RETIRED)", citing `04 §4.3`. `modules/core.md:156-167` names **10** states; the glossary's ellipsis hides them. This is not wrong but contributes to glossary drift.

---

## 2. Pros

- **Glossary is extensive and back-links into source sections.** `appendix-a-glossary.md` alphabetizes 90+ terms and cross-references each to the defining section. This is a strong ubiquitous-language foundation (D7) rarely achieved in draft architectures.
- **Logical → Development mapping is near-complete.** `02-logical-view.md:§2.7 / §3` (module breakdown table) maps every Logical pillar to a concrete Development View module (`03-development-view.md:107-118`, `:200-212`), and per-module Section-8 docs under `modules/` exist for all 10 modules (`03-development-view.md:157-169`). This satisfies V2 for the Logical↔Development pair.
- **Cross-View Consistency section exists.** `06-scenario-view.md` explicitly contains a "Cross-View Consistency Checks" table (per prior review of the file), signalling authorial awareness of V2. (Verified by preceding task summary; file present per `00-index.md:17`.)
- **Anchor convention is otherwise consistent.** Reference links use the GitHub-flavored digit-only anchor form (`#2131a-...`, `#2135-...`) uniformly across `appendix-a-glossary.md`, `08-design-decisions.md`, `modules/scheduler.md`, `02-logical-view/12-dependency-model.md`, `04-process-view.md:31`. Only one file (`07-task-model.md:45`) diverges — a localized fix, not a systemic one.
- **Ubiquitous language applied to interface names.** `ISchedulerLayer`, `IMemoryManager`, `IMemoryOps`, `IVerticalChannel`, `IHorizontalChannel`, `IPartitioner`, `ITaskSchedulePolicy`, `IWorkerSelectionPolicy`, `IResourceAllocationPolicy`, `ISchedulerEventSource`, `IEventQueue`, `IExecutionPolicy` match across Logical, Development, Interfaces, and all module docs (verified via grep of `02-logical-view.md:191-215` and `03-development-view.md:300-307`). D7 is largely honored at the interface layer.
- **Known Deviations section formalizes rule-exception process.** `10-known-deviations.md` explicitly lists rule violations with Rule IDs (S5, R5, V1), exactly as required by `04-agent-rules.md` "Rule Exceptions" procedure.

---

## 3. Cons

- **Case mismatch in HAL enum declarations vs. prose/glossary/ADR** — `modules/hal.md:163-164,394-395` uses CamelCase `Onboard`/`Sim` and `Performance`/`Functional`/`Replay`, while `02-logical-view/10-platform.md:19-20,30-32`, `appendix-a-glossary.md:69,79`, `08-design-decisions.md` ADR-011, and `03-development-view.md:238-241` use the UPPER_SNAKE forms `ONBOARD`/`SIM`/`PERFORMANCE`/`FUNCTIONAL`/`REPLAY`. A reader doing a code-search will disagree with the prose. Cites **D7, V5**.
- **Broken anchor** — `02-logical-view/07-task-model.md:45` link target `#21_3_1_a-submission-admission--outstanding-window` does not match the generated GitHub anchor for heading `2.1.3.1.A Submission Admission & Outstanding Window` at `02-logical-view/02-scheduler.md:37`. All other 11+ references use the correct form. Cites **D7, V5**.
- **Prose arithmetic bug in invariants block** — `02-logical-view/12-dependency-model.md:107` says "Two invariants hold" but lists three (I1, I2, I3). Any reviewer or downstream reader will stall here. Cites **D7** (ubiquitous language / document integrity), **G4** (clarity).
- **Out-of-order Open Questions** — `09-open-questions.md` section ordering `Q1…Q8, Q10…Q14, Q9` (`:7-193`) breaks numeric contract and cross-references that count-down from Q14 won't find Q9 adjacent. Cites **D7, V5**.
- **Glossary drift on event-loop infrastructure** — `EventLoopDeploymentConfig`, `SourceCollectionConfig`, `IEventCollectionPolicy` are formally introduced in `02-logical-view/02-scheduler.md:312` and `02-logical-view/09-interfaces.md:429-456` but absent from `appendix-a-glossary.md`. This specific trio appears 10+ times across logical-view + module docs and anchors ADR-010's implementation story — omissions undercut A4.5. Cites **D7**.
- **Top-level views never cite ADRs** — none of `01-introduction.md`, `02-logical-view.md` (overview), `03-development-view.md`, `04-process-view.md`, `05-physical-view.md`, `06-scenario-view.md` contain an `ADR-NNN` reference. Per the rubric ("ADRs referenced from the views they constrain, and vice versa"), a view enumerating cross-compile variants (03) should cite ADR-011 (Simulation Facility); a view describing the scheduler event loop (04) should cite ADR-010; a view describing deployment (05) should cite ADR-005; a view walking scenarios (06) should cite ADR-001, ADR-008, ADR-010, ADR-012. Cites **V2, G5**.
- **L0 component naming diverges across views** — Physical View (`05-physical-view.md:37`) labels L0 as "Dispatcher: AICore dispatch (register poll)"; Development View (`03-development-view.md:49`) names the file `aicore_dispatcher.cpp` under `scheduler/levels/` implementing `ISchedulerLayer`; Logical View text treats every level including L0 as a Scheduler. The mismatch ("Dispatcher" vs. "Scheduler") hurts V5 diagram consistency and forces a reader to infer that the L0 "Dispatcher" is actually an `ISchedulerLayer` implementation. Cites **V5, D7**.
- **Appendix-B state-count annotation out of date** — `appendix-b-codebase-mapping.md:42-43` labels the target `TaskState` as "(10 states)", but the current target enum in `modules/core.md:156-167` has 10 normal states plus `FAILED` (total 11 named values; the `04-process-view.md:229-285` diagram shows the `FAILED` branch explicitly). The appendix should be kept in sync or qualify as "10 success states + `FAILED`". Cites **D7**.
- **`Task State` glossary entry underspecified** — `appendix-a-glossary.md:88` collapses the entire 10-state lifecycle to "FREE → SUBMITTED → ... → RETIRED". This is readable prose but leaves `PENDING`, `DEP_READY`, `RESOURCE_READY`, `DISPATCHED`, `EXECUTING`, `COMPLETING` outside the glossary while they are load-bearing terms in ADR-010, `04-process-view.md` §4.3, and `modules/core.md` §2.3. Cites **D7**.

---

## 4. Proposals

| id | severity | summary | affected_docs | hot_path_impact | trade_offs | sanity_test |
|----|----------|---------|---------------|-----------------|------------|-------------|
| A4-P1 | high | Rewrite HAL enum declarations to the canonical UPPER_SNAKE forms (`ONBOARD`/`SIM`, `PERFORMANCE`/`FUNCTIONAL`/`REPLAY`) to match platform.md, glossary, and ADR-011 | `modules/hal.md` | none | Gain: D7 restored; give up: a one-shot edit in two table cells | `rg -n "Onboard|Performance,|Functional,|Replay " docs/pypto-runtime-design/modules/hal.md` returns zero hits afterward |
| A4-P2 | blocker | Fix broken anchor in task-model cross-reference (`#21_3_1_a-...` → `#2131a-...`) | `02-logical-view/07-task-model.md` | none | Gain: live link; give up: nothing | `rg -n "#21_3_1_a" docs/pypto-runtime-design/` returns zero hits; local markdown previewer lands on §2.1.3.1.A |
| A4-P3 | high | Fix invariant count prose in Dependency Model ("Two" → "Three") | `02-logical-view/12-dependency-model.md` | none | Gain: readers do not stall; give up: nothing | Line 107 reads "Three invariants hold..." and the enumerated count (I1/I2/I3) matches |
| A4-P4 | medium | Reorder Open Questions so Q9 sits between Q8 and Q10 | `09-open-questions.md` | none | Gain: numeric order restored; give up: one file movement | `grep "^## Q" 09-open-questions.md` yields Q1..Q14 strictly ascending |
| A4-P5 | medium | Add glossary entries for `EventLoopDeploymentConfig`, `SourceCollectionConfig`, `IEventCollectionPolicy` with source anchors | `appendix-a-glossary.md` | none | Gain: A4.5 coverage of event-loop plumbing; give up: three rows of table edit | Glossary grep finds all three terms; every use site in §02-logical-view and modules/scheduler.md is a defined term |
| A4-P6 | medium | Add an "ADR cross-references" paragraph (or inline citation) to each top-level view: 03 cites ADR-011; 04 cites ADR-010, ADR-012, ADR-013; 05 cites ADR-005, ADR-011; 06 cites ADR-001, ADR-008, ADR-010, ADR-012 | `03-development-view.md`, `04-process-view.md`, `05-physical-view.md`, `06-scenario-view.md` | none | Gain: V2 back-links honored, G5 justification traceable from scenario outward; give up: slight increase in view-prose volume | `for v in 03 04 05 06; do rg -c "ADR-" docs/pypto-runtime-design/${v}-*.md; done` returns non-zero for every view |
| A4-P7 | low | Unify L0 component label. Use **`AicoreDispatcher`** (an `ISchedulerLayer` implementation) everywhere — Logical prose, Development file, Physical box-text, Scenario narrative. The Physical View box should read `Scheduler: AicoreDispatcher (register poll)` rather than "Dispatcher:" | `05-physical-view.md`, possibly `06-scenario-view.md` | none | Gain: V5 diagram consistency; give up: 1-line edit per view | `rg -n "Dispatcher: AICore dispatch" docs/pypto-runtime-design/` returns zero hits; every L0 mention reads as an `ISchedulerLayer` implementation |
| A4-P8 | low | Update Appendix-B state-count annotation: change "(10 states)" to "(10 lifecycle states + `FAILED`)" and add explicit cross-reference to `modules/core.md#23-task-taskkey-taskhandle-taskstate` | `appendix-b-codebase-mapping.md` | none | Gain: appendix matches `modules/core.md`; give up: one-cell edit | Appendix B row for `TaskState (Host)` and `(Device)` matches the enumeration in `modules/core.md:156-167` verbatim |
| A4-P9 | low | Expand glossary entry for `Task State` to enumerate all 10 lifecycle states (plus `FAILED`) inline instead of "..." | `appendix-a-glossary.md` | none | Gain: closes glossary drift on central lifecycle vocabulary; give up: one row of table prose grows ~3 lines | Glossary row lists `FREE, SUBMITTED, PENDING, DEP_READY, RESOURCE_READY, DISPATCHED, EXECUTING, COMPLETING, COMPLETED, RETIRED` (+ `FAILED`) and matches `modules/core.md:156-167` |

### Proposal detail

#### A4-P1: Canonicalize HAL enum member casing

- **Rationale:** D7 (consistent naming) and V5 (diagram consistency). Every non-HAL doc (`02-logical-view/10-platform.md:19-20,30-32`, `appendix-a-glossary.md:69,79`, ADR-011 in `08-design-decisions.md`) uses UPPER_SNAKE (`ONBOARD`, `SIM`, `PERFORMANCE`, `FUNCTIONAL`, `REPLAY`). `modules/hal.md:163-164` alone declares `enum class Variant { Onboard, Sim }` and `enum class SimulationMode { Performance, Functional, Replay }`, and `modules/hal.md:394-395` repeats the CamelCase form. This is load-bearing — readers matching docs against code will fail the naming audit.
- **Edit sketch:**
  - File: `docs/pypto-runtime-design/modules/hal.md`
  - Location: §2.7 Public Data Types table (lines 163-164), §5 Config table (lines 394-395)
  - Delta: Replace `enum class { Onboard, Sim }` with `enum class { ONBOARD, SIM }`; replace `enum class { Performance, Functional, Replay }` with `enum class { PERFORMANCE, FUNCTIONAL, REPLAY }`. Adjust the two "dev" / "deployed" config-column examples at `:394-395` to the same case.
- **Trade-offs:**
  - Gain: D7 compliance; a single authoritative enum spelling across the design package.
  - Give up: None beyond the edit itself (HAL enums are not yet in shipped code).
- **Sanity test:** `rg -n '(Variant|SimulationMode)[^A-Z_]*\{[^}]*(Onboard|Sim|Performance|Functional|Replay)[^A-Z_]' docs/pypto-runtime-design/` returns zero matches.

#### A4-P2: Fix broken anchor to Submission Admission & Outstanding Window

- **Rationale:** D7 (ubiquitous language across links) and V5 (diagram/reference consistency). `02-logical-view/07-task-model.md:45` uses `#21_3_1_a-submission-admission--outstanding-window` but the target heading renders as `#2131a-submission-admission--outstanding-window` (GitHub strips dots, lowers, keeps double-dash from `&`). Every other citation in the repo uses the correct form (11+ sites).
- **Edit sketch:**
  - File: `docs/pypto-runtime-design/02-logical-view/07-task-model.md`
  - Location: line 45 (Outstanding Submission Window pointer)
  - Delta: `02-scheduler.md#21_3_1_a-submission-admission--outstanding-window` → `02-scheduler.md#2131a-submission-admission--outstanding-window`.
- **Trade-offs:** None (pure fix).
- **Sanity test:** `rg -n "#21_3_1_a" docs/pypto-runtime-design/` returns zero. A markdown previewer at `07-task-model.md:45` lands on §2.1.3.1.A.

#### A4-P3: Correct invariant count prose

- **Rationale:** D7 and G4 (clarity over cleverness). `02-logical-view/12-dependency-model.md:107` asserts "Two invariants" but three are enumerated (I1 line 109, I2 line 111, I3 line 113).
- **Edit sketch:**
  - File: `docs/pypto-runtime-design/02-logical-view/12-dependency-model.md`
  - Location: line 107
  - Delta: "Two invariants hold across the pipeline and are relied upon by every stage:" → "Three invariants hold across the pipeline and are relied upon by every stage:" (or reorganize I3 into a sub-point of I1/I2 if the original intent was two — reviewer recommends the count correction because I3 asserts a separate obligation on the frontend, not derivative).
- **Trade-offs:** Gain accuracy; lose nothing. The count fix is the minimum edit; an author may alternatively promote I3 out of the invariants block if the wider pipeline contract is meant.
- **Sanity test:** Section 2.10.5 header, intro sentence, and the enumerated invariant count all agree.

#### A4-P4: Reorder Open Questions numerically

- **Rationale:** D7, V5. Q9 currently trails Q14 (`09-open-questions.md:99→193`). Readers and later rounds of review should not need to scroll past newer questions to find the earlier one.
- **Edit sketch:**
  - File: `docs/pypto-runtime-design/09-open-questions.md`
  - Location: Q9 section currently at line 193
  - Delta: Cut the `## Q9: Stages 1b–8 Design Document Status` section and its body; paste between current Q8 (line 99) and Q10 (line 112).
- **Trade-offs:** Low-risk reorder; no link will break because internal references use section titles or ADR IDs.
- **Sanity test:** `rg -n "^## Q[0-9]+" 09-open-questions.md` returns Q1..Q14 in strictly ascending order.

#### A4-P5: Add missing glossary entries for event-loop plumbing

- **Rationale:** D7 (A4.5: glossary covers every formal term). `EventLoopDeploymentConfig`, `SourceCollectionConfig`, and `IEventCollectionPolicy` are referenced in ≥10 places in Logical View and module docs but have no glossary entry; sibling types `EventHandlingConfig`, `IEventSource`, `IEventQueue`, `IExecutionPolicy` *do* have entries (glossary lines 28, 40-42).
- **Edit sketch:**
  - File: `docs/pypto-runtime-design/appendix-a-glossary.md`
  - Location: Insert alphabetically (near current `EventHandlingConfig` / `Event Loop` entries at lines 28-29)
  - Delta: Add three rows:
    - `EventLoopDeploymentConfig` → struct binding Stage A/B/C to execution units; defined at [§2.6.4](02-logical-view/09-interfaces.md#264-event-and-execution-interfaces).
    - `SourceCollectionConfig` → per-source Stage-A event-collection ordering/fairness config; defined at [§2.6.4](02-logical-view/09-interfaces.md#264-event-and-execution-interfaces).
    - `IEventCollectionPolicy` → strategy interface governing Stage-A event collection across registered sources; defined at [§2.6.4](02-logical-view/09-interfaces.md#264-event-and-execution-interfaces).
- **Trade-offs:** Gain A4.5 coverage; give up three rows of table space.
- **Sanity test:** `rg -n "EventLoopDeploymentConfig|SourceCollectionConfig|IEventCollectionPolicy" docs/pypto-runtime-design/appendix-a-glossary.md` returns three matches.

#### A4-P6: Cross-link top-level views to ADRs they encode

- **Rationale:** A4.4 / V2: the rubric asks that "ADRs be referenced from the views they constrain, and vice versa". Currently only sub-docs and modules/ cite ADRs. This leaves a reader entering via `03-development-view.md` unaware that its variant matrix is ADR-011, or a reader in `04-process-view.md` ignorant of ADR-010 / ADR-012 / ADR-013. G5 (justify decisions) is weakened when the rationale is two hops away from the view that encodes the decision.
- **Edit sketch:**
  - Files: `03-development-view.md`, `04-process-view.md`, `05-physical-view.md`, `06-scenario-view.md`
  - Location: Either a new paragraph at end of each file ("Related ADRs") or inline citations next to the structures they govern (preferred where the decision is local — e.g., inline at the variant matrix `03-development-view.md:235-241` cite ADR-011; at the scheduler event-loop section of 04 cite ADR-010).
  - Delta: 1–3 `[ADR-NNN](08-design-decisions.md#adr-NNN-...)` links per view, with a one-line justification each.
- **Trade-offs:** Gain V2 traceability and rubric compliance; give up slight increase in view-prose length and a maintenance coupling between views and the ADR index (mitigated by stable ADR IDs).
- **Sanity test:** `rg -c "ADR-" docs/pypto-runtime-design/0{3,4,5,6}-*.md` returns non-zero for each of the four files.

#### A4-P7: Unify L0 component label ("Scheduler: AicoreDispatcher")

- **Rationale:** V5 (diagram consistency) and D7. Physical View calls L0 a "Dispatcher", Development View ships it as an `ISchedulerLayer` implementation named `aicore_dispatcher`, and Logical View treats it as a Scheduler. A single reading pass hits three names for one concept.
- **Edit sketch:**
  - File: `docs/pypto-runtime-design/05-physical-view.md`
  - Location: the Single-Node Deployment diagram (lines 35-40, "`Core` (ordinal 0, leaf)"), and anywhere else the term "Dispatcher" appears at L0
  - Delta: Replace `│ Dispatcher: AICore dispatch (register poll)  │` with `│ Scheduler: AicoreDispatcher (register poll)  │`; leave Development View file name `aicore_dispatcher.cpp` alone (filename reflects implementation).
- **Trade-offs:** Gain one unambiguous term for an `ISchedulerLayer` instance; give up the "Dispatcher" color-word, which was informal.
- **Sanity test:** `rg -n "Dispatcher:" docs/pypto-runtime-design/` returns zero hits; L0 text reads as "Scheduler: AicoreDispatcher" everywhere.

#### A4-P8: Sync Appendix-B TaskState count

- **Rationale:** D7. `appendix-b-codebase-mapping.md:42-43` advertises "(10 states)" but `modules/core.md:156-167` lists 10 lifecycle values + `FAILED` (11). Reviewers doing migration planning will undercount.
- **Edit sketch:**
  - File: `docs/pypto-runtime-design/appendix-b-codebase-mapping.md`
  - Location: lines 42-43 (TaskState Host / Device rows, target column)
  - Delta: `core/task.h (10 states)` → `core/task.h (10 lifecycle states + FAILED; see modules/core.md §2.3)`.
- **Trade-offs:** Negligible; keeps the table column readable while removing the drift.
- **Sanity test:** Appendix B cell references the exact enum names in `modules/core.md:156-167`.

#### A4-P9: Expand `Task State` glossary entry

- **Rationale:** D7. The glossary currently collapses the full 10-state lifecycle to "FREE → SUBMITTED → ... → RETIRED", obscuring `PENDING`, `DEP_READY`, `RESOURCE_READY`, `DISPATCHED`, `EXECUTING`, `COMPLETING` — all of which are referred to as formal terms in ADR-010 / `04-process-view.md:229-285` / `modules/core.md:156-167`.
- **Edit sketch:**
  - File: `docs/pypto-runtime-design/appendix-a-glossary.md`
  - Location: `Task State` entry (line 88)
  - Delta: "Lifecycle phase (FREE → SUBMITTED → ... → RETIRED)" → "Lifecycle phase: `FREE → SUBMITTED → PENDING → DEP_READY → RESOURCE_READY → DISPATCHED → EXECUTING → COMPLETING → COMPLETED → RETIRED` (plus `FAILED`). See [modules/core.md §2.3](modules/core.md#23-task-taskkey-taskhandle-taskstate)."
- **Trade-offs:** Gain closed-form enumeration; give up ~3 lines of table real estate.
- **Sanity test:** Glossary `Task State` row enumerates all 10 states plus `FAILED` and matches the enum in `modules/core.md:156-167` exactly.

---

## 5. Revisions of own proposals (round ≥ 2 only)

*Not applicable — round 1.*

## 6. Votes on peer proposals (round ≥ 2 only)

*Not applicable — round 1.*

## 7. Cross-aspect tensions

| pair | with_proposal | preferred_resolution |
|------|---------------|----------------------|
| A2 vs A9 | A4-P5 (add three glossary entries) | Two-tier prose: glossary row states purpose + source anchor only; deep semantics remain in `09-interfaces.md §2.6.4`. No new abstraction introduced; only vocabulary — neutralizes A9/YAGNI challenge while satisfying A2/E6 that formal API types are named. |
| A7 vs A9 | A4-P7 (rename L0 "Dispatcher" → "Scheduler: AicoreDispatcher") | The L0 implementation *is* a fine-grained `ISchedulerLayer` (A7 concern). A9 may push back on extra prose. Resolution: one-word label change per view — does not add modules or ceremony; purely aligns terminology. |
| A1 vs A4 (general doc-fix category) | All A4 proposals | All A4 proposals above are pure prose / anchor edits. None allocates, blocks, relayouts data, or extends any latency on the hot path. `hot_path_impact: none` is attested per proposal. No two-tier path construction is required; A4 does not propose runtime-behavioral changes. |
| A4 vs A6 (if security-relevant terms added later) | None in round 1 | N/A now; flagged for round 2 should any A6 proposal introduce new formal types (e.g., `ITrustBoundary`, `IAuditSink`) that would require glossary additions. |

## 8. Stress-attack on emerging consensus (round 3+ only)

*Not applicable — round 1.*

## 9. Status

- **Satisfied with current design?** Partially. The design package is structurally complete (all 4+1 views present, glossary extensive, module docs drafted, ADR set populated, known deviations enumerated). Document-consistency defects are localized (one broken anchor, one prose count error, one out-of-order section, one HAL casing discrepancy, three missing glossary terms, one cross-view naming mismatch, and the systemic absence of ADR back-references from top-level views). None are structural; all are minimum-edit fixes per A4-P1…A4-P9.
- **Open items expected in next round:**
  - A4-P1 (HAL enum casing) — high severity, expect A1 to weigh in only to confirm `hot_path_impact: none`.
  - A4-P2 (broken anchor) — blocker-severity doc fix; trivial convergence expected.
  - A4-P3 (invariant count) — expect authorial clarification on whether I3 is a true third invariant (reviewer's read) or mis-filed content.
  - A4-P6 (ADR back-references from views) — medium; expect debate with A9 over how much prose to add; A4 will defend minimal inline cites.
  - A4-P7 (L0 naming) — expect A7/A9 discussion on whether "Dispatcher" is informal enough to keep or should be formalized.
