# SANDESH HOSPITALITY — V1 Product and Architecture Baseline

**Status:** Consolidated planning baseline — internal product defaults proposed; external legal/provider gates remain open  
**Version:** 1.0.0  
**Applies to:** First usable V1 release  
**Authority:** Read with PRODUCT-REQUIREMENTS.md, SYSTEM-ARCHITECTURE.md, SECURITY-ARCHITECTURE.md, FOUNDATION-SCHEMA.md, PERMISSION-MATRIX.md, FINANCIAL-APPROVAL-SCHEMA.md, PAYMENT-FOLIO-LEDGER-DESIGN.md, API-DESIGN.md, and workflow documents.

> This document consolidates V1 decisions and implementation boundaries. It does not claim that the product is implemented, that legal/accounting requirements have been certified, or that external integrations are ready. Where a safe internal default is possible, it is stated below. Items requiring a hotel operator, qualified accountant/lawyer, payment provider, or infrastructure-plan confirmation remain explicit launch gates.

## 1. Product definition

SANDESH HOSPITALITY is a multi-tenant hotel-management SaaS. An organization may own multiple properties. Staff access is scoped to an organization and/or explicitly assigned properties. V1 is a responsive web application, cloud-first, and requires internet connectivity for normal operation.

### V1 target and positioning
- Initial launch market: Nepal; operating currency default: NPR.
- Initial customer profile: independent hotels and small-to-mid-sized hotel operators. Multi-property organizations are supported by the tenancy model, but enterprise chain-specific workflows are not a first-release requirement.
- Admin/staff interface: English-first. Store locale/time-zone/currency as explicit property configuration so localization can be added without schema redesign. Nepali UI and guest-message localization are a later delivery item unless required by the first pilot.
- V1 is not a public OTA/channel manager or a general-purpose ERP. It is the hotel's operational system for its own reservations and front-desk workflows.
- Subscription billing, public self-service signup, marketplace, and usage billing are deferred. Initial tenant onboarding is controlled/manual.

## 2. V1 release boundary

### V1 core — build in this order
1. Identity, authentication, organization/property setup, memberships, fixed roles, permission enforcement, audit foundation.
2. Property configuration: timezone, operating settings, room types, physical rooms, room status, basic rate plans.
3. Reservation core: staff-created reservations, availability, confirmation, modification, cancellation, no-show, reservation history.
4. Front desk: arrivals/in-house/departures, room assignment, check-in, room moves only after the atomic workflow is tested, checkout.
5. Minimal guest records and stay history, with data minimization.
6. Reservation folio and payment recording only after the finance gates in section 11 are met.
7. Basic housekeeping task handoff from checkout and room readiness/status.
8. Basic operational reports required to run the above workflows.
9. Automation engine foundation and a small approved set of templates; expose the visual builder only after the runner, permissions, test mode, and audit controls pass their gates.

### Not in the first usable release
- Restaurant POS, KOT, QR room ordering, virtual concierge, HIMS, biometric attendance, door-lock integrations.
- OTA/channel-manager sync and public direct-booking engine.
- Live online payment gateway until a provider and settlement contract are selected and tested.
- Tenant-authored arbitrary code, unrestricted HTTP, arbitrary SQL, tenant-managed workers, or unrestricted n8n.
- Full offline mode, offline sync, Tauri desktop app, native mobile apps.
- SaaS subscription billing, self-service tenant signup, tenant marketplace.
- Multi-currency transactions, multi-folio per reservation, guest/company/house master folios.

These modules remain product-roadmap candidates, not V1 commitments. HIMS is undefined and must not be implemented until its meaning and requirements are supplied.

## 3. Confirmed foundational decisions

