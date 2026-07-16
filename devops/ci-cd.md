# CI/CD

## Purpose
Define the automated pipeline that takes a commit from proposed change to verified, deployable artifact.

## Responsibilities
- Owned by: **DevOps Engineer**.

## Inputs
- Test strategy from `/testing/strategy.md`.
- Coding standards from `/reviews/coding-standards.md`.

## Outputs
- A pipeline definition that enforces quality gates automatically rather than relying on manual discipline.

## Rules
- No change reaches `main`/production without passing automated tests and lint/style checks.
- Pipeline failures block merge — they are not advisory.
- Pipeline configuration lives in version control alongside the code it builds, not in a UI-only config.

## Checklist
- [ ] Pipeline runs tests defined in `/testing/strategy.md`.
- [ ] Pipeline enforces standards from `/reviews/coding-standards.md`.
- [ ] Pipeline config is version-controlled.
- [ ] Failing pipeline blocks merge/deploy.

## Notes
_No pipeline configured yet — pending tech-stack and hosting decisions._

## Pipeline Stages
1. Lint / static analysis
2. Unit tests
3. Integration tests
4. Build artifact
5. Deploy to target environment
