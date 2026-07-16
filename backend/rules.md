# Backend Rules

## Purpose
Codify how backend code should be written so implementation is consistent regardless of who (or which agent) writes it.

## Responsibilities
- Owned by: **Backend Developer**; enforced by **Code Reviewer**.
- Must stay consistent with `/architecture/architecture.md` and `/architecture/api-design.md`.

## Inputs
- Architectural constraints, API contracts, business rules from `/business/business-rules.md`.

## Outputs
- Backend implementation that is predictable, testable, and reviewable against a known standard.

## Rules
- Validate all input at system boundaries (API handlers); never trust client-supplied data.
- Business rules from `/business/business-rules.md` are enforced in the backend, not just in the UI.
- No secrets, credentials, or environment-specific config committed to source — use environment variables per `/devops/infrastructure.md`.
- Errors are handled explicitly and surfaced with meaningful messages; no silent failures.
- Database migrations are additive/reversible where possible; document breaking migrations in `/architecture/decisions.md`.
- New dependencies require a stated reason — no adding libraries "just in case."

## Checklist
- [ ] Input validated at the boundary.
- [ ] Relevant business rules enforced and referenced by ID.
- [ ] Errors handled explicitly, not swallowed.
- [ ] No secrets committed.
- [ ] Tests added/updated per `/testing/strategy.md`.

## Notes
_Tech stack not yet chosen — see `/architecture/decisions.md`. This file should be revisited once the stack is decided to add language/framework-specific conventions._
