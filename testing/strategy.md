# Testing Strategy

## Purpose
Define the overall approach to testing: what layers get tested, by whom, and with what tools.

## Responsibilities
- Owned by: **QA/Testing Engineer**, in collaboration with **Backend/Frontend Developers**.

## Inputs
- Requirements from `/business/requirements.md`, business rules from `/business/business-rules.md`.
- Architecture from `/architecture/architecture.md` (what layers exist to test).

## Outputs
- A test pyramid/approach that `/testing/test-plan.md` and `/testing/regression-checklist.md` are built from.

## Rules
- Every business rule and requirement must be traceable to at least one test case.
- Unit tests are the developer's responsibility as part of implementation; QA owns integration/end-to-end and exploratory testing.
- A bug found in QA gets a regression test added before it's marked resolved — see `/testing/regression-checklist.md`.
- Flaky tests are fixed or removed, never ignored indefinitely.

## Checklist
- [ ] Test layers defined (unit / integration / e2e / manual/exploratory).
- [ ] Ownership per layer is clear.
- [ ] Tooling choice recorded (and justified as an ADR if non-trivial).

## Notes
_No stack chosen yet — testing tools should be selected alongside the tech stack decision in `/architecture/decisions.md`._

## Test Pyramid
- **Unit** — owned by developers, run on every change.
- **Integration** — owned by developers + QA, run on every change to shared boundaries.
- **End-to-end** — owned by QA, run before milestone sign-off.
- **Exploratory/manual** — owned by QA, run before release.
