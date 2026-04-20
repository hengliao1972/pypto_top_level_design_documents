# Aspect A8: Testability & Observability — Round 1

## Metadata

- **Reviewer:** A8
- **Round:** 1
- **Target:** `/data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/` (full doc set)
- **Runtime Mode:** yes
- **Timestamp (UTC):** 2026-04-19T00:00:00Z

---

## 1. Rubric Checks

| # | Check | Result | Evidence (file:line) | Rule(s) |
|---|-------|--------|----------------------|---------|
| 1 | Every external dependency behind an interface replaceable with a test double (HW / net / clock / OS) | **Weak** | `modules/hal.md:46-156` (IDevice / IMemory / IExecutionEngine / IRegisterBank / INetwork); `modules/hal.md:111-119` (SIM leaf factories); **clock is not an injectable interface**: `modules/profiling.md:104-106` (`CLOCK_MONOTONIC_RAW` accessed through `hal` but no `IClock`), `modules/profiling.md:16` ("Depends on: `hal/` (clock access)" — informal); no watchdog / timeout timer is behind a fake-timer seam (`modules/scheduler.md:485` `worker_timeout_ms`, `modules/runtime.md:146-151` `Timeouts`, `modules/hal.md:397` `engine_timeout_ms`) | X5, §8.2 DfT |
| 2 | All critical operations carry trace IDs, timestamps, component identifiers | **Pass** | `modules/profiling.md:82-99` (`TraceEvent` carries `timestamp_ns`, `sequence`, `correlation_id`, `task_key_packed`, `thread_id`, `layer_id`); `modules/error.md:131-137` (`ErrorContext` has `correlation_id`, `task_key`, `layer_id`); `modules/transport.md:94, 316` (`correlation_id` threaded end-to-end); `modules/distributed.md:88-90, 317` | O1 |
| 3 | Logs are structured & context-rich | **Weak** | `07-cross-cutting-concerns.md:109-117` (format defined, KV mentioned but "may evolve"); `modules/profiling.md:118-126` (`Logger::log(Severity, category, message, correlation_id)` — **message is a flat `string_view`, no structured key/value API**; `modules/profiling.md:463` calls this out as an open question) | O2 |
| 4 | Health metrics defined — error rate, latency percentiles, queue depths, resource utilization | **Weak** | `07-cross-cutting-concerns.md:137-146` (alert table **cites** P95/P99 Host→AICore dispatch, REMOTE_SUBMIT, pool utilization, error rate) but no histogram / percentile data structure is defined anywhere; `modules/core.md:76-77` (`LayerStats get_stats()` declared opaque, fields not enumerated); `modules/scheduler.md:168` (`LayerStats`, `TaskManagerStats` only documented as "read-only counters"); `modules/memory.md:61` (`MemoryStats get_stats()` declared, fields absent); `modules/runtime.md:206` (`RuntimeStats` = aggregate of under-specified structs). Alerts therefore reference counters that the modules do not yet commit to exposing. | O3, O5 |
| 5 | Can the system be debugged in production without redeployment? | **Weak** | `modules/profiling.md:359-361` (runtime `set_level` via atomic; good); `modules/profiling.md:125-126` (`set_min_severity` dynamic; good); **but** no state-inspection / dump API exists. `modules/runtime.md:30-42` exposes only `stats()`; no `dump_state()` / `describe()` entry point; no per-layer queue dump, no outstanding-submission dump, no worker-state matrix dump, no `producer_index` snapshot for diagnosing cross-Submission dep bugs; `modules/bindings.md:30-54` mirrors the same narrow surface. | O4, X6, §8.3 DfD "State inspection at runtime" |
| 6 | Alerts fire on SLO breach, not arbitrary thresholds | **Weak** | `07-cross-cutting-concerns.md:133-135` (claim: "Alerts fire when SLOs are at risk"); rows at `07-cross-cutting-concerns.md:139-146` do key off the SLO numbers in `07-cross-cutting-concerns.md:188-195` and budgets in `04-process-view.md:641-700`, **but** the thresholds are hard-coded in prose and not externalized in config (no `AlertConfig` / `AlertRule` schema in `modules/runtime.md:96-152` `DeploymentConfig`), conflicting with E3/X8. Moreover row 7 ("Profiling overhead exceeded … Level 1 overhead > 1%") has no counter specified that would let an operator evaluate it in production. | O5, E3, X8 |

Additional Testability & Observability spot checks:

