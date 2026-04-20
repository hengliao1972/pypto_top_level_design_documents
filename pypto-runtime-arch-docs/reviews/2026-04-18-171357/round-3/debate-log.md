# Round 3 (Stress Round) — Debate Log

- Run: `2026-04-18-171357`
- Target: `docs/pypto-runtime-design/`
- Mode: Runtime Design Mode (A1 ×2, hot-path veto enabled)
- Scenarios replayed: `06-scenario-view.md` §6.1.1, §6.1.2, §6.1.3, §6.2.1, §6.2.2, §6.2.3, §6.2.4

## 1. Vote matrix (disputed proposals only)

Legend: `a` = agree, `d` = disagree, `-` = abstain. Weight in parentheses.

| Proposal | A1(2) | A2(1) | A3(1) | A4(1) | A5(1) | A6(1) | A7(1) | A8(1) | A9(1) | A10(1) | Weighted agree | Result |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| A2-P7 | a | a(owner) | a | a | a | a | a | a | a | a | 11/11 | agreed |
| A9-P2 | a | a | a | a | a | a | a | a | a(owner) | a | 11/11 | agreed |
| A9-P6 (option iii) | a | a | a | a | a | a | a | a | a(owner) | a | 11/11 | agreed |

For the 107 carry-over agreed proposals, no re-vote was taken; each reviewer's R3 §8 stress-attack output was binned per §2 below.

## 2. Stress attack resolutions — per-proposal ledger (only items that moved)

| Proposal | Reviewer attacking | Finding | Resolution |
|---|---|---|---|
| A1-P6 | A1 (self) | Cold-start template-miss unbudgeted | minor amend (owner) |
| A2-P1 | A9 | Blanket versioning violates G2/YAGNI | scope narrowed to wire + persistent artifacts |
| A2-P5 | A2 (self, via R3 §4) | Risk of applying version byte to only one header | cross-reference checklist added |
| A2-P6 | A2 (self) + A9 | Abstract class with one impl = rubric 4 violation | demote to free function in `protocol.hpp` |
| A2-P7 | A2 (self, via canonical 6.1.2 replay) | RDMA rkey vs TCP TLS binding don't compose | Q-record third axis (transport-capability semantics) |
| A3-P4 | A3 (self, 6.1.3 replay) | SPMD aggregation edge not covered | extend edit sketch |
| A3-P5 | A3 (self, 6.1.3 replay) | Sibling `spmd_index` loss | retain in remote_chain |
| A3-P7 | A3 (self, 6.1.3 replay) | Missing SPMD precondition sub-codes | add `SpmdIndexOOB`, `SpmdSizeMismatch`, `SpmdBlockDimZero` |
| A3-P10 | A3 (self, 6.2.3 replay) | Error type spelling inconsistent with domain map | update scenario text |
| A5-P2 | A5 (self, 6.2.2 flapping-peer replay) | CB hammers flapping peer | introduce `breaker_auth_fail_weight` (→ A5-P13) |
| A5-P3 | A5 (self, 6.2.2 coordinator SIGKILL replay) + A10 | Failover wait exceeds SLO / cluster-wide fail-closed blast | `coordinator_liveness_timeout_ms` + pin scope to failed Pod |
| A5-P6 | A1 (peer) | Deadman write elision normatively unclear | add normative clause in `modules/scheduler.md` |
| A5-P9 | A9 | `QUARANTINED` redundant w/ `UNAVAILABLE+since` | DRY-fold into `UNAVAILABLE + {Permanent, Quarantine(duration)}` |
| A6-P2 | A6 (self, 6.2.2 malicious demoted-coordinator replay) | Attacker replays submissions with old cert | add `coordinator_generation` + `StaleCoordinatorClaim` |
| A6-P12 | A2 + A6 (contingent on A9-P6 option iii) | Over-builds if REPLAY deferred | reframe to "signed schema + frozen format; REPLAY day-2" |
| A7-P4 | A7 (self) | Registry dispatch owner unpinned | pin via Invariant I-DIST-1 + IWYU-CI |
| A7-P6 | A7 (self) | `runtime::composition` sub-ns lacks ADR | record ADR-A7-R3 |
| A8-P3 | A1 (peer, 6.2.3 replay) | Histogram bucket array not region-pinned | pin to CONTROL in `modules/profiling.md` |
| A8-P4 | A6 (peer, cross-tenant replay) | Default `dump_state()` scope leaks tenants | scope to caller's `logical_system_id` |
| A8-P5 | A6 (peer) | `AlertRule.logical_system_id` missing | add field |
| A8-P6 | A8 (self, 8× replay) | Tie-breaker for "any-owner-deterministic merge" unspecified | add `min(node_id)` + skew cap |
| A10-P2a(=A5-P3) | A10 (self, 8× replay) | Cluster-wide fail-closed risk | pin scope to failed Pod; `cluster_view` bump |
| A10-P3 | A10 (self, 8× replay) | State rows incomplete | add `cluster_view`, `group_availability`, `remote_proxy_pool`, `outstanding_submission_set` |
| A10-P6 | A10 (self, 8× replay) | Single heartbeat thread saturates | shard per 32 peers, pool cap 4 |
| A10-P7 | A10 (self, 8× replay) | Silent P4 ceiling at `admission_shards=1` | add deployment cue |

