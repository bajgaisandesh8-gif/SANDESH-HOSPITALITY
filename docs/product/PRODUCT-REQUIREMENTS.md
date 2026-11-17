# SANDESH HOSPITALITY
## Product Requirements Document (PRD)

**Product name:** SANDESH HOSPITALITY  
**Product type:** Multi-tenant Hotel Management & Automation SaaS  
**Document status:** Initial baseline — for review  
**Version:** 0.1.0  
**Platform strategy:** Web-first, cloud-first  
**Owner:** Product owner approval required

> This document records the current product direction. Items marked as open decisions are not approved requirements and must not be silently assumed during implementation.

---

## 1. Product overview

SANDESH HOSPITALITY is a commercial hotel management and automation platform intended to centralize hospitality operations: reservations, property management, front desk, guest services, restaurant operations, housekeeping, financial workflows, reporting, and automation.

The platform is planned as a multi-tenant SaaS product. It must support multiple hotel organizations and multiple properties within each organization, with strict data isolation and permission enforcement.

### Product vision

Build a reliable, secure, maintainable, and automation-focused hospitality platform that helps hospitality businesses coordinate daily operations through one system.

### Core principles

- Security and tenant isolation by design.
- Reliable reservation, inventory, and financial operations.
- Automation as a core platform capability.
- Modular architecture and maintainable code.
- Clear user permissions and accountability.
- Consistent and accessible user experience.
- Incremental delivery with documented acceptance criteria.

## 2. Target users and customers

Potential customer types:
- Independent hotels.
- Hotel groups and chains.
- Resorts and other hospitality businesses.

Potential user groups:
- Owners and organization administrators.
- Property managers.
- Reservation and front desk staff.
- Housekeeping teams.
- Restaurant and kitchen staff.
- Finance and accounting staff.
- Operations managers and auditors.

**Open decisions:** initial target countries, hotel segment, supported languages, and launch customer profile.

## 3. SaaS organization model

The platform is expected to support:
1. Multiple organizations (tenants).
2. Multiple properties under an organization.
3. Organization-level administration.
4. Property-level configuration.
5. Users assigned to one or more permitted organizations/properties.
6. Role-based and, where needed, property-scoped permissions.
7. Strict isolation between organizations.
8. Explicitly authorized cross-property access within an organization.

No user may access another tenant's private data without explicit authorization enforced by trusted server-side controls and database policies.

## 4. Platform strategy and scope

### V1: web-first and cloud-first

V1 is a web application. Normal operations require an active internet connection.

Proposed technology direction:
- Next.js and TypeScript for the web application.
- Supabase PostgreSQL and Auth, with Storage and Realtime only where justified.
- Protected server-side application boundaries and narrowly scoped backend functions.
- Vercel for hosting.
- GitHub for source control and review.

This is a proposed stack, not a completed implementation. Detailed decisions belong in the architecture document.

### V1 outage procedure

Full offline operation is out of scope. The product must provide an operational procedure for internet outages, including:
- Printable arrival, in-house, and departure lists.
- Manual folio and payment-recording procedures.
- A controlled process for entering and reconciling outstanding records after service returns.

The product must not imply that V1 supports offline transactions or automatic local synchronization.

### Future considerations (not V1 commitments)

- Tauri 2 desktop application.
- Offline-first workflows and local persistence.
- Controlled synchronization after connectivity returns.
- Additional hardware integrations.

## 5. Product modules

The following are candidate modules. Their final release order and acceptance criteria require separate planning.

### 5.1 Organization and property management
- Organization configuration.
- Property creation and configuration.
- Property-specific settings.
- User assignment and access management.
- Organization and property permission boundaries.

### 5.2 Property Management System (PMS)
- Room types and physical room records.
- Room status and occupancy visibility.
- Stay management.
- Room assignment.
- Check-in and check-out support.
- Operational dashboards.

### 5.3 Reservation management
- Create, view, modify, and cancel reservations.
- Availability checking.
- Guest and stay details.
- Room or room-type assignment.
- Reservation status and history.
- Booking confirmation.

Reservation and inventory operations must remain consistent under concurrent requests.

