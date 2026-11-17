# API Design

**Status:** Initial conceptual baseline — not an approved API contract  
**Product:** SANDESH HOSPITALITY  
**Scope:** V1 web-first, cloud-first platform

This document proposes API conventions and candidate resource boundaries for SANDESH HOSPITALITY. It is a design baseline, not evidence that any endpoint or backend service has been implemented. Final contracts must be reviewed alongside the product requirements, system architecture, database design, and security architecture.

## 1. Goals and design principles

The API must support a multi-tenant hospitality platform with multiple organizations and properties while preserving strict data isolation and business integrity.

Principles:

- Use HTTPS for all network communication.
- Use JSON for request and response bodies unless a specific integration requires another format.
- Keep business rules in server-side domain/application services; do not trust browser calculations for inventory, payments, permissions, or financial totals.
- Treat the authenticated user's identity and organization/property membership as trusted only after server-side validation.
- Apply authorization at the application boundary and enforce database Row Level Security (RLS) as defense in depth.
- Validate every request and return explicit response DTOs rather than exposing database rows directly.
- Use stable error codes, request correlation IDs, pagination limits, idempotency, and concurrency controls.
- Keep public app APIs separate from internal service calls and third-party provider webhooks.
- Do not expose Supabase service-role credentials or other privileged secrets to browsers.

## 2. API boundary and runtime

### 2.1 V1 application API

Candidate approach:

- Next.js Route Handlers provide the public application API boundary for the web client.
- Route Handlers validate the Supabase Auth session and resolve the user's active organization/property context.
- Domain/application services perform business operations and enforce permissions.
- Database access uses appropriately scoped server-side clients and controlled write paths.
- Supabase RLS remains a second enforcement layer for tenant-owned data.

The browser must not perform privileged direct database writes for reservations, room inventory, payments, refunds, folio postings, or other integrity-sensitive operations.

### 2.2 Edge Functions and internal services

Use Supabase Edge Functions only where a separately deployed function is justified, such as a provider webhook receiver or a narrowly scoped integration endpoint. Avoid implementing the same authorization and business rules independently in multiple runtimes.

Internal services should expose explicit, typed operations. They must not become a bypass around tenant scope, permission checks, audit logging, or database constraints.

## 3. URL and versioning conventions

Candidate resource-oriented structure:

`/api/v1/{resource}`

Examples below are illustrative, not committed endpoint contracts. The project must decide whether versioning belongs in the URL or headers before external integrations are published.

Use plural resource names, predictable nesting, and HTTP methods consistently. Avoid deeply nested URLs when a resource can be addressed directly by its own identifier.

Do not expose internal database table names as a permanent public contract. API DTOs and database schemas may evolve independently.

## 4. Candidate resource groups

| Area | Candidate resources and operations |
|---|---|
| Organizations | organizations, organization settings, organization context |
| Properties | properties, property settings, operating configuration |
| Identity and access | users, memberships, roles, permissions, invitations |
| Inventory setup | room types, rooms, rate plans, inventory configuration |
| Availability | availability searches, date ranges, room-type availability |
| Reservations | reservations, reservation changes, cancellations, holds |
| Guests | guest profiles, contact details, preferences, consent records |
| Front desk | arrivals, departures, in-house stays, check-in, check-out |
| Folios and finance | folios, folio entries, invoices, payment records, refunds |
| Restaurant | menus, tables, orders, order items, kitchen tickets |
| Housekeeping | tasks, room status, assignments, completion |
| Maintenance | maintenance requests, assignments, status changes |
| Automation | workflows, versions, test runs, executions, enable/disable |
| Reporting | report definitions, report jobs, generated report downloads |
| Integrations | provider connections, incoming webhooks, outgoing webhook subscriptions |
| Operations | health and readiness checks, where appropriate |

These groups need prioritization. They do not imply that all modules belong in the first implementation milestone.

## 5. HTTP methods and status codes

Use standard HTTP semantics:

- `GET`: retrieve a resource or collection; must not mutate state.
- `POST`: create a resource or execute a clearly defined command.
- `PATCH`: partially update a resource.
- `PUT`: replace a resource only where full replacement semantics are appropriate.
- `DELETE`: delete or deactivate a resource where the domain permits it; financial and audit records generally require controlled reversal or archival rather than destructive deletion.

Common status codes:

| Status | Intended use |
|---|---|
| 200 | Successful retrieval or update |
| 201 | Resource created |
| 202 | Accepted asynchronous operation |
| 204 | Successful operation with no response body |
| 400 | Malformed request or invalid request structure |
| 401 | Missing or invalid authentication |
| 403 | Authenticated user lacks permission |
| 404 | Resource not found or intentionally not disclosed |
| 409 | State, inventory, idempotency, or concurrency conflict |
| 422 | Valid request structure but domain validation failed |
| 429 | Rate limit exceeded |
| 500 | Unexpected server failure |
| 503 | Temporarily unavailable dependency or service |

Use status codes consistently. Do not return 200 for failed business operations.

