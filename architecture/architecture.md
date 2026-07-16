# System Architecture

## Purpose
Describe the current system architecture: major components, how they interact, and why the shape is what it is.

## Responsibilities
- Owned by: **Software Architect**.
- Consumed by: **Backend Developer**, **Frontend Developer**, **DevOps Engineer**, **Code Reviewer**.

## Inputs
- Business requirements from `/business/requirements.md`.
- Business rules from `/business/business-rules.md`.
- Workflow definitions from `/business/workflows.md`.
- Non-functional constraints (scale, compliance, audit).

## Outputs
- Architecture diagrams/descriptions that constrain and guide implementation.
- Feeds `/architecture/decisions.md` when a choice needs to be justified, and `/architecture/api-design.md` for interface contracts.

## Rules
- Architecture describes the *current* system truthfully; if implementation diverges from this doc, this doc must be updated — it is not aspirational.
- Any deviation a developer needs to make from this document must first go through the Architect and be logged as an ADR in `/architecture/decisions.md`.
- Prefer boring, well-understood technology unless a documented requirement justifies novelty.

## Checklist
- [x] Every major component has a stated responsibility and owner boundary.
- [x] Data flow between components is documented.
- [x] Non-functional requirements (scale, security, availability) are addressed, not just the happy-path shape.

---

## Tech Stack

| Component | Technology | Rationale | ADR |
|-----------|------------|-----------|-----|
| **Backend Framework** | Laravel 9+ (PHP) | Existing stack, team expertise, Eloquent ORM for multi-DB routing | ADR-001 |
| **Database** | MySQL 5.7+ | Existing infrastructure, DECIMAL(18,2) precision, JSON column support for config | ADR-001 |
| **Frontend** | Laravel Blade (server-rendered) or SPA (Vue/React) | Desktop-first (REQ-040), form-heavy app; decision TBD per module | — |
| **Authentication** | Laravel Session + Cookie | Existing pattern; token-based (Bearer) deferred to future ADR | ADR-007 |
| **API Style** | REST + JSON | Standard, testable, integrates with frontend easily | ADR-007 |
| **File Storage** | Local filesystem or S3 | PDF uploads (RULE-005: max 5MB per journal) | — |
| **Job Queue** | Laravel Queue (Redis/Database driver) | Async sync (WF-012), report generation (optional) | — |
| **Logging** | Laravel Log (file-based) + custom audit table | Audit trail (RULE-020) + operational errors | ADR-005 |

---

## Component Overview

### High-Level System Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                        FRONTEND (UI)                        │
│  ├─ Journal Form (multi-entry, real-time balance calc)     │
│  ├─ Period/Year Selector (year-aware context)               │
│  ├─ Master Data Management (COA, MEMO, users)              │
│  ├─ Report Viewer + PDF download/print                      │
│  ├─ Admin Dashboard (period close, user mgmt)               │
│  └─ Audit Log Viewer (read-only)                            │
└──────────────────┬──────────────────────────────────────────┘
                   │ HTTP + JSON
     ┌─────────────▼──────────────┐
     │   API Gateway/Middleware   │
     │ ├─ Auth & Authorization    │
     │ ├─ TenantRouter (year→DB)  │
     │ ├─ Request Logging         │
     │ └─ Error Handling          │
     └─────────────┬──────────────┘
                   │
