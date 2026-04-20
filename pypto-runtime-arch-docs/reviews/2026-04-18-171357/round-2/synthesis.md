# Round 2 Synthesis

## Metadata

- **Round:** 2
- **Timestamp (UTC):** 2026-04-18 17:45:00
- **Runtime Mode:** yes
- **Reviewers that returned output:** A1, A2, A3, A4, A5, A6, A7, A8, A9, A10
- **Reviewers missing / failed:** none

Inputs for this synthesis:
- `round-2/extract.md` — consolidated revisions + vote matrix + override + merge register.
- `round-2/tally.md` — per-proposal classification with Runtime-Mode weighting.

## Classification Summary

| Status | Count | Proposal IDs |
|--------|-------|--------------|
| agreed | 107 | all proposals except the three below |
| disputed | 3 | A2-P7, A9-P2, A9-P6 |
| rejected | 0 | — |
| deferred | 0 | — |
| incomplete | 0 | — |

Highlights:
- **Zero blocking objections.** The two `blocking=true` rows on A7-P1 were both cast alongside `agree` votes invoking D6 (no dependency cycles) as a hard rule that *forces* cycle removal. They are hard-rule citations, not blocking objections. Classification treats `blocking=false` for convergence purposes and records `has_hard_rule_citation=true` in the debate log.
- **Zero A1 hot-path vetoes.** A1 voted `agree` (or was the owner) on every HPI-flagged proposal. Every hot-path-touching proposal carries a documented two-tier (fast-path / slow-path) bridge in the proposer's §5 amendment.
- **Three override requests filed, all by A2** — on A1-P9 (bitmask cap), A9-P2 (policy-enum collapse), A9-P6 (defer PERFORMANCE/REPLAY). A1 filed zero vetoes this round, so no override lifting is required — the overrides attach to peer disagreements on ordinary proposals, not to hot-path vetoes.

## Convergence Status

- **Status:** `pre-stress-converged`
- **Next action:** Round 3 is **mandatory** as a stress round (per SKILL.md §Phase 4). During round 3, each reviewer attacks the emerging consensus from their aspect's strongest position; at least one scenario from `06-scenario-view.md` must be replayed against each accepted proposal. Three `disputed` proposals (A2-P7, A9-P2, A9-P6) also need targeted engagement in round 3 — their owners' amendments (§5 in round-2 reviews) already propose amendments that would flip most dissenters; the stress round will verify.

## Proposal Catalog (final status at end of round 2)

110 proposals total. See `round-2/tally.md` for the full per-proposal table (weighted agree fractions).

