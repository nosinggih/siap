# Infrastructure

## Purpose
Document what infrastructure the system runs on and how it's configured/provisioned.

## Responsibilities
- Owned by: **DevOps Engineer**.

## Inputs
- Non-functional requirements (scale, availability, budget) from `/business/requirements.md`.
- Architecture from `/architecture/architecture.md`.

## Outputs
- Infrastructure documentation that lets the environment be reproduced or reasoned about without access to a live console.

## Rules
- Infrastructure is defined as code wherever the target platform supports it; manual console changes are documented as debt, not left undocumented.
- Secrets/credentials are never stored in this repo — reference the secret manager/vault used and how to access it, not the values.
- Every external service dependency (email, storage, third-party API) is listed with what happens if it's unavailable.

## Checklist
- [ ] Provisioning is defined as code, or the manual gap is explicitly flagged.
- [ ] No secrets committed — only references to where they live.
- [ ] External dependencies listed with failure-mode notes.

## Notes
_No infrastructure provisioned yet — pending hosting/tech-stack decisions._

## External Dependencies
| Service | Purpose | Failure mode |
|---------|---------|--------------|
| — | — | — |
