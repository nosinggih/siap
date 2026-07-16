# Decisions Log

**Quick reference for cross-cutting architectural and project decisions that impact all roles.**

---

## Format

| Decision ID | Title | Impact | Status | ADR Ref | Owner |
|-------------|-------|--------|--------|---------|-------|
| DEC-001 | Continue Laravel + MySQL | All (Backend, Frontend, DevOps) | Accepted | ADR-001 | Architect |

---

## Decisions

### DEC-001: Continue Laravel + MySQL Stack

| Field | Value |
|-------|-------|
| **Title** | Continue Laravel + MySQL stack; do not migrate to other tech |
| **Impact** | All roles (Backend, Frontend, DevOps, QA) |
| **Status** | Accepted ✅ |
| **ADR** | ADR-001 |
| **Owner** | Software Architect |
| **What this means** | - Backend continues PHP + Laravel framework<br>- Database remains MySQL 5.7+<br>- Existing hosting infrastructure reused<br>- No Node.js, Python, or Java rewrite<br>- SDM familiarity leveraged |
| **For Backend Dev** | Use Laravel best practices: Eloquent ORM, migrations, artisan commands |
| **For Frontend Dev** | Frontend can be Blade (server-render) or SPA (Vue/React); API is RESTful JSON |
| **For DevOps** | No new runtime environments; deploy Laravel app to existing PHP infrastructure |
| **For QA** | Test against Laravel app; no tech-specific edge cases from platform switch |

---

### DEC-002: Year-Based Dynamic Database Routing (Tenant-Aware)

| Field | Value |
|-------|-------|
| **Title** | Dynamically route requests to `keuangan_YYYY` database based on `tahun_aktif` |
| **Impact** | Backend (middleware), Frontend (year selector context) |
| **Status** | Accepted ✅ |
| **ADR** | ADR-002 |
| **Owner** | Software Architect |
| **What this means** | - All databases named `keuangan_2024`, `keuangan_2025`, `keuangan_2026`, etc.<br>- System-wide `tahun_aktif` config (stored in DB or env)<br>- Middleware injects correct DB connection before each request<br>- No manual year parameter in queries; transparent to services |
| **For Backend Dev** | - Implement `TenantRouterMiddleware`<br>- All Model queries auto-route to correct DB<br>- Schema migration script runs on all year DBs<br>- DO NOT hardcode database names in code |
| **For Frontend Dev** | - UI shows year selector (e.g., dropdown "2025")<br>- Selected year sent in session or request header<br>- Middleware handles routing; frontend doesn't worry about DB |
| **For DevOps** | - Provision MySQL databases for each fiscal year: `keuangan_2024`, `2025`, `2026`<br>- Each DB has identical schema (via migrations)<br>- No connection pooling per year; handled by middleware |
| **For QA** | - Test journal creation in different fiscal years<br>- Verify year-switching doesn't corrupt data<br>- Verify old-year data read-only (no writes) |

---

### DEC-003: Soft Delete Pattern with `deleted_at` Column

| Field | Value |
|-------|-------|
| **Title** | Use Laravel SoftDeletes; set `deleted_at` timestamp instead of hard-delete |
| **Impact** | Backend (Models, queries), Frontend (show/hide deleted entities), QA (audit verification) |
| **Status** | Accepted ✅ |
| **ADR** | ADR-003 |
| **Owner** | Software Architect |
| **What this means** | - No row is permanently deleted from DB<br>- `deleted_at` nullable timestamp column on journal, tax_form, coa tables<br>- Default queries exclude soft-deleted (via Eloquent scope)<br>- Soft-deleted still visible in audit log and admin views<br>- Restoration (undo delete) possible by nullifying `deleted_at` |
| **For Backend Dev** | - Add SoftDeletes trait to relevant Models<br>- Use `withTrashed()` in audit/archive queries<br>- Unique constraints use `withoutTrashed()` scope<br>- Document which entities are soft-deletable |
| **For Frontend Dev** | - Show "Archive" button instead of "Delete" in UI<br>- Confirm: "This will archive the record — you can restore it later."<br>- Admin view can toggle "Show archived" to see soft-deleted |
| **For QA** | - Verify soft-deleted not in reports<br>- Verify soft-deleted IN audit trail<br>- Verify restore (un-archive) works |
| **Audit requirement** | RULE-009: Journals never permanently deleted from historical record |

---

### DEC-004: Cross-Database Read for Historical Year Data

