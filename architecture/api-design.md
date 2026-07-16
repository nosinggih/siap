# API Design

## Purpose
Define the conventions and contracts for how the frontend, backend, and any external consumers communicate.

## Responsibilities
- Owned by: **Software Architect**; implemented by **Backend Developer**; consumed by **Frontend Developer**.

## Inputs
- Data and workflow needs from `/business/workflows.md`.
- System boundaries from `/architecture/architecture.md`.

## Outputs
- API contract (endpoints, payloads, error shapes) that backend implements against and frontend codes against — reducing integration guesswork.

## Rules
- API contracts are documented *before* parallel frontend/backend implementation begins, so both sides can work independently.
- Breaking changes to an existing endpoint require a version bump or an explicit migration note here, not a silent change.
- Error responses follow one consistent shape across the whole API.

## Checklist
- [x] Endpoint has method, path, request shape, response shape, and error cases documented.
- [x] Naming conventions are consistent with the rest of the API.
- [x] Auth/permission requirements are stated per endpoint.

---

## Conventions

### Style
**REST + JSON** (ADR-007). All endpoints use standard HTTP methods:
- `POST` → Create
- `GET` → Read (list or single)
- `PUT` → Update
- `DELETE` → Soft-delete (logical, not physical)

### Base URL
```
https://siap.uns.ac.id/api
```

### Auth
- **Method:** Laravel session cookie (stateful) for MVP.
- **Header:** No explicit Authorization header required; session cookie implicit in requests.
- **Fallback:** Bearer token support (future, ADR-008).
- **Verification:** Every request validates user session + role + unit assignment (RULE-019).

### Versioning
- **Current:** `v1` (implicit; no version prefix yet).
- **Future:** Use `/api/v2/` if breaking changes occur; never modify existing `/api/` endpoints silently.

### Response Shape — Success

```json
{
  "success": true,
  "data": {
    "id": 123,
    "nomor_jurnal": "001.25.00001",
    ...
  },
  "meta": {
    "count": 1,
    "page": 1,
    "per_page": 50,
    "total": 250
  }
}
```

### Response Shape — Error

```json
{
  "success": false,
  "data": null,
  "errors": {
    "field_name": ["Error message 1", "Error message 2"],
    "global": ["General error message"]
  }
}
```

### Status Codes

| Code | Meaning | When Used |
|------|---------|-----------|
| 200 | OK | Successful GET, PUT, DELETE |
| 201 | Created | Successful POST |
| 400 | Bad Request | Validation error (malformed payload, business logic violation) |
| 401 | Unauthorized | No valid session |
| 403 | Forbidden | Insufficient permission or unit access (RULE-019) |
| 404 | Not Found | Resource doesn't exist |
| 409 | Conflict | Business rule conflict (e.g., journal already corrected, batch already final) |
| 422 | Unprocessable Entity | Validation errors (detailed field-level errors in `errors` object) |
| 500 | Server Error | Unhandled exception; logged for debugging |

### Pagination
- **Query params:** `?page=1&per_page=50`
- **Default:** page=1, per_page=50
- **Max:** per_page=200 (prevent DoS)
- **Response:** Includes `meta.count` (current page), `meta.total` (overall)

### Filtering & Sorting
- **Query params:** `?unit_id=1&status=OPEN&sort=-created_at`
  - Prefix `-` for descending order
  - Multiple filters combined with `&`
- **Supported filters:** Documented per endpoint

### Timestamps
- **Format:** ISO 8601 (e.g., `2025-01-15T10:30:45Z`)
- **Timezone:** All timestamps in UTC; frontend converts to local
- **Nullable:** `null` for unset timestamps (e.g., `deleted_at` for non-deleted records)

---

## Endpoints

### Journal Management

#### POST /journals
**Create a new journal (all types: Umum, Transaksi, Pajak Terima, Pajak Setor, Koreksi)**

**Auth:** Operator, BPP, Operator Pusat (role-based); unit must be in user assignment (RULE-019)

