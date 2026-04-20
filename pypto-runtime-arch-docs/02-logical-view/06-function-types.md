# 2.3 Function Types

> Part of the [Logical View](../02-logical-view.md). This module describes the **Function** — a registered callable with a declared signature, target Machine Level, and optional compiled binary. Functions are *definitions*; [Tasks](07-task-model.md) are *calling instances*.

A **Function** is a registered callable defining a function body and a function interface (in/out data parameters and attributes). A Function is a **definition**; a **Task** ([§2.4](07-task-model.md)) is a calling instance.

Every Function has:
- A **Function ID** (`func_id: uint32_t`).
- A **Function Name** (max 64 characters).
- A **Signature** — ordered `ArgDirection` values (`SCALAR`, `IN`, `OUT`, `INOUT`).
- A **Target Layer** — the Machine Level at which it executes.
- A **Binary** (optional) — compiled machine code for device-side Functions.
- A **Content Hash** (optional) — for cache-based deduplication ([§2.3.5](#235-function-cache)).

## 2.3.1 AICore Function

Executes on one or more AICores at a leaf or near-leaf Machine Level. It is a **leaf** — its body does NOT submit child Tasks.

- Compiled to device binary (AICore instruction set).
- Dispatched via a Dispatch Payload written to core-accessible memory.
- Parameterized by Block Dimension (how many AICores execute in parallel).

> [UPDATED: A3-P9: SPMD `spmd_index` / `spmd_size` delivery contract]
> For SPMD Submissions ([§2.4.6](07-task-model.md#246-spmd-submission)), the per-rank `spmd_index` and `spmd_size` are delivered in `TaskArgs.scalar_args` at **reserved indices 0 and 1**. The ABI is preserved under A9-P4's removal of the `SubmissionDescriptor::Kind` discriminator — SPMD-ness is derivable from `spmd.has_value()` and does not require a separate tag. Non-SPMD Submissions leave these two scalar slots unused (frontends MUST NOT repurpose them).

## 2.3.2 Orchestration Function

Executes on a Worker at an intermediate Machine Level. Its body MAY:
- Submit child Tasks to its own level's Scheduler (which dispatches them downward via the Vertical Channel).
- Read/write the current Layer's Memory Scope.
- Implement control flow (loops, conditionals) over child Task submissions.

This is the mechanism for hierarchical execution: runs on a level-L Worker, submits tasks to the level-L Scheduler, dispatched to level-(L−1) Workers. When an Orchestration Function submits children, the parent Task enters COMPLETING and does not finish until all children complete.

## 2.3.3 Host Function

Executes on the host CPU within the host process. MAY:
- Submit Tasks to child Machine Levels via the Vertical Channel.
- Invoke Python callbacks (with GIL management).
- Access host-level Memory Scope and manage data transfers.

## 2.3.4 Distributed Orchestration Function

Executes at a distributed Machine Level (`"Pod"`, `"Supernode"`, `"Global"`). MAY:
- Submit Tasks to sibling instances via the Horizontal Channel and Distributed Scheduler.
- Coordinate cross-instance data movement.
- Implement collective operations as compositions of data movement and compute Tasks.

## 2.3.5 Function Cache

A per-Machine-Level-Instance cache keyed by Content Hash:

```
FunctionCache[content_hash] → { FunctionRef, binary_pointer, last_access_time }
```

- Subsequent Tasks reuse cached Functions without re-deployment.
- For distributed levels, binaries are sent to remote nodes only if their cache lacks the hash.
- Cache eviction is configurable (LRU, TTL, or manual). Lazy loading is supported: a remote node may request a binary when it encounters an unknown hash.
