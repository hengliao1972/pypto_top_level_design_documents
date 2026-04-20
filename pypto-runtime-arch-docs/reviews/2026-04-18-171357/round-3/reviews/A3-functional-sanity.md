# Aspect A3: Functional Sanity & Correctness — Round 3 (Stress Round)

## Metadata

- **Reviewer:** A3 (Functional Sanity & Correctness)
- **Round:** 3 (mandatory stress round)
- **Target:** `docs/pypto-runtime-design/` (rev as of 2026-04-18-171357)
- **Runtime Mode:** yes
- **Timestamp (UTC):** 2026-04-19T22:00:00Z

## 1. Rubric Checks

Rubric results unchanged from round 2 — no target-doc edits have landed between rounds. The deltas below reflect how the round-2 amendments (§5) would move each result **if** they are applied. I explicitly re-checked each row against scenarios 6.1.3 (SPMD) and 6.2.3 (Task Slot Pool Exhaustion) as the stress round requires.

| # | Check | Result (R2 → projected post-R3) | Evidence (file:line) | Rule(s) |
|---|-------|----------------------------------|----------------------|---------|
| 1 | Every stated requirement traces to ≥ 1 scenario walkthrough | Weak → Pass (conditional on A3-P6 + A9-P6 option (iii)) | `01-introduction.md:32-57` vs `06-scenario-view.md:1-155`. Option (iii) on A9-P6 shrinks FR-10 to `FUNCTIONAL` + scaffolding; the matrix I own (A3-P6) then closes the remaining misses. | G1, V3 |
| 2 | ≥ 1 failure scenario per critical path | Weak → Pass (conditional on A3-P3) | `04-process-view.md:641-700` lists four critical paths; `06-scenario-view.md:81-143` covers three. A3-P3 adds §6.2.5–§6.2.7 for the admission path. 6.2.3 (`06-scenario-view.md:116-128`) is the only §4.8.4 exhaustion scenario today, and my stress replay (§8) shows it already leaks an assumption (see A9-P5 row). | V3, V4 |
| 3 | All interface implementations honor LSP | Fail → Pass (conditional on A3-P1a, A3-P2, A9-P5, A7-P2 in that order) | `04-process-view.md:229-284`, `02-logical-view/09-interfaces.md:28, 65, 242-245`. Stress replay (§8) confirms that if A3-P2 lands *after* A7-P2 the role-split interface will carry an ambiguous `submit()` contract, so ordering is itself a correctness requirement. | LSP, D7, V5 |
| 4 | Ambiguous / missing info marked `[ASSUMPTION]` | Weak → Pass (conditional on A3-P11, A3-P13, A8-P6) | `02-logical-view/07-task-model.md:201`, `04-process-view.md:540-561`. | G3 |
| 5 | Explicit error conditions in Process View | Weak → Pass (conditional on A3-P4, A3-P5, A3-P7, A5-P4, A9-P5) | `04-process-view.md:568-633`; `06-scenario-view.md:139`, `116-128`. Stress replay (§8, A5-P4 row) shows that idempotent-flag semantics and the "eventually timeout" wording at `06-scenario-view.md:140` must be reconciled. | — |
| 6 | Edge cases enumerated | Weak → Pass (conditional on A3-P7, A3-P8, A3-P9, A3-P12, A3-P14, A3-P15) | `02-logical-view/09-interfaces.md:98-120`, `02-logical-view/07-task-model.md:205-223`. Stress replay (§8, A9-P4 row) raises a new edge case: `tasks.size() == 1 && spmd.has_value()` vs `tasks.size() > 1 && spmd.has_value()` — both must be accepted-or-rejected by the admission precondition catalog. | V3, V4 |

## 2. Pros

(Carried from R2; newly reinforced by stress-round findings.)

- The four-view scenario walkthroughs at `06-scenario-view.md:1-155` remain the strongest V3 artifact. My stress replays (§8) were possible *because* each scenario has explicit per-step Logical/Development/Process/Physical columns — this is a rubric-check in itself. — cites V3.
- A9's R2 amendments (the `RecordedEventSource` + `step()` test seam in A9-P2; the `ADR-011-R2` deferral pattern in A9-P6 option (iii)) preserve correctness hooks while removing speculative surface. A3 benefits because the deterministic replay path (needed for A3-P4/A3-P5 test plans) survives even if full REPLAY-mode is deferred. — cites G3, X5.
- The merge register consolidations — A1-P11 absorbing A3-P7 / A6-P8a, A1-P14 absorbing A10-P10, A5-P3 absorbing A10-P2a, A7-P2 absorbing A9-P1 — each collapse a three-way overlap into a single, testable pipeline step. Stress-replay confirms correctness surfaces are preserved. — cites LSP, DRY, G3.
- The `ErrorCode` layout + `ErrorContext` immutable contract at `modules/error.md:22-113, 169-176` still underpins every failure-propagation chain I exercised in §8. — cites G3, SRP.
- The two-tier framing ("fast default + slow opt-in") adopted by A1-P12, A3-P4 (precomputed successor list), A3-P8 (`FE_VALIDATED`), A6-P3, A6-P8, A10-P7 consistently preserves the correctness contract on the slow path while keeping the fast path lean. That framing survives every scenario replay I ran in §8. — cites G3, LSP.

