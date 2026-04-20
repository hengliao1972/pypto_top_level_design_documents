# Round 3 (Stress Round) — Vote Tally

- Run: `2026-04-18-171357`
- Target: `docs/pypto-runtime-design/`
- Mode: Runtime Design Mode (A1 weight = ×2, others = ×1)

## 1. Scope of Round 3

Round 3 is the **mandatory stress round**. Reviewers stress-attacked the 107 R2-agreed proposals and voted on the 3 R2-disputed proposals (`A2-P7`, `A9-P2`, `A9-P6`). No new full vote was taken on the 107 already-agreed proposals; instead, each reviewer surfaced stress findings which produced either (a) minor amendments to previously-agreed proposals, (b) six brand-new R3 proposals, or (c) re-vote triggers conditional on final text.

## 2. Votes on previously-disputed proposals

All three disputed proposals converged under weighted voting. A1 weight = 2; others = 1. Total weight = 11 per proposal.

| Proposal | Weighted Agree | Weighted Disagree | Abstain | Blocking | A1 veto | Final landing zone | Classification |
|---|---|---|---|---|---|---|---|
| A2-P7 | 11 / 11 = 1.00 | 0 | 0 | 0 | N | Q-record only in `09-open-questions.md`; no v1 interface | **agreed** |
| A9-P2 | 11 / 11 = 1.00 | 0 | 0 | 0 | N | Single `IEventLoopDriver` test seam + closed `DeploymentMode`/`PolicyKind` enums + appendix-of-future-extensions in `08-design-decisions.md` | **agreed** |
| A9-P6 | 11 / 11 = 1.00 | 0 | 0 | 0 | N | **Option (iii):** v1 FUNCTIONAL-only; `SimulationMode` enum open; REPLAY scaffolding declared but not factory-registered; ADR-011-R2 names triggers; A6-P12 re-scoped to "schema frozen; no v1 impl" | **agreed (option iii)** |

**All three disputes resolve at `agreed`.** All R2-carryover override requests (three, all filed by A2) are **withdrawn** in R3. Three reviewers (A5, A7, A8) document conditional re-vote triggers for A9-P6 if final text ever drifts to option (i); similarly three (A3, A5, A9) for A2-P7 if final text reintroduces a v1 interface slot. None of these trigger today.

## 3. Stress outcomes on previously-agreed proposals

Scanning R3 §8 of every reviewer: **0** proposals from the R2-agreed set flipped to disagree; **0** were withdrawn; **0** new blocking objections raised. All stress findings resolved into minor amendments.

| Proposal | Minor R3 amendment (1-line) |
|---|---|
| A1-P6 | Owner adds cold-start template-miss budget row (≤ 15 μs first-use) in §4.8.2 |
| A2-P1 | Narrow versioning obligation to multi-node wire messages + persistent artifacts (per A9) |
| A2-P5 | Cross-reference checklist added listing the headers that must receive version byte |
| A2-P6 | Demote `IDistributedProtocolHandler` to free function in `protocol.hpp`; defer abstract class to ADR + second backend |
| A2-P7 | Add third axis "transport-capability semantics" to Q-record text |
| A3-P4 | Extend edit sketch to include SPMD aggregation edge (parent notification on sub-task failure) |
| A3-P5 | Retain `spmd_index` on cancelled siblings in `ErrorContext.remote_chain` |
| A3-P7 | Add `SpmdIndexOOB`, `SpmdSizeMismatch`, `SpmdBlockDimZero` to precondition catalog |
| A3-P10 | Update 6.2.3 scenario text to `simpler.SchedulerError` (match full domain mapping) |
| A5-P2 | Introduce `breaker_auth_fail_weight=10` (see new A5-P13) |
| A5-P3 | Add `coordinator_liveness_timeout_ms < heartbeat_timeout_ms`, default `3 × heartbeat_interval_ms` |
| A5-P6 | Add normative clause in `modules/scheduler.md` pinning deadman write-elision |
| A5-P9 | DRY-fold `QUARANTINED` into `UNAVAILABLE + {Permanent, Quarantine(duration)}` policy variants (per A9 amendment) |
| A6-P2 | Add `coordinator_generation: uint64_t` to `HandshakePayload`; add `StaleCoordinatorClaim` reject rule |
| A6-P12 | Reframe to "signed envelope + frozen schema in v1; REPLAY engine in v2" |
| A7-P4 | Second-pass pin: Invariant **I-DIST-1** (distributed header non-includable from `transport/`), IWYU-CI enforced |
| A7-P6 | Record **ADR-A7-R3** freezing `runtime::composition` sub-namespace + promotion triggers |
| A8-P3 | Pin histogram bucket array to CONTROL region in `modules/profiling.md` |
| A8-P4 | Default `dump_state()` scope to caller's `logical_system_id`; unscoped needs `diagnostic_admin_token` |
| A8-P5 | Add `AlertRule.logical_system_id` field |
| A8-P6 | Add tie-breaker: `min(node_id)` in youngest all-online epoch; `skew_max_ns ≤ 100 µs` |
| A10-P2a (=A5-P3) | Pin "deterministic fail-fast" scope to failed Pod only; bump `cluster_view` generation |
| A10-P3 | Add rows `cluster_view`, `group_availability`, `remote_proxy_pool`, `outstanding_submission_set` |
| A10-P6 | Shard heartbeat thread per 32 peers, thread-pool cap = 4 |
| A10-P7 | Add deployment cue in `05-physical-view.md` (threshold `concurrent_submitters × cluster_nodes ≥ 64`) |

