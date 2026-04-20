# Edit: docs/pypto-runtime-design/02-logical-view/09-interfaces.md

- **Driven by proposal(s):** A2-P1, A2-P6, A2-P7, A3-P2, A3-P7, A3-P12, A7-P2, A9-P1, A9-P2, A9-P4, A9-P5, A9-P7
- **ADR link(s):** ADR-015 (distributed protocol header independence), ADR-017 (`std::expected` admission return), ADR-021 (extension_map), ADR-024 (role-interface ISP split)
- **Timestamp (UTC):** 2026-04-19
- **Edit kind:** structural (role-interface split) + modification (surgical callouts) + additive (three new subsections §2.6.5–§2.6.8)

## Location

- Path: `docs/pypto-runtime-design/02-logical-view/09-interfaces.md`
- Sections touched: §2.6.1 (ISchedulerLayer), §2.6.1.A (Submission types), §2.6.3 (schedule policy), §2.6.4 (EventHandlingConfig), new §2.6.5 (role interfaces), new §2.6.6 (IEventLoopDriver), new §2.6.7 (distributed protocol handler), new §2.6.8 (Q-record async policy).
- Line range before edit: 28–341 (insertion/change points; file grew to 679 lines)

## Before

- `ISchedulerLayer::submit` returned `SubmissionHandle` by value; three submit overloads existed (`submit(TaskDescriptor)`, `submit_group`, `submit_spmd`).
- `SubmissionDescriptor` had `enum class Kind`, no `schema_version`, no `flags`.
- `IResourceAllocationPolicy` / `ResourceAllocationResult` used ad-hoc enums; admission outcomes diverged across policy and result.
- `SourceCollectionConfig` was a standalone struct.
- No role-interface split; no test-only event-loop driver; no distributed protocol handler; no async-policy placeholder.

## After

### A3-P2 — normative `std::expected<>` return (§2.6.1)

- **Before:** `virtual SubmissionHandle submit(const SubmissionDescriptor&)`.
- **After (interface edit + callout):** returns `std::expected<SubmissionHandle, ErrorContext>`; throwing overload is convenience.

> [UPDATED: A3-P2: normative std::expected<SubmissionHandle, ErrorContext> return shape] …

### A9-P1 — collapse submit overloads (§2.6.1)

- **Before:** `submit(TaskDescriptor)`, `submit_group(...)`, `submit_spmd(...)` on the interface.
- **After:** overloads REMOVED; free-function builders `build_single`/`build_group`/`build_spmd` live in `runtime/submission_builders.h`. Absorbed into A7-P2 ISP split (atomic landing).

### A3-P12 — `drain()` + `submit()` concurrency (§2.6.1)

> [UPDATED: A3-P12: drain sticky + REMOTE_DRAIN propagation] `submit()` returns `AdmissionStatus::REJECT(Drain)` once `drain()` is entered; sticky; idempotent; distributed semantics via `REMOTE_DRAIN`.

### A2-P1 / A9-P4 — `SubmissionDescriptor` version + drop `Kind` (§2.6.1.A)

- **Before:** struct contained `enum class Kind` + `kind` member, no `schema_version`.
- **After:** `uint16_t schema_version = 1;` added; `enum class Kind` + `kind` removed. Inline comments mark the change.

### A3-P7 — precondition catalog (§2.6.1.A)

- **After (callout below `SubmissionHandle`):** full precondition list (`EmptyTasks`, `SelfLoopEdge`, `EdgeIndexOOB`, `BoundaryIndexOOB`, `WorkspaceSubrangeOOB`, `WorkspaceSizeMismatch`, `SpmdIndexOOB`, `SpmdSizeMismatch`, `SpmdBlockDimZero`), routed to `AdmissionStatus::REJECT(Validation|Exhaustion|Drain)`. Also reiterates `schema_version` for `MessageHeader`, `TaskDescriptor`, `FunctionDescriptor`, stats, trace header.

### A2-P1 — `FE_VALIDATED` bit (§2.6.1.A)

- **After:** `uint32_t flags = 0; // bitfield; FE_VALIDATED = 0x1` added to `SubmissionDescriptor`; release-mode fast-path documented in `02-scheduler.md` A3-P8 callout.