| Field | Value |
|-------|-------|
| **Title** | Read old fiscal-year data from separate DB (e.g., `keuangan_2024`); write only to active year DB |
| **Impact** | Backend (queries), DevOps (DB backup), QA (multi-year testing) |
| **Status** | Accepted ✅ |
| **ADR** | ADR-004 |
| **Owner** | Software Architect |
| **What this means** | - Active year DB: all writes (journals, periods, tax forms)<br>- Old-year DBs: read-only; no updates from new app<br>- Summary tables in active DB cache old-year balances (optimization)<br>- Reports can compare current year vs. previous year efficiently<br>- No data migration; old app data remains in old DB |
| **For Backend Dev** | - Use `DB::connection('keuangan_2024')` for explicit old-year reads<br>- DO NOT use Eloquent (Model) for old years; use raw queries only<br>- Never attempt writes to old years<br>- Cache summary balances in active DB on year-switch<br>- Handle error if old DB unreachable (fallback to cached summary) |
| **For Frontend Dev** | - Year selector shows current + previous years only<br>- Can't select future years (validation on backend)<br>- Comparative reports show side-by-side current vs. previous |
| **For DevOps** | - Backup all year DBs; old DBs can be read-only replicas<br>- Monitor cross-DB query performance (may be slow)<br>- Archive very old DBs after N years (retention policy TBD) |
| **For QA** | - Test comparative reports with data from 2+ years<br>- Verify old-year data never modified<br>- Verify summary refresh on year switch |
| **Business requirement** | REQ-022 (comparative reports), REQ-034 (historical data access) |

---

### DEC-005: Audit Log Implementation — Action + Data Diff