**Request Body:**
```json
{
  "jenis": "UMUM",
  "sub_jenis": "UANG_MUKA",
  "tanggal": "2025-01-15",
  "nomor_bukti": "25.0001",
  "keterangan": "Pembayaran uang muka proyek X",
  "unit_id": 1,
  "periode": "202501",
  "entries": [
    {
      "akun_coa_id": 10001,
      "jenis_entry": "DEBET",
      "nominal": 1000000.00
    },
    {
      "akun_coa_id": 20001,
      "jenis_entry": "KREDIT",
      "nominal": 1000000.00
    }
  ],
  "memo_id": null,
  "file_bukti": "[multipart form data or base64]",
  "referensi_jurnal_id": null
}
```

**Special fields by journal type:**
- **Jurnal Transaksi** (`jenis: TRANSAKSI`):
  - `memo_id` (required): Links to MEMO table; auto-fills entries from MEMO
  - `entries` should have only 1 Debet entry (akun belanja from MEMO) + 1+ Kredit entries (akun lawan)

- **Jurnal Pajak Terima** (`jenis: PAJAK_TERIMA`):
  - `jenis_pajak` (required): PPh21, PPh22, PPh23, PPh26, PPN, PPh4(2)
  - `nama_wajib_pajak` (required)
  - `npwp` (required, format: 15 digits)
  - `nilai_bruto` (required)
  - Additional fields per jenis_pajak (TBD, OI-001)

- **Jurnal Pajak Setor** (`jenis: PAJAK_SETOR`):
  - `jenis_pajak` (required)
  - `pajak_terima_ids` (required): Array of Jurnal Pajak Terima IDs to consolidate
  - Sistem auto-creates entries summing nominal from referenced terima

- **Jurnal Koreksi** (`jenis: KOREKSI`):
  - `referensi_jurnal_id` (required): ID of journal being corrected (RULE-007)

**Validation (RULE-001, RULE-002, RULE-003, RULE-005, RULE-006, RULE-008):**
- Σ Debet = Σ Kredit (to 2 decimal places)
- All accounts in mapping for jenis jurnal (RULE-003)
- Period not closed (RULE-008)
- File: PDF only, ≤5MB (RULE-005)
- (Pajak Terima) field wajib lengkap (RULE-012)
- (Pajak Terima) MEMO status not already USED (RULE-016)

**Response (201 Created):**
```json
{
  "success": true,
  "data": {
    "id": 12345,
    "nomor_jurnal": "001.25.00001",
    "nomor_bukti": "25.0001",
    "jenis": "UMUM",
    "status": "RECORDED",
    "total_debet": 1000000.00,
    "total_kredit": 1000000.00,
    "created_at": "2025-01-15T10:30:45Z",
    "entries": [
      { "id": 1, "akun": { "id": 10001, "nama": "Kas Tangan" }, "jenis_entry": "DEBET", "nominal": 1000000.00 },
      { "id": 2, "akun": { "id": 20001, "nama": "Utang" }, "jenis_entry": "KREDIT", "nominal": 1000000.00 }
    ],
    "file_bukti_url": "https://siap.uns.ac.id/storage/bukti/J25000001.pdf"
  }
}
```

**Error cases:**
- 422: Unbalanced (Debet ≠ Kredit) → return delta in `errors.balance`
- 422: Account not in mapping (RULE-003)
- 409: Period closed (RULE-008)
- 409: Referenced journal (Koreksi) not found
- 422: Pajak Terima field wajib missing (RULE-012)

---

#### GET /journals
**List journals (filtered by user unit, period, status)**

**Auth:** Operator, BPP, Operator Pusat, Admin, Auditor

**Query params:**
- `unit_id` (optional): Filter by unit; default = user's assigned units
- `periode` (optional): e.g., `202501`
- `status` (optional): RECORDED, CORRECTED, FINAL, etc.
- `jenis` (optional): UMUM, TRANSAKSI, PAJAK_TERIMA, PAJAK_SETOR, KOREKSI
- `page` (optional, default=1)
- `per_page` (optional, default=50)
- `sort` (optional, default=`-created_at`): e.g., `nomor_jurnal`, `-tanggal`

