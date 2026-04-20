# Edit: docs/pypto-runtime-design/10-known-deviations.md

- **Driven by proposal(s):** A5-P10 (per-`REMOTE_*` idempotency; future `REMOTE_COLLECTIVE_FLUSH` placeholder)
- **ADR link (if any):** —
- **Timestamp (UTC):** 2026-04-19
- **Edit kind:** additive (one new Deviation entry appended at end-of-file)

## Location

- Path: `docs/pypto-runtime-design/10-known-deviations.md`
- Section / heading: end of file — appended after existing Deviation 4.
- Line range before edit: 49–51 (end-of-file anchor).

## Before

```49:51:docs/pypto-runtime-design/10-known-deviations.md
**Resolution:** Per-module detailed design drafts now exist under [`modules/`](modules/) (one document per module, Section 8 template). They are **Draft** status pending review alongside the rest of the architecture package.

**Remaining gap:** Internal architecture details may still be refined during implementation provided public interface contracts are preserved ([Module Design Conventions in the Architecture Design Guide](../architecture-design-guide/05-output-templates.md#module-design-conventions)).
```

(end-of-file)

## After

One new Deviation entry appended, preceded by the `[UPDATED: A5-P10: future-type annotation]` marker:

```markdown
**Remaining gap:** Internal architecture details may still be refined during implementation provided public interface contracts are preserved (...).

---

<!-- [UPDATED: A5-P10: future-type annotation] -->

## Deviation 5: `REMOTE_COLLECTIVE_FLUSH` Dedup-Window Annotation Placeholder (Rule DS4)

**Rule:** DS4 — Every `REMOTE_*` handler MUST declare `idempotency ∈ {safe, at-most-once, at-least-once}` with a link to its dedup key; doc-lint CI enforces the annotation.

**Violation:** `REMOTE_COLLECTIVE_FLUSH` is reserved as a future `MessageType` (see Q16 and A9-P3 orchestration-composed collectives path) but is not implemented in v1 and therefore carries no concrete dedup-key binding.

**Justification:** Reserving the type now prevents future type-tag churn; a placeholder annotation would be misleading. Per A5-P10, the type is allowlisted in doc-lint CI as "reserved, no v1 handler" until the engine lands.

**Mitigation:** When the orchestration-composed collective handler is implemented (Q16), the annotation becomes normative and is CI-enforced. Until then, `REMOTE_COLLECTIVE_FLUSH` sends return `ErrorCode::MessageTypeNotImplemented`.
```

## Rationale

A5-P10 requires every `REMOTE_*` handler to carry an idempotency annotation linked to a dedup key, enforced by doc-lint CI. `REMOTE_COLLECTIVE_FLUSH` is reserved for the future orchestration-composed collective handler (Q16 + A9-P3 path) but has no v1 handler; the correct resolution is to record the gap as a Known Deviation with a clear activation trigger, rather than introduce a dummy annotation that would drift from reality. The entry is short (3–5 substantive lines) and cross-links Q16, A9-P3, and the DS4 rule so reviewers can trace the deferral rationale.

## Verification steps

1. `grep -n "^## Deviation" docs/pypto-runtime-design/10-known-deviations.md` returns five entries (Deviations 1–5).
2. `grep -n "\[UPDATED: A5-P10" docs/pypto-runtime-design/10-known-deviations.md` returns exactly one match.
3. Cross-doc: Q16 in `09-open-questions.md` references A9-P3 and §6.2.2; Deviation 5 links to Q16 — consistent.
4. Doc-lint CI (future) should treat `REMOTE_COLLECTIVE_FLUSH` as allowlisted; confirm allowlist entry matches Deviation 5 wording when the lint rule is added.
