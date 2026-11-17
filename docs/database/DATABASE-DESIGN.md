# SANDESH HOSPITALITY
## Database Design

**Document status:** Initial conceptual baseline — for technical review  
**Version:** 0.1.0  
**Database direction:** Supabase PostgreSQL  
**Related documents:** docs/product/PRODUCT-REQUIREMENTS.md and docs/architecture/SYSTEM-ARCHITECTURE.md

> This document proposes logical entities, relationships, integrity requirements, and security constraints. It is not an approved executable schema or migration. Entity names and boundaries are candidates until reviewed against detailed workflows, reporting, fiscal requirements, and the final permission model.

## 1. Database design objectives

The database must support multiple organizations (tenants), multiple properties per organization, scoped access, reliable reservation and room-inventory operations, guest stays, auditable folios and financial transactions, restaurant orders, housekeeping, versioned automation, strict tenant isolation, and future extensibility.

PostgreSQL is the proposed system of record for core business data.

## 2. General modeling conventions

### Identifiers
- Use UUID primary keys.
- UUIDv7 is a candidate if supported reliably by the chosen PostgreSQL/runtime approach.
- Do not treat unguessable identifiers as authorization.
- Use explicit unique constraints for human-facing identifiers within their intended scope.

### Dates and timestamps
- Store instants as timestamptz.
- Store hotel business dates as date where they represent calendar dates rather than instants.
- Each property must have an explicit IANA time-zone identifier.
- Never assume the server's local time zone is the hotel's time zone.
- Define check-in/check-out times and business-day boundaries at property level.
- A half-open stay interval (check-in inclusive, check-out exclusive) is a candidate and must be confirmed.

### Audit and lifecycle fields
For applicable mutable records, consider created_at, updated_at, created_by, updated_by, and version for optimistic concurrency. Use deleted_at only where soft deletion is appropriate. Do not soft-delete financial ledger entries or audit history; use explicit reversals or adjustments.

### Tenant and property scope
- Tenant-owned records should include organization_id.
- Property-owned records should include organization_id and property_id where appropriate.
- Enforce that a referenced property belongs to the same organization as its child record.
- Prefer composite foreign keys or equivalent database constraints for cross-scope integrity.
- Never treat client-supplied organization_id or property_id as proof of access.

### Money and quantities
- Never use floating-point types for money.
- Select exact numeric precision and currency behavior after launch markets are confirmed.
- Store currency codes explicitly where multi-currency is supported.
- Define tax rounding and calculation precision.
- Document units and constraints for inventory quantities and rates.

## 3. Logical domain groups and candidate entities

These are conceptual candidates, not a final table list.

### 3.1 Identity, organizations, and access

| Candidate entity | Purpose | Scope |
|---|---|---|
| user_profiles | Application profile linked to an Auth user | User |
| organizations | Tenant/business account | Tenant |
| properties | Hotel or hospitality property | Tenant |
| organization_memberships | User membership in an organization | Tenant |
| property_memberships | Explicit property access where required | Tenant + property |
| roles | Role definitions, if configurable | Platform or tenant |
| permissions | Permission catalog | Platform |
| role_permissions | Role-to-permission mapping | Platform or tenant |
| membership_roles | Role assignment to a membership | Tenant |
| audit_logs | Sensitive administrative and business activity | Tenant/property |

Supabase Auth owns authentication identities. Application profile and membership records must reference the Auth user identifier through a controlled relationship. Do not duplicate passwords or credentials in application tables.

Open questions: Can users belong to multiple organizations? Are roles global, tenant-customizable, or hybrid? What audited break-glass process, if any, is permitted for platform support staff?

### 3.2 Property configuration and room inventory

| Candidate entity | Purpose | Scope |
|---|---|---|
| room_types | Sellable room categories | Property |
| rooms | Physical room records | Property |
| room_features | Feature catalog or room attributes | Property |
| room_type_features | Room type feature mapping | Property |
| room_status_history | Historical operational status changes | Property |
| rate_plans | Pricing plan definitions | Property |
| inventory_nightly | Per-night sellable inventory, if aggregate inventory is used | Property + date |

