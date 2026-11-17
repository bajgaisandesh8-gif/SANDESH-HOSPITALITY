# Open Decisions and Assumptions Register

**Status:** Consolidated V1 register  
**Primary reference:** [V1 Product and Architecture Baseline](V1-PRODUCT-AND-ARCHITECTURE-BASELINE.md)  
**Rule:** Resolved defaults are implementation guidance. External validation gates are not closed by documentation alone.

## Decision states

- **Resolved:** product/architecture direction is established and reflected in the baseline.
- **Default:** a conservative V1 default is selected for implementation planning; it must be surfaced if the first pilot requires a change.
- **Deferred:** intentionally excluded from the first usable release.
- **External gate:** requires evidence or confirmation from a hotel operator, qualified professional, provider, or infrastructure plan before the affected production feature.

## 1. Resolved V1 direction

| Topic | V1 decision |
|---|---|
| Product | Multi-tenant hotel-management SaaS |
| Initial market | Nepal-first |
| Initial customer profile | Independent and small-to-mid-sized hotels; multi-property organizations supported |
| Client/hosting | Web-first, cloud-first; internet required for normal operations |
| Offline / desktop | Deferred; manual printable outage procedures only |
| Initial priority | Identity/tenancy, property/rooms, reservations, front desk |
| Currency | NPR-only for initial financial operations; reject non-NPR |
| Membership | Hybrid organization/property membership |
| Roles | Fixed platform-managed role and permission catalog; no tenant-custom roles in V1 |
| Guest scope | Organization-scoped profiles; no cross-organization merge |
| Inventory | Hybrid: per-night room-type inventory plus physical-room overlap protection |
| Reservation | One primary room booking and at most one active stay per reservation in V1; group/multi-room bookings deferred |
| Dates | Property-local calendar dates; half-open [check-in, check-out) interval |
| Booking channel | Staff-created/manual bookings; public booking and OTA sync deferred |
| Folio | One primary folio per reservation; stay reuses it |
| Split tender | Multiple separately recorded tenders may allocate to one folio; one tender cannot span multiple folios |
| Advance payments | Reservation-linked deposits may be posted to that reservation's folio before check-in |
| Automation | Platform-owned constrained engine; templates first, visual builder only after safe runner |
| Integrations | No live external payment/messaging/OTA/biometric/door-lock integration until provider and legal gates close |
| Source of truth | GitHub; focused branches/PRs; no merge/deploy without explicit approval |
| Architecture | Modular monolith first; avoid premature microservices |

## 2. Financial policy decisions already recorded

| Topic | Decision |
|---|---|
| Refund approval threshold | NPR 25,000 per property |
| Folio adjustment approval threshold | NPR 5,000 per property |
| Threshold changes | Active Organization Owner only |
| Approver | Active Organization Owner only |
| Self-approval | Prohibited; another active Owner must approve |
| Mandatory approval exceptions | Cash refund, changed refund destination/method, trusted risk flag, closed-period adjustment, reversal/correction of posted financial transaction |
| Anti-splitting | Aggregate refunds against original payment; aggregate folio adjustments over rolling 24 hours and linked correction lifecycle; reserve pending amounts |
| Currency | NPR-only initially; exact numeric amounts; no silent FX |

The full policy and execution model remain in FINANCIAL-APPROVAL-SCHEMA.md and PAYMENT-FOLIO-LEDGER-DESIGN.md. These are design proposals, not implemented controls.

## 3. Conservative V1 defaults

