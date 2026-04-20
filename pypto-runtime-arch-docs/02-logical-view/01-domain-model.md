# 2.1 Domain Model

> Part of the [Logical View](../02-logical-view.md). This module describes the core domain concepts: the Abstract Machine and the Execution Layer — the recursive building block on which every other component rests. Other module components of the domain model (Scheduler, Worker, Memory) are described in separate sub-documents linked from the [Logical View index](../02-logical-view.md#21-module-breakdown).

The Simpler runtime is built upon a formal **Abstract Machine** — a computational model that defines how hardware resources are organized, work is scheduled, data is moved, and execution is managed. All terms defined here are **normative** and used consistently throughout all views.

## 2.1.1 The Abstract Machine

The Abstract Machine is composed of five pillars:

1. A **Machine Level Registry** ([§2.2](05-machine-level-registry.md)) — the configurable, named hierarchy of compute and memory resources, with registered vertical and horizontal communication implementations.
2. A stack of **Execution Layers** ([§2.1.2](#212-execution-layer)) — the recursive scheduling and execution framework, instantiated from the Machine Level Registry.
3. A set of **Function** types ([§2.3](06-function-types.md)) — the callable units of computation.
4. A **Task** model ([§2.4](07-task-model.md)) — the unit of schedulable work.
5. A **Communication** model ([§2.5](08-communication.md)) — how layers, workers, and nodes exchange control and data, implemented through the registered channels.

The Abstract Machine is parameterized by a **Platform** ([§2.8](10-platform.md)) — a concrete hardware configuration (e.g., `a2a3`, `a5`) with specific core counts, memory sizes, and capabilities. Runtime behavior is platform-independent; only the Machine Level implementations and HAL are platform-specific.

## 2.1.2 Execution Layer

An **Execution Layer** is the fundamental recursive building block:

```
Layer = (Scheduler, Workers[], MemoryScope)
```

- **Scheduler** ([§2.1.3](02-scheduler.md)): Dispatches Tasks to Workers within this Layer.
- **Workers** ([§2.1.4](03-worker.md)): Execute Tasks. During execution, a Worker MAY submit child Tasks to its own Layer's Scheduler, which dispatches them downward to the child Layer via the Vertical Channel.
- **Memory Scope** ([§2.1.5](04-memory.md#215-memory-scope)): The addressable memory region accessible to all Workers in this Layer.

A Layer is identified by a **Layer Ordinal** `k ∈ {0, 1, ..., N}` where Layer 0 is the lowest (closest to hardware) and Layer N is the highest (closest to the user).

A **Layer Stack** is an ordered sequence `[L₀, L₁, ..., Lₙ]` where:
- `Lₖ₊₁` is the Parent Layer of `Lₖ`.
- `Lₖ` is the Child Layer of `Lₖ₊₁`.
- Adjacent layers communicate through Vertical Channels; siblings through Horizontal Channels.