**Response (200 OK):**
```json
{
  "success": true,
  "data": [
    { "id": 1, "nomor_jurnal": "001.25.00001", "jenis": "UMUM", "total_debet": 1000000, "created_at": "2025-01-15T10:30:45Z", ... },
    { "id": 2, "nomor_jurnal": "001.25.00002", "jenis": "TRANSAKSI", "total_debet": 500000, "created_at": "2025-01-15T11:00:00Z", ... }
  ],
  "meta": { "count": 2, "page": 1, "per_page": 50, "total": 2 }
}
```

**Error cases:**
- 403: User tries to list unit outside assignment (RULE-019)

---

#### GET /journals/{id}
**Retrieve single journal with full details**

**Auth:** Operator/BPP (own unit), Admin, Auditor

**Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "id": 12345,
    "nomor_jurnal": "001.25.00001",
    "jenis": "UMUM",
    "tanggal": "2025-01-15",
    "nomor_bukti": "25.0001",
    "keterangan": "...",
    "status": "RECORDED",
    "unit": { "id": 1, "nama": "Rektorat" },
    "periode": "202501",
    "entries": [
      { "id": 1, "akun": { "id": 10001, "kode": "1011", "nama": "Kas Tangan" }, "jenis_entry": "DEBET", "nominal": 1000000.00 },
      { "id": 2, "akun": { "id": 20001, "kode": "2011", "nama": "Utang" }, "jenis_entry": "KREDIT", "nominal": 1000000.00 }
    ],
    "total_debet": 1000000.00,
    "total_kredit": 1000000.00,
    "file_bukti_url": "https://siap.uns.ac.id/storage/bukti/J25000001.pdf",
    "created_by": { "id": 123, "nama": "Operator Keuangan" },
    "created_at": "2025-01-15T10:30:45Z",
    "deleted_at": null,
    "referensi_jurnal_id": null
  }
}
```

**Error cases:**
- 404: Journal not found
- 403: User cannot access journal's unit (RULE-019)

---

#### PUT /journals/{id}
**Update journal (only if period OPEN and not locked by other actors)**

**Auth:** Operator, BPP (own unit); Admin (any unit)

**Request Body:** Same as POST (partial update allowed; omit unchanged fields)

**Validation:** Same as POST (balance, account mapping, period check)

**Pre-check (can_edit_journal?):**
- Period must be OPEN (RULE-008)
- Journal must not be soft-deleted (RULE-009)
- Journal must not be referenced by another Jurnal Koreksi (RULE-007)
- (Pajak Terima) must not have status SUDAH_DISETOR (RULE-013)

**Response (200 OK):** Updated journal object (same shape as GET /{id})

**Error cases:**
- 409: Period closed → "Periode sudah ditutup, gunakan Jurnal Koreksi untuk perubahan"
- 409: Journal already corrected → "Jurnal sudah direferensikan dalam Jurnal Koreksi, tidak dapat diubah"
- 409: Referenced Pajak Terima already DISETOR (RULE-013)

---

#### DELETE /journals/{id}
**Soft-delete journal (sets deleted_at, RULE-009)**

**Auth:** Operator, BPP (own unit); Admin (any unit)

**Pre-check (same as PUT):**
- Period OPEN
- Not already soft-deleted
- Not referenced by Jurnal Koreksi

**Response (200 OK):**
```json
{
  "success": true,
  "data": { "id": 12345, "deleted_at": "2025-01-15T11:00:00Z" }
}
```

**Audit trail:** AuditLogger captures action=DELETE, entity=journal, with old_value/new_value (ADR-005)

---

### Period Management

#### POST /periods/close
**Close period (Admin only, RULE-008)**

**Auth:** Admin only

**Request Body:**
```json
{
  "unit_ids": [1, 2, 3],
  "periode": "202501",
  "mode": "SELECTIVE"
}
```
- `unit_ids`: Array of unit IDs to close (or empty array for all units)
- `periode`: Month to close (e.g., "202501")
- `mode`: SELECTIVE (specific units) | ALL (all units)

**Validation:**
- User must be Admin
- Units must exist
- Periode must be valid
- No journals pending approval (optional business rule; TBD)

**Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "closed_at": "2025-01-15T15:00:00Z",
    "unit_count": 3,
    "journal_count_closed": 42,
    "message": "3 unit(s) closed for periode 202501"
  }
}
```

