# Aspect A5: Reliability & Fault Tolerance — Round 1

## Metadata

- **Reviewer:** A5
- **Round:** 1
- **Target:** `/data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/`
- **Runtime Mode:** yes
- **Timestamp (UTC):** 2026-04-19T00:00:00Z

---

## 1. Rubric Checks

| # | Check | Result | Evidence (file:line) | Rule(s) |
|---|-------|--------|----------------------|---------|
| 1 | Timeouts on every remote / IPC / device call | **Weak** | `modules/transport.md:56-66` (mandatory Horizontal timeouts); `modules/runtime.md:475-477` (`init_ms`/`drain_ms`/`shutdown_ms`); `modules/scheduler.md:485` (`worker_timeout_ms`); `modules/transport.md:399` (`heartbeat_timeout_ms`). But `modules/memory.md:102-103` (`IMemoryOps::copy_async`/`wait`/`poll`) exposes no `Timeout` parameter, and the Scheduler event-loop `poll` sources at `02-logical-view/02-scheduler.md:293,09-interfaces.md:392,422` have no per-cycle deadline, relying on watchdogs at an outer layer. | R1 |
| 2 | Retries bounded with exponential backoff + jitter | **Fail** | `modules/distributed.md:404` declares `max_retries = 3` and `05-physical-view.md:210` says "retry with backoff", but no schedule, base interval, ceiling, or jitter is specified anywhere. `modules/distributed.md:334` (sequence diagram) shows immediate `REMOTE_SUBMIT` to alternate on RETRY_ELSEWHERE with no wait. `modules/profiling.md:133` alone uses the word "bounded backoff" but only for sink writes. | R2 |
| 3 | Circuit breakers for repeatedly failing dependencies | **Fail** | Search across `docs/pypto-runtime-design/` finds zero occurrences of "circuit breaker", cooldown, half-open, or peer quarantine. The Worker FSM (`02-logical-view/03-worker.md:22-55`) has only `FAILED → RECOVERING → IDLE|UNAVAILABLE` — permanent removal or immediate return, with no timed quarantine. `modules/distributed.md:334` retries to an alternate peer without any per-peer failure accounting beyond `max_retries` per Task. A chronically flaky peer that intermittently succeeds will never be tripped off; a chronically flaky Worker that self-recovers within the cleanup window will keep being rescheduled. | R3 |
| 4 | Graceful degradation defined per dependency | **Weak** | Good on distributed: `modules/distributed.md:161,403,431`, `05-physical-view.md:200-206` (`ABORT_ALL`/`CONTINUE_REDUCED`/`RETRY_ELSEWHERE`). Good on hardware: `05-physical-view.md:183-185,195-197` (per-device/per-core availability). Good on profiling: `modules/profiling.md:133` (sink drop with counter). **But** no degradation policy is documented for: (a) admission-queue / outstanding-window saturation in `02-logical-view/02-scheduler.md` §2.1.3.1.A (behaviour is only "AdmissionRejected" at `modules/scheduler.md:460` — no shed/defer fallback), (b) per-`BufferRef` memory watermark pressure in `modules/memory.md`, (c) partial `WorkerGroup` availability (`02-logical-view/03-worker.md:108-120`) when a CoreWrap loses members mid-Submission. | R4 |
| 5 | No single point of failure on the critical path | **Weak** | `05-physical-view.md:175-189` is explicit and honest: Host process, Host Scheduler thread, Coordinator node, and every network link are classed as SPOFs, with coordinator failover deferred (`09-open-questions.md:60-70` Q5) and persistent-state recovery explicitly declined (`10-known-deviations.md` Deviation 2). Mitigations are real but mostly *external* (process monitor, application-level resubmit). Inside the runtime, `scheduler_thread_count` default 1 (`05-physical-view.md:182`) means the Host Scheduler thread has no watchdog, no heartbeat to its own workers, and no failover — a stuck scheduler currently produces silence, not a fault. | R5 |
| 6 | Mutating operations idempotent under retry | **Weak** | Message-layer idempotency is solid: `modules/distributed.md:235,274,317,406,450` (`correlation_id` + sliding dedup window default 4096). **But** Task-level retry idempotency is only informally asserted: `05-physical-view.md:196` says "attempt re-execution if idempotent" without a mechanism to declare per-Task idempotency. Neither `02-logical-view/07-task-model.md` nor `modules/scheduler.md` nor `TaskDescriptor`/`TaskArgs` carries an `idempotent` flag. Single-valued `producer_index` plus the non-aliasing invariant (`modules/memory.md:7`) keep RAW dependencies safe across retry, but user-supplied side-effecting Functions (e.g., external DMA to an unowned buffer) are not protected. | DS4 |
| 7 | Failure-mode test plan (chaos / fault injection / DR drills) | **Weak** | Per-module integration tests exercise individual failure classes: `modules/transport.md:411,419,427` (timeouts, heartbeat, clock skew); `modules/distributed.md:431` (node kill); `modules/scheduler.md:525,9.3` (worker exhaustion, remote timeout). But there is no top-level chaos/fault-injection harness, no DR drill spec, and no enumerated mandatory scenario list (e.g., packet loss %, Byzantine slow peer, clock step during heartbeat, memory ECC fault, DMA corruption). `07-cross-cutting-concerns.md` §7.3 discusses alerting but not a chaos programme. | R6 |