| # | Spot check | Result | Evidence |
|---|-----------|--------|----------|
| 7 | Deterministic replay of scheduler (step-by-step) for unit tests | **Weak** | `modules/scheduler.md:490-514` enumerates unit tests but the event loop is described only as a continuous run (`04-process-view.md:47-107`); no `step()` / `tick(n)` entry point on Stage A/B/C is exposed for drive-by-a-test-harness execution. `test_event_loop_stages` (`modules/scheduler.md:503-505`) compares deployments on the same trace but relies on a running loop. |
| 8 | Fault-injection seam in HAL / transport for chaos tests | **Weak** | `modules/hal.md:422-423` mentions "DMA timeout fault injection at the sim memory layer" as an *edge-case test*, but no `IFaultInjector` API or declarative fault schedule is defined anywhere; `modules/transport.md:423` similarly treats injection ad hoc. |
| 9 | Distributed trace time-alignment | **Weak** | `07-cross-cutting-concerns.md:97-102` declares "Coordinator Node merges all node traces with global timestamp alignment" **with no algorithm, no clock-sync assumption (NTP/PTP), no max-skew bound** — yet distributed integration tests at `modules/distributed.md:427-432` rely on correct cross-node causal ordering. |
| 10 | Profiling drop / overflow observability | **Weak** | `modules/profiling.md:275, 385, 411-413` — drop counters exist **but no alert in `07-cross-cutting-concerns.md:137-146` fires on non-zero drops**. Silent loss of trace events invalidates latency-budget validation (`04-process-view.md:696-702`). |

---

## 2. Pros

- **HAL interfaces are a clean test-double seam (X5).** `modules/hal.md:24-156` abstracts `IDevice` / `IMemory` / `IExecutionEngine` / `IRegisterBank` / `INetwork` / `IPlatformFactory`; the `SIM` variant (`modules/hal.md:111-119`) cleanly swaps the leaf `IExecutionEngine` per `SimulationMode {PERFORMANCE, FUNCTIONAL, REPLAY}`, enabling full-stack testing without hardware — textbook DfT (§8.2) and X5 compliance.
- **REPLAY simulation mode delivers reproducible failures (§8.2 DfT).** `modules/hal.md:117, 346-349` + `07-cross-cutting-concerns.md:131` establish that binary Level-2 traces captured live are valid input for `REPLAY` mode, meeting the "reproducible failures" tactic.
- **Correlation-id plumbing is end-to-end (O1).** Shared field exists in `TraceEvent` (`modules/profiling.md:82-99`), `ErrorContext` (`modules/error.md:137`), `VerticalMessage`/`HorizontalMessage` (`modules/transport.md:94, 316`), and distributed messages (`modules/distributed.md:88, 317`); a single span id threads from Python call to remote completion.
- **Zero-overhead profiling structure is a credible basis for A1-compatible observability (X5 + A1).** `07-cross-cutting-concerns.md:119-123` and `modules/profiling.md:248-253, 438-446` use compile-time strip (`PROFILING_LEVEL`), TLS SPSC ring buffers, monotonic sequence allocator, cache-line alignment, and release-build inlining — the design anticipates A1's hot-path-cost objection and pre-empts it.
- **Runtime log-level / profiling-level adjustment is documented (O4).** `modules/profiling.md:359-361, 125-126` expose atomic `set_level` / `set_min_severity`; `modules/bindings.md:53, 319-321` re-exports them to Python (`simpler.logging.set_level`). No redeploy required.
- **SLO targets exist and alerts nominally tie to them (O5 partial).** `07-cross-cutting-concerns.md:188-195` (five SLO targets) and `04-process-view.md:641-700` (per-stage budgets) provide the quantitative foundation alerts can reference, rather than arbitrary thresholds.
- **Structured error chain preserves remote context (§8.3 DfD, O1).** `modules/error.md:131-137` + `modules/bindings.md:81-93` preserve `remote_error_chain`, `cause`, `severity`, `task_key` end-to-end into the Python exception hierarchy.
- **Testing strategy sections are present in every module doc.** `modules/{hal,core,scheduler,memory,transport,distributed,profiling,error,runtime,bindings}.md §9` each enumerate Unit / Integration / Edge / Failure tests (e.g. `modules/profiling.md:411-434`, `modules/scheduler.md:490-527`). This is the scaffolding testability design (DfT) requires; it needs tightening but not invention.

---

## 3. Cons

