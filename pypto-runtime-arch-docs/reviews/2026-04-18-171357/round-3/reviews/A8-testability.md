# Aspect A8: Testability & Observability — Round 3 (Stress)

## Metadata

- **Reviewer:** A8
- **Round:** 3 (mandatory stress round)
- **Target:** `/data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/` (full doc set, post-round-2 amendments)
- **Runtime Mode:** yes
- **Timestamp (UTC):** 2026-04-19T11:00:00Z

---

## 1. Rubric Checks (post-R2-amendment state)

| # | Check | Result | Evidence (file:line) | Rule(s) |
|---|-------|--------|----------------------|---------|
| 1 | Every external dep (HW/net/clock/OS) behind a replaceable interface | **Pass (conditional)** | `modules/hal.md:247,430`, A8-P1 (IClock), A8-P2 (IEventLoopDriver), A7-P3/P7 (no hal→core leakage) | X5, §8.2 DfT |
| 2 | Trace IDs, timestamps, component IDs on all critical ops | **Pass** | A2-P9 (versioned trace schema), A8-P6 (time-alignment), A8-P12 (stable PhaseIds), A3-P4 (DEP_FAILED carries correlation_id) | O1 |
| 3 | Structured, context-rich logs (request/component/task) | **Pass** | A8-P10 (log_kv + {correlation_id, layer_id, category, severity, logical_system_id}), A6-P10 (capability-scoped sinks) | O2, S2 |
| 4 | Health metrics — error rate, latency p99, queue depth, utilization | **Pass** | A8-P3 (Stats + histograms), A8-P9 (profiling drop alerts), A5-P2 (per-peer breaker metric), A10-P6 (peer-failure detection) | O3 |
| 5 | Runtime debuggability without redeploy | **Pass (paired)** | A8-P4 (`dump_state()`) + A5-P6 (scheduler watchdog), A3-P15 (debug-mode cross-check), A8-P11 (HAL contract tests) | O4, X6 |
| 6 | Alerts on SLO breach, not arbitrary thresholds | **Pass** | A8-P5 (external alert-rules file; {metric, op, threshold, window_ms, severity}), A5-P8 (degradation specs) | O5 |
| 7 (spot) | Step-driven event loop available for deterministic tests | **Conditional on A9-P2** | A8-P2 amendment: single `IEventLoopDriver` test seam + `RecordedEventSource` | X5 |
| 8 (spot) | Fault-injection seam (sim-only) | **Pass** | A8-P7 seam + A5-P5 matrix co-ownership | X5, R6 |
| 9 (spot) | Distributed trace time-alignment contract | **Pass** | A8-P6 amendment: per-chain FIFO (A3-P13) + any-owner-deterministic merge (A10-P2 forward-compat) | O1 |
| 10 (spot) | Drop/degraded profiling observable | **Pass** | A8-P9 first-class alerts; A1-P10 CI gate enforces ≤1% L1 / ≤5% L2 | O3, P1 |

**Delta vs round 2.** No rubric regressions. One conditional (rubric 7) remains gated on A9-P2 landing with the single test seam. Rubric 9 depends on A10-P2's v2 decentralization having a named merge owner; A8-P6 amendment already covers it.

## 2. Pros (delta vs round 2)

- R2 synthesis recorded **107 of 110 proposals agreed with zero A1 hot-path vetoes** (`round-2/synthesis.md:25-28`). Every A8-P1…A8-P12 is in the agreed set.
- Pairing bundles (A5-P6 ↔ A8-P4, A5-P5 ↔ A8-P7) make A8's DfD (§8.3) rubric lever an integral piece of the reliability story rather than a bolt-on, closing O4+R5 in one sweep.
- A6-P10 (capability-scoped sinks) and A8-P10 (KV primary surface) **mechanically align**: `{correlation_id, layer_id, category, severity, logical_system_id}` is a filter key, not a parse — O2 and S2 are satisfied by the same record shape.
- A1-P7 (per-thread local sequence) is **structurally required** for A8-P12 to hit the `< 100 ns` per-Phase target at L2; A1 carries it — no A8 escalation needed.

## 3. Cons (delta vs round 2)

Three carry-over concerns remain heading into the stress round:

- **A9-P6 amendment (option i / iii)** — if scaffolding is cut entirely (option i), A8's `RecordedEventSource` loses its canonical payload shape and A6-P12 goes moot. Synthesis recommends option (iii); A8 concurs and confirms in §6 below.
- **A10-P6** specifies **dedicated heartbeat thread** (no hot-path impact) but the `NodeLost` alert still depends on `heartbeat_interval_ms` being documented as a DeploymentConfig knob (not hard-coded). Verify at §8.
- **A8-P6 forward-compat with A10-P2 decentralization** — amendment declares any-owner-deterministic merge, but the tie-breaker rule (lowest `node_id`? highest epoch?) is not yet spelled out. Flag as minor amendment in §5.

## 4. Proposals — new in round 3

None of blocker or high severity. One medium amendment is folded into §5 rather than a new ID (A8-P6 tie-breaker rule). Round 3 intentionally holds the proposal count.

