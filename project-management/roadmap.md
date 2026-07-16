# Roadmap

## Purpose
Provide a high-level, time-ordered view of where the product is headed so every role can align their work to the same horizon.

## Responsibilities
- Owned by: **Project Manager**, in collaboration with **Business Analyst** (priorities) and **Software Architect** (feasibility).
- Kept current at least once per milestone or when priorities materially shift.

## Inputs
- Business goals from `/business/requirements.md`
- Architectural constraints from `/architecture/architecture.md`
- Backlog items from `/project-management/backlog.md`

## Outputs
- Quarter/phase-level roadmap consumed by all roles for planning.
- Signal to `/project-management/milestones.md` for concrete milestone breakdown.

## Rules
- Roadmap items are outcomes ("Enable X for users"), not implementation tasks.
- Every roadmap item must trace back to a business goal or a documented technical necessity (e.g., a decision in `/architecture/decisions.md`).
- Do not remove completed items — move them to a "Shipped" section instead of deleting, to preserve history.

## Checklist
- [ ] Every active roadmap item has an owner role.
- [ ] Every roadmap item links to relevant requirements or ADRs.
- [ ] Roadmap reviewed against backlog for drift at each milestone boundary.

## Notes
_No roadmap items defined yet — this project is at the framework-initialization stage. Populate once the first product scope is agreed with the Business Analyst._
