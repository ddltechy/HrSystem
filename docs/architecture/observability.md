# Observability — HRSystem

## Tracing
- Use OpenTelemetry to instrument API requests, message handlers, and saga transitions.
- Propagate `traceId` and `correlationId` in HTTP headers and message headers.
- Local dev: export traces to Jaeger; production: export to OTLP collector.

## Metrics
- Expose `/metrics` (Prometheus format). Collect:
  - request latency (per-route)
  - message processing success/failure counts
  - saga state counts
  - heldDays and availableBalance gauges for spot checks

## Logs
- Structured JSON logs (timestamp, level, service, correlationId, traceId, actorId, message)
- Include actor id and roles for audit events (approvals, rejections)

## Dashboards
- Basic dashboards: service latency, error rate, message backlog (RabbitMQ), pending approvals

## Tools (local)
- Jaeger: tracing UI
- Prometheus + Grafana: metrics & dashboards
- RabbitMQ management UI: queue health

## Instrumentation notes
- Ensure MassTransit instrumentation is enabled for outgoing/incoming messages.
- Make consumers idempotent and log duplicate event occurrences for monitoring.