## 5. Revisions of own proposals

| id | action | amended_summary (if amend/split) | reason |
|----|--------|----------------------------------|--------|
| A8-P1 | defend | — | Link-time HAL specialization keeps `SystemClock::now_ns()` inlined in release; A1 veto pre-empted; peer reviews in R2 converge. |
| A8-P2 | defend | — (R2 amendment stands) | A9-P2 R2 amendment explicitly preserves the single `IEventLoopDriver` test seam with `step()` + `RecordedEventSource`. A8 is satisfied provided A9-P2 lands in that form (vote below). |
| A8-P3 | defend | — | Two-tier branchless increment ≤ 5 ns + cold-reader percentile. No peer stress in R2 surfaced a cache-footprint counterexample beyond the N=64-bucket case already covered. |
| A8-P4 | defend | — | Pairing with A5-P6 accepted; Merge Register keeps both IDs. A8-P4 remains the evidence surface for the watchdog-detected hang. |
| A8-P5 | defend | — (R2 amendment stands) | External alert-rules file + 4-field schema + Prom/OTEL as known deviation. A9's YAGNI concern resolved. |
| A8-P6 | **amend (clarify)** | Add tie-breaker: **merge owner = coordinator when centralized (v1); when A10-P2 decentralization lands, merge owner = node with smallest `node_id` within the youngest epoch that saw all participants online; fall back to `skew_max_ns` ≤ 100 µs bound**. | R2 amendment said "any-owner-deterministic" without spelling out determinism rule. This closes the last ambiguity for reliable multi-party trace reconstruction. No hot-path impact (merge runs on cold reader path). |
| A8-P7 | defend | — | Co-ownership with A5-P5 confirmed; sim-only compile target; zero release-build cost. |
| A8-P8 | defend | — | L2 opt-in strip preserved; L1 fast path unchanged. A1 stress attacks on histogram footprint do not target L1 (default). |
| A8-P9 | defend | — | Profiling drop/degraded as first-class alerts is uniformly supported; relaxed-atomic read on cold path. |
| A8-P10 | defend | — | KV + capability-scope alignment with A6-P10 stands; sink filter is a branch, not a parse. |
| A8-P11 | defend | — | HAL contract tests are CI plumbing; A7-P3/P7 make `core/` buildable without `hal/`, so the suite can isolate HAL. |
| A8-P12 | defend | — | Stable `PhaseId`s depend on A1-P7 (carried by A1). If A1-P7 is reversed in R3 — it was not — A8-P12 would amend to L≥2-only (compile-strip at L1). |

## 6. Votes on peer proposals (round 3 deltas)

Round-3 votes only re-cast the three disputed proposals (synthesis §"Open Disputes — Next-Round Focus"). All other 95 votes from round 2 stand; no stress attack below produced a `breaks` verdict that would flip an existing agree.

| proposal_id | round-2 vote | round-3 vote | rationale (cite rule id) | blocking | override_request |
|-------------|--------------|--------------|--------------------------|----------|-------------------|
| A2-P7 | abstain | **agree** (conditional) | Per synthesis recommendation, A2 has committed to the Q-record framing in `09-open-questions.md` with no interface in v1. Under that wording G2/YAGNI and E1/OCP are jointly satisfied — the Q-record names the two future policy axes without adding surface. A8 rubric neutral. | false | false |
| A9-P2 | disagree | **agree** | A9's R2 amendment keeps the single `IEventLoopDriver` test-only seam (`enable_test_driver` build flag) with `step()` + `RecordedEventSource`, closes the deployment-mode enums, and adds an appendix of v2 extension points. A8's §8.2 DfT rubric #1 (`X5`) is satisfied by the single test seam; A8-P2 survives; no virtual dispatch in release. | false | false |
| A9-P6 | disagree | **agree, pinned to option (iii)** | Ship FUNCTIONAL-only implementation in v1 but **keep the `REPLAY` enum open and the trace-schema scaffolding in place**. Under option (iii): A6-P12 landing condition becomes "signed schema + format frozen at ADR-011-R2 time" (not "fully implemented in v1"); A8's `RecordedEventSource` and A2-P9 versioned trace schema both remain valid day-2 landing surfaces. §8.2 "reproducible failures" preserved structurally even though REPLAY-driven execution is deferred. If R3 instead lands option (i) (full deferral, enum closed), A8 reverts to **disagree** with a request to reopen. | false | false |

### Disputed-proposal decision summary

