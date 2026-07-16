# SIAP Architecture — Quick-Start Summary

**For:** Development Team (Backend, Frontend, QA, DevOps)  
**Purpose:** High-level overview of decisions and constraints  
**Read time:** ~15 minutes

---

## What is SIAP?

A financial management system for UNS (Universitas Sebelas Maret), tracking:
- **Journals:** Entry of all financial transactions (5+ types)
- **Taxes:** Recording, consolidation, batch reporting to DJP
- **Periods:** Monthly close/reopen per unit
- **Reports:** BKU, Neraca Saldo, LPJ, Daya Serap, Comparative
- **Audit:** Immutable log of all actions (compliance requirement)

**Scope:** Fiscal years 2024–2026 (potentially more)  
**Users:** 8 roles across multiple units (Rektorat, Aset, Keuangan, etc.)

---

## Key Principles (in 30 seconds)

| Principle | Means |
|-----------|-------|
| **One DB per year** | Separate `keuangan_2024`, `keuangan_2025`, etc.; no schema sharing |
| **Smart year-switching** | Middleware auto-routes requests to correct DB; transparent to code |
| **No permanent deletes** | All deletions are soft (set `deleted_at`); history preserved for audit |
| **Balance first** | Every journal must: Σ Debet = Σ Kredit, validated at backend |
| **Audit everything** | Action + data diff logged; immutable, append-only |
| **Respect unit boundaries** | Users can only see/edit their assigned units (RULE-019) |
| **REST API** | Standard HTTP methods; JSON payloads; no GraphQL |

---

## Architecture at a Glance

```
┌─────────────────┐
│   Frontend      │ Browser (Blade or Vue/React SPA)
└────────┬────────┘
         │ HTTP/REST
┌────────▼───────────────────┐
│  Laravel Backend            │
│  ├─ Middleware (auth,      │
│  │  tenant routing, audit)  │
│  ├─ Controllers (API)       │
│  ├─ Services (logic)        │
│  └─ Models (Eloquent)       │
└────────┬───────────────────┘
         │ SQL
┌────────▼───────────────────┐
│ MySQL Databases             │
│ ├─ keuangan_2024 (read-only)│
│ ├─ keuangan_2025 (active)   │
│ └─ keuangan_2026            │
└─────────────────────────────┘
         +
      ↙   ↖
 ┌───────────────┐
 │ SIreva API    │ (MEMO/PAGU sync)
 └───────────────┘
```

---

## Stack Decisions

### Why Laravel + MySQL?

- **Existing:** Already running in production since 2022; team knows it
- **Safe:** No migration risk; data stays put
- **Familiar:** Eloquent ORM, migrations, Blade templating
- **No overhead:** Proven infrastructure; lower operational complexity

**Not doing:** Node, Python, PostgreSQL, or complete rewrites

---

### Multi-Database Routing (ADR-002)

**Problem:** Multiple fiscal years; each year's data separate.  
**Solution:** Middleware reads `tahun_aktif` → connects to `keuangan_YYYY` → all queries auto-route.

**For developers:**
```php
// In middleware:
$year = config('app.tahun_aktif'); // e.g., 2025
config(['database.default' => "keuangan_{$year}"]);

// In models/services: No year parameter needed!
$journals = Journal::where('unit_id', 1)->get();
// ^ Automatically queries keuangan_2025
```

**For DevOps:**
```bash
# Provision databases per year
mysql> CREATE DATABASE keuangan_2024;
mysql> CREATE DATABASE keuangan_2025;
mysql> CREATE DATABASE keuangan_2026;

# Run migrations on all
php artisan migrate --database=keuangan_2024
php artisan migrate --database=keuangan_2025
php artisan migrate --database=keuangan_2026
```

---

### Soft Deletes (ADR-003)

**Problem:** Audit trail requires never permanently deleting records.  
**Solution:** Set `deleted_at` timestamp; default queries exclude soft-deleted rows.

