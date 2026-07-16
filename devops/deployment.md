# Deployment

## Purpose
Document how the application gets from committed code to a running environment.

## Responsibilities
- Owned by: **DevOps Engineer**.

## Inputs
- Architecture from `/architecture/architecture.md` (what needs to run where).
- CI/CD pipeline from `/devops/ci-cd.md`.

## Outputs
- A reproducible, documented deployment process any team member (or agent) can follow or automate.

## Rules
- Deployment steps must be scripted/automated wherever possible — manual steps are documented as a temporary state, not a permanent process.
- Every environment (dev/staging/prod) has its config differences explicitly documented, not tribal knowledge.
- Rollback procedure is documented alongside the deploy procedure — deploying without a rollback plan is not acceptable for prod.

## Checklist
- [ ] Deployment steps are scripted or explicitly marked as a manual interim step.
- [ ] Environment differences are documented.
- [ ] Rollback procedure exists and is documented.

## Notes
_No deployment target chosen yet — pending architecture/tech-stack decisions._

## Environments
| Environment | Purpose | URL | Notes |
|-------------|---------|-----|-------|
| dev | — | — | — |
| staging | — | — | — |
| prod | — | — | — |
