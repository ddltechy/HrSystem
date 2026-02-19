# Architecture overview — HRSystem

## Purpose
This repository contains the design and (planned) microservice scaffold for a Domain-Driven HR application composed of independent bounded contexts: `UserService`, `EmployeeService`, `AbsenceService`, `AttendanceService`, and `PerformanceService`.

## High-level decisions
- Architecture style: **DDD + microservices** with per-service persistence and clear bounded contexts.
- Communication: **Event-driven** (RabbitMQ + MassTransit) for integration; synchronous calls only when strong consistency is required.
- Auth: **UserService** with ASP.NET Identity + JWT (service-to-service auth via validated tokens).
- Patterns: **CQRS (read models)**, transactional Outbox, MassTransit Sagas for long-running workflows.
- Local dev: **Docker Compose** (RabbitMQ, Jaeger, services using SQLite). Production: recommend Kubernetes + Postgres/SQL Server.
- .NET target: **.NET 8 (LTS)**

## Components
- UserService — Identity & JWT issuance
- EmployeeService — employee profiles, contracts, manager relationships
- AbsenceService — leave requests, approvals, balances (authoritative)
- AttendanceService — check-ins, daily presence, integrates with AbsenceService
- PerformanceService — evaluations, KPIs, feedback loops
- API Gateway — Ocelot for unified surface (local/dev)
- Observability: OpenTelemetry (Jaeger + metrics)

## Where to find details
See `docs/architecture/*` for the bounded-context map, AbsenceService saga, API contracts, and event catalog.