**For developers:**
```php
// In Model:
use SoftDeletes;

// Soft-delete:
$journal->delete(); // Sets deleted_at = NOW()

// Query (excludes soft-deleted by default):
Journal::all(); // Returns only active journals

// Query including soft-deleted:
Journal::withTrashed()->all();

// Restore:
$journal->restore(); // Clears deleted_at
```

**In reports:**
```php
// Financial reports must exclude soft-deleted:
$balance = Journal::active()
    ->where('unit_id', $unitId)
    ->sum('nominal');
```

---

### Balance Validation (RULE-001)

**Problem:** Journals must balance; allow 3+ entries.  
**Solution:** Validate Σ Debet = Σ Kredit at backend, not frontend.

**For backend developers:**
```php
// Service layer:
public function validateBalance(array $entries) {
    $debet = collect($entries)
        ->where('jenis', 'DEBET')
        ->sum('nominal');
    
    $kredit = collect($entries)
        ->where('jenis', 'KREDIT')
        ->sum('nominal');
    
    if (bccomp($debet, $kredit, 2) !== 0) {
        throw new BalanceException(
            "Unbalanced: Debet $debet vs Kredit $kredit (delta: " . 
            bcsub($debet, $kredit, 2) . ")"
        );
    }
}
```

**For frontend developers:**
```javascript
// Real-time calculation:
const debet = entries
    .filter(e => e.jenis === 'DEBET')
    .reduce((sum, e) => sum + parseFloat(e.nominal), 0);

const kredit = entries
    .filter(e => e.jenis === 'KREDIT')
    .reduce((sum, e) => sum + parseFloat(e.nominal), 0);

const isBalanced = Math.abs(debet - kredit) < 0.01;
buttonSimpan.disabled = !isBalanced;
```

---

### Audit Logging (ADR-005)

**Problem:** Compliance requires immutable record of who-did-what-when.  
**Solution:** Event listeners capture mutations → audit_log table (append-only).

**For developers:**
```php
// Model:
class Journal extends Model {
    protected static function boot() {
        parent::boot();
        
        static::created(function ($model) {
            AuditLog::log('CREATE', 'journal', $model->id, null, $model->toArray());
        });
        
        static::updated(function ($model) {
            AuditLog::log('UPDATE', 'journal', $model->id, $model->getOriginal(), $model->toArray());
        });
    }
}

// AuditLog::log():
public static function log($action, $entityType, $entityId, $oldValue, $newValue) {
    DB::table('audit_logs')->insert([
        'user_id' => auth()->id(),
        'action' => $action,
        'entity_type' => $entityType,
        'entity_id' => $entityId,
        'old_value' => json_encode($oldValue),
        'new_value' => json_encode($newValue),
        'timestamp' => now(),
        'ip_address' => request()->ip(),
        'impersonation_by' => session('admin_id_impersonating'), // if applicable
    ]);
}
```

**For QA:**
```sql
-- Verify all journal creates are logged:
SELECT * FROM audit_logs WHERE action='CREATE' AND entity_type='journal';

-- Verify no one deleted audit_log itself:
SELECT * FROM audit_logs WHERE entity_type='audit_log' AND action='DELETE';
-- ^ Should be empty!
```

---

### Cross-DB Reads for Reports (ADR-004)

**Problem:** Reports need current year + previous year data; old DB must remain read-only.  
**Solution:** Cache previous-year summary in active DB; fall back to direct read if cache stale.

**For developers:**
```php
// Current year: regular query
$currentBalance = Journal::where('tahun', 2025)
    ->where('unit_id', $unitId)
    ->sum('nominal_debet');

// Previous year: read from summary cache (in active DB)
$previousBalance = DB::table('summary_2024')
    ->where('unit_id', $unitId)
    ->sum('balance');

// If summary cache missing, fall back to direct read:
// (only during year-switch setup, not runtime)
$fallback = DB::connection('keuangan_2024')
    ->table('journals')
    ->where('unit_id', $unitId)
    ->sum('nominal_debet');
```

