# Edit: docs/pypto-runtime-design/02-logical-view/04-memory.md

- **Driven by proposal(s):** A5-P7
- **ADR link(s):** — (cross-ref to ADR-015 / modules/memory.md)
- **Timestamp (UTC):** 2026-04-19
- **Edit kind:** modification (surgical callout + interface signature update)

## Location

- Path: `docs/pypto-runtime-design/02-logical-view/04-memory.md`
- Sections touched: §2.1.7 (IMemoryOps)
- Line range before edit: 213 (insertion point; file grew to 312 lines)

## Before

`IMemoryOps::wait(AsyncHandle)` and `IMemoryOps::poll(AsyncHandle)` had no deadline/timeout parameters; cross-Pod dedup-window policy for `copy_async` retries was not stated.

## After

A single callout below the `IMemoryOps` class, normatively updating the two async-support signatures and cross-referencing the dedup-window policy.

## Per-proposal edits

### A5-P7 — `Timeout` on `IMemoryOps` async; dedup-window cross-ref

- **After (callout):**

> [UPDATED: A5-P7: Timeout on IMemoryOps::wait/poll + dedup window] The async-support surface is updated to:
> ```cpp
> virtual Status wait(AsyncHandle handle, Timeout timeout) = 0;
> virtual bool   poll(AsyncHandle handle, Timeout timeout = Timeout::zero()) = 0;
> ```
> Expired deadlines return `Status::TransportTimeout` (retryable per A5-P4 DS4 catalog). Cross-Pod `copy_async` retries are coalesced by the dedup-window policy specified in `modules/memory.md §2.4`: dedup key `(peer_id, logical_stream_id, src_offset, size, epoch)` with window `transport_dedup_window_ms` (default 50 ms). Summary policy mirrored here; SSoT remains `modules/memory.md`.

- **Rationale:** R4 / V3 — bounded-wait is a contract requirement; dedup window is the cross-Pod back-pressure tie-in.

## Rationale

The pre-existing `IMemoryOps` shape allowed unbounded blocking waits on async handles — unacceptable under the runtime's deadline-first design. Adding a required `Timeout` on `wait` and a defaulted `Timeout` on `poll` keeps the API ergonomic while enforcing deadline propagation.

## Verification steps

1. `grep -n "\[UPDATED: A5-P7" docs/pypto-runtime-design/02-logical-view/04-memory.md` returns 1 hit.
2. Retry catalog in `12-dependency-model.md` A5-P4 callout must list `TransportTimeout` as retryable.
3. `modules/memory.md §2.4` dedup-window parameters must match this page's summary.

## Line-count delta

- Before: 300 lines (pre-callout)
- After: 312 lines (+12 lines)
