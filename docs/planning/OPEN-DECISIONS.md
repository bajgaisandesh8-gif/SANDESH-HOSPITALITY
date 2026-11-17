# Open Decisions and Assumptions Register

**Status:** Working register — unresolved items must not be silently treated as approved  
**Purpose:** Keep product, operational, legal, and technical unknowns visible before implementation.

## Decision states

- **Open:** requires an explicit decision.
- **Proposed:** a recommendation exists but is not approved.
- **Deferred:** intentionally outside V1.
- **Resolved:** decision approved and reflected in relevant documentation.

## 1. Confirmed V1 direction

| Topic | Current direction | State |
|---|---|---|
| Product shape | Multi-tenant hotel-management SaaS | Confirmed direction |
| Client | Web-first | Confirmed direction |
| Hosting model | Cloud-first; internet required for normal operation | Confirmed direction |
| Offline operation | No offline application/sync in V1; manual outage procedure only | Deferred to V2 |
| Desktop shell | Tauri deferred; no V1 desktop application | Deferred |
| Core priorities | Reservations, PMS foundation, front desk | Proposed priority |
| Automation | Templates plus constrained visual workflow builder | Confirmed product requirement; implementation design pending |
| Tenant workflows | Platform-owned workflow engine/runner | Proposed architecture |
| n8n | Limited to platform-owned integrations, subject to license review | Proposed boundary |
| Source of truth | GitHub repository | Confirmed workflow |
| Implementation | One feature at a time, with review and tests | Confirmed workflow |

## 2. Product and market decisions

| Question | Why it matters | State |
|---|---|---|
| Initial launch country and target customer segment? | Determines localization, tax, invoices, privacy, support, and onboarding | Open |
| Single-property hotels, chains, or both at launch? | Affects organization/property setup and pricing | Open |
| Supported languages and default locale? | UI, guest messages, templates, and support | Open |
| Supported currencies and exchange-rate policy? | Pricing, folios, reporting, and payment reconciliation | Open |
| Tax, service charge, fiscal invoice, and invoice-number rules? | Legal and accounting compliance | Open |
| Tenant subscription/billing model and plan limits? | SaaS entitlements, metering, and billing | Open |
| Data import/migration requirements? | Determines onboarding tools and validation | Open |
| Data retention, export, deletion, and privacy policy? | Guest data lifecycle and compliance | Open |

## 3. Hotel operations decisions

| Question | Why it matters | State |
|---|---|---|
| Guest identity/registration requirements by jurisdiction? | Check-in form and retention policy | Open |
| Reservation channels in V1: manual, website, OTA, channel manager? | Inventory synchronization and integration scope | Open |
| Cancellation, deposit, no-show, refund, and overbooking policies? | Reservation and financial state machines | Open |
| Room assignment timing and room-type inventory rules? | PMS/inventory model | Open |
| Night audit and business-date closing requirements? | Daily revenue, date rollover, and reports | Open |
| Cashier session model and reconciliation roles? | Cash controls and separation of duties | Open |
| Payment gateway(s), settlement, and manual fallback? | Payment flow and compliance scope | Open |
| Restaurant POS, kitchen, QR ordering scope in initial release? | Whether these are V1 or later modules | Open |
| Housekeeping statuses, inspections, and room-release rules? | Room readiness and front desk integration | Open |
| Biometric attendance vendor and lawful basis? | Hardware/API integration and sensitive data handling | Open |
| Door-lock/key-card vendors? | Hardware integration and check-in workflow | Open |
| Meaning and scope of “HIMS”? | Avoid building an undefined module | Open |

## 4. Architecture and operations decisions

| Question | Why it matters | State |
|---|---|---|
| Supabase region and data residency requirements? | Latency, legal compliance, and project configuration | Open |
| RPO/RTO and backup retention? | Recovery design and operational cost | Open |
| Monitoring, alerting, and support ownership? | Incident response and reliability | Open |
| Email/SMS/WhatsApp providers and consent rules? | Guest communications and deliverability | Open |
| File/document storage and retention? | Guest documents and operational attachments | Open |
| Exact permission roles and property-scoping rules? | RLS, API authorization, and staff UX | Proposed |
| Workflow builder trigger/action catalog and limits? | Prevent unsafe tenant-defined automation | Proposed |
| Tenant quotas and automation cost limits? | Shared infrastructure protection | Proposed |
| n8n licensing and commercial embedding review? | Avoid product licensing risk | Open |
| UUIDv7/library support, audit retention, soft-delete rules? | Schema and future compatibility | Proposed |

## 5. Explicit V1 exclusions / deferrals

Unless formally changed through a documented decision, V1 does not include:
- Full offline application operation or background offline synchronization.
- Tauri desktop client.
- Tenant-created arbitrary JavaScript execution, unrestricted HTTP calls, or direct database queries in workflows.
- Unrestricted shared n8n access for hotel tenants.
- Unspecified integrations or modules whose operational requirements are not defined.

## 6. Decision process

For each decision:
1. State the exact question and options.
2. Record operational, security, legal, cost, and delivery implications.
3. Name the decision owner.
4. Mark as Resolved, Deferred, or Open.
5. Update the affected requirements, architecture, database, API, and test documents.
6. Do not implement behavior based only on an unapproved suggestion.

## 7. Coding-entry blockers

The following must be resolved before their related implementation:
- Launch jurisdiction and basic localization/currency/tax approach before financial and invoice implementation.
- Reservation/channel scope and cancellation/deposit rules before reservation inventory/payment implementation.
- Night audit/business-date requirements before financial reporting.
- Guest identity and retention requirements before collecting identity documents.
- Payment provider and settlement behavior before live payment integration.
- Final permission model before broad staff access or tenant-facing APIs.
- RPO/RTO and backup configuration before production launch.

A coding prototype may begin with synthetic data and explicitly marked assumptions, but it must not turn unresolved assumptions into production behavior.
