# Style Guide

## Purpose
Define the concrete design tokens — color, typography, spacing, breakpoints — that back every component.

## Responsibilities
- Owned by: **Design System** role.

## Inputs
- Brand/institutional identity constraints (if any), accessibility contrast requirements.

## Outputs
- Token values consumed directly by `/frontend/rules.md` implementation (no hardcoded values in code).

## Rules
- All colors must meet WCAG AA contrast ratios for their intended use (text vs. background).
- Tokens are named semantically (`color-danger`, `space-md`) not by raw value (`color-red`, `space-16px`), so the underlying value can change without renaming call sites.
- Any new token must justify why an existing one doesn't already cover the need.

## Checklist
- [ ] Token is named semantically.
- [ ] Color tokens pass WCAG AA contrast where used for text.
- [ ] Breakpoints cover mobile, tablet, and desktop at minimum.

## Notes
_No tokens defined yet — pending branding/identity input and first UI work._

## Color Tokens
_TBD._

## Typography
_TBD._

## Spacing Scale
_TBD._

## Breakpoints
| Name | Min width |
|------|-----------|
| mobile | 0px |
| tablet | 768px |
| desktop | 1024px |
