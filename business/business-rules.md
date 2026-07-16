# Business Rules

## Purpose
Document the invariants and policies the business imposes on the system — the rules that must hold true regardless of implementation.

## Responsibilities
- Owned by: **Business Analyst**.
- Enforced in code by **Backend Developer**; verified by **QA/Testing Engineer**.

## Inputs
- Domain policy from stakeholders (e.g., institutional/regulatory constraints for this project).
- Edge cases discovered during implementation or QA.

## Outputs
- Authoritative rule list referenced by backend validation logic and test cases.

## Rules
- Each business rule gets a stable ID (e.g., `RULE-001`) and must state the *consequence* of violation, not just the constraint.
- Conflicting rules must be resolved and documented here before implementation proceeds — never resolved silently in code.
- Rules discovered mid-implementation (e.g., "actually, this field is required only for role X") must be added here retroactively.

## Checklist
- [ ] Rule has a stable ID and clear trigger condition.
- [ ] Rule has a defined consequence/enforcement point.
- [ ] Rule is cross-referenced from the requirement(s) it supports.

## Notes
_No business rules captured yet — pending product scope definition._

## Rule Log

| ID | Rule | Consequence | Status |
|----|------|-------------|--------|
| — | — | — | — |
