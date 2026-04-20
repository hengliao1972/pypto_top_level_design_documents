# Aspect A6: Security & Trust Boundaries — Round 2

## Metadata

- **Reviewer:** A6
- **Round:** 2
- **Target:** `docs/pypto-runtime-design/`
- **Runtime Mode:** yes
- **Timestamp (UTC):** 2026-04-19T00:00:00Z

## 1. Rubric Checks

Re-checks unchanged from round 1 (target docs were not edited between rounds). Peer evidence in R1 shifts three findings:

| # | Check | Result (R2) | Δ vs R1 | Rule(s) |
|---|-------|-------------|---------|---------|
| 1 | Every trust boundary identified and threat-modeled | Weak | unchanged; A4-P6 (ADR back-refs) will help once applied | S1 |
| 2 | Least privilege for every component/key/service account | Weak | reinforced by A7-P4 (payloads move to `distributed/` — trust surface clarified), A7-P1 (remove `scheduler/`↔`distributed/` cycle — trust flow becomes tractable) | S2 |
| 3 | Inputs validated at trust boundaries | Weak → Weak+ | strengthened structurally by A3-P7 (Submission preconditions), A3-P13 (ordering `[ASSUMPTION]`s), A2-P1 (schema versions at boundary only); still blocked on my A6-P3 for wire-bounds | S3 |
| 4 | Secure defaults | Weak | unchanged; depends on A6-P4 | S4 |
| 5 | Sensitive data encrypted in transit and at rest | Fail | unchanged; depends on A6-P4, A6-P14 | S5 |
| 6 | Tamper-evident audit trail | Fail | unchanged; A5-P2 (circuit-breaker state transitions) and A5-P3 (coordinator failover) add new audit-emit sites I must fold into A6-P6 | S6 |

## 2. Pros

Same set as R1. Two additions from R1 peer reading:

- **Peer A2-P1 (schema-version on every contract, validated only at trust-boundary handshakes) composes cleanly with my A6-P3 (bounded payload parsing).** Together they give a one-shot "is this input well-formed, well-versioned, and bounded?" gate at every ingress. [S1, S3, E6, `modules/transport.md:87-98,152`]
- **Peer A7-P4 (relocate distributed payload structs from `transport/` to `distributed/`) sharpens the trust boundary A6-P1 enumerates.** With the move, `transport/` becomes a dumb byte-pipe and `distributed/` is the single place that deserializes untrusted-peer content — a clean place to attach S3 validation. [S1/S3, `modules/transport.md:102-150`, `modules/distributed.md`]

## 3. Cons

Same as R1. Two peer-surfaced additions:

- **A8-P4 `Runtime::dump_state()` leaks cross-tenant information by default.** The diagnostic endpoint must be capability-scoped (at minimum per-`logical_system_id`) or the S2 least-privilege guarantee is lost the moment this lands. Resolved via A6-P10 (capability-scoped sinks) extended to diagnostic endpoints; see Tensions §7.
- **A5-P3 / A10-P2 coordinator failover reshapes the auth surface.** If the coordinator becomes a movable role, node authentication (A6-P2) must validate both *peer identity* and *current coordinator claim*. Otherwise a former coordinator can silently resume issuing REMOTE_SUBMIT after demotion. See §7 tension with A5 / A10.

## 4. Proposals (new this round)

None. All new concerns surface in the votes below, tensions §7, or merges §Merge Register.

## 5. Revisions of own proposals

