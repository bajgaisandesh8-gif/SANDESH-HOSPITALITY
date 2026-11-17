# SANDESH HOSPITALITY
## System Architecture

**Document status:** Initial architecture baseline — for review  
**Version:** 0.1.0  
**Related document:** `docs/product/PRODUCT-REQUIREMENTS.md`  
**Platform:** V1 web-first, cloud-first SaaS

> This document describes the proposed system boundaries and architectural constraints. It is not proof that any component has been implemented or deployed. Decisions marked as proposed or open must be reviewed before implementation.

---

## 1. Architecture goals

The architecture must support:

- Multiple organizations (tenants), with multiple properties per organization.
- Strict tenant and property data isolation.
- Reliable reservation, inventory, and financial transactions.
- Secure authentication and server-side authorization.
- Modular hotel operations.
- A controlled automation engine.
- Auditable business actions.
- Incremental delivery and independent testing.
- A cloud-first web application in V1.

The architecture should avoid premature microservices, unnecessary infrastructure, and duplicate authorization logic.

## 2. High-level system context

V1 is a web application accessed through supported browsers.

Primary request path:

1. A user accesses the Next.js application.
2. Next.js renders the UI and handles trusted server-side application requests.
3. The application authenticates the user and resolves their authorized organization/property context.
4. Business operations pass through validated application services and approved database operations.
5. PostgreSQL enforces constraints and row-level access policies as a defense-in-depth boundary.
6. Business events are recorded transactionally for asynchronous processing where needed.
7. External providers are called through controlled integration boundaries.

Conceptual flow:

```text
Browser
   |
   v
Next.js Web Application
   |-- Server Components / UI
   |-- Server Actions or Route Handlers (when appropriate)
   |-- Application Services
   |-- Authentication & Authorization
   |
   v
Supabase Platform
   |-- PostgreSQL
   |-- Supabase Auth
   |-- Storage (when required)
   |-- Realtime (only for justified use cases)
   |
   +--> Transactional Event Outbox
              |
              v
       Automation Worker / Runner
              |
              +--> Approved internal actions
              +--> Email / notification providers
              +--> Platform-owned integrations
```

This is a logical architecture, not a deployment diagram. Exact worker hosting, queue technology, and process topology remain open until workload and reliability requirements are known.

## 3. Proposed technology direction

| Layer | Proposed technology | Responsibility |
|---|---|---|
| Web application | Next.js + React | User interface and web application |
| Language | TypeScript | Typed application code |
| Styling | Tailwind CSS | UI styling |
| Primary database | Supabase PostgreSQL | Durable business data and transactions |
| Authentication | Supabase Auth | Identity and session lifecycle |
| Object storage | Supabase Storage | Approved uploaded files |
| Realtime | Supabase Realtime, selectively | Live updates where justified |
| Hosting | Vercel | Web application hosting |
| Source control | GitHub | Version history and review |
| Automation | Application-owned workflow engine | Tenant workflow execution |

This is a proposed stack based on the current product direction. Package versions and provider capabilities must be verified against current official documentation before implementation.

Do not add a separate backend framework or microservice merely because it is available. Introduce a separate service only when there is a documented requirement that cannot be met cleanly within the existing application boundary.

## 4. Application architecture

Organize the application around business domains rather than one large collection of unrelated components.

Proposed domain boundaries:

- Identity and access.
- Organizations and properties.
- Rooms and inventory.
- Reservations.
- Stays and front desk.
- Guests.
- Folios and payments.
- Restaurant and KOT.
- Housekeeping.
- Notifications.
- Reporting.
- Automation.
- Integrations.

Each domain should own its validation rules, application use cases, and data-access boundary. Cross-domain operations must use explicit application interfaces or well-defined database transactions rather than arbitrary table writes from UI components.

### Next.js usage principles

- Prefer Server Components for server-rendered, data-oriented UI by default.
- Use Client Components only where browser interactivity requires them.
- Keep privileged logic and secrets on the server.
- Use Route Handlers for suitable HTTP endpoints and webhooks.
- Use Server Actions only for appropriate application mutations with explicit validation and authorization.
- Do not treat Server Actions, hidden buttons, or unguessable URLs as authorization controls.
- Validate all untrusted input at the server boundary.
- Keep business rules out of presentation components.
- Avoid exposing database service credentials or privileged provider keys to the browser.

Exact folder structure and conventions should be defined in the development workflow and coding standards after reviewing the current Next.js version documentation.

## 5. Backend and data-access boundaries

Supabase PostgreSQL is the proposed system of record for core business data.

All data access must follow a documented trust boundary:

