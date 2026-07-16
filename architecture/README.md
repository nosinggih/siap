# SIAP Architecture & Design Documentation

Welcome to the SIAP (Sistem Informasi Akuntansi Pemerintah) financial management system. This folder contains all architectural decisions, API contracts, and project structure.

---

## What's in This Folder?

### Core Architecture Files

1. **`ARCHITECTURE_SUMMARY.md`** ← **START HERE**
   - 15-minute overview for entire team
   - Key principles, stack decisions, common pitfalls
   - Quick reference for "how do I...?" questions
   - Read this first before diving into details

2. **`architecture.md`**
   - High-level system design (components, data flow)
   - Non-functional requirements (performance, security, audit)
   - Tech stack justification
   - Deployment & infrastructure notes
   - **For:** Understanding "why" behind the shape of the system

3. **`decisions.md`** (Architecture Decision Records)
   - ADR-001 through ADR-007: detailed reasoning for each major decision
   - Context, alternatives considered, consequences
   - References to business requirements
   - **For:** Understanding the trade-offs and rationale behind each choice

4. **`api-design.md`**
   - Complete REST API specification
   - Every endpoint documented: request/response/errors
   - Error handling standards
   - Authentication & authorization per endpoint
   - **For:** Backend & Frontend developers during implementation

5. **`decisions-log.md`**
   - Quick-reference table of all cross-cutting decisions
   - Impact per role (Backend, Frontend, DevOps, QA)
   - Developer checklist ("before you code")
   - Index of all ADRs
   - **For:** Quick lookup during daily development

---

## How These Connect to Business Documents

```
Business Requirements (requirements.md)
├─ REQ-001: Users must create journals
│
├─→ Architecture Decisions (decisions.md)
│   └─ ADR-006: Multi-entry balance validation
│
├─→ API Contract (api-design.md)
│   └─ POST /journals: Create journal endpoint
│
└─→ Implementation (Backend code)
    └─ JournalService.validate_balance()
```

**Every architecture decision traces back to a business requirement or rule.** No decisions made in a vacuum.

---

## For Different Roles

### Backend Developers

**Read in order:**
1. `ARCHITECTURE_SUMMARY.md` (15 min)
2. `decisions.md` → ADR-002, ADR-003, ADR-005, ADR-006 (30 min)
3. `api-design.md` → Your specific endpoints (1 hr)

**Then:**
- Use **Multi-Database Routing (ADR-002)** — middleware auto-routes to correct DB
- Use **Soft Deletes (ADR-003)** — add SoftDeletes trait to Models
- Implement **Audit Logging (ADR-005)** — emit events, captured by listeners
- Implement **Balance Validation (ADR-006)** — backend enforces (not frontend)

**Reference during coding:**
- `decisions-log.md` → DEC-001 through DEC-007 section for your role
- `api-design.md` → Your endpoint spec (request/response/errors)

---

### Frontend Developers

**Read in order:**
1. `ARCHITECTURE_SUMMARY.md` (15 min)
2. `decisions.md` → ADR-007 (5 min)
3. `api-design.md` → Endpoints you consume (1 hr)

**Then:**
- Use **REST API (ADR-007)** — standard HTTP methods, JSON payloads
- Build forms that validate client-side, but expect backend validation errors
- Implement year selector tied to tenant routing (middleware handles DB switch)
- Use soft-delete UI patterns: "Archive" button instead of "Delete"

**Reference during coding:**
- `api-design.md` → Exact endpoint URL, request shape, response shape
- `decisions-log.md` → DEC-007 for API conventions

---

### QA / Testing Engineers

**Read in order:**
1. `ARCHITECTURE_SUMMARY.md` (15 min)
2. `/business/requirements.md` → All REQ-* requirements
3. `/business/business-rules.md` → All RULE-* constraints

**Then:**
- Map each test case to a REQ-* or RULE-*
- Test backend validation (balance, access control, period locks)
- Test audit logging (verify mutations logged with correct data)
- Test multi-database routing (verify year-switch works)
- Test soft deletes (deleted entities not in reports, but in audit log)