| id | action | amended_summary (if amend/split) | reason |
|----|--------|----------------------------------|--------|
| A6-P1 | defend | — | No peer disagreement; doc-only, `hot_path_impact: none`; enumerated boundaries are still missing (`07-cross-cutting-concerns.md:11-34`). Cited rule S1 is non-negotiable at boundary enumeration level. |
| A6-P2 | defend | — | No peer raised a blocking objection. A1 does not veto (handshake-time only). A2-P1 and A2-P5 converge with this (schema versioning + BC policy pin credential_id stability). Primitive choice remains mTLS cert pinned in `DeploymentConfig.peers[].public_key`; alternative SPIFFE tagged as v2 extension via A6-P14 ADR. |
| A6-P3 | amend | Collapse per-field bound checks into a **single length-prefix guard** at `MessageHeader` receive entry: reject frames whose advertised payload length exceeds `max_payload_bytes_by_type[MessageType]` **before** `message_factory` allocates any buffer (so the receive path remains allocation-free on the reject branch too). Per-field maxima (`max_task_count`, `max_edge_count`, `max_remote_error_message_bytes`, `max_tensor_meta_bytes`) are derived from the per-type ceiling, not checked independently. | Synthesizer R1 §Open-Disputes "A6-P3: cap check at boundary entry only"; A1's P8/P1 discipline (no receive-path allocation on reject). Keeps `hot_path_impact` at `extends(≤ 1 compare/frame)` for reject, `none` for accept, matching A1's receive-path posture. |
| A6-P4 | amend | Split the guardrail by channel: (i) TCP/control-plane: `require_encrypted_transport: bool = true` default, fail-closed at `init` on multi-host plaintext; (ii) RDMA data-plane: **TLS explicitly NOT applied** — security relies on A6-P5 `rkey` scoping + A6-P2 node authentication at connection setup. Document the two-tier story in `07-cross-cutting-concerns.md §7.1.3` and `10-known-deviations.md` Deviation 3. | Synthesizer R1 §Open-Disputes "A6-P4: RDMA path unaffected"; A1's hot-path concern was about RDMA latency, not control-plane TCP. Two-tier path satisfies X9 while closing the S4/S5 hole on the control plane. |
| A6-P5 | amend | Rotation cadence pinned to **per-Submission retirement** (not per-Task): the `rkey` outlives all Tasks inside one Submission but is revoked in `IMemoryOps::deregister_peer_read` called from `distributed_scheduler` on `SUBMISSION_RETIRED` (`04-process-view.md:491`, already documented as slow path). Steady-state rotation rate ≤ `max_outstanding_submissions / mean_submission_duration` per peer, ≪ control-plane budget. | Synthesizer R1 §Open-Disputes "A6-P5: rotation on Submission retirement, not per task"; A9 concern about lifecycle complexity mitigated because retirement is already an existing event. No new FSM. |
| A6-P6 | defend | — | No peer blocking objection. A8-P10 (structured KV logging) and A8-P5 (alert externalization) compose with A6-P6: the audit sink reuses the KV logging primary surface plus an `IAuditSink` role. A9 pushback anticipated; reply: ship only the *interface* + the in-memory ring in v1; file/syslog audit sinks are OCP extensions (E4) that ship alongside A8-P10. Zero hot-path allocation (pre-allocated audit ring). |
| A6-P7 | amend | Gate enforcement behind `DeploymentConfig.trust_boundary.multi_tenant: bool = false` (single-tenant / SIM / dev: `allow_unsigned_functions = true` by default; multi-tenant: **false** by default). `FunctionDesc` carries an optional `Attestation`; `register_function` rejects unsigned blobs only when `multi_tenant = true`. v1 ships the field, verifier, and a single signer; rotation/SPIFFE deferred to v2 via A6-P14 ADR. | Synthesizer R1 §Open-Disputes "A6-P7: gate behind `trust_boundary.multi_tenant=true`"; A9 YAGNI pushback neutralized: single-tenant users pay zero cost. A1: registration-time only, no hot-path impact. |
| A6-P8 | split | (a) **A6-P8a** — per-arg structural validation (dtype allowlist, device-type match, stride/shape overflow, capsule ownership). Fused into A1-P11 per-arg budget: each DLPack/CAI import ≤ 200 ns including the five validation checks. (b) **A6-P8b** — `DeploymentConfig.max_import_bytes` cap (independent S3 concern; config-only). | Synthesizer R1 §Open-Disputes "A6-P8: fuse with A1-P11"; Merge Register row "A6-P8 | A1-P11 (partial)". Splitting lets A1-P11 absorb the fast-path validation (single merged proposal) while the byte-cap remains as a small deployment-config addition that needs no per-arg cost. |
| A6-P9 | defend | — | `hot_path_impact: none` (≤ 5 ns integer compare on inbound dispatch, already on the slow framing path after A6-P3 length-prefix guard). A4-P6 (ADR cross-refs) can cross-link the new tenant field when `08-design-decisions.md` gains the ADR. No peer blocking objection. |
| A6-P10 | defend | — | API change to `add_sink(callable, *, severity_min, domains, logical_system_id)` is source-level only. A8-P10 (structured KV logging) and A8-P5 (alert sinks) need the same capability hook, so A6-P10 is factored out for their benefit too. No hot-path cost (dispatch loop filters, already off the hot path). |
| A6-P11 | defend | — | A2-P6 (pluggable `IDistributedProtocolHandler` registry) reinforces the need: any registry open for extension must have a privilege gate on `register_*` to prevent post-`init` injection. My A6-P11 text already matches A2-P6's "init-only" stance. A2 should welcome the gate (it protects their OCP extension point). |
| A6-P12 | amend | **Conditional on A9-P6 outcome.** If A9-P6 (defer PERFORMANCE/REPLAY) passes → A6-P12 becomes a v2 item (noted in deviations). If A9-P6 fails → A6-P12 applies as written and naturally composes with A2-P9 (versioned trace schema: the version field is the schema-binding target of the attached signature). | Synthesizer R1 §Open-Disputes "A6-P12: depends on A9-P6 outcome". The merge with A2-P9 is the minimal-overhead shape: `trace_schema_version` + `content_hash` + detached signature, verified at REPLAY `IExecutionEngine` init only. |
| A6-P13 | amend | Fold the per-tenant quota check into the **existing admission accounting path** (`02-logical-view/02-scheduler.md:§2.1.3.1.A` outstanding-window counter) — no new branch, just tag the existing per-Submission counter increment with `logical_system_id` and partition the counter. Quota breach surfaces via the existing `AdmissionDecision::REJECTED` channel (per A3-P2 unified status). | Synthesizer R1 §Open-Disputes "A6-P13: no new branch (fold into existing admission accounting)". A1 hot-path concern neutralized: HPI=`none` (same relaxed counter, just partitioned). A9 "multi-tenancy out of scope" mitigated: opt-in via `DeploymentConfig.per_tenant_submit_qps` (default `None` = disabled → zero cost for single-tenant). |
| A6-P14 | defend | — | Doc-only ADR. A2-P5 (Interface Evolution & BC policy) will naturally cite this ADR from its compatibility table. No peer objection expected. |

