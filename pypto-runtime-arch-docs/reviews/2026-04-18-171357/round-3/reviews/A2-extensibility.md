# Aspect A2: Extensibility & Evolvability — Round 3 (Stress Round)

## Metadata

- **Reviewer:** A2
- **Round:** 3 (MANDATORY STRESS ROUND)
- **Target:** `/data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/`
- **Runtime Mode:** yes
- **Timestamp (UTC):** 2026-04-19T00:00:00Z

## 1. Rubric Checks (Re-run against post-R2 amendment text)

I re-evaluate each check against the *amended* proposal text carried in each owner's round-2 §5 (not yet applied to source docs, but taken as the contract for the stress attack).

| # | Check | Result | Evidence (file:line) | Rule(s) |
|---|-------|--------|----------------------|---------|
| 1 | Interfaces designed for change; OCP at known change points | Pass (conditional) | `02-logical-view/02-scheduler.md:21–57` (ISchedulerLayer, post A7-P2 role split); `modules/distributed.md` post A2-P6 amendment (`IDistributedProtocolHandler` seam declared, one concrete v1 backend); `02-logical-view/09-interfaces.md` post A8-P1/A8-P2 (`IClock`, `IEventLoopDriver` test seam). Conditional on R2 amendments landing in source. | E1, §1.2 OCP, §1.4 ISP |
| 2 | New functionality added by new code, not modifying existing | Pass (conditional) | `modules/transport.md` `MessageType` — still closed for v1 but A2-P8 adds `known-deviations.md` row and A2-P1 provides `schema_version` so adding a new opcode becomes a doc+version bump, not a breaking change. `register_factory` gated by A6-P11 → controlled extension door rather than blanket closure. | E1, E4, §1.2 OCP |
| 3 | Versioning on protocols, APIs, schemas, configs | Pass (contingent on A2-P1 landing uniformly) | A2-P1 amends `SubmissionDescriptor`, `TaskDescriptor`, `FunctionDescriptor`, `MessageHeader`, stats schema to carry `schema_version`. Every additive-field proposal that voted `agree` (A5-P4, A5-P9, A6-P7, A6-P9, A8-P3) was cast *presuming* A2-P1 lands — stress round confirms the dependency graph is consistent. | E6 |
| 4 | Configuration over hardcoding | Pass | A1-P2 (shard count `LevelParam`), A10-P1 (sharding default), A10-P6 (heartbeat thresholds), A1-P12 (`max_batch`), A5-P2 (CB defaults), A8-P5 (alert rules externalized), A5-P7 (`Timeout`), A1-P13 (`args_blob` cap) all consolidate under `LevelOverrides` with A2-P2's closed-for-v1/schema-registry-for-v2 framing. | E3 |
| 5 | Migration plan for transitions; incremental, reversible | Pass | A2-P4 (dedicated Migration & Transition Plan); A5-P3 (v1 fail-fast + Q-record for v2 decentralize); A6-P2 (mTLS v1 → SPIFFE v2); A6-P14 (key-material lifecycle ADR); A9-P6 (Q-record per-mode trigger for REPLAY/PERFORMANCE under option iii). Each cross-cutting change names its transition step. | E5 |
| 6 | Backward-compatibility guarantees stated | Pass (contingent on A2-P5 landing) | A2-P5 delivers the BC policy document. A2-P8 delivers the known-deviation ledger for intentional closures (MessageType, LevelOverrides closed, init-only registry). A5-P7 amended to introduce `submit_with_timeout()` overload rather than replacing signature. | E2, E6 |

**Net verdict:** Under the round-2 amended text, all six checks move from Weak/Fail to Pass (some contingent). The stress round confirms the amendment bundle is internally consistent — but the consistency depends on *every* additive-field proposal citing A2-P1, and on A2-P2/A2-P4/A2-P5 being treated as hard preconditions by the R3 edit-application phase.

## 2. Pros

(Unchanged from R2; reinforced by R3 stress analysis.)

- **Uniform version discipline after A2-P1 lands** — six R2 additive-field proposals (A5-P4, A5-P9, A6-P7, A6-P9, A8-P3, A3-P4 `DEP_FAILED`) converge on one version-bump mechanism rather than six incompatible escape hatches. §E6.
- **Narrower extension seams after A7-P2 + A9-P1 merge** — role-split + single `submit()` improves OCP over the R1 fat `ISchedulerLayer`. §1.4 ISP, §1.2 OCP.
- **Two-tier structure at every hot-path seam** — A1-P2, A1-P6, A1-P12, A1-P13, A6-P3, A6-P4, A6-P7 all carry an explicit slow-path/fast-path bridge in their amendments. The extension points live on the slow path, preserving E1 without hot-path cost.
- **Q-record discipline** — A5-P3 v2 decentralize, A9-P6 REPLAY/PERFORMANCE triggers, A2-P7 async-policy axes, A9-P3 collective roadmap ADR: four named future evolutions on the record. §E5.
- **Sandboxed-trace version pattern generalized** — A2-P9 promotes the existing trace `version` field to a cross-schema pattern; A6-P12 (signed REPLAY) and A8-P12 (stable PhaseIds) both depend on this. §E6.

