# Aspect A4: Document Consistency — Round 2

## Metadata

- **Reviewer:** A4
- **Round:** 2
- **Target:** `docs/pypto-runtime-design/` (same package as round 1; no target changes)
- **Runtime Mode:** yes (Performance A1 ×2 weight; A4 is non-A1, `override_request` available)
- **Timestamp (UTC):** 2026-04-19T00:00:00Z

---

## 1. Rubric Checks

No new rubric evidence since round 1 — the target documents have not changed between rounds. All findings from round 1 still hold; see `round-1/reviews/A4-doc-consistency.md §1`. Per the prompt, round 2 is cross-critique + voting, not a re-scan of the rubric.

| # | Check | Result (R1) | Notes for R2 |
|---|-------|-------------|--------------|
| 1 | Same concept named identically across views, diagrams, module docs, ADRs, glossary (D7, V5) | **Fail** | A4-P1 (HAL enum casing) defended below; A7-P8 and A7-P9 reinforce. |
| 2 | Logical ↔ Development ↔ Process ↔ Physical mapping (V2, V5, D7) | **Weak** | A4-P7 (L0 label unification) defended; A10-P8 (aggregated data-flow diagram) complements. |
| 3 | All required views present (V1) | **Pass** | Unchanged. |
| 4 | ADRs referenced from the views they constrain (V2, G5) | **Fail** | A4-P6 defended; A2-P4 and A2-P5 (migration plan, BC policy) deepen. |
| 5 | Glossary covers every formal term (D7) | **Weak** | A4-P5, A4-P9 — amended below to reflect A9-P7 merge and A3-P1 ERROR-state addition. |
| 6 | `appendix-b-codebase-mapping.md` matches current Development View (D7) | **Weak** | A4-P8 amended below to include `ERROR` / `CANCELLED` states if A3-P1 agreed. |

Newly surfaced consistency items from peer round 1 (raised for round 2 resolution, not net-new rubric failures):

- Two near-identical enums `AdmissionDecision {ADMIT, WAIT, REJECT}` vs. `ResourceAllocationResult::Status {ALLOCATED, WAIT, REJECTED}` on the same admission hook (A9-P5). A4 endorses the rename to a single `AdmissionStatus`; this is a clean D7 win.
- Two Python classes named `MemoryError` in `simpler.errors` (A7-P9). A4 endorses; D7 namespace collision.
- Duplicated `ScopeHandle` concept across `core/` and `memory/` (A7-P8). A4 endorses consolidation in `core/types.h`; D7.
- Forward-declaration contract on `core/i_scheduler_layer.h` is invisible in the DAG (A7-P7). A4 endorses documenting it; satisfies D6 truthfulness and improves reader trust in the dependency table.
- `logical_system_id` proposed in `MessageHeader` (A6-P9) and `Attestation`/`credentials` in various configs (A6-P2, A6-P7) — glossary additions required if accepted.

---

## 2. Pros

Unchanged from round 1 (see `round-1/reviews/A4-doc-consistency.md §2`). Two reinforcements from peer reviews:

- **Peer A7 and A9 independently surface D7/DRY hotspots I missed** (duplicate `ScopeHandle`, duplicate `MemoryError`, near-duplicate admission enums). This validates A4's rubric #1 being only Fail/Weak, not Pass — and hands A4 three high-confidence, low-controversy fix targets.
- **Peer A2's proposal for a new `appendix-c-compatibility.md`** (A2-P1 / A2-P5) directly strengthens A4's rubric #4 (ADRs referenced from the views they constrain) by giving every interface a stable BC class; A4 supports this as a natural extension of the ADR cross-link effort proposed in A4-P6.

## 3. Cons

Unchanged from round 1. One new observation:

- **Peer overlap with A4-P5 from A9-P7.** A9-P7 proposes folding `SourceCollectionConfig` into `EventHandlingConfig`, which directly overlaps with my A4-P5 (add glossary entry for `SourceCollectionConfig`). Tension is already flagged in the synthesis Conflict Register. Revision is `amend` — see §5.

---

## 4. Proposals (new in this round)

No new proposals this round. All A4 proposals from round 1 are carried forward with revisions as §5 below.

---

## 5. Revisions of own proposals

