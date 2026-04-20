# Aspect A6: Security & Trust Boundaries — Round 3 (Stress Round)

## Metadata

- **Reviewer:** A6
- **Round:** 3 (MANDATORY STRESS ROUND)
- **Target:** `docs/pypto-runtime-design/`
- **Runtime Mode:** yes
- **Timestamp (UTC):** 2026-04-19T00:00:00Z

## 1. Rubric Checks

Re-evaluation given the round-2 amendment set assumed applied. No doc edits landed between rounds, so R2 rubric evidence still stands; the "projected" column shows the post-amendment state if all `agreed` §5 revisions (from round-2 reviews) are mechanically applied.

| # | Check | Result (R2 actual) | Projected (post-R2-amend) | Rule(s) |
|---|-------|--------------------|---------------------------|---------|
| 1 | Every trust boundary identified and threat-modeled | Weak | **Pass** under A6-P1 (+6 rows) composed with A7-P4 (payloads relocated to `distributed/`) and A4-P6 (ADR cross-links). Stress attack §8 confirms no new boundaries exposed by other proposals. | S1 |
| 2 | Least privilege for every component/key/service account | Weak | **Pass** under A6-P10 (capability-scoped sinks) + A6-P11 (gated `register_factory`) + A6-P13 (partitioned admission counter) + A8-P4 amended with `logical_system_id` scoping + A5-P3 coordinator handshake binding to `cluster_view.generation`. Stress §8.2 identifies one residual gap: `IFaultInjector` (A8-P7) must be sim-only (addressed by A8-P7 amendment). | S2 |
| 3 | Inputs validated at trust boundaries | Weak → **Pass** | Composite of A6-P3 (entry-gate length-prefix guard) + A6-P8a absorbed into A1-P11 (per-arg DLPack validation) + A6-P8b (byte-cap) + A3-P7 (Submission preconditions) + A3-P13 (FIFO+idempotent dup-detect). Stress §8.6 confirms the entry-gate is allocation-free on reject. | S3 |
| 4 | Secure defaults (no relaxation without explicit opt-in) | Weak → **Pass** | A6-P4 TCP-default-encrypted + A6-P7 `trust_boundary.multi_tenant=false` default + A2-P3 closed-enum caveat (new `TransportBackend` must declare `require_encrypted_transport`) + A6-P14 key-material ADR. | S4 |
| 5 | Sensitive data encrypted in transit and at rest | Fail → **Pass (control plane) / Documented deviation (RDMA data plane + at-rest)** | Two-tier split in A6-P4: TCP-default-TLS + RDMA explicitly excluded with rkey scoping (A6-P5) + node auth (A6-P2) as the compensating control. At-rest remains an E5 item tracked in `10-known-deviations.md` Deviation 3 (no reviewer proposed changing this). | S5 |
| 6 | Tamper-evident audit trail | Fail → **Pass** | A6-P6 + A8-P10 (structured KV is the substrate) + folds for A5-P2 (circuit-breaker state), A5-P3/A10-P2 (coordinator failover), A5-P6 (watchdog), A6-P7 (attestation decisions), A8-P9 (profiling-drop alerts → audit). | S6 |

Net rubric delta vs R2: checks 3–6 move from Weak/Fail to Pass once the §5 amendments are applied. Checks 1, 2 remain Pass-conditional on the stress-round amendments called out below (A8-P4 `dump_state()` tenant scoping, A5-P3/A10-P2 handshake-generation binding).

## 2. Pros

Unchanged from R2. Consolidated stress-round observations:

- **A6-P3 + A2-P1 + A7-P4 interlock** — once distributed payloads live in `distributed/protocol_payloads.h`, every ingress path lands on a single place where (length-prefix guard + schema version + tenant-id + auth) all hold simultaneously. No path bypasses the gate. [S1, S3, `modules/distributed.md`, `modules/transport.md:76-98`]
- **A8-P10 structured-KV substrate gives audit events the same cost model as logs** — no new code path on the hot side; cold-path audit emitters reuse the pre-allocated sink ring. [S6, `modules/profiling.md §2`]
- **A5-P10 per-REMOTE_* idempotency composes with A3-P13 FIFO-dup-detection** — the per-handler idempotency class declared under A2-P6's pluggable registry means "retry under attack" reasoning has a one-sentence answer per wire message. [DS4, R2, `modules/distributed.md §5.3`]
- **A10-P5 per-peer REMOTE_SUBMIT projection (absorbed into A1-P6)** gives S2 least-privilege "information per peer" as a natural side-effect of the perf optimization — a compromised peer only sees its own slice of the `producer_index`. [S2, P6]

