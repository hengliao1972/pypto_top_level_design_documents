# Round 2 Vote Tally

Source: `round-2/extract.md` §(B) vote matrix (110 proposals × 10 reviewers).

Conventions:
- `own` rows do not count toward agree/disagree/abstain.
- `missing` and `abstain (implicit)` rows are counted as abstain. (No `missing` rows occurred in round 2 — every reviewer submitted an explicit row for every proposal.)
- `blocking` column records **objection-blocking** only. The two `blocking=true` rows on **A7-P1** (A2, A6) were both cast alongside `agree` votes invoking hard-rule D6 (no dependency cycles). Per the brief, those are reclassified as `has_hard_rule_citation=true` and **not** counted as blocking objections for classification purposes. Net result: zero blocking objections in round 2.
- `override_reqs` counts rows marked `override_request=true`. Exactly three were filed this round (all by A2 on A1-P9, A9-P2, A9-P6).
- A1 weighting (Runtime Mode): A1's agree/disagree vote is counted **twice** in the weighted fraction. Mechanically, `A1_weight_extra = 1` is added to the numerator iff A1 voted `agree`, and to the denominator iff A1 voted `agree` or `disagree`. When A1 is `own` or `abstain`, no extra is added.
- `agree_frac = agree / (agree + disagree)`; if both are zero, the tally records "all-abstain" and treats `agree_frac = 1.000`.
- Classification:
  - `agreed` iff `weighted_agree_frac ≥ 0.667` AND no blocking objection AND no A1 veto (A1 voted `disagree` with `blocking=true`)
  - `rejected` iff `weighted_agree_frac < 0.334`
  - `disputed` otherwise

Fractions are shown rounded to 3 decimals.

## Per-Proposal Classification

| proposal_id | A1_vote | agree | disagree | abstain | blocking | override_reqs | agree_frac | weighted_agree_frac | classification |
|-------------|---------|-------|----------|---------|----------|---------------|------------|---------------------|----------------|
| A1-P1       | own     | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A1-P2       | own     | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A1-P3       | own     | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A1-P4       | own     | 7     | 1        | 1       | false    | 0             | 0.875      | 0.875               | agreed         |
| A1-P5       | own     | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A1-P6       | own     | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A1-P7       | own     | 8     | 0        | 1       | false    | 0             | 1.000      | 1.000               | agreed         |
| A1-P8       | own     | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A1-P9       | own     | 6     | 2        | 1       | false    | 1             | 0.750      | 0.750               | agreed         |
| A1-P10      | own     | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A1-P11      | own     | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A1-P12      | own     | 7     | 1        | 1       | false    | 0             | 0.875      | 0.875               | agreed         |
| A1-P13      | own     | 8     | 0        | 1       | false    | 0             | 1.000      | 1.000               | agreed         |
| A1-P14      | own     | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A2-P1       | agree   | 8     | 1        | 0       | false    | 0             | 0.889      | 0.900               | agreed         |
| A2-P2       | agree   | 8     | 1        | 0       | false    | 0             | 0.889      | 0.900               | agreed         |
| A2-P3       | agree   | 8     | 0        | 1       | false    | 0             | 1.000      | 1.000               | agreed         |
| A2-P4       | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A2-P5       | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A2-P6       | agree   | 7     | 1        | 1       | false    | 0             | 0.875      | 0.889               | agreed         |
| A2-P7       | agree   | 1     | 2        | 6       | false    | 0             | 0.333      | 0.500               | disputed       |
| A2-P8       | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A2-P9       | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A3-P1       | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A3-P2       | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A3-P3       | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A3-P4       | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A3-P5       | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A3-P6       | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A3-P7       | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A3-P8       | agree   | 8     | 1        | 0       | false    | 0             | 0.889      | 0.900               | agreed         |
| A3-P9       | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A3-P10      | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A3-P11      | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A3-P12      | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A3-P13      | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A3-P14      | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A3-P15      | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A4-P1       | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A4-P2       | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A4-P3       | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A4-P4       | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A4-P5       | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A4-P6       | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A4-P7       | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A4-P8       | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A4-P9       | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A5-P1       | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A5-P2       | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A5-P3       | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A5-P4       | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A5-P5       | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A5-P6       | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A5-P7       | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A5-P8       | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A5-P9       | agree   | 8     | 1        | 0       | false    | 0             | 0.889      | 0.900               | agreed         |
| A5-P10      | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A6-P1       | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A6-P2       | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A6-P3       | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A6-P4       | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A6-P5       | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A6-P6       | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A6-P7       | agree   | 5     | 1        | 3       | false    | 0             | 0.833      | 0.857               | agreed         |
| A6-P8       | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A6-P9       | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A6-P10      | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A6-P11      | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A6-P12      | agree   | 4     | 1        | 4       | false    | 0             | 0.800      | 0.833               | agreed         |
| A6-P13      | agree   | 3     | 1        | 5       | false    | 0             | 0.750      | 0.800               | agreed         |
| A6-P14      | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A7-P1       | agree   | 9     | 0        | 0       | false*   | 0             | 1.000      | 1.000               | agreed         |
| A7-P2       | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A7-P3       | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A7-P4       | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A7-P5       | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A7-P6       | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A7-P7       | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A7-P8       | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A7-P9       | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A8-P1       | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A8-P2       | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A8-P3       | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A8-P4       | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A8-P5       | agree   | 8     | 1        | 0       | false    | 0             | 0.889      | 0.900               | agreed         |
| A8-P6       | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A8-P7       | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A8-P8       | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A8-P9       | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A8-P10      | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A8-P11      | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A8-P12      | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A9-P1       | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A9-P2       | agree   | 2     | 2        | 5       | false    | 1             | 0.500      | 0.600               | disputed       |
| A9-P3       | agree   | 5     | 0        | 4       | false    | 0             | 1.000      | 1.000               | agreed         |
| A9-P4       | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A9-P5       | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A9-P6       | agree   | 4     | 3        | 2       | false    | 1             | 0.571      | 0.625               | disputed       |
| A9-P7       | agree   | 8     | 0        | 1       | false    | 0             | 1.000      | 1.000               | agreed         |
| A9-P8       | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A10-P1      | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A10-P2      | agree   | 7     | 1        | 1       | false    | 0             | 0.875      | 0.889               | agreed         |
| A10-P3      | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A10-P4      | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A10-P5      | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A10-P6      | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A10-P7      | agree   | 8     | 1        | 0       | false    | 0             | 0.889      | 0.900               | agreed         |
| A10-P8      | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A10-P9      | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |
| A10-P10     | agree   | 9     | 0        | 0       | false    | 0             | 1.000      | 1.000               | agreed         |

