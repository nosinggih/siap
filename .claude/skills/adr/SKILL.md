---
name: adr
description: Use when a non-trivial technical decision needs to be made or recorded — choosing a library/framework, changing a data model, picking an architectural pattern. Produces a properly formatted entry in /architecture/decisions.md instead of an undocumented ad hoc choice.
---

# Architecture Decision Record Workflow

Use this whenever a technical decision is significant enough that someone will later ask "why did we do it this way?" — new dependencies, framework/library choices, data model changes, breaking API changes, or any decision with real trade-offs.

## Steps

1. Read `/architecture/decisions.md` for the ADR template and existing numbering.
2. State the **Context**: what problem needs solving, and what constraints apply (reference `/business/requirements.md` or `/architecture/architecture.md` where relevant).
3. List real **Alternatives considered** — at least the obvious ones — and why each was rejected. A decision with no rejected alternatives is usually under-researched.
4. State the **Decision** plainly.
5. State the **Consequences**: what trade-offs does this decision accept? What does it foreclose?
6. Append the new ADR to `/architecture/decisions.md` with the next sequential ID (`ADR-001`, `ADR-002`, ...), status `Accepted` (or `Proposed` if it needs sign-off first).
7. If the decision changes the system's current shape, also update `/architecture/architecture.md` so it stays truthful.
8. If the decision supersedes a previous ADR, mark the old one `Superseded by ADR-###` — never edit its original decision text.

## Rules
- Never silently make a significant technical decision inside a code change without an ADR — the decision becomes unreviewable and unexplainable later.
- Do not delete or rewrite past ADRs to reflect new thinking; supersede instead, preserving history.
