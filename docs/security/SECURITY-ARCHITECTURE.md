# SANDESH HOSPITALITY
## Security Architecture

**Document status:** Initial security baseline — for technical review  
**Version:** 0.1.0  
**Applies to:** Web application, backend services, Supabase PostgreSQL/Auth/Storage, automation engine, and integrations

> This document defines proposed security requirements and trust boundaries. It is not evidence that these controls have been implemented or tested. Product-specific configuration and Supabase behavior must be verified against current official documentation before implementation.

---

## 1. Security objectives

SANDESH HOSPITALITY is a multi-tenant hospitality platform. A security failure could expose guest personal information, reservations, room occupancy, staff records, financial information, or operational data across organizations.

The security architecture must provide:

1. Strong organization and property data isolation.
2. Authentication and session protection.
3. Server-side authorization for every sensitive operation.
4. Database-enforced access controls using PostgreSQL Row Level Security (RLS).
5. Protection of guest, staff, and financial data.
6. Secure reservation, payment, and inventory transactions.
7. Safe automation execution with constrained capabilities.
8. Auditable privileged actions.
9. Secure integration and secret management.
10. Detection, response, backup, and recovery procedures.

Security is a product requirement, not a final-stage checklist.

## 2. System trust boundaries

### 2.1 Main trust zones

| Zone | Components | Trust level |
|---|---|---|
| User device | Browser and client-side UI | Untrusted |
| Public application edge | Vercel-hosted web entry points | Public-facing |
| Application server | Next.js server routes/actions and trusted backend services | Trusted only after authentication and validation |
| Database | Supabase PostgreSQL | System of record; enforce RLS and constraints |
| File storage | Supabase Storage | Private by default for sensitive files |
| Automation runtime | Platform-controlled workflow engine/runner | Restricted execution environment |
| External providers | Payment, email, SMS, OTA and other APIs | Third-party trust boundary |
| Operations | Platform administrators and support tooling | Privileged and audited |

### 2.2 Core rules

- Treat all browser input, headers, query parameters, cookies, and client state as untrusted.
- Never rely on hidden UI controls or client-side role checks as authorization.
- The server must derive the authenticated identity from a verified session.
- The server must verify organization, property, resource ownership, and permission for each sensitive action.
- Database RLS and constraints must independently restrict data access.
- External provider responses and webhook payloads must be validated.
- Do not allow tenant workflows to bypass application authorization or directly access the database.

## 3. Authentication and session security

Supabase Auth is the proposed identity provider.

Requirements:
- Use supported, maintained authentication flows.
- Use secure session handling appropriate for a Next.js server-rendered application.
- Validate the current user on the server before sensitive operations.
- Do not trust a user ID supplied by the browser.
- Protect session cookies with appropriate Secure, HttpOnly, and SameSite settings where applicable.
- Avoid storing long-lived access tokens in browser local storage.
- Apply rate limits and abuse protection to sign-in, password recovery, invitation, and other authentication endpoints.
- Require stronger authentication for platform administrators and high-risk tenant operations when supported by the final identity design.
- Define account recovery, staff offboarding, session revocation, and lost-device procedures.
- Keep session lifetimes and reauthentication requirements appropriate to risk.

Important Supabase considerations to verify during implementation:
- Never use user-editable user_metadata/raw_user_meta_data claims for authorization.
- If app_metadata or JWT claims are used for authorization, account for claim staleness until token refresh.
- Deleting a user does not necessarily invalidate already-issued access tokens immediately. Revoke/sign out sessions and define sensitive-operation session validation where strict revocation is required.
- Verify current SDK and SSR guidance before implementing session refresh and cookie handling.

## 4. Authorization model

Authentication answers who the user is. Authorization answers what that user may do, to which organization, property, and resource.

### 4.1 Proposed permission structure

Use a centralized permission catalog with scoped assignments. Candidate scopes:
- Platform
- Organization
- Property
- Resource-specific, only where justified

