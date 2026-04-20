# Round 3 (Stress Round) — Consolidated Extract

- **Round:** 3 (MANDATORY STRESS ROUND)
- **Target:** `/data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/`
- **Reviewers extracted:** A1 performance, A2 extensibility, A3 functional-sanity, A4 doc-consistency, A5 reliability, A6 security, A7 modularity, A8 testability, A9 simplicity, A10 scalability.
- **Scope:** extraction only; final decisions are not synthesized here.

---

## 1. Stress attacks per reviewer

Each row records a stress attack/objection raised in R3 §8 (or §3 cons feeding §8) against one or more of the 107 R2-agreed proposals. Columns:
- **target** — affected proposal id(s)
- **attack summary** — 1–2 lines
- **resolved?** — "resolved" if reviewer's own stress replay shows amendment closes it (no withdrawal of the underlying proposal) / "successful (amend)" if reviewer wants a minor amendment / "successful (withdraw)" if reviewer now wants to withdraw/reverse / "holds" if attack failed (proposal survives untouched)
- **A1 hot-path veto?** — Y/N (only A1 can issue this; others cite HPI observations)

### 1.1 A1 (performance) stress attacks

| target | attack summary | resolved? | A1 hot-path veto |
|---|---|---|---|
| A1-P1 | Admission storm at 10× `max_outstanding` may push open-addressed probe length beyond cache-line-resident fanout. | holds (R2 capacity formula keeps L<0.5, probe ≤ 2). | N |
| A1-P2 | `scheduler_thread_count=8` + concurrent 1000-output retirement — writer-mode hold bound. | holds (p99 ≤ 10 μs per shard, 8 shards). | N |
| A1-P3 | Function Cache LRU eviction at capacity may drop hot binary. | holds (cold-miss already budgeted as REMOTE_BINARY_PUSH slow path). | N |
| A1-P4 | Does S2 step 3 (`ErrorContext`) write on success-path critical section? | holds (ErrorContext is in cold tail; written only on FAILED). | N |
| A1-P5 | SPMD `block_dim=108` (a5 max) — batch prep + register-bank + 108-core ACK < 5 μs? | holds (ACK as 108-bit popcount ≈10 ns). | N |
| A1-P6 (self) | R2 amendment covers steady-state amortized but **cold-start first-use template-miss** is unbudgeted in §4.8.2. | **breaks (minor) — successful (amend)** — owner adds cold-path budget row (≤ 15 μs first-use). | N |
| A1-P7 | Three threads emit Phase at identical `timestamp_ns` — is total order defined? | holds (`thread_id` tiebreak deterministic). | N |
| A1-P8 | `WouldBlock` back-pressure only exercised under storm (not in scenario). | holds (self-validating via admission harness). | N |
| A1-P9 | a5 108 AICores needs 2×64 fallback (AND + ctzll). | holds (< 10 ns both paths). | N |
| A1-P10 | CI gate off-scenario. | holds (vacuous). | N |
| A1-P11 | 16-tensor submit: 16×200 ns = 3.2 μs vs §4.8.1 Row 1 "<0.5 μs"? | holds — structural separation is correct (boundary call vs per-arg term); canonical 4-arg fits. | N |
| A1-P12 | Does any default silently enable Batched? | holds (R2 amendment pins Dedicated default in §4.1.4). | N |
| A1-P13 | 1 KiB fast-path copy ≤ 2 μs? | holds (Host→Chip DMA ≥ 4 GB/s → ≤ 250 ns). | N |
| A1-P14 | Admission requires CONTROL region. | holds (R2 amendment places index in CONTROL_AREA/CONTROL_HEAP). | N |
| A5-P6 (peer) | Deadman write elision into per-cycle TSC is identified in R2 §7 but **not yet normative** in `modules/scheduler.md`; implementer could add redundant atomic store. | **uncertain → successful (amend)** — A1 asks A5 to add normative elision clause (minor). | N |
| A8-P3 (peer) | Histogram insert on completion path; R2 didn't pin bucket array to CONTROL region → first cache-line pull competes with admission-queue drain. | **uncertain → successful (amend)** — A1 asks A8 to add region-placement line in `modules/profiling.md` (minor). | N |
| A3-P4 (peer) | Successor walk on failure path; inline small-vector in A1-P4 cold tail? | holds (natural home; no amendment). | N |
| A2-P6 (peer) | Virtual-call cost for `IDistributedProtocolHandler`. | holds (R2 top-N builtin short-circuit + indexed load + indirect branch). | N |
| A6-P9 (peer) | `logical_system_id` placement on MessageHeader. | holds (outside 64 B hot line). | N |
| A5-P2 (peer) | `thread_local` TSC check placement; single-node scenario doesn't exercise. | holds (projected on 6.1.2). | N |
| A6-P3/P4/P5/P8/P13 (peers) | Various boundary/TLS/rkey/byte-cap/per-tenant stresses. | holds (all projected or single-tenant default). | N |
| A8-P8/P12 (peers) | L2-opt-in emits under SPMD. | holds (L1 strips; L2 contingent on A1-P7). | N |
| A10-P1/P7 (peers) | Default `shards=1` relayout. | holds (fast path preserved). | N |

**A1 R3 veto count: 0.** A1 declared **"0 hot-path vetoes applied"** and confirmed every HPI-flagged proposal holds under scenario replay.

### 1.2 A2 (extensibility) stress attacks

| target | attack summary | resolved? | A1 hot-path veto |
|---|---|---|---|
| A2-P1 (self) | Phase-5 edit might apply version byte only to `MessageHeader`, miss `TaskDescriptor`/`FunctionDescriptor`/stats → dependent additive-field proposals silently lose BC guarantee. | holds with A2-P5 amendment (cross-reference checklist added). | N |
| A2-P2 (self) | `extension_map` byte placeholder untested — v1 reader might reject unknown bytes instead of ignoring. | holds with A2-P5 amendment (unknown-tag tolerance clause). | N |
| A2-P6 (self) | `IDistributedProtocolHandler` abstract boundary declared but aspirational — concrete-only types might leak into abstract header. | **uncertain → successful (amend)** — add header-independence lint rule in A8-P11. | N |
| A2-P7 (self) | Canonical 6.1.2 replay step 7 (`copy_to_peer`): RDMA rkey vs TCP TLS binding don't compose; Q-record axes only cover async-policy, not transport-capability. | **uncertain → successful (amend)** — add THIRD axis "transport-capability semantics" to Q-record text. | N |
| A6-P12 (peer) | Contingent on A9-P6 outcome; if full REPLAY in v1, over-builds. | **successful (amend)** — reframe as "signed envelope + frozen schema in v1; REPLAY capture in v2". | N |
| A1-P1..P14, A3-P1..P15, A5-P1..P10, A6-P1..P14, A7-P1..P9, A8-P1..P12, A9-*, A10-P1..P10 (peers) | For each, tested whether extension seam holds under 6.1.2 replay with v2 additions. | holds (all). | N |