The following decisions are already reflected in the project design and must remain consistent:
- Multi-tenant organization/property model with strict cross-tenant isolation.
- V1 web-first/cloud-first; no offline application behavior or desktop shell.
- GitHub is the reviewed source of truth; focused branches and PRs; no merge/deploy without explicit approval.
- PostgreSQL is the system of record; migrations are the only schema-change mechanism.
- Hybrid membership: organization-scoped Owner/Admin roles and explicitly property-scoped operational roles.
- Fixed platform-managed V1 role and permission catalog; no tenant-created roles in V1.
- No implicit cross-property access for property staff and no implicit cross-tenant access.
- Organization Admin cannot manage Owners/Admins, transfer ownership/billing, alter the role catalog, or gain sensitive finance/guest-data powers by virtue of the Admin role.
- One primary folio per reservation; check-in reuses that folio; no guest-level shared folio.
- Split tender is supported: multiple separate payment/tender records may allocate to the same reservation folio. A single tender cannot be split across multiple folios.
- Financial approval thresholds: NPR 25,000 per property for refunds and NPR 5,000 for folio adjustments; only an active Organization Owner changes policy and approves; no self-approval; mandatory exceptions and anti-splitting controls remain as specified in the financial approval schema.
- Automation is supplemental and must never block core reservation, front-desk, or payment workflows.

## 4. Domain defaults for implementation

These are the conservative V1 defaults to use unless an external gate or explicit later decision overrides them.

### 4.1 Guest records
- Guest profile scope is organization-level, allowing recognition across properties within the same hotel organization only.
- Do not merge guest profiles across organizations.
- Do not use email or phone as a globally unique identity key. Use explicit, reviewable duplicate suggestions rather than destructive automatic merges.
- Store only the minimum contact and stay details needed for booking and service.
- Identity-document capture/storage is excluded until Nepal-specific legal requirements, access rules, retention, and secure-storage design are approved.
- Guest preferences are optional, purpose-limited, and not a prerequisite for V1.

### 4.2 Inventory and reservations
- Use a hybrid inventory model: room-type availability is tracked per night; physical-room assignments are separately protected against overlapping stays.
- Stay dates use property-local calendar dates and half-open intervals: check-in inclusive, check-out exclusive.
- Every property has an IANA timezone. Never use server-local timezone for hotel dates.
- V1 booking channel is staff-created/manual booking. Public booking engine, OTA and channel manager are deferred.
- Availability is advisory; final inventory validation and reservation write occur atomically in PostgreSQL.
- Do not introduce temporary inventory holds in the first staff-only booking flow. If a future online checkout needs holds, add a separately specified expiring hold workflow.
- One reservation represents one primary room booking and produces at most one active stay in V1. Multi-room/group reservations and split stays are deferred.
- Reservation and stay lifecycles are separate. Reservation states: draft, pending, confirmed, cancelled, no_show, completed. Check-in/check-out state belongs to the stay; do not duplicate mutable check-in state across both records.
- Walk-ins use the same reservation/inventory transaction path as ordinary staff-created bookings.
- Cancellation/no-show penalties and deposit rules are property-configurable only after finance/accounting policy is approved; do not hard-code a universal penalty.
- Room moves/extensions must be explicit atomic commands with inventory validation and audit history. If their full invariants are not ready, defer the UI action rather than implement a partial update.

### 4.3 Folio and payment
- A reservation folio is created with the reservation; the corresponding stay reuses it.
- A reservation folio may accept an advance payment/deposit before check-in; the deposit is allocated to that same reservation folio, not held in an unassigned guest wallet.
- Each payment record represents one tender/payment method. Split tender means multiple payment rows allocated independently to the same folio.
- V1 transaction currency is NPR only. Reject non-NPR payment/folio operations; do not silently convert.
- Cash and bank-transfer recording may be implemented only with explicit manual verification evidence and actor attribution. A browser confirmation is not proof of settlement.
- Live card/wallet/QR/online gateway integration is deferred until a provider, merchant onboarding, callback verification, settlement, refund, and reconciliation process are selected.
- A pending/failed tender contributes zero to the paid folio amount.
- Refunds and folio adjustments follow the existing owner-approval schema. No automation workflow may directly mutate payment or ledger state.
- Cashier sessions, shift handover, cash drawer over/short, night audit, and closed-period reopening are not silently assumed; they require the workflow decision gate in section 11 before finance launch.