## 6. Votes on peer proposals

Rule references in rationale follow `04-agent-rules.md`. I mark `blocking=true` only when the objection rests on a hard rule from `04-agent-rules.md`. As a non-A1 reviewer, I may set `override_request=true` to lift an A1 veto; I use it sparingly.

### A1 (14 proposals)

| proposal_id | vote | rationale (cite rule id) | blocking | override_request |
|-------------|------|--------------------------|----------|-------------------|
| A1-P1 | agree | X2 — zero-alloc producer_index also eliminates a DoS amplification path (allocator spike on admission storm, S1-adjacent). | false | false |
| A1-P2 | agree | X3 — bounded lock hold time is a prerequisite for R1/R3 reasoning about admission timeout budgets. | false | false |
| A1-P3 | agree | P2 — LRU + HEARTBEAT-published presence also prevents an unbounded memory-growth vector via repeated function registrations (S1 DoS surface). | false | false |
| A1-P4 | abstain | Pure cache-efficiency concern; no security surface touched. | false | false |
| A1-P5 | agree | X9 — latency budgets are needed to validate A6-P3 bounds check cost (≤ 10 ns) fits. | false | false |
| A1-P6 | agree | P6 + S3 — routing binaries on a staging channel is the natural hook for A6-P7 attestation gating; descriptor dedup bounds wire exposure. Strong synergy with A6-P7. | false | false |
| A1-P7 | abstain | Pure profiling perf; no security implications. | false | false |
| A1-P8 | agree | X2 — bounded OutstandingWindow/ReadyQueue closes the admission-saturation DoS amplification (S1). | false | false |
| A1-P9 | abstain | Pure layout optimisation; no security surface. | false | false |
| A1-P10 | agree | P1 — SLO enforcement; tangential to security but no objection. | false | false |
| A1-P11 | agree | X9 + S3 — absorbs A6-P8a (structural DLPack validation) into the per-arg budget per Merge Register. See Merge §5 acceptance. | false | false |
| A1-P12 | abstain | Tail-latency budget; no security surface, but note: max_batch caps the blast radius of a single malformed batch, mildly S1-positive. | false | false |
| A1-P13 | abstain | HAL contract; no security concern provided `args_size` ceiling is enforced at boundary (A6-P3 adjacency). | false | false |
| A1-P14 | agree | X4 — documentation, no security concern. | false | false |

### A2 (9 proposals)