### 1.3 A3 (functional-sanity) stress attacks

| target | attack summary | resolved? | A1 hot-path veto |
|---|---|---|---|
| A3-P1 | 6.2.3 step 5 parent→ERROR; without amendment two impls could diverge (COMPLETED+flag vs ERROR vs RETIRED). | holds — R2 `ERROR` state present. | N |
| A3-P2 | If binding choice is `Status+out-param` instead of `expected<>`, caller can silently skip null-check. | holds — §5 amendment pins `std::expected<>` as normative. | N |
| A3-P3 | New scenarios §6.2.5–§6.2.7 don't cross-reference §6.2.3. | holds. | N |
| A3-P4 (self) | SPMD sub-task failure: "precomputed successor list" silent on SPMD aggregation edge → parent not notified. | **breaks (minor) → successful (amend)** — extend edit sketch to include SPMD aggregation edge. | N |
| A3-P5 (self) | 24-way SPMD, one sibling fails; `ErrorContext.remote_chain` loses per-sibling `spmd_index`. | **breaks (minor) → successful (amend)** — retain `spmd_index` on cancelled siblings. | N |
| A3-P6 | Under A9-P6 option (iii) FR-10 must still map. | holds with row "FR-10 → §6.1.1–6.1.3 (FUNCTIONAL) + ADR-011-R2". | N |
| A3-P7 (self) | SPMD submission `block_dim=0` / `per_tile_args.size() != block_dim` not in precondition catalog. | **breaks (minor) → successful (amend)** — add `SpmdIndexOOB`, `SpmdSizeMismatch`, `SpmdBlockDimZero`. | N |
| A3-P8 | `FE_VALIDATED` fast path skips even 24-element DFS — correctness? | holds. | N |
| A3-P9 | `spmd_index/size` delivery vs A9-P4 discriminator. | holds (ABI preserved). | N |
| A3-P10 (self) | 6.2.3 step 5 raises `simpler.RuntimeError` but under full domain mapping Scheduler→`simpler.SchedulerError`. | **breaks (minor) → successful (amend)** — update scenario text to match. | N |
| A3-P11/P13/P15 | Not exercised by 6.1.3/6.2.3. | holds (vacuous). | N |
| A3-P12 | `drain()` + retry-on-exhaustion oscillation risk. | holds (drain flag sticky). | N |
| A3-P14 | Leaf FIN maps to `EXECUTING→COMPLETED`. | holds. | N |
| A1-P8 (peer) | 6.2.3 dual exhaustion paths — diagnostic sub-codes missing. | **uncertain (minor) → route to A1/A3-P7 merge** — add `TaskSlotExhausted`/`ReadyQueueExhausted` sub-codes. | N |
| A1-P11 (peer) | SPMD shape sub-codes must live in single validation pass. | **uncertain (minor)** — route to A1 (already folded). | N |
| A5-P4 (peer) | Scenario 6.2.3:125 allows retry without consulting `idempotent` bit. | **breaks (minor) → route to A5** — update scenario to clarify caller retry independent of flag. | N |
| A7-P2 (peer) | If A7-P2 lands before A3-P2, three role interfaces inherit ambiguous submit() contract. | **uncertain (ordering) → route to A7** — enforce A3-P2-before-A7-P2 or co-commit. | N |
| A9-P4 (peer) | Edge: `tasks.size()==1 && spmd.has_value() && spmd->size==1` — one-way SPMD. | **uncertain (edge) → route to A9 + A3** — explicitly accept/reject in catalog. | N |
| A9-P5 (peer) | 6.2.3:123 still uses `ResourceExhausted`; should be `AdmissionStatus::REJECT(Exhaustion)` per A9-P5. | **breaks (minor) → route to A9** — update scenario text. | N |
| A9-P6 option (iii) peer | Developer sets `REPLAY` in v1 — undefined behavior if silent fallback to FUNCTIONAL? | holds (stipulate scaffolding rejects explicitly). | N |
| Remaining 20+ peer rows (A1/A2/A3/A4/A5/A6/A7/A8/A9/A10) | various composition checks. | holds (all). | N |

### 1.4 A4 (doc-consistency) stress attacks