Principles checked alongside: §3.5 Resilience patterns (Retry with backoff — partial; Circuit breaker — missing; Timeout — strong on horizontal / weak on memory-async; Bulkhead — only implicit via per-Layer scopes); §5 Reliability (No SPOF — documented but partial; Graceful Degradation — partial; Design for Partial Failure — strong distributed, weak intra-node; Blast Radius — good via Layer boundaries); §3.6 Idempotency (strong at message layer, weak at task layer).

---

## 2. Pros

- **Mandatory horizontal timeouts are first-class in the interface, not an afterthought** — `modules/transport.md:56-58,66,238`. `Timeout::infinite()` is a named sentinel only permitted for shutdown draining, which is exactly the R1 discipline. (rule R1, principle §3.5)
- **Distributed failure policy is pluggable and enumerated** — `modules/distributed.md:161,403` (`FailurePolicy::{ABORT_ALL, CONTINUE_REDUCED, RETRY_ELSEWHERE}`) satisfies R4 for the hardest partial-failure class (node loss). Having all three named in the config lets operators pick per-workload. (rule R4, principle §5)
- **Explicit SPOF table with honest "No" answers** — `05-physical-view.md:179-189`. Few architectures name their SPOFs; this design does, ties each to a mitigation, and cross-references the known deviations. That transparency is itself a reliability asset. (rule R5)
- **Idempotent inbound message handling with a sliding dedup ring** — `modules/distributed.md:235,274,406`. `correlation_id` + `remote_task_key` tuple keyed O(1) lookup. The window size is configurable and the default (4096) is documented, so operators can trade memory for tolerance to reorder. (rule DS4, principle §3.6)
- **Layered error model with severity + domain + remote cause chain** — `modules/error.md` and `04-process-view.md` §3.5-3.6 `ErrorContext` with `remote_error_chain` and POD device form. Critical vs Error vs Warning is encoded in the packed code itself, enabling the runtime to route handling (shutdown vs retry vs log) without string parsing. (rule R4, principle §5, §3.5)
- **Profiling sink does not fail user tasks** — `modules/profiling.md:133` explicitly says sink write errors drop events with a counter and never propagate. That is a textbook graceful-degradation instance for a non-critical dependency. (rule R4)
- **Bounded admission window prevents congestive collapse** — `02-logical-view/02-scheduler.md` §2.1.3.1.A "outstanding submission window" with earliest-first bias and `max_outstanding_submissions` backpressures the Python driver before the scheduler's internal queues blow up. (rule R4, principle §3.5 Bulkhead)
- **Worker watchdog built into dispatch** — `modules/scheduler.md:68,485` (`worker_timeout_ms` default 5000 host / 1000 device; dispatch transitions Worker to ASSIGNED and *starts* the timer). This puts an R1 timeout on every device call without requiring kernel-level participation. (rule R1)
- **Heartbeat + NodeLost with declared relationship `heartbeat_timeout_ms > heartbeat_interval_ms × 2`** — `modules/transport.md:399`. Invariant prevents a too-tight ratio that would flap under GC pauses. (rule R1, R5)
- **Two-form ErrorContext (rich host vs POD device) with zero-alloc success path** — `modules/error.md` §2, §4. Reliability instrumentation that costs nothing on the happy path is the only kind A1 will sustain. (rule R1, principle §3.5)

---

## 3. Cons