**Consequences:**
- All subsequent POST/PUT /journals requests for closed unit+periode return 409 (period locked)
- Frontend disables journal form for that unit+month

---

#### POST /periods/reopen
**Reopen period (Admin only, RULE-009)**

**Auth:** Admin only

**Request Body:**
```json
{
  "unit_ids": [1],
  "periode": "202501"
}
```

**Validation:**
- Must be Admin
- Unit must exist
- Periode must be CLOSED (cannot reopen OPEN period)
- Only specified units reopened; others unaffected (RULE-009)

**Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "reopened_at": "2025-01-15T15:30:00Z",
    "unit_count": 1,
    "message": "Unit 1 reopened for periode 202501"
  }
}
```

---

#### GET /periods
**List periods (status per unit)**

**Auth:** Operator (own unit), Admin, Auditor

**Query params:**
- `unit_id` (optional)
- `periode` (optional)
- `status` (optional): OPEN, CLOSED
- `sort` (optional, default=`-periode`)

**Response (200 OK):**
```json
{
  "success": true,
  "data": [
    { "id": 1, "unit_id": 1, "periode": "202501", "status": "CLOSED", "closed_at": "2025-01-31T23:59:59Z" },
    { "id": 2, "unit_id": 1, "periode": "202502", "status": "OPEN", "closed_at": null }
  ],
  "meta": { "count": 2, "total": 2 }
}
```

---

### Tax Management (Pajak Terima/Setor/Batch)

#### POST /tax-forms
**Create Jurnal Pajak Terima (special endpoint; can also use POST /journals with jenis=PAJAK_TERIMA)**

**Auth:** Operator, BPP (own unit)

**Request Body:**
```json
{
  "jenis_pajak": "PPh21",
  "tanggal": "2025-01-15",
  "nomor_bukti": "25.0001",
  "nama_wajib_pajak": "PT Maju Jaya",
  "npwp": "123456789012345",
  "nilai_bruto": 10000000.00,
  "nominal_pajak": 1000000.00,
  "keterangan": "PPh 21 Gaji Karyawan"
}
```

**Validation (RULE-012):**
- All wajib fields present
- NPWP format 15 digits
- nilai_bruto > 0, nominal_pajak > 0

**Response (201 Created):** Same as POST /journals

---

#### POST /tax-forms/{id}/setor
**Create Jurnal Pajak Setor from one or more Pajak Terima (RULE-013)**

**Auth:** Operator, BPP (own unit)

**Request Body:**
```json
{
  "tanggal_setor": "2025-01-31",
  "nomor_bukti": "25.0099",
  "pajak_terima_ids": [1, 2, 3],
  "keterangan": "Setor PPh 21 Januari ke Bank"
}
```

**Validation (RULE-013, RULE-014):**
- All referenced Pajak Terima exist and belong to same unit
- All have status BELUM_DISETOR
- All have same jenis_pajak (RULE-014)
- All from same periode/bulan (RULE-014)

**Side effects:**
- Create Jurnal Pajak Setor
- Update referenced Pajak Terima: status → SUDAH_DISETOR

**Response (201 Created):**
```json
{
  "success": true,
  "data": {
    "id": 9999,
    "nomor_jurnal": "001.25.00099",
    "jenis": "PAJAK_SETOR",
    "total_nominal": 3000000.00,
    "pajak_terima_count": 3,
    "status": "RECORDED"
  }
}
```

**Error cases:**
- 422: Pajak Terima status not BELUM_DISETOR (RULE-013)
- 422: Mixed jenis_pajak (RULE-014)
- 422: Mixed periode/bulan (RULE-014)

---

#### POST /tax-batches
**Create Batch Pajak (consolidate Jurnal Setor from all units)**

**Auth:** Operator Pajak (role-based)

**Request Body:**
```json
{
  "jenis_pajak": "PPh21",
  "periode": "202501",
  "jurnal_setor_ids": [9999, 10000, 10001],
  "keterangan": "Batch PPh 21 Jan 2025 untuk lapor ke DJP"
}
```

**Validation (RULE-014):**
- All Jurnal Setor have same jenis_pajak
- All from same periode

**Response (201 Created):**
```json
{
  "success": true,
  "data": {
    "id": 999,
    "nomor_batch": "BATCH-2025-PPh21-00001",
    "jenis_pajak": "PPh21",
    "periode": "202501",
    "total_nominal": 15000000.00,
    "unit_count": 3,
    "journal_count": 3,
    "status": "DRAFT"
  }
}
```

---

#### PUT /tax-batches/{id}
**Update Batch (only if DRAFT, not FINAL)**

**Auth:** Operator Pajak

**Pre-check (RULE-015):**
- Batch must have status DRAFT (not FINAL)

**Request Body:** Same as POST (partial update)

**Error cases:**
- 409: Batch status FINAL → "Batch sudah di-finalisasi, tidak dapat diubah" (RULE-015)

---

#### POST /tax-batches/{id}/finalize
**Finalize Batch → status: FINAL (immutable, RULE-015)**

**Auth:** Operator Pajak

**Request Body:** Empty or `{ "note": "..." }`

**Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "id": 999,
    "status": "FINAL",
    "finalized_at": "2025-02-01T10:00:00Z"
  }
}
```