### A9-P5 — unified `AdmissionStatus` enum (§2.6.3)

- **After (callout below `ResourceAllocationResult`):**

> [UPDATED: A9-P5: unified AdmissionStatus enum] { ADMIT, WAIT, REJECT_Exhaustion, REJECT_Validation, REJECT_Drain }. Prior `AdmissionDecision` kept as compat typedef.

### A9-P7 — fold `SourceCollectionConfig` into `EventHandlingConfig` (§2.6.4)

- **Before:** standalone `SourceCollectionConfig` struct.
- **After (callout + struct):** `EventHandlingConfig` absorbs source list, per-source weight, max events per cycle; legacy type kept as `using SourceCollectionConfig = EventHandlingConfig;` compat alias for one release.

### A7-P2 — role interfaces (new §2.6.5)

New subsection defining `ISchedulerWiring`, `ISchedulerSubmit`, `ISchedulerCompletion`, `ISchedulerLifecycle`, `ISchedulerScope`, with `ISchedulerLayer` as aggregator. Per-interface contracts described. Absorbs A9-P1 (submit-overload collapse) atomically.

### A9-P2 — IEventLoopDriver test seam (new §2.6.6)

New subsection declaring test-only `IEventLoopDriver` + `RecordedEventSource`, gated on `LayerConfig.enable_test_driver`; release binaries omit both. Normative definition of determinism (bit-identical replay) lives here.

### A2-P6 — free-function distributed protocol handler (new §2.6.7)

New subsection declaring v1 free function `handle_peer_protocol_message`, registered through `register_handler(MessageType, DistributedMessageHandler)`. Abstract class `IDistributedProtocolHandler` deferred (ADR-015 / I-DIST-1 cross-ref).

### A2-P7 — Q-record placeholder only (new §2.6.8)

Normative "no v1 interface" statement. Async policy, transport capability semantics, and async-submit return path tracked as Q15 in `09-open-questions.md`.

## Per-proposal edits (summary table)

| Proposal | Section(s) | Kind |
|---|---|---|
| A2-P1 | §2.6.1.A | modification (field add) |
| A2-P6 | §2.6.7 (new) | additive |
| A2-P7 | §2.6.8 (new) | additive (Q-record only) |
| A3-P2 | §2.6.1 | modification (signature) |
| A3-P7 | §2.6.1.A | modification (callout) |
| A3-P12 | §2.6.1 | modification (callout) |
| A7-P2 | §2.6.5 (new) | structural (ISP split) |
| A9-P1 | §2.6.1 | modification (drop overloads) |
| A9-P2 | §2.6.6 (new) | additive |
| A9-P4 | §2.6.1.A | modification (drop `Kind`) |
| A9-P5 | §2.6.3 | modification (unified enum) |
| A9-P7 | §2.6.4 | modification (fold struct) |

## Rationale

09-interfaces.md is the keystone of the logical view; nearly every other file cross-references it for contract shape. The structural ISP split (A7-P2) lets callers include the narrow header they actually need, which lowers compile dependencies and clarifies roles. The `std::expected` return (A3-P2) makes the admission taxonomy (A9-P5) first-class in the type system. Closed enums + `extension_map` (A2-P2 across views) + Q-record placeholders (A2-P7) preserve a single forward-compat axis without adding speculative polymorphism.

## Verification steps

1. `grep -n "\[UPDATED: A" docs/pypto-runtime-design/02-logical-view/09-interfaces.md` should return ≥ 10 hits.
2. `grep -n "2.6.5\|2.6.6\|2.6.7\|2.6.8" docs/pypto-runtime-design/02-logical-view/09-interfaces.md` — confirms the four new subsections landed.
3. Cross-reference `02-scheduler.md` (closed enums / latency budgets / precondition catalog), `07-task-model.md` (A9-P4 Kind documentary), `05-machine-level-registry.md` (A9-P7 fold).
4. `modules/distributed.md §3.2` and ADR-015 / I-DIST-1 should agree with §2.6.7 (free function only; abstract class deferred).
5. `modules/core.md §8` Task layout should reflect the hot/cold split that A7-P2 narrows the completion path over.

## Line-count delta

- Before: ~380 lines (pre-edits)
- After: 679 lines (+~300 lines, split across four new subsections and in-interface callouts)
