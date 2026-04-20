# Applied Changes — `docs/pypto-runtime-design/05-physical-view.md`

Run folder: `reviews/2026-04-18-171357/`.
Proposals applied (from `final/final-proposal.md`): **A10-P2**, **A10-P7**, **A10-P9**.

Line-count delta: 245 → 260 (+15).

Each change is a tightly-scoped `> [UPDATED: <id>: <reason>]` callout; no section rewrites.

---

## Change 1 — A10-P7 (sharded TaskManager deployment cue) and A10-P9 (WorkStealing × RETRY_ELSEWHERE gate)

**Anchor:** §5.3.3 Dynamic Scaling — immediately after the existing bullet list.

### Diff

```diff
 ### 5.3.3 Dynamic Scaling

 - Level elision enables runtime adaptation (e.g., disable `"Pod"` level for single-node runs).
 - Partition strategies (ROUND_ROBIN, DATA_LOCALITY, WORK_STEALING) adapt to changing node counts.
 - [ASSUMPTION] Dynamic node addition/removal at runtime is not in scope for the initial design. The registry is frozen at initialization.

+> [UPDATED: A10-P7: Deployment cue for opt-in admission sharding (two-tier TaskManager).]
+> Enable the opt-in sharded `TaskManager` path when `concurrent_submitters × cluster_nodes ≥ 64`. Below this threshold, the default single-threaded fast path (`shards = 1`) remains normative. Shard count defaults: **Host = 8, Device = 4, Chip and below = 1** (see A10-P1). See ADR-019 ("Admission shard default + deployment cue") and `02-logical-view/02-scheduler.md` "Concurrency and locking".
+>
+> [UPDATED: A10-P9: `WorkStealingPartitioner × RETRY_ELSEWHERE` requires a live assignment log; FUNCTIONAL mode preserves it.]
+> This combination is admissible only if an assignment log `(submission_id, task_index) → final_node_id` is written **before** dispatch. Under A9-P6 option (iii) the runtime ships `FUNCTIONAL` simulation mode only, and the log is live (in-memory) — REPLAY is not required. The log is preserved across retries so that `RETRY_ELSEWHERE` can honor per-task idempotency semantics (A5-P4). Authoritative definition in `modules/distributed.md §3.1`.
```

---

## Change 2 — A10-P2 (deployment-mode variants + `cluster_view` bump)

**Anchor:** §5.4.0 Critical Path SPOF Analysis — immediately after the "Summary:" paragraph.

### Diff

```diff
 **Summary:** The primary SPOFs are (1) the host process (inherent to a computation engine — mitigated by external monitoring), (2) the coordinator node in distributed mode (mitigated by application-level restart), and (3) individual network links (mitigated by transport fallback). All hardware-level components (Devices, AICores, AICPU threads) have built-in redundancy through multiple instances.

+> [UPDATED: A10-P2: Deployment-mode variants for coordinator failover; `cluster_view` bumps on coordinator demote.]
+> The v1 coordinator-failure model (fail-fast, absorbed into A5-P3) has **three deployment-mode variants**, selected in `DeploymentConfig`:
+>
+> 1. **`SingleCoordinator` (default v1):** one coordinator per Pod; on failure, surviving peers surface `CoordinatorLost` within `heartbeat_timeout_ms`; Python driver sees `DistributedError`; application-level re-submission.
+> 2. **`StickyCoordinator` (v1 opt-in):** coordinator role pinned to one member of a small sticky-routing set; on failure, **scope is pinned to the failed Pod only** and `cluster_view` generation bumps to the surviving-coordinator list. No cluster-wide fail-closed.
+> 3. **`QuorumCoordinator` (v2 roadmap, not implemented in v1):** decentralized via quorum-promoted `cluster_view` generation; recorded as an ADR-005 extension.
+>
+> On every coordinator demote / promote in modes (2)–(3), `cluster_view.generation` monotonically bumps and is included in the next `HEARTBEAT` and outbound `REMOTE_*` envelopes; peers with an older generation retry with backoff (A5-P1). See `modules/distributed.md` Coordinator Membership.
```

---

## Cross-references updated

- ADR-005 (v2 decentralized coordinator roadmap), ADR-019 (admission shard default + deployment cue).
- `modules/distributed.md` (A10-P2 / A10-P6 / A10-P9 primary owner), `02-logical-view/02-scheduler.md` (A10-P1 / A10-P7).