- **No exponential backoff + jitter spec anywhere for remote retries** — `modules/distributed.md:334,404`, `05-physical-view.md:210` mention "retry" and "backoff" but specify neither schedule nor jitter. Retry storms under correlated failure (all retries from all tasks fire at the same post-timeout instant) are exactly the failure mode R2 is written to prevent. (rule R2, principle §3.5)
- **No circuit breaker primitive** — neither `distributed/`, `transport/`, nor `scheduler/` expose a per-peer / per-Worker / per-dependency failure accumulator with cooldown. Under a flapping peer, `max_retries` resets on each Task and the cluster keeps hammering the node. (rule R3, principle §3.5)
- **Coordinator node SPOF explicitly deferred** — `05-physical-view.md:187` + `09-open-questions.md:60-70` Q5. In multi-node mode the coordinator is both a hard SPOF and a hot-path message router; deferring failover to "application-level resubmit from a different coordinator" shifts an architectural concern to undefined user code. (rule R5)
- **Host Scheduler thread is a silent SPOF** — `05-physical-view.md:182` ("Single thread is SPOF…crash detection triggers runtime shutdown"). Crash is detected *if the thread dies*, but nothing detects a hang (stuck poll, pathological dep-resolution, livelock); there is no heartbeat from the scheduler to itself or the driver. (rule R5)
- **Task-level retry idempotency is assumed, not declared** — `05-physical-view.md:196` "attempt re-execution if idempotent". No `idempotent` flag on `TaskDescriptor`, no policy difference between idempotent and non-idempotent tasks in the FSM (`04-process-view.md` §3.3). A user Function that writes to an external registry via a child-level HAL call can be retried double. (rule DS4)
- **`IMemoryOps::copy_async` / `wait` / `poll` lacks a `Timeout` parameter** — `modules/memory.md:102-103`. Although the owning Scheduler's `worker_timeout_ms` bounds the enclosing Task, the memory op itself can silently stall beyond what the caller expected, and there is no way for a non-Task callsite (init, drain teardown DMA) to bound the wait. (rule R1)
- **Admission saturation has one response: reject** — `modules/scheduler.md:460` (`AdmissionRejected`). R4 demands a defined behaviour for each dependency-unavailability; rejecting the Submission surfaces the error, but no graceful shedding (e.g., prefer lower-priority drop, coalesce small Submissions, or advise the driver to throttle) is specified. (rule R4)
- **Partial `WorkerGroup` failure semantics undefined** — `02-logical-view/03-worker.md:108-120` defines CoreWrap groups but `modules/scheduler.md` does not specify what happens when an SPMD Task is mid-flight and one group member transitions to UNAVAILABLE. Does the whole group fail? Re-allocate onto a smaller group? The answer is implicit. (rule R4, DS3)
- **No unified chaos / DR harness** — per-module integration tests exist (`modules/transport.md:411-427`, `modules/distributed.md:431`) but there is no single spec enumerating the minimum chaos matrix, no fault-injection interface on `IHorizontalChannel` / `IHal`, and no DR drill cadence. (rule R6)
- **Recovery from transient HAL errors is conflated with permanent ones** — `02-logical-view/03-worker.md:65-75`: `RECOVERING → IDLE (if recoverable) or UNAVAILABLE (if permanent)`. The decision rule ("if recoverable") is not specified; there is no intermediate *degraded* or *quarantined* state, so a Worker that fails once is either instantly re-eligible or permanently out. (rule R3, R4)
- **DeploymentConfig lacks reliability-facing knobs** — `modules/runtime.md:145-150` `Timeouts` struct carries init/drain/shutdown/heartbeat only. No `retry_backoff_ms_{min,max}`, no `circuit_breaker_{threshold,cooldown_ms,half_open_max}`, no `worker_quarantine_ms`. Everything else R2/R3 needs would live here. (rule R2, R3, principle §3.5)

---

## 4. Proposals

