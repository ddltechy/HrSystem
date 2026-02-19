# Event catalog — HRSystem (selected events)

All events MUST include metadata: `eventId`, `eventVersion`, `occurredAt`, `correlationId`, `causationId`, `traceId`.

| Event | Producer | Consumers | Purpose | Key fields |
|---|---:|---|---|---|
| `UserRegistered` | UserService | EmployeeService (optional) | Seed account to employee link | `userId`, `email`, `createdAt` |
| `EmployeeCreated` | EmployeeService | AbsenceService, AttendanceService, PerformanceService | Populate local read models | `employeeId`, `userId`, `position`, `startDate` |
| `LeaveRequested` | AbsenceService | (internal saga) | Start approval workflow | `leaveId`, `employeeId`, `period`, `type` |
| `LeaveHoldPlaced` | AbsenceService | Monitoring / audit | Indicate hold reserved for request | `leaveId`, `employeeId`, `heldDays` |
| `LeaveManagerApproved` | AbsenceService | Audit / notifications | Manager step completed | `leaveId`, `approverId` |
| `LeaveHRApproved` | AbsenceService | Audit / notifications | HR step completed | `leaveId`, `hrApproverId` |
| `LeaveFinalized` | AbsenceService | AttendanceService, PerformanceService | Final deduction and calendar update | `leaveId`, `employeeId`, `deductedDays`, `period` |
| `LeaveHoldReleased` | AbsenceService | Audit | Hold released due to rejection/timeouts | `leaveId`, `reason` |
| `AttendanceRecorded` | AttendanceService | PerformanceService | Presence recorded for KPIs | `attendanceId`, `employeeId`, `date`, `checkIn`, `checkOut` |

## Versioning & compatibility
- Use semantic event versioning in `eventVersion` and keep old event handlers until migration is complete.
- Avoid breaking changes; prefer additive fields and new event types (e.g., `LeaveFinalized.v2`) when necessary.

## Delivery guarantees & idempotency
- Use transactional Outbox pattern in AbsenceService to ensure events are published only after DB commit.
- Consumers must be idempotent; use `eventId` de-dup table or idempotency middleware.

## Headers & tracing
- Propagate `traceId` and `correlationId` in message headers for distributed tracing.