Candidate permission examples:
- organization.view / organization.manage
- property.view / property.manage
- reservation.view / reservation.create / reservation.update / reservation.cancel
- guest.view / guest.manage
- frontdesk.check_in / frontdesk.check_out
- folio.view / folio.post_charge / folio.adjust
- payment.collect / payment.refund
- restaurant.order.manage
- housekeeping.task.assign / housekeeping.task.complete
- automation.view / automation.edit / automation.publish
- reports.view
- staff.manage
- audit.view

These are examples, not an approved final permission catalog.

### 4.2 Authorization decision

For every protected operation, evaluate:
1. Is the user authenticated?
2. Is the account active and allowed to perform the operation?
3. Does the user belong to the requested organization?
4. Does the user have access to the requested property?
5. Does the user hold the required permission at the correct scope?
6. Is the target resource actually owned by that organization/property?
7. Are business-state and approval rules satisfied?

Deny by default. A valid session does not imply access to every tenant or property.

### 4.3 Server and database responsibilities

- Application server: enforce business permissions, validate input, and control operation sequencing.
- PostgreSQL: enforce row-level tenant/property boundaries and integrity constraints.
- Sensitive database functions: enforce their own authorization and invariants.
- UI: display permitted actions for usability only; never serve as the security boundary.

Do not create conflicting, independently maintained permission rules in multiple layers. Define one permission model and document how each layer evaluates it.

## 5. Multi-tenant isolation

Tenant isolation is a critical security boundary.

Requirements:
- Every tenant-owned record must have an explicit organization scope.
- Property-owned records must also have property scope where applicable.
- Enforce organization/property consistency with composite foreign keys or equivalent database constraints.
- Derive scope from trusted membership data, not from client assertions.
- Scope every server-side query and mutation.
- Enable and test RLS on every tenant-specific table in exposed schemas.
- Review views, RPCs, functions, storage policies, and Realtime subscriptions for tenant leakage.
- Do not permit cross-tenant references through IDs supplied by a client.
- Keep platform support access separate from ordinary tenant membership and audit all exceptional access.

Required negative tests:
- User from Organization A cannot read or mutate Organization B records.
- User assigned to Property A cannot access Property B records without permission.
- Changing an organization_id or property_id in a request cannot change the effective scope.
- Guessed or copied record identifiers do not grant access.
- Related records cannot be joined or referenced across tenant boundaries unless an explicitly approved feature permits it.

## 6. PostgreSQL and Supabase RLS

RLS is mandatory for tenant-specific tables in exposed schemas.

Policy requirements:
- Policies must reflect the actual organization/property membership and permission model.
- Use explicit target roles such as TO authenticated where appropriate; role targeting alone is not authorization.
- Use row ownership/membership predicates in USING and WITH CHECK.
- UPDATE policies must include correct row visibility and post-update checks; UPDATE also requires appropriate SELECT visibility.
- Avoid broad policies such as “all authenticated users can access all rows.”
- Avoid using user-editable metadata in policy predicates.
- Revoke unnecessary table and function privileges.
- Review grants and Data API exposure separately from RLS: RLS controls rows after access is granted, not whether a table is exposed.
- Enable RLS whenever access is granted to anon/authenticated roles for an exposed table.

### 6.1 Views and functions

- Review every view for whether it bypasses underlying RLS.
- Where supported and appropriate, use security-invoker views; otherwise restrict access or place the view in a non-exposed schema.
- Prefer SECURITY INVOKER functions.
- Do not use SECURITY DEFINER as a shortcut to fix permission errors.
- Any SECURITY DEFINER function requires a documented reason, minimal execution grants, a safe search_path, explicit identity/authorization checks, and security review.
- Do not expose privileged functions unintentionally through default PUBLIC execute privileges.
- Run available database security advisors and inspect findings before release.

Exact syntax and feature availability must be checked against the current PostgreSQL and Supabase versions selected for the project.

## 7. API and application security

All server endpoints must:
- Authenticate the caller.
- Authorize organization, property, resource, and action.
- Validate input with explicit schemas and size limits.
- Reject unexpected fields where practical.
- Use parameterized database access; never concatenate untrusted input into SQL.
- Return only fields needed by the caller.
- Avoid leaking stack traces, secrets, internal IDs, or sensitive existence information.
- Apply rate limits to sensitive and expensive operations.
- Use idempotency keys for operations that may be retried.
- Apply appropriate CORS, CSRF, content-type, and request-origin protections based on the actual request model.
- Use secure response headers and a reviewed Content Security Policy.
- Log security-relevant outcomes without logging credentials or unnecessary personal data.

