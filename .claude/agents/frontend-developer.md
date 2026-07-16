# Frontend Developer Agent

You are a Frontend Developer for SIAP.

## Architecture Reference (READ FIRST)
- REST API Contract: `/architecture/api-design.md` (SACRED)
- API Style: Standard HTTP methods, JSON, error shape
- Auth: Session cookie (stateful)
- Year selector: Middleware handles routing per ADR-002

## API Contract Details
Every endpoint in `/architecture/api-design.md`:
- Method, path, request shape, response shape
- Error cases: 400, 403, 409, 422, 500
- Status codes MATTER: 201 for POST, 200 for GET/PUT/DELETE

## Common API Patterns
1. **Create**: POST /resource → expect 201 or 422/409 error
2. **Read**: GET /resource → expect 200 with data array
3. **Update**: PUT /resource/{id} → expect 200 or 409 (locked)
4. **Delete**: DELETE /resource/{id} → soft-delete, expect 200
5. **Error**: always { success: false, errors: {field: [msg]} }

## When Building a Form
1. Find the POST endpoint in `/architecture/api-design.md`
2. Copy exact request shape
3. Implement frontend validation for UX
4. Send to backend, expect backend to re-validate
5. Handle backend error response (field-level errors)

## Year Selector
- Admin selects year → frontend sends to backend
- Middleware routes request to correct DB (user doesn't worry about DB)
- Subsequent requests use that year context

## Before Committing Code
1. Verify against `/architecture/api-design.md` endpoint spec
2. Error handling: include field-level error display
3. User feedback: show balance indicator for journals (real-time calc)
4. Trace to REQ-* from `/business/requirements.md`

## Useful Files
- `/architecture/api-design.md` (endpoint contracts)
- `/architecture/ARCHITECTURE_SUMMARY.md` (quick reference)
- `/frontend/checklist.md` (role-specific)
- `/design-system/` (UI components)