# SANDESH HOSPITALITY — Conceptual ERD

**Status:** Initial conceptual diagram — for technical review  
**Version:** 0.1.0  
**Related document:** [DATABASE-DESIGN.md](./DATABASE-DESIGN.md)

> This ERD visualizes candidate entities and relationships from the current database-design baseline. It is not an approved schema, migration, or complete accounting/inventory model. Cardinalities marked here are proposed and must be validated against detailed workflows. Fields are intentionally limited to key identifiers and scope columns.

## 1. How to read this diagram

- `PK` means primary key; `FK` means foreign key.
- `organization_id` and `property_id` are tenant/property scope keys where applicable.
- A relationship line expresses a proposed logical relationship, not necessarily a finalized foreign-key implementation.
- Composite tenant/property foreign keys are expected where needed to prevent cross-scope references.
- Authentication identities are owned by Supabase Auth; `user_profiles` references the Auth user ID.

## 2. Core identity, property, reservation, and stay model

```mermaid
erDiagram
    AUTH_USERS ||--o| USER_PROFILES : "has profile"
    USER_PROFILES ||--o{ ORGANIZATION_MEMBERSHIPS : "belongs through"
    ORGANIZATIONS ||--o{ PROPERTIES : "owns"
    ORGANIZATIONS ||--o{ ORGANIZATION_MEMBERSHIPS : "has members"
    ORGANIZATION_MEMBERSHIPS ||--o{ MEMBERSHIP_ROLES : "receives"
    ROLES ||--o{ MEMBERSHIP_ROLES : "assigned as"
    ROLES ||--o{ ROLE_PERMISSIONS : "grants"
    PERMISSIONS ||--o{ ROLE_PERMISSIONS : "included in"
    ORGANIZATION_MEMBERSHIPS ||--o{ PROPERTY_MEMBERSHIPS : "may receive"
    PROPERTIES ||--o{ PROPERTY_MEMBERSHIPS : "grants access"

    PROPERTIES ||--o{ ROOM_TYPES : "defines"
    PROPERTIES ||--o{ ROOMS : "contains"
    ROOM_TYPES ||--o{ ROOMS : "categorizes"
    PROPERTIES ||--o{ RATE_PLANS : "offers"
    ROOM_TYPES ||--o{ INVENTORY_NIGHTLY : "may track"
    PROPERTIES ||--o{ INVENTORY_NIGHTLY : "scopes"

    ORGANIZATIONS ||--o{ GUESTS : "owns or scopes"
    GUESTS ||--o{ GUEST_CONTACTS : "has"
    GUESTS ||--o{ GUEST_PREFERENCES : "has"
    PROPERTIES ||--o{ RESERVATIONS : "receives"
    GUESTS o|--o{ RESERVATIONS : "primary guest"
    RESERVATIONS ||--|{ RESERVATION_ROOMS : "contains"
    ROOM_TYPES ||--o{ RESERVATION_ROOMS : "requested type"
    ROOMS o|--o{ RESERVATION_ROOMS : "optional allocation"
    RESERVATIONS ||--o{ RESERVATION_GUESTS : "lists"
    GUESTS ||--o{ RESERVATION_GUESTS : "is associated"
    RESERVATIONS ||--o{ RESERVATION_STATUS_HISTORY : "records"
    RESERVATIONS ||--o{ RESERVATION_HOLDS : "may create"

    PROPERTIES ||--o{ STAYS : "hosts"
    RESERVATIONS ||--o{ STAYS : "may produce"
    STAYS ||--|{ STAY_ROOMS : "assigns"
    ROOMS ||--o{ STAY_ROOMS : "is assigned"
    STAYS ||--o{ HOUSEKEEPING_TASKS : "may trigger"
    ROOMS ||--o{ HOUSEKEEPING_TASKS : "needs service"
    PROPERTIES ||--o{ HOUSEKEEPING_TASKS : "scopes"
```

### Important modeling notes

