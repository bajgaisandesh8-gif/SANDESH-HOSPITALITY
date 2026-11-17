# Implementation Roadmap

**Status:** Proposed delivery sequence — not a delivery-date commitment  
**Project stage:** Planning and repository foundation  
**Primary rule:** Do not begin a phase until its entry criteria are met.

## Phase 0 — Repository and local foundation

### Goals
- Synchronize the local Next.js scaffold with the GitHub repository.
- Verify the actual committed Node/package versions and scripts.
- Establish lint, typecheck, build, and CI checks.
- Establish environment-variable examples without real secrets.
- Confirm branch and pull-request workflow.

### Exit criteria
- A clean clone of `main` can install dependencies and run the documented checks.
- The app scaffold is committed and verified on GitHub.
- No secrets or local environment files are committed.
- CI runs on pull requests.

## Phase 1 — Product decisions and security foundation

### Goals
- Confirm launch country, target hotel segment, organization/property model, and V1 boundary.
- Resolve the initial role and permission model.
- Decide authentication, invitation, membership, and property-selection flows.
- Define tenant and property isolation tests before business data tables are exposed.

### Exit criteria
- Approved requirements for identity, organization, property, membership, and permissions.
- Threat model and RLS strategy reviewed.
- Database tests prove cross-tenant and unauthorized-property access is denied.
- No business module can bypass the shared authorization model.

## Phase 2 — Property configuration

### Goals
- Organization and property setup.
- Property timezone, locale, currency configuration strategy.
- Room types, rooms, capacities, and operational statuses.
- Staff membership and property-scoped permissions.

### Exit criteria
- Authorized users can manage only their assigned organizations/properties.
- Configuration changes are audited.
- Property timezone and business-date behavior are tested.

## Phase 3 — Reservation and inventory core

### Goals
- Availability search.
- Rate plans and booking rules, once defined.
- Reservation creation, update, confirmation, cancellation, and no-show lifecycle.
- Room-type inventory and physical room assignment.
- Concurrency-safe holds and idempotent mutations.

### Exit criteria
- Concurrent requests cannot oversell inventory or allocate one physical room to overlapping stays.
- Reservation transitions follow the approved lifecycle.
- Duplicate submissions do not create duplicate bookings.
- All cross-tenant and cross-property tests pass.

## Phase 4 — Front desk and stay operations

### Goals
- Arrivals, in-house, and departures dashboard.
- Walk-in reservations through the standard booking path.
- Check-in, room moves, guest requests, and checkout.
- Housekeeping handoff.
- Shift handover and documented internet-outage procedures.

### Exit criteria
- Check-in and checkout are atomic and idempotent.
- Room status and stay status cannot drift through unsupported transitions.
- Staff permissions and audit records are verified.
- Manual outage reconciliation is documented and testable.

## Phase 5 — Folio and payments

### Entry blockers
- Launch jurisdiction, tax/invoice rules, currency, payment provider, refund policy, and settlement behavior must be decided.

### Goals
- Append-only folio/ledger model.
- Payment intent and provider confirmation flow.
- Reconciliation, refunds, adjustments, and cashier controls.
- Financial reporting and night-audit behavior.

### Exit criteria
- No payment is marked successful without authoritative confirmation or an explicitly approved manual process.
- Financial records preserve history.
- Authorization and separation-of-duties tests pass.
- Reconciliation behavior is documented and tested.

## Phase 6 — Automation foundation

### Goals
- Transactional event outbox.
- Versioned event contracts.
- Platform-owned workflow runner.
- Ready-made templates, execution history, retries, idempotency, and dead-letter handling.
- Constrained visual workflow builder after the underlying engine is proven.

### Exit criteria
- Duplicate events do not duplicate business actions.
- Automation failures do not block core hotel operations.
- Tenant scope and action permissions are enforced.
- Test mode cannot execute real-world side effects.

## Phase 7 — Additional modules and integrations

Candidate modules include restaurant POS, kitchen tickets, QR room ordering, concierge, reporting, attendance integrations, and external booking channels.

Each module requires:
- Approved product scope.
- Provider/vendor and licensing review where applicable.
- Data and permission design.
- Failure and reconciliation procedures.
- Tests and release criteria.

Do not treat this list as a commitment that every module ships in V1.

## Production readiness gate

Production launch requires, at minimum:
- Approved target market and legal/tax handling.
- Security review and tenant-isolation evidence.
- Tested backups and restore procedure.
- Defined RPO/RTO, monitoring, incident response, and support ownership.
- Payment/provider certification and reconciliation tests where payments are enabled.
- Data retention/export/deletion policy.
- Staging acceptance and explicit release approval.

## Current next action

Complete Phase 0 by getting the local scaffold into the GitHub repository, then validate a clean clone and CI. Do not create production database tables until Phase 1 decisions and security tests are ready.