**For DevOps:**
```bash
# When year switches (e.g., 2025 → 2026):
# 1. Verify keuangan_2026 exists and migrated
# 2. Create summary_2025 table in keuangan_2026
# 3. Populate summary_2025 from keuangan_2025 final balances
# 4. Set config tahun_aktif = 2026
# 5. Test: verify reports show current 2026 + previous 2025
```

---

### API Contract (ADR-007)

**Every endpoint follows this pattern:**

**Request:**
```http
POST /api/journals
Content-Type: application/json
Authorization: (via session cookie)

{
  "jenis": "UMUM",
  "entries": [ ... ],
  "tanggal": "2025-01-15"
}
```

**Response (Success):**
```json
{
  "success": true,
  "data": { "id": 123, "nomor_jurnal": "001.25.00001", ... },
  "meta": { "count": 1, "page": 1 }
}
```

**Response (Error):**
```json
{
  "success": false,
  "data": null,
  "errors": {
    "entries": ["Jurnal tidak balance. Debet: 1000000, Kredit: 900000"],
    "global": ["Periode sudah ditutup"]
  }
}
```

**Status codes:**
- `200` OK (GET, PUT, DELETE success)
- `201` Created (POST success)
- `400` / `422` Validation error
- `403` Forbidden (permission denied)
- `409` Conflict (business rule violation)
- `500` Server error

---

## Key Workflows (TL;DR)

### Create Journal (WF-001, WF-002, WF-003)

```
User fills form → Submit → API validates balance → Saves entry + entries
                           ↓
                    Auto-generates nomor_jurnal
                           ↓
                    Emits Journal.created event
                           ↓
                    Audit listener logs action
                           ↓
                    Response: success + journal ID
```

### Close Period (WF-006)

```
Admin selects unit + month → Clicks "Tutup Periode" → API checks Admin role
                                                    ↓
                                                Saves Period.status = CLOSED
                                                    ↓
                                                All journal creates blocked for unit+month
                                                    ↓
                                                Audit logs action
```

### Generate Report (REQ-016, REQ-022)

```
User selects report type + unit + period → Clicks "Generate" → API queues job
                                                               ↓
                                                           Job runs (async)
                                                               ↓
                                                        Queries journals
                                                               ↓
                                                        For comparisons:
                                                        - Query keuangan_2025
                                                        - Read summary_2024
                                                               ↓
                                                        Generate PDF/Excel
                                                               ↓
                                                        Response: report ready
                                                        User downloads
```

---

## Developer Checklists

### Before Writing Code

- [ ] Requirement traced to REQ-NNN or RULE-NNN
- [ ] Database changes planned (new tables/columns)
- [ ] API endpoint spec documented
- [ ] Business rules validation identified
- [ ] Access control (unit scoping, role check) considered
- [ ] Audit logging hooks planned (what to log?)

### Before Committing

- [ ] Code follows REST style (POST, GET, PUT, DELETE)
- [ ] Error responses include `{ success, data, errors }` shape
- [ ] Validation at backend, not frontend
- [ ] Queries respect tenant routing (no hardcoded DB)
- [ ] Soft deletes used (no hard deletes)
- [ ] Unit scoping enforced (RULE-019)
- [ ] Audit logging wired up (event listener or middleware)
- [ ] Tests pass (unit + integration)

### Before Review

- [ ] Update `/architecture/api-design.md` if new endpoint
- [ ] Update `/architecture/architecture.md` if new component
- [ ] Propose ADR if architecture decision
- [ ] Add decision to `/docs/decisions-log.md` if cross-cutting

---

## Common Pitfalls

