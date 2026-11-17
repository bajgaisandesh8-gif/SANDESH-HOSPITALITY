# Testing and Release Strategy

**Status:** Initial proposal — review required before implementation  
**Applies to:** SANDESH HOSPITALITY V1  
**Related:** System Architecture, Database Design, Security Architecture, API Design, Automation Engine

## 1. Purpose

Define the minimum evidence required before a feature can be merged, deployed to staging, or released to production. A successful build alone is not evidence that a hotel-management workflow is safe.

## 2. Test layers

### Unit tests
Use for deterministic domain rules: date boundaries, rate calculations, permissions, workflow condition evaluation, state transitions, and validation. Keep business rules independent from UI components.

### Database tests
Use migrations against a disposable test database. Add pgTAP or equivalent tests for:
- Row-level security for every tenant-scoped table.
- Cross-organization and cross-property access denial.
- Composite foreign-key integrity and required uniqueness.
- Reservation overlap and inventory constraints.
- Atomic financial ledger behavior.
- Function execution grants and SECURITY DEFINER boundaries.
- Migration upgrade from a clean database and supported prior schema.

### API/integration tests
Test authenticated and unauthenticated calls, role/property authorization, validation, idempotency, pagination, rate limits, error handling, and transaction rollback. Test that service credentials are never returned to clients.

### End-to-end tests
Cover critical staff journeys:
- Sign in and select an authorized property.
- Create/search/confirm a reservation.
- Assign a room and check in.
- Record a payment and view the folio.
- Check out and create housekeeping work.
- Handle a walk-in using the same inventory controls.
- Reject unauthorized access to another tenant/property.
- Recover safely from duplicate submissions and transient failures.

### Automation tests
Test trigger delivery, event schema versions, duplicate delivery, retry/backoff, dead-letter handling, tenant scoping, loop limits, disabled/draft workflows, test mode, and failure isolation. Test mode must never execute real external actions.

### Security tests
Include dependency and secret scanning, static analysis, security headers/CSP checks, authorization regression tests, input validation, storage access policies, webhook signature validation, and manual review of high-risk database functions.

## 3. Critical invariants

Release-blocking tests must prove:
1. No cross-tenant data read or write is possible through UI, API, database, storage, or automation.
2. Concurrent booking/assignment operations cannot oversell or allocate one physical room to overlapping stays.
3. Financial entries are not silently overwritten or deleted.
4. Payment success is recorded only after authoritative provider confirmation or an explicitly approved manual process.
5. Check-in, checkout, and payment operations are idempotent.
6. Every sensitive mutation has an attributable audit record.
7. Automation outages do not block core reservation, check-in, checkout, or payment flows.
8. Secrets and service-role credentials never reach browser code, logs, or workflow JSON.

## 4. Environments

- **Local:** developer-owned environment using non-production data and test credentials.
- **CI:** isolated automated test environment; no production secrets or real payment actions.
- **Staging:** production-like configuration with synthetic/test data and payment sandbox credentials.
- **Production:** separately configured project, restricted access, monitored migrations, verified backups.

Never use real guest personal data in ordinary development fixtures. Use synthetic records.

## 5. CI checks

A pull request should pass:
- Formatting/lint.
- TypeScript type checking.
- Unit tests.
- Database migration and policy tests.
- API/integration tests relevant to changed modules.
- Build.
- Dependency and secret scanning.
- End-to-end smoke tests for critical paths when practical.

CI should fail closed on missing required checks. Do not deploy production directly from an unreviewed local working tree.

## 6. Release gates

### Feature gate
- Requirements and acceptance criteria reviewed.
- Threat/permission impact assessed.
- Database migration reviewed and tested.
- Tests cover success, failure, concurrency, and authorization.
- Logs and audit events are adequate without leaking sensitive data.
- Rollback or forward-fix procedure documented.

### Staging gate
- Deployment succeeds.
- Smoke tests pass.
- RLS and role tests pass against the deployed environment.
- Backup/restore and migration status are checked.
- External integrations use sandbox/test credentials.

### Production gate
- Explicit release approval.
- Backup and recovery readiness confirmed.
- Migrations are backward-compatible or have a documented maintenance plan.
- Monitoring and incident contacts are ready.
- Rollback/forward-fix plan is understood.
- No known critical security or data-integrity defect remains.

## 7. Observability

Capture structured logs, request/correlation IDs, tenant/property context where safe, error rates, latency, job/automation queue health, webhook failures, and database health. Do not log passwords, tokens, full payment credentials, or unnecessary guest personal data.

Alert on:
- Authentication/authorization anomalies.
- Cross-tenant access test failures.
- Reservation conflict/transaction failures.
- Payment reconciliation discrepancies.
- Growing outbox/dead-letter backlog.
- Backup failures and elevated application error rates.

## 8. Backup and recovery

Before production, define and test:
- Recovery Point Objective (RPO) and Recovery Time Objective (RTO).
- Backup frequency and retention.
- Point-in-time recovery availability.
- Restore procedure and responsible operator.
- Periodic restore drills.
- Incident communication and data-integrity verification after restoration.

A backup that has never been restored in a test is not a proven recovery plan.

## 9. Current project status

The repository currently contains planning documents and a Next.js scaffold. The presence of a successful scaffold build does not mean the product's database, security, or operational acceptance criteria have been tested. These checks become applicable as implementation begins.
