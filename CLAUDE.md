# SIAP — Project Guide for Claude Code

This repository is currently a **software-house framework**, not yet an application. It was scaffolded per `README.md` to establish role-separated documentation and workflows *before* any product code is written. The actual "SIAP" application scope has not yet been defined — see `/business/requirements.md`.

## How this project is organized

Each directory is owned by one role and documents that role's Purpose, Responsibilities, Inputs, Outputs, Rules, Checklist, and Notes. Directories do not overlap in ownership — see `/docs/glossary.md` for shared vocabulary and `/docs/decisions-log.md` for cross-cutting decisions.

| Directory | Owning role | Contents |
|---|---|---|
| `/project-management` | Project Manager | roadmap, milestones, backlog |
| `/business` | Business Analyst | requirements, business rules, workflows |
| `/architecture` | Software Architect | architecture, ADRs, API design |
| `/backend` | Backend Developer | backend rules, checklist, notes |
| `/frontend` | Frontend Developer | frontend rules, checklist, notes |
| `/design-system` | Design System | UI/UX principles, components, style guide |
| `/testing` | QA/Testing Engineer | test strategy, test plan, regression checklist |
| `/data` | Data Analyst | analytics, reporting, data model |
| `/devops` | DevOps Engineer | deployment, infrastructure, CI/CD |
| `/reviews` | Code Reviewer | review checklist, coding standards |
| `/docs` | Shared | glossary, cross-cutting decisions log |

## Agents

Each role has a corresponding agent definition in `.claude/agents/`, invocable via the Agent tool: `software-architect`, `backend-developer`, `frontend-developer`, `code-reviewer`, `design-system`, `qa-testing-engineer`, `business-analyst`, `data-analyst`, `devops-engineer`, `project-manager`. Each agent's system prompt points it at the docs it owns and the docs it must respect from other roles.

## Skills

- `/new-feature` — sequences a new feature request through the full role pipeline (BA → Architect → Design System → Backend/Frontend → QA → Review).
- `/adr` — records a significant technical decision as a proper Architecture Decision Record in `/architecture/decisions.md`.

## Working in this repo

- **Most docs are currently templates**, not filled-in specs — the product scope hasn't been defined yet. Before writing application code, the Business Analyst step (`/business/requirements.md`) needs real content.
- When implementing anything, consult the owning role's docs first (e.g., backend work reads `/backend/rules.md` and `/architecture/api-design.md`, not just the request itself).
- Don't mix role concerns in one doc — e.g., don't put implementation details in `/business/requirements.md`, and don't put business rules only in backend code without also recording them in `/business/business-rules.md`.
- Significant technical decisions get an ADR (`/adr` skill) rather than being made silently inline.
- This structure is meant to be extended, not reorganized — add new agents/skills/docs under the existing directories rather than restructuring them.