The final inventory model must distinguish sellable room-type inventory, physical room assignment, out-of-order rooms, reservation holds, confirmed reservations, and operational room status. Housekeeping status is not the same as booking availability.

### 3.3 Guests, reservations, and stays

| Candidate entity | Purpose | Scope |
|---|---|---|
| guests | Guest identity/profile | Tenant or property; TBD |
| guest_contacts | Contact methods | Guest |
| guest_preferences | Optional guest preferences | Guest |
| reservations | Booking-level record | Tenant + property |
| reservation_rooms | Room type/room allocation lines | Tenant + property |
| reservation_guests | Guest-to-reservation relationship | Tenant + property |
| reservation_status_history | Reservation lifecycle history | Tenant + property |
| reservation_holds | Temporary inventory holds | Tenant + property |
| stays | Actual checked-in stay lifecycle | Tenant + property |
| stay_rooms | Physical room assignments during a stay | Tenant + property |

Guest deduplication and identity rules require a privacy-aware design. Email or phone must not be assumed to be a globally unique guest identifier.

The reservation lifecycle must distinguish a booking from an actual in-house stay. Modifications, cancellations, no-shows, room moves, extensions, and partial occupancy need explicit state-transition rules.

### 3.4 Folios and financial records

| Candidate entity | Purpose | Scope |
|---|---|---|
| folios | Guest or reservation account/folio | Tenant + property |
| folio_entries | Charges, credits, adjustments, and ledger entries | Tenant + property |
| payments | Payment transaction records | Tenant + property |
| payment_allocations | Payment-to-folio allocation | Tenant + property |
| refunds | Refund lifecycle and provider references | Tenant + property |
| tax_rules | Applicable tax configuration | Jurisdiction/property |
| invoices | Issued invoice records | Tenant + property |
| cashier_sessions | Cashier opening/closing records | Property |
| cashier_movements | Cash movements and reconciliation | Property |

Financial requirements:
- Preserve an auditable history of posted entries.
- Correct posted entries through explicit reversal or adjustment records, not silent overwrites.
- Use exact numeric types for money.
- Make currency and rounding behavior explicit.
- Use idempotency keys for payment initiation and provider webhook processing.
- Store provider references and status transitions without storing prohibited payment credentials.
- Use transactions for posting and allocation invariants.
- Define invoice numbering and fiscal rules per launch jurisdiction before implementation.

The final ledger model and accounting semantics require finance-domain review.

### 3.5 Restaurant, KOT, and QR ordering

| Candidate entity | Purpose | Scope |
|---|---|---|
| restaurant_outlets | Restaurant/outlet configuration | Property |
| dining_tables | Table records | Outlet |
| menu_categories | Menu grouping | Outlet |
| menu_items | Sellable menu items | Outlet |
| orders | Restaurant order header | Property |
| order_items | Ordered item lines | Property |
| kitchen_tickets | Kitchen preparation ticket | Property |
| kitchen_ticket_items | Ticket item lines | Property |
| room_order_tokens | Controlled room-order association/token | Property |

Order status, kitchen preparation status, payment status, and folio posting status should be separate concepts. QR ordering must validate the room/order token, property, active stay, and allowed order actions on the server. A QR code must not grant unrestricted access to guest or folio information.

### 3.6 Housekeeping and maintenance

| Candidate entity | Purpose | Scope |
|---|---|---|
| housekeeping_tasks | Cleaning and inspection tasks | Property |
| housekeeping_task_events | Task lifecycle history | Property |
| maintenance_requests | Maintenance issue records | Property |
| maintenance_assignments | Assignment and ownership | Property |

Housekeeping tasks should link to relevant rooms and, where appropriate, stays or checkout events. Assignment and completion must be auditable.

### 3.7 Notifications and integrations

| Candidate entity | Purpose | Scope |
|---|---|---|
| notification_templates | Approved message templates | Platform or tenant |
| notification_deliveries | Delivery attempts and provider references | Tenant/property |
| integration_connections | Provider connection metadata | Tenant/property |
| webhook_events | Verified inbound event records | Integration/tenant |
| external_id_mappings | Internal-to-provider identifier mapping | Tenant/property |