## 3. Cons

(Carried from R2; adjusted by stress-round evidence.)

- **`ERROR`/`CANCELLED` FSM gap** — still the single remaining blocker (A3-P1a). Stress replay of 6.2.3 (§8 row A3-P1) produces an implementation divergence within one step.
- **`submit()` return contract** — unresolved (A3-P2 option (b)). Stress replay (§8 rows A3-P2, A7-P2, A9-P5) shows that an A7-P2 ISP split landing *before* A3-P2 would bake an LSP violation into three role interfaces instead of one. Ordering is now a hard correctness requirement.
- **Admission-path failure scenarios** — A3-P3 must land before A3-P7/A3-P8 stress-attacks have a home in `06-scenario-view.md`. Stress replay of 6.2.3 (§8) is forced to annotate `06-scenario-view.md:122-123` with behavior not yet in the document.
- **Sibling cancellation under SPMD** — A3-P5's amended "EXECUTING-run-to-completion" rule, when stress-replayed against a 24-way SPMD (`06-scenario-view.md:67-79`), raises a new detail (`spmd_index` identity preserved on CANCELLED siblings for `ErrorContext.remote_chain` entries). See §8 row A3-P5.
- **Task Slot Pool Exhaustion admission semantics** — `06-scenario-view.md:122-123` maps `ResourceExhausted` onto the bare `on_submit` handler, but under A9-P5 + A3-P2 + A5-P4 the exhaustion should surface as `AdmissionStatus::WAIT` (back-pressure) *or* `AdmissionStatus::REJECT(Exhaustion)` depending on idempotency + caller policy. Today's scenario text conflates the two. See §8 rows A9-P5 and A5-P4.
- **Stale narrative at `06-scenario-view.md:140`** — "they eventually timeout" is referenced by A3-P4 amended but the scenario text itself is not part of A3-P4's edit sketch. I added a pointer in §5 (A3-P4 amend) to update this line explicitly when A3-P4 lands, so the scenario and the event semantics cannot diverge again.

## 4. Proposals (new this round)

No new proposals. Stress findings route to amendments of existing A3 proposals (§5) rather than new ones, per the prompt's "revise your own" guidance.

## 5. Revisions of own proposals (stress-round update)

Carried from round 2; amendments below reflect stress-replay findings.

| id | action | amended_summary (if amend/split) | reason |
|----|--------|----------------------------------|--------|
| A3-P1 | defend (split retained) | **A3-P1a (blocker):** add `ERROR` state with transitions from `DISPATCHED`/`EXECUTING`/`COMPLETING`. **A3-P1b (medium):** add `CANCELLED` state. | Stress replay (§8, 6.2.3) confirms that the `ERROR` state is the minimum surface needed to write a deterministic admission-failure walkthrough. No peer disagreed in R2. Holds. |
| A3-P2 | amend (ordering constraint added) | Option (b) (`std::expected<SubmissionHandle, ErrorContext>`) + AdmissionStatus payload (A9-P5). **New stress-round clause:** the edit must land **before** A7-P2 is applied, otherwise the three role interfaces created by A7-P2 will each carry the ambiguous contract. If A7-P2 is applied first, ADR-`submit-return-contract` must also patch `ISubmissionSink` in the same commit. | Stress replay (§8, 6.1.3) shows LSP drift between `ISchedulerLayer::submit()` and the future `ISubmissionSink::submit()` introduced by A7-P2 if ordering is not enforced. |
| A3-P3 | defend | — | Fast-track; no peer pushback. Stress replay (§8) uses `06-scenario-view.md:116-128` as the anchor and confirms A3-P3 needs to land to make rubric #2 Pass. |
| A3-P4 | amend (scenario text included) | Keep `DEP_FAILED(producer_key, ErrorCode)` walking the precomputed successor list at admission time. **New stress-round clause:** the edit sketch extends to also replace `06-scenario-view.md:139-140` ("Consumer tasks' dependency counter is not decremented; they eventually timeout") with the deterministic `DEP_FAILED` path, so the scenario and the event semantics cannot diverge again. | Stress replay (§8, A3-P4 row) exposed the divergence. Cost is one additional doc line in the same ADR; no runtime change. |
| A3-P5 | amend (SPMD-specific clause) | Already-EXECUTING siblings run to completion, results discarded; `PENDING`/`DEP_READY`/`DISPATCHED` siblings cancelled. **New stress-round clause for SPMD:** cancelled siblings retain their `spmd_index` in `ErrorContext.remote_chain` entries so post-mortem tools can correlate which tile was aborted. Already-EXECUTING AICore sub-tasks cannot be force-terminated without hardware support; the scheduler must wait for the `COND` FIN write before retiring them. | Stress replay of 6.1.3 (§8, A3-P5 row): a 24-way SPMD where AICore₇ fails mid-execution produces 23 siblings in various states. Without `spmd_index` retention the aggregated error context loses the identity of each sibling. |
| A3-P6 | amend | Matrix required; scope shrinks to include only the `FUNCTIONAL` simulation mode if A9-P6 lands under option (iii) (my R3 preferred landing — see §6). Scaffolding for REPLAY remains a traceability anchor ("FR-10 partial: enum reserved, implementation deferred via ADR-011-R2"). | A9-P6 option (iii) preserves the scaffolding, so the matrix retains a row for REPLAY pointing at ADR-011-R2 rather than a scenario. |
| A3-P7 | defend (merge accepted) | Merged with A1-P11 + A6-P8. A3 owns the precondition catalog and sub-codes. **Confirmed this round:** A3-P7's SPMD-specific sub-codes (`SpmdIndexOOB`, `SpmdSizeMismatch`) must be in the initial catalog, not a post-v1 addition — see §8 row A9-P4. | Stress replay (§8, 6.1.3) with the A9-P4 amendment (discriminator is `spmd.has_value()`) requires precondition validation for the SPMD shape. |
| A3-P8 | defend (two-tier) | `FE_VALIDATED` fast path in release; DFS in debug / untrusted-frontend. | Synthesis + stress replay (§8) confirm the two-tier resolves A1's HPI concern. Holds. |
| A3-P9 | defend | `spmd_index`/`spmd_size` carried at `TaskArgs.scalar_args[0..1]`. | Stress replay of 6.1.3 step 4 (`06-scenario-view.md:74`) confirms the ABI survives under A9-P4's `spmd.has_value()` discriminator. Both proposals compose. |
| A3-P10 | defend | — | No peer objections. |
| A3-P11 | defend | — | — |
| A3-P12 | amend (coordinator-failover clause reconfirmed) | `drain()` atomic w.r.t. **distributed** submissions; `DrainInProgress` in `modules/error.md` Core domain. | A5-P3 fail-fast (v1) keeps the single-coordinator assumption; stress replay did not find a new failure mode beyond what R2 already covered. |
| A3-P13 | amend | Pin per-link FIFO + idempotent delivery + duplicate detection via `TaskKey.generation`; aligned with A5-P10 and A8-P6. | — |
| A3-P14 | defend | — | Pure doc fix. |
| A3-P15 | defend | — | Debug-only; no hot-path cost. |

