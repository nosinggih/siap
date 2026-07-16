# Frontend Rules

## Purpose
Codify how frontend/UI code should be written so implementation stays consistent and matches the Design System.

## Responsibilities
- Owned by: **Frontend Developer**; enforced by **Code Reviewer**.
- Must stay consistent with `/design-system/principles.md`, `/design-system/components.md`, and `/architecture/api-design.md`.

## Inputs
- Design tokens/components from `/design-system/`.
- API contracts from `/architecture/api-design.md`.
- Workflows from `/business/workflows.md`.

## Outputs
- UI implementation that is responsive, accessible, and visually consistent with the Design System.

## Rules
- Reuse Design System components/tokens before creating new ones; new components are proposed back into `/design-system/components.md`, not created ad hoc in a feature.
- All interactive UI must be keyboard-navigable and screen-reader accessible (WCAG AA baseline) unless explicitly scoped out.
- Layouts must be responsive across the breakpoints defined in `/design-system/style-guide.md`.
- No hardcoded colors/spacing/typography — use design tokens.
- Loading, empty, and error states are implemented for every data-driven view, not just the happy path.

## Checklist
- [ ] Uses Design System components/tokens, not one-off styles.
- [ ] Keyboard and screen-reader accessible.
- [ ] Responsive at all defined breakpoints.
- [ ] Loading/empty/error states implemented.
- [ ] Matches the workflow steps in `/business/workflows.md`.

## Notes
_Tech stack not yet chosen — see `/architecture/decisions.md`. Revisit to add framework-specific conventions once decided._