Mutations involving inventory, payments, folios, and reservation state must use controlled server-side transaction boundaries. A frontend availability check is never the final booking integrity check.

## 8. Secrets and environment configuration

- Never expose Supabase service-role keys, secret keys, payment secrets, signing keys, or provider credentials in browser code.
- Any environment variable prefixed with NEXT_PUBLIC_ must be treated as public.
- Use publishable/appropriate public keys only in client contexts.
- Keep privileged secrets in protected server-side environment configuration or a suitable secret manager.
- Separate local, preview/staging, and production credentials.
- Rotate secrets after suspected exposure and during defined operational rotation events.
- Do not commit .env files or credentials to Git.
- Do not place secrets in workflow JSON, logs, URLs, client-visible errors, or database records readable by tenant users.
- Store secret references and access metadata where necessary, not plaintext secret values.
- Restrict who can view or change deployment environment variables.

## 9. Guest data, privacy, and file storage

Guest information may include contact details, identity documents, stay history, preferences, and other sensitive information.

Requirements:
- Collect only information required for defined hotel workflows and legal obligations.
- Define field-level access for identity documents and sensitive guest data.
- Keep sensitive storage buckets private by default.
- Use short-lived, narrowly scoped signed access where file sharing is required.
- Validate file type, size, and content; do not trust filename or browser MIME type alone.
- Restrict upload and download by tenant, property, resource, and permission.
- Avoid public object URLs for private guest or staff documents.
- Define retention, correction, export, and deletion procedures based on applicable law.
- Do not expose guest data in analytics or logs without a justified and protected use case.

The launch jurisdictions and applicable privacy/identity-document rules remain unresolved.

## 10. Financial and reservation security

### Reservations and inventory
- Enforce inventory invariants in PostgreSQL transactions.
- Prevent concurrent overbooking with locking/constraints or another reviewed database-level mechanism.
- Use idempotency for booking creation, modification, cancellation, and payment callbacks.
- Validate state transitions on the server.
- Record actor and timestamp for important reservation changes.
- Do not allow a client to assign a room or override inventory without the required permission.

### Folios and payments
- Treat financial entries as auditable records.
- Restrict posting, adjustment, refund, and void actions by explicit permissions.
- Use reversals/adjustments rather than silently modifying posted ledger history.
- Validate payment provider signatures and callback state.
- Do not trust a browser redirect as proof of payment.
- Reconcile provider transaction identifiers and amounts server-side.
- Require approval for configured high-risk actions.
- Never store prohibited card authentication data.

Financial controls must be finalized with the accounting and payment-provider requirements.

## 11. Automation security

Tenant automation is a constrained feature, not an unrestricted code execution platform.

Requirements:
- Workflows use a versioned, validated JSON/DSL schema.
- Only approved triggers, conditions, and actions are available.
- No arbitrary JavaScript, shell commands, direct SQL, or unrestricted HTTP requests from tenant workflows.
- Every action re-checks the workflow's tenant/property scope and required permissions at execution time.
- Workflow authors cannot grant themselves permissions by editing workflow data.
- Secrets are referenced through protected connections and never embedded in workflow definitions.
- Test mode must not perform real payments, send real messages, or mutate production records.
- Published versions are immutable; edits create a new version.
- Enforce execution time, step count, concurrency, retry, and tenant cost limits.
- Prevent recursion and runaway event loops.
- Use idempotency keys and at-least-once-safe event consumers.
- Keep execution logs scoped and redact sensitive payload fields.
- Automation failures must not block core booking or payment operations.

The V1 transactional event outbox is distinct from the V2 offline sync queue.

## 12. Webhooks and third-party integrations