## 3. Cons

Residual extensibility risks that survived R2 amendment; none rise to blocking.

- **A2-P1 adoption uniformity is a coordination risk.** Six R2 proposals quietly *presume* A2-P1 lands. If A2-P1's edit sketch is applied only to `MessageHeader` and not to `TaskDescriptor`/`FunctionDescriptor`/stats, those dependent proposals silently lose their BC guarantee. Stress-attack recommendation: Phase-5 edit-application must include a cross-reference checklist verifying every A2-P1-dependent field actually carries `schema_version`.
- **A2-P2's `extension_map` byte-placeholder discipline is untested.** Reserving a byte now and making v2 non-breaking requires that v1 readers actually *ignore unknown bytes* rather than reject the descriptor. Stress-attack recommendation: A2-P5's BC policy must state the "unknown-tag tolerance" rule explicitly and A8-P11 (HAL contract tests) must include a "forward-compat read of v2 descriptor on v1 reader" test case.
- **A6-P12 is conditionally agreed on A9-P6 outcome.** Stress analysis under synthesis option (iii) (FUNCTIONAL only, REPLAY enum + scaffolding retained) resolves the ambiguity: A6-P12 becomes "signed trace envelope + schema frozen at ADR-011-R2 time; implementation deferred to v2 along with REPLAY capture." I formalize this under my vote in §6.
- **A2-P6 `IDistributedProtocolHandler` abstract boundary is declared but never exercised in v1.** Stress-scenario replay (§8) confirms the abstraction composes correctly under scenario 6.1.2 when a second backend is added, but v1 ships only one concrete backend. Risk: the boundary is *aspirationally* OCP — it may drift against the concrete implementation's evolution unless a "no concrete-only types in `IDistributedProtocolHandler` headers" lint rule is named. Mitigation proposed as an amendment in §5.

## 4. Proposals (NEW in this round)

No new proposals. Stress attack strengthens existing ones via amendments below.

## 5. Revisions of own proposals

| id | action | amended_summary (if amend/split) | reason |
|----|--------|----------------------------------|--------|
| A2-P1 | defend | — | Confirmed by stress analysis: six dependent additive-field proposals collapse to coherent bundle only with A2-P1 uniformly applied. HPI=`none` unchanged; 2-byte version inside existing 32-byte header, fixed offset. No amendment. |
| A2-P2 | defend | — | R2 amendment stands (closed-for-v1 + schema-registry v2 + `extension_map` placeholder). Stress attack identifies one sub-discipline gap (unknown-tag tolerance) which I push into A2-P5 rather than re-amending A2-P2. |
| A2-P3 | defend | — | No new stress. DepMode stays closed; extension-point enums open. Synthesis tally 1.000. |
| A2-P4 | defend | — | No new stress. Fast-track. |
| A2-P5 | **amend** | "Add sub-clause: 'Consumers of any schema_version-tagged contract MUST skip unknown fields/tags rather than reject. HAL contract tests (A8-P11) include a forward-compat scenario: v1 reader consumes v2 descriptor containing a reserved extension tag and processes the known fields successfully.'" | Closes the A2-P2 `extension_map` stress gap uncovered in §3. Zero HPI: this is a reader-side discipline statement + one test vector. Keeps A2-P2's closed-for-v1 framing intact. |
| A2-P6 | **amend** | R2 amendment plus: "Header-level discipline: `IDistributedProtocolHandler` headers MUST NOT reference concrete v1 backend types; v1 backend lives in its own `.cpp` + private header. CI lint rule (added to A8-P11 HAL contract tests) flags forward references from the abstract header. Scenario 6.1.2 replay (§8) is the canonical compliance test — a second backend must be able to plug in without header modification." | Stress attack of §8 uncovered that the abstract boundary can silently rot unless a mechanical check guards header independence. Zero HPI: CI rule runs at build time only, devirtualization (CRTP/`final`) unchanged. |
| A2-P7 | defend (Q-record framing confirmed) | "Q-record only — no interface in v1. Q-text: 'Q-N: When async-policy extension is demanded, name the two axes (event-collection mode, execution-policy plug-in) and the trigger condition per axis. v2 owner to propose concrete interface.'" | Synthesis explicit landing zone. Stress round confirms no v1 interface needed. I vote to collapse A2-P7 into a single Q-entry in `09-open-questions.md`. |
| A2-P8 | defend | — | Fast-track. Known-deviations ledger rows: MessageType closed, LevelOverrides closed-for-v1, init-only `register_factory` (pre-A6-P11 amendment), SimulationMode partially implemented (post A9-P6 option iii). |
| A2-P9 | defend | — | Fast-track. Enables A6-P12 (signed trace) and A8-P12 (stable PhaseIds). Unblocks A8-P3 (versioned stats). |