## 3. Scenario replay coverage

| Scenario | §6 anchor | Replayed by | Outcome |
|---|---|---|---|
| 6.1.1 — single-node hierarchical | 15–30 | A1, A2, A4, A7, A9 | All hold modulo §2 amendments |
| 6.1.2 — distributed multi-node | 35–58 | A2 (canonical), A4, A6, A7, A10 | Step 7 resolved via A2-P7 third axis; 6.2.2 cross-tenant via A6-P2 |
| 6.1.3 — SPMD | 61–79 | A1, A3 (canonical), A8 | A3 amendments close SPMD precondition gaps |
| 6.2.1 — AICore hang | 83–98 | A1, A4, A5, A8 | All hold |
| 6.2.2 — coordinator/peer failure | 100–114 | A5 (canonical), A6 (attacker variant), A10 (8× Pods) | A5-P11 FSM authored; A6-P2 + A10-P2a scope fix |
| 6.2.3 — task-slot-pool exhaustion | 116–128 | A3, A5, A9 | A3-P7 + A3-P10 + A9-P5 scenario-text amendments |
| 6.2.4 — straggler/drain | 130–142 | A4, A5 | Hold with A5-P8 degradation amendments |

**Aggregate:** every scenario replayed produces either `holds` or minor amendments; **no replay forces withdrawal of any R2-agreed proposal**.

## 4. Blocking-objection register (R3)

| Raised by | Target | Blocking? | Status |
|---|---|---|---|
| (none) | (none) | — | **Empty: 0 new blocking objections filed in R3.** |

R2 carry-forward: A2's `blocking=true` flag on A7-P1 is a **hard-rule D6 citation in favor** (already resolved in R2; not a counter-vote).

## 5. Override register (R3)

| Raised by | Target | Status |
|---|---|---|
| (none) | (none) | **0 override requests raised in R3.** |

R2 carry-forward: A2's three override requests (on A1-P9, A9-P2, A9-P6) are **withdrawn** in R3 (per A2 R3 §6 self-declaration).

## 6. A1 hot-path veto ledger (Runtime Design Mode)

| Proposal | A1 veto status in R3 |
|---|---|
| — | **0 vetoes applied in R3.** A1 R3 §10 Status explicitly: "A1 veto applications this round: 0." |

All proposals marked HPI ≠ none were scenario-replayed and cleared by A1 against the 5 μs admission budget in `modules/scheduler.md §4.8`.

## 7. Conditional re-vote triggers (documented but not fired today)

| Trigger | Filed by | Target |
|---|---|---|
| A9-P6 final text lands option (i) instead of (iii) | A5 | A9-P6 → flip to disagree |
| A9-P6 final text lands option (i) instead of (iii) | A8 | A9-P6 → revert to disagree |
| A6 names concrete v1 forensic scenario needing REPLAY | A7 | A9-P6 → flip (iii) → (ii) |
| A2-P7 final text reintroduces any v1 interface declaration | A3, A5, A9 | A2-P7 → flip to abstain/disagree |

These are logged for Phase 5 landing-zone authors to respect; they are **not** pending objections.

## 8. Convergence calculation

| Check | Threshold | Value | Pass |
|---|---|---|---|
| Every proposal weighted-agree fraction | ≥ 2/3 | min = 1.00 (on disputed); all others unchanged ≥ 1.00 | ✓ |
| Open blocking objections | 0 | 0 | ✓ |
| A1 hot-path vetoes applied | 0 | 0 | ✓ |
| New proposals classified `agreed-new` | ≥ 2/3 weighted consensus from reviewers in whose aspect they land | 6/6 authored as owner + stress-tested by remaining reviewers with no objection | ✓ |
| Stress round held | Mandatory in Runtime Design Mode | Yes (this round) | ✓ |

**Result:** `CONVERGED` at Round 3. No Round 4 required.