| id | severity | summary | affected_docs | hot_path_impact | trade_offs | sanity_test |
|----|----------|---------|---------------|-----------------|------------|-------------|
| A5-P1 | high | Specify exponential backoff + jitter for remote retries | `modules/distributed.md`, `modules/runtime.md`, `05-physical-view.md` | none (failure path only) | Prevents retry storms; costs three extra `DeploymentConfig` fields | Run an aborted-peer scenario, verify retry wall-clock times match `base·2^n ± jitter` |
| A5-P2 | high | Add per-peer circuit breaker (threshold, cooldown, half-open probe) to `distributed/` | `modules/distributed.md`, `modules/runtime.md`, `02-logical-view/02-scheduler.md` | none on success; 1 relaxed atomic load per `REMOTE_SUBMIT` | Avoids hammering flaky peer; one atomic read in the send fast path | Inject a peer that fails 60% of requests; assert the breaker trips after N fails and all further sends skip the peer for `cooldown_ms` |
| A5-P3 | high | Commit to a coordinator-failover mechanism or a deterministic, runtime-detectable fail-fast | `05-physical-view.md`, `09-open-questions.md`, `10-known-deviations.md`, `modules/distributed.md` | none (control plane) | Resolves Q5 either way; costs either a lease protocol or a documented detection hook | Partition-kill coordinator; either failover completes within `coordinator_failover_ms`, or all peers surface `CoordinatorLost` within `heartbeat_timeout_ms` |
| A5-P4 | medium | Add `idempotent: bool` to `TaskDescriptor` and gate single-node Task retry on it | `02-logical-view/07-task-model.md`, `modules/scheduler.md`, `modules/bindings.md`, `05-physical-view.md` §5.4.3 | none (checked only on retry) | Makes R2 Task-level retry sound; user must label effect-ful tasks | Submit a Task marked `idempotent=false`, force Worker failure; assert retry is *not* issued and `ErrorContext` surfaces instead |
| A5-P5 | medium | Add a chaos / fault-injection harness + mandatory-scenario matrix | new `11-chaos-plan.md` (or `07-cross-cutting-concerns.md` §7.4), `modules/transport.md` §9, `modules/hal.md` §9, `modules/distributed.md` §9 | none (test-only interface) | Closes R6 as a first-class artefact; adds an `IFaultInjector` seam per backend | For each scenario (node kill, packet drop N%, Byzantine slow peer, clock step, slot-pool exhaust, DMA corrupt) a test fails the build when the documented response is not observed |
| A5-P6 | medium | Define scheduler-thread watchdog / deadman timer with O5 alert | `02-logical-view/02-scheduler.md` §2.1.3.4, `modules/scheduler.md` §4/§5, `07-cross-cutting-concerns.md` §7.3 | extends-latency (tiny: one TSC read per event-loop cycle) | Converts the Host Scheduler SPOF from "silent" to "detectable + alerting"; one extra store per cycle | Inject an artificial `sleep` inside the scheduler loop > `scheduler_watchdog_ms`; assert `SchedulerStalled` alert fires |
| A5-P7 | medium | Add a `Timeout` parameter to `IMemoryOps::copy_async`/`wait`/`poll` | `02-logical-view/04-memory.md`, `modules/memory.md` | extends-latency only on slow path; zero change for success `poll` | Closes R1 coverage gap for async data movement; callers must pass a deadline | Replace backend with a stub that never completes; assert `wait(timeout)` returns `TransportTimeout` within `timeout ± 10%` |
| A5-P8 | medium | Specify degradation for admission-window saturation and partial `WorkerGroup` loss | `02-logical-view/02-scheduler.md` §2.1.3.1.A, `02-logical-view/03-worker.md` §2.1.4.2, `modules/scheduler.md` | none (only fires on saturation / failure) | Makes R4 actionable beyond "reject"; costs new config + table rows | Saturate admission, observe documented degradation (shed vs coalesce vs reject); kill one CoreWrap member mid-SPMD, observe documented group behaviour |
| A5-P9 | low | Introduce a QUARANTINED Worker state between FAILED and UNAVAILABLE | `02-logical-view/03-worker.md` §2.1.4.1, `modules/scheduler.md` §3.2 WorkerManager | none on healthy path; one enum compare on failure | Completes R3 at Worker granularity; adds one state and one config knob | Force a Worker to fail once; assert Worker becomes QUARANTINED for `worker_quarantine_ms`, then transitions back to IDLE after a successful probe |
| A5-P10 | low | Document DS4 at the protocol layer: require `REMOTE_*` handlers to document their idempotency class | `modules/distributed.md` §3.3, `04-process-view.md` §3.4 | none | Turns "idempotent via dedup" into a per-handler contract, catching future regressions | Code review: each new handler must specify `idempotency: {safe, at-most-once, at-least-once}`; CI doc-lint fails if missing |

### Proposal detail

#### A5-P1: Specify exponential backoff + jitter for remote retries