┌──────────────────▼───────────────────────────────────────────┐
│              BACKEND (Laravel Application)                   │
│                                                               │
│  Controllers (API Endpoints)                                 │
│  ├─ JournalController                                        │
│  │  ├─ POST   /journals (create)                            │
│  │  ├─ GET    /journals (list)                              │
│  │  ├─ GET    /journals/{id}                                │
│  │  ├─ PUT    /journals/{id} (update if unlocked)           │
│  │  └─ DELETE /journals/{id} (soft-delete if unlocked)      │
│  │                                                            │
│  ├─ PeriodController                                         │
│  │  ├─ POST /periods/close (admin only)                     │
│  │  └─ POST /periods/reopen (admin only)                    │
│  │                                                            │
│  ├─ TaxController                                            │
│  │  ├─ POST /tax-batches                                    │
│  │  ├─ PUT  /tax-batches/{id}                               │
│  │  └─ POST /tax-batches/{id}/finalize                      │
│  │                                                            │
│  ├─ ReportController                                         │
│  │  ├─ POST /reports/generate (BKU, LPJ, Neraca, etc)      │
│  │  ├─ POST /reports/{id}/upload-signed                    │
│  │  └─ GET  /reports (list with upload status)              │
│  │                                                            │
│  ├─ SyncController                                           │
│  │  └─ POST /sync/memo (manual trigger)                     │
│  │                                                            │
│  ├─ UserController (admin)                                   │
│  │  ├─ POST   /users (create)                               │
│  │  ├─ PUT    /users/{id}                                   │
│  │  └─ POST   /users/{id}/impersonate (admin only)          │
│  │                                                            │
│  └─ MasterDataController                                     │
│     ├─ POST   /coas (create account)                        │
│     ├─ PUT    /coas/{id}                                    │
│     └─ GET    /coas (list available for form)               │
│                                                               │
│  Services (Business Logic)                                   │
│  ├─ JournalService                                           │
│  │  ├─ create_journal(data) → validate balance (RULE-001)   │
│  │  ├─ update_journal(id, data) → check unlock conditions   │
│  │  ├─ soft_delete_journal(id) → RULE-006/007              │
│  │  └─ can_edit_journal(id) → period+actors check          │
│  │                                                            │
│  ├─ PeriodService                                            │
│  │  ├─ close_period(unit_id, month) → RULE-008             │
│  │  ├─ reopen_period(unit_id, month) → RULE-009            │
│  │  └─ is_period_open(unit_id, month) → query helper       │
│  │                                                            │
│  ├─ TaxService                                               │
│  │  ├─ create_pajak_setor(...) → check referenced terima   │
│  │  ├─ create_batch(...) → validate same pajak+month       │
│  │  └─ finalize_batch(batch_id) → status→FINAL             │
│  │                                                            │
│  ├─ AccessControl (RULE-019)                                │
│  │  ├─ user_can_access_unit(user_id, unit_id)              │
│  │  └─ scope_query_by_unit(query, unit_id)                 │
│  │                                                            │
│  ├─ ReportService                                            │
│  │  ├─ generate_bku(unit, period, format)                   │
│  │  ├─ generate_neraca_saldo(...)                           │
│  │  ├─ generate_lpj(...)                                    │
│  │  └─ generate_comparative(current_year, prev_year)        │
│  │                                                            │
│  ├─ SyncService                                              │
│  │  └─ sync_memo_from_sireva() → poll REST API              │
│  │                                                            │
│  └─ AuditLogger (ADR-005)                                    │
│     └─ log_action(user, entity, action, old, new) → table  │
│                                                               │
│  Models (Eloquent ORM)                                       │
│  ├─ User                                                     │
│  ├─ Journal (+ soft-delete, RULE-009)                       │
│  ├─ JournalEntry (detail baris: Debet/Kredit)              │
│  ├─ Period (OPEN/CLOSED per unit per month, RULE-008)      │
│  ├─ CoA (Chart of Accounts, RULE-018: no permanent delete)  │
│  ├─ Memo (synced from SIreva, RULE-016)                    │
│  ├─ TaxForm (Pajak Terima/Setor, RULE-012/013)             │
│  ├─ TaxBatch (Pajak consolidated, RULE-014/015)            │
│  ├─ Report (cached report metadata + upload status)         │
│  ├─ Unit (organizational boundary, RULE-019)                │
│  └─ AuditLog (immutable, RULE-020)                          │
│                                                               │
│  Middleware (request pipeline)                               │
│  ├─ TenantRouterMiddleware (ADR-002: year → DB)            │
│  ├─ AuthMiddleware (verify user + session)                  │
│  ├─ AuditLoggerMiddleware (capture request + response)      │
│  └─ ErrorHandlerMiddleware (consistent error response)      │
│                                                               │
└──────────────────┬───────────────────────────────────────────┘
                   │
    ┌──────────────┴──────────────────────────┐
    │                                           │