**Net:** 25 minor amendments across 107 proposals; all remain classified `agreed`. **HPI=none** on every amendment per A1 replay.

## 4. New R3 proposals

Six new proposals introduced (all HPI=none, classified `agreed-new` by owner + peer consensus in R3 reviews):

| id | Owner | Severity | Target | Summary | HPI |
|---|---|---|---|---|---|
| A4-R3-P1 | A4 | medium | `appendix-b-codebase-mapping.md` + `appendix-a-glossary.md` | Add `WorkerState` row + glossary entry (`READY, BUSY, DRAINING, RETIRED, FAILED`, + `UNAVAILABLE+{Permanent,Quarantine}` variants per A5-P9 amendment) | none |
| A4-R3-P2 | A4 | low | `06-scenario-view.md:91,93,95,108,111` | Unify Task terminal-failed spelling → canonical `COMPLETING → ERROR` (Task) + `FAILED` (Worker) | none |
| A5-P11 | A5 | medium | `modules/distributed.md §3.5` (new) | Unified peer-health FSM with states `{HEALTHY, SUSPECT, OPEN, HALF_OPEN, UNAVAILABLE(Quarantine), LOST, AUTH_REVOKED}`; co-owned by breaker + heartbeat + auth | none |
| A5-P12 | A5 | low | `modules/hal.md` + `modules/scheduler.md §5` | Name `ErrorCode::ResourceExhausted` as failure class for `BufferRef` staging alloc failure | none |
| A5-P13 | A5 | low | `modules/distributed.md` CB section + auth section cross-ref | Record `breaker_auth_fail_weight` (default 10) | none (single compile-constant read on slow-path) |
| A5-P14 | A5 | medium | `09-open-questions.md` | Add Q-record: "Orchestration-composed collective partial-failure → FailurePolicy mapping" (v2 ADR trigger post A9-P3) | none |

All six are accepted under runtime-mode rules (HPI=none trivially passes A1 veto test; no blocking objection).

## 5. Final tallies

| Classification | Count | List |
|---|---|---|
| Agreed (R2 → R3 confirmed) | 107 | unchanged from R2 |
| Agreed (R3 resolved from disputed) | 3 | A2-P7, A9-P2, A9-P6 |
| Agreed (new in R3) | 6 | A4-R3-P1, A4-R3-P2, A5-P11, A5-P12, A5-P13, A5-P14 |
| Disputed | 0 | — |
| Rejected | 0 | — |
| Overridden | 0 | — |
| Vetoed by A1 | 0 | — |
| Blocking objections open | 0 | — |
| **Total agreed proposals** | **116** | |

**Convergence criteria:** ≥ 2/3 weighted agree on every proposal (achieved 100 %), **no open blocking objections** (0), **no A1 veto** (0) → **CONVERGED**.

## 6. Convergence decision

**Status: `CONVERGED` at end of Round 3.** No Round 4 required. Proceed to Phase 5 (apply edits).
