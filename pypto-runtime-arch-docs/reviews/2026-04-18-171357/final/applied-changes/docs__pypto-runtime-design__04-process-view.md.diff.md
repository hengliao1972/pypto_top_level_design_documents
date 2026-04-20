# Applied Changes — `docs/pypto-runtime-design/04-process-view.md`

Run folder: `reviews/2026-04-18-171357/`.
Proposals applied (from `final/final-proposal.md`): **A1-P3**, **A1-P5**, **A1-P6**, **A1-P12**, **A3-P14**, **A5-P1**, **A5-P3**, **A7-P5**.

Line-count delta: 721 → 767 (+46).

Each change is a tightly-scoped `> [UPDATED: <id>: <reason>]` callout; no section rewrites.

---

## Change 1 — A1-P12 (`BatchedExecutionPolicy.max_batch` + `Dedicated` default pin)

**Anchor:** §4.1.4 "Deployment Modes at Runtime" — immediately after the `Dedicated / Interleaved / Batched` table.

### Diff

```diff
 | **Batched** | `BatchedExecutionPolicy` | Accumulates N ready tasks, executes all inline, then returns to scheduling. Reduces context-switch overhead. | Chip (batch dispatch to cores) |

+> [UPDATED: A1-P12: Pin `Dedicated` as the default IExecutionPolicy; `BatchedExecutionPolicy` must be explicit.]
+> `Dedicated` is the default `IExecutionPolicy` at every level (including Chip). No path silently enables `BatchedExecutionPolicy`; it is opt-in via deployment config. When Batched is active, `BatchedExecutionPolicy.max_batch` default = 8 at Chip level and tail latency for a batched task ≤ `max_batch × Chip→Core stage` (≤ 16 μs at `max_batch = 8`). §4.8.1 uses the adjusted stage target whenever Batched is active.
+
 This flexibility allows the same Scheduler implementation to operate efficiently across different hardware contexts without code changes — only the `IExecutionPolicy` and `IEventSource` registrations differ.
```

---

## Change 2 — A1-P3 (Function Cache LRU + HEARTBEAT Bloom) and A1-P6 (REMOTE_BINARY_PUSH staging)

**Anchor:** §4.2.4 Function Registration and Caching Flow — immediately after the existing ASCII flow diagram.

### Diff

```diff
   │ For distributed (REMOTE_SUBMIT to Node₁):
   │   Check if Node₁ cache has content_hash
   │   If not: include binary in REMOTE_SUBMIT message
   │   If yes: reference by content_hash only
 ```

+> [UPDATED: A1-P3: Function Cache is bounded and LRU; cache presence advertised via HEARTBEAT Bloom filter.]
+> The per-node Function Cache is bounded by `function_cache_bytes` (default **64 MiB**) and evicts LRU. Evictions are counted in `RuntimeStats` and surfaced by A8-P5 alert rules. Peer cache presence is published in `HeartbeatPayload.function_bloom[4]` (`uint64_t[4]`); the coordinator Bloom-checks the target before deciding whether to inline a binary in `REMOTE_SUBMIT`.
+>
+> [UPDATED: A1-P6: Function binaries stage over `REMOTE_BINARY_PUSH`, not inlined in `REMOTE_SUBMIT`.]
+> When the Bloom check (A1-P3) indicates a peer does **not** hold the binary for `content_hash`, the coordinator issues `MessageType::REMOTE_BINARY_PUSH` **before** the first `REMOTE_SUBMIT` that needs it. `RemoteSubmitPayload` carries only `uint32_t descriptor_template_id` + delta-encoded `TaskDescriptor[]` against a per-peer template registry (see `modules/transport.md §2.3` / `§4.3`). Per-peer projection (absorbed A10-P5) emits only tasks/edges/boundaries touching that peer subset; a shared `correlation_id` lets `REMOTE_DEP_NOTIFY` still join across peers.
```

---

## Change 3 — A3-P14 (leaf `EXECUTING → COMPLETED` skip)

**Anchor 1:** §4.3.1 State Transition Diagram — add a skip edge on the `EXECUTING → COMPLETING` transition block.

### Diff (FSM diagram)

```diff
     ┌──────────┐
-    │EXECUTING │
-    └────┬─────┘
-         │ Function body returns
-         ▼
+    │EXECUTING │ ────────────┐
+    └────┬─────┘             │ (leaf worker, no children: skip COMPLETING)
+         │ Function body     ▼
+         │ returns       (→ COMPLETED)
+         ▼
     ┌───────────┐
     │COMPLETING │ ◄──── waiting for child Tasks (if any)
     └────┬──────┘