Provider secrets must not be stored as plaintext in ordinary application tables or returned to clients. Use an approved secret-management approach and store only references/metadata in the database.

### 3.8 Automation

| Candidate entity | Purpose | Scope |
|---|---|---|
| automation_workflows | Workflow identity and metadata | Tenant/property |
| automation_workflow_versions | Immutable version snapshots | Tenant/property |
| automation_executions | Workflow execution records | Tenant/property |
| automation_execution_steps | Step-level execution state | Tenant/property |
| automation_idempotency_keys | Duplicate execution protection | Tenant/property |
| event_outbox | Transactional business event outbox | Tenant/property where applicable |
| event_dead_letters | Events requiring investigation | Tenant/property |

Workflow definitions must be constrained, versioned JSON/DSL. They must not contain secrets, arbitrary SQL, or unrestricted executable code.

## 4. Core relationships (conceptual)

    Organization
      ├── Properties
      │     ├── Rooms and Room Types
      │     ├── Rate Plans
      │     ├── Reservations
      │     │      ├── Reservation Rooms
      │     │      └── Reservation Guests
      │     ├── Stays
      │     │      └── Stay Rooms
      │     ├── Folios
      │     │      ├── Folio Entries
      │     │      └── Payment Allocations
      │     ├── Restaurant Orders
      │     │      ├── Order Items
      │     │      └── Kitchen Tickets
      │     ├── Housekeeping Tasks
      │     └── Automation Workflows
      └── Organization Memberships
             └── Role Assignments

    Auth User
      └── User Profile
             └── Organization Memberships

This diagram is simplified. Exact cardinalities, guest scope, shared catalogs, and cross-property workflows require further design.

## 5. Referential integrity and tenant boundaries

For tenant-owned records:
- Include organization_id as a scope key.
- Reference the owning organization with a foreign key.
- For property-owned records, use a composite relationship proving the property belongs to the same organization.
- When a child references another tenant-owned row, enforce matching organization and property scope with composite foreign keys or equivalent constraints.
- Never rely on client-provided scope identifiers as proof of authorization.

Consider a unique key on properties (organization_id, id) so child tables can reference both values through a composite foreign key. The eventual ERD and migration review must show the exact constraints.

## 6. Reservation and inventory integrity

This area requires a dedicated technical design before implementation.

Requirements:
- Prevent overlapping confirmed assignments to the same physical room for the same stay dates.
- Confirm half-open stay-date interval semantics: [check-in, check-out).
- Protect per-night room-type inventory against concurrent booking attempts.
- Make holds expire predictably and release inventory safely.
- Make booking confirmation idempotent.
- Ensure cancellation and modification update inventory atomically.
- Define lock ordering to reduce deadlocks.
- Use database constraints as the final integrity boundary, not only frontend availability checks.
- Use optimistic versioning for appropriate operational edits.

Potential tools include PostgreSQL transactions, row-level locks, exclusion constraints for physical-room overlap, and carefully scoped database functions. Select the exact approach after testing the final model.

## 7. Financial integrity

- Posted ledger entries should be immutable to ordinary application roles.
- Corrections should use explicit reversal or adjustment records.
- Payment creation and webhook handling must be idempotent.
- Payment allocations must not exceed the applicable available amount.
- Folio posting and order-to-folio charging must be atomic.
- Provider callbacks must be authenticated and validated.
- Financial actions must record actor, timestamp, property, currency, amount, and relevant references.
- Sensitive financial actions may require approval based on the permission matrix.

The exact accounting model and legal invoice behavior remain open decisions.

## 8. RLS and database authorization strategy

RLS is mandatory on tenant-specific tables in exposed schemas.

Policy design must use the actual membership and permission model. Do not apply a generic auth.uid() = user_id policy to every table.

