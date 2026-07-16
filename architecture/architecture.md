# System Architecture

## Purpose
Describe the current system architecture: major components, how they interact, and why the shape is what it is.

## Responsibilities
- Owned by: **Software Architect**.
- Consumed by: **Backend Developer**, **Frontend Developer**, **DevOps Engineer**, **Code Reviewer**.

## Inputs
- Business requirements from `/business/requirements.md`.
- Non-functional constraints (scale, compliance, budget, hosting environment).

## Outputs
- Architecture diagrams/descriptions that constrain and guide implementation.
- Feeds `/architecture/decisions.md` when a choice needs to be justified, and `/architecture/api-design.md` for interface contracts.

## Rules
- Architecture describes the *current* system truthfully; if implementation diverges from this doc, this doc must be updated — it is not aspirational.
- Any deviation a developer needs to make from this document must first go through the Architect and be logged as an ADR in `/architecture/decisions.md`.
- Prefer boring, well-understood technology unless a documented requirement justifies novelty.

## Checklist
- [ ] Every major component has a stated responsibility and owner boundary.
- [ ] Data flow between components is documented.
- [ ] Non-functional requirements (scale, security, availability) are addressed, not just the happy-path shape.

## Notes
_No architecture defined yet. This should be authored once business requirements establish the actual scope of the "SIAP" application (tech stack, hosting target, integrations)._

## Tech Stack
_TBD — to be decided and recorded as an ADR in `/architecture/decisions.md` once requirements are known._

## Component Overview
_TBD._