| proposal_id | owner | co_owners | severity | hot_path_impact (post-amend) | status (end R2) | summary |
|-------------|-------|-----------|----------|------------------------------|------------------|---------|
| A1-P1 | A1 | — | H | none (was allocates) | agreed | Pre-size `producer_index`; forbid rehash |
| A1-P2 | A1 | A10 | H | blocks (bounded) | agreed | Cap DATA-mode lock hold; shard count LevelParam |
| A1-P3 | A1 | A6 | M | none | agreed | Function Cache LRU + HEARTBEAT presence |
| A1-P4 | A1 | A7 | M | relayouts (one-time doc) | agreed | Hot/cold `Task` split, normative in `modules/core.md §8` |
| A1-P5 | A1 | A3 | H | none | agreed | Latency budgets for SPMD + event-loop |
| A1-P6 | A1 | A10 | M | none (was extends) | agreed | Distributed payload hygiene (absorbed A10-P5) |
| A1-P7 | A1 | A8 | M | none | agreed | Per-thread local seq + offline merge |
| A1-P8 | A1 | — | M | none (was allocates) | agreed | Pre-size Outstanding/Ready; CONTROL placement |
| A1-P9 | A1 | — | L | relayouts (bounded; 2×64 fallback) | agreed | Bitmask-encode slot-type availability |
| A1-P10 | A1 | A8 | L | none | agreed | Profiling-overhead CI gate |
| A1-P11 | A1 | A3, A6 | L | none | agreed | Per-arg Python↔C budget + per-arg validation (absorbed A6-P8a) |
| A1-P12 | A1 | — | M | extends-latency (slow-path only) | agreed | `BatchedExecutionPolicy.max_batch` + tail budget |
| A1-P13 | A1 | A7 | M | extends-latency (>1 KiB slow-path only) | agreed | Bound `args_blob`; 1 KiB ring-slot fast path |
| A1-P14 | A1 | A10 | M | none | agreed | `producer_index` CONTROL placement + shard default (absorbed A10-P10) |
| A2-P1 | A2 | — | H | none | agreed | Version every public data contract |
| A2-P2 | A2 | — | M | none | agreed | `LevelOverrides`: closed-for-v1, schema-registered v2 |
| A2-P3 | A2 | — | M | none | agreed | Open extension-point enums; keep `DepMode` closed |
| A2-P4 | A2 | — | H | none | agreed | Migration & Transition Plan |
| A2-P5 | A2 | — | M | none | agreed | Interface Evolution & BC Policy |
| A2-P6 | A2 | — | M | none | agreed | `IDistributedProtocolHandler` abstract boundary; one concrete v1 backend, CRTP/`final` devirt |
| A2-P7 | A2 | — | M | none | **disputed** | Reserve async-policy extension seam |
| A2-P8 | A2 | — | L | none | agreed | Record intentional closures as known deviations |
| A2-P9 | A2 | A4, A8 | M | none | agreed | Versioned trace schema |
| A3-P1 | A3 | A4 | B | none | agreed | Add `ERROR` state (P1a blocker); optional `CANCELLED` (P1b medium) |
| A3-P2 | A3 | A9 | H | none | agreed | `std::expected<SubmissionHandle, ErrorContext>`; ADR |
| A3-P3 | A3 | A5 | H | none | agreed | Admission-path failure scenario |
| A3-P4 | A3 | — | H | none (precomputed successor) | agreed | Producer-failure → consumer propagation |
| A3-P5 | A3 | — | H | none | agreed | Sibling cancellation policy |
| A3-P6 | A3 | — | M | none | agreed | Requirement ↔ Scenario traceability matrix |
| A3-P7 | A3 | A1, A6 | M | none | agreed | Submission preconditions at Python↔C boundary |
| A3-P8 | A3 | — | M | bounded; FE_VALIDATED fast path | agreed | Cyclic `intra_edges` detection (debug / FE_VALIDATED fast path) |
| A3-P9 | A3 | — | M | none | agreed | SPMD index/size delivery contract |
| A3-P10 | A3 | A6 | M | none | agreed | Python exception mapping completeness |
| A3-P11 | A3 | — | L | none | agreed | `[ASSUMPTION]` marker at `complete_in_future` |
| A3-P12 | A3 | A5, A10 | M | none | agreed | `drain()` / `submit()` concurrency + distributed semantics |
| A3-P13 | A3 | A5, A8, A10 | M | none | agreed | Cross-node ordering assumptions (FIFO+idempotent+dup-detect) |
| A3-P14 | A3 | — | L | none | agreed | `COMPLETING`-skip transition at leaf |
| A3-P15 | A3 | — | L | none | agreed | Debug-mode NONE-dep-mode cross-check |
| A4-P1 | A4 | — | H | none | agreed | Canonicalize HAL enum member casing |
| A4-P2 | A4 | — | B | none | agreed | Fix broken anchor |
| A4-P3 | A4 | — | H | none | agreed | Fix invariant count prose |
| A4-P4 | A4 | — | M | none | agreed | Reorder Open Questions numerically |
| A4-P5 | A4 | A2 | M | none | agreed | Event-loop plumbing glossary (conditional on A9-P2/A9-P7) |
| A4-P6 | A4 | — | M | none | agreed | Cross-link views to ADRs |
| A4-P7 | A4 | — | L | none | agreed | Unify L0 component label |
| A4-P8 | A4 | A3 | L | none | agreed | Sync Appendix-B TaskState count (+ new states) |
| A4-P9 | A4 | A3 | L | none | agreed | Expand `Task State` glossary entry |
| A5-P1 | A5 | — | H | none | agreed | Exponential backoff + jitter on remote retries |
| A5-P2 | A5 | A6, A10 | H | none (thread-local TSC) | agreed | Per-peer circuit breaker |
| A5-P3 | A5 | A10 | H | none | agreed | v1 deterministic coordinator fail-fast (absorbed A10-P2a); v2 = decentralized roadmap ADR |
| A5-P4 | A5 | A3 | M | none | agreed | `idempotent: bool` on `TaskDescriptor` |
| A5-P5 | A5 | A8 | M | none | agreed | Chaos/fault-injection scenario matrix (co-own with A8-P7 seam) |
| A5-P6 | A5 | A8 | M | none | agreed | Scheduler watchdog (pair with A8-P4 `dump_state()`) |
| A5-P7 | A5 | — | M | none | agreed | `Timeout` on `IMemoryOps` async |
| A5-P8 | A5 | A8 | M | none | agreed | Degradation specs + alert rules |
| A5-P9 | A5 | — | L | none | agreed | QUARANTINED Worker state |
| A5-P10 | A5 | A3 | M | none | agreed | DS4 per-REMOTE_* idempotency |
| A6-P1 | A6 | — | H | none | agreed | Trust-boundary threat model |
| A6-P2 | A6 | A5 | H | none | agreed | Concrete node authentication in HANDSHAKE (mTLS cert-pinned v1; SPIFFE v2) |
| A6-P3 | A6 | A1 | H | extends (≤1 compare at entry; none on reject) | agreed | Bounded payload parsing (entry-gate guard) |
| A6-P4 | A6 | — | H | none (TCP only; RDMA untouched) | agreed | Default-encrypted TCP control; explicit RDMA exemption |
| A6-P5 | A6 | — | M | none (retirement-only rotation) | agreed | Scoped, revocable RDMA `rkey` |
| A6-P6 | A6 | — | M | none | agreed | Security audit trail |
| A6-P7 | A6 | — | M | none (startup-only; multi-tenant opt-in) | agreed | Function-binary attestation gated by `trust_boundary.multi_tenant` |
| A6-P8 | A6 | A1, A3 | H | none (absorbed into A1-P11) | agreed | A6-P8a absorbed into A1-P11; A6-P8b (byte-cap) remains A6-owned |
| A6-P9 | A6 | — | M | none | agreed | Enforce Logical System isolation in wire + code |
| A6-P10 | A6 | A8 | L | none | agreed | Capability-scoped log / trace sinks |
| A6-P11 | A6 | — | M | none | agreed | Gated `register_factory` |
| A6-P12 | A6 | — | M | none (moot if A9-P6 agreed; kept v1 only if REPLAY retained) | agreed **contingent** | Signed / schema-validated REPLAY trace |
| A6-P13 | A6 | — | M | none (gate behind multi-tenant) | agreed | Per-tenant submit rate-limit (multi-tenant deployments only) |
| A6-P14 | A6 | — | M | none | agreed | Key-material lifecycle ADR |
| A7-P1 | A7 | A2, A6 (both cite D6) | H | none | agreed | Break `scheduler/` ↔ `distributed/` cycle |
| A7-P2 | A7 | A9 | H | none | agreed | Split `ISchedulerLayer` into role interfaces (absorbed A9-P1) |
| A7-P3 | A7 | — | M | none | agreed | Invert `core/` ↔ `hal/` for handle types |
| A7-P4 | A7 | — | M | none | agreed | Move distributed payload structs to `distributed/` |
| A7-P5 | A7 | — | H | none | agreed | `distributed_scheduler` depends only on `ISchedulerLayer` |
| A7-P6 | A7 | — | M | none | agreed | Extract MLR + deployment parser |
| A7-P7 | A7 | — | L | none | agreed | Forward-decl contract |
| A7-P8 | A7 | — | M | none | agreed | Consolidate `ScopeHandle` ownership |
| A7-P9 | A7 | — | L | none | agreed | Dedupe Python `MemoryError` class names |
| A8-P1 | A8 | A5 | M | none | agreed | Injectable `IClock` |
| A8-P2 | A8 | A9 | M | none | agreed | Driveable event-loop (single `IEventLoopDriver`) |
| A8-P3 | A8 | A1 | M | none | agreed | Stats structs + latency histograms |
| A8-P4 | A8 | A5 | M | none | agreed | `dump_state()` diagnostic endpoint (pair with A5-P6) |
| A8-P5 | A8 | A5 | M | none | agreed | External alert-rules file; Prom/OTEL as opt-in deviation |
| A8-P6 | A8 | A3 | M | none | agreed | Distributed trace time-alignment contract |
| A8-P7 | A8 | A5 | M | none | agreed | `IFaultInjector` sim-only seam |
| A8-P8 | A8 | — | M | none (L2 opt-in) | agreed | Commit AICore in-core trace upload protocol |
| A8-P9 | A8 | — | L | none | agreed | Profiling drop/degraded as first-class alerts |
| A8-P10 | A8 | A6 | L | none | agreed | Structured KV logging primary surface |
| A8-P11 | A8 | — | M | none | agreed | HAL contract test suite (sim + onboard) |
| A8-P12 | A8 | A1 | L | none | agreed | Stable `PhaseId`s for Submission lifecycle |
| A9-P1 | A9 | A7 | M | none | agreed | Drop submit overloads (absorbed into A7-P2) |
| A9-P2 | A9 | — | H | none | **disputed** | Cut FULLY_SPLIT/SPLIT_DEFERRED + collapse pluggables |
| A9-P3 | A9 | — | H | none | agreed | Remove collectives from `IHorizontalChannel` (Q4 option A) + ADR |
| A9-P4 | A9 | — | M | none | agreed | Drop `SubmissionDescriptor::Kind` |
| A9-P5 | A9 | — | M | none | agreed | Unify admission enums (AdmissionStatus) |
| A9-P6 | A9 | — | H | none | **disputed** | Defer PERFORMANCE/REPLAY simulation |
| A9-P7 | A9 | A4 | M | none | agreed | Fold `SourceCollectionConfig` into `EventHandlingConfig` |
| A9-P8 | A9 | — | L | none | agreed | Move AICore companion-artifacts obligation out of design |
| A10-P1 | A10 | A1 | M | none | agreed | Default `producer_index` sharding policy |
| A10-P2 | A10 | A5 | H | none | agreed | v1 fail-fast (absorbed into A5-P3); v2 decentralize = roadmap-only ADR |
| A10-P3 | A10 | — | M | none | agreed | Per-data-element consistency model |
| A10-P4 | A10 | — | M | none | agreed | Stateful/stateless classification |
| A10-P5 | A10 | A1 | M | none | agreed | Per-peer REMOTE_SUBMIT projection (absorbed into A1-P6) |
| A10-P6 | A10 | — | M | none (dedicated heartbeat thread) | agreed | Faster peer-failure detection with hysteresis |
| A10-P7 | A10 | — | M | none (default `shards=1`) | agreed | Two-tier sharded TaskManager |
| A10-P8 | A10 | — | M | none | agreed | Single "Data & State Reference" §  in `07-cross-cutting-concerns.md` (absorbs P3+P4+P8) |
| A10-P9 | A10 | — | L | none | agreed | Gate `WorkStealing` × `RETRY_ELSEWHERE` |
| A10-P10 | A10 | A1 | M | none | agreed | `producer_index` cap/layout (absorbed into A1-P14) |