### 4.4 Room and front-desk operations
- Room operational status is separate from sellable inventory. Housekeeping status must not independently create or destroy booking inventory.
- Check-in requires a valid confirmed reservation, authorized actor, verified guest/occupancy inputs, and an assignable room meeting the property's readiness policy.
- Checkout closes the stay through an idempotent command and creates a housekeeping task/outbox event atomically.
- A room is not available for assignment merely because a stay has ended; it must pass the configured cleaning/inspection readiness process.
- Emergency/manual outage operations are paper/printable procedures, not offline writes into the application.

### 4.5 Restaurant and ancillary services
- Restaurant POS, KOT, QR ordering, room-charge posting, concierge, HIMS and attendance integrations are outside the first usable V1 release.
- When restaurant ordering is later introduced, order, kitchen, payment, and folio-posting states remain separate.
- Do not post restaurant charges to a room folio until the order-to-folio event, authorization, dispute handling, and void/refund behavior are approved.

## 5. Identity, roles, and authorization baseline

- Supabase Auth is the proposed identity provider; application authorization comes only from server-validated membership and role/permission records.
- Never use user-editable Auth metadata as an authorization source.
- The fixed V1 role catalog is defined in FOUNDATION-SCHEMA.md and PERMISSION-MATRIX.md.
- Organization Owner has organization-wide authority subject to explicit controls. Organization Admin has only the narrower capabilities defined in the permission matrix.
- Operational users require explicit property membership. No implicit access to sibling properties.
- A user may belong to multiple organizations; active context selection must be revalidated on every protected request.
- Platform support has no default tenant-data access. Any future support access must be time-limited, explicitly approved, narrowly scoped, and audited.
- Email/password plus verified email is the initial authentication baseline. MFA is strongly recommended for Owners/Admins and required before production finance administration if the selected Auth configuration supports it. Final session expiry/revocation and invitation delivery settings must be verified before launch.
- Invite-only onboarding for initial pilots; no public signup.
- Permission checks apply to reads, exports, downloads, background jobs, and writes—not only UI buttons.

## 6. Architecture and technology baseline

- Web application: Next.js App Router + TypeScript.
- Database and identity: Supabase PostgreSQL + Auth; Storage only for approved private files; Realtime only for workflows where latency materially helps.
- Hosting: Vercel for the Next.js application, subject to deployment-region, data-processing, and cost review.
- Source control and review: GitHub.
- API boundary: Next.js Route Handlers and server-side domain services. Supabase Edge Functions only for narrowly justified provider/webhook workloads; avoid duplicating domain authorization across runtimes.
- Database: PostgreSQL constraints, transactions, RLS, composite tenant/property foreign keys, and narrowly scoped RPCs for high-integrity operations.
- Identifiers: UUIDv7 only if supported cleanly by the chosen PostgreSQL/runtime libraries; otherwise UUIDv4 is acceptable. Never let identifier choice weaken authorization.
- Money: exact numeric types; no floating point.
- Frontend visual workflow builder: React Flow is a candidate, not a locked dependency. Select after accessibility, licensing, graph-validation and bundle review.
- Tenant automation runtime: platform-owned constrained engine. n8n may be used only for platform-owned fixed integrations after commercial license review.
- Avoid microservices in V1. Start with a modular monolith and separately deployed worker only when operationally justified.
- Do not add a second backend framework merely because an earlier idea mentioned Flask; the current Next.js server boundary is the default unless a measured requirement justifies a separate service.

## 7. Automation V1 baseline

- Start with event outbox, versioned event envelope, idempotent consumer, bounded retries/backoff, dead-letter review, and audit.
- Build and prove the runner before building a general visual editor.
- First templates: reservation confirmation (internal/in-app until messaging provider approved), pre-arrival task/reminder, checkout housekeeping task, daily operational summary.
- Initial action catalog excludes payment capture, refund, folio adjustment, inventory override, arbitrary SQL, arbitrary HTTP and code execution.
- Test mode uses synthetic/sanitized events and stubs all external effects. It must never mutate production data or send real messages.
- Published workflow versions are immutable. In-flight executions remain pinned to their original version; disabling a workflow prevents new executions and follows an explicit cancellation policy for delayed work.
- At-least-once delivery is expected; exactly-once external side effects are not promised.
- Apply graph size/depth limits, execution time limits, tenant concurrency/usage caps, loop protection, and payload minimization before broad rollout.
- Visual builder release is gated on action permissions, schema validation, test-mode isolation, execution history, and accessibility review.

