# Residual Risks — `pypto-runtime-design` Review Run `2026-04-18-171357`

This document lists items the review deliberately left open, items with conditional re-vote triggers, and risks surfaced during stress testing that require human follow-up. Nothing here **blocks** adoption of the 116 agreed proposals — every item has either a concrete trigger, a downstream ADR, or an open question already filed in `09-open-questions.md`.

## 1. Rejected proposals

None. The review rejected no proposals.

## 2. Deferred proposals

None. All 116 proposals are classified `agreed`.

## 3. Conditional re-vote triggers (non-blocking)

Recorded by reviewers during Round 3. These do not represent current objections. They tell the next maintainer which proposal entries must be treated as sticky and re-opened if the listed trigger fires.

| Trigger | Filed by | Target | Effect if fired |
|---|---|---|---|
| Phase 5 final text for A9-P6 lands option (i) (full deferral, enum closed) instead of option (iii) | A5 | A9-P6 | flip to `disagree`; re-open ADR-011-R2 |
| Phase 5 final text for A9-P6 lands option (i) | A8 | A9-P6 | revert to `disagree`; re-open ADR-011-R2 |
| A6 names a concrete v1 forensic scenario requiring REPLAY data during any R4+ work | A7 | A9-P6 | flip from option (iii) to (ii) |
| Phase 5 final text for A2-P7 reintroduces any v1 interface declaration | A3, A5, A9 | A2-P7 | flip to `abstain`/`disagree`; remove class and keep only Q-record |
| A5-P11 peer-health FSM ownership is not co-published in `modules/distributed.md §3.5` with CB+heartbeat+auth co-owners | A5 | A5-P11 | re-open 6.2.2 livelock analysis |

**Status after Phase 5:** The applied edits honor every trigger (A9-P6 landed option (iii), A2-P7 is Q-record only, A5-P11 §3.5 is co-owned). No trigger has fired.

## 4. Phase-1 survivals with long-horizon risks

| ID | Residual risk | Mitigation already in place | Re-open trigger |
|---|---|---|---|
| A1-P6 | Cold-start template-miss 15 μs budget assumes template binary cached at Node; first-cold-miss on a never-seen platform variant is unbudgeted | Doc cross-ref in §4.8.2 + runtime alert on uncached miss | Measured cold-miss latency > 15 μs in any CI profile |
| A5-P11 | Unified FSM requires co-ownership by CB + heartbeat + auth; divergent implementations reintroduce livelock | ADR-018 + Invariant I-DIST-1 IWYU-CI + `modules/distributed.md §3.5` | Any new state transition is added without updating ADR-018 |
| A6-P12 | "Signed envelope + frozen schema v1; REPLAY engine v2" defers forensic reconstruction; if a production incident needs replay before v2 lands, only signed envelope is available | Q17 in `09-open-questions.md`; ADR-011-R2 triggers | First incident where signed envelope proves insufficient |
| A10-P6 | Heartbeat shard cap = 4 thread-pool assumes ≤128 peers/node; scaling beyond requires Q18 resolution | Q18 in `09-open-questions.md` | Deployment topology exceeds 128 peers/node |
| A10-P7 | Admission-shard default = 1; silent P4 ceiling for operators who miss the deployment cue | Deployment cue in `05-physical-view.md` + ADR-019 | Any benchmark exceeding `concurrent_submitters × cluster_nodes ≥ 64` ships with `admission_shards=1` |
| A2-P1 | Versioning narrowed in R3 to wire messages + persistent artifacts; in-process data contracts are not versioned | Appendix-C BC policy + per-interface table | Any in-process contract crosses a process boundary in a future release |
| A2-P6 | IDistributedProtocolHandler demoted to free function in v1; re-promotion to abstract class requires ADR-015 re-open when second backend lands | ADR-015 + Invariant I-DIST-1 | Any second concrete backend (e.g. TCP fallback) is proposed |
| A3-P1 | TaskState canonical spelling + transitions (ADR-016) — if any module emits legacy `COMPLETED(error)` spelling, cross-view consistency breaks | ADR-016 + `06-scenario-view.md` unified spelling; D7/V5 spot-check passed | Any new module introduces legacy spelling |
| A5-P2 + A5-P13 | `breaker_auth_fail_weight=10` default is conservative; real-world credential-churn patterns may require tuning | Config field present; A8-P5 AlertRule covers CB oscillation | Any `peer_circuit_breaker_trip` alert fires under normal credential rotation |
| A5-P3 | `coordinator_liveness_timeout_ms < heartbeat_timeout_ms` default `3 × heartbeat_interval_ms` may be too aggressive on lossy fabrics | Policy externalized per A5-P8 | Any false-positive coordinator demote in chaos tests |
| A7-P4 / A7-P6 | ADR-014 + ADR-015 freeze `runtime::composition` + distributed header boundary; IWYU-CI enforces but developer discretion can still regress the invariant | IWYU-CI lint + ADR promotion triggers | Any new consumer outside the approved list imports composition symbols |
| A8-P6 | Clock-sync merge tie-breaker `min(node_id)` in youngest all-online epoch; `skew_max_ns ≤ 100 μs` assumed | IClockSync::offset_ns() in `modules/hal.md §2.8` | Measured skew exceeds 100 μs in production |