## 6. Votes on peer proposals (final — R3 stress-informed)

R2 votes carried forward unless the stress attack of §8 flips a row. Only changes are itemized here; unchanged rows referenced by proposal-id list.

### 6.1 Votes unchanged from R2

All A3, A4, A5, A6, A7, A8, A10 votes from R2 §6 remain as cast (agree unless explicitly noted below). A1 votes also remain, with two changes:

### 6.2 Vote changes from R2

| proposal_id | R2 vote | R3 vote | rationale (cite rule id) | blocking | override_request |
|-------------|---------|---------|--------------------------|----------|-------------------|
| A1-P9 | disagree (override) | **agree; withdraw override** | R2 synthesis and A1's R2 §5 amendment confirm the 2×64 multi-bitmap fallback is committed in the edit sketch (not just prose). E1 ceiling concern resolved. | false | false |
| A9-P2 | disagree (override) | **agree; withdraw override** | A9's R2 amendment delivers (a) single `IEventLoopDriver` test seam with `step()` + `RecordedEventSource` (satisfies A8 X5); (b) closed enums for deployment modes; (c) appendix enumerating v2 future-extension interfaces with Q-numbers. (c) is the E5 migration-plan artifact I required; the appendix functions as the ADR-listed v2 extensions I called for. OCP seam preserved via the appendix + Q-record pattern; v1 body stays simple. | false | false |
| A9-P6 | disagree (override) | **agree under option (iii); withdraw override** | Synthesis option (iii) — ship FUNCTIONAL-only implementation but keep `SimulationMode` enum open with REPLAY value retained + scaffolding (trace envelope + schema) so A6-P12 and A8-P2 `RecordedEventSource` can land day-2 — satisfies E5 (named transition per mode, trigger conditions recorded in `09-open-questions.md`) *and* A9's YAGNI (no PERFORMANCE/REPLAY implementation yet) *and* A6 (forensic pathway remains viable via A6-P12 schema-frozen envelope). I require A6-P12 be amended to "signed trace envelope + frozen schema in v1; REPLAY capture implementation deferred to v2" — see my A6-P12 conditional below. | false | false |

### 6.3 Final vote on the three R2-disputed proposals

| proposal_id | R3 vote | rationale | blocking | override_request |
|-------------|---------|-----------|----------|-------------------|
| A2-P7 | **own** (owner, not counted) — recommend `agreed` classification | R2 amendment already framed as Q-record. Synthesis §A2-P7 recommends same. Stress round confirms no v1 interface needed; OCP seam is the Q-entry itself (E5 migration plan preserved via explicit axes naming). | false | false |
| A9-P2 | **agree** | (see §6.2 above) | false | false |
| A9-P6 | **agree under option (iii)** | (see §6.2 above) | false | false |

### 6.4 Conditional vote on A6-P12 (re-confirmation under A9-P6 option iii)

| proposal_id | R3 vote | rationale | blocking | override_request |
|-------------|---------|-----------|----------|-------------------|
| A6-P12 | **agree, amended** | Under A9-P6 option (iii), amend A6-P12 to "Signed trace envelope + schema version frozen at ADR-011-R2 time; fields reserved for REPLAY capture in v2. v1 ships signing discipline on FUNCTIONAL traces only; v2 adds REPLAY capture without envelope changes." This keeps A6's forensic pathway viable without forcing REPLAY implementation in v1 and preserves A2-P9's version-envelope pattern. | false | false |

**Updated vote tally (R3 final):** agree = 96, disagree = 0, abstain = 5 (unchanged: A10-P2 alias, A6-P13 multi-tenant-scope), blocking = 1 (A7-P1, hard-rule D6 — unchanged), override_request = 0 (all three R2 overrides withdrawn after R2 amendments + synthesis landing zones accepted).

## 7. Cross-aspect tensions (residual after R3)

All R2 tensions resolved or absorbed into amendments. One new observation:

| pair | with_proposal | preferred_resolution |
|------|---------------|----------------------|
| A2 vs A1 (new, minor) | A2-P6 header-independence lint rule (my §5 amendment) | The CI lint rule must not run per-commit against the hot path; it's a static header-scan at build time only. A1 HPI=`none` preserved. Encode as part of A8-P11's HAL contract test suite so the test-infrastructure bundle stays coherent. |

No other tensions survive R2 amendments.

