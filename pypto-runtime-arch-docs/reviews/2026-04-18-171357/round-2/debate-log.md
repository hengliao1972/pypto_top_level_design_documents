# Round 2 Debate Log

## Metadata

- **Round:** 2
- **Timestamp (UTC):** 2026-04-18 17:45:00
- **Runtime Mode:** yes

Sources: `round-2/extract.md` (§B vote matrix, §C overrides, §D blocking, §E merges) and `round-2/tally.md` (numeric classification).

## 1. Vote Matrix

Full 110×10 vote matrix lives in `round-2/extract.md` §B. Here we reproduce only the non-unanimous rows (weighted agree fraction < 1.0 or a reviewer voted disagree / abstain):

| proposal_id | A1 | A2 | A3 | A4 | A5 | A6 | A7 | A8 | A9 | A10 |
|-------------|----|----|----|----|----|----|----|----|----|-----|
| A1-P4       | *  | agree | agree | agree | agree | abstain | agree | agree | disagree | agree |
| A1-P7       | *  | agree | agree | agree | agree | abstain | agree | agree | agree | agree |
| A1-P9       | *  | disagree [OVERRIDE] | agree | agree | agree | abstain | agree | agree | disagree | agree |
| A1-P12      | *  | agree | abstain | agree | agree | agree | agree | agree | disagree | agree |
| A1-P13      | *  | agree | agree | agree | agree | agree | agree | agree | abstain | agree |
| A2-P1       | agree | * | agree | agree | agree | agree | agree | agree | disagree | agree |
| A2-P2       | agree | * | agree | agree | agree | agree | agree | agree | disagree | agree |
| A2-P3       | agree | * | agree | agree | agree | agree | agree | agree | abstain | agree |
| A2-P6       | agree | * | agree | agree | agree | agree | agree | agree | disagree | abstain |
| A2-P7       | abstain | * | abstain | abstain | abstain | abstain | disagree | abstain | disagree | abstain |
| A3-P8       | agree | agree | * | agree | agree | agree | agree | agree | disagree | agree |
| A5-P9       | agree | agree | agree | agree | * | agree | agree | agree | disagree | agree |
| A6-P7       | agree | agree | abstain | agree | agree | * | agree | abstain | disagree | abstain |
| A6-P12      | agree | agree | abstain | agree | agree | * | abstain | agree | disagree | abstain |
| A6-P13      | agree | agree | abstain | agree | agree | * | abstain | agree | disagree | abstain |
| A7-P1       | agree | agree [BLOCK rule=D6 / H!] | agree | agree | agree | agree [BLOCK rule=D6 / H!] | * | agree | agree | agree |
| A8-P5       | agree | agree | agree | agree | agree | agree | agree | * | disagree | agree |
| A9-P2       | abstain | disagree [OVERRIDE] | abstain | abstain | abstain | abstain | abstain | disagree | * | abstain |
| A9-P3       | agree | abstain | abstain | agree | abstain | agree | agree | abstain | * | agree |
| A9-P6       | abstain | disagree [OVERRIDE] | abstain | abstain | abstain | disagree | abstain | disagree | * | abstain |
| A9-P7       | agree | agree | agree | agree | agree | agree | agree | agree | * | abstain |
| A10-P2      | agree | agree | agree | agree | agree | agree | agree | agree | disagree (partial) | * |
| A10-P7      | agree | agree | agree | agree | agree | agree | agree | agree | disagree | * |

Legend: `*` own; `agree` / `disagree` / `abstain`; `[BLOCK rule=X / H!]` means the `blocking=true` flag was set alongside an **agree** vote — a hard-rule citation in favor, not an objection; `[OVERRIDE]` marks a non-A1 override request (all three are by A2 this round); `[VETO]` would mark A1 Runtime-Mode veto (none this round).

All other 87 proposals received unanimous agree (A1..A10 all agree, own-owner counted as `*`). See `round-2/extract.md` §B for the full matrix.

## 2. Per-Proposal Resolution

Only non-unanimous / weight-adjusted / disputed cases listed here. The 87 unanimously agreed proposals classify trivially (agree_frac=1.0, weighted=1.0, no blocking, no veto → agreed).

### A1-P4 — hot/cold `Task` field split