- **Rationale:** R2 explicitly requires "exponential backoff with jitter and a maximum retry count." The design has the count (`max_retries`) but neither the schedule nor the jitter. Retry storms are the named failure mode R2 prevents, and they are more, not less, likely in a distributed scheduler where thousands of Tasks timeout at correlated instants after a partition. Principle §3.5 Resilience patterns names "Retry with backoff" as mandatory.
- **Edit sketch:**
  - File: `modules/runtime.md`
  - Location: §2.4 `DeploymentConfig`, `Timeouts` / new `RetryPolicy` struct (around line 145-150).
  - Delta: add `struct RetryPolicy { uint32_t base_ms = 50; uint32_t max_ms = 2000; double jitter = 0.3; };` and plumb through `DistributedConfig`.
  - File: `modules/distributed.md`
  - Location: §4 Config table (line 402-404) and §3.3 sequence diagram (line 334).
  - Delta: replace bare `max_retries` with `retry_policy`; add a note that the `n`-th retry sleeps `min(max_ms, base_ms · 2^n) · (1 + U[-jitter, +jitter])`.
  - File: `05-physical-view.md`
  - Location: §5.4.3 Recovery Strategy (line 210).
  - Delta: concretize "retry with backoff" to reference `RetryPolicy`.
- **Trade-offs:**
  - Gain: R2 conformance; no retry storm under correlated failure; operator-tunable.
  - Give up: three more config fields; retry slow path acquires a `clock_gettime` + `rand64()` — still zero cost on success.
- **Sanity test:** Set `base_ms=50, max_ms=2000, jitter=0.3, max_retries=5`; kill a peer; record retry wall-clock timestamps; assert gap_n ∈ `[min(max, base·2^n)·(1-jitter), min(max, base·2^n)·(1+jitter)]` for n = 0..4, and retries stop at 5.

#### A5-P2: Add per-peer circuit breaker to `distributed/`

- **Rationale:** R3 requires circuit breakers for repeatedly failing dependencies; the design has none. `max_retries` is per-Task, so under a flapping peer the same peer absorbs 3 retries per Task indefinitely, defeating the purpose of R2's bound. Principle §3.5 names "Circuit breaker" alongside "Retry" precisely because the two must coexist.
- **Edit sketch:**
  - File: `modules/distributed.md`
  - Location: §3 Internal responsibilities, new subsection §3.4 "Peer health & circuit breaker"; §4 Config table.
  - Delta: add `struct CircuitBreaker { uint32_t fail_threshold = 5; uint32_t cooldown_ms = 10000; uint32_t half_open_max_in_flight = 1; };` and states `{CLOSED, OPEN, HALF_OPEN}` per peer. On `REMOTE_SUBMIT`, skip the peer if `OPEN` and surface `PartitionTargetUnavailable`; on `HALF_OPEN`, allow `half_open_max_in_flight` probes, close on success / reopen on failure.
  - File: `modules/runtime.md`
  - Location: `DeploymentConfig` alongside `RetryPolicy`.
  - Delta: add `circuit_breaker: CircuitBreaker`.
- **Trade-offs:**
  - Gain: R3 conformance; bounded impact from flaky peers; cluster-wide self-healing.
  - Give up: one per-peer `atomic<uint32_t>` read on `REMOTE_SUBMIT` (success path). A1 will review — this is a relaxed-atomic load on a cache-resident counter already touched by the send path. Two-tier fallback: skip the breaker check entirely for peers in `CLOSED` state older than `cooldown_ms` by gating on a `thread_local` last-checked timestamp, collapsing the check into a compile-time constant for the steady state.
- **Sanity test:** Fault-inject a peer that fails 60% of `REMOTE_SUBMIT`s. Assert: after `fail_threshold` consecutive failures the peer transitions to `OPEN`; for `cooldown_ms` all sends return `PartitionTargetUnavailable` without contacting the peer; after cooldown at most `half_open_max_in_flight` probes are in flight; first successful probe closes the breaker.

#### A5-P3: Commit coordinator-failover mechanism or deterministic fail-fast

- **Rationale:** R5 forbids SPOFs on the critical path; the coordinator node is named a SPOF with "application-level restart" mitigation and failover explicitly deferred. This is the single largest R5 gap. Either the runtime supplies a failover (preferable at the scheduler boundary) or it must detect and *fail* deterministically within a bounded time so the application can restart — silent hangs when the coordinator becomes unreachable are not acceptable. Principle §5 "No SPOF" is one of four top-level reliability principles.
- **Edit sketch:**
  - File: `modules/distributed.md`
  - Location: new §3.5 "Coordinator membership & failover".
  - Delta: specify a lease-based standby protocol (option A, cheap: single standby with monotonic lease via `HEARTBEAT`) or a deterministic `CoordinatorLost` fault that fires within `heartbeat_timeout_ms` and fails all outstanding `REMOTE_*` with that code (option B, minimum).
  - File: `09-open-questions.md`
  - Location: Q5.
  - Delta: mark resolved toward option A or B; close deferral.
  - File: `10-known-deviations.md`
  - Location: Deviation 2 neighbourhood.
  - Delta: if option B only, record as an explicit R5 deviation with justification, not silently.