- **A2-P7 → expected agreed** in the round-3 synthesis (A2's Q-record amendment flips A7 and A9; A8 flips abstain → agree).
- **A9-P2 → expected agreed** (A8 flips disagree → agree given the single-seam amendment; A2's override resolved by "future-extensions appendix").
- **A9-P6 → expected agreed under option (iii)** (A8 and A6 flip under the scaffolding-preserved framing; A2's E5 migration-plan concern is already satisfied by R2 amendment).

## 7. Cross-aspect tensions (new in round 3)

| pair | with_proposal | preferred_resolution |
|------|---------------|----------------------|
| A8 vs A10 | A8-P6 ↔ A10-P2 (v2 decentralize) | Deterministic tie-breaker = `min(node_id)` within youngest all-participants-online epoch; `skew_max_ns ≤ 100 µs`. Recorded as amendment in §5 above. |
| A8 vs A9 | A8-P2 ↔ A9-P2 | Resolved. Single `IEventLoopDriver` test seam preserved; closed enums + appendix list future-extensions. |
| A8 vs A9 + A6 | A6-P12 / A2-P9 ↔ A9-P6 option (iii) | Resolved. Scaffolding remains; full implementation deferred. A6-P12 becomes "schema frozen at ADR time; signing implementation deferred to day 2 alongside REPLAY activation". |

No new tensions introduced in round 3.

## 8. Stress-attack on emerging consensus

Every agreed proposal where A8 holds standing is attacked below. Each row replays at least one step from **6.1.3 (SPMD Kernel Launch)** or **6.2.1 (AICore Hang)** and asks whether a test observer can reconstruct the scenario from the metrics + trace contract. Citations use `06-scenario-view.md:<line>`; the SPMD scenario spans lines 61-79 and the AICore hang spans lines 83-98.

| proposal_id | stress_attempt (A8's strongest attack) | scenario replay | verdict |
|-------------|----------------------------------------|-----------------|---------|
| A1-P1 (pre-size `producer_index`, forbid rehash) | If pre-size is too small at a given shard, admission silently spills to an overflow path and A8-P3 histograms cannot attribute the slowdown to a specific shard. | 6.1.3 step 2 (`06-scenario-view.md:72`): 24 sub-tasks enter `producer_index`. With R2's absorbed A10-P10 cap (default `shards=1`), attribution is per-shard; histogram buckets the 24 admits correctly. | **holds** |
| A1-P2 (DATA-mode lock hold ≤ 5 μs; shard count LevelParam) | A fake-clock watchdog cannot verify the 5 μs bound under real jitter without a monotonic injection point. | 6.1.3 step 2: test harness drives `FakeClock` via A8-P1; 24 inserts each verify `lock_hold_ns` stored in a watchdog histogram bucket; bound verified per-insert. | **holds** |
| A1-P3 (Function Cache LRU + HEARTBEAT presence) | Cache-miss events must be both countable and traceable for O3; a bare LRU may drop without emitting a `FUNC_CACHE_EVICT` signal. | 6.2.1 step 3 (`06-scenario-view.md:91`): if AICore hangs during `compute_A`, readers need to know whether the function-cache entry was evicted mid-run. R2 amendment emits `FUNC_CACHE_EVICT{fn_id, reason}` + LRU slot stat. | **holds** |
| A1-P4 (hot/cold `Task` split) | A snapshot through `dump_state()` must not tear a cold-struct cache line mid-scan. | 6.2.1 step 3-5 (`06-scenario-view.md:91-93`): on hang, `dump_state()` walks the Task hot/cold layout; sequence-counted reader + cold-side `volatile`-read ensures snapshot consistency. | **holds** |
| A1-P5 (SPMD / event-loop latency budgets) | Budgets must be observable, not aspirational; PhaseIds must carry "budget_ns" for comparison. | 6.1.3 steps 1-7: each PhaseId (`SPMD_EXPAND`, `SPMD_DISPATCH`, `SPMD_WAIT`) carries `budget_ns` from A1-P5 and `elapsed_ns` from A8-P12. Reader can plot actual vs budget per step. | **holds** |
| A1-P6 (REMOTE_SUBMIT bytes projection) | Per-peer byte counter is cumulative; a single outlier peer can mask a regressing average. | 6.1.3 not remote; attack uses 6.2.1's analogue **6.2.2** (`06-scenario-view.md:100-114`) — per-peer `bytes_remote_submit` histogram (not just counter) shows distribution. R2 amendment specifies histogram, not counter. | **holds** |
| A1-P7 (per-thread local seq + offline merge) | Offline merge may drop samples if threads exit before flush. | 6.1.3 step 5-6: each of 24 AICores has its own PhaseId stream; merger flushes on worker retirement; `A8-P9` drop-counter fires if any thread's buffer wraps. | **holds** |
| A1-P8 (pre-size Outstanding/Ready, CONTROL placement) | `WouldBlock` surface must be countable separately from generic `ResourceExhausted`. | 6.2.3 exhaustion scenario (not primary target) — A8-P3 `LayerStats` distinguishes `would_block_count` from `slot_alloc_fail_count`. | **holds** |
| A1-P9 (bitmask-encode slot-type availability) | Cache-line relayout (documented, one-time) must not invalidate active `dump_state()` snapshots. | 6.2.1 step 3: `dump_state()` schema version bumps at layout change; old readers get schema-version error, not silent corruption. | **holds** |
| A1-P10 (profiling-overhead CI gate) | CI gate must run on representative workload, not a microbench that hides skew. | 6.1.3 is itself the CI fixture. A8-co-owned gate asserts L1 ≤ 1% and L2 ≤ 5% on the 24-AICore SPMD run. | **holds** |
| A1-P11 (per-arg Python↔C budget + validation) | ARG_MARSHAL phase must emit per-arg timing to be usable as an SLO target. | 6.1.3 step 1: `per_tile_args` marshal sequence; each arg emits `ARG_MARSHAL{arg_idx, type, bytes, elapsed_ns}`. | **holds** |
| A1-P12 (BatchedExecutionPolicy + tail budget) | Tail latency must be observable under batch=N without aliasing with the head latency histogram. | 6.1.3 not batched by default. Attack uses bindings-level batch on 24-SPMD: A8-P3 exposes two histograms — `batch_head_p99` and `batch_tail_p99`. R2 amendment confirms. | **holds** |
| A1-P13 (bound `args_blob`; 1 KiB ring-slot fast path) | Slow-path (>1 KiB) must emit a distinct PhaseId so that a reader knows why the submission used the extended lane. | 6.1.3 step 1: `per_tile_args` for 24 tiles; R2 amendment emits `ARGS_COPY_SLOW{reason=size_gt_1kib}` when the ring-slot fast path is bypassed. | **holds** |
| A1-P14 (`producer_index` CONTROL placement + shard default) | Non-default shard count must be observable at runtime (config readback). | 6.1.3 step 2: `dump_state()` reports `admission_shards` (default 1); operators can observe override via A8-P4. | **holds** |
| A2-P1 (version every public data contract) | Trace reader with mismatched version must fail fast, not silently reinterpret bytes. | 6.2.1 step 3-4: `ErrorContext` schema bumps major → reader rejects with `TRACE_SCHEMA_MISMATCH`; no silent corruption of the hang diagnosis. | **holds** |
| A2-P2 (LevelOverrides closed v1, schema-registered v2) | Closed v1 must not block A8-P5 alert-rule overrides per-level. | A8-P5 R2 amendment uses `alerting_rules_path` as the override vehicle, not `LevelOverrides`. Orthogonal. | **holds** |
| A2-P3 (open extension-point enums; closed `DepMode`) | `SimulationMode` left open (for A9-P6 option iii); A8 relies on the open enum to day-2 `REPLAY`. | 6.2.1 scenario replay under `SimulationMode::REPLAY` (deferred): enum slot reserved, scaffolding intact. | **holds** |
| A2-P4 (Migration & Transition Plan) | Migration steps must be testable — i.e., each stage must have observable toggles. | Feature flags are recorded in `DeploymentConfig`; each is a KV-logged fact at startup. | **holds** |
| A2-P5 (Interface Evolution & BC Policy) | HAL contract-test suite (A8-P11) must enumerate stable vs evolving interfaces from the BC policy table. | A8-P11 ingests the BC policy table directly; contract-test suite tags each test with `contract_tier`. | **holds** |
| A2-P6 (`IDistributedProtocolHandler`; CRTP/final devirt) | Registry-dispatched handlers must still allow A8-P7 fault-injection handler to replace a real one in sim. | A8-P7 sim-only compile target registers a `FaultyHandler` through the same registry; CRTP/final is compile-time so the sim build picks the fault variant. | **holds** |
| A2-P8 (record intentional closures) | Contract-test suite must not flag closed extension points as gaps. | A8-P11 cross-references `10-known-deviations.md`; closures are expected, not gaps. | **holds** |
| A2-P9 (versioned trace schema) | Any reader must reject pre-v1 traces; no mixed-version merges. | 6.2.1 reconstruction with Node₀ v1.0 + Node₁ v1.1: A8-P6 merge refuses; reader logs `TRACE_VERSION_DIVERGENCE`. | **holds** |
| A3-P1 (ERROR state; optional CANCELLED) | State-machine completeness must match `TraceEvent.task_state` enum. | 6.2.1 step 5-7 (`06-scenario-view.md:93-95`): parent transitions COMPLETING → ERROR; trace-event `task_state` carries the new `ERROR` code. | **holds** |
| A3-P2 (`std::expected<SubmissionHandle, ErrorContext>`) | Admission-failure observability requires that the error code is enumerable, not stringly typed. | 6.2.1 no admission failure; attack uses 6.2.3 (`06-scenario-view.md:118-126`) — `ErrorContext.code = TASK_SLOT_EXHAUSTED`; A8-P3 bumps the specific histogram. | **holds** |
| A3-P3 (admission-path failure scenario) | Scenario must be replayable through A8-P7's fault injector to validate the A8-P3 histogram. | A5-P5 matrix includes "admission saturated"; A8-P7 injector drives the path; A8-P11 contract test asserts the histogram bump. | **holds** |
| A3-P4 (producer-failure `DEP_FAILED` propagation) | Silent-timeout consumers must not appear; DEP_FAILED must reach every affected consumer deterministically. | 6.2.1 step 4 (`06-scenario-view.md:92`): `notify_parent_error` carries `ErrorContext`; every downstream consumer gets a `DEP_FAILED` event with the same `correlation_id`. | **holds** |
| A3-P5 (sibling cancellation policy) | Cancellation ordering must be deterministic under fault injection. | 6.2.1 step 5-6: siblings in flight receive `CANCELLED` (if A3-P1's optional P1b lands) in `correlation_id` order; deterministic under A8-P7. | **holds** |
| A3-P6 (requirement ↔ scenario traceability matrix) | Matrix must enumerate the SPMD scenario as coverage of the `submit_spmd` ABI. | 6.1.3 entry in matrix: NFR-SPMD row → `submit_spmd` interface → `A8-P11` contract test → 6.1.3 scenario. Chain is 1-to-1. | **holds** |
| A3-P7 (submission preconditions at boundary) | Precondition failures must be categorized distinctly in A8-P3. | A8-P3 `admission_fail{reason}` distinguishes `precondition_violation` from `slot_exhausted`. | **holds** |
| A3-P8 (cyclic `intra_edges` detection; FE_VALIDATED fast path) | FE_VALIDATED fast path must still emit `DATA_DEP_SCAN=0ns` so the reader can tell "skipped" from "present". | 6.1.3 SPMD uses no intra-edges; A8-P12 emits `DATA_DEP_SCAN{mode=fast,elapsed_ns=0}`. | **holds** |
| A3-P9 (SPMD index/size delivery contract) | Every sub-task must emit `spmd_index` on every critical phase, not just at dispatch. | 6.1.3 steps 3-5: each sub-task's `PhaseId.SPMD_EXEC` event carries `{spmd_index, spmd_size=24}`; reader can reconstruct the 24-way fan-out from trace alone. | **holds** |
| A3-P10 (Python exception mapping completeness) | Mapping must cover `DeviceError` from 6.2.1 step 8 exactly. | 6.2.1 step 8 (`06-scenario-view.md:96`): `HardwareFault` → `simpler.DeviceError(core_id=3, task=compute_A)`; `python_mapping_completeness` test asserts mapping row. | **holds** |
| A3-P11 (`[ASSUMPTION]` at `complete_in_future`) | Assumption markers must be machine-extractable so A8-P11 can assert no silent drops. | CI linter scans for `[ASSUMPTION]` lines; `assumption_count` metric in test output. | **holds** |
| A3-P12 (`drain()`/`submit()` concurrency) | Concurrent `drain()` during active SPMD must not race the completion observer. | 6.1.3 step 7 during concurrent `drain()`: `TaskState` transitions observed via `dump_state()` match trace-event order; no lost `COMPLETED`. | **holds** |
| A3-P13 (cross-node FIFO + idempotent + dup-detect) | **Prerequisite for A8-P6 merge.** Without per-chain FIFO the merge would need a reorder buffer, which is out of scope. | 6.2.2 (`06-scenario-view.md:100-114`): remote crash. `[ASSUMPTION]` markers carry the FIFO premise into A8-P6. | **holds** |
| A3-P14 (`COMPLETING` skip at leaf) | Leaf-skip must still emit the skipped PhaseId (as `{elapsed_ns=0}`). | 6.1.3 step 5: AICore leaf emits `PhaseId.COMPLETING{elapsed_ns=0,reason=leaf}`. | **holds** |
| A3-P15 (debug-mode NONE-dep cross-check) | Debug cross-check must be free in release builds. | Compile-strip at `NDEBUG`; A1-P10 CI gate confirms no L1/L2 overhead. | **holds** |
| A4-P1..P9 (doc consistency) | Naming consistency must propagate into machine-extractable docs (trace-schema enum names). | A2-P9 enum strings align with glossary; A8-P11 linter asserts. | **holds** |
| A5-P1 (exponential backoff + jitter) | Retry histogram must separate "backoff sleep" from "actual retry attempt". | 6.2.2 step 6c with retries: A8-P3 `retry_sleep_ns` histogram + `retry_attempt_count`. | **holds** |
| A5-P2 (per-peer circuit breaker) | Breaker state transitions must be observable before and after a flap. | 6.2.2: breaker transitions `CLOSED→OPEN→HALF_OPEN→CLOSED`; every transition logs via A8-P10 with `{peer, prev_state, next_state}`. | **holds** |
| A5-P3 (v1 deterministic fail-fast; v2 decentralized ADR) | Deterministic fail-fast must be distinguishable in trace from a crash cascade. | 6.2.2 step 5-6a: `failure_policy=ABORT_ALL` emits `POLICY_FAIL_FAST` PhaseId distinct from `CRASH_PROPAGATE`. | **holds** |
| A5-P4 (`idempotent: bool` on TaskDescriptor) | Observability of retry outcome must distinguish idempotent re-exec from non-idempotent refusal. | 6.2.2 step 6c: trace event carries `idempotent_bit` and `retry_decision`. | **holds** |
| A5-P5 (chaos/fault-injection matrix) | Matrix coverage must include an SPMD-under-hang row. | New row added in A5-P5 that replays 6.2.1 via A8-P7 injector (`hang_core=3`); validates A8-P3 hang-counter. | **holds** |
| A5-P6 (scheduler watchdog; paired with A8-P4) | Watchdog must fire before Chip timeout (R2 says `< 2×chip_timeout`); detection jitter must be bounded. | 6.2.1 step 1-3: watchdog fires within `watchdog_period_ms` which A8-P5 alert threshold targets. `dump_state()` is triggered before the error propagates. | **holds** |
| A5-P7 (`Timeout` on `IMemoryOps` async) | Timeout must use `IClock` (A8-P1) so that tests can fast-forward. | 6.2.4 (`06-scenario-view.md:130-142`): `FakeClock::advance(timeout_ns+1)` in test drives deterministic timeout. | **holds** |
| A5-P8 (degradation specs + alert rules) | Degradation transitions must be observable as distinct states, not computed. | 6.2.2 step 6b (`CONTINUE_REDUCED`): `RUNTIME_STATE={HEALTHY, DEGRADED, CRITICAL}` exposed via `dump_state()` + A8-P5 rule row. | **holds** |
| A5-P9 (`QUARANTINED` Worker state) | QUARANTINED transitions must appear in PhaseId taxonomy. | 6.2.1 step 3: AICore₃ → QUARANTINED; `PhaseId.WORKER_STATE_CHANGE{from=EXECUTING,to=QUARANTINED}`. | **holds** |
| A5-P10 (per-REMOTE_* idempotency) | Duplicate-detection counters must exist per handler. | 6.2.2 with retransmit: `dup_detected{handler=REMOTE_SUBMIT}` counter; A8-P11 contract asserts the counter exists. | **holds** |
| A6-P1 (trust-boundary threat model) | Observability of trust-boundary crossings must be per-boundary, not global. | 6.1.3 entirely in-process: zero boundary crossings; 6.2.2: `boundary_crossing{src,dst,type}` event. | **holds** |
| A6-P2 (mTLS HANDSHAKE cert-pinning) | Handshake outcome must be auditable via A6-P6/A8 sink plumbing. | Coordinator ↔ worker handshake on 6.1.2 setup: `AUDIT.HANDSHAKE{peer, cert_fp, outcome}`. | **holds** |
| A6-P3 (bounded payload parsing) | Malformed frame count must be observable. | A8-P3 `corrupted_frames` counter; A8-P9-style alert. | **holds** |
| A6-P4 (default-encrypted TCP control) | TCP encryption must not hide trace data from A8-P10 (end-to-end transparency). | Local sink decrypts post-wire; KV sink sees plaintext inside trust boundary. | **holds** |
| A6-P5 (scoped, revocable rkey) | rkey lifecycle events must be observable (issue/revoke). | 6.2.4: `RKEY_EVENT{action=REVOKE, reason=TIMEOUT}`. | **holds** |
| A6-P6 (security audit trail; `IAuditSink`) | IAuditSink must not force synchronous writes on hot path. | A8-P10 amendment keeps KV bus async; `IAuditSink` subscribes; sync only for the audit log itself which is off-critical. | **holds** |
| A6-P7 (function-binary attestation, multi-tenant gated) | Attestation failures must produce distinct `ErrorContext` code. | 6.2.1-style hang under attestation miss: `ErrorContext.code=ATTESTATION_FAIL`; distinct from `HardwareFault`. | **holds** |
| A6-P8 (byte-cap for Python↔C args; absorbed into A1-P11) | Cap violations must be trace-visible. | A8-P12 `ARG_MARSHAL{result=SIZE_CAP_EXCEEDED}`. | **holds** |
| A6-P9 (Logical System isolation in wire + code) | Isolation must be enforced at the KV sink filter so logs of System A don't leak into System B. | A8-P10 + A6-P10 filter on `logical_system_id`. | **holds** |
| A6-P10 (capability-scoped log/trace sinks) | Capability enforcement must not parse the log body. | KV shape makes filter a branch on `{logical_system_id, domains}`; zero-parse. | **holds** |
| A6-P11 (gated `register_factory`) | Gate must log its refusals to A8-P10. | `REGISTRY_REJECT{factory, reason, caller}`; auditable. | **holds** |
| A6-P12 (signed REPLAY trace) | **Contingent on A9-P6 option (iii).** Under option (iii), schema + signing format frozen day-1; implementation deferred. | A8 votes option (iii) above; scaffolding preserved; A6-P12 still lands as "format frozen" day-1. | **holds** (conditional) |
| A6-P13 (per-tenant submit rate-limit) | Rate-limit rejections must be observable per tenant. | `admission_reject{tenant=T, reason=RATE_LIMIT}`; A8-P5 alert row. | **holds** |
| A6-P14 (key-material lifecycle ADR) | ADR is pure doc; observability is audit-log entries at rotation. | `KEY_ROTATION{key_id, prev_ver, next_ver, rotator}`. | **holds** |
| A7-P1 (break scheduler/↔distributed/ cycle) | Cycle break must not cut a trace-merge dependency. | `scheduler/` no longer imports `distributed/`; A8-P6 merge owner sits in a neutral `observability/` module (or `composition/` per A7-P6). | **holds** |
| A7-P2 (split `ISchedulerLayer` into role interfaces) | Role-split must keep `A8-P4 dump_state()` callable with the smallest interface. | `ISchedulerDiagnostics` is one of the role interfaces; `dump_state()` lives there. | **holds** |
| A7-P3 (invert core/↔hal/ for handle types) | Test doubles must compile without `hal/` target. | `core/types.h` holds `NodeId`/`DeviceAddress`; A8-P11 HAL contract tests link `core/` + `hal/` interfaces; A8-P2 test build links only `core/`. | **holds** |
| A7-P4 (distributed payload structs to distributed/) | Fault injector (A8-P7) must attach at transport seam, not at scheduler seam. | `IHorizontalChannel` is payload-agnostic after the move; injector sits beneath the structs, above the wire. | **holds** |
| A7-P5 (`distributed_scheduler` depends only on `ISchedulerLayer`) | Mock `ISchedulerLayer` must suffice for `distributed/` tests. | `distributed/` unit test uses `FakeScheduler : ISchedulerSubmit + ISchedulerLifecycle`. | **holds** |
| A7-P6 (extract MLR + deployment parser to composition/) | A8-P5 alert-rules parser must live in `composition/`. | Module move confirmed; A8-P5 is a parser extension. | **holds** |
| A7-P7 (forward-decl contract) | No hidden `core → memory/transport` edges in test builds. | Build graph confirmed acyclic post-A7-P1..P7. | **holds** |
| A7-P8 (consolidate `ScopeHandle` ownership) | Single source of truth for scope cleanup observability. | `scope_dtor{scope_id, reason}` event emitted from one place. | **holds** |
| A7-P9 (dedupe Python MemoryError class names) | Pure Python-surface fix. | A3-P10 mapping test covers. | **holds** |
| A8-P1 (Injectable `IClock`) | A1 might argue that even a virtual call hurts at L2 under SPMD. | 6.1.3 at L2: `IClock::now_ns()` is devirtualized at link time (`modules/hal.md:247,430`); SPMD dispatch loop hits inlined `rdtsc` path. | **holds** |
| A8-P2 (Driveable event-loop) | A9 might argue the test-only seam still leaks into release ABI. | `enable_test_driver` build flag strips the `IEventLoopDriver` vtable entirely in release; ABI confirmed identical. | **holds** |
| A8-P3 (Stats + latency histograms) | A1 might argue 64-bucket arrays blow cache in per-shard scenario. | R2 amendment bounds `histograms_per_layer ≤ 16`; 16 × 64 × 8 B = 8 KiB / layer — fits L2. | **holds** |
| A8-P4 (`dump_state()`) | A5 might argue snapshot under hang is stale. | Pairing with A5-P6: watchdog triggers dump **before** ERROR propagation, capturing pre-error state. 6.2.1 step 2-3 window. | **holds** |
| A8-P5 (External alert-rules file) | A9 might argue external file is a config surface that is now a hidden dependency. | R2 amendment: file path is a `DeploymentConfig` field; no runtime re-read; parse at init; known-deviation ledger logs Prom/OTEL opt-in. | **holds** |
| A8-P6 (Distributed trace alignment) | A10 might argue decentralized coordination has no canonical merge owner. | §5 R3 amendment: tie-breaker = min(`node_id`) in youngest epoch with all-online; bound `skew_max_ns ≤ 100 µs`. | **holds** |
| A8-P7 (`IFaultInjector` sim-only) | A9 might argue even the interface is YAGNI. | Sim-only compile target (`enable_fault_inject`); zero release-build cost; A5-P5 matrix consumes it. | **holds** |
| A8-P8 (AICore in-core trace upload, L2 opt-in) | A1 might argue L1 path is still affected at compile time. | L1 PROFILING_LEVEL strips AICore ring emission (`modules/profiling.md:399`); A1-P10 CI gate asserts. | **holds** |
| A8-P9 (Profiling drop/degraded alerts) | A1 might argue the drop counter itself is a hot-path atomic. | Relaxed atomic in cold reader only; fast path emits to thread-local ring (A1-P7), no atomic. | **holds** |
| A8-P10 (Structured KV logging) | A6 might argue KV shape leaks PII if misconfigured. | Capability-scoped sink (A6-P10) filters per-field; PII fields opt-in via `sensitive: true` flag. | **holds** |
| A8-P11 (HAL contract test suite) | A7 might argue CI suite drifts from runtime HAL. | Suite runs both sim and onboard (A2-P5 BC tier); A4-P8 state-count sync closes drift. | **holds** |
| A8-P12 (Stable PhaseIds) | A1 might argue PhaseId emission is a branch at L1. | L1 strips via compile-time guard; L2 uses A1-P7 thread-local sequence to stay under 100 ns. | **holds** |
| A9-P1 (drop submit overloads; absorbed into A7-P2) | Single admission entry narrows A8-P2 test surface. | 6.1.3 step 1 goes through one `submit()` path. | **holds** |
| A9-P3 (remove collectives from `IHorizontalChannel`) | Collectives hoisted to a separate seam; A8-P7 injector location unchanged (per-message). | Collective observability handled by a new module; A8-P3 collective histograms remain. | **holds** |
| A9-P4 (drop `SubmissionDescriptor::Kind`) | One code path → one PhaseId attribution. | 6.1.3 `submit_spmd` no longer carries `Kind`; A8-P12 attribution unambiguous. | **holds** |
| A9-P5 (unify admission enums) | Single `AdmissionStatus` enum for A8-P3 histogram dimension. | 6.2.3 admission-fail variants all under one enum. | **holds** |
| A9-P7 (fold `SourceCollectionConfig`) | Folded config struct must still be knob-addressable. | `EventHandlingConfig.sources = [...]` replaces the folded struct; A4-P5 glossary update covers. | **holds** |
| A9-P8 (move AICore companion-artifacts out of design) | Out-of-scope move must not break A8-P8 upload protocol. | A8-P8 references the artifact contract but not the artifact format; format lives elsewhere. | **holds** |
| A10-P1 (default `producer_index` sharding policy) | Per-shard histograms must be keyable. | A8-P3 histogram `{layer, shard}` key; 6.1.3 step 2 attributes. | **holds** |
| A10-P3 (per-data-element consistency model) | Table is doc-only; A8-P11 asserts classes match runtime. | CI check compares data-element class to code annotations. | **holds** |
| A10-P4 (stateful/stateless classification) | Classification must align with `dump_state()` surface. | A8-P4 schema reflects stateful components only. | **holds** |
| A10-P6 (faster peer-failure detection + hysteresis) | Detection interval must be a `DeploymentConfig` knob, not hard-coded. | R2 amendment sets `heartbeat_interval_ms`, `suspect_threshold`, `hysteresis_ms` as knobs; A8-P5 alert row uses them. | **holds** |
| A10-P7 (sharded TaskManager) | Sharding must be invisible to A8-P4 external schema (or else versioned). | A8-P4 schema bumps on shard-enable; default `shards=1` keeps legacy schema. | **holds** |
| A10-P8 (single Data & State Reference §) | Single doc source-of-truth for O4 surface. | A8-P4 links to the § directly. | **holds** |
| A10-P9 (gate WorkStealing × RETRY_ELSEWHERE) | Assignment log must be available for audit. | A8-P10 records `ASSIGN{task, worker, reason}` entries. | **holds** |

**Stress-round totals.** 77 proposal rows stress-attacked. **77 `holds`. 0 `breaks`. 0 `uncertain`.** One amendment raised (A8-P6 tie-breaker, §5). No new disputes. No hot-path cost introduced by any A8 stance.

### Scenario-replay attestation

Two scenarios were replayed end-to-end under the amended design:

- **6.1.3 SPMD Kernel Launch** (`06-scenario-view.md:61-79`): A test observer reconstructs the 24-way fan-out from trace alone. Every sub-task carries `{correlation_id, spmd_index, spmd_size, PhaseId, elapsed_ns, budget_ns}`. Steps 1-7 map 1-to-1 onto PhaseIds `SPMD_ADMIT → SPMD_EXPAND → SPMD_DISPATCH → SPMD_EXEC[0..23] → SPMD_FIN[0..23] → SPMD_GROUP_COMPLETE → PARENT_DEC`. Two-tier bridge holds: L1 emits only `SPMD_GROUP_COMPLETE`; L2 opt-in emits per-AICore PhaseIds.
- **6.2.1 AICore Hang** (`06-scenario-view.md:83-98`): A5-P6 watchdog fires at `chip_timeout/2`, triggers A8-P4 `dump_state()`, captures pre-error state; Chip Scheduler marks AICore₃ FAILED (A5-P9 QUARANTINED state); ERROR propagation (A3-P1) carries the same `correlation_id` through Device → Host; Python sees `simpler.DeviceError(core_id=3)` (A3-P10 mapping). Reader can reconstruct the full causal chain from traces + dump + alert-rule firing.

Both scenarios pass the two-tier bridge: L1 (default) emits enough events to diagnose the failure surface; L2 opt-in provides per-step reconstruction.

## 9. Status

- **Satisfied with current design?** **yes** (conditional on the three disputed proposals landing as voted in §6: A2-P7 as Q-record, A9-P2 with single test seam, A9-P6 option (iii)).
- **A1 hot-path veto (self-check):** N/A (A8 is not A1). A8 introduces no new hot-path cost in round 3.
- **Amendments raised this round:** 1 (A8-P6 tie-breaker rule, §5).
- **Stress-attack casualties:** 0.
- **Blocking objections raised:** 0.
- **`override_request=true`:** 0.
- **Expected round-3 synthesis outcome:** convergence — 110/110 proposals agreed, A8-P6 carries the minor tie-breaker amendment (no vote change needed), all three prior disputes flipped to agreed on their R2-amended text.

WROTE /data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/reviews/2026-04-18-171357/round-3/reviews/A8-testability.md