## Agreement Matrix

See `round-2/extract.md` section (B) for the full 10×110 matrix and `round-2/tally.md` for the per-proposal numeric classification. Summary cells (A=agree, D=disagree, ·=abstain, *=own, D!=blocking-disagree, V!=A1 veto, H!=hard-rule-citation-in-favor):

| proposal_id | A1 | A2 | A3 | A4 | A5 | A6 | A7 | A8 | A9 | A10 | agree_frac (weighted) | status |
|-------------|----|----|----|----|----|----|----|----|----|-----|-----------------------|--------|
| A1-P4  | * | A | A | A | A | · | A | A | D | A | 0.875 | agreed |
| A1-P9  | * | D | A | A | A | · | A | A | D | A | 0.750 | agreed |
| A1-P12 | * | A | · | A | A | A | A | A | D | A | 0.875 | agreed |
| A2-P7  | · | * | · | · | · | · | D | · | D | · | 0.500 | **disputed** |
| A9-P2  | · | D | · | · | · | · | · | D | * | · | 0.600 | **disputed** |
| A9-P6  | · | D | · | · | · | D | · | D | * | · | 0.625 | **disputed** |
| A7-P1  | A | A(H!) | A | A | A | A(H!) | * | A | A | A | 1.000 | agreed |

(All other 103 proposals land at `weighted_agree_frac ≥ 0.83` with unanimous or near-unanimous agree; full rows in `tally.md`.)