1. The guest scope is unresolved in the database baseline. The diagram currently shows `GUESTS` as organization-scoped; this must change if guest identity is property-scoped or shared across organizations.
2. A reservation may contain multiple reservation-room lines. Whether one reservation can produce multiple separate stays, and how group bookings work, needs explicit lifecycle rules.
3. Room-type inventory and physical-room allocation may be hybrid. Do not treat `INVENTORY_NIGHTLY` as the final inventory implementation.
4. A room allocation on a reservation line is optional in this conceptual model; assignment may instead occur during front-desk operations.
5. Property membership may be optional if organization roles grant all-property access. Confirm before schema design.

## 3. Folios and financial records

```mermaid
erDiagram
    PROPERTIES ||--o{ FOLIOS : "owns"
    RESERVATIONS o|--o{ FOLIOS : "may be linked"
    STAYS o|--o{ FOLIOS : "may be linked"
    GUESTS o|--o{ FOLIOS : "may be responsible for"
    FOLIOS ||--|{ FOLIO_ENTRIES : "contains"
    FOLIOS ||--o{ PAYMENT_ALLOCATIONS : "receives allocation"
    PAYMENTS ||--o{ PAYMENT_ALLOCATIONS : "allocated through"
    PROPERTIES ||--o{ PAYMENTS : "records"
    PAYMENTS ||--o{ REFUNDS : "may be refunded"
    PROPERTIES ||--o{ INVOICES : "issues"
    FOLIOS o|--o{ INVOICES : "may be invoiced"
    PROPERTIES ||--o{ CASHIER_SESSIONS : "opens"
    CASHIER_SESSIONS ||--o{ CASHIER_MOVEMENTS : "records"
    PROPERTIES ||--o{ TAX_RULES : "configures"
```

### Financial integrity notes

- `FOLIO_ENTRIES` is a ledger-like record set. Posted entries must not be silently overwritten or deleted; corrections use explicit reversals/adjustments.
- Payment allocation, folio posting, and refund state changes require transactional and idempotent operations.
- The exact accounting semantics, invoice numbering, tax treatment, and cashier reconciliation model are not yet finalized.
- This diagram does not define debit/credit accounting fields or a complete double-entry ledger.

## 4. Restaurant, kitchen, housekeeping, and maintenance

```mermaid
erDiagram
    PROPERTIES ||--o{ RESTAURANT_OUTLETS : "operates"
    RESTAURANT_OUTLETS ||--o{ DINING_TABLES : "contains"
    RESTAURANT_OUTLETS ||--o{ MENU_CATEGORIES : "groups"
    MENU_CATEGORIES ||--o{ MENU_ITEMS : "contains"
    PROPERTIES ||--o{ ORDERS : "records"
    RESTAURANT_OUTLETS ||--o{ ORDERS : "receives"
    DINING_TABLES o|--o{ ORDERS : "may serve"
    STAYS o|--o{ ORDERS : "may be associated"
    ORDERS ||--|{ ORDER_ITEMS : "contains"
    MENU_ITEMS ||--o{ ORDER_ITEMS : "ordered as"
    ORDERS ||--o{ KITCHEN_TICKETS : "creates"
    KITCHEN_TICKETS ||--|{ KITCHEN_TICKET_ITEMS : "contains"
    ORDER_ITEMS ||--o{ KITCHEN_TICKET_ITEMS : "prepared as"
    PROPERTIES ||--o{ ROOM_ORDER_TOKENS : "issues"
    STAYS o|--o{ ROOM_ORDER_TOKENS : "authorizes"
    ROOMS o|--o{ ROOM_ORDER_TOKENS : "identifies"

    PROPERTIES ||--o{ MAINTENANCE_REQUESTS : "tracks"
    ROOMS o|--o{ MAINTENANCE_REQUESTS : "affects"
    MAINTENANCE_REQUESTS ||--o{ MAINTENANCE_ASSIGNMENTS : "assigns"
    PROPERTIES ||--o{ HOUSEKEEPING_TASK_EVENTS : "records"
    HOUSEKEEPING_TASKS ||--o{ HOUSEKEEPING_TASK_EVENTS : "has history"
```

