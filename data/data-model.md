# Data Model

## Purpose
Describe the core entities, their relationships, and how they support reporting — the analytics-facing counterpart to the backend's implementation-level schema.

## Responsibilities
- Owned by: **Data Analyst**, in collaboration with **Software Architect**/**Backend Developer** who own the implementation schema.

## Inputs
- Entities implied by `/business/workflows.md` and `/business/business-rules.md`.
- Reporting needs from `/data/reporting.md`.

## Outputs
- A conceptual entity-relationship model that keeps reporting/analytics needs in sync with what the backend actually stores.

## Rules
- This is a conceptual model (entities, relationships, key attributes) — not a literal DB schema; the literal schema lives in backend migrations.
- Any entity needed for reporting that doesn't exist in the backend model must be flagged back to the Architect/Backend Developer, not worked around with ad hoc queries.

## Checklist
- [ ] Every entity needed for a report in `/data/reporting.md` is represented here.
- [ ] Relationships (1:1, 1:many, many:many) are stated.
- [ ] Divergence from the actual backend schema is flagged, not silently tolerated.

## Notes
_No data model defined yet — pending product scope and architecture decisions._