### 5.4 Front desk
- Arrival, in-house, and departure views.
- Guest lookup.
- Check-in and check-out.
- Room allocation and stay extension.
- Folio access.
- Front desk operational dashboard.

### 5.5 Guest management
- Guest profiles and contact details.
- Stay history.
- Guest preferences where appropriate.
- Guest communication history.

Sensitive guest information must be access-controlled and retained only according to an approved retention policy.

### 5.6 Restaurant POS
- Menu and category management.
- Order creation and status.
- Table management.
- Billing.
- Room-charge integration only through authorized, auditable workflows.

### 5.7 Kitchen Order Tickets (KOT)
- Kitchen order visibility.
- Preparation status.
- Pending and completed orders.
- Kitchen workflow notifications.

### 5.8 QR room ordering
- Room-associated QR ordering.
- Menu access.
- Order submission and tracking.
- Controlled room-charge integration.

### 5.9 Housekeeping
- Room cleaning status.
- Task assignment.
- Task completion and inspection.
- Housekeeping notifications.

### 5.10 Finance and folios
- Guest folios and charges.
- Payment records.
- Refund and adjustment workflows.
- Cashier operations.
- Financial reporting.

Financial history must be auditable. Posted financial entries must not be silently overwritten; corrections should use controlled adjustment or reversal workflows.

### 5.11 Virtual concierge
- Hotel and service information.
- Frequently asked questions.
- Guest assistance and approved service requests.

AI features must not bypass authorization, pricing, reservation, or financial business rules.

### 5.12 HIMS
HIMS is a provisional module name. Its full form, purpose, workflows, and boundaries are unresolved. Do not implement this module until the product owner confirms its meaning and requirements.

### 5.13 Biometric attendance integration
Potential attendance-device integration. Vendors, supported devices, data formats, consent, privacy, and integration method remain open decisions.

### 5.14 Email and notifications
Candidate notifications include booking confirmations, pre-arrival reminders, check-in updates, payment reminders, housekeeping tasks, restaurant orders, feedback requests, and operational alerts.

### 5.15 Reporting and analytics
Candidate reports include occupancy, reservations, revenue, payments, housekeeping, restaurant operations, and daily summaries. Definitions and access rules must be specified for each report.

### 5.16 Automation management
Automation is a core product capability. Both ready-made templates and a constrained visual workflow builder are required in the intended product direction. Detailed requirements belong in `docs/automation/AUTOMATION-ENGINE.md`.

## 6. Automation product requirements

### Ready-made template candidates
- Booking confirmation.
- Pre-arrival reminder.
- Check-in notification.
- Checkout-triggered housekeeping task.
- Restaurant order notification.
- Payment reminder.
- Maintenance task assignment.
- Daily operational report.
- Guest feedback notification.

### Visual workflow builder direction
The intended builder should support:
- Event-based and scheduled triggers.
- Conditions and AND/OR logic.
- Conditional branches and delays.
- A closed catalog of approved actions.
- Draft and published versions.
- Test mode that cannot perform real-world actions.
- Execution history and enable/disable controls.

### Automation safety and reliability
- No arbitrary JavaScript execution in tenant workflows.
- No unrestricted database queries or arbitrary outbound HTTP.
- Permission checks for every action.
- Organization and property scoping.
- Idempotent execution and duplicate-event handling.
- Retries, failure visibility, and dead-letter handling.
- Audit history and versioning.
- Approval for sensitive financial actions.
- Automation failure must not block core booking or payment operations.

## 7. Roles and permissions

Initial role candidates:
- Platform Super Administrator.
- Organization Owner.
- Organization Administrator.
- Property Manager.
- Reservation Manager.
- Front Desk Staff.
- Housekeeping Staff.
- Restaurant Manager.
- Restaurant Staff.
- Kitchen Staff.
- Finance Manager.
- Accountant.
- Read-Only Auditor.

This is not a final role matrix. Permissions must be defined by resource, action, organization, and property scope. Authorization must be enforced server-side and at the database boundary where applicable; hiding a UI control is not authorization.

## 8. Security and data protection requirements

