# Aspect A10: Scalability & Data Flow — Round 3 (Stress)

## Metadata

- **Reviewer:** A10
- **Round:** 3 (mandatory stress round)
- **Target:** `/data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/`
- **Runtime Mode:** yes
- **Timestamp (UTC):** 2026-04-19T00:00:00Z
- **Scenarios replayed:** 6.1.2 Distributed Multi-Node Execution at **8× Pods** (`06-scenario-view.md:35-58`); 6.2.2 Remote Node Crash in a **large fleet** (`06-scenario-view.md:100-114`).

## 1. Rubric Checks

Rubric results are unchanged from R2 for the as-written design, but the emerging consensus (after §5 amendments propagate) moves each row as follows.

| # | Check | R2 result | R3 projected result after amended proposals land | Evidence | Rule(s) |
|---|-------|-----------|--------------------------------------------------|----------|---------|
| 1 | Horizontal scaling first (P4) | Weak | Pass at Pod tier; **still Weak at cluster/coordinator tier** (A5-P3 v1 = deterministic fail-fast; A10-P2b decentralize is v2-only). | `05-physical-view.md:187`; `06-scenario-view.md:45` | P4 |
| 2 | Stateless services / externalized state (DS1) | Fail | Pass via A10-P4 DS1 classification table (absorbed into A10-P8 "Data & State Reference"). | A10-P4 edit sketch; `07-cross-cutting-concerns.md` (new §) | DS1 |
| 3 | Async messaging (DS2) | Pass | Pass. | `02-logical-view/08-communication.md:47,51` | DS2 |
| 4 | Minimize data movement (P6) | Pass | Pass; strengthened by A1-P6 (absorbed A10-P5 per-peer `REMOTE_SUBMIT` projection). | `06-scenario-view.md:48`; `modules/distributed.md:55` | P6 |
| 5 | Explicit consistency model per data element (DS6) | Weak | Pass via A10-P3 consistency table (folded into A10-P8). | A10-P3 edit sketch | DS6 |
| 6 | Single authoritative owner (DS7) | Pass | Pass; reinforced by A7-P8 `ScopeHandle` ownership consolidation + A10-P8 diagram. | `02-logical-view/04-memory.md:68`; `modules/distributed.md:139` | DS7 |

**Net:** five of six rubric rows converge after R2 amendments; **one residual Weak** (row 1, P4 at coordinator tier) remains as a deliberately-deferred v2 item, already ADR-roadmap'd per A5-P3 / A10-P2b. No new rubric violations surface in the stress replay.

## 2. Pros

Carry-forward from R1/R2; unchanged after R2 synthesis:

- Distributed-first is structural (`05-physical-view.md:159`) — P4.
- Control/data-plane split (`02-logical-view/08-communication.md:47,64,71`) — P6.
- Async-by-default hot path — DS2.
- Single authoritative owner per data element — DS7.
- Sharding seams pre-written for `producer_index` (A10-P1) and TaskManager admission (A10-P7) — P4 headroom preserved.
- Deterministic vs non-deterministic partitioners are explicitly labeled (`modules/distributed.md:55–56`).
- Bounded back-pressure via `SlotPoolExhausted` (`06-scenario-view.md:122–128`) — P4.
- Per-peer `REMOTE_SUBMIT` projection (A1-P6) linearizes aggregate wire bytes with fan-out.

## 3. Cons

R1/R2 cons remain, plus the following **stress-round-only observations** (detail in §8):

