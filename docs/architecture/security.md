# Security & PII retention

## Authentication & Authorization
- `UserService` issues JWTs using ASP.NET Identity. Services validate tokens and check claims.
- Roles: `employee`, `manager`, `hr`, `admin`.
- Manager approval: verify the caller is the employee's manager (via `EmployeeService` or manager claims).
- HR approval: requires `hr` role claim.

## Service-to-service auth
- Validate JWTs for service-to-service calls or use mTLS/service tokens in production.

## Audit & logging
- Approval actions must be audited (actor id, timestamp, leaveId, action, comments).
- Store audit logs in append-only storage and include retention/archival policies.

## PII retention (standard defaults)
- Employment records (contracts, HR files): retain until employment end + 7 years.
- Leave records & balances: retain for 6 years after termination.
- Audit logs (authentication/approval): retain 3 years.
- Backups: encrypted, retained 90 days.
- Anonymization: provide an anonymize/delete API to remove PII while retaining legal compliance records where required.

## Data protection
- Encrypt data at rest and in transit.
- Mask PII in logs where feasible.

## Compliance
- Document retention policies and provide export/deletion endpoints to satisfy GDPR/other legal requests.
