---
name: new-feature
description: Use when starting a new feature or significant change from scratch. Walks the request through the full software-house pipeline (Business Analyst → Software Architect → Backend/Frontend Developer → Code Reviewer → QA) so every role's documentation stays in sync instead of jumping straight to code.
---

# New Feature Workflow

This skill sequences a new feature through the project's defined roles so no step gets skipped. Follow it in order; do not jump ahead to implementation before the earlier steps produce their artifacts.

## Steps

1. **Business Analyst** — clarify the request. Add/update entries in `/business/requirements.md` (with a `REQ-###` ID), `/business/business-rules.md` (if new rules are implied), and `/business/workflows.md`. Stop and ask the user if anything is genuinely ambiguous — do not guess at business intent.

2. **Software Architect** — check the requirement against `/architecture/architecture.md`. If it fits the existing architecture, proceed. If it requires a new component, dependency, or pattern, record the decision as a new ADR in `/architecture/decisions.md` first. Update `/architecture/api-design.md` if new endpoints/contracts are needed.

3. **Design System** (UI-facing features only) — check `/design-system/components.md` for reusable components before any new UI is designed. Add new components/tokens if genuinely needed.

4. **Backend Developer** — implement server-side logic per `/backend/rules.md`, enforcing the business rules from step 1. Run `/backend/checklist.md` before moving on.

5. **Frontend Developer** — implement UI per `/frontend/rules.md`, using the Design System from step 3. Run `/frontend/checklist.md` before moving on.

6. **QA/Testing Engineer** — write test cases in `/testing/test-plan.md` tracing back to the `REQ-###`/`RULE-###` IDs from step 1. Verify against the Definition of Done.

7. **Code Reviewer** — review the resulting diff against `/reviews/review-checklist.md` and `/reviews/coding-standards.md` before considering the feature complete.

## Rules
- Do not skip step 1 — implementation without a documented requirement produces undocumented, unreviewable business logic.
- Each step's artifacts (the doc updates) matter as much as the code — a feature isn't "done" if the docs weren't updated.
- If a step reveals the previous step's output was wrong or incomplete, go back and fix it there rather than patching around it downstream.
- For small, obvious fixes (typos, trivial bugs), it's reasonable to compress steps 1-3 informally, but never skip Code Review.