## 6. Request and response conventions

### 6.1 Validation and DTOs

- Validate path parameters, query parameters, headers, and request bodies on the server.
- Enforce maximum lengths, numeric bounds, enum values, date ranges, and payload-size limits.
- Reject unknown or forbidden fields where accepting them could cause over-posting or privilege escalation.
- Never accept client-supplied organization IDs, property IDs, role names, prices, payment states, or ownership fields as proof of authorization.
- Resolve tenant and property scope from verified membership and an authorized request context.
- Return explicit response DTOs containing only fields the caller is permitted to see.
- Avoid returning raw database errors, SQL details, stack traces, secrets, or internal provider tokens.

### 6.2 Timestamps and dates

- Store timestamps consistently in UTC.
- Return timestamps in ISO 8601 format with an explicit timezone indicator.
- Keep property-local business dates and times distinct from UTC timestamps. Reservations, operating hours, night audit, and reports must use the property's configured timezone.
- Use date-only values for concepts that are genuinely calendar dates, such as a stay date, rather than converting them into arbitrary UTC instants.
- Define inclusive/exclusive date semantics consistently; reservation stay ranges should follow the database design's half-open interval convention.

### 6.3 Collection responses and pagination

Candidate response shape:

```json
{
  "data": [],
  "page": {
    "next_cursor": null
  },
  "request_id": "request-correlation-id"
}
```

This is illustrative. The exact response envelope remains an open decision.

- Prefer cursor pagination for large or frequently changing collections.
- Set a conservative default page size and a hard maximum.
- Validate sort fields against an allowlist; never interpolate arbitrary client-supplied SQL identifiers.
- Apply tenant/property scope before filtering, sorting, counting, or pagination.
- Do not expose cross-tenant counts or metadata through pagination behavior.

## 7. Error format

Candidate error shape:

```json
{
  "error": {
    "code": "RESERVATION_CONFLICT",
    "message": "The requested room inventory is no longer available.",
    "field_errors": []
  },
  "request_id": "request-correlation-id"
}
```

Requirements:

- `code` is a stable machine-readable identifier.
- `message` is safe for the intended caller and must not disclose sensitive internal details.
- `field_errors` is optional and used for actionable validation feedback.
- `request_id` allows support and operations teams to trace the request.
- Never include stack traces, SQL statements, access tokens, secrets, or raw provider responses in client-facing errors.

## 8. Authentication, authorization, and tenant scope

For every protected request:

1. Validate the Supabase Auth session server-side.
2. Resolve the user's active organization and property context from trusted membership records.
3. Confirm the requested resource belongs to that scope.
4. Check the required permission for the specific action.
5. Execute the operation through a scoped service and controlled database access path.
6. Record an audit event for sensitive or business-critical changes.

Rules:

- A client-provided tenant or property identifier is a selector, not proof of access.
- A user may belong to more than one organization or property; switching context requires membership validation.
- Check authorization on every request, including reads, exports, downloads, and background operations.
- Enforce tenant isolation in database queries and RLS policies.
- Avoid leaking whether another tenant's resource exists; use consistent not-found behavior where appropriate.
- Service-role credentials must remain server-side and must never be used as a shortcut around authorization.

## 9. Reservation and inventory operations

Availability searches are advisory. Inventory can change between a search response and a booking attempt.

Reservation creation must:

- Revalidate dates, occupancy, room type, rate, property scope, and eligibility on the server.
- Perform the final inventory check and reservation write atomically using the approved database transaction/RPC boundary.
- Apply per-night inventory rules and physical-room exclusion rules defined in the database design.
- Use a hold/expiry model where required and define what happens when a hold expires.
- Return a conflict response when inventory is no longer available.
- Avoid relying on a prior availability response as a guarantee.

Reservation modification and cancellation must also enforce current state, permission, financial rules, and idempotency. Concurrent updates should use a version field or equivalent optimistic concurrency mechanism.

## 10. Idempotency and concurrency

### 10.1 Idempotency

Require idempotency keys for operations where a retry could create duplicate business effects, including:

- Reservation creation and important reservation mutations.
- Payment intent creation, payment capture where applicable, and refunds.
- Folio postings and other money-affecting commands.
- Webhook/event consumption and automation actions that may be retried.

Define key scope using the authenticated tenant/context, operation, and caller. Store the request fingerprint and final result for a documented retention period. Reusing a key with a materially different request must return a conflict rather than silently replaying an unrelated result.

The same logical operation must not create duplicate reservations, charges, refunds, folio entries, or automation side effects when requests are retried.

### 10.2 Concurrency

- Use a version field or ETag/If-Match mechanism for mutable resources where lost updates are possible.
- Return 409 for stale versions or conflicting state transitions.
- Perform inventory and financial mutations inside explicit database transactions.
- Define lock ordering for operations that touch multiple inventory or financial records.
- Do not assume that client-side disabling of buttons prevents duplicate requests.

## 11. Payments and financial operations