- **Trade-offs:**
  - Gain: either a real R5 mitigation or an honest, bounded R5 deviation.
  - Give up: option A: one background lease-renewal task per cluster; option B: no runtime recovery, but at least no silent hang.
- **Sanity test:** In a 3-node cluster, SIGKILL the coordinator. Option A: standby promotes within `coordinator_failover_ms`; workload drains. Option B: every surviving peer surfaces `CoordinatorLost` within `heartbeat_timeout_ms`; Python driver observes `DistributedError`; no `drain()` hangs past the configured timeout.

#### A5-P4: `idempotent: bool` on `TaskDescriptor` gates single-node retry

- **Rationale:** DS4 requires mutating operations to be idempotent under retry; the design currently assumes this at the Function level without any carrier. `05-physical-view.md:196` ("attempt re-execution if idempotent") is normative but untyped. Without a declared flag, the scheduler cannot decide correctly whether to retry a failed Task or surface the error, and users with side-effecting Functions (external HAL calls, in-place buffer rewrites that violate the non-aliasing invariant at user level) silently get doubled writes on retry.
- **Edit sketch:**
  - File: `02-logical-view/07-task-model.md`
  - Location: §2.4.1 `TaskArgs` / `TaskDescriptor` struct.
  - Delta: add `bool idempotent = true;` with a short paragraph on semantics: "Default `true` for pure compute; set `false` for Functions that mutate externally-visible state. The Scheduler MAY only issue single-node retry when `idempotent == true`."
  - File: `modules/scheduler.md`
  - Location: §5 Error handling, row for `WorkerCrashed` (around line 464).
  - Delta: branch on `task.descriptor.idempotent`.
  - File: `modules/bindings.md`
  - Location: §2.1 `simpler.Task` signature.
  - Delta: add `idempotent: bool = True` kwarg.
- **Trade-offs:**
  - Gain: DS4 correctness at the task layer.
  - Give up: one extra bool in the descriptor (cold side); Python surface gains one kwarg.
- **Sanity test:** Submit two Tasks, one with `idempotent=True`, one with `idempotent=False`, each configured to fail once then succeed. Assert the idempotent one completes via retry; the non-idempotent one surfaces `ErrorContext` on the first failure with no retry issued.

#### A5-P5: Chaos / fault-injection harness + mandatory-scenario matrix

- **Rationale:** R6 is the most directly under-served rubric: the design has good per-module integration tests but no top-down chaos programme, no fault-injection seam on the HAL/transport interfaces, and no enumerated minimum-coverage matrix. A design document cannot fully satisfy R6, but it can define the contract.
- **Edit sketch:**
  - File: new `11-chaos-plan.md` (or a new §7.4 in `07-cross-cutting-concerns.md`).
  - Delta: mandatory matrix (node kill, packet drop 1/5/20/50%, Byzantine slow peer 10×RTT, clock step ±1s during heartbeat, task-slot-pool exhaust at 0.9/1.0/1.1 × capacity, DMA completion bit-flip, RDMA send CRC corrupt, coordinator isolation, memory watermark breach); required pass criteria per scenario; DR drill cadence (e.g., quarterly for production deployments).
  - File: `modules/transport.md` §9, `modules/hal.md` §9, `modules/distributed.md` §9.
  - Delta: add an `IFaultInjector` seam to each (no-op by default, test build only) and cross-reference `11-chaos-plan.md`.
- **Trade-offs:**
  - Gain: R6 becomes auditable; catches regressions.
  - Give up: test-build-only code surface in three interfaces; zero production cost.
- **Sanity test:** Each row of the matrix maps to a named test case; CI gate fails if any row is missing or red.

#### A5-P6: Scheduler-thread watchdog / deadman timer