- Verify provider signatures using the provider's current recommended method.
- Validate timestamp/replay windows where supported.
- Treat webhook delivery as at-least-once; deduplicate using stable provider event IDs or idempotency keys.
- Persist enough verified event metadata for investigation.
- Do not trust client-supplied provider status.
- Restrict outbound integration destinations to approved providers and endpoints.
- Protect against server-side request forgery if any server-side URL fetching is introduced.
- Apply timeouts, retries, and circuit-breaker behavior where appropriate.
- Scope each integration credential to the minimum permissions required.
- Provide tenant-level disconnect and credential rotation procedures.

## 13. Audit logging and monitoring

Record security-relevant events, including:
- Authentication successes/failures and session revocation.
- Membership, role, and permission changes.
- Organization/property configuration changes.
- Guest data access or export where required by policy.
- Reservation overrides and sensitive state changes.
- Folio adjustments, refunds, and payment exceptions.
- Automation creation, publication, disablement, and privileged execution.
- Integration connection and credential changes.
- Platform support access and break-glass events.

Audit records should include actor, action, target, scope, timestamp, outcome, and correlation/request ID where appropriate.

Do not store passwords, access tokens, secret values, full payment credentials, or unnecessary guest data in logs. Restrict audit-log modification and deletion. Define retention and alerting rules.

## 14. Deployment and supply-chain security

- Keep dependencies maintained and pin versions through the package manager lockfile.
- Review dependency and vulnerability alerts.
- Protect the main branch and require reviewed changes for security-sensitive code.
- Use separate deployment environments and least-privilege credentials.
- Protect CI/CD secrets and limit workflow token permissions.
- Review build output and client bundles for accidental secret exposure.
- Apply security headers and TLS through the hosting platform configuration.
- Keep production database access restricted.
- Review third-party packages, integrations, and licenses before adoption.
- Do not deploy unreviewed database migrations or security policy changes directly to production.

## 15. Backup, incident response, and recovery

Define and test:
- Database backup schedule and retention.
- Recovery Point Objective (RPO) and Recovery Time Objective (RTO).
- Restoration procedure and periodic restore exercises.
- Incident severity levels and escalation owners.
- Credential rotation and session revocation after compromise.
- Tenant notification and evidence-preservation procedures where required.
- Security incident log and post-incident review.

A backup is not proven until restoration has been tested.

## 16. Security testing and release gates

Required test categories:
- Authentication and session handling.
- Organization/property isolation and IDOR/BOLA attempts.
- RLS policy tests for SELECT, INSERT, UPDATE, and DELETE.
- View, function, and storage policy review.
- Permission matrix tests for each sensitive action.
- Input validation and API abuse/rate-limit tests.
- Reservation concurrency and inventory integrity tests.
- Payment webhook signature, replay, and idempotency tests.
- Automation sandbox, tenant-scope, loop, and resource-limit tests.
- Dependency and secret scanning.
- Backup restoration test.
- Manual security review of privileged functions and deployment configuration.

Release gates:
1. No known critical or high-severity unresolved security issue without documented risk acceptance.
2. RLS and tenant-isolation tests pass.
3. No privileged secret appears in client bundles or repository history.
4. Database migrations and permission changes are reviewed.
5. Incident, backup, and recovery owners are identified.
6. Security-sensitive workflows have audit coverage.

## 17. Open security decisions

1. Final authentication methods and MFA requirements.
2. Session lifetime, refresh, and revocation policy.
3. Final role and permission matrix.
4. Platform support access and break-glass process.
5. Guest identity-document handling and retention.
6. Launch-country privacy and data-residency requirements.
7. Payment provider and payment security requirements.
8. File types and storage requirements.
9. Rate limits and abuse thresholds.
10. Audit retention and alerting ownership.
11. RPO/RTO targets and incident response ownership.
12. Security testing tools and release thresholds.

## 18. Implementation sequence

1. Approve organization/property membership and permission model.
2. Map every sensitive operation to a permission.
3. Design RLS policies alongside the database ERD, not after feature development.
4. Define secure Next.js session and server authorization patterns using current official guidance.
5. Define storage access policies and secret handling.
6. Implement audit logging and security tests.
7. Review reservation, payment, and automation trust boundaries.
8. Test isolation with multiple synthetic organizations and roles.
9. Complete deployment, backup, and incident-response readiness review.

**Status:** Initial security baseline — not proof of implemented controls and not approved as a production security sign-off.