1. Identify the authenticated user.
2. Resolve the user's organization and property memberships from trusted records.
3. Verify the requested action against the permission model.
4. Validate input and business invariants.
5. Execute the operation through an approved data-access path.
6. Record audit information for sensitive business actions.
7. Return only data the caller is authorized to see.

### Write-path rules

- Simple, low-risk CRUD may use a narrowly scoped server-side data-access function with RLS protection.
- Critical reservation, room-inventory, folio, and payment operations must use reviewed transactional database functions or another explicitly approved atomic transaction boundary.
- Do not let the browser directly perform privileged writes.
- Do not allow generic client-controlled table or column selection for sensitive operations.
- Do not duplicate conflicting permission rules across unrelated layers. Define a central permission model and enforce it at trusted application and database boundaries.

Whether each operation uses a Postgres function, server-side transaction, or a combination must be specified in the domain/database design.

## 6. Authentication and authorization

Supabase Auth is the proposed identity provider.

Authentication answers: who is the user?

Authorization answers: what may that user do, and within which organization/property?

These must remain separate concepts.

### Required authorization model

- Organization membership.
- Property membership or explicitly granted property scope.
- Role assignment.
- Permission/action mapping.
- Active/inactive membership status.
- Explicit cross-property access where required.
- Server-side checks for every sensitive operation.
- Database RLS as defense in depth.

Do not use user-editable profile metadata as an authorization source. Authorization records must live in trusted database-controlled structures. If JWT claims are used for selected authorization data, token freshness and revocation behavior must be designed explicitly.

### Session handling

Use the current supported Supabase SSR integration for Next.js after verifying official documentation. Cookie/session handling, refresh behavior, logout, and sensitive-operation checks must be tested. Do not assume that deleting a user instantly invalidates every already-issued access token.

The final role and permission matrix is a prerequisite for implementing protected modules.

## 7. Multi-tenancy and data isolation

The organization is the primary tenant boundary. Properties belong to an organization.

Tenant-specific records must carry consistent organization scope. Property-specific records must also carry property scope where applicable.

Required design constraints:

- Enforce tenant relationships with database constraints and, where appropriate, composite foreign keys.
- Ensure a child record cannot reference a property belonging to another organization.
- Enable RLS on every table in an exposed schema that contains tenant data.
- Write policies based on the actual membership and permission model.
- Do not assume that checking only for an authenticated role provides authorization.
- Protect views; review their security behavior and use security-invoker behavior where appropriate and supported.
- Review privileged functions carefully. Prefer invoker permissions; any required definer function must have a documented justification, narrow execution grants, safe search path, and explicit authorization checks.
- Test cross-tenant and cross-property access attempts automatically.

The exact schema and policy patterns belong in `docs/database/DATABASE-DESIGN.md` and `docs/security/SECURITY-ARCHITECTURE.md`.

## 8. Core business transaction boundaries

### Reservations and room inventory

The reservation domain must preserve inventory integrity under concurrent requests.

The detailed design must specify:
- Inventory model and per-night availability.
- Reservation holds and expiration.
- Atomic booking confirmation.
- Physical-room overlap prevention.
- Ordered locking or another proven concurrency strategy.
- Idempotency keys.
- Cancellation and modification behavior.
- Time-zone and stay-date rules.

Do not implement availability as a simple frontend check followed by an unprotected insert.

### Folios and payments

The financial domain must preserve an auditable history.

The detailed design must specify:
- Append-only or otherwise tamper-evident financial ledger approach.
- Charge, payment, refund, reversal, and adjustment semantics.
- Idempotent payment processing.
- Provider webhook verification.
- Reconciliation.
- Invoice numbering and fiscal rules by market.
- Authorization and approval requirements.

Never store raw card details unless a separately approved compliant design requires it. Prefer a suitable payment provider's hosted or tokenized payment flow; provider choice remains open.

## 9. Event-driven processing and automation

Business transactions that need asynchronous follow-up should write an event to a transactional outbox in the same database transaction as the business change.

Proposed flow:

1. A business transaction commits its data change and outbox event atomically.
2. A worker claims eligible events.
3. The worker dispatches the event to a versioned handler or workflow.
4. The consumer checks idempotency and tenant/property scope.
5. Failed attempts are retried with bounded backoff.
6. Repeated failures become visible in a dead-letter state for review.

Requirements:
- At-least-once delivery must be assumed.
- Consumers must be idempotent.
- Events need stable identifiers and schema versions.
- Retry and dead-letter behavior must be explicit.
- Workflow recursion and loops must be bounded.
- Tenant rate and cost limits must be enforced.
- Secrets must be stored outside workflow definitions.
- Automation execution must not block core reservation or payment transactions.

### Workflow engine boundary

