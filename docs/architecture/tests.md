# Tests & acceptance checklist — AbsenceService

## Unit tests
- AccrualPolicyTests: validate monthly accrual, proration, carryover caps.
- LeaveRequestAggregateTests: no overlapping approved leaves, status transitions, hold logic.
- LeaveBalanceRepositoryTests: concurrency behavior (optimistic concurrency path).

## Integration tests
- SagaTests (MassTransit Test Harness): manager approve → HR approve → finalization flows.
- Event contract tests: `LeaveFinalized` consumed by a mocked `AttendanceService` updates calendar.
- Employment status check integration: AbsenceService rejects requests for terminated employees.

## Concurrency & E2E tests
- Parallel `POST /api/absences` for same employee: assert availableBalance never drops below zero and holds are respected.
- End-to-end scenario: register user → create employee → request leave → manager approve → HR approve → verify balances & attendance marked `OnLeave`.

## Smoke tests (docker-compose)
- Start stack: RabbitMQ, Jaeger, user, employee, absence, attendance
- Health checks: `GET /health` for each service
- Sample flow: run the E2E scenario via provided scripts/curl

## Acceptance checklist
- [ ] Hold-on-request implemented and verified under concurrency
- [ ] Saga persists state and transitions correctly
- [ ] Transactional outbox used for event publication
- [ ] Idempotency checks in HTTP and event handlers
- [ ] OpenTelemetry traces visible in Jaeger for cross-service flow
- [ ] API docs (OpenAPI/Swagger) available for AbsenceService