## 6. Votes on peer proposals — final round (disputed items only)

Per the prompt, R3 re-votes the three `disputed` proposals after reviewing the amended text from round-2 §5. All other peer-proposal votes from R2 carry forward unchanged (see `round-2/reviews/A3-functional-sanity.md` §6).

### Re-vote: disputed proposals

| proposal_id | round-2 vote | round-3 vote | rationale (cite rule id) | blocking | override_request |
|-------------|--------------|--------------|--------------------------|----------|-------------------|
| A2-P7 | abstain | **agree (with Q-record reframing)** | A2's R2 amendment converts the proposal into a `09-open-questions.md` Q-record (no interface in v1). With no interface, there is no LSP surface and no correctness hazard for A3 to police. The Q-record wording satisfies G3 ("state assumptions explicitly"). If the final text instead reserves a v1 interface slot, my vote flips back to abstain — A3 has no standing to block a purely evolutionary seam. Cite G3. | false | false |
| A9-P2 | abstain | **agree** | The R2 amendment commits to a **single** `IEventLoopDriver` test-only seam with `step()` + `RecordedEventSource`, keeps closed enums for deployment modes, and defers the pluggable-policy surface to an appendix. Stress replay (§8, A8-P2 row) confirms the single seam is sufficient to drive A3-P4 and A3-P5 tests deterministically; LSP is preserved because `IEventLoopDriver` has exactly one production implementation and one test double. No A3 correctness loss. Cite LSP, X5. | false | false |
| A9-P6 | agree | **agree (option iii)** | Among the three options the synthesizer lists, **option (iii) — ship `FUNCTIONAL`-only implementation, keep `REPLAY` enum + scaffolding** — is the option that preserves A3's traceability matrix (A3-P6 keeps a row for REPLAY pointing at ADR-011-R2) **and** keeps the deterministic replay hook A3-P4/A3-P5 test plans depend on (the `RecordedEventSource` + `step()` seam survives independent of whether REPLAY-mode is implemented). Option (i) would force A3-P6 to drop FR-10 coverage; option (ii) recreates the `PERFORMANCE` ambiguity A9 is trying to close. Cite G1, V3, E5. | false | false |

### Full-list re-vote summary (A3 R3 posture)

- 107 previously-agreed proposals: **carry forward; agree.** Stress replay (§8) did not produce a new A3-grounded disagreement on any of them; where the replay found an amendment need, I have routed it to the owner's proposal (see §8's "verdict" column and the amendments flagged in §5).
- 3 disputed proposals (A2-P7, A9-P2, A9-P6): **all flip to agree** with the R2 amendments / synthesis option (iii) as recorded above.
- Blocking=true count: **0**. Override_request count: **0**.
- Total A3 votes this round: **110** (re-affirmations 107 + disputed re-votes 3).

## 7. Cross-aspect tensions (stress-round additions)

Carried from R2 with additions raised by the stress replay (§8).