- **No injectable `IClock` (X5 / §8.2).** Clock is referenced indirectly ("hal clock", `modules/profiling.md:16, 104-106`) but not exposed as a named interface. Scheduler watchdogs (`modules/scheduler.md:485` `worker_timeout_ms`), engine wait (`modules/hal.md:100` `wait_complete(Timeout)`), heartbeat (`modules/runtime.md:149-151`), profiling timestamps (`modules/profiling.md:311-332`) all read wall clocks directly. Unit tests cannot deterministically advance time, which defeats "Deterministic by default" (`01-design-principles.md:389`).
- **Event loop not driveable by a test harness (§8.2, X5).** Stage A/B/C (`modules/scheduler.md:220-257`, `04-process-view.md:47-107`) has no `step()` / `tick()` API. Consequence: `test_event_loop_stages` (`modules/scheduler.md:503-505`) must race the loop to verify "same trace processed identically" across deployment modes — brittle and non-reproducible.
- **`LayerStats` / `RuntimeStats` / `TaskManagerStats` / `MemoryStats` / `DistributedStats` are declared but never enumerated (O3).** Alerts at `07-cross-cutting-concerns.md:137-146` reference "pool utilization", "error rate", "P95 dispatch latency", yet `modules/core.md:77`, `modules/scheduler.md:40, 168`, `modules/memory.md:61`, `modules/runtime.md:206`, `modules/distributed.md:162` only say "read-only counters". No latency-histogram primitive is defined; percentile alerts cannot be implemented against the current contract.
- **No state-inspection / dump endpoint (O4, X6, §8.3).** `modules/runtime.md:30-42` only exposes `stats()`. There is no way to obtain, at runtime, the outstanding-submission window contents, admission-queue depth per layer, ready-queue contents, worker state matrix, `producer_index` size, remote-proxy occupancy, or scope depths — the debuggability tactics in `01-design-principles.md:404` ("State inspection at runtime") are not satisfied.
- **Alert thresholds not externalized (O5, E3, X8).** `07-cross-cutting-concerns.md:137-146` hard-codes numeric thresholds in prose; `modules/runtime.md:96-152` `DeploymentConfig` has no `AlertConfig` / `AlertRule` field; `modules/bindings.md` has no `on_alert` or similar subscription API. Operators cannot tune alerting without a rebuild.
- **Structured KV logging is "open question" (O2).** `modules/profiling.md:463` explicitly defers structured (key/value) logging; the current facade passes a flat `std::string_view message`. Machine-parseable logs required by O2 therefore depend on call-site discipline.
- **Cross-node trace alignment is unspecified (O1 distributed).** `07-cross-cutting-concerns.md:97-102` asserts alignment without algorithm, clock-sync assumption, or bounded-skew guarantee. In practice merged distributed traces can reorder spans silently and break causal-ordering assertions in `modules/distributed.md:427-432`.
- **Profiling drop counters do not feed alerts (O3, O5).** `modules/profiling.md:275, 385, 413` document drop counters but no alert rule at `07-cross-cutting-concerns.md:137-146` fires on non-zero drops; silent trace loss invalidates budget validation at `04-process-view.md:696-702`.
- **No explicit fault-injection API for chaos/failure tests (DfT, R6-adjacent).** Sim does allow ad-hoc injection (`modules/hal.md:422-423`, `modules/transport.md:423`, `modules/error.md:385`), but there is no uniform `IFaultInjector` surface — each module reinvents its own test hook, and Python tests cannot drive failure scenarios cross-cutting the stack.
- **AICore in-core trace path is open (O3 on leaf budget).** `modules/profiling.md:461` and `07-cross-cutting-concerns.md:91-103` show AICore feeding Host collector, but the upload protocol is "under HAL capability review". Without a committed path, `04-process-view.md:652` ("AICore: detect dispatch → begin execution < 1 μs") cannot be validated in production, and §4.8.5 budget validation (`04-process-view.md:696-702`) degrades at Level ≥ 2.
- **`Runtime.stats()` surface is too coarse for Python observability (O3).** `modules/bindings.md:36, 102` returns "dict snapshot"; there is no streaming metrics export (e.g., OpenMetrics/Prometheus `/metrics`), no per-layer subscription — so external monitoring must scrape by wrapping the Python API, which is Python-threaded and GIL-bound.
- **Drop/overflow silent failure on sinks (O4).** `modules/profiling.md:386` says sinks "enter degraded state" and rate-limit warnings to `stderr`, but no metric is emitted via `LayerStats` / `RuntimeStats` for operators to alert on — see row 10 of §1.
- **No contract tests between onboard and sim HAL (X5 / DfT).** Sim and onboard HAL implementations share interfaces (`modules/hal.md:245-247`) but the doc does not require a single "contract test" suite runnable against both — the two variants can diverge undetected.

---

## 4. Proposals

