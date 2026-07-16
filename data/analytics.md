# Analytics

## Purpose
Define what user/system behavior needs to be measurable, and why, before instrumentation is built.

## Responsibilities
- Owned by: **Data Analyst**.

## Inputs
- Business goals from `/business/requirements.md` (what decisions the business needs to make).
- Workflows from `/business/workflows.md` (what events actually occur).

## Outputs
- An event/metric spec that `/backend/rules.md` implementation instruments against.

## Rules
- Every tracked event/metric must tie back to a decision it informs — no tracking "just in case."
- PII in analytics data must be explicitly flagged and justified; default to not collecting it.
- Metric definitions are precise enough that two people computing them by hand would get the same number.

## Checklist
- [ ] Metric/event ties to a stated business decision.
- [ ] PII handling is explicit.
- [ ] Definition is unambiguous (units, time window, inclusion/exclusion rules).

## Notes
_No analytics defined yet — pending product scope._