*A7-P1 has `has_hard_rule_citation = true` (A2 and A6 both cast `agree + blocking` invoking D6). These are reinforcing hard-rule flags on agreement, not blocking objections; classification treats `blocking = false`.

## Classification Summary

| Status   | Count |
|----------|-------|
| agreed   | 107   |
| disputed | 3     |
| rejected | 0     |

Total: 110 proposals.

## Open Disputes (classification = disputed)

### A2-P7 — reserve async-policy extension seam

- Owner: A2. Vote split: 1 agree / 2 disagree / 6 abstain (weighted 0.500).
- **Disagree:**
  - **A7:** "declaring dead interface violates D3 (SRP); defer until Q11 resolves"
  - **A9:** "async-policy seam without named future extender is speculative (G2); convert to Q-record"
- **Abstain:**
  - **A3:** "design-evolution choice; no correctness impact absent concrete consumer"
  - **A4:** "YAGNI debate A2↔A9; no D7/V5 stake if in Q11 companion"
  - **A5:** "no A5-specific interest; A9 YAGNI debate owns this"
  - **A6:** "future interface reservation; doc-level; no security dimension"
  - **A8:** "G2/E1 tension; orthogonal to A8 rubrics"
  - **A10:** "no named future extender tied to Q; speculative vs G2 YAGNI; would flip if A2 names extender"
- **A2's R2 amendment (§(A))** already concedes: *"Reframe as Q-record in `09-open-questions.md` naming the two future policy axes; no interface added in v1."* This amendment converts the proposal into a Q-record, which is what A9, A7, and A10 asked for — i.e., the amendment directly resolves the dispute but votes were cast against the as-written interface proposal.

### A9-P2 — collapse policy-enum/interface pluggability

- Owner: A9. Vote split: 2 agree / 2 disagree / 5 abstain (weighted 0.600). One override request (by A2).
- **Disagree:**
  - **A2 (+ override_request):** "OCP regression at 4 change axes; amend to keep policy enums closed + ADR-listed v2 extensions"
  - **A8:** "X5 §8.2 DfT; deletion acceptable only if single `IEventLoopDriver` test seam survives (synthesis bridge)"
- **Abstain:**
  - **A3:** "A2-vs-A9 question; A3 neutral if no LSP drift"
  - **A4:** "core A2↔A9 extensibility debate"
  - **A5:** "deployment-mode count does not affect R1–R6"
  - **A6:** "no direct security dimension"
  - **A10:** "would agree if A9 commits to keeping one `IEventLoopDriver` test-only seam"
- **A9's R2 amendment (§(A))** already commits to keeping the `IEventLoopDriver` test-only seam with `step()` + `RecordedEventSource`, and to closing policy enums with an appendix listing future extensions. This addresses A8's and A10's conditions directly, and partially addresses A2's OCP concern (closed enums + ADR-listed extensions). A2's override remains on the "interface vs. enum" framing; the amendment would likely flip A8 and A10 to agree.

### A9-P6 — ship v1 with `FUNCTIONAL` sim mode only