## Conflict Register

Only the three `disputed` proposals have meaningful conflict to carry into round 3.

### A2-P7 — reserve async-policy extension seam

- **Owner:** A2
- **Dissenters:** A7 (disagree; SRP/D3), A9 (disagree; G2 YAGNI)
- **Abstentions:** A3, A4, A5, A6, A8, A10 (six — all say this is an A2-vs-A9 debate that does not affect their rubrics)
- **Blocking objections:** none
- **Tension pair:** A2 vs A9
- **Synthesizer observation:** A2's R2 amendment **already converts the proposal into a Q-record** (no new interface in v1; just `09-open-questions.md` entry naming the two future policy axes). The dissenters' core concern (speculative interface) is addressed by the amendment. Voters voted on the R1 wording. R3 will likely flip both dissenters to agree once the Q-record framing is explicit.

### A9-P2 — collapse policy-enum/interface pluggability

- **Owner:** A9
- **Dissenters:** A2 (disagree; OCP/E1 regression; override_request=true), A8 (disagree; X5 DfT)
- **Abstentions:** A3, A4, A5, A6, A10 (five)
- **Blocking objections:** none
- **Tension pair:** A9 vs A2, A9 vs A8
- **Synthesizer observation:** A9's R2 amendment commits to a **single `IEventLoopDriver` test-only seam** with `step()` + `RecordedEventSource`, keeps closed enums for the deployment modes, and adds an appendix listing future extension interfaces. This directly answers A8's X5 DfT concern (single test double survives) and partially answers A2's OCP concern (closed enums + ADR-named v2 extensions). A2's override-request asks for the future-extension list to be ADR-recorded, which the amendment also delivers. **Expected to flip at least A8 and possibly A10 in R3.** A2's stance may still hang on whether "appendix-listed future extensions" counts as a valid OCP seam.

