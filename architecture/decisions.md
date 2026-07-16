# Architecture Decision Records (ADRs)

## Purpose
Preserve the *reasoning* behind significant technical decisions so future contributors don't have to reverse-engineer "why" from code.

## Responsibilities
- Owned by: **Software Architect**.
- Any role may propose an ADR; the Architect accepts, rejects, or supersedes it.

## Inputs
- Architectural questions raised during implementation (e.g., "which database?", "sync or async processing?").
- Constraints from `/business/requirements.md` and `/architecture/architecture.md`.

## Outputs
- A permanent, numbered log of decisions that `/architecture/architecture.md` and implementation should conform to.

## Rules
- Use the standard ADR format: Context, Decision, Consequences, Status (Proposed/Accepted/Superseded).
- Never edit or delete a past ADR to change its decision — supersede it with a new one and link back.
- Every ADR gets a sequential ID: `ADR-001`, `ADR-002`, ...

## Checklist
- [x] ADR states the problem/context, not just the answer.
- [x] ADR lists real alternatives considered and why they were rejected.
- [x] ADR consequences (trade-offs accepted) are explicit.

---

## ADR-001: Continue Laravel + MySQL Stack

**Status:** Accepted.

**Context:** SIAP is a financial management system for UNS (and potential adoption by other PTN-BH institutions). An existing application built with Laravel has been running in production since 2022 with accumulated improvements and data. The project has existing SDM (Software Development Team) experienced with the stack. Retirement of the old app requires new app to read/write data from 2022-2026.

**Decision:** Continue with **Laravel framework** (PHP) and **MySQL** database as primary tech stack. Do not migrate to Node.js, Python, or other stacks. This ensures:
- Minimal context-switching for existing team
- Preserve institutional knowledge of the Laravel codebase
- Reduce risk of data loss during migration
- Leverage existing hosting infrastructure

**Alternatives considered:**
1. **Rewrite in Node.js/TypeScript** — Rejected because: (a) team skill shift required, (b) higher risk of regressions, (c) MySQL driver maturity less proven in previous context.
2. **Use Laravel + PostgreSQL** — Rejected because: (a) existing MySQL infrastructure and team expertise, (b) DECIMAL(18,2) works equally well in both, (c) no requirement forces PostgreSQL.
3. **Greenfield with modern stack (Django/FastAPI + PostgreSQL)** — Rejected because: (a) OI-003 defers migration decision, (b) no business driver for rewrite, (c) existing data must be accessible.

**Consequences:**
- Team operates in familiar stack; onboarding faster.
- Laravel Eloquent ORM provides abstraction for multi-database tenant routing (ADR-002).
- PHP numeric handling requires care for financial calculations (DECIMAL(18,2), no rounding).
- MySQL version 5.7+ supports JSON columns for flexible master-data config (REQ-039).
- Cross-database reads (old years) must use MySQL's `USE database_name` within transactions, not connection pooling.

**Traces to:** REQ-001, REQ-033, REQ-039.

---

## ADR-002: Tenant-Aware Database Routing by Active Year

**Status:** Accepted.

**Context:** SIAP must support multiple fiscal years (2024, 2025, 2026, ...), each with separate MySQL database (`keuangan_2024`, `keuangan_2025`, `keuangan_2026`). All units within a fiscal year share one database (no database-per-unit sharding). Users must be able to switch active year; the system must then point to that year's database.

**Decision:** Implement **year-based dynamic database routing** via a Laravel middleware:

