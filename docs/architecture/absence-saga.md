# AbsenceService — Manager → HR Saga (Full specification)

## TL;DR
- Ownership: `AbsenceService` is authoritative for balances & accruals.
- Flow: Request → Manager approve (hold retained) → HR approve (finalize) → publish `LeaveFinalized`.
- Balance model: hold-on-request to prevent negative balances.
- Concurrency: optimistic concurrency (`rowVersion`) + transactional outbox.
- Messaging: MassTransit + RabbitMQ. .NET 8, EF Core, SQLite for local dev (Postgres/SQL Server recommended for prod).

---

## API (surface)

POST /api/absences
- Request
  ```json
  {
    "employeeId": "guid",
    "type": "paid|sick|unpaid",
    "fromDate": "YYYY-MM-DD",
    "toDate": "YYYY-MM-DD",
    "reason": "string"
  }
  ```
- Response 201
  ```json
  {
    "leaveId":"guid",
    "status":"Pending",
    "heldDays": 3,
    "availableBalance": 12.333
  }
  ```
- Errors: 409 (insufficient balance), 400 (invalid dates), 503 (EmployeeService unavailable)

POST /api/absences/{leaveId}/manager-approve
- Auth: caller must be manager or have `manager` claim

POST /api/absences/{leaveId}/hr-approve
- Auth: HR claim required

POST /api/absences/{leaveId}/reject
- Auth: manager or hr as applicable

GET /api/absences/{leaveId}
GET /api/employees/{employeeId}/balance

Idempotency: support `Idempotency-Key` header on `POST /api/absences`.

---

## Domain model & DB schema (core)

- EmployeeLeaveBalance
  - `employeeId PK`, `balance DECIMAL`, `availableBalance DECIMAL`, `holdAmount DECIMAL`, `accrualRatePerMonth DECIMAL`, `lastAccrualDate DATE`, `rowVersion`

- LeaveRequest
  - `leaveId PK`, `employeeId`, `type`, `fromDate`, `toDate`, `requestedDays`, `holdAmount`, `status`, `createdAt`, `updatedAt`, `rowVersion`

- AbsenceSagaState (MassTransit persistence)
  - `correlationId PK`, `currentState`, `requestedDays`, `employeeId`, `managerApprovedAt`, `hrApprovedAt`, `createdAt`, `updatedAt`

- OutboxMessage (transactional outbox)
  - `id`, `occurredOn`, `type`, `payload`, `processed`

Note: use `rowVersion` optimistic concurrency token for balance updates. For SQLite use integer/concurrency checks; in production use DB-native rowversion where available.

---

## Saga state machine

States: `Requested` → `ManagerApproved` → `HRApproved (Finalized)` → `Rejected/Cancelled`

Transitions:
- On `LeaveRequested` → create saga, place hold (persist hold on `EmployeeLeaveBalance`)
- Manager approves → move to `ManagerApproved` (publish `LeaveManagerApproved`)
- HR approves → finalize deduction (`holdAmount` → decrement `balance`), move to `HRApproved` and publish `LeaveFinalized`
- Any reject/cancel → release hold and publish `LeaveHoldReleased`

Timeouts / compensation:
- If Manager or HR does not act within configurable timeout (e.g., 72 hours), auto‑escalate or auto‑cancel (delayed message in MassTransit).

Persistence: saga state persisted in Absence DB (EF Core).

---

## Accrual policy (approved defaults)
- 20 days/year → 1.667 days/month
- Monthly accrual, prorated on hire
- Carryover cap: 5 days
- Accrual cap: 30 days
- Part-time prorated (FTE factor)

---

## Concurrency & transactionality
- Transactional Outbox: write domain change + OutboxMessage in same DB transaction; a dispatcher publishes Outbox entries to RabbitMQ after commit.
- Balance updates: optimistic concurrency (`rowVersion`) + retry/backoff (3 attempts) or return `409 Conflict` on persistent conflict.
- HTTP idempotency: support `Idempotency-Key` on `POST /api/absences`.
- Event idempotency: `ProcessedEvents` table keyed by `eventId`.

On request flow (atomic):
1. Sync verify employment status with `EmployeeService`.
2. Within DB transaction: verify `availableBalance >= requestedDays`; decrement `availableBalance`, increment `holdAmount`; persist `LeaveRequest(status=Pending)`; write Outbox message.

On HR approval (atomic):
- Transactionally move `holdAmount` → `balance` deduction; set `LeaveRequest(status=Approved)`; publish `LeaveFinalized` via Outbox.

---

## Event samples (abbreviated)

`LeaveHoldPlaced.v1`
```json
{ "eventId":"guid", "leaveId":"guid", "employeeId":"guid", "heldDays":3, "occurredAt":"..." }
```

`LeaveFinalized.v1`
```json
{ "eventId":"guid", "leaveId":"guid", "employeeId":"guid", "deductedDays":3, "period":{"from":"...","to":"..."}, "occurredAt":"..." }
```

All events include metadata: `eventId`, `eventVersion`, `occurredAt`, `correlationId`, `causationId`, `traceId`.

---

## Acceptance tests (must pass)
- Concurrency: simultaneous requests cannot overdraw availableBalance.
- End-to-end: request → manager approve → hr approve → `LeaveFinalized` consumed by `AttendanceService` and calendar updated.
- Saga timeouts: simulate manager no-response → auto-escalation/cancel behavior.

---

## Example curl + JWT (see `api-contracts.md` for more)
```bash
curl -X POST "http://gateway/api/absences" \
  -H "Authorization: Bearer <JWT>" \
  -H "Idempotency-Key: abcd-1234" \
  -H "Content-Type: application/json" \
  -d '{"employeeId":"...","type":"paid","fromDate":"2026-03-01","toDate":"2026-03-05","reason":"Vacation"}'
```

---

## Observability
- Instrument saga transitions and message handlers with OpenTelemetry spans. Propagate `correlationId` and `traceId` in message headers.
- Expose `/health` and `/metrics` endpoints.

---

## Implementation files (recommended)
- `AbsenceService.API/Controllers/AbsencesController.cs`
- `AbsenceService.Domain/Entities/LeaveRequest.cs`
- `AbsenceService.Infrastructure/Sagas/LeaveApprovalSaga.cs`
- `AbsenceService.Infrastructure/Repositories/LeaveBalanceRepository.cs`
- `Shared/Kernel/Events/Leave*.cs`
- `docker-compose.yml` entries for rabbitmq/jaeger/absence

---

## Notes / Risks
- Synchronous employment checks create coupling; use short timeouts and circuit breaker.
- SQLite has limited locking; use Postgres/SQL Server in production for stronger concurrency guarantees.