### A9-P6 — defer PERFORMANCE/REPLAY simulation to post-v1

- **Owner:** A9
- **Dissenters:** A2 (disagree; E5 migration plan; override_request=true), A6 (disagree; keep REPLAY for forensics), A8 (disagree; X5 DfT "reproducible failures")
- **Abstentions:** A4, A5 (two)
- **Blocking objections:** none
- **Tension pair:** A9 vs A2, A9 vs A6, A9 vs A8
- **Synthesizer observation:** A9's R2 amendment adopts **ADR-011-R2 with named triggers**, keeping `SimulationMode` enum open and moving PERFORMANCE and REPLAY to `09-open-questions.md` with trigger conditions. This addresses A2 (E5 migration plan present) but does NOT address the substantive "retain REPLAY in v1" preference of A6/A8. The stress round must arbitrate between three concrete options:
  - **(i)** A9's amendment as written (defer REPLAY + PERFORMANCE to v2 with triggers)
  - **(ii)** Hybrid — ship FUNCTIONAL + REPLAY in v1; defer only PERFORMANCE (A6/A8 preference)
  - **(iii)** Ship FUNCTIONAL only but keep REPLAY enum + scaffolding so A6-P12 and A8 debugging harness can land on day 2

The synthesizer's recommended landing zone for R3: option **(iii)** — the enum stays open (A2 satisfied), the scaffolding stays (A6 forensic pathway stays viable), the full REPLAY implementation is deferred (A9 YAGNI satisfied), and A6-P12 becomes "signed schema + format frozen at ADR-011-R2 time" rather than "fully implemented in v1". Round 3 is free to pick a different option if the stress-round evidence demands.