**Checklists:**
- `decisions-log.md` → "Cross-Cutting Checklist for New Features" section

---

### DevOps Engineers

**Read in order:**
1. `ARCHITECTURE_SUMMARY.md` (15 min)
2. `architecture.md` → "Deployment & Infrastructure" section
3. `decisions.md` → ADR-002, ADR-004 (database multi-year strategy)

**Then:**
- Provision MySQL databases per fiscal year: `keuangan_2024`, `keuangan_2025`, `keuangan_2026`
- Run Laravel migrations on all databases
- Configure `tahun_aktif` in app config / `.env`
- Monitor audit_log table growth (shouldn't decrease)
- Alert on database connectivity issues
- Archive old databases after retention period

**Reference during deployment:**
- `ARCHITECTURE_SUMMARY.md` → "Deployment Checklist" section
- `decisions-log.md` → DEC-002 (database routing)

---

### Project Managers / Stakeholders

**Read:**
1. `ARCHITECTURE_SUMMARY.md` (15 min) — Understand tech stack & key constraints
2. `/business/requirements.md` → Understand scope & features

**Then:**
- Features must map to REQ-* in `/business/requirements.md`
- No scope changes without updating requirements doc
- Architecture decisions logged in `decisions.md` — no silent changes

**For timeline estimates:** Ask developers about ADR-001-007 implementation effort (already decided), not re-debating stack choices.

---

## Development Workflow

### When Starting a New Feature

1. **Add requirement:** Document in `/business/requirements.md` (REQ-NNN)
2. **Add business rule:** If needed, document in `/business/business-rules.md` (RULE-NNN)
3. **Design API:** Add endpoint spec to `api-design.md`
4. **Implement backend:** Use `/architecture/api-design.md` as contract
5. **Implement frontend:** Use same contract from `/architecture/api-design.md`
6. **Test:** Map test cases to REQ-*/RULE-*

### When Making an Architectural Decision

1. **Identify the question:** What's ambiguous?
2. **Write ADR:** Use template from `decisions.md`
3. **Propose to Architect:** Present alternatives & trade-offs
4. **Approve:** Once approved, add to `decisions-log.md`
5. **Implement & document:** Code must match ADR
6. **Code review:** Verify implementation against ADR

### When Code Diverges from Architecture

**NEVER silently ignore architecture.** Instead:
1. **Propose change:** Write ADR explaining why original decision doesn't work
2. **Get approval:** Present to Architect + relevant leads
3. **Update docs:** Change architecture docs to match new reality
4. **Log it:** Add ADR superseding the old decision

---

## Key Decisions at a Glance

| # | Decision | Implication | ADR |
|---|----------|------------|-----|
| **DEC-001** | Laravel + MySQL (no migration) | Existing stack, no tech change risk | ADR-001 |
| **DEC-002** | Year-based DB routing via middleware | All queries auto-route; no hardcoding | ADR-002 |
| **DEC-003** | Soft deletes only (no hard deletes) | Audit trail preserved, restore possible | ADR-003 |
| **DEC-004** | Old-year DBs read-only; write only to active year | Cross-year reports efficient, old data immutable | ADR-004 |
| **DEC-005** | Append-only audit logging with data diff | Full compliance + reconstruction capability | ADR-005 |
| **DEC-006** | Balance validation at backend | RULE-001 enforced, not bypassable | ADR-006 |
| **DEC-007** | REST API style (JSON, stateless) | Standard, testable, easy to document | ADR-007 |

---

## FAQ

**Q: Where do I find the endpoint I need to implement?**  
A: `api-design.md` → search for HTTP method + path (e.g., "POST /journals")

**Q: How do I know if a user can access this data?**  
A: Check RULE-019 + `/architecture/api-design.md` → each endpoint lists auth requirements

**Q: Can I hard-delete a record?**  
A: No. Soft-delete only (ADR-003). Update `deleted_at = NOW()`.

**Q: I need to query the 2024 data from 2025 app. How?**  
A: Use `DB::connection('keuangan_2024')->table(...)->...` raw queries. No Eloquent Models.

**Q: The API contract says POST returns 201. Should I return 200?**  
A: No. Match the contract. Status codes matter for client-side logic.

**Q: Why multiple databases instead of one database with a year column?**  
A: See ADR-002 alternatives section. Year-per-DB isolates data, prevents accidental cross-year queries.

**Q: Who approves architecture changes?**  
A: Software Architect. Propose ADR, present alternatives, get approval before implementing.

**Q: Is this architecture final?**  
A: It's solid for MVP. Future decisions (e.g., token auth, event sourcing) will be added as ADR-008+. Nothing is forever frozen.

---

## Document Index

### Architecture & Design
- `ARCHITECTURE_SUMMARY.md` — Quick-start guide (read first)
- `architecture.md` — Component design & system structure
- `decisions.md` — Architecture Decision Records (ADR-001 ~ ADR-007)
- `api-design.md` — REST API specification (all endpoints)
- `decisions-log.md` — Cross-cutting decisions quick-reference

### Business Documents (in parent directory)
- `/business/requirements.md` — Functional requirements (REQ-001 ~ REQ-040)
- `/business/business-rules.md` — Business constraints (RULE-001 ~ RULE-020)
- `/business/workflows.md` — User workflows (WF-001 ~ WF-012)

### Code (to be created)
- `/backend/app/Services/` — Business logic services
- `/backend/app/Http/Controllers/` — API controllers
- `/backend/database/migrations/` — Schema migrations
- `/frontend/src/components/` — UI components
- `/tests/` — Unit, integration, e2e tests

---

## How to Update These Documents

### When Requirements Change
1. Update `/business/requirements.md` (add/modify REQ-*)
2. Update `architecture.md` if system design affected
3. Update `api-design.md` if API changes
4. Update `decisions-log.md` if cross-cutting impact

### When Architecture Evolves
1. Write new ADR in `decisions.md` (ADR-008, ADR-009, ...)
2. Supersede old ADR if changed (don't delete)
3. Update `architecture.md` with new components
4. Add to `decisions-log.md` (new row in DEC-* table)
5. Ensure decision traces to REQ-* or RULE-*

### Never
- Delete or modify past ADRs (supersede instead)
- Commit code that diverges from architecture (update docs first)
- Hide decisions in code (document publicly in ADRs)
- Assume architecture; always reference these docs

---

## Getting Started

### Day 1 (New Developer)
1. Read `ARCHITECTURE_SUMMARY.md` (15 min)
2. Read your role's section above (15 min)
3. Read `/business/requirements.md` first 10 lines (2 min)
4. Read `/business/workflows.md` WF-001 (5 min)

### Day 2 (Pick a Feature)
1. Find the REQ-* in `requirements.md`
2. Find the RULE-* in `business-rules.md`
3. Find the WF-* in `workflows.md`
4. Find the endpoint in `api-design.md`
5. Implement against that contract

### Before First Commit
1. Verify code matches ADR-* decisions (especially ADR-001 ~ ADR-007)
2. Verify API matches `api-design.md` contract
3. Verify business rules validated (backend, not frontend)
4. Run tests; verify audit logging works
5. Request code review

---

## Support & Questions

- **Architecture decisions:** Contact Software Architect
- **API contracts:** Check `api-design.md`; ask Backend Lead if unclear
- **Business requirements:** Check `/business/requirements.md`; ask Product Owner if ambiguous
- **Deployment:** Check `architecture.md` or `decisions-log.md` → DEC-002
- **Workflows:** Check `/business/workflows.md`

---

## Version History

| Date | Author | Changes |
|------|--------|---------|
| 2025-02-01 | Software Architect | Initial architecture: ADR-001~007, API design, system layout |

---

**Last Updated:** 2025-02-01  
**Status:** ✅ Ready for development

For the latest version, check the project repository.