| Topic | Default |
|---|---|
| UI language | English-first; locale-ready schema; Nepali localization can follow |
| Authentication | Invite-only, verified email/password baseline; MFA required for production finance administration if supported by selected configuration |
| Guest data | Minimal contact/stay data; no identity-document storage until legal/access/retention review |
| Duplicate guests | Suggest possible duplicates; no destructive automatic merge |
| Reservation holds | No temporary hold in staff-only first flow; introduce durable expiring holds only with a public/online booking use case |
| Cancellation/no-show | Explicit property policy; no universal hard-coded penalty |
| Cash/bank payments | Manual verification and actor/evidence record only; no client-side success trust |
| Payment gateway | None selected; live gateway deferred |
| Night audit | Property-local business date required; exact cutoff/close workflow must be confirmed with pilot hotel before finance reporting |
| Restaurant | POS/KOT/QR and room-charge posting deferred |
| Automation actions | No payment/refund/folio mutation, inventory override, arbitrary SQL/HTTP, or code execution |
| Tenant onboarding | Controlled/manual initial pilot; no public self-service signup |
| Tenant billing | Deferred |
| Support access | No default tenant data access; future break-glass must be time-limited, approved, scoped, audited |
| Deployment | Separate local/CI/staging/production environments; current Supabase project is not assumed production |
| Backups | Restore drill required; RPO/RTO and retention set after plan/cost review |

## 4. Deferred product areas

- Full offline application and synchronization; Tauri desktop; native mobile apps.
- Public direct-booking engine, OTA/channel manager, external partner API, outgoing tenant webhooks.
- Restaurant POS, kitchen tickets, QR room ordering, concierge.
- HIMS (undefined; do not implement until its meaning is supplied).
- Biometric attendance and door-lock/key-card integrations.
- Multi-currency, guest/company/house master folios, multiple folios per reservation.
- Tenant-custom roles, arbitrary workflow code, unrestricted n8n, workflow marketplace.
- Subscription billing, self-service signup, usage billing.

## 5. External validation gates — still open

These gates are intentionally not guessed or marked resolved.

| Gate | Evidence required | Blocks |
|---|---|---|
| Nepal tax/accounting | Qualified Nepal accountant confirms PAN/VAT fields, tax/service charge, invoice numbering, rounding, credit notes, fiscal retention and correction flow | Legal invoices, tax posting, production finance |
| Hotel operating policy | Pilot hotel confirms check-in/out times, business-day/night audit, cancellation/no-show/deposit policy, cashier sessions, shift handover, cash reconciliation and closed-period corrections | Production reservation policy and finance close |
| Payment provider | Provider selected; merchant onboarding, methods/currencies, signed callbacks, idempotency, settlement reports, refunds/disputes and sandbox behavior verified | Live online payments |
| Guest privacy | Qualified local review of collection purpose, identity documents, retention, access, export/deletion and incident response | Identity-document collection and privacy sign-off |
| Data residency/vendors | Supabase/Vercel region, subprocessors, contracts, backups and cross-border transfer acceptability confirmed | Real customer/guest data in production |
| Messaging | Email/SMS/WhatsApp provider, sender identity, consent, delivery state and cost confirmed | External guest messaging |
| Backup/recovery | RPO/RTO, retention, restore owner and successful restore drill on chosen service plan | Production launch |
| Integrations | Vendor API, commercial terms, lawful basis, security and operational acceptance | Biometric, door-lock, OTA and similar integrations |
| SaaS billing | Pricing, tax, plans, limits, collection and support terms | Self-service SaaS billing |

## 6. Coding-entry blockers

The first non-production coding slice is limited to invite-only authentication, organization/property context, fixed roles/permissions, audit foundation, and tenant-isolation tests.

Before that slice:
- Repository merge-conflict markers must be absent.
- Local and GitHub main must be clean and synchronized.
- Phase 0 lint/typecheck/test/build and CI baseline must be verified.
- The identity/membership schema, permissions, RLS policies, and negative tests must be reviewed together.
- A disposable local/test database must be used; no production Supabase migration.
- The user must explicitly confirm coding start.

Reservation/inventory implementation additionally requires the ERD/cardinality and concurrency contract. Finance/invoice/payment implementation additionally requires the external gates above.

## 7. Change control

If a pilot customer or external professional requires a different rule:
1. Record the exact change and its business/security/legal impact.
2. Update this register and the V1 baseline.
3. Reconcile affected product, workflow, API, database, permission, and test documents.
4. Add or update acceptance tests.
5. Only then implement the change.

Do not silently convert a default into a permanent policy or an external gate into an assumed fact.