- **C-R3-1:** A5-P3's v1 "deterministic fail-fast" landing, read literally, fails the *whole cluster* when any one Pod-coordinator dies — at 8× Pods this means Nodes in 7 other Pods abort submissions they could have served (R5 + P4). Amendment needed to confine fail-fast scope to one Pod.
- **C-R3-2:** A10-P6's "dedicated heartbeat thread" is implicitly single-threaded and unbounded in peer-fan-in. At fleet scale (≥ 32 peers per Node) the cadence budget blows. Cap needed.
- **C-R3-3:** A10-P7's `admission_shards=1` default is safe but leaves a *silent* P4 ceiling at 8× Pod scale — deployment guidance should say when operators must opt in.
- **C-R3-4:** A10-P3's consistency table must include `cluster_view` and `group_availability` rows explicitly — these are the elements 6.1.2 and 6.2.2 actually touch; without them the DS6 replay misses the scale-sensitive rows.
- **C-R3-5:** A3-P13's cross-node ordering contract (FIFO + idempotent + dup-detect) must be tagged *per channel pair*, not per node — at 8 peers there are 28 unordered-pairs and the duplicate-detect window sizing depends on pair-level inflight counts.

## 4. Proposals

No wholly new proposals. Stress-round deltas are **amendments to existing A10 proposals** (§5) plus one piggy-back clarification offered to peers (§7).

## 5. Revisions of own proposals

All R2 revisions carry forward. The stress replay reveals two additional amendments needed on A10-P6 and A10-P7, and one clarifying amendment on A10-P3. These are minor and consistent with the existing §(A) amendments — none re-open hot-path debate.

| id | action | amended_summary | reason |
|----|--------|-----------------|--------|
| A10-P1 | defend | — | `shards=1` default preserves hot path (A1 concern). `SHOULD partition at Host` still holds at 8× Pods. Replay confirms no new HPI risk. |
| A10-P2a (into A5-P3) | amend | "Fail-fast scope is **the failed Pod only**, not the cluster. Peer Pods with their own coordinators continue admitting Submissions. The Submissions bound to the failed Pod move to `ERROR` with `RemoteNodeFailure{pod_id=<failed>}`; other Pods are unaffected. `cluster_view` generation bumps on the surviving-coordinator list." | Stress replay of 6.2.2 at fleet scale (see §8 row `A5-P3/8× Pods`) shows that a naïve cluster-wide fail-fast mis-interprets v1 as "cluster goes fail-closed on coordinator crash" — which is P4 + R5-regressive. This amendment preserves v1 simplicity and bounds failure blast-radius to one Pod. HPI = none (off hot path). |
| A10-P2b | defend | — | v2 roadmap-only ADR unchanged. |
| A10-P3 | amend | "Consistency table MUST include rows for **`cluster_view` (strong, seqlock + generation)**, **`group_availability` (eventual, heartbeat-driven)**, **`remote_proxy_pool` (strong, owner-thread-exclusive)**, and **`outstanding_submission_set` (strong, atomic counter)** — these are the elements Scenarios 6.1.2 and 6.2.2 actually touch. Intra-Pod shared state (`producer_index`, `completion_counters`, `task_slot_pool.generation`) remains as previously listed." | Stress replay of 6.1.2 at 8× Pods needs DS6 answers for the cross-Pod elements specifically; previous wording permitted a table focused on intra-Pod state only. HPI = none. |
| A10-P4 | defend | — | DS1 classification is doc-only; no stress-round issue. |
| A10-P5 | concede-to-merge | — | Folded into A1-P6. Stress replay of 6.1.2 at 8× Pods shows per-peer projection reduces aggregate REMOTE_SUBMIT bytes near-linearly (see §8). No further amendment needed. |
| A10-P6 | amend | Add the following clause to the R2 amendment: "**Heartbeat thread is sharded by peer-id when `peer_count_per_node > 32`** (default sharding threshold); shards are ring-buffered across a pool of heartbeat workers with cardinality `min(4, ceil(peer_count/32))`. Each shard still uses the dedicated-thread path (no touch to scheduler event loop); the hysteresis window is evaluated per-peer within its owning shard." | Stress replay of 6.2.2 in a large fleet (≥ 32 peers / Node) shows the single-dedicated-thread path saturates at cadence × fan-in; sub-second detection goal breaks. Sharded heartbeat thread-pool keeps HPI = none (still off the scheduler event loop) and restores sub-second detection at fleet scale. Cardinality cap (4) bounds thread cost. |
| A10-P7 | amend | Add the following clause to the R2 amendment: "**Deployment guidance:** `05-physical-view.md` MUST list a rule-of-thumb: when `concurrent_submitters × cluster_nodes ≥ 64`, operators SHOULD set `admission_shards ≥ 4` at the Host Layer. This is documentation, not enforcement; the LevelParam remains opt-in. No change to the default (`shards=1`)." | Stress replay of 6.1.2 at 8× Pods with N concurrent submitters reveals that the silent P4 ceiling is the most likely scale-out surprise. Documentation cue (not a default change) removes the silent failure without disturbing A1's fast path. HPI = none (default unchanged). |
| A10-P8 | defend | — | "Data & State Reference" § absorbs amended P3 rows from this round; no standalone change. |
| A10-P9 | defend | — | WorkStealing × RETRY_ELSEWHERE gate is narrow and replay-robust; `FUNCTIONAL`-mode assignment log is enough under option-(iii) of A9-P6 (see §6.9). |
| A10-P10 | concede-to-merge | — | Folded into A1-P14. Replay confirms cache-line-aligned layout survives at 8× Pods (producer_index is still per-Layer, not cluster-global). |