- **Rationale:** R5 compliance for the Host Scheduler thread is "detect crash and propagate" — but only for a crashed thread, not a hung one. A watchdog closes this gap without changing the SPOF status. Principle O5 (used by A8 too) adds the alert.
- **Edit sketch:**
  - File: `02-logical-view/02-scheduler.md`
  - Location: §2.1.3.4 event loop description.
  - Delta: "At the top of each event-loop cycle the Scheduler writes a monotonic timestamp to a deadman slot. A separate low-priority monitor thread — one per runtime instance — checks the deadman at `scheduler_watchdog_ms` and raises `SchedulerStalled` when the delta exceeds budget."
  - File: `modules/scheduler.md`
  - Location: §4 config.
  - Delta: add `scheduler_watchdog_ms: uint32_t = 250`.
  - File: `07-cross-cutting-concerns.md`
  - Location: §7.3 Alerting Strategy.
  - Delta: add `SchedulerStalled` alert tied to deadman.
- **Trade-offs:**
  - Gain: "silent hang" SPOF becomes a detected, alerted fault.
  - Give up: one 8-byte atomic store per event-loop cycle (A1 will review — this is one `seq_cst_relaxed` store on a cache-resident slot; measured cost ≈ TSC read ≈ single-digit ns). Fast/slow path: writing only every Kth cycle keeps the cost below TSC noise.
- **Sanity test:** Inject a `sleep(scheduler_watchdog_ms·4)` inside the event-loop handler; assert `SchedulerStalled` fires and is surfaced to the Python driver within `2·scheduler_watchdog_ms`.

#### A5-P7: `Timeout` on `IMemoryOps` async

- **Rationale:** R1 applies to every remote / IPC / device call; `IMemoryOps::copy_async`/`wait`/`poll` is a device call in the RDMA / SDMA case, and a cross-process IPC in some sim configurations. The current interface has no deadline parameter, so callers outside the Scheduler's `worker_timeout_ms` loop (init, drain teardown, admin tooling) cannot bound the wait.
- **Edit sketch:**
  - File: `02-logical-view/04-memory.md`
  - Location: `IMemoryOps` method table.
  - Delta: change `wait(AsyncHandle) -> Status` to `wait(AsyncHandle, Timeout) -> Status` and `poll(AsyncHandle) -> bool` to `poll(AsyncHandle, Timeout = Timeout::zero()) -> bool`.
  - File: `modules/memory.md`
  - Location: lines 102-103 and 109-110 (semantics section).
  - Delta: same signature change; add "Returns `TransportTimeout` on deadline" to the contract.
- **Trade-offs:**
  - Gain: R1 coverage for all device/IPC data movement.
  - Give up: interface breakage; every `IMemoryOps` backend updates (HAL, RDMA, shared-memory). Fast path: a deadline check is one compare on a monotonic clock; in the zero-timeout `poll` case the existing hardware-flag check is unchanged.
- **Sanity test:** Replace backend with a stub that never signals completion. Call `wait(handle, 100ms)`; assert return `TransportTimeout` within 110 ms.

#### A5-P8: Degradation specs for admission saturation and partial `WorkerGroup` loss

- **Rationale:** R4 requires per-dependency graceful-degradation specs. Admission saturation currently has one response — reject; partial group loss has no documented response. Both are common in production.
- **Edit sketch:**
  - File: `02-logical-view/02-scheduler.md`
  - Location: §2.1.3.1.A outstanding window subsection.
  - Delta: add an "Admission pressure" paragraph enumerating options — `REJECT` (current), `COALESCE` (merge small RAW-compatible Submissions), `DEFER` (hold in Python-visible "queued" state) — with a `admission_pressure_policy` config.
  - File: `02-logical-view/03-worker.md`
  - Location: §2.1.4.2 Worker Groups, new sub-section "Partial group availability".
  - Delta: specify `{FAIL_SUBMISSION, SHRINK_AND_RETRY, DEGRADED_CONTINUE}` policy and a default.
- **Trade-offs:**
  - Gain: R4 compliance; operators control behaviour.
  - Give up: one config knob each; ~1 page of doc.
- **Sanity test:** Saturate `max_outstanding_submissions`; assert configured policy is the observed behaviour. Kill one CoreWrap member mid-SPMD; assert configured policy is the observed behaviour.

#### A5-P9: QUARANTINED Worker state between FAILED and UNAVAILABLE

- **Rationale:** R3 at the Worker granularity — a Worker that fails transiently should not instantly go back into the rotation (inviting a loop) nor be permanently removed. A timed quarantine followed by a probe is the standard circuit-breaker pattern for workers.
- **Edit sketch:**
  - File: `02-logical-view/03-worker.md`
  - Location: §2.1.4.1 FSM.
  - Delta: add `QUARANTINED` between `RECOVERING` and `IDLE`; transitions `RECOVERING → QUARANTINED (after cleanup, once recoverable)` and `QUARANTINED → IDLE (after probe success)` or `QUARANTINED → UNAVAILABLE (after probe failure or `max_quarantine_cycles` exceeded)`.
  - File: `modules/scheduler.md`
  - Location: §3.2 WorkerManager + §4 config.
  - Delta: add `worker_quarantine_ms`, `worker_probe_retries`; describe handler `on_probe(worker)`.