The platform must be designed to include:
- Strict organization and property data isolation.
- PostgreSQL Row Level Security on exposed tenant-specific data.
- Server-side authentication and authorization checks.
- Secure session handling.
- Input validation and output encoding.
- Appropriate rate limits and security headers.
- Secure secret management.
- Audit logging for sensitive operations.
- Database constraints and atomic operations.
- Automated authorization and tenant-isolation tests.
- Backup and restoration procedures with tested recovery.

Supabase service-role credentials and other privileged secrets must never be exposed to browser code. Security-sensitive database functions and views require explicit review.

Privacy, data retention, deletion, and jurisdiction-specific compliance requirements remain to be defined.

## 9. Reservation, inventory, and financial integrity

These are critical business domains. Detailed design must address:
- Concurrent reservation attempts.
- Per-night room inventory consistency.
- Physical-room double-booking prevention.
- Reservation holds and expiry.
- Idempotent booking and payment operations.
- Atomic database transactions.
- Payment reconciliation and refund history.
- Auditable folio and ledger records.
- Date, time-zone, and stay-date semantics.
- Controlled invoice numbering and night-audit workflows.

Implementation must not begin with assumptions about these rules. They require reviewed domain and database specifications.

## 10. Non-functional requirements

### Reliability
Core reservation, check-in, and financial workflows should fail safely and preserve data integrity.

### Security
Tenant isolation, least privilege, and auditable authorization are mandatory.

### Performance
Common operational workflows should be responsive under agreed workload assumptions. Numeric targets require measurement and approval.

### Scalability
The architecture should support growth in organizations, properties, users, reservations, and automation executions.

### Maintainability
Use modular boundaries, typed contracts, reviewed migrations, automated tests, and current documentation.

### Observability
Provide appropriate application logs, error monitoring, audit trails, and operational health visibility without logging secrets or unnecessary sensitive guest data.

### Accessibility
The interface should support keyboard navigation, readable content, and accessible interaction patterns.

## 11. Initial delivery priorities

Suggested sequence, subject to detailed planning:

1. Project foundation and architecture.
2. Authentication and user identity.
3. Organization and property management.
4. Roles, permissions, and tenant isolation.
5. Reservation and inventory foundation.
6. PMS and front desk.
7. Guest management.
8. Finance and folios.
9. Restaurant, KOT, QR ordering, and housekeeping.
10. Notifications, reporting, and automation.
11. Virtual concierge and approved integrations.

This sequence is a planning proposal, not a promise that every listed module ships in V1.

## 12. Explicit V1 exclusions

- Full offline operation.
- Local database synchronization.
- Tauri desktop application.
- Unrestricted tenant-authored code execution.
- Unrestricted tenant-created HTTP or database integrations.

## 13. Open product decisions

These must be answered and approved before the affected features are designed or implemented:

1. Initial target countries and customer segment.
2. Supported currencies and multi-currency behavior.
3. Tax, fiscal, and invoice rules by country.
4. Payment gateway providers and payment methods.
5. OTA and channel-manager scope.
6. Guest identity and verification requirements.
7. Invoice numbering and legal document requirements.
8. Night-audit workflow and business-day boundaries.
9. Tenant subscription, plans, and billing model.
10. Biometric hardware vendors and privacy requirements.
11. HIMS meaning and scope.
12. Data migration requirements.
13. Backup retention, recovery point objective (RPO), and recovery time objective (RTO).
14. Supported languages and localization requirements.
15. Final V1 release scope and acceptance criteria.

Each decision should record its status, owner, rationale, and approval date.

## 14. Product development rules

- Do not implement a feature without documented requirements and acceptance criteria.
- Do not silently convert open questions into decisions.
- Do not weaken tenant isolation for implementation convenience.
- Keep business rules behind trusted server-side boundaries.
- Use reviewed database migrations; do not rewrite migrations already applied to shared environments.
- Preserve audit history for important business and financial operations.
- Test authorization and tenant isolation.
- Deliver one coherent module or vertical slice at a time.
- Review architecture and security before large changes.
- Keep documentation aligned with actual implementation.

## 15. Document status and review

This is an initial product baseline, not a final technical specification.

It records the current product direction, candidate modules, V1 boundaries, and unresolved decisions. Architecture, database, security, automation, and module-specific documents must refine it before substantial feature implementation.

**Required next review:** Product-owner review and architecture review.  
**Approval status:** Not yet approved.
