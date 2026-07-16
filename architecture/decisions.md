# Architecture Decision Records (ADRs)

## Purpose
Preserve the *reasoning* behind significant technical decisions so future contributors don't have to reverse-engineer "why" from code.

## Responsibilities
- Owned by: **Software Architect**.
- Any role may propose an ADR; the Architect accepts, rejects, or supersedes it.

## Inputs
- Architectural questions raised during implementation (e.g., "which database?", "sync or async processing?").
- Constraints from `/business/requirements.md` and `/architecture/architecture.md`.

## Outputs
- A permanent, numbered log of decisions that `/architecture/architecture.md` and implementation should conform to.

## Rules
- Use the standard ADR format: Context, Decision, Consequences, Status (Proposed/Accepted/Superseded).
- Never edit or delete a past ADR to change its decision — supersede it with a new one and link back.
- Every ADR gets a sequential ID: `ADR-001`, `ADR-002`, ...

## Checklist
- [ ] ADR states the problem/context, not just the answer.
- [ ] ADR lists real alternatives considered and why they were rejected.
- [ ] ADR consequences (trade-offs accepted) are explicit.

## Notes
_No decisions recorded yet._

---

## ADR-000: Template

**Status:** Template — not a real decision.

**Context:** What problem are we solving? What constraints apply?

**Decision:** What did we decide?

**Alternatives considered:** What else was on the table, and why was it rejected?

**Consequences:** What trade-offs does this decision accept?