| target | attack summary | resolved? | A1 hot-path veto |
|---|---|---|---|
| A4-P1 | HAL enum casing mixes third axis (`onboard`/`sim` lowercase). | holds (rename contained in `modules/hal.md` + ADR-011). | N |
| A4-P2 | Fixed anchor might be renumbered by R2 amendment. | holds (no R2 touches §2.1.3.1.A). | N |
| A4-P5 (self) | Conditional glossary branch pending A9-P2 outcome. | resolved — A9-P2 lands → `IEventCollectionPolicy` becomes one-line pointer. | N |
| A4-P6 | ADR inline cites clutter views. | holds (single-citation only where local). | N |
| A4-P7 | Diagram box length under A7-P2 role-interface list. | holds (role list goes in glossary, not diagram). | N |
| A4-P8/P9 | `CANCELLED` conditional. | holds with "if adopted" footnote. | N |
| A3-P1 (peer) | Three spellings for terminal state in scenario text (`COMPLETED(error)`, `ERROR`, `FAILED`). | **uncertain (minor) → A4 raises new R3-P2**: unify spelling across `06-scenario-view.md:91/93/95/108/111`. | N |
| A5-P9 (peer) | `QUARANTINED` added to WorkerState but **no Appendix-B or glossary entry** for WorkerState at all. | **uncertain (minor) → A4 raises new R3-P1**: add WorkerState row to Appendix-B + glossary. | N |
| A7-P2 (peer) | Atomic PR requirement for `ISchedulerLayer` rename. | holds (edit sketch enumerates role interfaces). | N |
| A4-P6 + A2-P5 (Appendix-C) | New appendix could drift from existing appendix voice. | holds (A4 authors skeleton, two sections). | N |
| A9-P5 peer | Rename from `ResourceExhausted` to `AdmissionStatus`. | holds (scenario prose doesn't mention enum). | N |
| A9-P7 peer | Drop `SourceCollectionConfig` glossary ghosts. | holds (no existing entry; clean). | N |
| A7-P8/P9 | Consolidate ScopeHandle / dedupe MemoryError. | holds (textbook D7). | N |
| A6-P9/P2/P4/A8-P1/P2 | New types → glossary rows. | holds (mechanical). | N |
| A10-P3/P4/P8 | "Data & State Reference" composition. | holds (A4 authors single section). | N |
| A5-P1/P2/P8 (retry/CB/deg types) | New formal types glossary growth. | holds. | N |
| A6-P12 contingent on A9-P6 option (iii) | Glossary for REPLAY + `replay_trace_signature`. | holds. | N |
| A7-P4 | Ownership column in Appendix-B. | holds (no scenario edit needed). | N |
| A1-P6 | `REMOTE_BINARY_PUSH` new MessageType. | holds (glossary row + one narrative sentence). | N |

**A4 summary:** 22 `holds`, 0 `breaks`, 2 `uncertain (minor)` — covered by new proposals R3-P1 and R3-P2 (see §4 below).

### 1.5 A5 (reliability) stress attacks

| target | attack summary | resolved? | A1 hot-path veto |
|---|---|---|---|
| A5-P1 | Under 6.2.2 RETRY_ELSEWHERE, `base_ms · 2^n` at n=4 silently breaches SLO. | holds (amendment caps `max_ms=2000`; A8-P5 surfaces alert). | N |
| A5-P2 (self) | Flapping credential counts `AuthenticationFailed` with weight 1 → cluster hammers flapping peer until generic `fail_threshold`. | **breaks → successful (amend)** — new proposal A5-P13 introduces `breaker_auth_fail_weight=10`. | N |
| A5-P3 ⊕ A10-P2 (self) | SIGKILL coordinator at 6.2.2 step 3 — peers wait `heartbeat_timeout_ms=10s` (worse than A10-P6's <1s target). | **breaks → successful (amend)** — §5 amend: `coordinator_liveness_timeout_ms < heartbeat_timeout_ms`, default `3 × heartbeat_interval_ms`. | N |
| A5-P4 | Hard dep on A3-P1/P4/P5; if A3-P4 amended to eager retry, `idempotent=false` has no terminal. | holds (A3-P4 is agreed unanimously). | N |
| A5-P5 ⇔ A8-P7 | Under A9-P6 option (i), "deterministic reproduction" row loses vehicle. | holds under option (iii) — RecordedEventSource + A8-P2 + A8-P1 suffice. | N |
| A5-P6 ⇔ A8-P4 | Deadman skipped >K=16 cycles on burst → appears stale. | holds (1.6 ms < 250 ms watchdog). | N |
| A5-P7 | RDMA backend internal retry adds 200 ms → caller sees timeout but write still pending (duplicate delivery). | holds (dedup window 4096; A5-P10 class). | N |
| A5-P8 | DEFER admission-pressure with no upper deferral bound → memory exhaust. | holds (amendment ties each policy value to A8-P5 AlertRule; DEFER has `max_deferred`). | N |
| A5-P9 | A9 YAGNI: IDLE-with-probe would suffice. | holds (IDLE-with-probe has pre-probe reassign race). | N |
| A5-P10 | Future `REMOTE_COLLECTIVE_FLUSH` handler skipped annotation. | holds (doc-lint CI rule makes attack moot). | N |
| Cross-proposal A5-P2 + A5-P9 + A10-P6 + A6-P2 | 6.2.2 replay → oscillating HEALTHY ↔ SUSPECT ↔ HALF_OPEN livelock under recovering network + credential churn. | **breaks → successful (amend via new proposal A5-P11)** — unified peer-health FSM authored. | N |

### 1.6 A6 (security) stress attacks

| target | attack summary | resolved? | A1 hot-path veto |
|---|---|---|---|
| A6-P1 | Enumeration completeness vs new boundaries (A2-P6 registry, A7-P4, A8-P4 dump, A5-P3 failover). | holds (existing rows cover; no new boundary). | N |
| A6-P2 (self) | **6.2.2 replay with malicious demoted-coordinator** — attacker retains cert, resumes REMOTE_SUBMIT after failover. | **breaks → successful (amend)** — add `coordinator_generation: uint64_t` to `HandshakePayload` + `StaleCoordinatorClaim` reject rule. | N |
| A6-P3 | Attacker sends `task_count=2^31-1`, `edge_count=2^31-1`, `args_blob=16 MiB`. | holds (single length-prefix guard; ≤1 compare, no alloc on reject). | N |
| A6-P4 | Plaintext TCP downgrade attempt. | holds (TLS resistance + fail-closed `require_encrypted_transport=true` default). | N |
| A6-P5 | Replayed `REMOTE_DATA_READY` 100 ms after retirement. | holds (rkey invalidated at `SUBMISSION_RETIRED`). | N |
| A6-P6 | v1 in-memory ring: security events enumeration per 6.1.2 steps. | holds (5 named event classes cover scenario). | N |
| A6-P7 | Attacker-controlled Node₁ registers unsigned binary via REMOTE_SUBMIT. | holds (multi-tenant `FunctionNotAttested`). | N |
| A6-P8 | DLPack capsule with overflowing stride × shape. | holds (A1-P11 + A6-P8b `max_import_bytes`). | N |
| A6-P9 | Forged `logical_system_id=tenant-B` inbound. | holds (drop with `CrossTenantDenied`). | N |
| A6-P10 | Sink filtering under dump. | holds once A8-P4 adopts caller-scoped default. | N |
| A6-P11 | Python-side `register_factory(malicious_handler)` post-`freeze()`. | holds (`RegistrationClosed`; audit event). | N |
| A6-P12 (self) | Contingent on A9-P6 option. | **successful (amend)** — pivot to "signed schema + format frozen at ADR-011-R2 time; REPLAY engine deferred to v2". | N |
| A6-P13 | Tenant-A flood at 10× QPS. | holds (partitioned counter; `TenantQuotaExceeded`). | N |
| A8-P4 (peer) | `dump_state()` default scope unspecified → tenant-A sees tenant-B. | **breaks → propagate amendment to A8** — default scoped to caller's `logical_system_id`; unscoped needs `diagnostic_admin_token`. | N |
| A8-P5 (peer) | `AlertRule.logical_system_id` not named in R2 amendment. | **successful (amend-propagation)** — A8 add field. | N |
| A8-P7 (peer) | IFaultInjector sim-only wording. | holds (adequate). | N |
| A5-P5 ⇔ A8-P7 chaos matrix | Must enumerate 5 A6 categories (handshake-downgrade, cert-rotation, rkey-race, quota-skew, audit-sink). | **uncertain** — pending matrix file existence. | N |

**A6 totals (§8.24):** 27 attacks, 25 holds, 0 breaks, 2 uncertain (pending adjacent amendments).

### 1.7 A7 (modularity) stress attacks

| target | attack summary | resolved? | A1 hot-path veto |
|---|---|---|---|
| A7-P1 | D6 cycle removal after amendments. | holds (strictly descending layer edges confirmed via 6.1.1 + 6.1.2 ledger). | N |
| A7-P2 | "11th consumer" wanting Wiring+Completion but not Submit — narrow surface? | holds (role interfaces addressable separately). | N |
| A7-P3 | Handle-type inversion pulling hal symbols. | holds (core::TaskHandle compiles without hal). | N |
| A7-P4 (self) | R2 amendment reads "registry dispatch" without pinning owner module → future edit might land in `transport/` by inertia, unwinding SRP. | **successful (amend)** — second-pass pin: Invariant I-DIST-1 enforces header non-includable from `transport/` via IWYU-CI. | N |
| A7-P5 | `distributed_scheduler` stress-link against only `ISchedulerLayer.h`. | holds. | N |
| A7-P6 (self) | `runtime::composition` sub-namespace lacks ADR → fragile D1 argument. | **successful (amend)** — record **ADR-A7-R3** freezing decision + promotion triggers. | N |
| A7-P7/P8/P9 | Forward-decl / ScopeHandle / MemoryError dedupe. | holds. | N |
| A2-P6 (peer) | Handler registry smuggling new `DepMode` variants. | holds (Invariant I-DIST-1 + D6 + closed enum in `core/`). | N |
| A2-P3 | Open extension vs closed hot-path enum policy. | holds (policed at D6 layering). | N |
| A2-P5 | Single BC policy table for ~30 interfaces → cohesion risk. | holds (distributed per-interface, centralized index). | N |
| A6-P6 | `IAuditSink` as new top-level module? | holds (sibling in `profiling/`; fission trigger at N≥6 sinks noted). | N |
| A6-P9 | `logical_system_id` on MessageHeader — framing or protocol? | holds (routing framing in `transport/`, validation in `distributed/`). | N |
| A6-P11 | `register_factory` gate centralizing cross-module concern. | holds (single guard function in `runtime/`, per-module factories retained). | N |
| A8-P1/P2/P4/P5/P11 | Various DfT seam attacks. | holds (all). | N |
| A9-P3 | Removing collectives forces N caller re-implementations? | holds (OCP via future `ICollectiveOps` sibling). | N |
| A9-P4/P5/P7 | Drop `Kind` / unify admission / fold source config. | holds. | N |
| A10-P6/P7/P9 | Heartbeat thread ownership / sharded TaskManager / WorkStealing×RETRY gate. | holds. | N |

**A7 totals:** 32 rows, 32 holds, 0 breaks.

### 1.8 A8 (testability) stress attacks

| target | attack summary | resolved? | A1 hot-path veto |
|---|---|---|---|
| A1-P1..P14 | 14 rows — can test observer reconstruct scenario from metrics + trace? | holds (all 14). | N |
| A2-P1/P2/P3/P5/P6/P9 | Versioning + closed enums + handler registry observability. | holds (all). | N |
| A3-P1..P15 | State machine + admission + expected<>. | holds (all). | N |
| A5-P1..P10 | Backoff / CB / fail-fast / idempotent / chaos / watchdog / timeout / degradation / QUARANTINED / DS4. | holds (all). | N |
| A6-P1..P14 | Boundary / mTLS / bounds / rkey / audit / attestation / KV / sinks / register_factory / REPLAY / rate-limit / keys. | holds (A6-P12 conditional on A9-P6 option iii). | N |
| A7-P1..P9 | All SOLID edits — observability preserved. | holds. | N |
| A8-P6 (self) | "any-owner-deterministic merge" doesn't spell out tie-breaker. | **successful (amend)** — add `min(node_id)` in youngest all-online epoch; `skew_max_ns ≤ 100 µs`. | N |
| A8-P1/P2/P3/P4/P5/P7/P8/P9/P10/P11/P12 | Self-stress vs A1/A7/A9 counter-arguments. | holds (all). | N |
| A9-P1/P3/P4/P5/P7/P8 | Various simplicity cuts. | holds (all). | N |
| A10-P1/P3/P4/P6/P7/P8/P9 | Scale / consistency / heartbeat / TaskManager shard / assignment log. | holds (all). | N |

**A8 totals:** 77 proposal rows stress-attacked → 77 holds, 0 breaks, 0 uncertain; 1 minor amendment (A8-P6 tie-breaker).

### 1.9 A9 (simplicity) stress attacks

| target | attack summary | resolved? | A1 hot-path veto |
|---|---|---|---|
| A1-P4/P9/P12/P13 | Two-tier hot/cold + bitmask fallback + batched default + 1 KiB ring-slot preserves hot path. | holds. | N |
| A2-P1 | R2 amended text still reads as **blanket versioning** on every public data contract — YAGNI per G2. | **uncertain → successful (amend)** — narrow to multi-node wire messages + persistent artifacts; fold blanket obligation into A2-P5. | N |
| A2-P2/P3/P4/P5/P8/P9 | Closed-for-v1, open-enum semantics, trace versioning. | holds (all). | N |
| A2-P6 | Abstract class with single v1 impl (rubric 4 violation) — synth amendment is rename not retraction. | **uncertain → successful (amend)** — demote to free function in `protocol.hpp`; abstract class waits for ADR + second backend. | N |
| A2-P7 | async-policy seam. | holds under Q-record reframe → A9 flips to agree (§6). | N |
| A3-P1/P2/P8 | ERROR state / expected<>/cyclic detection. | holds (correctness overrides G2). | N |
| A5-P1/P2/P3/P4/P5/P6/P7/P8/P10 | Various reliability proposals. | holds (all). | N |
| A5-P9 | **Still** A9's sole dissent: `QUARANTINED` is a time-windowed sub-state of `UNAVAILABLE`; one state + `unavailable_since` expresses same. | **breaks (DRY) → successful (amend)** — fold into `UNAVAILABLE + {Permanent, Quarantine(duration)}` policy variants. | N |
| A6-P1..P14 | All hold under two-tier / gated / doc-only classification. | holds. | N |
| A7-P1..P9 | Role split + cycle break + SRP moves. | holds. | N |
| A8-P1..P12 | All DfT seams / histograms / alert rules / PhaseIds. | holds. | N |
| A10-P1/P3/P4/P6/P7/P8/P9 | All hold under amended text. | holds. | N |

**A9 summary:** introduces **three minor amendments** (A2-P1 scope narrow, A2-P6 demote, A5-P9 DRY-fold); does NOT flip any previously-agreed proposal to disagree.

### 1.10 A10 (scalability) stress attacks

| target | attack summary | resolved? | A1 hot-path veto |
|---|---|---|---|
| A1-P1/P2/P3/P4/P5/P6/P14 | At 8× Pods / 10K tasks — per-Layer scaling. | holds (all). | N |
| A2-P6 | At 8× Pods vtable stall frequency. | holds (CRTP/final devirt). | N |
| A3-P12 | `drain()` cross-Pod semantics at 8× Pods. | holds (depends on amended A10-P3 table rows). | N |
| A3-P13 (peer) | Dup-detect window is node-global → 28 pairs at 8 Pods overflow. | **uncertain → soft handoff to A3** — tag window as **per-peer-pair**. | N |
| A5-P1 | Synchronized retry bursts after single peer crash. | holds (jitter). | N |
| A5-P2 | Per-peer state O(peers × Pods). | holds (bounded). | N |
| A5-P3 (self A10-P2a) | **"Deterministic fail-fast" read literally = cluster-wide fail-closed** when one coordinator dies → 7 peer Pods abort unnecessarily. | **breaks → successful (amend)** — **pin scope to failed Pod only**; `cluster_view` generation bumps to surviving-coordinator list. | N |
| A5-P7 | Fabric saturation at 8× Pods causes en-masse timeouts. | holds (A5-P8 degradation covers). | N |
| A5-P8 | 8× Pod admission-saturation not in degradation matrix. | holds (externalized rules file parameterizes). | N |
| A5-P10 | Dup-detect window × idempotency at 8× fan-out. | holds (co-depends on A3-P13 per-pair fix). | N |
| A6-P2 | TLS cost per new peer at 8× Pods. | holds (one-time; steady-state not hot). | N |
| A6-P3 | 8× REMOTE_SUBMIT fan-in aggregate entry-gate rate. | holds (1 compare per message). | N |
| A8-P6 | At 8× Pods clock-sync accuracy determines A10-P3 `cluster_view` auditability. | holds (soft dep ADR tag). | N |
| A10-P1 (self) | `shards=1` default at 8× Pods — silent P4 ceiling. | holds once A10-P7 deployment cue lands. | N |
| A10-P3 (self) | R2 amendment lists only intra-Pod state; 6.1.2 + 6.2.2 need `cluster_view`, `group_availability`. | **uncertain → successful (amend)** — add rows `cluster_view`, `group_availability`, `remote_proxy_pool`, `outstanding_submission_set`. | N |
| A10-P4 | Coordinator role not explicitly in DS1 classification. | holds (R2 amendment already covers). | N |
| A10-P6 (self) | "Dedicated heartbeat thread" is singular — at ≥32 peers/Node cadence × fan-in saturates. | **uncertain → successful (amend)** — shard per 32 peers, cap=4 thread-pool. | N |
| A10-P7 (self) | `admission_shards=1` default at 8× Pods → silent P4 ceiling. | **uncertain → successful (amend)** — add deployment cue in `05-physical-view.md` (threshold `concurrent_submitters × cluster_nodes ≥ 64`). | N |
| A10-P8 | One-page summary at Pod scale readability. | holds. | N |
| A10-P9 | Under A9-P6 option (iii) does FUNCTIONAL-mode assignment log still work? | holds (live-only, REPLAY not required). | N |

**A10 totals:** 19 holds, 4 breaks/uncertain → all amended in §5; 1 soft handoff to A3; 0 blocking; 0 veto.

---

## 2. Scenario replay results (from `06-scenario-view.md`)

Each reviewer replayed specific scenarios in R3 §8. Summary per reviewer:

| Reviewer | Scenarios used | Pass/Fail/Notes |
|---|---|---|
| **A1** | S1 = 6.1.1 (single-node hierarchical, `:9-31`); S2 = 6.2.1 (AICore hang, `:83-98`); S3 = 6.1.3 (SPMD, `:61-79`). | **All HPI-flagged peer proposals hold**; A1 own 14 proposals stress-tested, 13 **hold** + 1 **break-minor (A1-P6)**; 2 peer **uncertain → amend** (A5-P6 deadman elision; A8-P3 histogram region). 0 vetoes applied. |
| **A2** | **Canonical:** 6.1.2 (distributed multi-node, `:43-55`) walked step-by-step under A2-P6 amended with hypothetical TcpBackend alongside v1 RdmaBackend. Secondary: 6.1.1, 6.2.1, 6.2.4. | Steps 1–6 + 8–11 **hold** with §5 header-independence amendment. **Step 7 (`copy_to_peer`) uncertain** — RDMA rkey vs TCP TLS binding don't compose; resolved by A2-P7 Q-record **third-axis** amendment (transport-capability semantics). 107 proposals hold overall. |
| **A3** | **Primary:** 6.1.3 (SPMD, `:61-79`) and 6.2.3 (Task Slot Pool Exhaustion, `:116-128`). Cross-ref: 6.1.1, 6.2.1. | 5 **`breaks (minor)`** (A3-P4 SPMD agg edge; A3-P5 spmd_index retention; A3-P7 SPMD precondition sub-codes; A3-P10 scenario text mapping; A5-P4 scenario idempotent clarification) + 5 **`uncertain`** (A1-P8 sub-codes; A1-P11 single-pass validation; A7-P2 ordering; A9-P4 spmd->size==1 edge; A9-P5 scenario text rename). 0 hard breaks; all amendments one-line. |
| **A4** | 6.1.1 (`:15-30`), 6.1.2 (`:43-55`), 6.2.1 (`:83-98`), 6.2.4 (`:130-142`). | 22 `holds`, 0 `breaks`, 2 `uncertain (minor)` — A5-P9 `QUARANTINED` WorkerState glossary gap (→ R3-P1) and A3-P1 terminal-state spelling triad (`COMPLETED(error)`/`ERROR`/`FAILED`) across `06-scenario-view.md:91/93/95/108/111` (→ R3-P2). |
| **A5** | S1 = 6.2.1 (`:83-98`); S2 = 6.2.2 (`:100-114`); S3 = 6.2.3 (`:116-128`); S4 = 6.2.4 (`:130-142`). | All 10 own A5-P1..P10 hold (with 2 breaks → amended: A5-P2 via new A5-P13 weight; A5-P3 via §5 `coordinator_liveness_timeout_ms`). **Cross-proposal livelock** under S2 flapping-peer surfaces → new **A5-P11 unified peer-health FSM** required. |
| **A6** | **S1 = 6.1.2 with attacker-controlled Node₁** (`:35-58`); **S2 = 6.2.2 with malicious partial state** (`:100-114`). | 27 attacks → 25 `holds`, 0 `breaks`, 2 `uncertain` (A8-P4 dump scope + A5-P5 chaos matrix pending). Key findings: A6-P2 `coordinator_generation` binding required to defeat stale-coordinator replay under 6.2.2. |
| **A7** | 6.1.1 (`:17-29`) + 6.1.2 (`:43-55`) step-by-step ledger verifying strictly descending layer edges `bindings > runtime > scheduler > distributed > transport > hal > core`. | 32 proposals attacked, **32 holds**. D6 DAG claim confirmed (no reverse/sibling edges). 2 own minor amendments (A7-P4 Invariant I-DIST-1; A7-P6 ADR-A7-R3). |
| **A8** | 6.1.3 (`:61-79`) and 6.2.1 (`:83-98`) replayed end-to-end under amended design. | **77 rows stress-attacked, 77 holds, 0 breaks, 0 uncertain.** Test-observer can reconstruct both scenarios from trace alone; L1 default + L2 opt-in bridges hold. 1 amendment (A8-P6 tie-breaker rule). |
| **A9** | **Primary rotation:** 6.1.1 (`:15-30`) + 6.2.3 (`:116-128`). Additional: 6.1.2 (`:35-58`), 6.2.1, 6.2.2, 6.2.4. | All proposals hold in scenario replays except 3 stress-surface concerns → owner-routable amendments (A2-P1 scope narrow, A2-P6 demote to free function, A5-P9 DRY-fold into UNAVAILABLE + policy). 0 disagree flips on previously-agreed. |
| **A10** | **6.1.2 at 8× Pods** (`:35-58`); **6.2.2 in large fleet** (`:100-114`). | 19 `holds` / 4 `breaks-or-uncertain` → all amended in §5 (A5-P3/A10-P2a blast-radius; A10-P3 rows; A10-P6 heartbeat shard; A10-P7 deployment cue); 1 soft handoff to A3 (per-pair dup window). No new disputes; HPI=none on all amendments. |

**Aggregate:** every R3 scenario replay produced either `holds` or a minor/doc amendment; **no reviewer reported a scenario replay that forces withdrawal of a previously-agreed proposal**.

---

## 3. Votes on disputed proposals (A2-P7, A9-P2, A9-P6)

Weight: **A1 = ×2** (hot-path veto), all others = ×1.

### 3.1 A2-P7 — reserve async-policy extension seam

Synthesizer recommended landing zone: **Q-record only in `09-open-questions.md`; no interface in v1.**

| Reviewer | Weight | Vote | Rationale (1-line) | Blocking | Override |
|---|---|---|---|---|---|
| A1 | 2 | agree | Q-record = zero hot-path impact (E1-compliant doc only); scenario replay untouched. | false | false |
| A2 (owner) | 1 | **own / agree** | Owner already committed R2 amendment to Q-record framing; synth matches. | false | false |
| A3 | 1 | agree (flip from abstain) | No interface → no LSP surface; Q-record satisfies G3. Flips back to abstain if final text reintroduces v1 interface slot. | false | false |
| A4 | 1 | agree (flip from abstain) | Stable Q-entry → no glossary churn, no V5 impact; cite D7/V5/G5. Prefer Q15 at end of file (no renumber). | false | false |
| A5 | 1 | agree (flip from abstain) | Pure E5 migration-plan entry; no R1–R6 interaction. Reverts if v1 interface reintroduced. | false | false |
| A6 | 1 | agree | No new plugin path to gate (S4/G2). | false | false |
| A7 | 1 | agree (flip from disagree) | No dead interface ships → D3/SRP restored; G2 satisfied. | false | false |
| A8 | 1 | agree (flip from abstain) | G2/YAGNI + E1/OCP jointly satisfied. | false | false |
| A9 | 1 | agree (flip from disagree) | Flips conditional on Q-record reframe; flips back to disagree if A2's final edit adds any class/interface declaration. | false | false |
| A10 | 1 | agree (flip from abstain) | Q-record matches R2 conditional; HPI=none. | false | false |

**Tally:** 11 weighted / 10 reviewers — all **agree**. 0 disagree, 0 abstain, 0 blocking, 0 override.

### 3.2 A9-P2 — collapse policy-enum/interface pluggability

Synth recommended landing zone: **single `IEventLoopDriver` test-only seam + closed deployment-mode/policy enums + appendix in `08-design-decisions.md` listing future extension interfaces.**

| Reviewer | Weight | Vote | Rationale (1-line) | Blocking | Override |
|---|---|---|---|---|---|
| A1 | 2 | agree | Removes 2 vcalls on Stage B; test seam gated by `enable_test_driver`; release-path unaffected. | false | false |
| A2 | 1 | agree (flip from disagree, withdraw override) | Single seam + closed enums + appendix-of-v2-extensions = E5 migration artifact. | false | false |
| A3 | 1 | agree (flip from abstain) | Single seam sufficient to drive A3-P4/P5 tests deterministically; LSP preserved. | false | false |
| A4 | 1 | agree (flip from abstain) | One new glossary row (`IEventLoopDriver`); closed enums prevent glossary drift. | false | false |
| A5 | 1 | agree (flip from abstain) | Preserves `SINGLE_THREADED + SPLIT_COLLECTION`; A5-P6 watchdog cadence unaffected. | false | false |
| A6 | 1 | agree | Closed enums reduce A6-P11 gating surface; no new attack path. | false | false |
| A7 | 1 | agree (held) | D4 narrow seam ≪ five enum-dispatch interfaces with one impl each. | false | false |
| A8 | 1 | agree (flip from disagree) | X5 DfT rubric satisfied by single test seam. | false | false |
| A9 (owner) | 1 | agree (self; amendment augmented with **trigger conditions** in appendix). | Amendment preserved; A2's OCP closed by appendix + triggers. | false | false |
| A10 | 1 | agree | Matches A10 R2 conditional agreement verbatim. | false | false |

**Tally:** 11 weighted / 10 reviewers — all **agree**. 0 disagree, 0 abstain, 0 blocking, 0 override (A2's override on this proposal **withdrawn**).

### 3.3 A9-P6 — defer PERFORMANCE/REPLAY simulation

Synth recommended landing zone: **option (iii)** — ship FUNCTIONAL-only implementation; keep `SimulationMode` enum open + REPLAY scaffolding (placeholder interfaces declared but not factory-registered); ADR-011-R2 names triggers; A6-P12 re-scoped to "schema frozen; no v1 implementation".

| Reviewer | Weight | Vote | Rationale (1-line) | Blocking | Override |
|---|---|---|---|---|---|
| A1 | 2 | agree (option (iii)) | Sim is compile-time-selected; REPLAY scaffolding is L ≥ 2 profiling-level only; zero hot-path cost; indifferent between options (ii) and (iii). | false | false |
| A2 | 1 | agree option (iii) (flip from disagree, withdraw override) | Named triggers in `09-open-questions.md` = E5 migration plan; A6-P12 conditional amendment. | false | false |
| A3 | 1 | agree option (iii) | Preserves A3-P6 traceability for FR-10 + deterministic replay hook for A3-P4/P5 tests. | false | false |
| A4 | 1 | agree option (iii) (flip from abstain) | Minimum doc-consistency cost; glossary entries for `SimulationMode`/`Replay` stay; ADR-011-R2 triggers listed. | false | false |
| A5 | 1 | agree option (iii) (flip from abstain; conditional) | Preserves both FUNCTIONAL day-1 and REPLAY day-2 reproduction vehicles for chaos matrix. **Flips to disagree if option (i) chosen.** | false | false |
| A6 | 1 | agree option (iii) (flip from disagree) | A6-P12 amendment ("signed schema + frozen format") is the v1 forensic anchor; REPLAY engine can ship day-2. | false | false |
| A7 | 1 | agree option (iii) (conditional) | Modularity dominates over option (ii). **Flips to (ii) if A6 names concrete v1 forensic scenario requiring REPLAY data in R4.** | false | false |
| A8 | 1 | agree option (iii) (flip from disagree) | `RecordedEventSource` + A2-P9 versioned trace schema = day-2 landing surface. **Reverts to disagree if option (i) chosen (full deferral, enum closed).** | false | false |
| A9 (owner) | 1 | agree option (iii) | Final form: v1 FUNCTIONAL; enum open; placeholders declared but not registered; ADR-011-R2 triggers; A6-P12 rescoped. | false | false |
| A10 | 1 | agree option (iii) | A10-P9 FUNCTIONAL-mode assignment log preserved. | false | false |

**Tally:** 11 weighted / 10 reviewers — all **agree under option (iii)**. 0 disagree, 0 abstain, 0 blocking, 0 override (A2's override **withdrawn**). Three reviewers (A5, A7, A8) explicitly name re-vote triggers if a different option lands.

### 3.4 Override / blocking flags — aggregate across all three disputed

- **Blocking objections filed:** 0 across all three proposals.
- **Override requests filed in R3:** 0.
- **R2 override requests withdrawn at R3:** A2's overrides on A1-P9, A9-P2, A9-P6 all **withdrawn** (§6 of each respective reviewer).

---

## 4. New proposals introduced in R3

Per-reviewer, numbered as `A<x>-P<n+1>…` (or reviewer-defined id where given).

| Reviewer | New id | Severity | Target / Summary | Rationale | Hot-path impact |
|---|---|---|---|---|---|
| **A1** | — | — | **None.** A1-P15 conditional branch from R2 §4 explicitly **withdrawn** (merge it was contingent on was accepted). | — | — |
| **A2** | — | — | **None.** All A2-side concerns routed as amendments to existing proposals (A2-P5, A2-P6, A2-P7, and A6-P12 conditional). | — | — |
| **A3** | — | — | **None.** All R3 stress findings routed to amendments of existing A3-P1..P15 or co-edits to peers (A5-P4, A9-P5, A7-P2, A1-P8, A9-P4). | — | — |
| **A4** | **A4-R3-P1** | medium | Add `WorkerState` row to `appendix-b-codebase-mapping.md` + glossary entry in `appendix-a-glossary.md`; enumerate `READY, BUSY, DRAINING, RETIRED, FAILED, QUARANTINED` (last conditional on A5-P9). | A4 rubric #6 symmetry with `TaskState` (A4-P8/P9); otherwise regresses the moment A5-P9 lands. | **none** |
| **A4** | **A4-R3-P2** | low | Unify terminal-failed-Task spelling across `06-scenario-view.md:91,93,95,108,111`. Three spellings (`COMPLETED(error)`, `ERROR`, `FAILED`) collapse to canonical `COMPLETING → ERROR` (Task) + `FAILED` (Worker). | D7 + V5 after A3-P1 lands. | **none** |
| **A5** | **A5-P11** | medium | Author a single unified **peer-health FSM** co-owned by A5 (breaker) + A10 (heartbeat) + A6 (auth). New `modules/distributed.md §3.5` with states `{HEALTHY, SUSPECT, OPEN, HALF_OPEN, QUARANTINED, LOST, AUTH_REVOKED}`. | Prevents cross-proposal livelock between A5-P2/P9 + A10-P6 + A6-P2 demonstrated under 6.2.2 replay. | **none** |
| **A5** | **A5-P12** | low | Name `ErrorCode::ResourceExhausted` as the failure class for staging `BufferRef` allocation failure on A1-P13's slow path. | Closes R2 Con 1 residual; one sentence in `modules/hal.md` + one row in `modules/scheduler.md` §5 error table. | **none** |
| **A5** | **A5-P13** | low | Record `breaker_auth_fail_weight` (default 10) cross-reference in both A5-P2 and A6-P2 sections; add config field. | Prevents credential-rollout retry storms from defeating `fail_threshold`. | **none** (one compile-time-constant read on already-slow breaker-update path) |
| **A5** | **A5-P14** | medium | Q-record in `09-open-questions.md` for "Orchestration-composed collective partial-failure → FailurePolicy mapping" as v2 ADR trigger post A9-P3. | Pre-empts a known R4 gap; follow-up ADR triggered, not pre-written. | **none** |
| **A6** | — | — | **None.** All A6-side concerns surface as stress-attack amendments (own: A6-P2 generation binding; A6-P12 reframe) or §7 cross-aspect propagations (A8-P4 scope; A8-P5 AlertRule id). | — | — |
| **A7** | — | — | **None.** Stress-revealed amendments routed to own A7-P4 (Invariant I-DIST-1) and A7-P6 (ADR-A7-R3). | — | — |
| **A8** | — | — | **None.** One medium amendment folded into existing A8-P6 (tie-breaker rule) rather than new ID. | — | — |
| **A9** | — | — | **None.** Three concrete amendments to peer proposals (A2-P1, A2-P6, A5-P9); no new A9 proposals. | — | — |
| **A10** | — | — | **None.** Stress-round deltas are amendments to A10-P2a/P3/P6/P7 (§5) plus one piggy-back clarification to A3-P13 (per-pair dup-detect window). | — | — |

**Totals:** 6 new R3 proposals introduced, all by A4 (2) and A5 (4); 0 hot-path impact across all 6.

---

## 5. Late blocking objections on previously-agreed proposals

Comprehensive scan across all 10 R3 §6 / §8 / §9 sections:

| Reviewer | Late blocking? | Notes |
|---|---|---|
| A1 | **none** | 0 vetoes applied; 0 blocking. §10 Status explicitly: "A1 veto applications this round: 0". |
| A2 | **none** | R3 §6.3 tally: "blocking = 1 (A7-P1, hard-rule D6 — **unchanged** from R2 carryover, in favor)". No new blocking. |
| A3 | **none** | R3 §6 / §8.3: "Blocking objections filed by A3: 0". |
| A4 | **none** | R3 §8 / §9: "No proposal `breaks` under A4's stress attack; no `blocking=true` filed". |
| A5 | **none** | R3 §6 / §9: "No blocking objections. No A1 veto triggered. No A5 override_request". |
| A6 | **none** | R3 §9: "Blocking objections raised this round: 0". |
| A7 | **none** | R3 §8 / §9: 0 blocking filed across R1/R2/R3; 0 overrides ever. |
| A8 | **none** | R3 §9: "Blocking objections raised: 0". |
| A9 | **none** | R3 §9: "Override requests: 0. Blocking objections: 0. A9's primary rules (G2, G4) are design-discipline soft rules; A9 does not file blocking". |
| A10 | **none** | R3 §9: "No A10 hot-path concerns; no A10 blocking objections; no A10 override requests". |

**Aggregate across all reviewers:** **zero new late blocking objections** raised in R3 on any previously-agreed proposal. The only `blocking=true` recorded anywhere is A2's pre-existing in-favor hard-rule citation on **A7-P1** (D6 cycle-break), which is a carry-forward from R2 and reinforces agreement rather than opposing it.

Two reviewers declared conditional re-vote triggers (not blocking today):
- **A5** on A9-P6: flips to disagree if final text lands option (i) (full defer including enum).
- **A8** on A9-P6: reverts to disagree if option (i) (full deferral, enum closed) lands.
- **A7** on A9-P6: flips from (iii) to (ii) if A6 names a concrete v1 forensic scenario requiring REPLAY data in R4.
- **A3, A5, A9** on A2-P7: flip back to abstain/disagree if A2's final edit reintroduces any v1 interface slot (must remain a pure Q-record).

---

## Appendix — R3 summary counts

| Metric | Value |
|---|---|
| Reviewers reporting convergence | 10 / 10 |
| Disputed proposals voted | 3 (A2-P7, A9-P2, A9-P6) |
| All three disputed proposals land at | **agree** (A9-P6 under option (iii)) |
| Blocking objections raised in R3 | **0** |
| Override requests raised in R3 | **0** |
| R2 override requests withdrawn at R3 | 3 (all A2-filed: A1-P9, A9-P2, A9-P6) |
| A1 hot-path vetoes in R3 | **0** |
| New R3 proposals introduced | 6 (A4 ×2, A5 ×4) |
| Minor R3 amendments to existing agreed proposals | ~20 (distributed across A1-P6, A2-P5, A2-P6, A2-P7, A3-P4/P5/P7/P10, A5-P2/P3, A5-P6, A6-P2, A6-P12, A7-P4, A7-P6, A8-P3, A8-P4, A8-P6, A10-P2a, A10-P3, A10-P6, A10-P7) |
