# Business Workflows

## Purpose
Describe the end-to-end processes users and the organization follow, independent of UI or technical implementation.

## Responsibilities
- Owned by: **Business Analyst**.
- Consumed by: **Frontend Developer** (to sequence UI steps) and **QA/Testing Engineer** (to build realistic test scenarios).

## Inputs
- Stakeholder interviews and observed current processes (including manual/offline steps being digitized).

## Outputs
- Workflow diagrams/step lists referenced by `/frontend/notes.md` and `/testing/test-plan.md`.

## Rules
- Workflows are described as actor → action → outcome steps, including exception/error paths, not just the happy path.
- Any workflow step that maps to a business rule must reference its ID from `/business/business-rules.md`.
- Changes to a live workflow must note what the previous behavior was, so downstream roles understand what changed.

## Checklist
- [ ] Workflow includes actors, trigger, steps, and outcome.
- [ ] Exception/edge-case paths are documented, not just the happy path.
- [ ] Referenced business rules are linked by ID.

## Notes
_No workflows captured yet — pending product scope definition._