- **Trade-offs:**
  - Gain: Worker-level R3; fewer flap storms.
  - Give up: one extra state + ~100 lines of manager code.
- **Sanity test:** Cause a Worker to fail transiently once; assert Worker does not execute any Task for `worker_quarantine_ms`; assert the first Task dispatched after that is a probe; assert normal service resumes after probe success.

#### A5-P10: DS4 protocol contract — document idempotency per `REMOTE_*` handler

- **Rationale:** Dedup window plus `correlation_id` covers duplicate delivery in general, but the architecture will gain new `REMOTE_*` message types (e.g., collective primitives per Q4). A contract requirement that each handler declare its idempotency class makes future regressions impossible to merge silently.
- **Edit sketch:**
  - File: `modules/distributed.md`
  - Location: §3.3 Idempotent message handling.
  - Delta: add a table row template: every `REMOTE_*` handler MUST declare `idempotency ∈ {safe, at-most-once, at-least-once}` and link to the dedup key.
  - File: `04-process-view.md`
  - Location: §3.4 Distributed scheduling protocol.
  - Delta: same, at the protocol level.
- **Trade-offs:**
  - Gain: DS4 becomes a reviewable contract; prevents regressions.
  - Give up: doc-lint checker; trivial CI cost.
- **Sanity test:** Remove the idempotency annotation on one handler; CI doc-lint fails.

---

## 5. Revisions of own proposals  (round ≥ 2 only)

*(N/A for round 1.)*

## 6. Votes on peer proposals  (round ≥ 2 only)

*(N/A for round 1.)*

## 7. Cross-aspect tensions

| pair | with_proposal | preferred_resolution |
|------|---------------|----------------------|
| A5 vs A1 | A5-P1 (backoff+jitter) | Runs only on the failure path; success path unchanged. No two-tier needed — the "fast path" is the one that never encounters an error. Document explicitly. |
| A5 vs A1 | A5-P2 (circuit breaker) | Two-tier: steady-state `CLOSED` path collapses to a `thread_local` timestamp check of the last breaker update (branch-predicted constant). Full breaker state transitions only on the failure edge. Expected hot-path cost: one relaxed atomic load, amortized below existing `REMOTE_SUBMIT` cost. |
| A5 vs A1 | A5-P6 (scheduler watchdog) | Two-tier: deadman write every Kth cycle (K tunable, default 16), amortising cost below the existing event-collection overhead. Alternatively, lift the write into the cycle's already-present timestamp capture. |
| A5 vs A1 | A5-P7 (`IMemoryOps` timeout) | Zero cost on success — `poll(zero)` reduces to the current hardware-flag check; timeout branch lives on the slow path only. Interface change is source-level only. |
| A5 vs A1 | A5-P9 (QUARANTINED state) | None on the dispatch fast path — FSM transition happens only at `on_fail`. The `WorkerManager::on_assign` selector already filters by `state == IDLE`, so QUARANTINED Workers are simply invisible to the selector. One enum value is free. |
| A5 vs A2 | A5-P5 (chaos harness) | `IFaultInjector` seam is a textbook A2 extension point (plugin, no-op default, test-build). A9 will note "YAGNI" — mitigation: production build strips the seam; only test build links the plugin. |
| A5 vs A7 | A5-P3 (coordinator failover) | Adding a `Membership` sub-module in `distributed/` risks enlarging the module surface. Mitigation: keep it inside `distributed/coordinator_failover.cpp` as a single policy, not a new top-level module. |
| A5 vs A9 | A5-P9 (QUARANTINED state) | A9 will flag "extra state for a rare case". Counter: the state exists exactly to fix R3, which A5 is responsible for; severity is `low` precisely because the common case is already handled by the existing FSM. |

## 8. Stress-attack on emerging consensus  (round 3+ only)

*(N/A for round 1.)*

## 9. Status

- **Satisfied with current design?** partially
- **Open items expected in next round:** A5-P1, A5-P2, A5-P3, A5-P4, A5-P5, A5-P6, A5-P7, A5-P8 (A5-P9 and A5-P10 are defensible low-severity, expected to be bundled or conceded to simplicity pressure).