**Consequences (RULE-015):**
- All subsequent PUT/DELETE on this batch return 409

---

#### GET /tax-batches
**List Batch Pajak (filter by status, jenis, periode)**

**Auth:** Operator Pajak, Admin, Auditor

**Query params:**
- `jenis_pajak` (optional)
- `periode` (optional)
- `status` (optional): DRAFT, FINAL
- `sort` (optional, default=`-created_at`)

**Response (200 OK):** Array of batch objects

---

### Report Management

#### POST /reports/generate
**Generate report (BKU, Neraca Saldo, LPJ, Daya Serap, Comparative)**

**Auth:** Operator (own unit), Operator Pusat (any unit), Admin, Auditor

**Request Body:**
```json
{
  "jenis_laporan": "BKU",
  "unit_ids": [1, 2, 3],
  "periode_awal": "202501",
  "periode_akhir": "202501",
  "format": "PDF",
  "tahun_pembanding": null
}
```

- `jenis_laporan`: BKU, NERACA_SALDO, LPJ, DAYA_SERAP, COMPARATIVE
- `unit_ids`: Array of unit IDs (Operator restricted to own unit; Admin/Auditor can choose any)
- `periode_awal`, `periode_akhir`: Month range (e.g., "202501")
- `format`: PDF or EXCEL
- `tahun_pembanding`: Only for COMPARATIVE; must be previous year (RULE-011)

**Validation (RULE-011, RULE-019):**
- User can only report on assigned units (RULE-019) or is Admin
- Tahun pembanding must be previous year, not future (RULE-011)
- Periode must be valid

**Response (201 Created - async):**
```json
{
  "success": true,
  "data": {
    "id": 5555,
    "jenis_laporan": "BKU",
    "status": "PENDING",
    "format": "PDF",
    "download_url": null,
    "created_at": "2025-02-01T10:00:00Z",
    "estimated_ready_at": "2025-02-01T10:05:00Z"
  }
}
```

**Flow:**
1. Save Report record in DB (status: PENDING)
2. Queue async job (ReportGenerationJob)
3. Return immediately with report ID
4. Frontend polls GET /reports/{id} or listens to WebSocket event when ready

---

#### GET /reports/{id}
**Check report generation status + download link**

**Auth:** Owner, Admin, Auditor

**Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "id": 5555,
    "jenis_laporan": "BKU",
    "status": "READY",
    "format": "PDF",
    "download_url": "https://siap.uns.ac.id/api/reports/5555/download",
    "created_at": "2025-02-01T10:00:00Z",
    "completed_at": "2025-02-01T10:04:30Z"
  }
}
```

**Statuses:**
- PENDING: Queued, not yet generated
- PROCESSING: Currently generating
- READY: Available for download
- FAILED: Generation error (check error log)

---

#### GET /reports/{id}/download
**Download report file (PDF/Excel)**

**Auth:** Owner, Admin, Auditor

**Response:** File binary stream (Content-Type: application/pdf or application/vnd.ms-excel)

---

#### POST /reports/{id}/upload-signed
**Upload signed copy of report (WF-008)**

**Auth:** Unit that generated report, Admin

**Request Body:** Multipart form-data with PDF file

**Validation (REQ-020):**
- File is PDF
- Size ≤ 10MB (generous for multi-page reports)

**Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "id": 5555,
    "signed_file_url": "https://siap.uns.ac.id/storage/reports-signed/LPJ-2025-01-001.pdf",
    "upload_status": "SUDAH_UPLOAD",
    "uploaded_at": "2025-02-05T14:30:00Z"
  }
}
```

**Consequences:**
- Report now visible to Admin, Operator Pusat, Auditor (REQ-021)
- Upload status shows in list (WF-008)

---

#### GET /reports
**List all generated reports (with upload status)**

**Auth:** User (own unit reports), Admin, Auditor (all)

**Query params:**
- `unit_id` (optional)
- `jenis_laporan` (optional)
- `periode` (optional)
- `upload_status` (optional): BELUM_UPLOAD, SUDAH_UPLOAD
- `sort` (optional, default=`-created_at`)

**Response (200 OK):**
```json
{
  "success": true,
  "data": [
    {
      "id": 5555,
      "jenis_laporan": "LPJ",
      "periode": "202501",
      "upload_status": "SUDAH_UPLOAD",
      "uploaded_at": "2025-02-05T14:30:00Z",
      "created_at": "2025-02-01T10:00:00Z"
    }
  ],
  "meta": { "count": 1, "total": 1 }
}
```

---

### Sync & Integration

#### POST /sync/memo
**Manually trigger sync from SIreva API (WF-012)**

**Auth:** Admin only

**Request Body:** Empty or `{ "force": true }`

**Response (202 Accepted - async):**
```json
{
  "success": true,
  "data": {
    "sync_id": "sync-20250201-123456",
    "status": "QUEUED",
    "message": "MEMO sync queued, check back in 30 seconds"
  }
}
```

**Async flow:**
1. Queue SyncMemoJob
2. Job calls SIreva API → GET /memo?status=verified
3. Upsert `memos` table
4. Log sync result (count inserted, updated, errors)

**Status polling:** GET /sync/status/{sync_id}

---

#### GET /sync/status/{sync_id}
**Check sync status**

**Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "sync_id": "sync-20250201-123456",
    "status": "COMPLETED",
    "memo_inserted": 25,
    "memo_updated": 10,
    "errors": [],
    "completed_at": "2025-02-01T10:30:00Z",
    "message": "Synced 35 MEMO records"
  }
}
```

---

### User & Master Data Management

#### POST /users
**Create user (Admin only)**

**Auth:** Admin only

**Request Body:**
```json
{
  "nama": "Operator Keuangan",
  "username": "op.keuangan",
  "email": "op.keuangan@uns.ac.id",
  "password": "SecurePassword123!",
  "role": "OPERATOR",
  "unit_ids": [1, 2]
}
```

**Validation:**
- Username unique
- Email unique (optional)
- Role in allowed list: ADMIN, OPERATOR_PUSAT, OPERATOR, BPP, VERIFIKATOR, OPERATOR_PAJAK, OPERATOR_ASET, AUDITOR
- Unit IDs exist

**Response (201 Created):**
```json
{
  "success": true,
  "data": {
    "id": 123,
    "nama": "Operator Keuangan",
    "username": "op.keuangan",
    "role": "OPERATOR",
    "unit_ids": [1, 2],
    "is_active": true,
    "created_at": "2025-02-01T10:00:00Z"
  }
}
```

---

#### PUT /users/{id}
**Update user (Admin only)**

**Auth:** Admin only

**Request Body:** Same as POST (partial update allowed)

**Response (200 OK):** Updated user object

---

#### POST /users/{id}/deactivate
**Deactivate user (soft-disable, not delete)**

**Auth:** Admin only

**Request Body:** Empty

**Response (200 OK):**
```json
{
  "success": true,
  "data": { "id": 123, "is_active": false, "deactivated_at": "2025-02-01T11:00:00Z" }
}
```

---

#### GET /users
**List users (Admin only)**

**Auth:** Admin only

**Query params:**
- `is_active` (optional): true, false
- `role` (optional)
- `unit_id` (optional)
- `sort` (optional, default=`nama`)

**Response (200 OK):** Array of user objects

---

#### POST /users/{id}/impersonate
**Start impersonation session (Admin only, RULE-017)**

**Auth:** Admin only

**Request Body:** Empty

**Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "impersonation_id": "imp-123456",
    "impersonating_as": { "id": 456, "nama": "Operator Keuangan", "role": "OPERATOR" },
    "admin_identity": { "id": 123, "nama": "Admin User" },
    "started_at": "2025-02-01T11:00:00Z"
  }
}
```