## Open Disputes — Next-Round Focus

### A2-P7
- **Required engagement:** A2 (explicit Q-record wording), A9 (vote-flip confirmation), A7 (SRP concern resolved by "no interface" framing)
- **Evidence needed:** A2's proposed Q text ready to paste into `09-open-questions.md`
- **Expected R3 outcome:** agreed

### A9-P2
- **Required engagement:** A2 (scope of "appendix future-extensions" OCP coverage), A8 (confirm `IEventLoopDriver` + `step()` + `RecordedEventSource` meets X5 needs), A9 (final amendment text)
- **Evidence needed:** explicit list of "future-extensions appendix" entries; demonstration that single test double covers the DfT rubric
- **Expected R3 outcome:** agreed (pending A2's override withdrawal)

### A9-P6
- **Required engagement:** A6 (which v1 forensic workflows require REPLAY data), A8 (whether A8-P2 driveable event-loop + `RecordedEventSource` subsumes REPLAY for DfT), A9 (final option choice)
- **Evidence needed:** concrete v1 forensic scenario requiring REPLAY payload (A6 owns), OR decision to defer
- **Expected R3 outcome:** agreed under option (iii) — the synthesizer's recommended middle ground

## Semantic-Duplicate Merge Register (final state)

All six round-1-proposed merges accepted by every reviewer.

| Merge | Status | Notes |
|-------|--------|-------|
| A1-P6 ← A10-P5 | **applied** | A10-P5 is now an alias for A1-P6 |
| A1-P14 ← A10-P10 | **applied** | A10-P10 is alias |
| A5-P3 ← A10-P2 | **applied (as split)** | A10-P2a absorbed; A10-P2b remains as v2 roadmap-only ADR |
| A5-P6 ↔ A8-P4 | **paired** | Both retain their IDs; watchdog evidence surface bound to `dump_state()` |
| A7-P2 ← A9-P1 | **applied** | A9-P1 absorbed; ISP role-split + single `submit()` |
| A1-P11 ← A6-P8a | **applied (partial)** | Only structural-validation half absorbed; A6-P8b (byte-cap) remains |

## Stress-Round Results

Omitted — round 3 only.

## Notes for the parent

- **Runtime-mode hot-path audit:** pass. A1 voted agree on every HPI-flagged proposal; every proposer supplied a two-tier bridge in their §5 amendment. `round-2/tally.md` documents each case.
- **A1 veto applications this round:** 0.
- **Non-A1 override requests:** 3 (all by A2). None required lifting because they attach to peer dissents on ordinary proposals, not to A1 vetoes.
- **Deduplication actions:** 6 merges applied cleanly. All absorbed IDs (A10-P5, A10-P10, A10-P2a, A6-P8a, A9-P1) remain in the catalog with status "absorbed into <parent>" so R3 reviewers and Phase-5 edit application can trace both endpoints.
- **Round 3 launch conditions:** give every reviewer (a) this synthesis, (b) `round-2/extract.md`, (c) `round-2/tally.md`, and (d) the explicit instruction to attack the emerging consensus from their aspect's strongest position, with at least one scenario from `06-scenario-view.md` replayed against each accepted proposal. Reviewers must also engage with the three disputed items — the synthesis's recommended landing zones for A2-P7, A9-P2, A9-P6 should be evaluated and confirmed or overturned.