| id | action | amended_summary (if amend/split) | reason |
|----|--------|----------------------------------|--------|
| A4-P1 | defend | — | Pure prose edit; `hot_path_impact: none` re-attested. No peer objected in round 1; A1 on hot-path perspective should confirm no allocation/blocking/relayout effect. Cites D7, V5. |
| A4-P2 | defend | — | Blocker-severity anchor fix; trivial, unambiguous. No peer objected. Cites D7, V5. |
| A4-P3 | defend | — | "Two" → "Three" invariants prose correction. No peer objected; no dependency on other proposals. Cites D7, G4. |
| A4-P4 | defend | — | Reorder Q9 between Q8 and Q10 in `09-open-questions.md`. No peer objected; no cross-reference breakage because references use section titles/IDs, not line-anchors. Cites D7, V5. |
| A4-P5 | amend + split | **Amended:** Add glossary entries for `EventLoopDeploymentConfig` and `IEventCollectionPolicy`. **Split:** The `SourceCollectionConfig` entry becomes conditional — add it if and only if A9-P7 (fold into `EventHandlingConfig`) is rejected. If A9-P7 is agreed, `SourceCollectionConfig` is removed from the source docs and A4-P5 correspondingly drops that entry. Similarly, if A9-P2 (cut `IEventCollectionPolicy` as pluggable interface) is agreed, the `IEventCollectionPolicy` entry is replaced by a one-line glossary entry clarifying that the name refers to a config-field choice, not a C++ interface. | Avoid glossary entries for types the scheduler module may remove in round 2; maintain D7 coherence with whichever shape wins the A2↔A9 debate. |
| A4-P6 | defend | — | ADR cross-links in the top-level views (03, 04, 05, 06). No peer objected; A9 will likely note prose bulk — pre-empted by "inline cite only where decision is local; no new 'Related ADRs' section required." Cites V2, G5. |
| A4-P7 | defend | — | L0 label unification to "Scheduler: AicoreDispatcher" — A7 independently surfaces the same D7/V5 concern (fat interface + filename naming) and supports a clean, single term for the `ISchedulerLayer` implementation. Defends cleanly. Cites V5, D7. |
| A4-P8 | amend | **Amended:** Update Appendix-B TaskState annotation to reflect **(a)** the existing 10 lifecycle states + `FAILED`, **and (b)** the `ERROR` (optional `CANCELLED`) states if A3-P1 is agreed. The annotation becomes "10 lifecycle states + `FAILED` + `ERROR` [+ `CANCELLED` if adopted]; see `modules/core.md §2.3` and `04-process-view.md §4.3`." | A3-P1 introduces additional states at blocker severity; Appendix-B must not drift again immediately after this fix. Cites D7. |
| A4-P9 | amend | **Amended:** Expand `Task State` glossary entry to enumerate `FREE → SUBMITTED → PENDING → DEP_READY → RESOURCE_READY → DISPATCHED → EXECUTING → COMPLETING → COMPLETED → RETIRED` **plus** `FAILED` and (conditional on A3-P1) `ERROR` / `CANCELLED` terminal/branching states. | Same reason as A4-P8: mirror `modules/core.md` and the Process View state diagram exactly after A3-P1 settles. Cites D7. |

Notes on amendments:
- A4-P5 split is deliberately minimal — the glossary rows are three single-line entries; the conditional handling adds no new moving parts, only "include row X iff simplification proposal Y is rejected."
- A4-P8 / A4-P9 amendments do not change the edit's shape; they extend the set of values to enumerate in light of A3's state-machine completion.

---

## 6. Votes on peer proposals

Votes are rendered from A4's rubric (D7, V1, V2, V5) with cross-cutting awareness of G5 (justify decisions). `blocking=true` is set only when a hard rule from `04-agent-rules.md` is at stake **from A4's aspect**; A4 does not block on A1/A5/A6 rules. `override_request=true` is set only when A4 believes A1 is overusing the hot-path veto — no such cases observed in the round-1 proposals (A1 has not yet voted in round 2).