**Consequences (RULE-017):**
- Session cookie updated to reflect target user
- All subsequent requests routed as if from impersonated user
- Audit log captures: `[Admin: 123] impersonating [User: 456]` (ADR-005)
- Banner in UI: "Mode Pinjam User — Acting as [Operator Keuangan]"

---

#### POST /users/impersonate/end
**End impersonation session**

**Auth:** Any (during impersonation)

**Request Body:** Empty

**Response (200 OK):**
```json
{
  "success": true,
  "data": { "impersonation_ended_at": "2025-02-01T11:05:00Z" }
}
```

**Audit log:** Captures end of impersonation session

---

#### POST /coas
**Create Chart of Accounts (Admin only)**

**Auth:** Admin only

**Request Body:**
```json
{
  "kode": "1011",
  "nama": "Kas Tangan",
  "tipe": "ASET",
  "posisi_normal": "DEBET",
  "deskripsi": "Uang tunai ditangan"
}
```

**Validation:**
- Kode unique (format BASK)
- Nama not empty

**Response (201 Created):**
```json
{
  "success": true,
  "data": {
    "id": 10001,
    "kode": "1011",
    "nama": "Kas Tangan",
    "tipe": "ASET",
    "posisi_normal": "DEBET",
    "is_active": true,
    "created_at": "2025-02-01T10:00:00Z"
  }
}
```

---

#### PUT /coas/{id}
**Update CoA (Admin only)**

**Auth:** Admin only

**Restrictions (RULE-018):**
- If CoA already used in journal: only name, deskripsi updatable (not kode, tipe, posisi)

**Response (200 OK):** Updated CoA object

---

#### POST /coas/{id}/deactivate
**Deactivate CoA (soft-disable, RULE-018)**

**Auth:** Admin only

**Restrictions (RULE-018):**
- CoA with journal references: only deactivate, not delete
- Inactive CoA hidden from form selection, but data still readable

**Response (200 OK):**
```json
{
  "success": true,
  "data": { "id": 10001, "is_active": false, "deactivated_at": "2025-02-01T11:00:00Z" }
}
```

---

#### GET /coas
**List Chart of Accounts (with active filter)**

**Auth:** Operator, BPP, Admin, Auditor

**Query params:**
- `is_active` (optional, default: true)
- `tipe` (optional): ASET, LIABILITAS, EKUITAS, PENDAPATAN, BELANJA
- `sort` (optional, default: `kode`)

**Response (200 OK):**
```json
{
  "success": true,
  "data": [
    { "id": 10001, "kode": "1011", "nama": "Kas Tangan", "tipe": "ASET", "is_active": true },
    { "id": 10002, "kode": "1012", "nama": "Kas Bank", "tipe": "ASET", "is_active": true }
  ],
  "meta": { "count": 2, "total": 250 }
}
```

---

#### GET /memos
**List MEMO (from SIreva sync, with usage status)**

**Auth:** Operator, BPP

**Query params:**
- `unit_id` (optional, required for non-Admin)
- `status` (optional): AVAILABLE, USED
- `sort` (optional, default: `-updated_at`)

