# Design System Principles

## Purpose
State the guiding UI/UX principles that every screen and component should reflect, independent of specific tokens or components.

## Responsibilities
- Owned by: **Design System** role, in collaboration with **Frontend Developer**.

## Inputs
- Business workflows from `/business/workflows.md` (who uses this and how).
- Accessibility and usability best practices.

## Outputs
- Principles that guide `/design-system/components.md` and `/design-system/style-guide.md`, and that `/frontend/rules.md` references.

## Rules
- Principles must be specific enough to make a real design decision easier — vague statements ("be user-friendly") are not useful and should be rejected.
- Accessibility (WCAG AA) is a baseline principle, not an optional add-on.
- Any new principle must not contradict an existing one; if priorities conflict, state the tie-breaker explicitly.

## Checklist
- [ ] Principle is specific enough to resolve a real design disagreement.
- [ ] Principle doesn't contradict an existing one (or the conflict is explicitly resolved).

## Notes
_No principles ratified yet — draft below is a starting point, to be confirmed once the product's actual users and context are known._

## Draft Principles
1. **Clarity over cleverness** — a user should never have to guess what an element does.
2. **Consistency over novelty** — reuse existing patterns before inventing new ones.
3. **Accessible by default** — WCAG AA is the floor, not a stretch goal.
4. **Responsive by default** — every view must work from mobile to desktop.
5. **Fast feedback** — every user action gets an immediate, visible response (loading, success, error).
