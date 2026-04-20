# Simpler Runtime — Architecture Design

This document set describes the architecture of the redesigned Simpler runtime, organized according to the **4+1 View Model** as prescribed by the [Architecture Design Guide](../../.cursor/guides/architecture-design-guide/00-index.md).

The source material originates from the [architecture-redesign](../architecture-redesign/) working documents (Stages 1–8). This restructured version reorganizes that content into the standard architectural views for clarity, cross-view consistency, and reviewability.

## Document Index

| # | Document | Content |
|---|----------|---------|
| WP | [White paper](white-paper.md) | Internal on-ramp: goals, philosophy, principles, scenarios, architecture summary; links to 01–10 and ADRs (no implementation detail) |
| 01 | [Introduction](01-introduction.md) | Purpose, scope, requirements summary, assumptions |
| 02 | [Logical View](02-logical-view.md) | Overview of the domain model (Abstract Machine, Execution Layers, Machine Levels, Functions, Tasks, Communication) with a module breakdown linking to per-component sub-documents under [`02-logical-view/`](02-logical-view/) (Domain Model, Scheduler, Worker, Memory, Machine Level Registry, Function Types, Task Model, Communication, Interfaces, Platform, Machine & Memory Model, Dependency Model) |
| 03 | [Development View](03-development-view.md) | Module structure (`hal/`, `core/`, `scheduler/`, `memory/`, `transport/`, `distributed/`, `profiling/`, `error/`, `runtime/`, `bindings/`), dependency graph, build targets |
| — | [Module detailed designs](modules/) | Per-module Section 8 designs ([template](../../.cursor/guides/architecture-design-guide/05-output-templates.md#8-module-detailed-design)), one `.md` per module |
| 04 | [Process View](04-process-view.md) | Concurrency model, key interaction flows (hierarchical task submission, distributed scheduling), task state machine, error handling flows |
| 05 | [Physical View](05-physical-view.md) | Deployment topology, configuration examples, network topology, scaling strategy, availability |
| 06 | [Scenario View](06-scenario-view.md) | Single-node hierarchical execution, distributed multi-node execution, failure scenarios |
| 07 | [Cross-Cutting Concerns](07-cross-cutting-concerns.md) | Security, observability (profiling/tracing/logging), performance |
| 08 | [Design Decisions](08-design-decisions.md) | ADRs for key architectural choices |
| 09 | [Open Questions](09-open-questions.md) | Unresolved questions flagged for review |
| 10 | [Known Deviations](10-known-deviations.md) | Rule violations with justification (per [04-agent-rules.md](../../.cursor/guides/architecture-design-guide/04-agent-rules.md)) |
| A | [Appendix A — Glossary](appendix-a-glossary.md) | Formal term definitions and cross-references |
| B | [Appendix B — Codebase Mapping](appendix-b-codebase-mapping.md) | Mapping from current codebase to new architecture |

## Traceability to Source Material

| Source Document | Restructured Into |
|----------------|-------------------|
| `architecture-redesign/00-PLAN.md` (Goals, Principles) | [01-introduction.md](01-introduction.md), [08-design-decisions.md](08-design-decisions.md) |
| `architecture-redesign/00-PLAN.md` (Stages 1–8 summaries) | Distributed across views 02–07 |
| `architecture-redesign/00-ACTION-BREAKDOWN.md` | Distributed across views 02–07 |
| `architecture-redesign/01-abstract-machine-model.md` | [02-logical-view.md](02-logical-view.md) (§§1–5), [04-process-view.md](04-process-view.md) (§§2–3), [05-physical-view.md](05-physical-view.md) (§§1–3), [appendix-a-glossary.md](appendix-a-glossary.md) |

## Conventions

- All design docs follow the [Architecture Design Guide](../../.cursor/guides/architecture-design-guide/00-index.md).
- Code examples use C++17 (matching the current codebase).
- Python examples target Python 3.8+ with nanobind.
- Interface definitions use C++ abstract classes.
- Data structures include memory layout where relevant.
- Hierarchical patterns are described once as a generic layer, then instantiated per concrete level.
- Distributed scenarios use sequence diagrams for cross-node message flows.
- Diagrams use ASCII art or Mermaid syntax.
- Formal terms from the [Glossary](appendix-a-glossary.md) are capitalized on first use in each document.

## Design Status

| View | Status |
|------|--------|
| Introduction | Draft |
| Logical View | Draft (Stage 1 content restructured) |
| Development View | Draft (Stage 4, 8 content restructured) |
| Process View | Draft (Stage 2, 5, 6 content restructured) |
| Physical View | Draft (Stage 1 §7, §16 content restructured) |
| Scenario View | Draft (Stage 2 §2.10 content restructured) |
| Cross-Cutting Concerns | Draft (Stage 5, 6 content restructured) |
| Design Decisions | Draft |
| Open Questions | Draft |
| Known Deviations | Draft |
| Module detailed designs (`modules/*.md`) | Draft (per-module Section 8) |
| White paper (`white-paper.md`) | Draft |