| Mistake | Why Bad | Fix |
|---------|---------|-----|
| Hardcoded DB name in query | Multi-year support breaks | Use Model (auto-routed) or `DB::connection(config('database.default'))` |
| Hard-delete records | Audit trail lost, compliance violation | Use `$model->delete()` (soft delete) or prevent deletion |
| Frontend-only validation | API can be bypassed | Always validate at backend service layer |
| Untracked mutations | Audit log incomplete | Emit event or call AuditLog::log() |
| Ignoring unit assignment | Users see/edit others' data | Check `AccessControl::userCanAccessUnit()` |
| Floating-point math on money | Rounding errors accumulate | Use DECIMAL(18,2) + bcmath functions |
| Leaking user data across units | Privacy/security violation | Apply unit WHERE clause to every query (model scope) |

---

## Testing Strategy

### Unit Tests
- Model relationships
- Service validation logic (balance, access control)
- Helper functions (formatCurrency, validateNPWP, etc.)

### Integration Tests
- Full request/response flow: POST /journals
- Database tenant routing (verify request routes to correct DB)
- Audit logging (verify mutations logged)
- Cross-year queries (verify old-year read-only)

### E2E / QA Tests
- User workflows (WF-001 ~ WF-012)
- Period close/reopen logic
- Report generation (current + comparative)
- Impersonation audit trail

### Performance Tests
- 200-500 journals/day throughput (REQ-037)
- Report generation time (<5s for BKU)
- Audit log query response (<500ms)

---

## Deployment Checklist

### Before Production

- [ ] All three databases created: `keuangan_2024`, `2025`, `2026`
- [ ] Migrations run on all databases
- [ ] Laravel config: `tahun_aktif` set correctly
- [ ] `.env` secrets configured (DB passwords, API keys)
- [ ] Backup strategy in place (daily snapshots)
- [ ] Monitoring alerts set (DB health, audit log growth)

### Post-Deployment

- [ ] Smoke test: Create journal, verify it appears in list
- [ ] Test year-switch: Change `tahun_aktif`, verify queries route to new DB
- [ ] Verify audit log populated (check DB)
- [ ] Test report generation (at least BKU)
- [ ] Test user access control (verify unit scoping works)

---

## References

| Document | Purpose |
|----------|---------|
| `/architecture/decisions.md` | Detailed ADRs (context, alternatives, consequences) |
| `/architecture/architecture.md` | Component overview, data flows, non-functional requirements |
| `/architecture/api-design.md` | Full API endpoint spec (request/response/errors) |
| `/docs/decisions-log.md` | Cross-cutting decisions quick-ref |
| `/business/requirements.md` | Functional requirements (REQ-001 ~ REQ-040) |
| `/business/business-rules.md` | Business rules (RULE-001 ~ RULE-020) |
| `/business/workflows.md` | Step-by-step user workflows (WF-001 ~ WF-012) |

---

## Quick Questions?

**Q: How do I query the old fiscal year (2024) from the 2025 app?**  
A: Use `DB::connection('keuangan_2024')->table('journals')->...`. No Eloquent Model; raw queries only. Read-only.

**Q: Can I hard-delete a journal if I made a mistake?**  
A: No. Use soft-delete (`$journal->delete()`). If you need to undo, call `$journal->restore()`.

**Q: How do I know if a user can edit this journal?**  
A: Call `JournalService::canEditJournal($journalId)`. Checks: period open, no references, not already soft-deleted.

**Q: I need to add a new business rule. What do I do?**  
A: Add to `/business/business-rules.md` as RULE-NNN. Reference it in the service layer validation. Add test case.

**Q: How do I debug which database the app is using?**  
A: Check Laravel logs. Middleware logs active DB connection at request start. Or add `dd(config('database.default'));` in code.

**Q: Why no GraphQL?**  
A: REST is simpler for CRUD forms, easier for team, good enough for this system. GraphQL is overkill. Can add later if needed (ADR-008).

---

**Last Updated:** 2025-02-01  
**For questions:** Contact Software Architect