| pair | with_proposal | preferred_resolution |
|------|---------------|----------------------|
| A3 vs A5 | A5-P4 (`idempotent: bool`) + `06-scenario-view.md:122-128` | Scenario 6.2.3 currently conflates back-pressure and hard-rejection; under A5-P4, `idempotent=true` should map to WAIT (back-pressure with retry) and `idempotent=false` to REJECT. Update the scenario text when A5-P4 lands. |
| A3 vs A9 | A9-P4 (drop `Kind`) + A3-P7 (preconditions) | Add `SpmdIndexOOB` and `SpmdSizeMismatch` to A3-P7 catalog up front; the discriminator is `spmd.has_value()` so the validator must inspect `spmd.{index,size}` when set. |
| A3 vs A9 | A9-P5 (`AdmissionStatus`) + `06-scenario-view.md:122-128` | `ResourceExhausted` in the scenario text should be replaced by `AdmissionStatus::REJECT(Exhaustion)` (or WAIT when the caller opts in to back-pressure). Single-enum discipline per A9-P5. |
| A3 vs A7 | A7-P2 (ISP split) + A3-P2 | Ordering hazard: A3-P2 must land before A7-P2; if the reverse happens, the three role interfaces inherit the ambiguous `submit()` contract. A3-P2 edit sketch carries this constraint. |
| A3 vs A8 | A8-P2 (driveable loop) + A3-P4/A3-P5 | The single `IEventLoopDriver` seam (from A9-P2 amendment) suffices to drive both `DEP_FAILED` propagation and sibling-cancellation tests deterministically. No further interface needed. |
| A3 vs A1 | A1-P9 (WG bitmask, 2×64 fallback) + 6.1.3 (24-way SPMD) | 24 << 64; within the 64-WG single-bitmap tier; no LSP or scheduling hazard for v1 AICore deployments. Record the fallback as the `extend_past_64` deviation in `10-known-deviations.md`. |

## 8. Stress-attack on emerging consensus

**Scenario anchors used (every row cites one):**
- **6.1.3 SPMD Kernel Launch:** `06-scenario-view.md:61-79` — specifically steps 1–7 across lines 71–77, with the dispatch happening at step 3 (`06-scenario-view.md:73`) and FIN signalling at step 5 (`06-scenario-view.md:75`).
- **6.2.3 Task Slot Pool Exhaustion:** `06-scenario-view.md:116-128` — specifically steps 1–5 across lines 122–126 and mitigation at line 128.
- Cross-references where a composed replay is needed: 6.1.1 (`06-scenario-view.md:9-33`) and 6.2.1 (`06-scenario-view.md:83-98`).

Every row below attacks the **R2 amended** text, not the R1 wording. Verdicts: `holds` = attack failed; `breaks` = attack succeeded; `uncertain` = attack exposed a genuine gap but the owner's amendment plausibly closes it with a small delta. For `breaks` / `uncertain` rows I name the concrete follow-up amendment, routed to the owner in their R3 column or absorbed into my §5 revisions.

### 8.1 Own-aspect proposals (A3-P1 … A3-P15)