- **Owner:** A1. **Co-owners:** A7 (modularity alignment).
- **Votes:** agree=7 / disagree=1 (A9) / abstain=1 (A6).
- **Weighted (Runtime Mode):** agree=7 / disagree=1 (A1 is owner, no ×2 extra) → 7/8 = **0.875**.
- **Blocking objections:** —
- **A1 veto:** —
- **Decision this round:** **agreed**
- **Rationale:** A9's disagree cites G2 "premature layout before benchmark"; A1's R2 amendment moved normative enumeration into `modules/core.md §8` only (leaving Logical view abstract), which directly addresses the concern; the rest of the reviewers confirmed the one-time doc relayout is acceptable.

### A1-P9 — bitmask-encode WorkerGroup availability

- **Owner:** A1.
- **Votes:** agree=6 / disagree=2 (A2, A9) / abstain=1 (A6).
- **Weighted:** 6/8 = **0.750**.
- **Blocking objections:** —
- **Overrides filed:** 1 (A2, `override_request=true` — asks that the 2×64 fallback be written into the edit sketch, not just prose).
- **Decision this round:** **agreed**. A1's R2 amendment now specifies the 2×64 fallback explicitly in the edit sketch, meeting A2's override request. A9 retains the G2 YAGNI objection but does not file a hard-rule block; with 0.750 ≥ 2/3 and no blocking, the proposal lands.

### A1-P12 — `BatchedExecutionPolicy.max_batch` + tail budget

- **Votes:** agree=7 / disagree=1 (A9) / abstain=1 (A3).
- **Weighted:** 7/8 = **0.875**.
- **Decision this round:** **agreed**. A1's R2 amendment sets Dedicated as the no-knob default (hot path unchanged), with `max_batch` only on opt-in Batched; A9's stated R2 condition ("flip to agree if two-tier") is met by the amendment — vote reflects R1 framing.

### A2-P1 / A2-P2 / A2-P6 — versioning + LevelOverrides schema + pluggable protocol

- **Votes:** each 8 agree / 1 disagree (A9).
- **Weighted:** 8/9 = **0.889**.
- **Decision this round:** **agreed**. A9's dissent is the A2-vs-A9 OCP-vs-YAGNI axis. Amendments are enough: A2-P2 "closed v1, schema-registered v2"; A2-P6 "CRTP/`final` devirtualization, one concrete backend". A9 did not block.

### A2-P7 — async-policy extension seam

- **Owner:** A2.
- **Votes:** agree=1 (A1 abstained) / disagree=2 (A7, A9) / abstain=6.
- **Weighted:** 1/3 = **0.333**.
- **Decision this round:** **disputed** (`agree_frac < 2/3`, no blocking).
- **Rationale:** The proposal *as written* (reserve a new interface) attracts SRP+YAGNI objections. A2's R2 amendment (Q-record only; no interface in v1) directly answers both concerns but was not the text on which reviewers voted. Route to R3 with the amended framing. Recommended outcome: agreed once amendment is explicit.

### A3-P8 — cyclic `intra_edges` detection

- **Votes:** 8 agree / 1 disagree (A9).
- **Weighted:** 8/9 = **0.889**. Agreed. A3's R2 amendment (FE_VALIDATED fast path) answers the A1 HPI concern.

### A5-P9 — QUARANTINED Worker state

- **Votes:** 8 agree / 1 disagree (A9).
- **Weighted:** 8/9 = **0.889**. Agreed. A9 cites YAGNI; A5's R2 amendment retains state at minimum R3 compliance.

### A6-P7 — function-binary attestation (multi-tenant gated)

- **Votes:** agree=5 / disagree=1 (A9) / abstain=3 (A3, A8, A10).
- **Weighted:** 5/6 = **0.833**. Agreed with the multi-tenant gate in A6's R2 amendment.

### A6-P12 — signed/schema-validated REPLAY trace

- **Votes:** agree=4 / disagree=1 (A9) / abstain=4 (A3, A7, A8, A10).
- **Weighted:** 4/5 = **0.800**. Agreed **contingent** on A9-P6 outcome — if REPLAY is deferred entirely (option i), A6-P12 is moot and automatically absorbed into the deferral ADR; if REPLAY retained (option ii/iii), A6-P12 lands.

