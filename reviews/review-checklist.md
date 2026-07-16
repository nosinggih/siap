# Review Checklist

## Purpose
Give the Code Reviewer a consistent, repeatable checklist so review quality doesn't depend on memory or mood.

## Responsibilities
- Owned by: **Code Reviewer**.

## Inputs
- `/reviews/coding-standards.md`, `/backend/rules.md`, `/frontend/rules.md`.
- The specific diff/PR under review.

## Outputs
- Review feedback that is specific, actionable, and consistently applied across contributors.

## Rules
- Every finding cites *why* it's a problem (concrete failure scenario), not just a style preference.
- Correctness and security issues block approval; style nits do not (they're suggestions).
- Review checks against the actual rules docs, not the reviewer's personal taste — if a rule isn't written down, propose adding it rather than enforcing an unwritten standard.

## Checklist
- [ ] Correctness: does the change do what it claims, including edge cases?
- [ ] Security: input validation, auth checks, no secrets, no injection vectors.
- [ ] Consistency: follows `/backend/rules.md` or `/frontend/rules.md` as applicable.
- [ ] Tests: adequate coverage per `/testing/strategy.md`.
- [ ] No unrequested scope creep (matches `/reviews/coding-standards.md` on minimal diffs).
- [ ] Naming/readability: could a new team member follow this without extra explanation?

## Notes
_None yet._