- The server creates payment intents or equivalent provider-side objects using approved provider credentials.
- The browser redirect or client success screen is not proof that a payment succeeded.
- Verify payment status through authenticated provider callbacks/webhooks or server-side provider verification.
- Verify webhook signatures, timestamps, and replay protections.
- Make provider event processing idempotent.
- Store provider references and the minimum necessary payment metadata.
- Do not collect or store raw card numbers or security codes in SANDESH HOSPITALITY.
- Refunds, reversals, and folio postings must use explicit permissions, audit records, and append-only financial ledger rules.
- Never let an automation workflow directly edit payment state or financial balances outside approved domain operations.

Payment provider, supported payment methods, currencies, settlement behavior, and target jurisdictions remain open decisions.

## 12. Webhooks and integrations

Keep two directions distinct:

**Incoming provider webhooks**
- Receive events from payment or other approved providers.
- Verify signatures and replay windows before processing.
- Persist the event safely before acknowledging receipt where the provider's delivery model requires it.
- Process asynchronously and idempotently.
- Keep provider-specific parsing separate from domain operations.

**Outgoing platform webhooks**
- Deliver approved tenant-scoped events to configured customer endpoints only if this capability is approved for V1.
- Sign payloads, include event IDs and schema versions, and document retry behavior.
- Protect against SSRF, unsafe redirects, private-network destinations, DNS rebinding, and abusive destinations.
- Apply per-tenant rate limits, delivery limits, and secret rotation.
- Provide delivery history and controlled endpoint verification.

Do not expose arbitrary unrestricted HTTP requests as a tenant workflow action. External integration capabilities require a reviewed allowlist and security model.

## 13. Automation API boundaries

The automation engine may expose APIs to create, validate, test, publish, enable, disable, and inspect versioned workflows.

- Workflow definitions must use the approved constrained DSL/action catalog.
- The API must validate triggers, conditions, branches, delays, and action parameters against schemas.
- Every action must re-check tenant/property scope and user/system permissions at execution time.
- Test mode must not produce real external or financial side effects.
- Published versions must be immutable; edits create a new draft/version.
- Execution APIs must support idempotency, traceability, retry limits, and loop protection.
- Tenant workflows must call approved domain services rather than arbitrary database queries or unrestricted code.

The API must not allow a workflow editor to bypass the business rules of reservations, payments, housekeeping, or other modules.

## 14. Security and operational controls

- Apply per-user, per-tenant, and per-endpoint rate limits where appropriate.
- Use request-size limits and strict content-type handling.
- Configure CORS narrowly for approved application origins.
- Protect cookie-authenticated mutation endpoints against CSRF using an appropriate session strategy and CSRF controls.
- Use secure cookie settings and appropriate security headers.
- Avoid logging credentials, session tokens, guest-sensitive data, payment secrets, or full webhook secrets.
- Log request IDs, actor IDs, tenant/property context, operation names, outcomes, and latency while minimizing personal data.
- Keep secrets in managed server-side secret storage, not source control, browser bundles, or workflow JSON.
- Define health and readiness endpoints that reveal operational status without exposing environment variables, internal topology, or sensitive dependency details.
- Document API deprecation and compatibility policy before external consumers depend on an endpoint.

## 15. Testing requirements

Before release, API tests should cover:

- Authentication required/invalid/expired session behavior.
- Role and permission enforcement for each sensitive operation.
- Cross-organization and cross-property access attempts.
- RLS behavior and server-side tenant scoping.
- Request validation, over-posting, and safe error responses.
- Reservation inventory races and conflicting updates.
- Idempotency under duplicate and concurrent requests.
- Payment webhook signature verification and replay handling.
- Automation test-mode isolation, action permissions, and loop limits.
- Rate limiting, payload limits, and abuse cases.
- Contract tests for request/response schemas and provider integrations.
- Audit records for sensitive changes.

A successful happy-path test is not sufficient. Tenant isolation and concurrency tests are release blockers.

## 16. Open decisions

The following must be resolved before treating this document as a stable contract:

1. URL path versioning versus header-based versioning.
2. Public app API, internal service API, and partner API boundaries.
3. Final response envelope and error schema.
4. Cursor pagination format and supported filtering/sorting fields.
5. Whether external API keys or OAuth clients are needed for integrations.
6. OTA/channel-manager and other third-party integration scope.
7. Outgoing tenant webhooks in V1 or a later release.
8. Payment gateway, supported countries, currencies, and fiscal requirements.
9. Guest identity, consent, retention, and data export/deletion requirements.
10. API compatibility and deprecation policy.
11. Tenant billing and usage metering API boundaries.
12. Whether any endpoints need asynchronous job resources for long-running reports or exports.

## 17. Implementation gate

Do not begin implementing the complete API from this baseline alone. First review and approve the product requirements, architecture, database design, security architecture, and automation design together. Then define the first vertical slice—recommended starting point: organization/property context plus room setup and reservation availability/creation—with concrete schemas, authorization rules, transaction boundaries, and automated tests.