| id | severity | summary | affected_docs | hot_path_impact | trade_offs | sanity_test |
|----|----------|---------|---------------|-----------------|------------|-------------|
| A8-P1 | high | Introduce `hal::IClock` as the single injectable time source (real, monotonic-raw, fake, deterministic) used by profiling / schedulers / transport timeouts / heartbeat | `modules/hal.md`, `modules/profiling.md`, `modules/scheduler.md`, `modules/transport.md`, `modules/runtime.md`, `07-cross-cutting-concerns.md` | none (release-build inlines to today's `CLOCK_MONOTONIC_RAW` via link-time HAL specialization, §10 of `modules/hal.md`) | gain: deterministic unit tests, time-warp for watchdog tests, skew injection for distributed tests. give up: one extra interface in the HAL surface | Add a test that advances a `FakeClock` by 10 s inside a scheduler unit test; verify `worker_timeout_ms`-based failure fires without wall-clock sleep |
| A8-P2 | high | Expose a `tick()` / `step(n_events)` entry on the scheduler event-loop and a `RecordedEventSource` in `core/` for deterministic scheduler tests | `modules/scheduler.md`, `modules/core.md`, `02-logical-view/02-scheduler.md`, `02-logical-view/09-interfaces.md` | none (test-only target gated on `enable_test_driver` compile flag; release binary unaffected) | gain: reproducible event-ordering tests, A3-friendly scenario replays, A5 fault-injection stacking. give up: small extra build target; minor surface area growth | Run the same recorded event trace through Single-Threaded / Split-A / Split-C / Fully-Split loop deployments and assert bit-identical state transitions |
| A8-P3 | high | Enumerate every stats struct (`LayerStats`, `TaskManagerStats`, `WorkerStats`, `MemoryStats`, `ChannelStats`, `DistributedStats`, `RuntimeStats`) with concrete fields, units, and lifecycle; add a lock-free bucketed latency-histogram primitive for the three SLO budgets (submit→DISPATCHED, REMOTE_SUBMIT, AICore FIN→Python) | `modules/core.md`, `modules/scheduler.md`, `modules/memory.md`, `modules/transport.md`, `modules/distributed.md`, `modules/runtime.md`, `07-cross-cutting-concerns.md` | allocates (at init only — histograms are fixed bucket arrays), hot-path inserts are branchless single-bucket increments < 5 ns | gain: alerts at `07-cross-cutting-concerns.md:139-146` become implementable; O3 fully satisfied. give up: ~1 KB per histogram; one extra write on completion paths | Query a histogram after a 100-task burst; assert P95 / P99 within SLO and that monotonic counter totals equal submitted-task count |
| A8-P4 | high | Add `Runtime::dump_state()` (and `ISchedulerLayer::describe()`, `IMemoryManager::describe_state()`, `DistributedRuntime::describe_cluster_state()`) returning a structured JSON snapshot: window occupancy, admission-queue depth, ready-queue sizes, worker state matrix, `producer_index` size, remote-proxy occupancy, scope stack depths. Expose via `simpler.Runtime.dump_state() -> dict` | `modules/runtime.md`, `modules/scheduler.md`, `modules/memory.md`, `modules/distributed.md`, `modules/bindings.md`, `07-cross-cutting-concerns.md` | none (cold diagnostic endpoint: seqlock / relaxed-atomic snapshot reads only; never runs on hot path) | gain: O4 / X6 / §8.3 DfD "state inspection" satisfied; incident response time reduced. give up: must carefully lay out stats as SoA to permit safe snapshotting without stopping the loop | Trigger an artificial SLOT_POOL_EXHAUSTED in sim; assert `dump_state()` shows the exhausted pool, pending submissions, and the blocking worker before the test releases the pool |
| A8-P5 | medium | Externalize alert rules into `DeploymentConfig::alerting`; add `AlertRule { metric, op, threshold, window_ms, severity }`; expose `Runtime::on_alert(callback)` and a built-in OpenMetrics/Prometheus sink for `RuntimeStats` | `modules/runtime.md`, `modules/bindings.md`, `modules/profiling.md`, `07-cross-cutting-concerns.md` | none (rule evaluation runs on a low-priority collector thread, not on any scheduler or dispatch path) | gain: O5, E3, X8 satisfied simultaneously; ops can tune thresholds per deployment without rebuild. give up: small config-schema expansion; one more sink option | Add an alert rule with threshold 0 μs; submit one kernel; verify the alert fires through both the callback and the /metrics endpoint |
| A8-P6 | medium | Specify the distributed trace-alignment algorithm: declare NTP/PTP dependency, document max-allowed skew, add `IClockSync` in `hal/` (or `distributed/`) and a trace-merge algorithm with a bounded-skew window. Require `correlation_id`-based causality as tiebreaker | `07-cross-cutting-concerns.md`, `modules/profiling.md`, `modules/distributed.md`, `modules/hal.md` | none (clock sync is a background facility; merge runs offline or on the coordinator in the export path) | gain: distributed O1 soundness; merged traces retain causal order. give up: new HAL interface; operators must configure a time source | Inject a 1 ms skew between two sim nodes; run the `modules/distributed.md:427-432` integration test; verify merged trace respects `REMOTE_SUBMIT` → `REMOTE_COMPLETE` happens-before across nodes |
| A8-P7 | medium | Define a uniform `IFaultInjector` test seam (sim-only compile target) with declarative schedules for DMA-timeout, register-read-fault, RDMA-loss, heartbeat-miss, AICore-hang, slot-pool-exhausted; hook into `hal/sim`, `transport/`, and `distributed/` dedup | `modules/hal.md`, `modules/transport.md`, `modules/distributed.md`, `modules/scheduler.md`, `06-scenario-view.md` | none (sim-only symbol; never compiled into onboard builds) | gain: DfT chaos / failure-mode coverage; A5 / A3 / A8 all benefit. give up: additional test-binary surface | Load a fault schedule "drop heartbeat at T=100 ms"; run scenario `06-scenario-view.md:100-114`; assert the configured `failure_policy` path is exercised |
| A8-P8 | medium | Commit AICore in-core trace upload protocol: declare a fixed-size device-side ring plus Chip-level periodic DMA uploader as the v1 default; add a Level-2 contract test asserting N events per kernel at a fixed cadence | `modules/profiling.md`, `modules/hal.md`, `07-cross-cutting-concerns.md`, `04-process-view.md` | extends-latency at Level ≥ 2 only — mitigated by default `PROFILING_LEVEL = L1_Coarse` compile-time strip (`modules/profiling.md:399`) so Level-2 is an opt-in slow path; fast path unchanged. Two-tier: Level 0/1 emits no device event; Level 2 ring write < 50 ns amortized | gain: §4.8.5 budget validation becomes executable on onboard; closes open question at `modules/profiling.md:461`. give up: one more device-side ring to provision and DMA bandwidth budget at high profiling levels | Run the same Host→AICore scenario at Level 1 and Level 2; assert Level 1 shows identical latencies to Level 0 within noise, and Level 2 produces a contiguous AICore event stream with zero drops |
| A8-P9 | medium | Turn profiling drop counters and sink-degraded state into first-class alert rows; add `TraceEvents_dropped`, `TraceSink_degraded_ms`, `LogEvents_dropped` to `LayerStats` / `RuntimeStats`; define SLO alert "drop rate > 0 over 1 min → Warning" | `modules/profiling.md`, `07-cross-cutting-concerns.md`, `modules/runtime.md` | none (drop counter already exists; exposing it adds a relaxed-atomic read in the cold stats path) | gain: closes silent-failure hole in `modules/profiling.md:275, 386`; O3 + O5 strengthened; protects §4.8.5 budget validation. give up: negligible | Saturate a TLS ring at 10× normal rate; verify the new alert fires and the counter is non-zero in the snapshot |
| A8-P10 | medium | Adopt a structured key/value logging API (`Logger::log_kv(Severity, category, {kv...}, correlation_id)`) as the primary surface; keep string-message shim for legacy sites; require machine-parseable JSON line sink | `modules/profiling.md`, `07-cross-cutting-concerns.md` | none (branch-predicted severity gate unchanged; KV emit allocates only when above threshold) | gain: O2 satisfied; closes `modules/profiling.md:463` open question. give up: porting cost for existing call sites | Grep logs during a 10 k-task run; confirm every line is valid JSON and contains `{correlation_id, layer_id, category}` |
| A8-P11 | low | Freeze a single "HAL Contract Test" suite runnable against both `sim` and `onboard` variants of every HAL interface (all methods, all documented error codes, all concurrency contracts); CI gate on divergence | `modules/hal.md`, `03-development-view.md` | none (test-only) | gain: prevents silent drift between sim and onboard (the primary testability foundation). give up: test maintenance burden | Run the same suite against `a2a3sim` and `a2a3` (onboard); identical green required; any behavior drift surfaces as a contract-test failure |
| A8-P12 | low | Enumerate stable `PhaseId` values for Submission lifecycle (`SUBMISSION_ADMIT`, `SUBMISSION_RETIRE`, `DATA_DEP_SCAN`, `BARRIER_ATTACH`, `WORKSPACE_ALLOC`, `ADMISSION_QUEUE_DRAIN`) and for Worker lifecycle so admission-budget validation (`04-process-view.md:681-702`) is directly measurable | `modules/profiling.md`, `modules/scheduler.md`, `07-cross-cutting-concerns.md` | allocates (one extra ring slot per phase) — Level-2+ only, already stripped at L1 | gain: precise traceability of `§4.8.4` admission budget stages; independent verification of `ADR-013` producer_index soundness | Verify a trace file contains per-phase timing summing to the observed admission latency within < 1% |

### Proposal detail

#### A8-P1: Injectable `IClock` interface

- **Rationale:** Rule X5 ("Every external dependency … behind an interface that can be replaced with a test double") and `01-design-principles.md:389` ("Inject clocks, random seeds, and configuration"). Currently `modules/profiling.md:16, 104-106` treats clock as an informal HAL capability; `modules/scheduler.md:485` / `modules/runtime.md:149` / `modules/hal.md:397-400` all depend on wall time without an injection seam. Finding: row 1 + row 7 of §1.
- **Edit sketch:**
  - File: `modules/hal.md` — add §2.8 `IClock` interface (`now_ns()`, `monotonic_raw_ns()`, `steady_tick()`), list realizations `SystemClock`, `FakeClock`, `PerformanceSimClock` (already implicitly used by `PERFORMANCE` mode, `modules/hal.md:115`).
  - File: `modules/profiling.md:2.1 / §10` — change `CLOCK_MONOTONIC_RAW` reference to `hal::IClock*` injected via `ProfilingConfig`.
  - File: `modules/scheduler.md:§8` — document `worker_timeout_ms` evaluated against the injected clock.
  - File: `modules/runtime.md:DeploymentConfig` — add `clock_factory` field.
- **Trade-offs:** Gain — deterministic tests, clean watchdog coverage, required by every other testability proposal (A8-P2, A8-P6, A8-P7). Give up — one more HAL interface; care to keep it link-time specialized so release binaries are identical to today.
- **Sanity test:** In a unit test under sim, install `FakeClock`; `scheduler.dispatch(task)`; advance clock by `worker_timeout_ms + 1`; assert `WorkerCrashed` fires and no real sleep occurred.

#### A8-P2: Driveable event-loop (`step()` + `RecordedEventSource`)

- **Rationale:** X5 + §8.2 DfT ("Reproducible failures"). `modules/scheduler.md:503-505` implies "same trace identical across deployments" but the event loop (`04-process-view.md:47-107`) is only described as a continuous run. A harness cannot drive it one event at a time.
- **Edit sketch:**
  - File: `modules/scheduler.md:§2.5 / §5.2` — add `EventLoopRunner::step(max_events)` returning a `StepReport`; gated behind `enable_test_driver`.
  - File: `modules/core.md:§2` — add `RecordedEventSource` implementing `IEventSource` and a `EventTrace` serialization format.
- **Trade-offs:** Gain — reproducible, cross-deployment identical scheduler behaviour tests; enables A8-P7 and A3 scenario replays. Give up — test-only surface, small additional compile target.
- **Sanity test:** Serialize a 200-event trace (dep-satisfied + worker-completed mixed); run it through Single-Threaded and Fully-Split loop deployments; assert resulting Task/Worker state sequences are byte-identical.

#### A8-P3: Enumerate stats structs and add bucketed latency histograms

- **Rationale:** O3 + O5. Alerts at `07-cross-cutting-concerns.md:137-146` reference P95 / P99 / pool utilization / error rate but the underlying data is under-specified (row 4 of §1). Principle `01-design-principles.md:404` ("State inspection at runtime") also impacted.
- **Edit sketch:**
  - File: `modules/core.md:§2.5` — enumerate `LayerStats { submitted, in_flight, retired, failed, pool_depth, pool_peak, dispatch_histogram, … }` with units.
  - File: `modules/scheduler.md:§2.6 / §4.3` — define `LatencyHistogram` (pre-allocated N-bucket log-scale; branchless insert; seqlock snapshot reads).
  - File: `modules/memory.md:§2.5` — enumerate `MemoryStats { per_region_free, per_region_peak, workspace_live, workspace_peak, alloc_failures }`.
  - File: `modules/distributed.md:§2.6` — enumerate `DistributedStats` fields.
  - File: `07-cross-cutting-concerns.md:§7.2.8` — retarget alert rows to the enumerated counters.
- **Trade-offs:** Gain — all alert rows become implementable; external monitoring no longer needs side-channel instrumentation. Give up — ~1 KB per histogram at init (A8-P5 helps keep this configurable).
- **Sanity test:** Submit N=100 kernels under sim; `dump_state()` → histogram; P95 of `submit→DISPATCHED` must be within the `04-process-view.md:645-653` budget, count equals 100.

#### A8-P4: Runtime `dump_state()` diagnostic endpoint

- **Rationale:** O4, X6, §8.3 DfD bullet "State inspection at runtime". Current surface (`modules/runtime.md:30-42`, `modules/bindings.md:30-54`) exposes only counter aggregates; an incident responder cannot see admission-queue contents, ready queues, worker state matrix, or `producer_index` occupancy.
- **Edit sketch:**
  - File: `modules/runtime.md:§2.1` — add `Runtime::dump_state(DumpOptions) -> StateDump` returning structured JSON.
  - File: `modules/scheduler.md:§2.5 / §3` — add `ISchedulerLayer::describe()` recurring into TaskManager / WorkerManager / ResourceManager snapshots.
  - File: `modules/bindings.md:§2.1` — expose `simpler.Runtime.dump_state() -> dict`.
  - File: `07-cross-cutting-concerns.md:§7.2` — add a new subsection "Runtime state inspection".
- **Trade-offs:** Gain — fast incident diagnosis without restart; closes the largest O4 gap. Give up — must carefully use seqlock / atomic snapshot reads so a running scheduler is never stopped.
- **Sanity test:** Script: saturate a layer's task-slot pool; call `dump_state()`; assert `pool_exhausted = true`, `admission_queue_depth > 0`, the blocking submission id visible, and no measurable regression in submit latency while `dump_state()` runs.

#### A8-P5: Externalize alert rules and add Prometheus/OTEL metrics sink

- **Rationale:** O5 ("Alerts fire on SLO breach rather than arbitrary thresholds"), E3 ("Configuration over hardcoding"), X8. `07-cross-cutting-concerns.md:137-146` currently hard-codes thresholds in prose; `modules/runtime.md:96-152` has no `AlertConfig` field.
- **Edit sketch:**
  - File: `modules/runtime.md:DeploymentConfig` — add `alerting: std::vector<AlertRule>` plus `metrics_sinks`.
  - File: `modules/profiling.md:§2.5 / §11` — add `PrometheusSink` / `OpenMetricsSink` emitting `RuntimeStats`.
  - File: `modules/bindings.md:§2.1` — add `simpler.Runtime.on_alert(callable)`.
- **Trade-offs:** Gain — ops tune alerts per deployment; integrates into existing monitoring; closes O5. Give up — small schema expansion; one more sink.
- **Sanity test:** Configure an alert rule "pool utilization > 0 → Warning"; submit one task; verify alert is delivered through the Python callback and scraping `/metrics` shows the counter.

#### A8-P6: Distributed trace time-alignment contract

- **Rationale:** O1 across node boundary. `07-cross-cutting-concerns.md:97-102` declares "global timestamp alignment" without specifying an algorithm; merged distributed traces can silently reorder spans, breaking `modules/distributed.md:427-432` causal assertions.
- **Edit sketch:**
  - File: `07-cross-cutting-concerns.md:§7.2.3` — document assumed PTP/NTP sync, bound `skew_max_ns`, specify merge algorithm: primary sort by `(sequence, correlation_id, happens_before)` with skew-windowed re-order.
  - File: `modules/hal.md:§2.8` — add `IClockSync::offset_ns()` (delivered alongside A8-P1 `IClock`).
  - File: `modules/distributed.md:§3.3` — require correlation-id chain to dominate timestamp ties.
- **Trade-offs:** Gain — merged trace correctness. Give up — operators must configure a clock source.
- **Sanity test:** Inject a 1 ms skew between two sim nodes; run the cross-node submit scenario; merged trace must keep `REMOTE_SUBMIT` before `REMOTE_COMPLETE` of the same correlation_id.

#### A8-P7: Uniform `IFaultInjector` test seam (sim-only)

- **Rationale:** DfT "Simulation mode" tactic + Rule X5; A5 / R6 adjacent but owned here for cross-module test orchestration. Today fault injection is ad hoc (`modules/hal.md:422-423`, `modules/transport.md:423`, `modules/error.md:385`).
- **Edit sketch:**
  - File: `modules/hal.md:§2.x` — add `IFaultInjector` compiled only under sim with `schedule_fault(FaultSpec)`.
  - File: `modules/transport.md`, `modules/distributed.md` — subscribe-side hooks.
  - File: `06-scenario-view.md:§6.2` — retarget failure scenarios to the declarative schedule.
- **Trade-offs:** Gain — uniform chaos surface; A3 / A5 / A8 coverage improves together. Give up — a sim-only module more to maintain.
- **Sanity test:** Fault schedule "drop heartbeat from Node₁ at T=200 ms"; run `06-scenario-view.md:100-114`; verify `CONTINUE_REDUCED` policy exercised end-to-end with deterministic timing.

#### A8-P8: Commit AICore in-core trace upload protocol

- **Rationale:** O3 at leaf level; §4.8.5 budget validation impossible on hardware without this. `modules/profiling.md:461` is an open question; `07-cross-cutting-concerns.md:91-103` hand-waves "drain via DMA".
- **Edit sketch:**
  - File: `modules/profiling.md:§3.1 / §3.4` — specify fixed-size AICore event ring + Chip-level DMA uploader at configurable cadence (`sink_flush_interval_ms`).
  - File: `modules/hal.md:§2.4` — declare capability bit `AICORE_TRACE_RING_SUPPORTED`.
  - File: `04-process-view.md:§4.8.5` — add explicit Level-2 onboard-validation step.
- **Trade-offs:** Gain — closes an open question blocking onboard budget validation. Give up — extends-latency at Level ≥ 2 only; mitigated by `PROFILING_LEVEL` compile-time strip (fast path unchanged).
- **Sanity test:** Onboard integration test: run same Host→AICore flow at Level 1 and Level 2; Level-1 latency within noise of Level-0; Level-2 trace contains a contiguous per-core event stream with zero drops over a 60 s burst.

#### A8-P9: Profiling drop/degraded state as first-class alerts

- **Rationale:** O3 + O5; closes the silent-failure hole at `modules/profiling.md:275, 386, 413`.
- **Edit sketch:**
  - File: `modules/profiling.md:§4.1 / §7` — expose drop counters through `ProfilingStats` (already declared `modules/profiling.md:170`) into `RuntimeStats`.
  - File: `07-cross-cutting-concerns.md:§7.2.8` — add alert rows: "Trace drop rate > 0 over 1 min", "Sink degraded > 5 s".
- **Trade-offs:** Gain — trace-loss cannot silently invalidate budget validation. Give up — two more alert rows.
- **Sanity test:** Saturate a TLS ring at 10× normal rate; verify both the counter increments and the alert fires through the Prometheus sink.

#### A8-P10: Structured KV logging as the primary surface

- **Rationale:** O2 ("Structured, meaningful logs"). `modules/profiling.md:463` itself flags string-only logging as an open question.
- **Edit sketch:**
  - File: `modules/profiling.md:§2.3` — add `Logger::log_kv(Severity, category, std::initializer_list<KV>, correlation_id)`; keep `log(..., std::string_view)` as a trivial overload.
  - File: `07-cross-cutting-concerns.md:§7.2.4` — declare JSON-line sink default.
- **Trade-offs:** Gain — machine-parseable logs everywhere. Give up — porting cost.
- **Sanity test:** Run any integration test; `jq` the log file with no errors; every line has `{correlation_id, layer_id, category, severity, …}`.

#### A8-P11: HAL contract test suite run against both sim and onboard

- **Rationale:** X5 + DfT. `modules/hal.md:245-247` posits "Simulation parity … for all non-leaf engines" but does not require a shared contract-test suite; the two variants can silently drift.
- **Edit sketch:**
  - File: `modules/hal.md:§9` — add an explicit "HAL Contract Test Suite" bullet; name a single `.cpp` set runnable under both `a2a3sim` and `a2a3` onboard CI.
  - File: `03-development-view.md:§3.3.6` — reference the suite.
- **Trade-offs:** Gain — prevents parity rot; strengthens every higher module's test credibility. Give up — suite maintenance.
- **Sanity test:** CI required check; both `a2a3sim` and `a2a3` onboard jobs must run the identical suite and be green.

#### A8-P12: Stable `PhaseId`s for Submission lifecycle

- **Rationale:** O1 + `04-process-view.md:681-702` admission budget validation. Without stable phase ids, trace consumers must infer phase boundaries from heuristics.
- **Edit sketch:**
  - File: `modules/profiling.md:§2.5 / ids.h` — add stable `PhaseId` constants: `SUBMISSION_ADMIT`, `SUBMISSION_RETIRE`, `DATA_DEP_SCAN`, `BARRIER_ATTACH`, `NONE_ATTACH`, `WORKSPACE_ALLOC`, `ADMISSION_QUEUE_DRAIN`, `WORKER_DISPATCH`, `WORKER_COMPLETE`.
  - File: `modules/scheduler.md:§5.1 / §5.3` — add `Phase p(...)` at the matching boundaries.
- **Trade-offs:** Gain — direct validation of `§4.8.4` admission budget; reproducibility of ADR-013 producer_index soundness claims. Give up — more ring slots at L2 (stripped at L1).
- **Sanity test:** Trace over a 10 k-submission burst shows per-phase durations summing to observed admission latency within 1%.

---

## 5. Revisions of own proposals  (round ≥ 2 only)

_Not applicable in round 1._

## 6. Votes on peer proposals  (round ≥ 2 only)

_Not applicable in round 1._

## 7. Cross-aspect tensions

| pair | with_proposal | preferred_resolution |
|------|---------------|----------------------|
| A1 vs A8 | A8-P3 (stats enumeration + latency histograms) | Two-tier path: hot-path insert is a single branchless log-scale bucket-increment (< 5 ns); percentile computation is pulled by the cold stats reader on demand. Histogram memory is pre-allocated at init (no hot-path allocation). A1 retains hot-path veto over the exact insert instruction sequence. |
| A1 vs A8 | A8-P8 (AICore in-core trace upload at Level 2) | Two-tier path: release default `PROFILING_LEVEL = L1_Coarse` strips Level-2 AICore emits entirely (fast path = today); Level-2 is opt-in by operators (e.g., triggered by A8-P5 SLO alerts). A1 retains veto on the exact ring-emit sequence. |
| A1 vs A8 | A8-P12 (lifecycle PhaseIds) | Two-tier: phase emits compiled out below `L2_Phase`; with `L2_Phase` enabled, each emit < 100 ns per `Phase` per `modules/profiling.md:334`. A1 may demand measured microbenchmarks before accepting default-on. |
| A2 vs A9 | A8-P5 (alert rule schema) / A8-P7 (IFaultInjector) | These add new configuration / interface surface. A9 may flag as over-engineering. Preferred: (a) alert rules — minimal 4-field schema only; no DSL, no dynamic compilation. (b) `IFaultInjector` — sim-only, hidden behind `enable_test_driver`; zero impact on release builds. |
| A7 vs A9 | A8-P1 (`IClock`), A8-P2 (`RecordedEventSource` + `step()`), A8-P11 (contract tests) | These add small new interfaces. Preferred resolution: each new interface has a single production implementation and a single test implementation; no speculative implementations introduced. Keeps A7 happy (clear module boundary), keeps A9 happy (no plugin ceremony). |
| A5 vs A1 | A8-P7 (`IFaultInjector`) collides with A5's failure-path proposals | Resolution: uniform test seam satisfies both — A5 specifies *what* faults must be injected; A8-P7 specifies *how*. No hot-path cost because injector is compiled out in release. |

---

## 8. Stress-attack on emerging consensus  (round 3+ only)

_Not applicable in round 1._

## 9. Status

- **Satisfied with current design?** **partially**.
  - Strong foundation (HAL interfaces, SIM, correlation-ids, compile-time profiling strip, testing sections in every module) — the scaffolding is correct.
  - But O3 (concrete metrics & latency histograms), O4 (state inspection), O5 (externalized alerts), X5 (clock / timer / recorded-source injection seams) and the distributed O1 story (trace alignment) all have documented gaps that must close before implementation can meet the stated SLOs and budget-validation claims.
- **Open items expected in next round:**
  - A8-P1 (`IClock` interface) — expected to be contested on hot-path grounds; answer via link-time specialization (already established in `modules/hal.md:430, 247`).
  - A8-P3 (stats + latency histograms) — expected tension with A1; two-tier resolution above.
  - A8-P4 (`dump_state`) — expected tension with A9 (surface growth) and A1 (must guarantee zero hot-path cost); answer via cold reader + seqlock.
  - A8-P5 (alert rule externalization + metrics sink) — expected tension with A9 (YAGNI); answer via minimal 4-field schema.
  - A8-P7 (`IFaultInjector`) — expected convergence with A5's failure-mode proposals; route to joint ownership.
  - A8-P8 (AICore trace upload) — expected tension with A1; answer via default Level-1 strip.