## 5. Cross-aspect coupling left for downstream work

- **A5-P11 ↔ A10-P6**: Peer-health FSM state transitions interact with heartbeat sharding cap. If heartbeat shard cap grows beyond 4, SUSPECT → OPEN transition rate must be re-measured.
- **A6-P2 ↔ A10-P2**: `coordinator_generation` and `cluster_view.generation` must increment atomically; the apply layer uses a single ADR-020 write barrier, but any future handoff path that bypasses the barrier re-opens the stale-coordinator attack.
- **A2-P7 ↔ A6-P4 ↔ A6-P5**: Async-policy extension seam (Q15) must preserve both TLS resistance and scoped-revocable rkey semantics in any future transport. The third axis added in R3 ("transport-capability semantics") flags this but does not constrain the v2 design.
- **A3-P4/P5/P7 ↔ A9-P4**: SPMD precondition catalog interacts with Kind-discriminator removal. The apply layer preserves sibling cancellation via `spmd` field presence, but any future SPMD-variant (e.g. heterogeneous tiles) requires a new Q.

## 6. Documentation-consistency residuals

- **Glossary continuation section** (`appendix-a-glossary.md`) — 34 new terms landed in a continuation block rather than alphabetically interleaved with the existing v1 terms. Low-risk; recommend a follow-up doc-hygiene pass to interleave.
- **`error.md` module** — The final-proposal catalog maps some proposals (A2-P1 versioning of error struct, A3-P2 `expected<>`, A3-P7 precondition sub-codes, A3-P10, A3-P12, A5-P12, A7-P9) to `modules/error.md`. Phase 5 apply did not touch this file (owner was not in the assigned-targets cluster). Recommend a follow-up: propagate the relevant `[UPDATED:]` markers from `core.md`/`scheduler.md`/`bindings.md` into `error.md` to restore D7/V5 consistency. Risk is low — the source-of-truth normative text lives in the other modules.
- **Appendix-C (new file)** — Freshly created; has not yet been cross-referenced from every relevant source doc. Low risk (D7 soft rule).

## 7. Runtime-Design-Mode audit

- **A1 weight ×2** applied every round; recorded in `round-2/debate-log.md` and `round-3/debate-log.md`.
- **A1 hot-path veto applications:** 0 across all three rounds. HPI tags on all agreed proposals recorded in `final/final-proposal.md` section 1.
- **Non-A1 overrides:** 0 succeeded (A2 filed 3 overrides in R2, all withdrawn in R3).
- **Scenario replay:** Every scenario in `06-scenario-view.md` (§6.1.1, §6.1.2, §6.1.3, §6.2.1, §6.2.2, §6.2.3, §6.2.4) replayed in Round 3; results recorded in `round-3/debate-log.md §3`.

## 8. Triage guidance

- **Nothing in this file blocks merge.** The 116 agreed edits are consistent and can be shipped.
- **Next scheduled review trigger:** when any item in §3 fires, or when any of the four new open questions (Q15-Q18) receive a concrete proposal for resolution, file a new run under `docs/pypto-runtime-design/reviews/` linking back to this run via `run-manifest.md`.
- **ADR evolution:** ADR-011-R2, ADR-014, ADR-015, and ADR-018 each carry their own promotion triggers; honor those before reopening debate.