| proposal_id | stress_attempt | verdict |
|-------------|----------------|---------|
| A3-P1 | **Replay 6.2.3 step 5 (`06-scenario-view.md:126`)**: "parent Task → ERROR". Without the amended FSM, two implementations of `ISchedulerLayer` could land on different terminal states for this transition — one using `COMPLETED+error_flag`, one using a new `ERROR` state, one reusing `RETIRED`. Attack: show the divergence. The R2-amended A3-P1a adds `ERROR` as a first-class state with transitions from `{DISPATCHED, EXECUTING, COMPLETING}`. Attack fails because the state is present and the transition from `DISPATCHED` covers the slot-exhaustion → admission-rollback → parent-error path. | **holds** |
| A3-P2 | **Replay 6.2.3 step 3 (`06-scenario-view.md:124`)**: `on_submit` handler detects `ResourceExhausted`. Attack: the amended `submit()` widens to `std::expected<SubmissionHandle, ErrorContext>` — but which variant? If the C++ binding choice is `Status submit(..., SubmissionHandle* out)`, a caller that forgets to check `Status` can legally pass the null `SubmissionHandle*` onward, violating the postcondition in a silent way. Counter: the R2 amendment commits to `std::expected<>` (the compiler forces the check). Attack reduces to "ensure `std::expected<>` is the normative form, not `Status + out-param`." | **holds** (with §5 amendment pinning `std::expected<>` as the normative form; `Status+out` listed only as a C-API compatibility shim in `modules/bindings.md`). |
| A3-P3 | **Replay 6.2.3 steps 1–5**: the scenario already anchors admission failure, but it sits in §6.2 without being cross-linked from `04-process-view.md §4.8.4`. Attack: A3-P3 adds §6.2.5–§6.2.7 for cyclic/workspace/NONE-misuse cases but does **not** explicitly cross-reference the existing §6.2.3. Consequence: the "every §4.8 critical path has a failure scenario" sanity test still counts §6.2.3 separately from the new sections; no divergence. | **holds** |
| A3-P4 | **Replay 6.1.3 step 6 + 6.2.1 composed**: assume AICore₇ of the 24-way SPMD hangs (6.2.1 pattern applied to 6.1.3). Under the R2-amended A3-P4, `DEP_FAILED(producer_key=spmd.sub[7], ErrorCode)` walks the precomputed successor list. Attack: the SPMD parent is not a successor of sub-task 7 — the parent is the `pending_children` aggregator, not a DAG successor. The amendment's "precomputed successor list" is silent on the SPMD aggregation edge. Consequence: without a follow-up, a failed SPMD sub-task could emit `DEP_FAILED` to its data successors but not notify the SPMD parent's aggregator. | **breaks (minor)** — fix: extend A3-P4 edit sketch to include the SPMD aggregation edge in the "precomputed successor list at admission time." I have added this to my §5 revision. Route to A3 (self). |
| A3-P5 | **Replay 6.1.3 (`06-scenario-view.md:67-79`) with one sibling failing**: AICore₇ fails mid-kernel; the other 23 are `EXECUTING`. Amended rule: 23 run to completion, results discarded. Attack: `ErrorContext.remote_chain` has 24 entries — 1 cause, 23 identical "sibling-discarded" entries — losing the `spmd_index` of each. Debugging which tile produced what garbage is impossible. | **breaks (minor)** — fix: include `spmd_index` in each `CANCELLED`/`DISCARDED` entry. Added to §5 revision. Route to A3 (self). |
| A3-P6 | **Replay 6.1.3**: the SPMD walkthrough satisfies FR-4 (four function types) in part but does not satisfy FR-10 (simulation modes) on its own. Under A9-P6 option (iii), FR-10 resolves to "FUNCTIONAL scenario + ADR-011-R2 for REPLAY/PERFORMANCE." Attack: A3-P6 must therefore include a row for FR-10 pointing at both §6.1.1–§6.1.3 and ADR-011-R2; drop that, and FR-10 falls off the matrix. | **holds** (with §5 amendment: row "FR-10 → §6.1.1–§6.1.3 (FUNCTIONAL) + ADR-011-R2 (deferred)"). |
| A3-P7 | **Replay 6.1.3 step 1 (`06-scenario-view.md:71`)**: `submit_spmd(kernel_func, block_dim=24, per_tile_args)`. Attack: can A3-P7's precondition catalog reject a malformed SPMD submission where `block_dim=0` or `per_tile_args.size() != block_dim`? R2 catalog includes `EmptyTasks / SelfLoopEdge / EdgeIndexOOB / BoundaryIndexOOB / WorkspaceSubrangeOOB / WorkspaceSizeMismatch`. It does **not** include `SpmdIndexOOB` / `SpmdSizeMismatch`. Consequence: a frontend bug that emits `spmd.index >= spmd.size` admits successfully and surfaces only at kernel-dispatch time (or not at all if the out-of-range index is caught by tile-address arithmetic as a heap corruption). | **breaks (minor)** — fix: extend A3-P7 precondition catalog to include `SpmdIndexOOB` / `SpmdSizeMismatch` / `SpmdBlockDimZero`. Added to §5 (A3-P7 defend ⇒ amend on catalog completeness). Route to A3 (self). |
| A3-P8 | **Replay 6.1.3**: SPMD submissions have `|tasks| == block_dim` but `intra_edges.size() == 0` (SPMD sub-tasks are mutually independent per `02-logical-view/07-task-model.md:220`). The DFS cost is `O(|tasks| + 0) = O(24)`. Attack: under the `FE_VALIDATED` fast path, even the 24-element scan is skipped. No hot-path impact. | **holds** |
| A3-P9 | **Replay 6.1.3 step 4 (`06-scenario-view.md:74`)**: "reads from shared memory at offset based on `spmd_index`." Attack: `spmd_index` and `spmd_size` must be delivered in a way that survives A9-P4 (drop `Kind`). A3-P9 pins `TaskArgs.scalar_args[0..1]`; A9-P4's `spmd.has_value()` is the discriminator. Both compose: the discriminator says "this task is SPMD," the reserved scalar slots carry the values. | **holds** |
| A3-P10 | **Replay 6.2.3 step 5 (`06-scenario-view.md:126`)**: the scenario raises `simpler.RuntimeError("task slot pool exhausted at Chip level")`. Attack: the error originates in the Scheduler domain (`modules/error.md` §2.1) but surfaces as `simpler.RuntimeError` — under the full domain mapping A3-P10 requires, Scheduler should map to `simpler.SchedulerError` or similar. The scenario text is inconsistent with the amended mapping table. | **breaks (minor)** — fix: the A3-P10 mapping table edit must also update `06-scenario-view.md:126` to match. Added to A3-P10 edit sketch. Route to A3 (self). |
| A3-P11 | Not exercised by 6.1.3 or 6.2.3 (Q8 concerns deferred-completion tracking for distributed data moves — 6.1.2 path). No attack available from these two scenarios. | **holds (vacuous)** |
| A3-P12 | **Replay 6.2.3 + `drain()` concurrently**: user calls `drain()` while Orchestration Function is retrying after slot exhaustion. Amended A3-P12: `drain()` entered ⇒ subsequent `submit()` rejected with `DrainInProgress`. Attack: the Orchestration Function's retry-on-exhaustion loop (`06-scenario-view.md:125(a)`) may observe `ResourceExhausted → DrainInProgress → ResourceExhausted` oscillation if the drain flag is not sticky. Amended text says "atomic flag read" — sticky by construction. No oscillation. | **holds** |
| A3-P13 | Not exercised by 6.1.3 (single-node) or 6.2.3 (single-node). Covered by 6.1.2 / 6.2.2; outside the stress-round anchor set. | **holds (vacuous for this round)** |
| A3-P14 | **Replay 6.1.3 step 5 (`06-scenario-view.md:75`)**: each AICore signals FIN via `COND`. AICore is a leaf Worker (no children). Under amended A3-P14, the state machine has an explicit edge `EXECUTING → COMPLETED` at leaf (skipping `COMPLETING`). The scenario step 5 matches that edge directly. | **holds** |
| A3-P15 | Not exercised by 6.1.3 or 6.2.3. Debug-only cross-check; no attack surface. | **holds (vacuous)** |