| proposal_id | vote | rationale (cite rule id) | blocking | override_request |
|-------------|------|--------------------------|----------|-------------------|
| A2-P1 | agree | E6 — schema versions at the trust boundary are necessary for A6-P3 (bounds) and A6-P12 (signed REPLAY) to attach to a stable binding. Two-tier: validation only at handshake / open, not on per-submit hot path. | false | false |
| A2-P2 | agree | E4 — schema-registered `LevelOverrides` is compatible with A6-P11 provided registration remains init-only. | false | false |
| A2-P3 | agree | E4 + OCP for the opened enums; keep `DepMode` closed. Caveat: opening `TransportBackend` via string ID must not weaken A6-P4 secure-default (a new plugin backend must declare its `require_encrypted_transport` posture at registration). Agree conditional on that note being added to A2-P5's BC policy. | false | false |
| A2-P4 | agree | E5 — migration plan is also a security scope boundary (what's still served by the legacy code path vs the new). No tension. | false | false |
| A2-P5 | agree | E2 — BC policy should explicitly classify security-load-bearing interfaces (`ISchedulerLayer::submit`, `IHorizontalChannel::send`, `register_function`, `add_sink`) as `stable-frozen` or `versioned-subinterface`. I will pre-file this in round 3 if needed. | false | false |
| A2-P6 | agree | E4 — pluggable `IDistributedProtocolHandler` registry is fine provided (i) table is init-only (mirrors my A6-P11 gate) and (ii) new handlers must declare their idempotency class (mirrors A5-P10). | false | false |
| A2-P7 | abstain | Future interface reservation only; doc-level. No security dimension. | false | false |
| A2-P8 | agree | G5 — Rule Exceptions bookkeeping directly supports S4 reasoning (intentional closures documented, not silent). | false | false |
| A2-P9 | agree | E6 — `trace_schema_version` is the exact hook A6-P12 needs to attach its signature to. Natural composition. | false | false |

### A3 (15 proposals)

| proposal_id | vote | rationale (cite rule id) | blocking | override_request |
|-------------|------|--------------------------|----------|-------------------|
| A3-P1 | agree | LSP — explicit `ERROR`/`CANCELLED` states give audit-log (A6-P6) deterministic emit points; without them `AuditEvent.outcome` cannot be computed reliably. | false | false |
| A3-P2 | agree | LSP — picking one of (a)/(b) for submit() error channel is required so A6-P13 quota breaches surface consistently. Prefer option (b) for future-proof `AdmissionStatus`. | false | false |
| A3-P3 | agree | V4 — admission-failure scenarios are the natural location for A6-P3 bounds-violation scenarios too. | false | false |
| A3-P4 | agree | R1, DS3 — `DEP_FAILED` is deterministic error propagation; removes unbounded wait that is an S1 DoS surface. | false | false |
| A3-P5 | agree | DS4 — sibling cancellation policy bounds error propagation time; prerequisite for auditable error trails (S6). | false | false |
| A3-P6 | agree | G1, V3 — traceability matrix helps enumerate requirement coverage including FR-9 (multi-tenant) which is directly A6 territory. | false | false |
| A3-P7 | agree | S3, G3 — Submission precondition validation fully overlaps with A6 input-validation goals. Co-owner with A6. | false | false |
| A3-P8 | agree | G3, X9 — cyclic `intra_edges` detection must be bounded because an unbounded check is itself a DoS surface (S1). O(V+E) with cap from `LevelParams`. | false | false |
| A3-P9 | agree | G3 — SPMD index/size ABI; no direct security concern but adds determinism. | false | false |
| A3-P10 | agree | D7 — completeness of Python exception mapping is prerequisite for honest audit-log outcomes. | false | false |
| A3-P11 | agree | G3 — `[ASSUMPTION]` marker; no tension. | false | false |
| A3-P12 | agree | G3, R4 — `drain()`/`submit()` contract defines the tenant-shutdown safety window (S2 least privilege: no admission after teardown begins). | false | false |
| A3-P13 | agree | DS3 — bounded reorder window also caps a replay-injection attack surface (aligns with my A6-P3 bounded-parse discipline). | false | false |
| A3-P14 | agree | LSP — one extra state-machine edge; no security impact. | false | false |
| A3-P15 | agree | G3, X6 — debug-mode cross-check; no security concern. | false | false |

### A4 (9 proposals)

| proposal_id | vote | rationale (cite rule id) | blocking | override_request |
|-------------|------|--------------------------|----------|-------------------|
| A4-P1 | agree | D7 — naming consistency. | false | false |
| A4-P2 | agree | D7, V5 — pure doc fix. | false | false |
| A4-P3 | agree | D7, G4 — prose arithmetic fix. | false | false |
| A4-P4 | agree | D7, V5 — numeric ordering. | false | false |
| A4-P5 | agree | D7 — glossary entries. Any new A6 formal types (`IAuditSink`, `Attestation`, `RegistrationToken`) must also be added — will cross-file in round 3. | false | false |
| A4-P6 | agree | V2, G5 — ADR cross-references. When A6 proposals land, each affected ADR should cite its §7.1.x location. | false | false |
| A4-P7 | agree | V5, D7 — L0 label unification. | false | false |
| A4-P8 | agree | D7 — state count sync. | false | false |
| A4-P9 | agree | D7 — glossary expansion. | false | false |

### A5 (10 proposals)

| proposal_id | vote | rationale (cite rule id) | blocking | override_request |
|-------------|------|--------------------------|----------|-------------------|
| A5-P1 | agree | R2 — exponential backoff + jitter also prevents correlated retry storms that amplify DoS (S1-adjacent). | false | false |
| A5-P2 | agree | R3 — per-peer circuit breaker state transitions are new security-relevant events; must be audit-log sites (fold into A6-P6). | false | false |
| A5-P3 | agree | R5 — coordinator failover or deterministic fail-fast; either path is a new authenticated-role boundary (see my §7 tension note: whichever option wins, A6-P2 handshake must bind "current coordinator" claim to prevent demoted-coordinator resurrection). | false | false |
| A5-P4 | agree | DS4 — `idempotent: bool` on `TaskDescriptor` is a correctness-critical label; must be captured in audit-log (A6-P6) for retry decisions. | false | false |
| A5-P5 | agree | R6 — chaos harness; composes with A8-P7 `IFaultInjector`. No security cost; actively helps chaos-test the authentication / rkey / quota paths (A6-P2/P5/P13). | false | false |
| A5-P6 | agree | R5, O5 — scheduler watchdog is a new audit-worthy event on fire. Fold alert emission into A6-P6 audit path. | false | false |
| A5-P7 | agree | R1 — `Timeout` on `IMemoryOps` closes a silent-stall window which is indistinguishable from targeted resource exhaustion without a deadline (S1 DoS). | false | false |
| A5-P8 | agree | R4 — graceful degradation specs; `admission_pressure_policy` must be per-tenant-aware (compose with A6-P13). | false | false |
| A5-P9 | agree | R3 — QUARANTINED Worker state is the intra-node equivalent of my A6-P2/A5-P2 circuit-breaking. No security objection. | false | false |
| A5-P10 | agree | DS4 — per-handler idempotency contract; foundational for retry-under-attack reasoning. | false | false |

### A7 (9 proposals)

| proposal_id | vote | rationale (cite rule id) | blocking | override_request |
|-------------|------|--------------------------|----------|-------------------|
| A7-P1 | agree | D6 — hard rule; cycle removal also clarifies the trust-flow direction between `scheduler/` (tenant-local) and `distributed/` (untrusted peer boundary). Foundational for A6-P9. | **true** | false |
| A7-P2 | agree | ISP, D4 — role-split reduces `bindings/` attack surface to `ISchedulerSubmit` + `ISchedulerLifecycle` only. S2 least-privilege at the interface layer. | false | false |
| A7-P3 | agree | D2 — core/hal inversion; no security dimension but cleanly satisfies DIP. | false | false |
| A7-P4 | agree | D3, D5 — moving distributed payloads to `distributed/` sharpens the trust boundary A6-P1 enumerates; `transport/` becomes a byte-pipe so S3 validation lives in one place (`distributed/protocol_payloads.h`). Strong A6 synergy. | false | false |
| A7-P5 | agree | D2 — `distributed_scheduler` depending only on `ISchedulerLayer` narrows the attack surface if tenant isolation (A6-P9) is later enforced at that interface. | false | false |
| A7-P6 | agree | D3, D5 — `composition/` extraction; no direct security concern. | false | false |
| A7-P7 | agree | D2, D6 — forward-decl contract explicit; no security concern. | false | false |
| A7-P8 | agree | D7 — `ScopeHandle` consolidation. | false | false |
| A7-P9 | agree | D7 — Python `MemoryError` deduplication; tangentially helps clear error-mapping surface for A6 audit. | false | false |

### A8 (12 proposals)

| proposal_id | vote | rationale (cite rule id) | blocking | override_request |
|-------------|------|--------------------------|----------|-------------------|
| A8-P1 | agree | X5 — injectable `IClock` is prerequisite to deterministically testing A5-P1 backoff, A6-P2 handshake timeouts, A6-P5 rkey rotation cadence. | false | false |
| A8-P2 | agree | X5, §8.2 DfT — driveable event-loop enables determinism tests of admission / quota (A6-P13) paths. | false | false |
| A8-P3 | agree | O3, O5 — enumerated stats + latency histograms; security alerts (failed handshakes, circuit-breaker trips, quota breaches, audit-sink degraded) need these primitives. | false | false |
| A8-P4 | agree | O4, X6 — `dump_state()` is operationally essential, **but** the default output must be scoped to a single `logical_system_id`; full cross-tenant dump must require an elevated capability. See §7 tension with A8. | false | false |
| A8-P5 | agree | O5, E3, X8 — externalized alert rules compose with A6-P6 audit events (`handshake_failed`, `circuit_open`, `quota_exceeded`). | false | false |
| A8-P6 | agree | O1 — cross-node trace alignment; no direct security concern but prerequisite for forensic analysis via A6-P12 REPLAY. | false | false |
| A8-P7 | agree | R6 — uniform `IFaultInjector` is sim-only; zero production cost; useful for fuzzing A6-P3 bounds and A6-P2 auth. | false | false |
| A8-P8 | agree | O3 — AICore trace upload at Level 2 only; default Level-1 strip keeps hot path unchanged. | false | false |
| A8-P9 | agree | O3, O5 — profiling drop as first-class alerts; same pattern should apply to audit-sink degraded state (fold into A6-P6). | false | false |
| A8-P10 | agree | O2 — structured KV logging is the natural substrate for A6-P6 audit events; lets audit reuse existing sink primitives. Strong synergy. | false | false |
| A8-P11 | agree | X5 — HAL contract tests; no security concern. | false | false |
| A8-P12 | agree | O1 — stable PhaseIds; audit events can adopt the same stable-id discipline. | false | false |

### A9 (8 proposals)

| proposal_id | vote | rationale (cite rule id) | blocking | override_request |
|-------------|------|--------------------------|----------|-------------------|
| A9-P1 | agree | G2 — dropping submit overloads reduces the attack surface at the C/Python boundary (S2 least privilege). | false | false |
| A9-P2 | abstain | No direct security dimension; deployment-mode choice. Note: if `IEventCollectionPolicy` is collapsed to an enum, its registration ceases to be a factory-registration site (simplifies A6-P11 gating). | false | false |
| A9-P3 | agree | G2 — removing collectives from `IHorizontalChannel` keeps the base transport interface minimal; every plug-in implements fewer methods → fewer stubs that could be misused. | false | false |
| A9-P4 | agree | G2, DRY — drop `SubmissionDescriptor::Kind`; no security dimension. | false | false |
| A9-P5 | agree | G2, DRY — unified admission enum; mandatory for A6-P13's reuse of the admission status channel. | false | false |
| A9-P6 | disagree | G2 motivation is legitimate, but REPLAY is the *one* forensic-analysis tactic for post-incident security investigation (S6-adjacent: replaying a suspected breach trace). Deferring REPLAY loses forensic capability and makes A6-P12 moot, weakening S6. Prefer `FUNCTIONAL` + `REPLAY` in v1, defer `PERFORMANCE`. Not blocking — this is a scope argument, not a hard-rule objection. | false | false |
| A9-P7 | agree | G2 — config merge; no security dimension. | false | false |
| A9-P8 | agree | G2, SRP — artifact-pipeline scope; no security dimension. | false | false |

### A10 (10 proposals)

| proposal_id | vote | rationale (cite rule id) | blocking | override_request |
|-------------|------|--------------------------|----------|-------------------|
| A10-P1 | agree | P4, DS6/DS7 — sharded `producer_index` should carry `logical_system_id` in the shard key to avoid cross-tenant hash collisions (A6-P9 adjacency). | false | false |
| A10-P2 | agree | P4, R5 — decentralized coordinator; see §7 tension: A6-P2 handshake must bind "current coordinator" claim via `cluster_view.generation` so a demoted coordinator cannot resume. Blocking unless addressed. | false | false |
| A10-P3 | agree | DS6 — per-data-element consistency table; explicitly add `audit_ring` and `rkey_scope_table` rows from A6-P5/A6-P6. | false | false |
| A10-P4 | agree | DS1 — stateful/stateless classification; audit ring (A6-P6) and rkey scope table (A6-P5) must be classified. | false | false |
| A10-P5 | agree | P6 + S2 — per-peer projection ships *only* the minimum information per peer, which is least-privilege-aligned (a compromised peer sees only its slice). Strong security co-benefit. | false | false |
| A10-P6 | agree | R5 — faster peer-failure detection; shortens the window during which a compromised peer can act with stale credentials. | false | false |
| A10-P7 | agree | P4 — sharded TaskManager; the global outstanding counter and per-tenant quota (A6-P13) must be compatible. Agree conditional on quota staying partition-aware. | false | false |
| A10-P8 | agree | DS7 — data-flow diagram; audit-log path and rkey-scope path should appear on the diagram. | false | false |
| A10-P9 | agree | DS6 — gate `WorkStealing` against `RETRY_ELSEWHERE`; assignment log is an audit-log emission site (fold into A6-P6). | false | false |
| A10-P10 | agree | P6, DS6 — `producer_index` cap + layout; aligns with A6-P3 "no unbounded growth" discipline. | false | false |

## 7. Cross-aspect tensions

| pair | with_proposal | preferred_resolution |
|------|---------------|----------------------|
| A6 vs A1 | A6-P3 (amended) | Single length-prefix guard at `MessageHeader` entry, derived from per-`MessageType` ceiling. ≤ 1 integer compare per frame on accept, same on reject (before any allocation). `hot_path_impact: extends(≤1 compare)`, far inside the 5 μs RDMA-transit budget of `04-process-view.md §4.8.2`. |
| A6 vs A1 | A6-P4 (amended) | TLS on TCP control-plane only; RDMA data-plane unaffected. `hot_path_impact: none` on the data path; TCP control-plane is already a slow path relative to the 2 μs Chip→Core budget. |
| A6 vs A1 | A6-P5 (amended) | Rotation at `SUBMISSION_RETIRED`, which is already a slow-path event per `04-process-view.md:491`. One MR deregister per retired Submission; steady-state rate ≪ 1/ms. |
| A6 vs A1 | A6-P8a (absorbed into A1-P11) | Per-arg structural validation merged with A1-P11's 200 ns DLPack import budget. Merge confirmed; see §Merge Register. |
| A6 vs A1 | A6-P9 | 4-byte header field + single integer compare on the inbound framing stage after A6-P3 length-prefix guard. `hot_path_impact: none` (< 5 ns, off hot path by `04-process-view.md §4.8.2` accounting). |
| A6 vs A1 | A6-P13 (amended) | Fast path = partitioned counter increment (≤ 1 ns relaxed atomic, same as today). Slow path (quota breached) surfaces via the existing `AdmissionDecision::REJECTED` channel. No new branch on the success path. |
| A6 vs A2 | A2-P6 (pluggable `IDistributedProtocolHandler`) | A2-P6 is compatible with A6-P11 provided registration is init-only; every registered handler must declare idempotency class (A5-P10) and passes through A6-P3 bounds. Agree conditional on A2-P5 BC policy capturing these requirements. |
| A6 vs A2 | A2-P3 (open `TransportBackend`) | A6 agrees provided each registered backend declares its `require_encrypted_transport` posture at registration and a new backend cannot lower the cluster default without an explicit `DeploymentConfig.allow_insecure_transport=true` (fail-closed S4 default). |
| A6 vs A5 | A5-P3 (coordinator failover) / A10-P2 | Critical: if the coordinator is a movable role, A6-P2 `HandshakePayload` must bind to the *current* `cluster_view.generation`. Otherwise a demoted coordinator with live credentials can silently resume issuing `REMOTE_SUBMIT`. Amendment: add `coordinator_generation: uint64_t` to `HandshakePayload`; message handlers reject messages whose `generation < cluster_view.current_generation - k` (small k for reorder tolerance). |
| A6 vs A7 | A7-P4 (move payloads to `distributed/`) | Strong alignment. Once payloads live in `distributed/protocol_payloads.h`, A6-P3 bounds and A6-P9 tenant-id check also consolidate there. |
| A6 vs A8 | A8-P4 (`Runtime::dump_state()`) | Without capability scoping, dump_state leaks cross-tenant information. Amendment: default dump output is filtered by the caller's `logical_system_id`; unscoped dump requires `DeploymentConfig.diagnostic_admin_token`. Blocking until addressed (S2 least privilege). |
| A6 vs A8 | A8-P5 (alert rules + sink) | Alert rules must be capability-scoped same as A6-P10 sinks — a tenant-scoped sink sees only that tenant's alerts. Add `logical_system_id` to `AlertRule`. |
| A6 vs A9 | A9-P6 (defer PERFORMANCE/REPLAY) | Disagree on REPLAY only: REPLAY is the forensic analysis tactic for post-incident investigation (S6-adjacent). Agree on deferring PERFORMANCE. Compromise path: v1 ships FUNCTIONAL + REPLAY; defer PERFORMANCE. If A9 insists on full deferral, A6-P12 becomes a v2 item and is registered in `10-known-deviations.md`. |
| A6 vs A9 | A6-P6 (audit trail) | A9 YAGNI preempted: v1 ships only the `IAuditSink` interface + in-memory pre-allocated ring + a built-in file sink. Audit events set is finite (8 event types in R1); no dynamic dispatch on hot path (emitted only at cold security-relevant sites). |
| A6 vs A9 | A6-P7 (function attestation) | A9 concern addressed via `multi_tenant=false` default: single-tenant users pay zero cost; multi-tenant users explicitly opt in. |

## 8. Stress-attack on emerging consensus (round 3+ only)

_Not applicable in round 2._

## 9. Status

- **Satisfied with current design?** partially — R2 voting suggests broad convergence on A6-P1/P6/P9/P10/P14 (no blocking peer objection) and amended convergence on A6-P3/P4/P5/P7/P8/P13. A6-P2 (node authentication) and A6-P12 (signed REPLAY) are the two items still requiring round-3 attention; the latter depends on A9-P6 outcome.
- **Open items expected in next round (R3 stress-attack):**
  - A6-P2 — node authentication primitive; expect A1 to confirm zero hot-path impact; expect A5/A10 to require `cluster_view.generation` binding.
  - A6-P4 (amended) — verify the TCP-only scope with A1; no hot-path regression.
  - A6-P6 — expect A9 to stress the interface surface; reply: interface + in-memory sink only in v1; file/syslog sinks ship as OCP extensions.
  - A6-P7 (amended) — expect A1 to confirm startup-only; expect A2 to cross-register with the BC policy.
  - A6-P11 — expect A2 to confirm alignment with A2-P6 registry gating.
  - A6-P12 — conditional on A9-P6 outcome; if REPLAY is deferred, A6-P12 auto-defers.
  - A6-P13 (amended) — expect A1 to confirm partitioned counter is zero-cost.
  - Cross-aspect blocker: A8-P4 `dump_state` tenant-scoping (S2) — must be resolved before `dump_state` ships.
  - Cross-aspect blocker: A5-P3 / A10-P2 coordinator failover must bind A6-P2 handshake to `cluster_view.generation`.

## Merge Register response

The synthesizer proposed six semantic-duplicate merges. A6 response:

| Merge | A6 action | Reason |
|-------|-----------|--------|
| A1-P6 ⇐ A10-P5 | accept (not A6-owned) | Neutral; no A6-owned content. Co-benefit for A6: per-peer projection implements S2 minimum-information-per-peer naturally. |
| A1-P14 ⇐ A10-P10 | accept (not A6-owned) | Neutral. |
| A5-P3 ⇐ A10-P2 | accept (not A6-owned) | Accept, with the §7 amendment that the merged proposal must bind A6-P2 handshake to `cluster_view.generation`. |
| A5-P6 ⇐ A8-P4 | accept (not A6-owned) | Neutral provided A8-P4 adopts tenant-scoping per §7. |
| A7-P2 ⇐ A9-P1 | accept (not A6-owned) | Neutral. |
| **A6-P8 ⇐ A1-P11 (partial)** | **accept with split** | A6-P8 is split per §5: A6-P8a (per-arg structural validation) merges into A1-P11's per-arg budget; A6-P8b (`max_import_bytes` deployment cap) remains as a standalone small config-only proposal owned by A6. |

WROTE /data/linjiashu/Code/pto-dev-folder/docs/pypto-runtime-design/reviews/2026-04-18-171357/round-2/reviews/A6-security.md