┌───▼──────────────────────┐    ┌─────────────▼──────────┐
│  MySQL Databases         │    │  External Services     │
│  (per fiscal year)        │    │                        │
│                           │    │  ┌──────────────────┐  │
│  ├─ keuangan_2024        │    │  │  SIreva API      │  │
│  │  ├─ journals          │    │  │ (REST endpoints) │  │
│  │  ├─ journal_entries   │    │  │ ├─ GET /memo     │  │
│  │  ├─ periods           │    │  │ │ (verified)      │  │
│  │  ├─ coas              │    │  │ └─ GET /pagu     │  │
│  │  ├─ tax_forms         │    │  └──────────────────┘  │
│  │  ├─ tax_batches       │    │                        │
│  │  ├─ audit_log         │    └────────────────────────┘
│  │  ├─ users             │
│  │  ├─ units             │
│  │  ├─ memos             │
│  │  └─ reports           │
│  │                       │
│  ├─ keuangan_2025 (active)
│  │  ├─ [same schema]     │
│  │  ├─ summary_2024 (ref)│  ← Cache of 2024 final balance
│  │  └─ ...               │
│  │                       │
│  └─ keuangan_2026        │
│     ├─ [same schema]     │
│     └─ ...               │
│                           │
└───────────────────────────┘
```

### Component Responsibilities

#### 1. **Frontend (Web UI)**
- **Responsibility:** Display forms, capture user input, real-time validation, render reports.
- **Technology:** Laravel Blade (server-rendered) or SPA (Vue/React) — decision per module.
- **Key workflows implemented:**
  - Journal entry form (WF-001, WF-002, WF-003, WF-004): Multi-entry balance calculation in-browser.
  - Period/year selector: Context-aware navigation (REQ-033).
  - Report viewer: PDF preview, download, upload signed copy.
  - Master data forms: COA, MEMO, user management (admin only).
  - Audit log viewer: Readonly display of actions.

#### 2. **API Gateway & Middleware Layer**
- **Responsibility:** Route requests, enforce auth, tenant routing, audit logging, error handling.
- **Key middleware:**
  - **TenantRouterMiddleware** (ADR-002): Reads `tahun_aktif` from session/config, sets Eloquent DB connection.
  - **AuthMiddleware:** Verify Laravel session; check role + unit permissions.
  - **AuditLoggerMiddleware:** Capture request method/endpoint/user/timestamp/IP for audit trail.
  - **ErrorHandlerMiddleware:** Convert exceptions to consistent JSON error response.

#### 3. **Backend Services (Business Logic)**
- **Responsibility:** Implement business rules, validate state transitions, coordinate multi-model operations.
- **Key services:**
  - **JournalService:** RULE-001 (balance validation), RULE-003 (account mapping), RULE-004/005 (numbering/upload).
  - **PeriodService:** RULE-008/009 (close/reopen per unit/month).
  - **TaxService:** RULE-013/014/015 (pajak terima→setor→batch lifecycle).
  - **AccessControl:** RULE-019 (unit-scoped access).
  - **ReportService:** Generate BKU, Neraca, LPJ, comparative reports (REQ-016~022).
  - **SyncService:** WF-012 (pull MEMO from SIreva when verified).
  - **AuditLogger:** ADR-005 (log all mutations + errors).

#### 4. **Models (Eloquent ORM)**
- **Responsibility:** Abstract database schema, provide query builder, enforce relationships.
- **Key models:**
  - **Journal:** Central entity; has many JournalEntries; references Period, User, Unit; soft-deleted.
  - **JournalEntry:** Detail baris (Debet/Kredit); child of Journal.
  - **Period:** Status (OPEN/CLOSED) per unit per month; controls edit/delete lock (RULE-008).
  - **CoA:** Chart of Accounts; immutable once used in journal (RULE-018).
  - **Memo:** Synced from SIreva; marked SUDAH_DIGUNAKAN when used (RULE-016).
  - **TaxForm:** Pajak Terima/Setor; status transitions (BELUM_DISETOR → SUDAH_DISETOR).
  - **TaxBatch:** Aggregation of Pajak Setor; status: DRAFT → FINAL (immutable when FINAL, RULE-015).
  - **AuditLog:** Append-only; records action + data diff (ADR-005).

#### 5. **Database Layer**
- **Responsibility:** Persistent storage, transactions, integrity constraints.
- **Strategy:**
  - **Multi-database (per fiscal year):** `keuangan_2024`, `keuangan_2025`, `keuangan_2026`, ...
  - **Tenant routing (ADR-002):** TenantRouterMiddleware dynamically connects to correct DB based on `tahun_aktif`.
  - **Cross-DB reads (ADR-004):** Old-year data read via `DB::connection('keuangan_2024')` (raw queries, no Eloquent).
  - **Summary table:** `summary_2024` in `keuangan_2025` caches final balances from previous year (optimization for REQ-022).
- **Schema inheritance:** All year DBs have identical schema (managed via migrations).

#### 6. **External Integration (SIreva API)**
- **Responsibility:** Pull verified MEMO/PAGU data.
- **Trigger:** Manual trigger (Admin) or scheduled job (daily/weekly).
- **Flow (WF-012):** Backend calls SIreva REST API → receives MEMO list → upserts local `memos` table.
- **Error handling:** Log failures; notify Admin; graceful degradation (MEMO list from previous sync available).

---

## Data Flow Walkthrough — Key Scenarios

### Scenario 1: User Creates Journal (WF-001, WF-002)

```
User Form (Frontend)
├─ Select journal type (Jurnal Umum / Transaksi / Pajak)
├─ Fill multiple entry rows (Debet/Kredit split)
├─ Real-time SUM calculation (frontend)
│  └─ if ΣDebet ≠ ΣKredit: disable Simpan button
├─ Upload PDF (optional, ≤5MB, PDF only)
└─ Click Simpan

