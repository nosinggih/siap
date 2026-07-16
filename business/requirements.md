# Business Requirements

## Purpose
Capture *what* the business needs, independent of how it will be technically implemented.

## Responsibilities
- Owned by: **Business Analyst**.
- Consumed by: **Software Architect** (to design), **QA/Testing Engineer** (to derive test plans), **Project Manager** (to prioritize).

## Inputs
- Stakeholder conversations, domain knowledge, regulatory/institutional constraints.
- Existing workflows described in `/business/workflows.md`.

## Outputs
- Functional requirements consumed by `/architecture/architecture.md` and `/testing/test-plan.md`.

## Rules
- Requirements describe outcomes and constraints, never a specific implementation ("users must be able to export reports as PDF", not "add a PDF button using library X").
- Every requirement gets a stable ID (e.g., `REQ-001`) so it can be referenced from architecture, tests, and code review.
- Ambiguous requirements must be flagged as `[NEEDS CLARIFICATION]` rather than guessed at.

## Checklist
- [ ] Requirement has a stable ID.
- [ ] Requirement is testable (QA can write a test case against it).
- [ ] Requirement does not prescribe implementation details.

## Notes
_No requirements captured yet. This document should be the first one populated once the actual application scope (the "SIAP" product) is defined with the stakeholder._

## Requirements Log

| ID | Requirement | Source | Status |
|----|-------------|--------|--------|
| — | — | — | — |