**Response (200 OK):**
```json
{
  "success": true,
  "data": [
    { "id": 1, "nomor_memo": "M-2025-001", "akun_belanja": "5211", "nominal": 50000000, "status": "AVAILABLE" },
    { "id": 2, "nomor_memo": "M-2025-002", "akun_belanja": "5212", "nominal": 30000000, "status": "USED" }
  ],
  "meta": { "count": 2, "total": 250 }
}
```

---

### Audit & Logging

#### GET /audit-log
**View audit trail (immutable, RULE-020)**

**Auth:** Admin, Auditor

**Query params:**
- `user_id` (optional)
- `entity_type` (optional): journal, period, user, coa, tax_batch
- `action` (optional): CREATE, UPDATE, DELETE, APPROVE, CLOSE_PERIOD
- `period_from` (optional, ISO date)
- `period_to` (optional)
- `sort` (optional, default: `-created_at`)
- `page`, `per_page`

**Response (200 OK):**
```json
{
  "success": true,
  "data": [
    {
      "id": 99999,
      "user_id": 456,
      "user_name": "Operator Keuangan",
      "action": "CREATE",
      "entity_type": "journal",
      "entity_id": 12345,
      "old_value": null,
      "new_value": { "nomor_jurnal": "001.25.00001", "total_debet": 1000000 },
      "timestamp": "2025-02-01T10:30:45Z",
      "ip_address": "192.168.1.100",
      "impersonation_by": null
    },
    {
      "id": 99998,
      "user_id": 123,
      "user_name": "Admin",
      "action": "IMPERSONATE",
      "entity_type": "impersonation",
      "entity_id": 456,
      "old_value": null,
      "new_value": { "target_user_id": 456 },
      "timestamp": "2025-02-01T11:00:00Z",
      "ip_address": "192.168.1.50",
      "impersonation_by": 123
    }
  ],
  "meta": { "count": 2, "page": 1, "total": 5000 }
}
```

**Notes:**
- `impersonation_by` field: if non-null, real actor was Admin (RULE-017)
- Audit log is append-only; no DELETE/UPDATE endpoint exists (RULE-020)

---

## Error Handling

### Standard Error Response Structure

```json
{
  "success": false,
  "data": null,
  "errors": {
    "field1": ["Error message for field1"],
    "field2": ["Error message 1 for field2", "Error message 2 for field2"],
    "global": ["General server error or business logic error"]
  }
}
```

### Common Error Cases

| Scenario | Status | Error Message |
|----------|--------|---------------|
| Unbalanced journal | 422 | `"Jurnal tidak balance. Debet: 1000000, Kredit: 900000. Selisih: 100000"` |
| Period closed | 409 | `"Periode sudah ditutup. Gunakan Jurnal Koreksi untuk perubahan."` |
| Account not in mapping | 422 | `"Akun 5555 (Belanja Operasional) tidak tersedia untuk Jurnal Transaksi. Hubungi Admin."` |
| Unauthorized unit access | 403 | `"Anda tidak memiliki akses ke unit 2."` |
| MEMO already used | 409 | `"MEMO M-2025-001 sudah digunakan dalam jurnal lain. Pesan: 'Partial usage tidak diizinkan'."` |
| Batch already final | 409 | `"Batch sudah di-finalisasi dan tidak dapat diubah."` |
| Invalid session | 401 | `"Sesi tidak valid atau telah kadaluarsa. Silakan login kembali."` |

---

## Future Enhancements (Not in MVP)

- **Token-based auth** (Bearer token, JWT) for mobile/external integrations (ADR-008)
- **WebSocket events** for real-time report generation status updates
- **Graphql layer** if N+1 query optimization becomes critical
- **Rate limiting** per user/IP to prevent DoS
- **Caching** (Redis) for frequently accessed master data (CoA, MEMO)

---

## API Versioning Strategy

Current version is implicit `v1` (no prefix). If breaking changes needed:
1. Create new `/api/v2/` endpoints
2. Keep `/api/v1/` alive for 2 release cycles (backward compat)
3. Document migration path in this file
4. Record decision in `/architecture/decisions.md` as ADR-008+

Never modify existing endpoints; always create new version.