- Owner: A9. Vote split: 4 agree / 3 disagree / 2 abstain (weighted 0.625). One override request (by A2).
- **Disagree:**
  - **A2 (+ override_request):** "closes two extension points without migration plan; amend to enum-open + Q-record"
  - **A6:** "REPLAY is *one* forensic-analysis tactic; prefer FUNCTIONAL+REPLAY in v1, defer PERFORMANCE"
  - **A8:** "X5 §8.2 DfT 'Reproducible failures'; prefer keep REPLAY, defer only PERFORMANCE"
- **Abstain:**
  - **A4:** "neutral on scope; if accepted, edits mechanical"
  - **A5:** "R6 reachable with FUNCTIONAL alone if A8-P2 lands"
- **A9's R2 amendment (§(A))** already adopts "ADR-011-R2 with named triggers" for deferral — `SimulationMode` enum stays open, and both PERFORMANCE and REPLAY move to `09-open-questions.md` with named trigger conditions. This resolves A2's migration-plan concern. A6 and A8 remain in disagreement because they specifically want **REPLAY in v1**, not deferred. The split is substantive (retain REPLAY vs. defer REPLAY) and is not fully resolved by the amendment; round 3 will need to pick one of: (i) A9's amendment as written (defer REPLAY with trigger); (ii) FUNCTIONAL+REPLAY in v1 (A6/A8 preference); (iii) ship FUNCTIONAL-only *implementation* but keep REPLAY enum + scaffolding (hybrid).

## Runtime Mode Hot-Path Audit

Proposals with HPI in {allocates, blocks, relayouts, extends-latency} — drawn from round-1 synthesis "Runtime-Mode Hot-Path Audit" table. For each, report A1's round-2 vote and whether the proposer's §(A) amendment supplies the required two-tier bridge.

| proposal_id | R1 HPI           | A1 R2 vote | Two-tier bridge in §(A) amend? | Notes |
|-------------|------------------|------------|--------------------------------|-------|
| A1-P4       | relayouts        | own (agree, self-owned) | **yes** — "Move normative hot/cold split into `modules/core.md §8` only; Logical View stays abstract." | One-time doc relayout; A1 self-amended the location so the relayout is localized to the layout module. A9's R1 dissent (premature layout before benchmark) is explicitly addressed. |
| A1-P9       | relayouts        | own (agree, self-owned) | **yes** — "Spec explicit 2×64-bitmap fallback for G>64 groups per slot-type; above 128 groups: N/64 words." | Fallback ceiling removed; A2's override-request (hard-cap ceiling) satisfied by the fallback in the edit sketch. A9 remains at disagree on YAGNI grounds, but the two-tier bridge is supplied. |
| A1-P12      | extends-latency  | own (agree, self-owned) | **yes** — "Two-tier: fast-path default = `Dedicated` (no knob, 2 µs unchanged); slow-path = `Batched` only when explicitly selected." | Fast-path/slow-path split is explicit; Dedicated remains the no-knob default, so hot path is unaffected. A9's YAGNI dissent (tunable proliferation) stated it would flip to agree under this exact amendment. |
| A1-P13      | extends-latency  | own (agree, self-owned) | **yes** — "HAL contract: `args_size ≤ 1 KiB` fast-path uses pre-registered ring slot from CONTROL pool; 1 KiB < args ≤ 4 KiB slow-path via `BufferRef`; > 4 KiB rejected at admission." | Three-tier bound on `args_blob` copy with admission rejection for the >4 KiB case. HAL contract tightening is explicit. |
| A6-P4       | extends-latency  | agree       | **yes** — "Two-tier: TCP/control-plane `require_encrypted_transport: bool = true` default, fail-closed multi-host plaintext; RDMA data-plane TLS explicitly NOT applied — security = A6-P5 rkey scoping + A6-P2 node auth. Doc in `10-known-deviations.md` Deviation 3." | TLS is confined to TCP control/submission path; RDMA hot path is explicitly excluded. A1's R1 RDMA-latency concern is addressed verbatim. |
| A10-P7      | relayouts        | agree       | **yes** — "Two-tier sharded TaskManager: default `admission_shards=1` (zero-overhead); opt-in `admission_shards>1` at Host Layer only via `LevelParams` (align A1-P2); global outstanding-Submission window single counter; per-shard ordering + global `submission_id` tiebreak." | Default single-shard preserves today's RW-lock fast path; sharded path is opt-in via `LevelParams`. A1's R1 veto concern and A9's "premature sharding for v1" concern are both addressed. A9 remains at disagree; synth amendment would flip A9 to agree. |

**Result:** A1 voted `agree` (or is the self-owning proposer) on every HPI-flagged proposal, and every proposer's §(A) amendment supplies the required two-tier bridge. A1 filed zero blocking objections and zero override requests in round 2 — consistent with the R1 synthesizer's prediction that all hot-path-touching proposals would land with documented two-tier bridges.

---

**Tally complete.** 110 proposals processed; 107 agreed, 3 disputed (A2-P7, A9-P2, A9-P6), 0 rejected. Zero blocking objections (A7-P1's two `blocking=true` rows reclassified as hard-rule citations per the brief). Three override requests (all A2-filed: A1-P9, A9-P2, A9-P6).