Required principles:
- Policies must scope rows to the authenticated user's active organization/property membership and required permission.
- The authenticated database role alone is not sufficient authorization.
- UPDATE policies need correct SELECT visibility plus both USING and WITH CHECK rules.
- Do not use user-editable Auth metadata for permissions.
- Review views for RLS behavior; use security-invoker views where supported and appropriate.
- Prefer SECURITY INVOKER functions.
- Any SECURITY DEFINER function requires a documented reason, restricted execution grants, safe search path, and explicit authorization checks.
- Revoke unnecessary table/function privileges.
- Test policies with users from different organizations, properties, and roles.

RLS is defense in depth and does not replace server-side business authorization.

## 9. Indexing and query design

Index based on actual query patterns and measured workload.

Likely candidates:
- Organization and property scope keys.
- Reservation lookup by property, status, and stay dates.
- Guest lookup using approved normalized identifiers.
- Room and room-type lookup.
- Folio and payment references.
- Housekeeping task status and assignment.
- Outbox pending status and creation time.
- Automation execution status and workflow version.

Avoid adding indexes blindly. Consider write overhead, selectivity, query plans, and unique constraints. Use EXPLAIN/ANALYZE with representative non-production data before tuning.

## 10. Data retention, privacy, and deletion

Before launch, define:
- Guest personal-data retention periods.
- Legal retention of invoices and financial records.
- Guest profile correction/deletion workflows.
- Audit-log retention.
- Automation execution/event retention.
- Uploaded file retention.
- Backup retention and deletion limitations.
- Data export and tenant offboarding.

Do not cascade-delete financial history, audit evidence, or legally retained records merely because a guest or user profile is deleted.

Jurisdiction-specific privacy and fiscal requirements remain open.

## 11. Migrations and schema governance

- Use version-controlled database migrations.
- Never edit a migration already applied to a shared or production environment; create a new corrective migration.
- Review migrations for data loss, locks, backfills, and recovery implications.
- Apply migrations to a local/disposable environment first.
- Test schema and RLS behavior before staging and production.
- Use automated migration checks and database tests.
- Keep generated schema types synchronized with the approved schema.
- Do not use ad hoc production SQL as a substitute for migration history.

Migration naming and exact CLI workflow must follow the current installed Supabase CLI documentation.

## 12. Testing requirements

Database tests must cover:
- Foreign-key and check constraints.
- Cross-organization reference rejection.
- Cross-property reference rejection where not allowed.
- RLS SELECT, INSERT, UPDATE, and DELETE behavior.
- Unauthorized role/action combinations.
- Concurrent reservation attempts.
- Room overlap prevention.
- Hold expiry and inventory release.
- Idempotent booking/payment operations.
- Folio balance and allocation invariants.
- Financial reversal behavior.
- Event outbox atomicity and duplicate delivery.
- Automation tenant scoping.
- Migration upgrade and restore paths.

Use pgTAP or an equivalent PostgreSQL testing approach where appropriate, plus application-level integration tests.

## 13. Open database decisions

1. Final organization and property membership model.
2. Global versus tenant-customizable roles.
3. Guest profile scope and cross-property guest recognition.
4. Room-type inventory versus physical-room-only inventory, or a hybrid.
5. Reservation lifecycle and group booking requirements.
6. Stay and room-move modeling.
7. Folio and accounting ledger semantics.
8. Currency precision, exchange rates, and rounding.
9. Tax and invoice rules by launch country.
10. Night-audit and business-day model.
11. Payment provider and reconciliation design.
12. OTA/channel-manager external identifiers and sync model.
13. Data retention and privacy rules.
14. Automation event catalog and retention.
15. Required reporting and analytics workloads.

## 14. Approval and next steps

Before executable schema work:
1. Review this conceptual model against the PRD.
2. Resolve high-impact open decisions for the first release slice.
3. Produce an ERD with exact cardinalities and scope keys.
4. Define the initial permission matrix and RLS strategy.
5. Design the reservation/inventory transaction model.
6. Design the folio/payment ledger model.
7. Review migration and test strategy.
8. Only then create migrations in a local development database.

**Status:** Initial conceptual baseline — not approved for production schema implementation.