## 8. Data lifecycle, audit, and operational resilience

- Use created_at/updated_at/actor/version fields where appropriate; soft-delete only mutable operational records where restoration/history is useful.
- Financial ledger and audit history are append-only; corrections use linked reversals/adjustments.
- Audit sensitive membership, role, guest-data access/export, reservation override, room assignment, payment, refund, folio, automation, integration, and support actions.
- Do not log passwords, tokens, provider secrets, full card data, identity-document contents, or unnecessary guest personal data.
- Retention periods for guest data, audit evidence, financial records, files, events, and logs must be defined against applicable Nepal law and customer contracts before production.
- Local, CI, staging, and production environments must be isolated. Use synthetic data in local/CI/staging.
- The existing Supabase project is a development/planning environment, not automatically a production environment. Its Mumbai region must be reviewed for customer data residency and contractual requirements before real guest data is stored.
- Printable arrivals/in-house/departures and a manual folio/payment log are required outage aids. After connectivity returns, authorized staff reconcile records with audit attribution. This is not offline sync.
- Backups must be restored in a test before production. RPO/RTO and retention targets must be selected based on the actual Supabase/Vercel plans, costs, and customer expectations.

## 9. API and integration conventions

- Candidate application API prefix: /api/v1. Keep this internal until the response/error envelope and compatibility policy are finalized.
- Use explicit request/response DTOs; never expose raw database rows.
- Validate all input server-side; reject over-posted ownership, scope, status, price, permission, and payment fields.
- Use stable error codes, request IDs, bounded cursor pagination, and idempotency for commands with business side effects.
- Provider webhooks are separate from user APIs; verify signature, timestamp/replay controls, merchant/account mapping, amount, currency, and expected state transition.
- No external partner API, public tenant API key, outgoing tenant webhook, or arbitrary outbound URL feature in the first usable release.
- Never store raw card numbers or card security codes.

## 10. Quality and release gates

A feature is not complete merely because it renders or builds. Required evidence includes:
- Unit tests for deterministic domain rules and state transitions.
- Database tests for constraints, RLS, grants, composite scope integrity, and transaction invariants.
- API tests for auth, authorization, validation, idempotency, concurrency, safe errors, and rate limits.
- End-to-end tests for booking, check-in, checkout, and unauthorized cross-tenant/property access.
- Security review of functions, views, storage, webhooks, secrets, CSP, and deployment configuration.
- Migration tests from a clean database and supported previous schema.
- CI lint, typecheck, tests, build, dependency/secret scanning.
- Staging smoke tests with synthetic data.
- Explicit release approval before production.

Critical release blockers: cross-tenant leakage; booking oversell/double assignment; duplicate financial posting; unverified payment success; missing audit on sensitive mutations; secrets exposed to browser/logs; failed backup restore; unresolved critical/high security findings.

## 11. External validation gates — cannot be responsibly guessed

These are not coding questions to repeatedly ask during planning. They are hard gates for the affected production features and must be closed with the relevant external party or evidence.