**Net:** 3 of 10 revised (A10-P2a/A5-P3 blast-radius, A10-P3 table rows, A10-P6 thread sharding, A10-P7 deployment cue). All HPI = none. None opens a new dispute; the owner of A5-P3 (A5) should accept the blast-radius scope amendment as a clarification, not a semantic change.

## 6. Votes on peer proposals

Only **disputed** proposals and any proposal whose amended text is re-opened for new votes are re-voted here. For all other 107 agreed proposals, A10's R2 vote stands (see R2 review §6.1–6.9).

### 6.1 Disputed — A2-P7 (reserve async-policy extension seam)

| proposal_id | vote | rationale (cite rule id) | blocking | override_request |
|-------------|------|--------------------------|----------|-------------------|
| A2-P7 (amended: Q-record in `09-open-questions.md`; no interface in v1) | **agree** | The R2 amendment converts the proposal into a Q-record — exactly the landing zone A10 (and A7, A9) called for in R2. With no interface added in v1, G2 YAGNI is satisfied; with the two named future policy axes written into `09-open-questions.md`, E5 migration hygiene is preserved. HPI = none. | false | false |

### 6.2 Disputed — A9-P2 (collapse policy-enum/interface pluggability)

| proposal_id | vote | rationale (cite rule id) | blocking | override_request |
|-------------|------|--------------------------|----------|-------------------|
| A9-P2 (amended: single `IEventLoopDriver` test-only seam with `step()` + `RecordedEventSource`; closed policy enums; appendix listing future extensions) | **agree** | The amendment satisfies A10's R2 conditional agreement verbatim ("would agree if A9 commits to keeping one `IEventLoopDriver` test-only seam"). Closed deployment-mode enums with an appendix of v2 extensions meets E4 at document level without paying G2 cost in v1. A8's X5 DfT concern is met by the single test seam; A2's OCP concern is met by the appendix. HPI = none. | false | false |

### 6.3 Disputed — A9-P6 (defer PERFORMANCE/REPLAY simulation)

| proposal_id | vote | rationale (cite rule id) | blocking | override_request |
|-------------|------|--------------------------|----------|-------------------|
| A9-P6 (synthesis option **(iii)**: ship FUNCTIONAL-only implementation; keep `REPLAY` enum open + scaffolding so A6-P12 and A8 harness can land day 2) | **agree** | Option (iii) is the smallest move that closes the dispute without losing any aspect's concern: A9 gets v1 simplicity (implementation deferred — G2), A2 gets enum-open + ADR-011-R2 migration plan (E4 + E5), A6 gets the forensic pathway viable on day 2 (S6 via A6-P12 format frozen), A8 gets the DfT reproducibility seam via A8-P2 + scaffolding (X5). A10's own A10-P9 "WorkStealing × RETRY_ELSEWHERE" gate requires only the FUNCTIONAL-mode assignment log, which is retained. HPI = none. | false | false |