## 3. Cons

Two residual issues surface under stress-round replay of the scenarios in §8:

- **A8-P4 `Runtime::dump_state()` default scope is still under-specified in the amended text** — the R2 §5 amendment from A8 pairs with A5-P6 watchdog but does not explicitly state that the default scope is caller-`logical_system_id`. Stress §8.5 flags this and proposes the exact amendment text. [S2, `modules/runtime.md`]
- **A5-P3 / A10-P2 coordinator-failover amendment does not carry `cluster_view.generation` binding into `HandshakePayload` yet** — A5 and A10 R2 amendments describe the v1 fail-fast decision cleanly, but A6-P2's handshake text must be amended in parallel to reject messages whose `coordinator_generation < cluster_view.current_generation - k`. Stress §8.4 shows this is required to close scenario 6.2.2 under a malicious former-coordinator attacker. [S1, S2, `modules/transport.md §5.3`]

Both are **amendments to adjacent proposals, not new A6 proposals**, consistent with the round-3 rule "revise your own proposals only if peer stress breaks them" (§7 propagations below).

## 4. Proposals (NEW this round)

None. All A6-side concerns surface as stress-attack amendments or §7 cross-aspect propagations.

## 5. Revisions of own proposals

| id | action | amended_summary (if amend/split) | reason |
|----|--------|----------------------------------|--------|
| A6-P1 | defend | — | Stress §8.1 confirms no new unnamed boundary emerges from R2 amendments; R2 text stands. |
| A6-P2 | **amend** | Add `coordinator_generation: uint64_t` to `HandshakePayload` and an inbound-check rule: **"messages whose `coordinator_generation < cluster_view.current_generation - k_reorder_tolerance` (default `k=1`) are rejected with `ErrorCode::StaleCoordinatorClaim` and emit an audit event `coordinator_claim_rejected`."** Applies to HANDSHAKE and to all control-plane messages that carry the coordinator claim. | Stress §8.4 replay of 6.2.2 with a malicious demoted-coordinator shows that without this binding, a former coordinator with still-live credentials can silently resume issuing REMOTE_SUBMIT after Node₁ failover. Closes the gap identified in R2 §7 that A5-P3/A10-P2 opened. |
| A6-P3 | defend | — | Stress §8.6 confirms the single length-prefix guard at `MessageHeader` entry holds under 6.1.2 replay with an attacker-controlled peer sending `task_count = 2^31` and oversized trailing arrays. No allocation before guard → no DoS amplification. |
| A6-P4 | defend | — | Stress §8.3 confirms the TCP-only scope + RDMA explicit exemption holds under 6.1.2 replay. RDMA data-plane security still rides on A6-P5 rkey + A6-P2 auth (now generation-bound). |
| A6-P5 | defend | — | Stress §8.8 confirms per-Submission rkey retirement holds under a captured-`REMOTE_DATA_READY` replay scenario after `SUBMISSION_RETIRED`. |
| A6-P6 | defend | — | Stress §8.9 confirms the audit ring is allocation-free on hot paths (security events emit only at cold sites). A9 YAGNI preempted by in-memory-ring-only v1 surface. |
| A6-P7 | defend | — | Stress §8.10 confirms multi-tenant gate closes the unsigned-blob injection path in 6.1.2 when Node₁ is attacker-controlled; single-tenant SIM path unaffected. |
| A6-P8 | defend (split from R2) | — | A6-P8a absorbed into A1-P11 confirmed stable; A6-P8b `max_import_bytes` defends 6.1.2 step 5 (DLPack capsule from attacker process on Node₁). |
| A6-P9 | defend | — | Stress §8.7 confirms `logical_system_id` field in `MessageHeader` defeats cross-tenant forged-inbound scenario (attacker peer claims tenant-B when Node₀ runs tenant-A layer). |
| A6-P10 | defend | — | Stress §8.12 confirms sink filtering defeats `dump_state`-adjacent info leak when paired with A8-P4 amendment. |
| A6-P11 | defend | — | Stress §8.11 confirms init-only gate + registration-token composition with A2-P6's pluggable registry under adversarial-plug-in scenario. |
| A6-P12 | **amend** | Reframe for option-(iii) landing zone of A9-P6: **"Ship the REPLAY enum + trace-schema scaffolding in v1 (frozen format version `trace_schema_version=1`, `content_hash`, detached-signature field defined). The REPLAY execution engine itself is deferred to v2 per A9-P6. In v1, the signed-schema format IS the forensic anchor — any captured trace that lands on disk is immediately verifiable even before the REPLAY engine ships."** | Synthesizer recommendation for A9-P6 option (iii). Decouples A6-P12 from REPLAY engine availability; the schema binding (A2-P9's `trace_schema_version`) + signature + hash are the forensic-load-bearing parts. This satisfies the S6 forensic pathway without forcing A9 to ship a full REPLAY engine in v1. |
| A6-P13 | defend | — | Stress §8.13 confirms fold into existing admission counter keeps HPI=none; per-tenant flood in 6.1.2 is rejected via the existing `AdmissionDecision::REJECTED` channel. |
| A6-P14 | defend | — | Doc-only ADR; no stress-round impact. |

## 6. Votes on peer proposals (disputed items only — R3 mandatory re-vote)

For the three items that remained `disputed` after round 2, using the **amended text from round-2 §5 of each owner**.

| proposal_id | vote | rationale (cite rule id) | blocking | override_request |
|-------------|------|--------------------------|----------|-------------------|
| A2-P7 | **agree** | Amended to Q-record in `09-open-questions.md` (synthesis landing zone). No interface added in v1 → no security-boundary exposure from a speculative plugin hook. G2/S4 both satisfied. | false | false |
| A9-P2 | **agree** | Amended text keeps single `IEventLoopDriver` test seam + closed `DeploymentMode`/`EventCollectionPolicy` enums + appendix of v2 extensions. Closing the policy enums actually *reduces* A6-P11's gating surface (no post-init factory registration for policies). No S-rule concern. | false | false |
| A9-P6 | **agree** under synthesis option **(iii)** | Option (iii): "ship FUNCTIONAL-only implementation but keep REPLAY enum + scaffolding so A6-P12 and A8 debugging harness can land on day 2." My A6-P12 amended (§5 above) pivots to "signed schema + format frozen at ADR-011-R2 time," which IS the v1 forensic anchor. The REPLAY execution engine can ship in v2; the schema binding does not need the engine to be load-bearing for S6. This flips my R2 disagree to agree because my S6-adjacent concern is met by the schema+signature scaffolding, not by the engine availability. | false | false |

All three previously disputed proposals converge to `agree` under the amended forms. No new overrides requested.

## 7. Cross-aspect tensions (stress-round propagations)

| pair | with_proposal | preferred_resolution |
|------|---------------|----------------------|
| A6 vs A5 & A10 | A5-P3 / A10-P2 (v1 fail-fast coordinator) | **My A6-P2 amendment** (§5) adds `coordinator_generation` to `HandshakePayload` and a generation-window check on inbound control messages. A5 and A10 do not need to re-amend — their proposals only decide the failover *policy*, while my amendment binds identity to the policy's authoritative view. |
| A6 vs A8 | A8-P4 (`dump_state()`) | **A8 must amend**: default output is filtered by the caller's `logical_system_id`; unscoped dump requires `DeploymentConfig.diagnostic_admin_token`. Without this, S2 breaks under scenario 6.1.2 step 11 (Python side) when the runtime is multi-tenant. **Not blocking on the hard-rule axis** (no S-rule is literally violated as long as multi-tenant gate stays false-by-default per A6-P7), but blocking in the operational sense — must be applied before multi-tenant deployment. Severity: medium. |
| A6 vs A8 | A8-P5 (alert rules) | Propagation of the same pattern: `AlertRule` carries `logical_system_id`. A8's R2 amendment mentions capability scoping but does not name the field. Add `logical_system_id: Optional[str]` to the YAML schema. |
| A6 vs A8 | A8-P7 (`IFaultInjector`) | Stress §8.14 confirms A8-P7's R2 amendment explicitly gates the fault injector to "sim-only deployment." If ever ported to onboard, it becomes an S2 surface. The amendment's sim-only wording is tight; no further change needed unless onboard port is proposed. |
| A6 vs A2 | A2-P3 (open `TransportBackend`) | Reaffirm R2 condition: every registered backend must declare `require_encrypted_transport` and cannot lower the cluster default without explicit `allow_insecure_transport=true`. Confirmed in A2's R2 amendment; no further change. |
| A6 vs A9 | A9-P6 (defer REPLAY) | **Resolved** via option (iii) landing. See A6-P12 amendment §5 and vote §6. |
| A6 vs A1 | all HPI-flagged own proposals | Stress §8.6, §8.7, §8.13 confirm no HPI regression on A6-P3, A6-P9, A6-P13. A1 had already voted agree; no change. |

## 8. Stress-attack on emerging consensus

Scenarios used:

- **S1 = 6.1.2 (Distributed Multi-Node Execution)** replayed with **attacker-controlled Node₁** — `06-scenario-view.md:35-58`. Attacker owns Node₁'s credentials (or has live-captured them) and may deviate arbitrarily from the protocol: forged `REMOTE_SUBMIT`, oversized trailing arrays, cross-tenant `logical_system_id`, replayed `REMOTE_DATA_READY`, unsigned function blob injection, unsigned DLPack import from the attacker process, QPS flood, handshake downgrade.
- **S2 = 6.2.2 (Remote Node Crash)** replayed with **malicious partial state** — `06-scenario-view.md:100-114`. Node₁ "crashes" but the attacker retains its credentials and, post-`CONTINUE_REDUCED` / `RETRY_ALTERNATE`, attempts to re-enter as a ghost coordinator or as a demoted-coordinator re-issuing REMOTE_SUBMIT after the failover has promoted Node₀.

Verdict key: `holds` | `breaks` | `uncertain`. Every row cites the scenario line range.

### 8.1 A6-P1 — STRIDE trust-boundary completeness

| proposal_id | stress_attempt | verdict |
|-------------|----------------|---------|
| A6-P1 | S1 step 4 (`REMOTE_SUBMIT` at `06-scenario-view.md:48`): is every new attack point introduced by peer R2 amendments enumerated in §7.1 after my P1 amendment lands? Cross-check: A2-P6 adds pluggable `IDistributedProtocolHandler`, A7-P4 moves payloads into `distributed/`, A8-P4 adds `dump_state()`, A5-P3 adds coordinator failover. Each surfaces a trust boundary or expands an existing one. | **holds** with a note: add a row for `Runtime::dump_state()` and for `IDistributedProtocolHandler` registration, both covered by existing "Plug-in Factory" and "Python↔C" rows after A7-P4 / A6-P11 gate. No new untracked boundary. |

### 8.2 A6-P2 — concrete node authentication

| proposal_id | stress_attempt | verdict |
|-------------|----------------|---------|
| A6-P2 | S2 replay (`06-scenario-view.md:100-114`): Node₁ is "crashed" but attacker retains Node₁'s mTLS private key. Cluster enters `CONTINUE_REDUCED` / `RETRY_ALTERNATE`. Attacker, still holding valid cert, re-connects to Node₀ and issues `REMOTE_SUBMIT`. **Under R2 A6-P2 text alone**, Node₀'s handshake accepts the cert (it is valid), and the attacker silently resumes. | **breaks** without generation binding. Amendment (§5): add `coordinator_generation: uint64_t` to `HandshakePayload`; messages with `coordinator_generation < cluster_view.current_generation - k` rejected with `StaleCoordinatorClaim` and an audit event. After amendment → **holds**. |

### 8.3 A6-P4 — secure-default TLS (TCP control plane only)

| proposal_id | stress_attempt | verdict |
|-------------|----------------|---------|
| A6-P4 | S1 step 4 (`06-scenario-view.md:48`) with plaintext TCP downgrade attempt by attacker-controlled Node₁: attacker responds to TLS `ClientHello` with protocol-version downgrade. **Under R2 amended text** (`require_encrypted_transport: bool = true` default, fail-closed multi-host plaintext) → handshake aborts at `init` on Node₀ if TLS is not negotiable, or at `connect` if downgrade attempted. | **holds**. A6-P4 + standard TLS downgrade resistance closes it. RDMA data-plane explicitly excluded from TLS (per amendment), compensated by A6-P5 rkey scoping + A6-P2 generation-bound auth. |

### 8.4 A6-P2 under 6.2.2 — coordinator-failover attack

| proposal_id | stress_attempt | verdict |
|-------------|----------------|---------|
| A6-P2 + A5-P3 + A10-P2 | S2 (`06-scenario-view.md:100-114`): Node₁ lost → Node₀ promotes itself to coordinator (generation bumped). Attacker now *from Node₁'s IP* replays previously-captured HANDSHAKE frames trying to convince Node₂ (if any) that Node₁ is still coordinator. | **breaks** under R2 without my §5 amendment. **holds** after my amendment: the replayed HANDSHAKE carries an old `coordinator_generation` < current; rejected with `StaleCoordinatorClaim`. Combined with A6-P6 audit event, the forensic trail is intact. |

### 8.5 A8-P4 — `Runtime::dump_state()` cross-tenant leak

| proposal_id | stress_attempt | verdict |
|-------------|----------------|---------|
| A8-P4 | S1 step 11 equivalent on a multi-tenant deployment (`06-scenario-view.md:55`, adapted): tenant-A Python caller invokes `runtime.dump_state()`. Under R2 amended text, the dump is intended to aid operator debug, but the amendment does not explicitly restrict cross-tenant visibility. Tenant-A sees tenant-B state. | **breaks** S2 unless A8 amends. **Proposed amendment to A8-P4 (cross-aspect propagation)**: default `dump_state()` is scoped to caller's `logical_system_id`; unscoped dump requires `DeploymentConfig.diagnostic_admin_token`. After amendment → **holds**. Flagged in §7; A6 does not set `blocking=true` because A6-P7 multi-tenant gate defaults to single-tenant, so the leak only activates under an explicit opt-in, but it is still a required fix before multi-tenant deployment. |

### 8.6 A6-P3 — bounded payload parsing

| proposal_id | stress_attempt | verdict |
|-------------|----------------|---------|
| A6-P3 | S1 step 4 (`06-scenario-view.md:48`): attacker-controlled Node₁ sends `REMOTE_SUBMIT` with `task_count = 2^31 − 1`, `edge_count = 2^31 − 1`, and a trailing `args_blob` advertised as 16 MiB. Under the R2 amendment (single length-prefix guard at `MessageHeader` entry, derived from `max_payload_bytes_by_type[REMOTE_SUBMIT]`), the frame is rejected *before* `message_factory` allocates any buffer. Receive path remains allocation-free on reject branch. | **holds**. ≤1 integer compare on the entry branch; no heap activity. A1's HPI="extends(≤1 compare)" label accurate. |

### 8.7 A6-P9 — logical-system isolation in wire + code

| proposal_id | stress_attempt | verdict |
|-------------|----------------|---------|
| A6-P9 | S1 step 4–5 (`06-scenario-view.md:48-50`): attacker-controlled Node₁ forges `REMOTE_SUBMIT` with `logical_system_id=tenant-B` while the receiving layer is serving tenant-A only. Under R2 amended text, the protocol handler at `distributed/` (post-A7-P4 move) checks the header field and drops with `CrossTenantDenied`; metric `cross_tenant_rejected` increments; audit event emitted. | **holds**. Composition with A7-P4 (payloads co-located with validation) is exactly what A6-P9 needs to remain a one-stop gate. |

### 8.8 A6-P5 — rkey scope + revocation

| proposal_id | stress_attempt | verdict |
|-------------|----------------|---------|
| A6-P5 | S1 step 7 (`06-scenario-view.md:51`) with replay: attacker-controlled Node₁ captures a legitimate `REMOTE_DATA_READY{global_address, rkey}` from Node₀. Submission retires on Node₀ (`04-process-view.md:491`). Attacker replays the captured `REMOTE_DATA_READY` and attempts RDMA READ 100 ms after retirement. Under R2 amended text, `IMemoryOps::deregister_peer_read` is called at `SUBMISSION_RETIRED`, so the rkey is invalid; RDMA READ fails with `TransportDisconnected` / `RkeyRevoked`. | **holds**. Blast radius bounded to one Submission window; per-Submission retirement avoids per-Task FSM churn (A9's concern from R1). |

### 8.9 A6-P6 — audit trail

| proposal_id | stress_attempt | verdict |
|-------------|----------------|---------|
| A6-P6 | S1 replay — enumerate every security-relevant site in 6.1.2 steps 1–11 and verify each emits exactly one audit event: node_join (step 4→5 handshake in steady state), cross-tenant drop (8.7), quota breach (8.13), attestation rejection (8.10), coordinator_claim_rejected (8.4). Compose with A8-P10 (KV logging substrate). | **holds**. A9-YAGNI preempted: v1 ships only `IAuditSink` interface + pre-allocated in-memory ring; file/syslog sinks are OCP extensions. No hot-path allocation. |

### 8.10 A6-P7 — function-binary attestation (multi-tenant gated)

| proposal_id | stress_attempt | verdict |
|-------------|----------------|---------|
| A6-P7 | S1 step 4/5: attacker-controlled Node₁ registers an unsigned AICore binary via cross-node `REMOTE_SUBMIT` referencing a new `FunctionId`. Under R2 amended text with `DeploymentConfig.trust_boundary.multi_tenant=true`, `register_function` returns `FunctionNotAttested` on Node₀; single-tenant default (`multi_tenant=false`) allows unsigned, as intended. | **holds** under multi-tenant deployment. Single-tenant SIM unaffected — A9 YAGNI concern neutralized. |

### 8.11 A6-P11 — gated `register_factory` under A2-P6 pluggable registry

| proposal_id | stress_attempt | verdict |
|-------------|----------------|---------|
| A6-P11 + A2-P6 | S1 step 5 (`06-scenario-view.md:49`, adapted): attacker-controlled Node₁, after deserialization, attempts a Python-side `runtime.register_factory(malicious_protocol_handler)` post-`freeze()`. Under R2 amendments, `RegistrationClosed` returned; attacker with wrong token pre-freeze gets `Unauthorized`. A2-P6's registry is init-only, so no post-init insertion path exists. | **holds**. A6-P11 gate + A2-P6 init-only semantic + audit event `factory_register_rejected` = closed surface. |

### 8.12 A6-P10 — capability-scoped sinks under A8-P4 dump

| proposal_id | stress_attempt | verdict |
|-------------|----------------|---------|
| A6-P10 + A8-P4 (post-amend) | S1 step 11 adapted (`06-scenario-view.md:55`): tenant-A registers a log sink and invokes `dump_state()`. Under A8-P4 (post-A6 amendment §8.5) + A6-P10 the sink filter drops any event whose `logical_system_id ≠ A`; dump output is pre-filtered. | **holds** once A8-P4 adopts the §8.5 amendment. Without that amendment → breaks. Severity: medium. |

### 8.13 A6-P13 — per-tenant submit rate-limit

| proposal_id | stress_attempt | verdict |
|-------------|----------------|---------|
| A6-P13 | S1 step 1 (`06-scenario-view.md:45`): attacker-controlled tenant-A Python floods `runtime.submit()` at 10× configured QPS. Under R2 amended text (fold into existing admission counter partitioned by `logical_system_id`), fast path = partitioned counter increment (≤1 ns relaxed atomic); breach surfaces via existing `AdmissionDecision::REJECTED` channel with `TenantQuotaExceeded`. Tenant-B unaffected (separate partition). | **holds**. HPI=none confirmed (no new branch on success path). A1's admission perf discipline intact. |

### 8.14 A8-P7 — `IFaultInjector` sim-only

| proposal_id | stress_attempt | verdict |
|-------------|----------------|---------|
| A8-P7 | S1 replay: if `IFaultInjector` were installed onboard, an attacker with Python code execution could inject faults into peer state. Under R2 amendment, `IFaultInjector` is explicitly "sim-only" and rejected at `init` when running on real hardware. | **holds**. A8's amendment language is adequate. |

### 8.15 A10-P1 — sharded `producer_index` with tenant-aware key

| proposal_id | stress_attempt | verdict |
|-------------|----------------|---------|
| A10-P1 | S1 step 2 (`06-scenario-view.md:46`): two tenants contend on the sharded index. R2 amendment adopts tenant-in-shard-key; cross-tenant hash collision thus impossible. | **holds**. Co-benefit for A6-P9 (tenant-id on the wire) — the same tenant key is used at two layers consistently. |

### 8.16 A5-P2 — per-peer circuit breaker audit hook

| proposal_id | stress_attempt | verdict |
|-------------|----------------|---------|
| A5-P2 + A6-P6 | S2 (`06-scenario-view.md:106-108`): Node₁ heartbeat loss triggers circuit breaker. Verify: audit event `circuit_open` emitted with `peer_id`, prev hash, sequence no. | **holds** provided A5-P2's `CircuitBreakerState` transitions are wired into the audit emit list. (A5's R2 amendment already co-owns this with A6-P6.) |

### 8.17 A2-P1 / A2-P9 — schema version attachment for signed REPLAY

| proposal_id | stress_attempt | verdict |
|-------------|----------------|---------|
| A2-P1 + A2-P9 + A6-P12 (amended) | S1/S2 adapted: an attacker submits a REPLAY trace purporting to be `trace_schema_version=1` but with a mismatched `content_hash` and a forged signature. Under A6-P12 amended + A2-P9 version field, the verifier rejects `InvalidReplayTrace` on load; even if REPLAY engine is deferred to v2 (A9-P6 option iii), the forensic verification still runs. | **holds**. Decoupling the schema validator from the engine (my A6-P12 amendment) is exactly what closes this. |

### 8.18 A3-P7 + A6-P3 + A1-P11 — end-to-end input validation

| proposal_id | stress_attempt | verdict |
|-------------|----------------|---------|
| A3-P7 + A6-P3 + A1-P11 | S1 step 1 → step 5 (`06-scenario-view.md:45-50`) end-to-end: attacker-controlled tenant submits a Submission whose DLPack arg has a negative stride (caught by A1-P11/A6-P8a); the Submission also has a cyclic `intra_edges` (caught by A3-P8); wire-level retransmit from attacker peer tries oversized `edge_count` (caught by A6-P3). Three layers, three different rejections. | **holds**. No gap between the three gates; each has its own error-code surface (ValueError at bindings, PreconditionFailed at scheduler, TransportCorrupted at wire). Audit events fire at each rejection. |

### 8.19 A7-P1 — cycle removal clarifies trust flow

| proposal_id | stress_attempt | verdict |
|-------------|----------------|---------|
| A7-P1 | S1 replay: verify that after cycle break, `distributed/` → `scheduler/` is one-way. Any inbound wire data lands on `distributed/` and crosses into `scheduler/` only via the narrowed `ISchedulerLayer` interface (A7-P5). | **holds**. D6 hard-rule satisfied; S2 trust flow tractable. A6 hard-rule citation on A7-P1 (from R2) stays as "in favor". |

### 8.20 A3-P13 — FIFO + idempotent + dup-detect

| proposal_id | stress_attempt | verdict |
|-------------|----------------|---------|
| A3-P13 + A5-P10 | S1 step 7 (`06-scenario-view.md:51`): attacker-controlled Node₁ replays `REMOTE_DATA_READY` multiple times. Under bounded-reorder-window + per-handler idempotency class (A5-P10), duplicate detection drops replays; the rkey scoping (A6-P5) already made replay useless post-retirement. Defense in depth. | **holds**. Redundant but cheap — A1 HPI=none confirmed. |

### 8.21 A5-P5 — chaos/fault-injection matrix covers A6 paths

| proposal_id | stress_attempt | verdict |
|-------------|----------------|---------|
| A5-P5 + A8-P7 | Verify the chaos matrix includes at least: handshake-downgrade, cert-rotation-mid-connection, rkey-revocation-race, quota-partition-skew, audit-sink-degraded. R2 amended text defers the concrete matrix to a `tests/chaos/` folder but enumerates the categories. | **holds** provided the five A6-side categories above land in the matrix. I will verify in R4 via a simple grep once `tests/chaos/matrix.md` exists. **uncertain** until that file exists; not blocking. |

### 8.22 A6-P8 / A1-P11 boundary for DLPack from attacker DLPack producer

| proposal_id | stress_attempt | verdict |
|-------------|----------------|---------|
| A1-P11 (absorbing A6-P8a) + A6-P8b | S1 step 1 (`06-scenario-view.md:45`): attacker-controlled Python process produces a DLPack capsule with overflowing stride × shape product. Under R2 merged text: A1-P11 per-arg budget (≤200 ns) includes the structural checks from A6-P8a; A6-P8b `max_import_bytes` deployment cap catches attack sizes > cap. | **holds**. Merge preserves both properties; fast-path budget unaffected. |

### 8.23 Disputed proposals under stress

| proposal_id | stress_attempt | verdict | vote under amended text |
|-------------|----------------|---------|--------------------------|
| A2-P7 (amended → Q-record) | Security surface of "reserved async-policy extension seam": if no interface ships in v1 (Q-record only), attack surface unchanged from today. No new plugin registration path to gate. | **holds** | agree |
| A9-P2 (amended → single test seam) | S1 replay: only seam is `IEventLoopDriver` (test-only). Policy enums are closed in v1 → one fewer factory-registration site to gate via A6-P11. Appendix-listed v2 extensions are not wired, so no attack surface. | **holds** | agree |
| A9-P6 (amended → option (iii), defer PERFORMANCE/REPLAY engines but keep schema) | S2 replay: post-incident forensics in multi-tenant deployment after a suspected breach. Under option (iii), the captured trace (if any) carries `trace_schema_version`, `content_hash`, and detached signature per A6-P12 amended; verifier runs without the REPLAY engine. Forensic pathway intact in v1. | **holds** | agree |

### 8.24 Summary table

| category | # attacks | holds | breaks | uncertain |
|----------|-----------|-------|--------|-----------|
| A6-own proposals under S1/S2 | 11 (A6-P1/2/3/4/5/6/7/9/10/11/12/13) | 11 after amendments §5 | 0 | 0 |
| A6 vs peer (A5/A10 coordinator, A8 dump, A8 alert, A8 fault-inj) | 6 | 4 | 0 | 2 pending amendments (A8-P4 dump scope, A5-P5 chaos matrix) |
| Cross-cutting (A2-P1+9 schema, A3-P7+13, A7-P1, A10-P1, A1-P11) | 7 | 7 | 0 | 0 |
| Disputed-after-R2 (A2-P7, A9-P2, A9-P6) | 3 | 3 | 0 | 0 |
| **Total** | **27** | **25** | **0** | **2 (amendments pending adjacent proposals)** |

No S-rule-grounded blocking objections raised. Two amendments to adjacent proposals are required and documented above.

## 9. Status

- **Satisfied with current design?** **yes**, conditional on:
  - **A6-P2 §5 amendment** — `coordinator_generation` in `HandshakePayload` + generation-window reject rule. This is a revision of my own proposal; no peer amendment required.
  - **A8-P4 cross-aspect amendment** — `dump_state()` defaults to caller's `logical_system_id`; unscoped dump requires `diagnostic_admin_token`. Flagged to A8 (§7, §8.5). Not blocking (multi-tenant gate defaults off per A6-P7), but operationally required before multi-tenant deployment.
  - **A5-P5 / A8-P7 chaos matrix** — must include the five A6 categories listed in §8.21. Currently uncertain; will re-verify once matrix file exists. Not blocking.
- **Vote changes from R2:**
  - A9-P6: **disagree → agree** under option (iii) + A6-P12 amendment.
  - A2-P7: **abstain → agree** (Q-record framing).
  - A9-P2: **abstain → agree** (single test seam confirmed).
- **Open items expected in next round (if R4 runs):**
  - Verify A8-P4 adopts the `logical_system_id` default scoping (my §7/§8.5 propagation).
  - Verify `A5-P5 / A8-P7` chaos matrix enumerates the five A6 categories.
  - Verify A6-P2 §5 amendment lands in `modules/transport.md §5.3` and `distributed.md §2.4`.
- **Blocking objections raised this round:** 0.
- **Override requests raised this round:** 0.
- **Hard-rule citations in favor:** D6 on A7-P1 (reinforced from R2); S3 on A6-P3 entry-gate; S2 on A6-P2 generation-binding; S6 on A6-P6 audit-chain.

No new disputes introduced. Convergence criteria (per SKILL.md §Phase 4): 3 previously disputed proposals flipped to agree; zero new disputes; two minor amendments documented. A6 is ready for parent to mark **converged**.

WROTE /data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/reviews/2026-04-18-171357/round-3/reviews/A6-security.md
