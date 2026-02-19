# API contracts — HRSystem (Absence focus)

## Authentication & tokens
- JWTs are issued by `UserService` (ASP.NET Identity). Services validate tokens and claims.
- Example JWT payload (do not use in production):
```json
{
  "sub":"user-guid",
  "name":"Jane Manager",
  "email":"jane@company.com",
  "roles":["manager"],
  "iss":"https://auth.hr.local",
  "exp": 1893456000
}
```

## Idempotency
- `POST /api/absences` supports `Idempotency-Key` header. The server must dedupe requests by `Idempotency-Key` for a configurable period.

## Endpoints (examples)

### Request leave
POST /api/absences
- Request body
```json
{
  "employeeId":"GUID",
  "type":"paid|sick|unpaid",
  "fromDate":"2026-03-01",
  "toDate":"2026-03-05",
  "reason":"Annual leave"
}
```
- Success: 201 Created
- Errors: 400 Bad Request, 409 Conflict (insufficient balance), 401/403 (auth)

### Manager approve
POST /api/absences/{leaveId}/manager-approve
- Auth: manager role required
- Response: 200 OK { status: "ManagerApproved" }

### HR approve (finalize)
POST /api/absences/{leaveId}/hr-approve
- Auth: hr role required
- Response: 200 OK { status: "Approved" }

### Sample curl (request + auth)
```bash
curl -X POST "http://gateway/api/absences" \
  -H "Authorization: Bearer <JWT>" \
  -H "Idempotency-Key: req-12345" \
  -H "Content-Type: application/json" \
  -d '{"employeeId":"...","type":"paid","fromDate":"2026-03-01","toDate":"2026-03-05","reason":"Vacation"}'
```

## Error model
```json
{
  "error": "InsufficientBalance",
  "message": "Requested days exceed available balance",
  "details": null
}
```

## Security notes
- Manager approval requires verification that the authenticated user is the employee's manager. This check should call `EmployeeService` or validate a manager claim.
- HR approval requires `hr` role claim.

## Contracts / OpenAPI
- Provide OpenAPI/Swagger for each API project. Include example responses and validation constraints.