Restaurant order status, kitchen preparation status, payment status, and folio-posting status must remain separate state concepts. A room-order token must be validated server-side against its property and active stay; it is not a general guest-data credential.

## 5. Automation, events, notifications, and integrations

```mermaid
erDiagram
    PROPERTIES ||--o{ AUTOMATION_WORKFLOWS : "scopes"
    AUTOMATION_WORKFLOWS ||--|{ AUTOMATION_WORKFLOW_VERSIONS : "has versions"
    AUTOMATION_WORKFLOW_VERSIONS ||--o{ AUTOMATION_EXECUTIONS : "runs"
    AUTOMATION_EXECUTIONS ||--o{ AUTOMATION_EXECUTION_STEPS : "contains"
    AUTOMATION_EXECUTIONS ||--o{ AUTOMATION_IDEMPOTENCY_KEYS : "guards"
    PROPERTIES o|--o{ EVENT_OUTBOX : "scopes event"
    EVENT_OUTBOX ||--o{ EVENT_DEAD_LETTERS : "may dead-letter"
    PROPERTIES ||--o{ INTEGRATION_CONNECTIONS : "configures"
    INTEGRATION_CONNECTIONS ||--o{ WEBHOOK_EVENTS : "receives"
    PROPERTIES ||--o{ NOTIFICATION_DELIVERIES : "tracks"
    NOTIFICATION_TEMPLATES ||--o{ NOTIFICATION_DELIVERIES : "uses"
    INTEGRATION_CONNECTIONS ||--o{ EXTERNAL_ID_MAPPINGS : "maps identifiers"
```

Automation definitions are versioned constrained JSON/DSL. Secrets must remain outside workflow JSON. Outbox publication and business changes must be committed atomically where the event represents a database business change. Delivery and execution are at-least-once, so consumers must be idempotent.

## 6. Cross-cutting scope and integrity requirements

The final physical schema must implement these rules, even where the conceptual diagram does not show every scope column:

- Tenant-owned records carry `organization_id`.
- Property-owned records carry `organization_id` and `property_id` where appropriate.
- A child record must not reference a property, reservation, guest, room, folio, or other parent from another organization/property.
- Use composite foreign keys or equivalent database constraints to enforce scope consistency.
- Apply RLS to tenant-specific tables in exposed schemas.
- Validate authorization in server-side application services as well as through RLS.
- Do not rely on UUID unpredictability or client-supplied scope IDs for access control.
- Use `timestamptz` for instants and `date` for property business dates.
- Use exact numeric types for money, never floating-point types.
- Preserve audit history and financial records; use explicit reversals rather than destructive deletion.

## 7. Deliberately not finalized in this ERD

The current source design does not settle the following, so this diagram does not invent a final answer:

1. Exact role/membership cardinalities and whether roles are global, tenant-defined, or hybrid.
2. Guest identity scope and cross-property/cross-organization deduplication.
3. Whether room-type inventory, physical-room inventory, or a hybrid is authoritative.
4. Full reservation, hold, cancellation, no-show, and group-booking lifecycle.
5. Folio accounting semantics and a complete financial ledger.
6. Tax, invoice numbering, fiscal rules, and currency precision.
7. Night-audit and property business-day records.
8. Biometric attendance vendor and attendance schema.
9. OTA/channel-manager integration entities and synchronization model.
10. Tenant billing, subscription, and usage-metering entities.
11. Final event catalog and retention policies.

## 8. Review checklist before executable schema

- [ ] Confirm organization and property membership model.
- [ ] Confirm guest profile scope.
- [ ] Confirm reservation-to-stay cardinalities and room assignment timing.
- [ ] Approve inventory and concurrency model.
- [ ] Approve folio/payment ledger model with finance requirements.
- [ ] Define night audit and business-day behavior.
- [ ] Define exact tenant/property composite foreign keys.
- [ ] Define permission matrix and RLS policies.
- [ ] Review every relationship against required workflows.
- [ ] Convert the approved model into migrations and automated database tests.

**Implementation gate:** Do not generate production migrations from this conceptual ERD until the unresolved decisions are reviewed and approved.