API Request: POST /api/journals
├─ TenantRouterMiddleware: Connect to keuangan_2025 (active year)
├─ AuthMiddleware: Verify user, check unit assignment (RULE-019)
├─ JournalController.store()
│  └─ JournalService.create_journal()
│     ├─ Validate balance: Σ Debet = Σ Kredit (RULE-001)
│     ├─ Validate accounts in mapping (RULE-003)
│     ├─ Check period is OPEN for user's unit (RULE-008)
│     ├─ Generate nomor_jurnal (RULE-004)
│     ├─ Save Journal + JournalEntry rows
│     ├─ Save PDF to storage (RULE-005)
│     └─ Emit Journal.created event
│
├─ Event listener: AuditLogger
│  └─ Insert audit_log row: action=CREATE, entity=journal, new_value={...}
│
└─ Response: { success: true, data: { journal_id, nomor_jurnal } }
```

**Immutability checks (RULE-006, RULE-007):**
- If period is CLOSED: API returns 403 "Periode sudah ditutup, gunakan Jurnal Koreksi".
- If journal is referenced in Jurnal Koreksi: return 409 "Jurnal sudah dikoreksi, tidak dapat diubah".

---

### Scenario 2: Admin Closes Period (WF-006)

```
Admin Dashboard
├─ Select unit (Operasional / Aset / Keuangan)
├─ Select month (Januari / Februari / ...)
├─ Click "Tutup Periode"

API Request: POST /api/periods/close
├─ AuthMiddleware: Only ADMIN role
├─ PeriodService.close_period(unit_id, month)
│  ├─ Set Period.status = CLOSED
│  ├─ All subsequent writes to this unit+month blocked (RULE-008)
│  └─ Emit Period.closed event
│
├─ AuditLogger: Insert audit_log: action=CLOSE_PERIOD, unit, month
│
└─ Response: { success: true }

→ Frontend automatically blocks journal form for this unit/month
→ UI shows "Periode Tertutup — Gunakan Jurnal Koreksi untuk perubahan"
```

---

### Scenario 3: Compare Reports, Current Year vs. Previous (REQ-022)

```
User: Report → Comparative (Neraca Saldo)
├─ Active year: 2025
├─ Period: December
├─ System checks: summary_2024 exists in keuangan_2025