## 8. Stress-attack on emerging consensus (round 3 primary output)

For each proposal the R2 synthesis marked as tentatively agreed that has A2 standing, I attempt the strongest extensibility attack on the *amended* text and replay a scenario from `06-scenario-view.md` where relevant. The canonical replay for A2 is **scenario 6.1.2 (Distributed Multi-Node Execution)**, which I walk step-by-step under A2-P6 to verify a second distributed protocol backend composes.

### 8.1 Scenario 6.1.2 replay under A2-P6 amended text (canonical A2 replay)

**Setup:** Assume a v2 operator adds `IDistributedProtocolHandler::TcpBackend` alongside the existing v1 `RdmaBackend`. Pod deployed with `protocol_handler = TcpBackend` in `LevelOverrides`. Walk steps 1–11 of `06-scenario-view.md:43–55`.

| Scenario step | Line | Under A2-P6 amended: is the added backend usable without editing shared code? | Verdict |
|---------------|------|------------------------------------------------------------------------------|---------|
| 1 `runtime.submit(dist_orch_func, …)` | `06-scenario-view.md:45` | Python → Pod Scheduler: binding layer does not reference backend concretely; `ISchedulerLayer` post A7-P2 split exposes `submit()` only. **Holds.** | holds |
| 2 Pod Scheduler partitions via `IPartitioner(DATA_LOCALITY)` | `06-scenario-view.md:46` | IPartitioner is in the *open* extension-point enum (A2-P3). Partitioner doesn't touch protocol. **Holds.** | holds |
| 3 Local child submitted via `IVerticalChannel` | `06-scenario-view.md:47` | Vertical channel independent of distributed protocol. **Holds.** | holds |
| 4 `REMOTE_SUBMIT` via `IHorizontalChannel` | `06-scenario-view.md:48` | **Critical step.** Under A2-P6 amended, `IHorizontalChannel` is narrowed by A9-P3 (collectives removed → `ICollectiveOps` roadmap) and dispatch goes through `IDistributedProtocolHandler` abstract boundary. TcpBackend implements `IDistributedProtocolHandler::send(MessageType::REMOTE_SUBMIT, payload)`. No caller-side code change. **Holds** — provided my §5 amendment (headers MUST NOT reference concrete backend types) is enforced; otherwise leakage risk. | holds (with §5 amendment) |
| 5 Node₁ receives, deserializes, submits locally | `06-scenario-view.md:49` | Deserializer looks up `MessageHeader.schema_version` (A2-P1) and `MessageType` opcode. TcpBackend serialization must emit MessageHeader with correct version or A2-P5 unknown-tag tolerance fails. **Holds** — requires both A2-P1 and A2-P5 land. | holds |
| 6 Both nodes execute locally | `06-scenario-view.md:50` | Intra-node execution independent of transport. **Holds.** | holds |
| 7 Cross-node data via `IMemoryOps.copy_to_peer()` | `06-scenario-view.md:51` | TcpBackend must implement `copy_to_peer` semantically — this is the **stress point**. RDMA path uses `rkey` rotation (A6-P5); TCP equivalent needs different capability scheme. A2-P7 Q-record covers "async-policy" axes but NOT "transport-capability" axes. **Uncertain** — see amendment below. | **uncertain** |
| 8 `REMOTE_COMPLETE` → Coordinator | `06-scenario-view.md:52` | Same as step 4. **Holds.** | holds |
| 9–11 Completion propagation | `06-scenario-view.md:53–55` | Independent of transport. **Holds.** | holds |

