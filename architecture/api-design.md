# API Design

## Purpose
Define the conventions and contracts for how the frontend, backend, and any external consumers communicate.

## Responsibilities
- Owned by: **Software Architect**; implemented by **Backend Developer**; consumed by **Frontend Developer**.

## Inputs
- Data and workflow needs from `/business/workflows.md`.
- System boundaries from `/architecture/architecture.md`.

## Outputs
- API contract (endpoints, payloads, error shapes) that backend implements against and frontend codes against — reducing integration guesswork.

## Rules
- API contracts are documented *before* parallel frontend/backend implementation begins, so both sides can work independently.
- Breaking changes to an existing endpoint require a version bump or an explicit migration note here, not a silent change.
- Error responses follow one consistent shape across the whole API.

## Checklist
- [ ] Endpoint has method, path, request shape, response shape, and error cases documented.
- [ ] Naming conventions are consistent with the rest of the API.
- [ ] Auth/permission requirements are stated per endpoint.

## Notes
_No API surface defined yet — pending architecture and tech-stack decisions. Populate this once `/architecture/architecture.md` establishes the backend framework and communication style (REST/GraphQL/RPC)._

## Conventions
- **Style:** _TBD (REST / GraphQL / RPC) — record the choice as an ADR in `/architecture/decisions.md`._
- **Auth:** _TBD._
- **Versioning:** _TBD._

## Endpoints

| Method | Path | Purpose | Auth | Status |
|--------|------|---------|------|--------|
| — | — | — | — | — |
