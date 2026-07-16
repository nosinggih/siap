# Design System Components

## Purpose
Catalog reusable UI components so the Frontend Developer builds *with* the system instead of duplicating it.

## Responsibilities
- Owned by: **Design System** role; consumed by **Frontend Developer**.

## Inputs
- Recurring UI patterns identified across features.
- Proposals from Frontend Developer when an existing component doesn't fit.

## Outputs
- A component catalog (name, purpose, states, props/variants) that is the source of truth for what already exists.

## Rules
- A component is added here *before* or *at the same time* it's built in code — the catalog and the implementation must not drift.
- Every component must document its states (default, hover, focus, disabled, error, loading where applicable).
- Breaking changes to a shared component must be flagged here with what consumes it, to avoid silent regressions elsewhere.

## Checklist
- [ ] Component has a stated purpose and when (not) to use it.
- [ ] All interactive states are documented.
- [ ] Accessibility notes included (focus order, ARIA roles where relevant).

## Notes
_No components cataloged yet — pending first UI implementation work._

## Component Catalog

| Component | Purpose | States | Status |
|-----------|---------|--------|--------|
| — | — | — | — |