### 6.4 Re-vote snapshot

| — | agree | disagree | abstain | total |
|---|-------|----------|---------|-------|
| A10 R3 (only the three disputed rows re-cast) | 3 | 0 | 0 | 3 |

The 95 / 0 / 5 vote profile from R2 (§6.10) stands on all other proposals; the three R3 flips (A2-P7 abstain→agree, A9-P2 abstain→agree, A9-P6 agree→agree under option iii) move A10's total to 98 / 0 / 2.

## 7. Cross-aspect tensions

(R2 tensions carry forward; new R3 observations added.)

| pair | with_proposal | preferred_resolution |
|------|---------------|----------------------|
| A1 vs A10 (new, R3 stress) | A10-P6 heartbeat thread-pool shard (see §5) | Shard-per-32-peers, cap=4, stays on dedicated thread-pool — zero touch to scheduler event loop. A1 HPI = none. |
| A5 vs A10 (new, R3 stress) | A5-P3 blast-radius clarification (see §5 A10-P2a amendment) | Fail-fast scope = failed Pod only, not cluster. A5 accepts as clarification; does not change the v1 architectural choice. |
| A2 vs A10 (R3, clarifying) | A10-P7 deployment guidance cue | `05-physical-view.md` deployment rule-of-thumb is documentation only; LevelParam surface (A2's E3) is untouched. |
| A8 vs A10 (R3, replay-reinforced) | A8-P6 distributed trace time-alignment | At 8× Pods the time-alignment contract is the *precondition* for auditing A10-P3's consistency table rows for `cluster_view`. Keep the soft dependency tag from R2; no new amendment needed. |
| A7 vs A10 (R3, holds) | A7-P5 `distributed_scheduler` depends only on `ISchedulerLayer` | At 8× Pod scale this cycle-break keeps the distributed layer independently scalable. No tension. |
| A9 vs A10 (R3, resolved) | A9-P6 option (iii) + A10-P9 FUNCTIONAL-mode assignment log | Compatible: A10-P9's assignment log is FUNCTIONAL-only and needed for live RETRY_ELSEWHERE correctness. REPLAY scaffolding deferred. |

## 8. Stress-attack on emerging consensus

**Ground rules for this table (per round-3 prompt §4):**

- Each row targets a tentatively-agreed proposal where A10 has standing (P4, P6, DS1, DS2, DS6, DS7, R5).
- The stress attempt replays **at least one of 6.1.2 (8× Pods) or 6.2.2 (Remote Node Crash)** step-by-step against the *amended* text from R2 §5.
- Verdict: `holds` | `breaks` | `uncertain`. `breaks` / `uncertain` rows carry a concrete amendment in §5 above.

| proposal_id | amended-text summary (post-R2 §5) | stress_attempt | scenario replay cite | verdict |
|-------------|-----------------------------------|----------------|----------------------|---------|
| A1-P1 | Pre-size `producer_index`; forbid rehash on hot path | At 8× Pods, admission of 10 K-task Submissions from many submitters hammers `producer_index` on each Node's Host Layer. Attack: "the capacity bound `max_outstanding × expected_out` is a static guess and the 8× Pod workload blows it." **Replay:** 6.1.2 step 5 (`06-scenario-view.md:49`) — Node₁ receives remote tasks, local Host Layer admission runs; producer_index lookups occur on insertions of OUT/INOUT edges. With A1-P14's capacity formula applied per-Layer (not per-Pod), the 8× Pod workload just scales the number of *Layers*, not the per-Layer bound — formula holds. | `06-scenario-view.md:44-51` | **holds** |
| A1-P2 | Cap DATA-mode lock hold; `shard_count` as `LevelParam` | Attack: "shard_count must default to >1 at Pod scale or A10-P1 / A10-P7 become silent bottlenecks." **Replay:** 6.1.2 step 5 admission onto Node₁'s Host Layer at 8× Pods. A10-P7 amendment in §5 adds the deployment cue — operator is informed to bump `admission_shards ≥ 4`. A1-P2's `shard_count` LevelParam is the exact surface used. | `06-scenario-view.md:49-50` | **holds** (with A10-P7 doc cue; see §5) |
| A1-P3 | Function Cache LRU + capacity | Attack: "at 8× Pods with many distinct functions per node the LRU can evict hot functions and starve admission." **Replay:** 6.1.2 step 5 — Node₁ looks up `dist_orch_func`'s image. If LRU is sized to default and the workload has wider function set per Pod, cold-miss path touches CONTROL / HEARTBEAT presence check (A1-P3). Per-Layer, not per-Pod sizing, is fine at 8× Pods because each Node carries the same function set. | `06-scenario-view.md:49` | **holds** |
| A1-P4 | Hot/cold `Task` split, normative in `modules/core.md §8` | Attack: none with scale standing; this is a one-time relayout. **Replay:** 6.1.2 step 3 Task creation — layout is set once per Pod at init. At 8× Pods each node pays the one-time cost independently. | `06-scenario-view.md:45` | **holds** |
| A1-P5 | Latency budgets for SPMD + event-loop | Attack: "SPMD budget is specified for single-node (6.1.3); at 8× Pods there is no cross-Pod SPMD budget." **Replay:** 6.1.2 step 6–7 (`06-scenario-view.md:50-51`) is not SPMD; A1-P5 scope is event-loop stage budgets inside each Node. Budgets are per-Node, stack independently across 8 Pods — no cross-Pod constraint needed. | `06-scenario-view.md:50-51` | **holds** |
| A1-P6 (absorbed A10-P5) | Per-peer projection + payload hygiene | Attack: "projection adds a partition pass that is O(tasks + edges) — at 10 K tasks × 8 peers this may regress partition latency." **Replay:** 6.1.2 step 2 (`06-scenario-view.md:46`) runs `IPartitioner` on the coordinator. Amended A1-P6 runs projection *inside the same partition pass* — one pass emits 8 per-peer sub-submissions, not 8 serial passes. Net wire-bytes ≤ 1.5× one-payload vs 8× without projection (A10-P5 sanity test). Partition CPU cost scales O(tasks+edges), not O(peers × tasks+edges). | `06-scenario-view.md:46-48` | **holds** |
| A1-P14 (absorbed A10-P10) | `producer_index` CONTROL placement + shard default | Attack: "at 8× Pods the per-Layer cap × 8 Pods requires a Pod-level cluster-wide hashmap." **Replay:** 6.1.2 step 5–7: producer_index is **per-Layer** (Host layer has its own; Node₁'s Host layer owns its own entries). At 8× Pods each Node holds only its own Host-Layer entries; no cluster-wide map is implied. Amended A1-P14 formula (capacity ≥ `max_outstanding × expected_out × 2`) is per-Layer and survives 8× fan-out. | `06-scenario-view.md:49-51` | **holds** |
| A2-P6 | `IDistributedProtocolHandler` abstract; one concrete v1 backend; CRTP/`final` devirt | Attack: "at 8× Pods the indirect call frequency × 8 peers × REMOTE_* message types risks vtable stalls." **Replay:** 6.1.2 steps 4, 7, 8 (`REMOTE_SUBMIT`, RDMA `copy_to_peer`, `REMOTE_COMPLETE`). Synthesis requires single-impl devirt via `final` — these calls inline. A2-P6's HPI=none tag holds. | `06-scenario-view.md:48,51-52` | **holds** |
| A3-P12 | `drain()` / `submit()` concurrency + distributed semantics | Attack: "at 8× Pods `drain()` must specify cross-Pod semantics — does drain block until all 8 Pods drain?" **Replay:** 6.1.2 step 11 (`06-scenario-view.md:55`) — Python receives aggregated result; this is effectively drain at the Pod level. A3-P12's amended contract covers distributed semantics per R2 §(A). If rows in A10-P3 consistency table include `outstanding_submission_set` as strong/atomic, drain is consistent. | `06-scenario-view.md:55` | **holds** (depends on amended A10-P3 table rows; see §5) |
| A3-P13 | Cross-node ordering (FIFO + idempotent + dup-detect) | Attack: "dup-detect window sizing is node-global; at 8× Pods the pairwise fan-out (28 pairs at 8 Pods) pushes aggregate inflight past the window." **Replay:** 6.1.2 step 8 + 6.2.2 step 4 — `REMOTE_COMPLETE` and `REMOTE_ERROR` messages must be de-duplicated. If the dup-detect window is per-peer-pair (not global), sizing is local — 8× 8 pairs × `max_inflight_per_pair` is still bounded. Amendment offered to A3 (see §7): tag dup-detect window as per-pair. | `06-scenario-view.md:52`; `06-scenario-view.md:107-109` | **uncertain** → amendment offered to A3 (per-pair window scope) |
| A5-P1 | Exponential backoff + jitter on remote retries | Attack: "at fleet scale, many Nodes retrying simultaneously after a single peer crash causes retry synchronization bursts." **Replay:** 6.2.2 step 3 — `DistributedScheduler` marks Node₁ FAILED; if retry policy fires from many peers simultaneously, synchronized retries hit alternates together. A5-P1's jitter clause breaks the synchronization; amended text keeps jitter at cluster scale. | `06-scenario-view.md:108-113` | **holds** |
| A5-P2 | Per-peer circuit breaker (thread-local TSC) | Attack: "at 8× Pods the per-peer state is 8 × 8 × |Pods|; breaker state is node-local but cross-Pod visibility is missing." **Replay:** 6.2.2 step 1–4 — circuit breaker on Node₀ trips for Node₁. Each Node's breaker is independent; no cross-Pod propagation needed because each Node only calls its own peers. State cost O(peers-per-Node) is bounded. | `06-scenario-view.md:106-109` | **holds** |
| A5-P3 (absorbed A10-P2a) | v1 deterministic coordinator fail-fast; v2 decentralize roadmap ADR | **Strongest A10 stress attack.** Read literally, "deterministic fail-fast" at 8× Pods could be misread as *cluster-wide* fail-closed — when one Pod's coordinator crashes, the other 7 Pods should continue accepting submissions. **Replay:** 6.2.2 at 8× Pods — Coordinator of Pod₀ crashes. If scope is cluster-wide, Pods 1–7 go fail-closed (unnecessary). If scope is Pod-only, Pods 1–7 continue. The R2 §(A) amendment does not explicitly state scope. A10-P2a (see §5) pins scope to "the failed Pod only." Without this clarification, v1 could regress P4 + R5 at cluster tier. | `06-scenario-view.md:100-114` | **breaks** → amendment in §5 (A10-P2a blast-radius scope) |
| A5-P7 | Timeout on `IMemoryOps` async | Attack: "at 8× Pods the RDMA fabric bandwidth saturates and timeouts fire en-masse — without cluster-aware degradation, many Submissions fail together." **Replay:** 6.1.2 step 7 RDMA data exchange at 8× Pods. A5-P7's timeout is per-op, not per-fabric. A5-P8 degradation specs should include fabric-saturation graceful-degrade; no new ask. | `06-scenario-view.md:51` | **holds** |
| A5-P8 | Degradation specs + alert rules | Attack: "8× Pod scale's admission-saturation + WG-loss profile is not in the degradation matrix." **Replay:** 6.2.2 step 6 policy evaluation branches (ABORT_ALL / CONTINUE_REDUCED / RETRY_ALTERNATE) must all have deg-alert rules. A5-P8 externalizes alert rules; at fleet scale just need to parameterize thresholds, which the externalized rules-file already enables. | `06-scenario-view.md:111-113` | **holds** |
| A5-P10 | DS4 per-REMOTE_* idempotency | Attack: "per-peer-pair dup-detect window interacts with idempotency at 8× fan-out — if window evicts a seqno still retried, DS4 weakens." **Replay:** 6.2.2 step 6c (RETRY_ALTERNATE) combined with A10-P9 gate. Per-REMOTE_* idempotency coupled with per-pair FIFO dup-detect (see A3-P13 row above) closes the hole. | `06-scenario-view.md:113` | **holds** (co-depends on A3-P13 per-pair-window amendment in §7) |
| A6-P2 | Concrete node authentication in HANDSHAKE (mTLS cert-pinned v1; SPIFFE v2) | Attack: "at 8× Pods the HANDSHAKE TLS cost per new peer connection is non-trivial." **Replay:** 6.1.2 step 4 (`06-scenario-view.md:48`) — Node₀ sends `REMOTE_SUBMIT` to Node₁ via RDMA. A6-P2's mTLS is on the control channel (TCP, per A6-P4), not RDMA. At 8× Pods, each new Pod does a one-time handshake; steady-state traffic is not TLS-hot-path. HPI = none at scale. | `06-scenario-view.md:48` | **holds** |
| A6-P3 | Bounded payload parsing (entry-gate guard; ≤1 compare at entry) | Attack: "at 8× Pod fan-in of `REMOTE_SUBMIT` with projected per-peer payloads, entry-gate aggregate rate is 8× but per-message cost unchanged." **Replay:** 6.1.2 step 4–5 with per-peer projection (A1-P6) — each peer receives its own projected sub-submission; entry-gate cost per message is 1 compare, aggregate over 8 Pods is still bounded (one compare per inbound REMOTE_SUBMIT). | `06-scenario-view.md:48-49` | **holds** |
| A8-P6 | Distributed trace time-alignment contract | Attack: "at 8× Pods the clock-sync accuracy determines whether A10-P3's `cluster_view` row can be audited." **Replay:** 6.2.2 step 1–3 — heartbeat-loss detection timestamps must align across Pods to attribute the NodeLost edge. A8-P6 is a soft dependency for A10-P3's cross-Pod rows; no amendment to A8-P6 needed — tag the soft dependency in ADR. | `06-scenario-view.md:106-109` | **holds** |
| A10-P1 (self) | Default `producer_index` sharding (`shards=1` default; `SHOULD partition at Host`) | Self-attack: "at 8× Pods with many submitters, `shards=1` default at Host is a silent P4 ceiling." **Replay:** 6.1.2 step 5 admission. A10-P7's new deployment-cue amendment (see §5) covers this case by documenting the opt-in threshold in `05-physical-view.md`. A10-P1 default itself is preserved because A1's fast path depends on it. | `06-scenario-view.md:49-50` | **holds** (once A10-P7 deployment cue lands) |
| A10-P3 (self) | Per-data-element consistency model | Self-attack: "R2 amendment lists only intra-Pod state; 6.1.2 and 6.2.2 need `cluster_view` and `group_availability` rows." **Replay:** 6.1.2 step 2–4 references `cluster_view` implicitly; 6.2.2 step 1–3 directly references heartbeat-driven `group_availability`. Amendment in §5 adds those rows explicitly. | `06-scenario-view.md:46-48`; `06-scenario-view.md:106-109` | **uncertain → amended** (see §5 A10-P3) |
| A10-P4 (self) | Stateful/stateless classification | Self-attack: "at 8× Pods, `coordinator` role in Pod-level scheduler is a stateful role not explicitly classified." **Replay:** 6.1.2 step 1 "Pod Scheduler creates Task; state → SUBMITTED" — the Pod scheduler is stateful; A10-P4 table must list it with "Deviation 2" justification (persistence would exceed budget). Already covered by the R2 amendment text; no new change. | `06-scenario-view.md:45` | **holds** |
| A10-P6 (self) | Faster peer-failure detection (dedicated heartbeat thread + hysteresis) | Self-attack: "dedicated thread" is singular — at fleet scale (≥ 32 peers/Node) cadence × fan-in saturates it, breaking sub-second detection. **Replay:** 6.2.2 step 1–3 at fleet scale — kill one peer; detection time must still be sub-second. Amended A10-P6 (see §5) adds shard-per-32-peers thread-pool, cap=4. HPI still = none (off scheduler event loop). | `06-scenario-view.md:106-109` | **uncertain → amended** (see §5 A10-P6 shard clause) |
| A10-P7 (self) | Two-tier sharded TaskManager | Self-attack: "default `admission_shards=1` at 8× Pods with many submitters is a silent P4 ceiling." Amended (see §5) with explicit deployment cue. | `06-scenario-view.md:49-50` | **uncertain → amended** (see §5 A10-P7 deployment cue) |
| A10-P8 (self) | Single "Data & State Reference" § in `07-cross-cutting-concerns.md` | Self-attack: "one page absorbs P3+P4+P8 — at 8× Pods the page must be readable and not a dump." Replay-friendly: the reader can trace any `BufferRef` through its producer → consumer → retirement on one page even at Pod scale (ownership is still per-Layer, not cluster-global). Holds. | `06-scenario-view.md:49-55` | **holds** |
| A10-P9 (self) | Gate `WorkStealing` × `RETRY_ELSEWHERE` | Self-attack: "under A9-P6 option (iii) REPLAY enum+scaffolding only, does the FUNCTIONAL-mode assignment log still work?" **Replay:** 6.2.2 step 6c — RETRY_ALTERNATE with WorkStealing. Assignment log is FUNCTIONAL-mode and live-only; REPLAY is not required for correctness of this gate. | `06-scenario-view.md:113` | **holds** |

**Summary of stress-attack results:**

- 19 rows **`holds`** — the amended R2 text survives scenario replay at 8× Pod scale and during fleet-scale Remote-Node-Crash.
- 4 rows **`breaks` or `uncertain`** — all A10-owned or A10-co-owned, all **amended in §5** with concrete deltas that preserve HPI=none and do not re-open dispute:
  1. **A5-P3 / A10-P2a** — blast-radius clarification: fail-fast scope is the failed Pod, not the cluster. (`breaks` → amended.)
  2. **A10-P3** — add `cluster_view`, `group_availability`, `remote_proxy_pool`, `outstanding_submission_set` rows to the consistency table. (`uncertain` → amended.)
  3. **A10-P6** — shard the dedicated heartbeat thread per 32 peers (cap=4). (`uncertain` → amended.)
  4. **A10-P7** — add `05-physical-view.md` deployment cue for `admission_shards ≥ 4` at large-cluster thresholds. (`uncertain` → amended.)
- 1 **soft handoff** to A3: A3-P13 per-pair dup-detect window scope (noted in §7; A3 to confirm in their R3 review).

**No new disputes introduced.** All four amended rows are clarifications, not semantic shifts; HPI = none on all. No blocking objections. No A1 hot-path re-veto needed (A10 does not hold veto; no hot-path surface touched).

## 9. Status

- **Satisfied with current design?** **yes, conditional on §5 amendments landing** — specifically: (a) A5-P3 Pod-scope blast-radius language, (b) A10-P3 cross-Pod consistency rows, (c) A10-P6 heartbeat thread-pool shard clause, (d) A10-P7 deployment cue. All four are non-controversial clarifications consistent with each proposal's R2 §(A) amendment and carry HPI = none.
- **Open items expected in next round (if any):** none from A10. A10 votes **agree** on all three previously-disputed proposals (A2-P7, A9-P2, A9-P6 option (iii)) and amends four of its own without re-opening debate. Remaining work is application of amended text in Phase-5 edits.
- **Convergence readiness:** A10 sees the review at `converged` for its aspect after the four minor amendments are accepted. No A10 hot-path concerns; no A10 blocking objections; no A10 override requests.