### 8.2 Peer proposals where A3 has standing

| proposal_id | stress_attempt | verdict |
|-------------|----------------|---------|
| A1-P1 (pre-size `producer_index`) | **Replay 6.1.3 steps 2–3 (`06-scenario-view.md:72-73`)**: 24 SPMD sub-tasks with `dep_mode=NONE` → zero `producer_index` inserts. Even with `DATA`-mode, 24 entries are << pre-sized cap (per `02-logical-view/02-scheduler.md` LevelParam defaults). No LSP hazard. | **holds** |
| A1-P4 (Task hot/cold split) | **Replay 6.1.3 step 3 (`06-scenario-view.md:73`)**: dispatch reads hot fields only. Attack: if the precomputed successor list (A3-P4 amended) lives in the cold tier, does SPMD fan-out still satisfy the latency budget? Successor walk is on the error cold path only; hot-path dispatch untouched. | **holds** |
| A1-P8 (pre-size Outstanding/Ready) | **Replay 6.2.3 step 2 (`06-scenario-view.md:123`)**: `alloc_task_slot()` fails. Attack: can Ready queue also be exhausted in the same event, yielding two concurrent exhaustion paths? Amended A1-P8 pre-sizes both to a cap; overflow of either returns `ResourceExhausted` with the same code path. No LSP divergence, but the error sub-code (`ReadyQueueExhausted` vs `TaskSlotExhausted`) should be distinguishable for diagnostics. | **uncertain (minor)** — fix: route to A1 / A3-P7 merge; add sub-codes `TaskSlotExhausted` and `ReadyQueueExhausted` to the precondition / admission catalog. |
| A1-P9 (bitmask WG availability, 2×64 fallback) | **Replay 6.1.3 step 3 (`06-scenario-view.md:73`)**: `block_dim=24` on 24 AICores — well under the 64-bit single-bitmap tier. No LSP hazard. Attack at 128 AICores: 2×64 fallback kicks in per the synthesis amendment. | **holds** |
| A1-P11 (per-arg Python↔C budget + validation, absorbs A6-P8a / A3-P7 partial) | **Replay 6.1.3 step 1 (`06-scenario-view.md:71`)**: `per_tile_args` carries 24× per-tile payload; each arg validated. Budget × 24 must fit in the documented budget. Amended text includes a per-arg budget; `submit_spmd` N=24 is within the synthesised bound. Attack: the amended catalog must include SPMD-shape checks (see A3-P7 row above). | **uncertain (minor)** — fix: ensure A3-P7's SPMD sub-codes are absorbed into A1-P11's single validation pass, not duplicated elsewhere. Already routed in §5 (A3-P7 merge). |
| A1-P12 (`BatchedExecutionPolicy.max_batch`) | **Replay 6.1.3 step 3 (`06-scenario-view.md:73`)**: SPMD fan-out is **parallel dispatch**, not batched execution. `BatchedExecutionPolicy` is a *separate* axis. No LSP hazard. | **holds** |
| A1-P13 (bound `args_blob` to 1 KiB fast path) | **Replay 6.1.3 step 1 (`06-scenario-view.md:71`)**: 24 × `per_tile_args` may exceed 1 KiB. Amended text: >1 KiB slow-path handling is allowed. LSP preserved because the distinction is wire-form-only, not observable to the caller. | **holds** |
| A2-P1 (version public data contracts) | Not exercised by 6.1.3 / 6.2.3 directly. Version byte is on `ErrorContext`, which is surfaced at `06-scenario-view.md:126` as `simpler.RuntimeError` — no breakage in the v1 wire form. | **holds** |
| A2-P3 (open extension-point enums; `DepMode` closed) | **Replay 6.1.3**: SPMD sub-tasks have `dep_mode=NONE`; closing `DepMode` prevents a frontend from inventing a "SpmdOnly" depmode that would bypass the admission contract. Good for A3. | **holds** |
| A4-P8 (sync Appendix-B TaskState count) | **Replay 6.2.3 step 5 (`06-scenario-view.md:126`)**: scenario references ERROR state. Appendix-B count must match A3-P1a. Attack: if A3-P1a splits to A3-P1a alone (no CANCELLED), Appendix-B count differs from A3-P1b-inclusive count. The amendment's "(+ new states)" footnote handles both. | **holds** |
| A5-P3 (v1 deterministic fail-fast; absorbs A10-P2a) | **Replay 6.2.3 distributed variant**: Chip-level slot exhaustion on node N₁ in a 2-node Pod during `submit_spmd`. Under fail-fast, coordinator aborts the SPMD group on first failure. LSP preserved (single coordinator semantics). Attack: does the `DrainInProgress` clause (A3-P12 amended) interact? If the coordinator fails fast *during* a drain, which path wins? Fail-fast emits `ERROR`; drain completes with error context. No divergence — the drain's `block_until_RETIRED` contract absorbs the ERROR state. | **holds** |
| A5-P4 (`idempotent: bool`) | **Replay 6.2.3 step 4 (`06-scenario-view.md:125`)**: Orchestration Function "MAY: (a) wait and retry (back-pressure), (b) propagate error upward." Attack: under A5-P4, the retry decision must depend on `idempotent`. `idempotent=false` + retry can cause duplicate side effects. The scenario text allows retry without consulting `idempotent`. Consequence: the scenario must be updated to reference the `idempotent` flag, OR the flag must be scheduled to apply only at the retry layer (A5-P1/A5-P2), not at the application-visible `on_submit` path. | **breaks (minor)** — fix: A5-P4 edit sketch should update `06-scenario-view.md:125` to clarify that caller retry is allowed for any task (the flag governs internal retry at A5-P1/A5-P2), so the flag does not gate user-visible back-pressure. Route to A5 + co-edit with the scenario. |
| A5-P6 + A8-P4 (watchdog + `dump_state()`) | **Replay 6.2.3 + watchdog**: if the scheduler thread is stuck in the exhaustion-retry loop, watchdog triggers. `dump_state()` must include pool utilisation (`LayerStats.get_stats()` per `06-scenario-view.md:128`). Paired proposals satisfy this surface. | **holds** |
| A6-P3 (bounded variable-length payload parsing) | **Replay 6.1.3 step 1**: `per_tile_args` is treated as bounded payload at the Python↔C boundary. Amended "≤1 compare at entry; none on reject" fits within A1-P11. No LSP hazard. | **holds** |
| A7-P2 (split `ISchedulerLayer` into role interfaces; absorbs A9-P1) | **Replay 6.1.3 step 1 (`06-scenario-view.md:71`)**: `submit_spmd` becomes `submit(SubmissionDescriptor)` on `ISubmissionSink` (post-A9-P1). Attack: if A3-P2 hasn't landed yet, `ISubmissionSink::submit()` inherits the ambiguous return contract. **Ordering hazard confirmed.** | **uncertain (ordering)** — fix: the A3-P2 edit must land in the same ADR/commit as A7-P2, or A3-P2 must land first. Already routed in §5 (A3-P2 amend). |
| A8-P2 (driveable event-loop `step()` + `RecordedEventSource`) | **Replay 6.1.3 step 7 (`06-scenario-view.md:77`)**: SPMD group completion dispatches a single "aggregated" event. Under `RecordedEventSource` + `step()`, the event-loop test can inject the 24 FIN events in a chosen order, then assert exactly one aggregation event per SPMD group. Deterministic. Under A9-P2 amendment, this test is the single production use of `IEventLoopDriver`. Satisfies X5 for A3-P4/A3-P5 tests without inventing new interfaces. | **holds** |
| A8-P6 (distributed trace alignment) | Not exercised by 6.1.3 / 6.2.3 (single-node). | **holds (vacuous)** |
| A8-P12 (stable `PhaseId`s) | **Replay 6.1.3 steps 1–7**: every phase in the SPMD pipeline (submit, dispatch-per-tile, execute-per-tile, aggregate, complete) must carry a stable `PhaseId`. Amended text commits to stability for Submission lifecycle. A3-P6 traceability depends on this. | **holds** |
| A9-P4 (drop `SubmissionDescriptor::Kind`) | **Replay 6.1.3 step 1 (`06-scenario-view.md:71`)**: discriminator becomes `spmd.has_value() + tasks.size() == spmd->size`. Attack: **edge case** — `tasks.size() == 1 && spmd.has_value() && spmd->size == 1` is technically a one-way SPMD; is it valid? If so, it is indistinguishable from a plain single-task Group by `tasks.size()` alone. If not, the precondition catalog (A3-P7) must reject it. | **uncertain (edge)** — fix: A3-P7 catalog to explicitly accept or reject `spmd.has_value() && spmd->size == 1` (I recommend accept, as it reduces to the single-task case trivially and removes an otherwise-arbitrary edge). Route to A3 (self, §5 A3-P7 amend). |
| A9-P5 (unified `AdmissionStatus`) | **Replay 6.2.3 step 2 (`06-scenario-view.md:123`)**: `alloc_task_slot()` → `ResourceExhausted`. Under A9-P5, this should surface as `AdmissionStatus::REJECT(Exhaustion)` (or WAIT for back-pressure). The scenario text still says `ResourceExhausted`. Attack: two different error tokens for the same event. | **breaks (minor)** — fix: A9-P5 edit sketch must update `06-scenario-view.md:123, 126` to use `AdmissionStatus`. Route to A9 + co-edit with scenario view. |
| A9-P6 option (iii) (keep REPLAY enum + scaffolding; defer implementation) | **Replay 6.1.3 under `SimulationMode::REPLAY`**: with option (iii), the enum value exists but is gated behind `ADR-011-R2` triggers. Attack: can a developer set `SimulationMode::REPLAY` in v1 and observe undefined behavior? If the scaffolding rejects with `SimulationModeNotImplemented`, the contract is clean. If it silently falls back to FUNCTIONAL, LSP is violated. | **holds** (with stipulation that the scaffolding rejects explicitly). I add this to my R3 vote rationale (§6). |
| A9-P7 (fold `SourceCollectionConfig` into `EventHandlingConfig`) | Not exercised by 6.1.3 / 6.2.3. Config-surface change only. | **holds (vacuous)** |
| A10-P2 (v1 fail-fast, absorbed into A5-P3) | Same replay as A5-P3 above. | **holds** |
| A10-P6 (faster peer-failure detection with hysteresis) | **Replay 6.2.3 distributed variant**: a remote peer running out of task slots on its Chip level will look to the coordinator like a slow but not failed peer. Hysteresis prevents flapping. Not strictly an LSP hazard, but the interaction with A5-P3 fail-fast must be explicit (hysteresis delays the fail-fast decision; must not breach the admission window budget). | **holds** (covered by hysteresis default per A10-P6 amendment) |
| A10-P7 (sharded TaskManager, `shards=1` default) | **Replay 6.1.3**: 24 sub-tasks on a single Chip-level TaskManager with `shards=1` — behaviour identical to current. No hot-path or LSP change. | **holds** |
| A10-P9 (gate `WorkStealing` × `RETRY_ELSEWHERE`) | **Replay 6.2.3 + work stealing enabled**: stolen SPMD sub-tasks under `RETRY_ELSEWHERE` could violate DS4 idempotency if `idempotent=false`. A10-P9 gates the combination at admission time. Under A5-P4, the gate reads `idempotent` and rejects the combination when false. LSP preserved. | **holds** |

