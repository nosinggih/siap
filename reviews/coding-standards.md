# Coding Standards

## Purpose
Define cross-cutting code quality expectations that apply regardless of layer (backend/frontend).

## Responsibilities
- Owned by: **Code Reviewer**, in collaboration with **Software Architect**.

## Inputs
- Language/framework conventions once the stack is chosen (`/architecture/decisions.md`).

## Outputs
- A shared baseline referenced by `/backend/rules.md`, `/frontend/rules.md`, and `/reviews/review-checklist.md`.

## Rules
- Prefer clarity over cleverness; code is read far more often than it's written.
- No dead code, commented-out code blocks, or unused variables left in committed changes.
- Functions/modules do one thing; if a change description needs "and," consider splitting it.
- No premature abstraction — duplicate simple logic is better than the wrong shared abstraction.
- Every non-obvious decision in code gets a short comment explaining *why*, not *what*.
- Security: no command/SQL/HTML injection vectors, no secrets in source, validate all external input.

## Checklist
- [ ] No dead/commented-out code.
- [ ] No unexplained "why" left implicit where it matters.
- [ ] No obvious security anti-patterns (injection, hardcoded secrets, missing auth checks).
- [ ] Change is scoped to what was asked, not bundled with unrelated cleanup.

## Notes
_Language-specific style rules (formatting, linting) should be added here once the tech stack is chosen._