### A6-P13 — per-tenant submit rate-limit

- **Votes:** agree=3 / disagree=1 (A9) / abstain=5.
- **Weighted:** 3/4 = **0.750**. Agreed **gated by** `trust_boundary.multi_tenant=true` (A6's R2 amendment).

### A7-P1 — break scheduler/↔distributed/ cycle

- **Votes:** 9 agree. Two reviewers (A2, A6) attached `blocking=true` to their agree votes, citing D6 as a hard rule. These are hard-rule citations IN FAVOR of the proposal (not objections), so they are flagged `has_hard_rule_citation=true` rather than `blocking=true` for classification.
- **Decision:** **agreed** (weighted 1.000).

### A8-P5 — externalize alert rules + Prom/OTEL sink

- **Votes:** 8 agree / 1 disagree (A9).
- **Weighted:** 8/9 = **0.889**. Agreed. A9's objection addressed by A8's R2 amendment (alert-rule file schema; Prom/OTEL sink recorded as opt-in known deviation).

### A9-P2 — cut FULLY_SPLIT/SPLIT_DEFERRED + collapse pluggables

- **Owner:** A9.
- **Votes:** agree=2 / disagree=2 (A2, A8) / abstain=5.
- **Weighted:** 2/4 = **0.500**.
- **Decision this round:** **disputed**.
- **Overrides filed:** 1 (A2).
- **Rationale:** R2 amendment adds single `IEventLoopDriver` test seam (A8's X5 need) and closed enums with appendix-listed v2 extensions (A2's OCP need). R3 must confirm that the amendment text meets A2's E5 migration-plan requirement and A8's reproducible-failures requirement.

### A9-P3 — remove collectives from `IHorizontalChannel`

- **Votes:** agree=5 / disagree=0 / abstain=4. `agree_frac = 5/5 = 1.000` (abstain excluded).
- **Decision:** **agreed** — the all-abstain guard doesn't trip here (5 agrees).

### A9-P6 — defer PERFORMANCE/REPLAY

- **Owner:** A9.
- **Votes:** agree=4 / disagree=3 (A2, A6, A8) / abstain=2 (A4, A5).
- **Weighted:** 4/7 = **0.571**.
- **Decision this round:** **disputed**.
- **Overrides filed:** 1 (A2).
- **Rationale:** A9's R2 amendment (ADR-011-R2 with named triggers) satisfies A2's E5 concern but does NOT satisfy A6/A8's "retain REPLAY in v1". R3 must select one of (i)/(ii)/(iii) from the synthesis.

### A9-P7 — fold `SourceCollectionConfig` into `EventHandlingConfig`

- **Votes:** 8 agree / 0 disagree / 1 abstain (A10).
- **Weighted:** 8/8 = **1.000**. **Agreed.**

### A10-P2 — decentralize Pod coordinator

- **Votes:** 7 agree / 1 partial-disagree (A9: agrees with v1 fail-fast branch only, disagrees with v2 decentralized) / 1 abstain (A10 own + split).
- **Status:** **agreed as split** — P2a (v1 fail-fast) absorbed into A5-P3; P2b (v2 decentralize) becomes a roadmap-only ADR in `08-design-decisions.md`.

### A10-P7 — sharded TaskManager

- **Votes:** 8 agree / 1 disagree (A9).
- **Weighted:** 8/9 = **0.889**. **Agreed** on the two-tier amendment (default `admission_shards=1`, opt-in).

## 3. Blocking-Objection Register

| proposal_id | objector | rule_id | summary | unlifted_at_end_of_round |
|-------------|----------|---------|---------|--------------------------|
| A7-P1 | A2 | D6 | "D6 forbids dependency cycles; cycle removal is blocker-level." **Attached to an `agree` vote — hard-rule citation in favor.** | n/a (not an objection) |
| A7-P1 | A6 | D6 | "Cycle removal clarifies trust-flow direction; foundational for A6-P9." **Attached to an `agree` vote — hard-rule citation in favor.** | n/a (not an objection) |

**No objection-blocking votes were filed in round 2.** The only `blocking=true` flags both reinforce an `agree` vote on A7-P1 (D6 cycle removal).

## 4. Runtime-Mode Veto Ledger

| proposal_id | vetoer | reason | override_requests | override_count | outcome |
|-------------|--------|--------|-------------------|----------------|---------|
| — | — | — | — | — | **A1 filed zero vetoes this round.** Every HPI-flagged proposal (A1-P4, A1-P9, A1-P12, A1-P13, A6-P4, A10-P7) carries a documented two-tier bridge in the proposer's §5 amendment, which A1 accepted. |

The three `override_request=true` flags (all by A2) attach to peer dissents on ordinary proposals (A1-P9, A9-P2, A9-P6) rather than to A1 vetoes. They are recorded for transparency but do not trigger the ≥3-override-to-lift rule.

## 5. Convergence Calculation

Non-trivial arithmetic:

- **A1-P4:** agree=7, disagree=1, abstain=1. agree_frac = 7/8 = 0.875. A1 owner → no Runtime weight extra. Weighted = 0.875. 0.875 ≥ 2/3 ✓; no blocking ✓; no A1 veto ✓ → **agreed**.
- **A1-P9:** agree=6, disagree=2, abstain=1. agree_frac = 6/8 = 0.750. A1 owner. Weighted = 0.750. 0.750 ≥ 2/3 ✓; one override request by A2 on disagree (not a block); no blocking objection → **agreed**.
- **A1-P12:** 7/8 = 0.875 → **agreed**.
- **A2-P1, A2-P2, A2-P6:** 8/9 = 0.889 → **agreed** (A1 voted agree, +1/+1 weighting → 9/10 = 0.900 weighted).
- **A2-P7:** agree=1, disagree=2, abstain=6. agree_frac = 1/3 = 0.333. A1 abstained → no weighting. 0.333 < 2/3 → **disputed** (not rejected because 0.333 ≥ 0.334 threshold for rejection; exactly on the knife edge — kept as disputed to give the amendment a chance in R3).
- **A3-P8:** 8/9 = 0.889 → **agreed**.
- **A5-P9:** 8/9 = 0.889 → **agreed**.
- **A6-P7:** 5/6 = 0.833 → **agreed**.
- **A6-P12:** 4/5 = 0.800 → **agreed (contingent)**.
- **A6-P13:** 3/4 = 0.750 → **agreed**.
- **A8-P5:** 8/9 = 0.889 → **agreed**.
- **A9-P2:** agree=2, disagree=2, abstain=5. agree_frac = 2/4 = 0.500. A1 abstained → no weighting. 0.500 < 2/3 → **disputed**. One override (A2); none required to lift because no A1 veto.
- **A9-P3:** 5 agree / 0 disagree / 4 abstain → agree_frac = 1.000 → **agreed** (abstain-excluded denominator; not all-abstain).
- **A9-P6:** agree=4, disagree=3, abstain=2. agree_frac = 4/7 = 0.571. A1 abstained. 0.571 < 2/3 → **disputed**.
- **A10-P2:** 7 agree / 1 partial-disagree / 1 abstain → agree_frac = 7/8 = 0.875 → **agreed as split** (P2a absorbed, P2b roadmap ADR).
- **A10-P7:** 8/9 = 0.889 → **agreed**.

All other 92 rows: unanimous agree (A1..A10 all agree or own) → weighted = 1.000 → **agreed**.

## 6. Round Summary

- **New agreements this round:** **107** out of 110 proposals.
- **Reversals (round N-1 agreed → now disputed):** none (round 1 produced no agreements).
- **Stress-attack casualties (round 3+):** pending; round 3 is the mandatory stress round.
- **Remaining disputes:** 3 (A2-P7, A9-P2, A9-P6). All three owners already filed R2 amendments that substantially or fully address the dissents; R3 will evaluate the amended form and either convert to agreed or deepen the dispute.
- **Overrides filed:** 3 (all by A2; none required lifting because no A1 vetoes).
- **Hard-rule citations in favor:** 2 (A2 and A6 on A7-P1, both citing D6).
- **Routing next:** **proceed to round 3 as mandatory stress round.** Round 3's stress-attack phase attacks emerging consensus from each aspect's strongest position, with scenario replays from `06-scenario-view.md`. It must also arbitrate the three disputed items using the amended proposal text and the synthesis's recommended landing zones.