ReportService.generate_comparative()
├─ Query keuangan_2025.journals (current year, 2025)
│  └─ Join CoA, sum nominal per account
├─ Query keuangan_2025.summary_2024 (previous year reference)
│  └─ Join CoA, retrieve cached 2024 balances
├─ Combine results in side-by-side format
└─ Output PDF or Excel

→ Old-year data (2024) is read-only, never modified from new app
→ Never query keuangan_2024 directly at runtime (cached summary instead)
```

**If summary_2024 doesn't exist:**
- Report generation falls back to direct query to `keuangan_2024` (slow, but works).
- Admin notified: "Cache outdated, run yearly summary refresh".

---

### Scenario 4: Soft Delete & Audit Trail (RULE-009, RULE-020)

```
User: Delete Journal (before period close, unlocked)
├─ JournalService.soft_delete_journal(journal_id)
│  ├─ Set Journal.deleted_at = NOW()
│  ├─ Emit Journal.deleted event
│  └─ Return success

AuditLogger: Insert audit_log
├─ action: DELETE
├─ entity: journal
├─ entity_id: journal_123
├─ old_value: { nomor_jurnal, total_debet, ... }
├─ new_value: { deleted_at: NOW() }
└─ timestamp, user_id, ip_address

→ Journal.all() query excludes soft-deleted (default scope)
→ Journal::withTrashed() includes soft-deleted (for audit viewers)
→ Report queries explicitly exclude soft-deleted (financial accuracy)

Later, Auditor views audit log:
├─ Filters: user=Operator1, action=DELETE, entity=journal, period=May
└─ Sees: "Operator1 deleted journal J-2025-00123 on May 15 14:30"
   with old_value showing what was deleted
```

---

## Non-Functional Requirements

### REQ-037: Performance (200-500 journals/day, responsive)

**Approach:**
- **Indexing:** `journals(unit_id, periodo, created_at)`, `journal_entries(journal_id)`, `audit_log(user_id, created_at)`.
- **Query optimization:** N+1 prevention via Eloquent eager loading.
- **Caching:** Report summaries cached (invalidate on journal save).
- **Pagination:** Frontend lists journals 50 per page.
- **Async jobs:** Report generation queued (not blocking HTTP response).

**Acceptable latency:**
- Journal create/update: <1s.
- Report generation (BKU): <5s (PDF export).
- Audit log query: <500ms.

---

### REQ-038 & RULE-020: Audit Trail & Immutability

**Approach:**
- **Append-only design:** No UPDATE/DELETE on audit_log; only INSERT.
- **Database triggers (optional):** Prevent manual modification via SQL.
- **DevOps monitoring:** Alert if audit_log table size shrinks or data disappears.
- **Application logs:** Separate error log (errors.log) for operational debugging, not audit.

---

### RULE-019: Access Control (Unit-Scoped)

**Approach:**
- **Model scope:** Every query adds WHERE clause: `WHERE unit_id IN (user.assigned_units)`.
- **Service layer:** All queries run through `AccessControl.scope_query_by_unit()`.
- **API validation:** 403 Forbidden if user requests data outside assigned units.
- **Exception:** Admin, Operator Pusat, Auditor can access all units (no scoping).

---

## Deployment & Infrastructure

**Hosting:** Internal UNS infrastructure (on-premise or private cloud).
**CI/CD:** Git (internal repo) → GitHub Actions/GitLab CI → Deploy to production.
**Reverse proxy:** Nginx (TLS termination, rate limiting).
**Database backup:** Daily snapshots of all `keuangan_*` databases; retention 30 days.
**Monitoring:** Health checks on API endpoints, database connectivity, audit log health.

---

## Future Considerations (Not in MVP)

- **Mobile app:** Currently desktop-only (REQ-040); future mobile version requires API redesign (token auth).
- **Report builder:** User-defined reports (drag-and-drop UI) instead of hardcoded queries.
- **Coretax integration:** DJP API integration for tax reporting (ADR-007 allows REST expansion).
- **Event sourcing:** Full state reconstruction if audit trail becomes complex; currently unnecessary.
