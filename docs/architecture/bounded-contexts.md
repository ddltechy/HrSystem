# Bounded contexts & aggregates

This file describes the bounded contexts, aggregate roots, and ownership model for the HRSystem.

## Bounded contexts

- UserService (Identity)
  - Aggregate root: `UserAccount`
  - Responsibility: authentication, authorization, JWT issuance, user lifecycle

- EmployeeService (Core HR)
  - Aggregate root: `Employee`
  - Responsibility: employee profiles, contract records, manager relationships, employment status

- AbsenceService (Time-off)
  - Aggregate root: `LeaveRequest`
  - Responsibility: leave balances, accruals, request lifecycle, approvals (manager → HR), holds

- AttendanceService (Presence)
  - Aggregate root: `DailyAttendance`
  - Responsibility: recording check-ins/out, marking `OnLeave` days, publishing `AttendanceRecorded`

- PerformanceService (Evaluations)
  - Aggregate root: `Evaluation` / `EmployeePerformance`
  - Responsibility: reviews, KPI aggregation, feedback collection

## Ownership & data partitioning
- Each microservice owns its schema (one DB per service). Services communicate via domain events.
- AbsenceService is authoritative for leave balances and accrual logic.
- EmployeeService is authoritative for employee status and manager relationships (queried synchronously when necessary).

## Ubiquitous language (selected terms)
- LeaveRequest: an employee-initiated time-off request (Pending, ManagerApproved, HRApproved, Rejected)
- Hold: temporary reservation of leave days while approval is pending
- Finalize: operation that converts a hold into a permanent deduction
- Domain event: immutable notification published when aggregates change state

## Event-driven integration
- Use MassTransit + RabbitMQ for reliable delivery. Events are versioned; consumers are idempotent.

## Diagram
Refer to `docs/architecture/sequence-diagrams.puml` for the Absence saga sequence diagram.