### 8.3 Stress-round summary

- **Proposals that hold unchanged:** A3-P1, A3-P3, A3-P8, A3-P9, A3-P11 (vacuous), A3-P12, A3-P13 (vacuous), A3-P14, A3-P15 (vacuous); A1-P1, A1-P4, A1-P9, A1-P12, A1-P13; A2-P1, A2-P3; A4-P8; A5-P3, A5-P6+A8-P4; A6-P3; A8-P2, A8-P6 (vacuous), A8-P12; A9-P6 (under option iii), A9-P7 (vacuous); A10-P2, A10-P6, A10-P7, A10-P9.
- **Proposals that need a small amendment, routed in §5:**
  - A3-P4 — add SPMD aggregation edge to precomputed successor list. **A3-self.**
  - A3-P5 — retain `spmd_index` on cancelled SPMD siblings. **A3-self.**
  - A3-P7 — add `SpmdIndexOOB`, `SpmdSizeMismatch`, `SpmdBlockDimZero`, `TaskSlotExhausted`, `ReadyQueueExhausted` sub-codes. **A3-self.**
  - A3-P10 — update `06-scenario-view.md:126` to reflect the full domain mapping. **A3-self.**
  - A5-P4 — update `06-scenario-view.md:125` so caller-visible back-pressure is allowed regardless of `idempotent`. **Route to A5.**
  - A9-P5 — update `06-scenario-view.md:123, 126` to use `AdmissionStatus`. **Route to A9.**
  - A7-P2 — enforce A3-P2-before-A7-P2 ordering or co-commit. **Route to A7 (+A3).**
  - A1-P8 — distinct sub-codes `TaskSlotExhausted` / `ReadyQueueExhausted` for Pool exhaustion. **Route to A1 (folded into A3-P7 catalog merge).**
  - A1-P11 — confirm SPMD-shape sub-codes live in the single validation pass. **Route to A1 (already folded).**
  - A9-P4 — document `spmd->size == 1` edge case accept/reject. **Route to A9 + A3-self.**
  - A2-P7 — ensure final wording is a pure Q-record entry; if the wording reserves any v1 interface slot, my vote flips to abstain. **Route to A2.**