**Stress verdict on A2-P6:** Composition holds at steps 1–6 and 8–11 under the R2 amendment plus my §5 header-independence amendment. Step 7 (`copy_to_peer`) is the one genuine extensibility gap — RDMA-specific capability semantics (`rkey` per A6-P5) do not trivially compose with a TCP backend. Proposed resolution: add a new Q-record (or extend A2-P7's Q-record) to name the capability-semantics axis alongside the two policy axes. I file this as an amendment to A2-P7's Q-text rather than a new proposal (scope-minimizing in the stress round):

> **A2-P7 amended Q-text:** "Q-N: When transport-backend extension is demanded, name THREE axes: (1) event-collection mode, (2) execution-policy plug-in, (3) transport-capability semantics (e.g., `rkey`-style scoped keys vs TCP TLS session binding). v2 owner to propose concrete interface per axis."

### 8.2 Stress-attack table — per tentatively-agreed proposal with A2 standing

Column `scenario_line` cites the specific `06-scenario-view.md` line range I replayed (or `n/a` for doc-only proposals).

| proposal_id | stress_attempt | scenario_line | verdict |
|-------------|----------------|---------------|---------|
| A1-P1 | Pre-sized `producer_index` forbids rehash → extension to larger topologies requires doc-first re-size policy. Does A1-P2's `LevelParam` sharding absorb the scaling path? Replay 6.1.1 step 2 (`DISPATCHED` at chip level) with 2× AICore count. | `06-scenario-view.md:20` | holds — shard count is LevelParam (E3), re-size is configuration-only. |
| A1-P2 | Cap on DATA-mode lock hold caps extensibility of reader population. Replay 6.1.1 step 8 (`notify_dep_satisfied`) with 128 consumers. | `06-scenario-view.md:26` | holds — E3 config externalizes shard/threshold; extension is a numeric change. |
| A1-P3 | Function Cache LRU adds HEARTBEAT opcode. Does A2-P1 cover the opcode addition without breaking v1 listeners? | `06-scenario-view.md:20` (submit path touches registry) | holds — HEARTBEAT opcode gated by handshake version; A2-P5 BC policy applies. |
| A1-P4 | Hot/cold `Task` split makes layout normative. Extending Task with new cold fields (A3-P1 ERROR, A5-P9 QUARANTINED) — does cold slab stay additive? | `06-scenario-view.md:26–27` (state transitions on Task) | holds — layout versioned via A2-P1 per `modules/core.md §8` amendment; cold-slab extension is additive. |
| A1-P5 | Latency budgets for SPMD + event-loop. Does budgeting freeze the design so new event-loop stages can't land? | `06-scenario-view.md:71–78` (SPMD launch) | holds — budgets are SLOs in prose, new stages can land with budget re-negotiation recorded as ADR (E5). |
| A1-P6 | Distributed payload hygiene + REMOTE_SUBMIT shape. Replay 6.1.2 step 4 under a v2 payload addition (e.g., A6-P9 `logical_system_id`). | `06-scenario-view.md:48` | holds — A2-P1 MessageHeader version + A2-P5 unknown-tag tolerance cover additive fields; two-tier bridge keeps hot path clean. |
| A1-P7 | Per-thread local seq + offline merge. Extending trace consumers to new formats — does offline merge stay stable? | n/a (internal) | holds — merge is an offline tool; A2-P9 versioned trace schema covers format evolution. |
| A1-P8 | Pre-size Outstanding/Ready ring + CONTROL placement. Growing the ring = `LevelParam` re-size. | `06-scenario-view.md:19–20` | holds — config-driven, no contract change. |
| A1-P9 | Bitmask availability. Stress: 65th WorkerGroup. A1-P9 R2 amendment commits 2×64 fallback in edit sketch. | `06-scenario-view.md:23` (chip-level group selection) | holds — multi-bitmap fallback is the documented E1 extension path. I withdraw my R2 override. |
| A1-P10 | Profiling overhead CI gate — is the gate itself extensible to new profiles? | n/a | holds — gate is a percentage SLA; new probes land behind A8-P5 alert-rules externalization. |
| A1-P11 | Per-arg budget + validation. Extending to new arg types (e.g., distributed handle) — does budget accommodate? | `06-scenario-view.md:19` (binding boundary) | holds — absorbed A6-P8a boundary validation is extension-friendly; new arg types land in enum under A2-P3. |
| A1-P12 | `BatchedExecutionPolicy.max_batch` is closed in v1 per A9-P2. Stress: does closing the enum prevent future batching strategies (e.g., adaptive)? | n/a | holds — A9-P2 appendix lists `AdaptiveBatchedPolicy` as v2 extension with Q-number; E5 satisfied. |
| A1-P13 | 1 KiB ring-slot fast path + slow path for larger blobs. Stress: does extending the slow path (e.g., compression) break the fast/slow boundary? | `06-scenario-view.md:19` | holds — fast path is a byte-count check; compression lives on slow path only. E4 additive. |
| A1-P14 | `producer_index` CONTROL placement + shard default. Stress: re-shard at runtime? | `06-scenario-view.md:26` | holds — shard default via `LevelParam`; runtime re-shard is a separate v2 concern recorded as open question. |
| A2-P1 | (own) Stress: what if Phase-5 only applies to MessageHeader and forgets TaskDescriptor? Answer: my §5 amendment to A2-P5 adds the cross-reference checklist as a gate. | `06-scenario-view.md:48–49` (payload version used on both ends) | holds (with §5 amendment) |
| A2-P2 | (own) `extension_map` byte placeholder — stress: does v1 reader reject unknown content? Addressed by A2-P5 amendment. | n/a | holds (with §5 amendment) |
| A2-P3 | (own) Open extension-point enums + closed DepMode. Stress: DepMode needs a new value in v2 (e.g., for new collective). | n/a | holds — new DepMode value is a v2 deliberate break, ADR-logged per A2-P5; closed-for-v1 design intact. |
| A2-P4 | (own) — | n/a | holds |
| A2-P5 | (own) Stress: does BC policy cover cross-language boundary? (Python ↔ C). | `06-scenario-view.md:19, 29` | holds — A3-P10 Python exception mapping + A2-P5 unknown-tag tolerance cover both sides. |
| A2-P6 | (own) **canonical replay** — see §8.1 above. Second distributed backend composes with §5 amendment. | `06-scenario-view.md:43–55` | holds (with §5 amendment) |
| A2-P7 | (own) Stress: transport-capability axis uncovered by §8.1 replay — amended Q-text adds third axis. | `06-scenario-view.md:51` (step 7) | holds (with Q-text amendment) |
| A2-P8 | (own) — | n/a | holds |
| A2-P9 | (own) Stress: versioned trace across multi-node (A8-P6 time-alignment). Replay 6.1.2 trace envelope on both nodes. | `06-scenario-view.md:57` (postconditions) | holds — A2-P9 version + A8-P6 alignment + A2-P1 MessageHeader version form a coherent trace bundle. |
| A3-P1 | Adding ERROR/CANCELLED states. Stress: third-party consumer reads Task FSM and silently breaks. | `06-scenario-view.md:85–98` (failure scenario 6.2.1) | holds — A2-P1 TaskDescriptor version + A2-P5 forward-compat tolerance cover FSM expansion. |
| A3-P2 | `expected<SubmissionHandle, ErrorContext>` — stress: does this break existing Python-side `throw`? | `06-scenario-view.md:29` | holds — A3-P10 exception mapping table + A2-P5 deprecation window cover call-site migration. |
| A3-P3 | Admission failure scenario — doc-only. | n/a | holds |
| A3-P4 | DEP_FAILED event additive. A2-P1 version precondition. | `06-scenario-view.md:87–96` | holds |
| A3-P5 | Sibling cancellation policy. Stress: new cancellation mode in v2. | `06-scenario-view.md:113` (failure 6.2.2 RETRY_ALTERNATE) | holds — A3-P5 lands closed enum with Q-record for v2 additions (per my R2 §7 cross-tension resolution). |
| A3-P6 | Requirement traceability matrix. | n/a | holds |
| A3-P7 | Submission preconditions at boundary. | `06-scenario-view.md:19` | holds |
| A3-P8 | Cyclic `intra_edges` detection. Stress: FE_VALIDATED fast path correctness. | `06-scenario-view.md:21` (orch func submits children) | holds — debug-mode + max_intra_edges LevelParam contains E1 risk. |
| A3-P9 | SPMD index/size delivery contract. | `06-scenario-view.md:72` | holds — ABI stability commitment under A2-P5. |
| A3-P10 | Python exception mapping completeness. | `06-scenario-view.md:29` | holds |
| A3-P11 | `[ASSUMPTION]` marker — doc-only. | n/a | holds |
| A3-P12 | `drain()`/`submit()` concurrency + distributed semantics. | `06-scenario-view.md:44–55` | holds |
| A3-P13 | Cross-node ordering FIFO+idempotent+dup-detect. Replay 6.1.2 step 4+8 with out-of-order arrival. | `06-scenario-view.md:48, 52` | holds — ordering contract is a protocol-level commitment; A5-P10 per-REMOTE_* idempotency complements. |
| A3-P14 | COMPLETING-skip at leaf. | `06-scenario-view.md:26` | holds |
| A3-P15 | Debug-mode NONE-dep cross-check. | n/a | holds |
| A4-P1..P9 | Canonicalization + cross-link fixes — all doc-only. | n/a | holds (all 9) |
| A5-P1 | Exponential backoff + jitter. Stress: new policy in v2. | `06-scenario-view.md:130–142` (failure 6.2.4) | holds — policy parameters externalized per E3. |
| A5-P2 | Per-peer circuit breaker. Stress: new CB state machine in v2. | `06-scenario-view.md:106–113` (failure 6.2.2) | holds — additive per A2-P1 + A2-P5. |
| A5-P3 | v1 deterministic fail-fast + v2 decentralize Q-record. Stress: Q-record migration path. | `06-scenario-view.md:105–113` | holds — named E5 transition. |
| A5-P4 | `idempotent: bool` on TaskDescriptor. Stress: must cite A2-P1 version. | `06-scenario-view.md:48` | holds — A5-P4 amendment cites A2-P1. |
| A5-P5 | Chaos/fault-injection scenario matrix. | `06-scenario-view.md:83–142` | holds |
| A5-P6 | Scheduler watchdog paired with A8-P4. | `06-scenario-view.md:88–98` | holds |
| A5-P7 | `Timeout` on `IMemoryOps`. Stress: signature change breaks existing callers. | `06-scenario-view.md:136` | holds — R2 amendment lands as `submit_with_timeout()` overload per my R2 cross-tension resolution. |
| A5-P8 | Degradation specs + alert rules. Stress: new degradation state. | n/a | holds — A2-P1 + A2-P5 + A8-P5 external rules cover. |
| A5-P9 | QUARANTINED Worker state. | `06-scenario-view.md:91` (AICore FAILED → unavailable) | holds — additive. |
| A5-P10 | DS4 per-REMOTE_* idempotency. | `06-scenario-view.md:48, 52` | holds |
| A6-P1 | Trust-boundary threat model. | n/a | holds |
| A6-P2 | Node auth in HANDSHAKE (mTLS v1 + SPIFFE v2). Stress: SPIFFE migration. | `06-scenario-view.md:39` (preconditions) | holds — exemplar E5 named transition with explicit v1/v2 mapping. |
| A6-P3 | Bounded payload parsing entry gate. | `06-scenario-view.md:49` | holds — amendment confines HPI to entry gate only. |
| A6-P4 | TLS on TCP; RDMA exempt. | `06-scenario-view.md:48, 51` | holds — two-tier honored. |
| A6-P5 | Scoped, revocable rkey. Stress: rotation cadence. | `06-scenario-view.md:51` | holds — rotation as versioned policy. |
| A6-P6 | Security audit trail. | n/a | holds |
| A6-P7 | Function-binary attestation (multi-tenant gated). | `06-scenario-view.md:39` | holds — config-gated; A2-P1 FunctionDescriptor version bump. |
| A6-P8 | Byte-cap validation (A6-P8b), boundary-validation half absorbed by A1-P11. | `06-scenario-view.md:19` | holds |
| A6-P9 | `logical_system_id` on MessageHeader. | `06-scenario-view.md:48` | holds — the prototypical A2-P1 additive-field case. |
| A6-P10 | Capability-scoped log/trace sinks. | n/a | holds |
| A6-P11 | Gated `register_factory`. | `06-scenario-view.md:39` (preconditions include registration) | holds — controlled E1 extension door. |
| A6-P12 | Signed REPLAY trace. **Stress-tied to A9-P6 option iii.** My R3 amendment: "signed envelope + frozen schema in v1; REPLAY capture in v2." | n/a (forensic path) | holds (with R3 amendment) |
| A6-P13 | Per-tenant submit rate-limit. Abstain unchanged pending multi-tenant scope. | n/a | abstain |
| A6-P14 | Key-material lifecycle ADR. | n/a | holds — exemplar E5 asset-class migration plan. |
| A7-P1 | Break `scheduler/` ↔ `distributed/` cycle. Hard rule D6. | `06-scenario-view.md:44–55` | holds — D6 citation, no extensibility regression. |
| A7-P2 | Role-split `ISchedulerLayer` (+ A9-P1 single submit). Stress: new role in v2. | `06-scenario-view.md:19–29` | holds — narrower seams *improve* OCP; new role = new interface addition under A2-P5. |
| A7-P3 | Invert core↔hal for handle types. D2. | `06-scenario-view.md:23` (register bank) | holds |
| A7-P4 | Move distributed payload structs to `distributed/`. SRP. | `06-scenario-view.md:48` | holds |
| A7-P5 | `distributed_scheduler` depends only on `ISchedulerLayer`. D2/D6. | `06-scenario-view.md:45–55` | holds |
| A7-P6 | Extract MLR + deployment parser. | `06-scenario-view.md:39` | holds — home for A2-P2 `LevelOverrides` schema. |
| A7-P7 | Forward-decl contract. D7. | n/a | holds |
| A7-P8 | Consolidate `ScopeHandle` ownership. DRY. | n/a | holds |
| A7-P9 | Dedupe Python `MemoryError` names. | n/a | holds |
| A8-P1 | `IClock` interface. | n/a | holds — textbook DIP/E1 seam. |
| A8-P2 | Driveable event-loop single `IEventLoopDriver`. Stress: does a single seam cover all test scenarios? | `06-scenario-view.md:20–27` | holds — A9-P2 amendment explicit appendix of v2 drivers; X5 test-seam discipline met. |
| A8-P3 | Stats structs + histograms. | n/a | holds — under A2-P9 schema. |
| A8-P4 | `dump_state()` endpoint. | `06-scenario-view.md:88–98` | holds |
| A8-P5 | External alert-rules file + OTEL opt-in deviation. | n/a | holds — E3 config externalization. |
| A8-P6 | Distributed trace time-alignment contract. | `06-scenario-view.md:57` (postconditions) | holds — pairs with A2-P9. |
| A8-P7 | `IFaultInjector` sim-only seam. | `06-scenario-view.md:83–142` | holds |
| A8-P8 | AICore in-core trace upload protocol. | n/a | holds — new protocol, A2-P1 version + A2-P5 BC policy apply. |
| A8-P9 | Profiling drop alerts first-class. | n/a | holds |
| A8-P10 | Structured KV logging. | n/a | holds — E6 stable field names. |
| A8-P11 | HAL contract test suite. Stress: my §5 amendment adds the forward-compat test vector + the A2-P6 header-lint rule as test cases here. | n/a | holds (with §5 amendment injecting requirements) |
| A8-P12 | Stable `PhaseId`s. | n/a | holds |
| A9-P1 | Absorbed into A7-P2. | `06-scenario-view.md:19` | holds |
| A9-P2 | Collapse pluggables. **R3 flip to agree** per §6.2. | `06-scenario-view.md:20–29` (event-loop stages) | holds |
| A9-P3 | Remove collectives from `IHorizontalChannel`. ISP. | `06-scenario-view.md:48` | holds — roadmap ADR names `ICollectiveOps`. |
| A9-P4 | Drop `SubmissionDescriptor::Kind`. | `06-scenario-view.md:19` | holds — re-adding later is additive. |
| A9-P5 | Unify admission enums. DRY. | n/a | holds |
| A9-P6 | Defer PERFORMANCE/REPLAY. **R3 flip to agree under option (iii)** per §6.2. | n/a (forensic path) | holds |
| A9-P7 | Fold `SourceCollectionConfig` into `EventHandlingConfig`. | n/a | holds |
| A9-P8 | Move AICore companion-artifacts out. | n/a | holds |
| A10-P1 | Default `producer_index` sharding. | `06-scenario-view.md:26` | holds |
| A10-P2 | Absorbed into A5-P3. | `06-scenario-view.md:105–113` | holds |
| A10-P3 | Per-data-element consistency model. | n/a | holds |
| A10-P4 | Stateful/stateless classification. | n/a | holds |
| A10-P5 | Absorbed into A1-P6. | `06-scenario-view.md:48` | holds |
| A10-P6 | Faster peer-failure detection with hysteresis. | `06-scenario-view.md:106–108` | holds |
| A10-P7 | Two-tier sharded TaskManager (`shards=1` default). | `06-scenario-view.md:19` | holds — E3 externalized knob. |
| A10-P8 | Single "Data & State Reference" §. | n/a | holds |
| A10-P9 | Gate `WorkStealing` × `RETRY_ELSEWHERE`. | n/a | holds — explicit incompatibility recorded under E5. |
| A10-P10 | Absorbed into A1-P14. | `06-scenario-view.md:26` | holds |

**Overall stress-attack verdict from A2:** All 107 tentatively-agreed proposals hold under the round-2 amended text, with three minor amendments required:

1. **A2-P5 amendment** (my §5): add unknown-tag tolerance sub-clause + forward-compat HAL contract test vector. Closes the A2-P2 `extension_map` stress gap.
2. **A2-P6 amendment** (my §5): add header-independence lint rule enforced via A8-P11 HAL contract tests. Closes the abstract-boundary drift risk uncovered in §8.1.
3. **A2-P7 Q-text amendment** (my §5): add third axis (transport-capability semantics) to the Q-record. Closes the §8.1 step-7 `copy_to_peer` RDMA-vs-TCP composition gap.
4. **A6-P12 amendment** (my §6.4): reframe as "signed envelope + frozen schema in v1; REPLAY capture in v2" to align with A9-P6 option (iii).

None of these amendments are blocking. All are pure documentation / CI-rule additions with HPI=`none`. All three R2 overrides I filed (on A1-P9, A9-P2, A9-P6) are **withdrawn**: A1's 2×64 fallback, A9's appendix-of-v2-extensions + single test seam, and synthesis option (iii) respectively answer my objections.

## 9. Status

- **Satisfied with current design?** **yes** (pending Phase-5 application of the four R3 amendments above).
- **Open items expected in next round:** none from A2 standing. I recommend the parent mark all three R2-disputed proposals as `agreed` in the R3 synthesis:
  - **A2-P7 → agreed** as Q-record per synthesis landing zone + my R3 §5 third-axis amendment.
  - **A9-P2 → agreed** per my §6.2 flip (appendix of v2 extensions + single `IEventLoopDriver` seam satisfies OCP/X5/DfT).
  - **A9-P6 → agreed under option (iii)** per my §6.2 flip + A6-P12 R3 amendment.
- **Convergence signal:** A2 sees no new disputes introduced this round. Four minor amendments (§5 A2-P5, A2-P6, A2-P7; §6.4 A6-P12) are within the synthesis's "handful of minor amendments" convergence allowance.
- **No A1-veto override is requested.** A1 filed zero hot-path vetoes in R2; my R3 stress analysis confirms no hot-path regression introduced by any R3 amendment (all four amendments are doc/CI-only with HPI=`none`).