| Gate | Required evidence | Blocks |
|---|---|---|
| Nepal tax/accounting | Qualified Nepal accountant confirms PAN/VAT fields, invoice numbering, tax/service-charge calculation, rounding, credit notes, fiscal retention and correction workflow | Legally compliant invoices, tax posting, finance launch |
| Hotel operating policy | Pilot hotel confirms check-in/out times, business-day/night-audit process, cancellation/no-show/deposit rules, cashier shifts, cash reconciliation and closed-period handling | Production reservation policy and finance close |
| Payment provider | Selected provider confirms merchant onboarding, supported methods/currencies, signed callbacks, idempotency, settlement reports, refunds, disputes and sandbox tests | Live online payment integration |
| Guest privacy | Qualified local review confirms lawful collection, identity-document need, retention, access, export/deletion and incident handling | Identity-document collection and production privacy sign-off |
| Data residency and vendors | Confirm Supabase/Vercel region, subprocessors, contract terms, backup plan, data processing and cross-border transfer acceptability | Real customer/guest data in production |
| Messaging providers | Select email/SMS/WhatsApp vendor, sender identity, consent, delivery status and pricing | External guest messaging |
| Backup/recovery | Select RPO/RTO, retention, restore owner and prove restore on selected paid/free service plans | Production launch |
| Biometric/door locks/OTA | Vendor APIs, commercial terms, lawful basis, security model and operational acceptance | Those integrations (deferred) |
| Tenant billing | Pricing, taxes, payment collection, plan limits, support model and subscription terms | SaaS self-service billing (deferred) |

No document or AI recommendation can substitute for these external confirmations.

## 12. Implementation sequence and entry criteria

### Phase A — repository and engineering hygiene
- Resolve repository merge-conflict markers.
- Verify a clean clone, package scripts, lint/typecheck/build, and CI.
- Preserve Next.js-generated agent instructions and add project instructions.
- Keep environment examples secret-free.

### Phase B — identity and tenant foundation
- Approve the fixed role/permission seed and hybrid membership schema.
- Define invitation acceptance, account recovery, session revocation, Owner recovery/transfer.
- Implement RLS and cross-tenant/property negative tests before business tables are exposed.
- Add audit foundation and platform support boundary.

### Phase C — property and inventory
- Property timezone/configuration; room types, rooms, room status, rates.
- Approve ERD cardinalities and room-type/physical inventory transaction model.
- Test concurrent inventory mutations and room overlap.

### Phase D — reservation and front desk
- Staff booking, lifecycle, arrivals/in-house/departures.
- Atomic check-in, room assignment, checkout and housekeeping handoff.
- Print/manual outage procedures.

### Phase E — finance
- Close Nepal accounting and hotel operating policy gates.
- Reconcile parent payment/folio schema with financial approval schema.
- Implement reservation folio, advance deposit on same folio, split tender, manual payment verification, refunds/adjustments and reconciliation.
- Live provider integration only after provider gate.

### Phase F — automation
- Outbox, runner, templates, test mode, execution history.
- Visual builder after safe runtime is proven.
- Never let automation failures block core transactions.

### Phase G — later modules and production launch
- Restaurant/POS, QR ordering, concierge, HIMS, biometrics, OTA/channel manager, public booking, billing and offline mode each require separate scope and approval.
- Production launch only after external gates, restore drill, security review, staging acceptance and release approval.

## 13. Coding entry checklist

A developer may begin the first non-production coding slice only when:
- [ ] README and AGENTS.md are free of merge-conflict markers.
- [ ] GitHub main and local main are clean and synchronized.
- [ ] Phase 0 lint/typecheck/test/build and CI baseline is verified.
- [ ] First slice is limited to identity + organization/property context, not hotel financial behavior.
- [ ] Schema, permission seed, RLS policies, and tests are reviewed together.
- [ ] A disposable/local database environment is available; no production Supabase migration is used.
- [ ] User explicitly confirms the coding start.

The first recommended vertical slice is: invite-only authentication → create/select organization → create/select property → membership/role enforcement → tenant-isolation tests. Do not begin reservations or finance before this foundation passes.

## 14. Current implementation status

- The repository contains a Next.js scaffold and planning documents.
- Supabase project provisioning exists, but the database has no approved application schema/migrations.
- The documents in this repository are design artifacts, not proof of implemented product behavior.
- No hotel-management feature, payment integration, workflow engine, or production deployment is claimed by this baseline.

**Planning outcome:** Internal V1 defaults are consolidated. The external validation gates above remain open by design. They must be closed before the affected production feature or launch, not papered over with assumptions.