| Field | Value |
|-------|-------|
| **Title** | Append-only audit logging: track who did what, when, and what changed |
| **Impact** | Backend (event listeners), DevOps (log storage), QA (audit trail testing) |
| **Status** | Accepted ✅ |
| **ADR** | ADR-005 |
| **Owner** | Software Architect |
| **What this means** | - `audit_logs` table: id, user_id, action, entity_type, entity_id, old_value (JSON), new_value (JSON), timestamp, ip_address, impersonation_by<br>- Every CREATE/UPDATE/DELETE emits event → captured by listener → inserted into audit_log<br>- Audit log itself is immutable (no UPDATE/DELETE on audit_log)<br>- Separate `error.log` file for app exceptions (operational, not audit)<br>- Impersonation tracked: `impersonation_by` field shows real admin |
| **For Backend Dev** | - Wire up event listeners to Model create/update/delete events<br>- In listener, call `AuditLogger.log_action()`<br>- Capture old_value (before) and new_value (after) as JSON<br>- For impersonation: include admin ID in audit entry (RULE-017)<br>- Ensure audit_log insert fails gracefully (don't block transaction) |
| **For Frontend Dev** | - No direct frontend dependency; backend handles<br>- Admin can view audit log via GET /audit-log endpoint |
| **For DevOps** | - Monitor audit_log table growth (may grow 500-1000 rows/day)<br>- Implement log rotation for error.log (daily, 30-day retention)<br>- Alert if audit_log table size shrinks (data loss detection)<br>- Archive audit logs to cold storage after 1-2 years |
| **For QA** | - Verify all mutations logged (create journal, update period, etc.)<br>- Verify impersonation entries show admin identity<br>- Verify audit_log matches actual system state |
| **Compliance requirement** | RULE-020 (immutable audit trail), REQ-038 |

---

### DEC-006: Multi-Entry Journal Validation (Σ Debet = Σ Kredit)

| Field | Value |
|-------|-------|
| **Title** | Validate journal balance at backend (not just frontend); allow 3+ entry lines |
| **Impact** | Backend (validation logic), Frontend (real-time calc), API contract |
| **Status** | Accepted ✅ |
| **ADR** | ADR-006 |
| **Owner** | Software Architect |
| **What this means** | - Journal can have N entries (2, 3, 4, ...)<br>- Each entry is DEBET or KREDIT<br>- Sum of all DEBET = Sum of all KREDIT (to exact 2 decimals)<br>- Validation happens at backend, before save (RULE-001)<br>- Frontend shows real-time balance, disables save if unbalanced |
| **For Backend Dev** | - Service: `JournalService.validate_balance(entries)`<br>- Calculate `sum_debet` and `sum_kredit` using DECIMAL(18,2)<br>- Return error if unequal; include delta for user feedback<br>- No rounding; exact comparison (==)<br>- Test with 3-4 entry combinations |
| **For Frontend Dev** | - Form allows adding multiple Debet rows (and multiple Kredit rows)<br>- Real-time sum display: "Debet: 1,000,000 | Kredit: 1,000,000 | Status: ✓ Balance"<br>- Disable "Simpan" button if unbalanced<br>- On save attempt, show error with delta |
| **For QA** | - Test 2-entry journal (standard)<br>- Test 3-entry (1 Kredit, 2 Debet; or 2 Kredit, 1 Debet)<br>- Test with odd decimal values (e.g., 1234567.89)<br>- Verify API rejects unbalanced POSTs |
| **Business requirement** | RULE-001 (balance), REQ-002 (double-entry) |

---

### DEC-007: RESTful API Style (JSON, Stateless)

| Field | Value |
|-------|-------|
| **Title** | Use REST API style (not GraphQL/RPC) with JSON payloads and stateless auth |
| **Impact** | Backend (endpoint design), Frontend (API client), QA (integration testing) |
| **Status** | Accepted ✅ |
| **ADR** | ADR-007 |
| **Owner** | Software Architect |
| **What this means** | - Endpoints: POST /journals, GET /journals, PUT /journals/{id}, DELETE /journals/{id}<br>- Payloads: JSON (request + response)<br>- Auth: Laravel session (stateful for MVP); Bearer token deferred<br>- Error response: consistent `{ success, data, errors }` shape<br>- Versioning: `/api/` (implicit v1); future `/api/v2/` if breaking changes<br>- No GraphQL; no RPC-style calls |
| **For Backend Dev** | - Use Laravel ResourceController pattern<br>- Define OpenAPI/Swagger doc for each endpoint (in code or separate file)<br>- Follow REST naming: resources (journals, periods, reports), CRUD verbs (POST, GET, PUT, DELETE)<br>- State transitions via POST (e.g., POST /periods/close, POST /tax-batches/{id}/finalize)<br>- Error responses always include field-level errors in `errors` object |
| **For Frontend Dev** | - Use standard fetch/axios HTTP client<br>- No GraphQL setup needed<br>- Parse response: check `success` flag, extract `data` or `errors`<br>- Retry logic on 5xx (exponential backoff)<br>- Session cookie auto-managed by browser |
| **For QA** | - Test endpoints with curl, Postman, or automated tests<br>- Verify error responses have correct status code + error shape<br>- Test pagination (page, per_page params)<br>- Test filtering (unit_id, status, etc.) |
| **DevOps/Future** | - API easily documentable (Swagger UI)<br>- Mobile app can consume same API (add Bearer token auth later, ADR-008) |

---

## Cross-Cutting Checklist for New Features

**Before implementing any feature, verify:**

- [ ] Follows REST API style (DEC-007): POST, GET, PUT, DELETE methods; JSON payloads
- [ ] Respects multi-database routing (DEC-002): uses active year DB transparently
- [ ] Implements audit logging (DEC-005): entity changes tracked in audit_log
- [ ] Uses soft deletes (DEC-003) if entity can be deleted: sets `deleted_at`
- [ ] Validates business rules (RULE-*) at backend, not just frontend
- [ ] Respects access control (RULE-019): checks user unit assignment
- [ ] Documented in `/architecture/api-design.md` before implementation
- [ ] Traces back to requirement in `/business/requirements.md`

---

## Project Milestones (Tentative)

| Milestone | Features | Est. ADRs | Owner |
|-----------|----------|----------|-------|
| **MVP Phase 1** | Journal CRUD (Umum, Transaksi), Period close/reopen, Basic reports (BKU, Neraca) | ADR-001~007 | Backend/Frontend |
| **MVP Phase 2** | Tax forms (Pajak Terima/Setor/Batch), Sync MEMO from SIreva | ADR-001~007 | Backend/Frontend |
| **Phase 3** | Advanced reports (LPJ, Daya Serap, Comparative), Audit log UI | ADR-001~007 + ADR-008 (token auth) | Backend/Frontend/QA |
| **Phase 4** | Mobile app, Report builder, Coretax integration | ADR-008, ADR-009 (TBD) | Backend/Mobile Dev |

---

## ADR Index

| ADR | Title | Status | Impact | See |
|-----|-------|--------|--------|-----|
| ADR-001 | Continue Laravel + MySQL Stack | ✅ Accepted | All roles | DEC-001 |
| ADR-002 | Tenant-Aware Database Routing by Year | ✅ Accepted | Backend, DevOps | DEC-002 |
| ADR-003 | Soft Delete Pattern with `deleted_at` | ✅ Accepted | Backend, Frontend, QA | DEC-003 |
| ADR-004 | Cross-Database Read for Historical Year Data | ✅ Accepted | Backend, DevOps, QA | DEC-004 |
| ADR-005 | Audit Log Implementation | ✅ Accepted | Backend, DevOps, QA | DEC-005 |
| ADR-006 | Multi-Entry Journal Validation | ✅ Accepted | Backend, Frontend, QA | DEC-006 |
| ADR-007 | RESTful API Style (JSON, Stateless) | ✅ Accepted | Backend, Frontend, QA | DEC-007 |
| ADR-008 | Token-Based Auth (Bearer) | 📅 Planned | Backend, Mobile Dev | — |
| ADR-009 | Event Sourcing (if needed) | ❓ TBD | — | — |

---

## Contacts & Escalation

| Role | Decision Authority | Contact |
|------|-------------------|---------|
| **Software Architect** | All technical decisions, ADRs, API contracts | — |
| **Backend Lead** | Implementation approach, DB schema, performance | — |
| **Frontend Lead** | UI/UX, form validation, API client design | — |
| **DevOps Lead** | Infrastructure, database provisioning, deployment | — |
| **Project Manager** | Timeline, scope, priority | — |

---

## How to Propose a New Decision

1. **Identify the problem:** What's ambiguous or needs deciding?
2. **Write an ADR:** Use template from `/architecture/decisions.md`
3. **Present to Architect:** Discuss alternatives and consequences
4. **Approve:** Once approved, add to this log as DEC-NNN
5. **Implement & document:** Update relevant architecture files
6. **Review:** Code review ensures implementation matches ADR

---

**Last Updated:** 2025-02-01
**Reviewed by:** Software Architect