```

**Anchor 2:** §4.3 opening paragraph — explanatory callout.

### Diff (opening paragraph)

```diff
 The task state machine is **uniform across all layers**. Every Task progresses through the same states, though not all transitions are meaningful at every level.

+> [UPDATED: A3-P14: Leaf workers (AICore and any other non-orchestrator executor) take the `EXECUTING → COMPLETED` skip edge, bypassing `COMPLETING`.]
+> Orchestration Tasks still transition `EXECUTING → COMPLETING → COMPLETED` because they wait on children. Leaf Tasks with no children (AICore compute, data-movement, barrier) transition directly `EXECUTING → COMPLETED`. §4.3.5 level-specific table records which layers take the skip edge via the `Skip COMPLETING?` column.
```

**Anchor 3:** §4.3.5 Level-Specific Notes — add `Skip COMPLETING?` column to the per-level table.

### Diff (§4.3.5 table)

```diff
-| Level | Worker Types | Grouping | ASSIGNED Duration | COMPLETING Applicable? | Failure Detection |
-|-------|-------------|----------|-------------------|----------------------|-------------------|
-| `"Core"` | AIC, AIV (heterogeneous) | Core Wraps (1 AIC + 2 AIV) | Register write → ACK | No (leaf executor) | COND register timeout |
-| `"Chip"` | AICore cluster (logical) | None (flat pool) | Dispatch payload write | No | Aggregate core failures |
-| `"Device"` | AICPU thread | None (homogeneous) | Thread wakeup | Yes (orchestration children) | Thread exit detection |
-| `"Host"` | Device worker | None (homogeneous) | DMA setup | Yes (device-level children) | DMA timeout |
-| `"Pod"` | Node worker | None (homogeneous) | Network message | Yes (remote children) | Heartbeat timeout |
+| Level | Worker Types | Grouping | ASSIGNED Duration | COMPLETING Applicable? | Skip COMPLETING? (A3-P14) | Failure Detection |
+|-------|-------------|----------|-------------------|----------------------|-------------------------|-------------------|
+| `"Core"` | AIC, AIV (heterogeneous) | Core Wraps (1 AIC + 2 AIV) | Register write → ACK | No (leaf executor) | Yes — leaf; `EXECUTING → COMPLETED` | COND register timeout |
+| `"Chip"` | AICore cluster (logical) | None (flat pool) | Dispatch payload write | No | Yes — leaf aggregator; `EXECUTING → COMPLETED` when the dispatched cores terminate | Aggregate core failures |
+| `"Device"` | AICPU thread | None (homogeneous) | Thread wakeup | Yes (orchestration children) | Only when submitted Task has no children | Thread exit detection |
+| `"Host"` | Device worker | None (homogeneous) | DMA setup | Yes (device-level children) | Only when submitted Task has no children | DMA timeout |
+| `"Pod"` | Node worker | None (homogeneous) | Network message | Yes (remote children) | Only when submitted Task has no children | Heartbeat timeout |
```

---

## Change 4 — A5-P1, A5-P3, A7-P5 (distributed protocol section)

**Anchor:** §4.6.3 Consistency Guarantee — immediately after the existing paragraph.

### Diff

```diff
 **Causal consistency** per task chain: if Task A completes before Task B is submitted and B depends on A, then B observes A's outputs. Enforced by dependency tracking and `REMOTE_DEP_NOTIFY` protocol. Cross-chain ordering is eventual.

