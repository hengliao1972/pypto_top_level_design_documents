# Edit: docs/pypto-runtime-design/02-logical-view/12-dependency-model.md

- **Driven by proposal(s):** A5-P4
- **ADR link(s):** — (cross-ref to A3-P4 propagation; `07-cross-cutting-concerns.md §5 Error Taxonomy`)
- **Timestamp (UTC):** 2026-04-19
- **Edit kind:** modification (single callout appended to §2.10.6)

## Location

- Path: `docs/pypto-runtime-design/02-logical-view/12-dependency-model.md`
- Sections touched: §2.10.6 (Scope Limits / v1) end
- Line range before edit: 128 (insertion point; file grew to 156 lines)

## Before

The dependency model specified `producer_index` maintenance and Scope Limits but did not pin down the **retry** semantics that the DS4 invariant (idempotent bit) implies, nor the retry catalog that binds A3-P4 failure propagation to A5-P4 retry eligibility.

## After

### A5-P4 — DS4 idempotent bit semantics + retry catalog (§2.10.6)

- **After (callout):**

> [UPDATED: A5-P4: DS4 idempotent bit semantics + retry catalog]
> **DS4 — Idempotent retry contract.** The runtime MAY retry a `FAILED` Task iff `TaskDescriptor.idempotent == true` AND the failure cause is in the retry catalog. Flag defaults `true`; side-effectful Tasks MUST set `false`. Non-idempotent transient failures → Submission `ERROR` + `DEP_FAILED` via A3-P4 successor walk.
>
> **Retry catalog.**
>
> | ErrorCode | Cause | Default retry budget |
> |---|---|---|
> | `TransportTimeout` | Vertical/horizontal channel wait deadline (A5-P7) | 3 |
> | `WorkerTransientFault` | Worker soft-error status (ECC-corrected read, throttle) | 2 |
> | `QuarantineCleared` | Worker quarantine expired and recovered (A5-P9) | 1 |
> | `AdmissionStatus::WAIT` | Back-pressure from resource policy | unbounded, bounded by `max_deferred` |
>
> Failures not in the catalog (`InvariantViolation`, `ValidationFailure`, `InternalError`, memory-manager non-aliasing violations) are **never** retried. SSoT: `07-cross-cutting-concerns.md §5 Error Taxonomy`.
>
> **producer_index interaction.** Retry success leaves the entry unchanged (same `(submission_id, task_handle)`; slot not recycled; generation stable). Retry exhaustion walks the cold-tail successor list and emits `DEP_FAILED` (A3-P4).

- **Rationale:** R4 / R5 / O5 — retries tied to a minimal, explicit catalog that is disabled by default for side-effectful Tasks.

## Rationale

The idempotent-retry policy is where failure handling (A3-P4), Worker health (A5-P9), and memory-copy deadlines (A5-P7) all meet. Centralizing the retry catalog here — adjacent to DS4's runtime contract — keeps the behavior auditable and gives a single page that a frontend author can consult to reason about whether a Task is safe to mark idempotent.

## Verification steps

1. `grep -n "\[UPDATED: A5-P4" docs/pypto-runtime-design/02-logical-view/12-dependency-model.md` returns 1 hit.
2. `07-cross-cutting-concerns.md §5` Error Taxonomy must include the four retryable `ErrorCode`s and classify the others as non-retryable.
3. `07-task-model.md §2.4.1` must declare `bool idempotent = true;` on `TaskDescriptor`.
4. `02-scheduler.md` A3-P4 callout must emit `DEP_FAILED` on retry exhaustion.

## Line-count delta

- Before: 141 lines
- After: 156 lines (+15 lines)