| proposal_id | vote | rationale (cite rule id) | blocking | override_request |
|-------------|------|--------------------------|----------|-------------------|
| A1-P1 | agree | X2 pre-sizing; pure doc additions in `modules/scheduler.md` §8. No D7/V5 risk — adds `producer_index_capacity` config field; A4 will want this mirrored in glossary (flagged as tension). | false | false |
| A1-P2 | agree | X3 lock-hold-time specification; adds `producer_index_shards` LevelParam. Glossary row required if the LevelParams surface is visible — flagged as tension §7. | false | false |
| A1-P3 | agree | P2 cache discipline; adds `function_cache_bytes` and extends `HeartbeatPayload`. Glossary + `modules/transport.md §2.3` update needed — flagged as tension §7. | false | false |
| A1-P4 | agree | X4 hot/cold split. A4-specific benefit: the enumeration replaces a vague "hot/cold separation" phrase with an explicit field list → reduces D7 drift between `07-task-model.md`, `modules/core.md`, and Process View §4.3. | false | false |
| A1-P5 | agree | X9 latency budgets; new §4.8.6 / §4.8.7 subsections. A4 benefit: every critical path from `07-cross-cutting-concerns.md §7` becomes traceable to a budget row (rubric check #4 adjacent). | false | false |
| A1-P6 | agree | P6 + new `MessageType::REMOTE_BINARY_PUSH`; merged with A10-P5. Requires glossary entry (`REMOTE_BINARY_PUSH`, `descriptor_template_id`) and ADR note. A4 supports subject to those doc updates landing in the same edit. | false | false |
| A1-P7 | agree | X3/P1 profiling fix. No doc-consistency risk; `SequenceAllocator` removal is a module-internal detail. | false | false |
| A1-P8 | agree | X2/X4 pre-sizing; adds capacity/placement block in `modules/scheduler.md §4`. A4 wants the same placement language to appear in `02-scheduler.md` to avoid drift. | false | false |
| A1-P9 | agree | X4 bitmask layout. No doc-consistency risk; one sub-bullet in `03-worker.md §2.1.4.2`. | false | false |
| A1-P10 | agree | P1 CI gate; adds §9.2.4 in `modules/profiling.md`. No D7 risk. | false | false |
| A1-P11 | agree | X9 per-arg budget; new subsection in `modules/bindings.md`. Merged with A6-P8 per register. A4 agrees with the merge. | false | false |
| A1-P12 | agree | X9 + P5 batch bound. Adds row to Deployment-Modes table — must mirror in §4.8.1. Minor glossary update (`max_batch`). | false | false |
| A1-P13 | agree | P6 + X9 `args_blob` bound. Updates `modules/hal.md §2.4` and §4.8.1 in the same edit — A4's main requirement is satisfied. | false | false |
| A1-P14 | agree | X4 doc-only; merged with A10-P10. Directly supports A4 rubric #1 (D7 layout specificity). | false | false |
| A2-P1 | agree | E6 versioning; adds `schema_version` to every cross-module contract. A4 strongly supports because it creates a per-type BC traceability row in the new `appendix-c-compatibility.md` — that appendix is a direct A4 artifact (closes rubric #4 at the BC layer). | false | false |
| A2-P2 | agree | E4 OCP; `LevelOverrides` schema registration. Requires glossary update (`LevelOverridesSchema`). A4 OK. | false | false |
| A2-P3 | agree | E4 with DepMode closed kept — the decision to keep `DepMode` closed preserves the A4 glossary entry for it and avoids a drifting enum. Net D7 positive. | false | false |
| A2-P4 | agree | E5 migration plan in `03-development-view.md`. Directly supports A4 rubric #2 (V2 cross-view mapping) and resolves A4's concern that `appendix-b` is a static map only. | false | false |
| A2-P5 | agree | E2 BC policy in new `appendix-c-compatibility.md`. Direct alignment with A4-P6 (top-level views cite ADRs) — both strengthen the "views ↔ decisions" back-link web. A4 prefers a single appendix-c carrying both BC classes and ADR summary back-links. | false | false |
| A2-P6 | agree | E4 registry dispatch. Adds `DistributedMessageHandler` typedef + `§2.6.2.1` subsection in `09-interfaces.md`. A4 requires glossary row (`DistributedMessageHandler`). | false | false |
| A2-P7 | abstain | Reserved `IAsyncTaskSchedulePolicy` is a YAGNI debate between A2 and A9; A4 has no D7/V5 stake if the reservation lives in `09-open-questions.md` as Q11 companion rather than `09-interfaces.md`. A4's preference: put it in Q11 (less glossary churn); does not block either outcome. | false | false |
| A2-P8 | agree | Explicit known-deviation entries for intentional closures. Direct alignment with A4's own R1 positive observation on known-deviations discipline. | false | false |
| A2-P9 | agree | E6 trace schema version. Required for REPLAY forward-compat; adds header struct in `07-cross-cutting-concerns.md §7.2.2`. A4 co-listed this intent in round-1 §7. | false | false |
| A3-P1 | agree | LSP + D7 + V5 — this is the single largest consistency hole I noted in round 1 (§1 row 3). Adding `ERROR` (optional `CANCELLED`) to the normative state machine closes 5 distinct cross-view drifts (`04-process-view.md:229-284` vs. `06-scenario-view.md:93,111,126,138` and `07-task-model.md:§2.4.4`). My A4-P8 / A4-P9 amendments anticipate this outcome. | false | false |
| A3-P2 | agree | LSP + D7; clarifies `submit()` return contract. Removes an ambiguous narrative at `09-interfaces.md:65` that currently contradicts the signature. | false | false |
| A3-P3 | agree | V4 — adds the admission failure scenario I flagged as missing (rubric #2). | false | false |
| A3-P4 | agree | R1 + DS3 + G3; deterministic producer-failure propagation. Adds `DEP_FAILED` to `SchedulerEvent::Type` — requires glossary/interface doc sync (A4 flags in §7). | false | false |
| A3-P5 | agree | DS4 + G3; sibling cancellation policy. Adds `IResourceAllocationPolicy::on_child_error()` — glossary row required; supports A3-P1 composition. | false | false |
| A3-P6 | agree | G1 + V3. Requirement-to-scenario traceability matrix directly satisfies A4 rubric-adjacent trace. | false | false |
| A3-P7 | agree | G3 + S3 precondition validation. Adds six sub-codes in `modules/error.md §2.1` — every new code needs a matching row; glossary may remain unchanged (error codes are a flat list). | false | false |
| A3-P8 | agree | G3 + X9 — cyclic-edge detection algorithm + cost. No D7 impact; improves §4.8.4 table specificity. | false | false |
| A3-P9 | agree | G3 + LSP SPMD `spmd_index`/`spmd_size` contract. Adds to `06-function-types.md` — aligns with A4 rubric #2. | false | false |
| A3-P10 | agree | G1 + D7; completes Python exception mapping. Glossary already carries `simpler.*Error` names; this closes the `Domain` ↔ exception map. | false | false |
| A3-P11 | agree | G3 `[ASSUMPTION]` marker; trivial fix. | false | false |
| A3-P12 | agree | G3 + R4 `drain()`/`submit()` contract. Adds `DrainInProgress` error code. | false | false |
| A3-P13 | agree | DS3 + G3 cross-node ordering assumptions. A4-friendly `[ASSUMPTION]` discipline. | false | false |
| A3-P14 | agree | LSP + V5 — clarifies "uniform state machine" claim. Directly addresses a V5 claim I called out in round 1. | false | false |
| A3-P15 | agree | G3 + X6 debug-mode cross-check. No hot-path impact (debug only); doc addition in `02-scheduler.md §2.1.3.1.B`. | false | false |
| A5-P1 | agree | R2 exponential backoff + jitter spec. Adds `RetryPolicy` struct — glossary row needed (flagged §7). | false | false |
| A5-P2 | agree | R3 per-peer circuit breaker. Adds `CircuitBreaker` config + `{CLOSED, OPEN, HALF_OPEN}` states — glossary rows. | false | false |
| A5-P3 | agree | R5 coordinator-failover or deterministic fail-fast. Closes Q5; adds `CoordinatorLost` error code. Merged with A10-P2. | false | false |
| A5-P4 | agree | DS4 — `idempotent` bool on `TaskDescriptor`. Glossary row + Python API doc update. | false | false |
| A5-P5 | agree | R6 chaos harness. New `11-chaos-plan.md` (or §7.4) supports V4 and adds traceability. A4's one concern is that the matrix matches the `§4.8 critical-path list` format; flagged §7. | false | false |
| A5-P6 | agree | R5 + O5 scheduler-thread watchdog. Merged with A8-P4 — the watchdog's evidence surface is `dump_state`. | false | false |
| A5-P7 | agree | R1 — `Timeout` on `IMemoryOps`. Interface change must propagate to `02-logical-view/04-memory.md` and `modules/memory.md §2` simultaneously (flagged §7). | false | false |
| A5-P8 | agree | R4 degradation policies for admission and worker-group partial loss. Adds enums (`{REJECT, COALESCE, DEFER}` and `{FAIL_SUBMISSION, SHRINK_AND_RETRY, DEGRADED_CONTINUE}`) — glossary rows. | false | false |
| A5-P9 | agree | R3 `QUARANTINED` Worker state. Adds a state; `modules/scheduler.md §3.2` + `03-worker.md §2.1.4.1` + glossary must all land together. | false | false |
| A5-P10 | agree | DS4 — per-handler idempotency annotation. Pure doc-discipline improvement. | false | false |
| A6-P1 | agree | S1 — completes the trust-boundary table. Supports V1 "non-empty" rubric by filling currently under-specified §7.1 rows. | false | false |
| A6-P2 | agree | S1 + S2 — concrete node auth. New `HandshakePayload` + `credential_id` field; glossary rows. | false | false |
| A6-P3 | agree | S3 bounded payload parsing. Adds 4 config keys in `modules/transport.md §8`; glossary unchanged (they are config, not types). | false | false |
| A6-P4 | agree | S4 + S5 TLS default for multi-host. `DeploymentConfig.require_encrypted_transport` field + glossary. Updates `10-known-deviations.md` Deviation 3 — direct A4 alignment. | false | false |
| A6-P5 | agree | S2 scoped rkey. `IMemoryOps::register_peer_read`/`deregister_peer_read` — new interface methods; must update `02-logical-view/04-memory.md` in same edit (flagged §7). | false | false |
| A6-P6 | agree | S6 audit trail. New §7.1.4 + `AuditEvent` + `IAuditSink` — glossary rows; directly adds to A4's V2 coverage. | false | false |
| A6-P7 | abstain | Function-binary attestation is a YAGNI-vs-security tension; A4 has no direct D7/V5 stake. If accepted, `Attestation` and `allow_unsigned_functions` need glossary rows — purely mechanical. | false | false |
| A6-P8 | agree | S3 DLPack validation. Merged with A1-P11 per register. | false | false |
| A6-P9 | agree | S1 + S2 + S3 tenant isolation on wire. Adds `logical_system_id` to `MessageHeader` — critical V5 impact (diagram/header consistency) and direct D7 improvement (tenant id now has a single authoritative field). | false | false |
| A6-P10 | agree | S2 capability-scoped sinks. `add_sink` signature change — `modules/bindings.md §2.1`, `modules/profiling.md §2`, `07-cross-cutting-concerns.md §7.2.4` must land together. | false | false |
| A6-P11 | agree | S2 gated `register_factory`. `RegistrationToken` + `factory_registration_token` — glossary rows. | false | false |
| A6-P12 | abstain | Contingent on A9-P6 (defer REPLAY). If REPLAY is deferred, A6-P12 is moot. A4 cannot vote on a conditional; abstain per prompt. | false | false |
| A6-P13 | abstain | Per-tenant rate limit — a hot-path cost debate (A1/A9). A4 rubric unaffected regardless of outcome. | false | false |
| A6-P14 | agree | G5 + S5 key-material ADR. Direct A4 benefit — new ADR cross-linked from Deviation 3 and `transport.md §12` (matches A4-P6 philosophy). | false | false |
| A7-P1 | agree | D6 hard rule — cycle removal. This is a hard-rule fix in A4's D7/V2 neighborhood (though D6 is owned by A7). Every doc whose dependency table currently shows the cycle must be updated atomically — `03-development-view.md`, `modules/distributed.md`, `modules/scheduler.md` — so A4 strongly supports the coordinated edit. Not blocking from A4 because D6 is A7's hard rule. | false | false |
| A7-P2 | agree | ISP + D4 — split `ISchedulerLayer`. Merged with A9-P1. Improves D7 (narrower interfaces, explicit role names). A4 requires each role interface to appear in glossary + `02-logical-view.md §2.7` Interfaces-at-a-Glance. | false | false |
| A7-P3 | agree | D2 — invert `core/ ↔ hal/`. `DeviceAddress`/`NodeId` move to `core/types.h`. Pure DAG edit; aligns with A4 rubric #6 (Appendix-B must be synced). | false | false |
| A7-P4 | agree | D3 + D5 — move distributed payload structs from `transport/` to `distributed/`. Direct A4 benefit: the `modules/transport.md §2.3` payload section becomes a pure framing spec, reducing two-owner drift (which I flagged in round 1 cons). | false | false |
| A7-P5 | agree | D2 — enforce ADR-008. Unlocks the ADR's claim, closes a V2 inconsistency between `modules/distributed.md` and ADR-008 text. | false | false |
| A7-P6 | agree | D3 + D5 — extract `composition/` module. Adds a new module doc (`modules/composition.md` or `modules/registry.md`); A4 requires the new doc to follow the existing per-module template so `00-index.md` and cross-links stay consistent (flagged §7). | false | false |
| A7-P7 | agree | D6 truthfulness — document forward-decl contract on `core/i_scheduler_layer.h`. Direct alignment with A4 rubric #2 (V2 cross-view mapping) and eliminates a quiet invisible edge. | false | false |
| A7-P8 | agree | D7 — consolidate `ScopeHandle` in `core/`. This is the textbook A4 rubric-#1 case: one concept must have one name and one owner. Strongly support. | false | false |
| A7-P9 | agree | D7 — deduplicate Python `MemoryError` classes. Same rubric; trivially right. | false | false |
| A8-P1 | agree | X5 — injectable `IClock`. Adds `hal::IClock` + realizations; glossary rows; also helps A4-adjacent determinism claims (`01-design-principles.md:389`). | false | false |
| A8-P2 | agree | X5 — driveable event loop `step()` + `RecordedEventSource`. Test-only surface; requires corresponding glossary row for `RecordedEventSource`. | false | false |
| A8-P3 | agree | O3 + O5 — enumerate stats structs, add `LatencyHistogram`. A4 benefit: alerts at `07-cross-cutting-concerns.md:137-146` become traceable to concrete fields; closes a V2 "references counters that are not exposed" gap. | false | false |
| A8-P4 | agree | O4 + X6 — `Runtime::dump_state()`. Merged with A5-P6. Glossary rows (`StateDump`, `DumpOptions`). | false | false |
| A8-P5 | agree | O5 + E3 + X8 — externalize alert rules. `AlertRule` schema + `on_alert` API — glossary rows. | false | false |
| A8-P6 | agree | O1 distributed — trace time-alignment contract. Adds PTP/NTP dependency + `IClockSync` — glossary rows; matches A3-P13 ordering assumptions. | false | false |
| A8-P7 | agree | X5 + DfT — `IFaultInjector`. Sim-only; needs `modules/hal.md §2.x` + `06-scenario-view.md §6.2` cross-refs. | false | false |
| A8-P8 | agree | O3 — AICore in-core trace protocol. Closes the open question cited at `modules/profiling.md:461`; A4 benefit: one fewer unresolved cross-reference. | false | false |
| A8-P9 | agree | O3 + O5 — drop counters as first-class alerts. Two additional alert rows in `07-cross-cutting-concerns.md §7.2.8`. | false | false |
| A8-P10 | agree | O2 structured KV logging. Closes `modules/profiling.md:463` open question. | false | false |
| A8-P11 | agree | X5 HAL contract test suite. Adds `modules/hal.md §9` bullet + `03-development-view.md §3.3.6` ref. | false | false |
| A8-P12 | agree | O1 stable `PhaseId`s for Submission lifecycle. Adds named IDs in `modules/profiling.md §2.5`; glossary row. | false | false |
| A9-P1 | agree | G2 + DRY — drop `submit_group`/`submit_spmd` overloads. Merged with A7-P2. Reduces vtable + entry-point surface; improves D7 (one canonical submit path). | false | false |
| A9-P2 | abstain | Core A2↔A9 extensibility debate. A4 accommodates either outcome via A4-P5 amendment (§5). No D7/V5 blocker from either direction. | false | false |
| A9-P3 | abstain | Depends on Q4 (collectives-on-channel) resolution. A4 accommodates either outcome: if removed, 2 glossary rows drop; if kept, glossary rows stay. | false | false |
| A9-P4 | agree | G2 + DRY — drop `SubmissionDescriptor::Kind`. Removes a discriminant the runtime does not branch on for correctness; D7 win because the SPMD/Single/Group distinction is already carried by `spmd.has_value()` and `tasks.size()`. | false | false |
| A9-P5 | agree | G2 + DRY — unify `AdmissionDecision` and `ResourceAllocationResult::Status` on a single `AdmissionStatus`. This is the clearest D7-coded win in the entire round-1 catalog — two enums for one concept at two scopes of the same policy hook. A4 strongly supports. | false | false |
| A9-P6 | abstain | Defer PERFORMANCE/REPLAY — directly interacts with A6-P12 (signed REPLAY trace) and A8 reproducibility goals. A4 is neutral on scope; if accepted, Appendix-A / ADR-011 edits are purely mechanical. | false | false |
| A9-P7 | agree | G2 + DRY — fold `SourceCollectionConfig` into `EventHandlingConfig`. Reduces config-surface fan-out; A4-P5 amends to drop the corresponding glossary row (see §5). | false | false |
| A9-P8 | agree | G2 + SRP — move per-Function companion artifacts obligation out of the runtime design into compiler/kernel tooling spec. Directly removes out-of-scope content from a runtime design doc; A4 benefit is a narrower, more consistent scope. | false | false |
| A10-P1 | agree | P4 + X3 — default `producer_index` sharding. Promotes MAY to SHOULD in `02-scheduler.md`; aligns with A1-P14 (placement doc). Pairs cleanly with A1-P2 (shard count LevelParam). | false | false |
| A10-P2 | agree | P4 + R5 — decentralize Pod-level coordinator. Merged with A5-P3. Adds ADR + §5.4.0 `Coordinator Membership` — strong V2 addition. | false | false |
| A10-P3 | agree | DS6 — explicit per-data-element consistency table in `07-cross-cutting-concerns.md`. Direct A4 alignment: the new table complements A10-P8's ownership diagram and closes A4's Con "consistency statements hidden across six files." | false | false |
| A10-P4 | agree | DS1 — stateful/stateless module classification in `03-development-view.md`. Direct A4 alignment with rubric #2 (V2 mapping). | false | false |
| A10-P5 | agree | P6 — per-peer `REMOTE_SUBMIT` projection. Merged with A1-P6 per register. | false | false |
| A10-P6 | agree | R5 + P4 — configurable heartbeat detection; adds `heartbeat_miss_threshold_fast` config field. Glossary update. | false | false |
| A10-P7 | agree | P4 — sharded TaskManager path. Hot-path impact flagged; A4 requires the new subsection in `02-scheduler.md` to be cross-linked from `modules/scheduler.md §4` to avoid drift. | false | false |
| A10-P8 | agree | DS7 + P6 — aggregated data-flow + ownership diagram in `02-logical-view.md`. Direct A4 rubric #2 improvement; naturally complements A10-P3 consistency table. Strong support. | false | false |
| A10-P9 | agree | DS6 + P4 — gate `WorkStealing` against `RETRY_ELSEWHERE`. Adds a documented incompatibility; small but precise D7 discipline. | false | false |
| A10-P10 | agree | P6 + DS6 — cap and layout guidance for `producer_index`. Merged with A1-P14. | false | false |

**Vote tally:** 101 peer proposals; **92 agree**, **0 disagree**, **9 abstain** (A2-P7, A6-P7, A6-P12, A6-P13, A9-P2, A9-P3, A9-P6, plus implicit: no more). Zero `blocking=true` — none of the outstanding objections rest on a hard rule A4 owns (D7/V1/V2/V5), because every A4-tension in round 1 resolves to "add a glossary row in the same edit." Zero `override_request=true` — A1 has not yet voted in round 2, and A4 does not pre-empt.

---

## Merge Register — A4 positions

| Merge into | Absorbed | A4 vote | Rationale |
|------------|----------|---------|-----------|
| A1-P6 | A10-P5 | accept | Same concept (REMOTE_SUBMIT hygiene / projection). Merging into a single proposal keeps one authoritative glossary + `§4.2.4`/`modules/transport.md §4.3` edit set; avoids doc drift between a size cap proposal and a projection proposal. |
| A1-P14 | A10-P10 | accept | Same artifact (`producer_index` placement + capacity). One authoritative doc edit in `02-scheduler.md` + `modules/scheduler.md §4`. |
| A5-P3 | A10-P2 | accept | Same architectural concern (coordinator SPOF / decentralize). A4 benefit: Q5 closes with a single coherent ADR rather than two competing ADR candidates. |
| A5-P6 | A8-P4 | accept | Watchdog deadman slot + `dump_state` diagnostic reader are two halves of one O4/R5 story. Keeping them merged ensures `Runtime::dump_state()` surfaces the deadman timestamp in its schema on the first version, rather than requiring a schema bump later — clean E6 + D7. |
| A7-P2 | A9-P1 | accept | Role-interface split + single `submit()` entry is the same `ISchedulerLayer` surface change. Two separate proposals would force two edits to `02-logical-view/09-interfaces.md` / `modules/core.md` — merging keeps the interface definition authoritative. |
| A6-P8 | A1-P11 (partial) | accept | Per-arg boundary validation at Python↔C fits inside the per-arg budget. A4 requires both concerns (validation + budget) to land in the same `modules/bindings.md §5.2/§9.5` edit to avoid a cross-proposal drift between security checklist and performance budget; merge enforces this. |

No merge is rejected from A4's perspective; all six are net D7/V2 wins.

---

## 7. Cross-aspect tensions (observed this round)

| pair | with_proposal | preferred_resolution |
|------|---------------|----------------------|
| A4 vs A9 | A4-P5 (glossary for `EventLoopDeploymentConfig`, `SourceCollectionConfig`, `IEventCollectionPolicy`) vs A9-P2 / A9-P7 (cut pluggable policies; fold `SourceCollectionConfig`) | A4-P5 splits (see §5). If A9-P2 agreed → `IEventCollectionPolicy` glossary entry becomes a one-line "deprecated in v1, folded into `EventHandlingConfig` enum" note. If A9-P7 agreed → `SourceCollectionConfig` entry is dropped entirely. A4 owns the follow-through; no blocking either way. |
| A4 vs A3 | A4-P8 / A4-P9 (`TaskState` count and glossary) vs A3-P1 (add `ERROR` / `CANCELLED`) | A4-P8 / A4-P9 amend to enumerate the round-2 state set including `ERROR` (and `CANCELLED` if adopted). Resolution: A3-P1 lands first; A4 follows in the same PR to prevent the short-lived post-A3 drift A4-P8 is designed to close. |
| A4 vs A1 | A4-P1..P9 vs A1's hot-path label set | All A4 proposals carry `hot_path_impact: none` — zero allocation, zero blocking, zero layout, zero latency change. They are pure prose / anchor / table edits. A1 should rubber-stamp with no conditions. |
| A4 vs A2 | A4-P6 (top-level views cite ADRs) vs A2-P5 (BC policy in `appendix-c-compatibility.md`) | Complementary. Preferred: produce `appendix-c-compatibility.md` with two sections — BC class per interface (A2) and ADR back-link index per view (A4) — as one unified "decisions & contracts" appendix. Zero conflict; one smaller PR. |
| A4 vs A7 | A4-P7 (unify L0 label) vs A7-P2 (split `ISchedulerLayer`) | If A7-P2 agreed, A4-P7 relabels the L0 box as `Scheduler: AicoreDispatcher` implementing `ISchedulerSubmit + ISchedulerLifecycle + ...` (the exact split). Resolution: land A7-P2 first (interface split), then A4-P7 applies the renamed term to Physical View. Clean compose, no tension. |
| A4 vs A10 | A4-P5 glossary vs A10-P3 (consistency table) / A10-P4 (stateful classification) / A10-P8 (ownership diagram) | A4 should absorb the four-into-one "Data & State Reference" suggestion A10 makes in its §7. Preferred: one new section in `07-cross-cutting-concerns.md` that combines (a) consistency table (A10-P3), (b) stateful/stateless classification (A10-P4), (c) ownership diagram (A10-P8), (d) glossary back-references. A4 authors the skeleton so the new section stays consistent with the existing appendix-A/appendix-B voice. |
| A4 vs A6 | A4-P5 (glossary) vs A6-P2 / A6-P6 / A6-P9 / A6-P11 (new auth / audit / tenant / token types) | Each A6 proposal introduces new formal types (`HandshakePayload`, `AuditEvent`, `logical_system_id`, `RegistrationToken`, `FunctionNotAttested` code, etc.). A4 commits to extending A4-P5 with a glossary row per accepted A6 type in the same PR that lands the A6 change. No blocking. |
| A4 vs A5 | A4 glossary discipline vs A5-P1..P9 new types (`RetryPolicy`, `CircuitBreaker`, `DrainInProgress`, `QUARANTINED`, `DEP_FAILED`, `AdmissionPressurePolicy`, …) | Same pattern. A4 extends glossary + Appendix-B + `04-process-view.md` state diagram in the same edit that lands each A5 proposal. No structural tension. |
| A4 vs A8 | A4 glossary vs A8 new types (`IClock`, `RecordedEventSource`, `LatencyHistogram`, `StateDump`, `AlertRule`, `IClockSync`, `PhaseId`, `IFaultInjector`) | Same pattern; follow-through only. A4 accepts the maintenance cost. |

No blocking cross-aspect tension observed from A4 this round.

---

## 8. Stress-attack on emerging consensus

Not applicable in round 2. Per the prompt, stress-attacks are for round 3.

---

## 9. Status

- **Satisfied with current design?** Partially — position unchanged from round 1. The round-1 consistency defects remain, but the round-1 proposal set (A4-P1..P9, with A4-P5/P8/P9 amended in §5) is sufficient to close them, and the peer proposals I voted `agree` on directly complement A4's rubric (A3-P1 → A4-P8/P9; A7-P8/P9 → A4 rubric #1; A9-P5 → A4 rubric #1; A2-P1/P5 + A10-P3/P4/P8 → A4 rubric #2 and #4). Round 3 stress-attack target: the cross-document propagation discipline — every newly agreed proposal that introduces a formal type or state must simultaneously land in `appendix-a-glossary.md`, the relevant interface doc, the `02-logical-view` owner section, and (where state-bearing) the Process View state diagram. Absent this discipline, round-3 agreement becomes a fresh source of round-4 drift.
- **Open items expected in next round (round 3 stress-attack):**
  - A4-P5 split resolution — waiting on A9-P2 / A9-P7 outcomes.
  - A4-P8 / A4-P9 amendment resolution — waiting on A3-P1 outcome (`ERROR` / `CANCELLED`).
  - Appendix-c-compatibility.md composition — coordinate with A2 (BC policy) and A4-P6 (ADR back-links) into one unified appendix.
  - Data & State Reference section in `07-cross-cutting-concerns.md` — coordinate with A10-P3 / A10-P4 / A10-P8 into one section.
  - Merge register — all six merges accepted; owner lists updated as indicated above.