+> [UPDATED: A5-P1: Remote retries use exponential backoff with jitter, capped at `max_ms = 2000`.]
+> `RetryPolicy { base_ms = 50, max_ms = 2000, jitter = 0.3, max_retries = 5 }`. The n-th retry waits `min(max_ms, base_ms · 2^n) · (1 + U[-jitter, +jitter])`. A8-P5 alert rule surfaces an SLO breach at `n = 4`. Authoritative definition in `modules/distributed.md`.
+>
+> [UPDATED: A5-P3: v1 deterministic coordinator fail-fast; add `coordinator_liveness_timeout_ms`.]
+> v1 is fail-fast: every surviving peer surfaces `CoordinatorLost` within `heartbeat_timeout_ms`; the Python driver observes `DistributedError` with no silent hang. Config knob **`coordinator_liveness_timeout_ms < heartbeat_timeout_ms`**, default **`3 × heartbeat_interval_ms`**. Scope is pinned to the failed Pod only; `cluster_view` generation bumps to the surviving-coordinator list (no cluster-wide fail-closed). v2 decentralization roadmap is recorded as an ADR-005 extension (see A10-P2 / A5-P3).
+>
+> [UPDATED: A7-P5: `distributed_scheduler` links only against `core::ISchedulerLayer`.]
+> The distributed-scheduler implementation depends on `core/`'s `ISchedulerLayer` (plus role interfaces from A7-P2) rather than on `scheduler/`'s `TaskManager`/`WorkerManager`/`ResourceManager`. Shared machinery lifts to `scheduler/core/` abstract base if needed. This upholds ADR-008 and the layered DAG restated in §3.2 of `03-development-view.md` (A7-P1).
```

---

## Change 5 — A1-P6 cold-start budget row (§4.8.2)

**Anchor:** §4.8.2 Cross-Node Task Submission budget table.

### Diff

```diff
 | Remote node: Host → AICore (local path) | < 15 μs | Same as §4.8.1 | |
 | **Margin** | 20 μs | | Absorbs network jitter, TCP fallback |
+| First-use template-miss (cold start, A1-P6) | ≤ 15 μs | One-off `descriptor_template_id` install on the peer's template registry | Applies only on the first `REMOTE_SUBMIT` for a new template; subsequent sends reuse the registered id. |
```

---

## Change 6 — A1-P5 (§4.8.6 SPMD fan-out + §4.8.7 event-loop budgets)

**Anchor:** after §4.8.5 Budget Validation Strategy; before `---` / §4.9 Layer Lifecycle.

### Diff (new sections)

```diff
 ### 4.8.5 Budget Validation Strategy

 - Latency budgets are validated through profiling at Level 2 (phase-level timing, §7.2.1).
 - Per-stage timestamps are captured at stage boundaries using monotonic nanosecond clocks.
 - Budget violations trigger alerts when SLO thresholds are exceeded (§7.3.3, §7.2.8).
 - [ASSUMPTION] Initial budgets are based on hardware specifications and estimates. They will be refined with benchmark data during implementation.

+### 4.8.6 SPMD Fan-out (Chip → N × AICore)
+
+> [UPDATED: A1-P5: Normative SPMD fan-out latency budget.]
+
+**End-to-end target:** ≤ 5 μs (Chip Scheduler decides SPMD dispatch → all N AICore workers begin executing).
+
+| Stage | Budget | Mechanism | Notes |
+|-------|--------|-----------|-------|
+| Batch prep (per-SPMD descriptor projection, `spmd_index`/`spmd_size` write) | ≤ 1 μs | Single pass over the SPMD shard; no allocation on hot path | Reserved `TaskArgs.scalar_args[0..1]` per A3-P9 |
+| Register-bank block write (N workers) | ≤ 2 μs | Coalesced MMIO burst; one write per Core Wrap | Bounded by register-bank fan-out |
+| 108-bit-popcount ACK fan-in | ≤ 2 μs | A1-P9 bitmask ANDs + `__builtin_ctzll`; a5 108-core path uses AND + 2 × 64 ctzll fallback (< 10 ns both paths) | Single ACK event per ready mask |
+
+### 4.8.7 Event-Loop Iteration Stages
+
+> [UPDATED: A1-P5: Per-stage budgets for the Scheduler event loop (§4.1.4).]
+
+| Stage | Budget | Notes |
+|-------|--------|-------|
+| Stage 1 — COLLECT (per registered source) | ≤ 50 ns / source | Poll or queue drain; idle sources short-circuit. |
+| Stage 2 — DISPATCH (per event) | ≤ 100 ns / event | Inline handlers only; deferred events queued. |
+| Stage 3 — PROCESS DEFERRED (per event) | ≤ 200 ns / event | Drains the internal pending queue. |
+| Idle iteration (no events) | ≤ 300 ns | Cost floor when all sources are empty. |

 ---
```

---

## Cross-references updated

- `modules/distributed.md` (A5-P1/A5-P3 primary owners), `modules/transport.md §2.3` (A1-P6 primary owner), `02-logical-view/02-scheduler.md` (A1-P5 co-owner).
- ADR-008 (A7-P5), ADR-016 (Task canonical spelling, A3-P14 companion), ADR-005 extension (A5-P3 v2).