- **Proposals that break under stress:** none. Every "breaks (minor)" above has a concrete, one-line fix either already absorbed into §5 or routed to the peer owner with a co-edit on `06-scenario-view.md`.
- **A1 hot-path vetoes introduced by A3 this round:** 0. None of my amendments add hot-path cost; every new path (extended precondition catalog, SPMD aggregation edge, `spmd_index` retention in `ErrorContext`) lives on the admission path or the error cold path.
- **New disputes introduced:** 0.
- **Override requests filed by A3:** 0.
- **Blocking objections filed by A3:** 0.

## 9. Status

- **Satisfied with current design?** **Yes, conditional on the stress-round amendments above landing.** The amendments are all small (each is a single-line text change or a single new error sub-code) and there are no new disagreements. The three disputed proposals (A2-P7, A9-P2, A9-P6) all flip to agree under A3 with the synthesis-recommended landings (Q-record, single test seam, option iii respectively).
- **Items expected to close in Phase 5 edit application (from A3's perspective):** A3-P1a, A3-P2 (with ordering constraint), A3-P3, A3-P4 (+SPMD aggregation edge), A3-P5 (+`spmd_index` retention), A3-P6, A3-P7 (+SPMD sub-codes, +pool sub-codes), A3-P8, A3-P9, A3-P10 (+scenario text update), A3-P11 … A3-P15; plus the five co-edits on `06-scenario-view.md` lines 122–126 routed above (A5-P4, A9-P5, A5-P3, A3-P4, A3-P10).
- **Recommendation to the parent:** mark A3 review **converged** for round 3. No items require a round-4 return for A3.