Tenant-created workflows will be stored as versioned, constrained JSON/DSL and executed by an application-owned workflow engine.

The engine will expose only an approved action catalog. It must not execute arbitrary JavaScript, arbitrary SQL, or unrestricted outbound HTTP supplied by tenants.

n8n is not the tenant workflow runtime. It may be evaluated only for platform-owned integrations using fixed, signed, narrowly scoped calls. Licensing and commercial redistribution terms must be reviewed before embedding or reselling it.

The worker runtime, scheduling mechanism, queue implementation, and scaling model remain open architecture decisions.

## 10. Storage, notifications, and integrations

### File storage

Use object storage only for approved file types and business use cases. Define:
- Private versus public buckets.
- Tenant/property ownership metadata.
- Upload size and content-type validation.
- Access policies and signed URL expiry.
- Retention and deletion rules.
- Malware scanning where the risk warrants it.

### Notifications

Notification delivery should be asynchronous where possible. Persist delivery state and provider identifiers without storing unnecessary sensitive content. Handle retries and duplicate delivery safely.

### External integrations

All external integrations must have:
- A named owner and documented purpose.
- Server-side credential storage.
- Narrow scopes and least privilege.
- Input/output validation.
- Timeouts and retry limits.
- Auditability.
- Failure handling.
- Tenant-specific authorization and rate limits where applicable.

OTA/channel manager, payment gateway, email provider, biometric devices, and other vendors are not selected yet.

## 11. Deployment and environments

Proposed environments:

- Local development.
- Shared staging.
- Production.

Each environment must have separate credentials and appropriate data boundaries. Production data must not be copied into development without an approved sanitization process.

### Deployment direction

- GitHub is the source of truth.
- Feature branches and reviewed pull requests are preferred.
- Vercel is the proposed web hosting platform.
- Supabase environments/projects must be separated appropriately for staging and production.
- Database changes must be versioned migrations.
- Production migrations require review and a recovery plan.
- Environment secrets must be configured in the hosting/provider secret stores, never committed to Git.

No Vercel or Supabase project has been provisioned by this document. Deployment topology, domain, CI checks, preview environments, and release approvals must be agreed before production deployment.

## 12. Observability and operations

The platform should provide:
- Structured application logs.
- Error and exception monitoring.
- Security and business audit logs.
- Health checks for critical services.
- Automation execution visibility.
- Queue/outbox lag and failure monitoring.
- Database performance monitoring.
- Alerts for critical failures.

Logs must not expose passwords, tokens, service keys, payment secrets, or unnecessary guest personal data.

Define service-level objectives, alert thresholds, retention, and on-call ownership before launch.

## 13. Backup, recovery, and business continuity

Before production:
- Select backup and point-in-time recovery capabilities.
- Define recovery point objective (RPO) and recovery time objective (RTO).
- Document restore procedures.
- Test restoration in a non-production environment.
- Document provider outage procedures.
- Document manual hotel operations during internet or platform outages.

Provider backup availability alone is not proof that recovery works. A restore test is required.

## 14. Security architecture review checklist

Before a module is considered ready:
- Authentication and authorization are verified.
- Organization and property boundaries are tested.
- RLS is enabled and tested for exposed tenant tables.
- Privileged credentials remain server-side.
- Input validation and output encoding are applied.
- Sensitive operations are transactional and idempotent where needed.
- Audit events are recorded.
- Secrets and personal data are not leaked in logs.
- Automated tests cover denied as well as permitted access.
- Dependency and configuration checks are performed.

## 15. Architecture decisions still open

The following require explicit review and approval:

1. Exact Next.js server/API conventions and module folder layout.
2. Supabase project/environment topology.
3. Final organization/property membership and permission schema.
4. Which critical operations use Postgres functions versus server-side transaction boundaries.
5. Automation worker hosting and queue/scheduler technology.
6. Email and notification providers.
7. Payment gateway and webhook design.
8. OTA/channel manager integration approach.
9. Storage bucket and retention strategy.
10. Observability provider and alert ownership.
11. RPO, RTO, backup retention, and restore-test schedule.
12. Target countries, data residency, privacy, and fiscal requirements.
13. Production release and migration approval process.

## 16. Architecture governance

- This document is a proposal until reviewed and approved by the product owner.
- Do not implement unresolved decisions by guessing.
- Keep architecture and implementation synchronized.
- Record material decisions and trade-offs in an Architecture Decision Record (ADR).
- Review changes that affect tenant isolation, money, inventory, authentication, or automation security before merging.
- Prefer the simplest architecture that satisfies verified requirements.
- Avoid premature microservices and infrastructure complexity.

**Status:** Initial baseline — not yet approved.  
**Next documents:** Database design, security architecture, and automation engine.
