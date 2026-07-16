# Backend Checklist

## Purpose
A pre-submission checklist the Backend Developer runs before handing work to Code Review.

## Responsibilities
- Owned by: **Backend Developer**.

## Inputs
- `/backend/rules.md`, the feature/bugfix being implemented.

## Outputs
- A implementation that passes review on the first pass more often, reducing review round-trips.

## Rules
- This checklist is run before requesting review, not after receiving feedback.
- If an item doesn't apply, note why rather than silently skipping it.

## Checklist
- [ ] Code follows `/backend/rules.md`.
- [ ] Input validation present at all new/changed endpoints.
- [ ] Relevant business rules enforced (cross-checked against `/business/business-rules.md`).
- [ ] Tests written/updated and passing locally.
- [ ] No secrets or environment-specific values hardcoded.
- [ ] API changes reflected in `/architecture/api-design.md`.
- [ ] Migrations (if any) are reversible or explicitly flagged as breaking.

## Notes
_None yet._