1. **Config storage:** Store active fiscal year (`tahun_aktif`) in a system-wide setting table (in a shared config database or in each year's DB).
2. **Middleware implementation:** Create `TenantRouterMiddleware` that runs on every request:
   - Read user's session or request header for target year (default: `tahun_aktif` from config).
   - Validate year against allowed list (2024, 2025, 2026).
   - Dynamically set Eloquent connection to `keuangan_{year}`.
   - All subsequent Model queries automatically route to correct DB.
3. **Connection pooling:** Use Laravel's `config/database.php` with template connection `keuangan-template` that interpolates year at runtime.
4. **Fallback:** If year cannot be determined (edge case), default to current fiscal year.

**Alternatives considered:**
1. **Single database with `tahun_anggaran` column in every table** — Rejected because: (a) requires schema changes across old app, (b) violates tenant isolation, (c) increases complexity of cross-year reports.
2. **Connection pool per year (hardcoded)** — Rejected because: (a) not scalable if future years are added, (b) requires app restart to add new year.
3. **Hybrid: Connection pool for active year, cross-DB query for old years** — Accepted as secondary decision (ADR-004).

**Consequences:**
- Every Model query implicitly routes to correct database; developers don't need year-awareness in service code.
- Middleware error handling critical — must fail gracefully if year unknown.
- Tests must mock middleware or use separate test databases per year.
- Performance: dynamic connection routing adds ~1-2ms per request (acceptable for 200-500 journals/day, REQ-037).
- No application restart needed to activate a new fiscal year.

**Traces to:** REQ-033, REQ-034.

---

## ADR-003: Soft Delete Pattern with `deleted_at` Column

**Status:** Accepted.

**Context:** RULE-006 and RULE-007 require conditional immutability: journals can be edited/deleted only if period is OPEN and not yet processed by other actors. RULE-009 mandates that journals are never permanently deleted from historical record. Audit compliance and reconstruction of historical state require a complete record of all changes.

**Decision:** Use **Laravel's SoftDeletes trait** (Eloquent) to implement logical deletion:

1. **Schema:** Add `deleted_at` timestamp column (nullable) to affected tables:
   - `journals`
   - `tax_forms`
   - `coas` (for RULE-018 — inactive COAs not deletable)
   - Any entity that must remain audit-traceable

2. **Behavior:**
   - `soft_delete()` sets `deleted_at = NOW()`, does not remove row.
   - `restore()` sets `deleted_at = NULL`, reactivates entity.
   - Default query scopes exclude soft-deleted rows (except in audit/archive reports).

3. **Audit trail:** Soft-deleted entities remain visible in audit_log table with action `DELETED`. Report generators explicitly exclude soft-deleted journals from financial summaries (but include them in reconciliation reports).

4. **Backward compat:** Existing hard-delete operations in old app are read-only from new app (ADR-004). New app never hard-deletes; only soft-deletes.

**Alternatives considered:**
1. **Hard delete with archive table** — Rejected because: (a) two-table query logic complexity, (b) Laravel SoftDeletes is standard pattern.
2. **Event sourcing (store every state change)** — Rejected because: (a) overengineering for current audit needs, (b) can be added later if requirements expand.
3. **Logical flags (is_active boolean)** — Rejected because: `deleted_at` provides timestamp of deletion (RULE-020 audit compliance).

**Consequences:**
- `Journal::all()` excludes soft-deleted; must use `Journal::withTrashed()` to include them.
- Unique constraints (e.g., nomor_jurnal UNIQUE) must account for soft deletes — use scoped uniqueness (`withoutTrashed()`).
- Reporting queries must explicitly exclude soft-deleted to avoid phantom entries in financial statements.
- Foreign key constraints should **not** cascade-delete; instead, prevent delete if referenced (RULE-018 for COAs).

**Traces to:** RULE-006, RULE-009, RULE-018, REQ-007, REQ-029.

---

## ADR-004: Cross-Database Read for Historical Year Data

**Status:** Accepted.

**Context:** Old application (2022-2026) is being retired but data must remain accessible and updatable from the new app. Full data migration is deferred (OI-003) and risky. New app must read balances/summaries from old year databases for:
- REQ-022: Comparative reports (current year vs. previous year)
- REQ-034: Historical data access (read-only from current-year app context)
- RULE-011: Summary tables in active-year DB for efficient queries

**Decision:** Implement **read-only cross-database access** pattern:

1. **Active year database:** All writes (journals, tax forms, periods) go to current fiscal year DB (`keuangan_2025` when 2025 is active).
2. **Old year databases:** Accessible read-only from new app context:
   - Direct queries via `DB::connection('keuangan_2024')->table('journals')->...` for cross-DB reads.
   - No ORM (Eloquent) for old years — use raw queries to prevent accidental writes.
3. **Summary tables in active DB:** Instead of querying old DB directly for reports, populate a `summary_previous_year` table in active DB:
   - Triggered when year is activated (Admin: "Set 2025 active").
   - Pulls final balances from `keuangan_2024` and caches in `keuangan_2025`.
   - Used by REQ-022 (comparative reports) for performance.
4. **Error handling:** If old DB is unreachable, report generation falls back to summary table (graceful degradation).

**Alternatives considered:**
1. **Full data migration** — Rejected because: (a) OI-003 defers decision, (b) high risk of data loss/corruption, (c) old app may still need read access.
2. **Federated queries (MySQL FEDERATED engine)** — Rejected because: (a) deprecated, poor performance, (b) requires both DBs on same MySQL server.
3. **ETL pipeline to pull old data** — Partially accepted; this is the summary-table approach (hybrid).

**Consequences:**
- Old year data is immutable and never updated from new app (new transactions only go to active year).
- Performance: summary table queries fast; direct old-DB queries slower (acceptable for EOY close, not real-time).
- Schema divergence possible if old and new apps evolve differently — mitigated by read-only enforcement.
- Developers must use `DB::connection()` explicitly for old-year reads (prevents accidental writes via Eloquent).

**Traces to:** REQ-022, REQ-033, REQ-034, RULE-011.

---

## ADR-005: Audit Log Implementation — Action + Data Diff

**Status:** Accepted.

**Context:** RULE-020 and REQ-038 mandate immutable audit logging of all user activities (including impersonation, WF-009). Audit log must capture "what happened" and "what changed" for reconstruction and compliance. App errors must also be logged separately for debugging.

**Decision:** Implement **dual-log pattern**:

1. **Audit Log table** (`audit_logs` in active DB):
   - Schema: `id`, `user_id`, `action` (CREATE/UPDATE/DELETE/APPROVE/CLOSE_PERIOD), `entity_type` (journal/period/tax_batch), `entity_id`, `old_value` (JSON), `new_value` (JSON), `timestamp`, `ip_address`, `impersonation_by` (if applicable).
   - Populated via Laravel event listeners (e.g., `Journal::created`, `Journal::updated`).
   - Append-only: no UPDATE/DELETE on audit_log itself; only INSERT.
   - For impersonation (WF-009): `impersonation_by` field traces real user; action is still recorded.

2. **Application Error Log** (file-based or external, e.g., Laravel's storage/logs/):
   - Separate from audit log; captures exceptions, failed validations, sync errors.
   - Schema: `timestamp`, `level` (ERROR/WARNING/INFO), `context` (which request/user), `message`, `stack_trace`.
   - Rotated daily; archived periodically.

3. **No modification ever:** Both logs are write-once. DevOps monitoring detects/alerts if logs are modified externally (RULE-020).

**Alternatives considered:**
1. **Single event-sourced log** — Rejected because: (a) overengineering, (b) audit_log is sufficient for compliance, (c) error log is operational, not audit.
2. **External logging service (e.g., ELK, DataDog)** — Acceptable future upgrade; ADR-005 is self-contained within DB for now.
3. **Quarterly archives to cold storage** — Acceptable future optimization; not required for MVP.

**Consequences:**
- Every mutation (create/update/delete journal) generates 1-2 audit entries; ~1-1000 new audit rows per day (manageable).
- JSON diff in `old_value`/`new_value` enables full reconstruction of state at any point (audit trail completeness).
- Developers must wire up event listeners; easy to forget (code review responsibility).
- Audit table can grow large over years; archival strategy needed post-MVP.
- Impersonation auditing adds `impersonation_by` tracking; session must carry admin identity.

**Traces to:** RULE-020, REQ-026, REQ-038.

---

## ADR-006: Multi-Entry Journal Validation — Σ Debet = Σ Kredit

**Status:** Accepted.

**Context:** RULE-001 and REQ-002 mandate that journals must balance: sum of all Debet entries = sum of all Kredit entries. Unlike traditional double-entry (2 lines), SIAP journals can have 3+ lines (e.g., one Kredit split across three Debet accounts, or vice versa). Balance validation must occur at:
- Frontend (UX feedback, real-time calculation).
- Backend (business-logic enforcement, cannot bypass via API).

**Decision:** Implement **multi-entry balance validator**:

1. **Data model:**
   - `Journal` table: id, nomor_jurnal, nomor_bukti, tanggal, keterangan, status, user_id, unit_id, periodo, created_at, updated_at, deleted_at.
   - `JournalEntry` table (detail baris): id, journal_id, akun_coa_id, jenis (DEBET/KREDIT), nominal (DECIMAL(18,2)).

2. **Validation logic** (JournalService.validate_balance):
   ```
   Σ JournalEntry.nominal WHERE jenis = DEBET
   must equal
   Σ JournalEntry.nominal WHERE jenis = KREDIT
   ```
   - Performed in service layer before save.
   - Returns error if unbalanced; shows delta (difference) for user correction.

3. **Precision:** All calculations use DECIMAL(18,2); no intermediate rounding. Comparison is exact (`==`, not `~=`).

4. **Frontend validation:** Real-time sum calculation in form; visual feedback (red/green balance indicator).

**Alternatives considered:**
1. **Allow unbalanced on creation, enforce on finalize** — Rejected because: (a) complicates workflow, (b) RULE-001 is clear: balance on save.
2. **Manual balance entry (user confirms)** — Rejected because: (a) error-prone, (b) system must validate, not trust user.

**Consequences:**
- JournalEntry must always be created with its parent Journal; no orphaned entries.
- Journal foreign key in JournalEntry is NOT ON DELETE CASCADE (soft-delete handles cleanup).
- API payload for journal creation must include entries array; atomicity ensured via transaction.
- Testing balance validation requires fixtures with 3+ entries per journal.

**Traces to:** RULE-001, RULE-002, REQ-002.

---

## ADR-007: RESTful API Style with Stateless Authentication

**Status:** Accepted.

**Context:** SIAP is a web application (REQ-040: desktop-first). Frontend and backend must communicate over HTTP. API must support concurrent requests from multiple users, each with different unit assignments (RULE-019) and role-based access control.

**Decision:** Use **REST API** style (not GraphQL or RPC) with the following conventions:

1. **Stateless auth:** No session-based auth (except for Laravel session cookie for legacy compat). Use:
   - Session cookie (Laravel default) for initial MVP.
   - Token-based (Bearer token) for future mobile/external integrations (defer to ADR-008 if needed).

2. **Endpoint conventions:**
   - `POST /api/journals` — Create journal.
   - `GET /api/journals/{id}` — Retrieve single journal.
   - `PUT /api/journals/{id}` — Update journal (if unlocked).
   - `DELETE /api/journals/{id}` — Soft-delete journal (if unlocked).
   - `GET /api/journals` — List journals (filtered by user unit + period).
   - Bulk operations: `POST /api/tax-batches/{batchId}/finalize` (state transition).

3. **Response shape:** Consistent error/success envelope:
   ```json
   {
     "success": true,
     "data": { ... },
     "meta": { "count": 10, "page": 1 },
     "errors": null
   }
   ```

4. **Pagination:** Default 50 items/page; query param `?page=2&limit=100`.

5. **Filtering:** Query params for common filters (unit, period, status, etc.); pass to service layer.

**Alternatives considered:**
1. **GraphQL** — Rejected because: (a) overkill for CRUD-heavy forms, (b) team familiarity with REST, (c) frontend doesn't require N+1 optimization.
2. **RPC style** — Rejected because: (a) less intuitive for CRUD, (b) no caching benefits.

**Consequences:**
- API is familiar and easy to test (curl, Postman, automated tests).
- Versioning strategy needed if breaking changes occur (use `/api/v1/`, `/api/v2/`, not header-based).
- Response documentation is simple (OpenAPI/Swagger).
- Frontend can use standard fetch/axios; no GraphQL client required.

**Traces to:** REQ-040.

---

## ADR-000: Template (for future ADRs)

**Status:** Template — not a real decision.

**Context:** What problem are we solving? What constraints apply?

**Decision:** What did we decide?

**Alternatives considered:** What else was on the table, and why was it rejected?

**Consequences:** What trade-offs does this decision accept?
