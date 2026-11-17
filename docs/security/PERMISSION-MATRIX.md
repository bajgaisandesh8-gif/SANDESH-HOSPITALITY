# SANDESH HOSPITALITY — Permission Matrix

**Status:** V1 access decisions confirmed; detailed mapping is a proposed implementation baseline requiring final product/security review  
**Version:** 0.2.0  
**Related documents:** `docs/security/SECURITY-ARCHITECTURE.md`, `docs/database/DATABASE-DESIGN.md`, `docs/api/API-DESIGN.md`

> This matrix is a starting point for authorization design. It does not define final roles, legal responsibilities, or implemented permissions. The product owner must approve role names, scope, and sensitive-action rules before implementation.

**Controlling V1 reference:** [V1 Product and Architecture Baseline](../planning/V1-PRODUCT-AND-ARCHITECTURE-BASELINE.md). This document defines the detailed design for its domain; the baseline controls release scope and the remaining external validation gates. Design text is not implemented behavior.

## 1. Purpose

Define a consistent permission model for users operating within organizations and hotel properties. The model must support multi-tenant isolation, least privilege, explicit authorization for sensitive actions, and auditable changes.

Authorization must be checked server-side for every protected operation. Database RLS is an additional enforcement layer, not a replacement for application authorization.

## 2. Authorization principles

1. **Deny by default:** an action is unavailable unless an explicit permission grants it.
2. **Scope every grant:** permissions apply at organization, property, or self scope. A role assignment at one property must not grant access to another property.
3. **Separate identity from authorization:** Supabase Auth authenticates users; trusted membership and role records determine access.
4. **Do not trust client claims:** organization IDs, property IDs, role names, and permission lists sent by the browser are not proof of access.
5. **Least privilege:** grant only the actions needed for a person's assigned duties.
6. **Sensitive actions need stronger controls:** financial adjustments, refunds, user administration, exports, and workflow publishing require explicit permissions and audit records.
7. **Re-check at execution time:** background jobs and automation actions must validate tenant scope and action permissions when they run.
8. **Audit privilege changes:** role grants, removals, scope changes, and sensitive operations must be recorded.
9. **No implicit cross-tenant support access:** platform support access, if required, must be a separate approved and audited process.

## 3. Proposed role model

These are candidate role templates, not a final role catalog.

| Candidate role | Intended responsibility | Default scope |
|---|---|---|
| Organization Owner | Owns the tenant account and manages organization-level settings | Organization |
| Organization Administrator | Manages properties, memberships, and configuration within granted authority | Organization |
| General Manager | Oversees hotel operations and property-level reporting | Assigned property or properties |
| Front Desk Manager | Supervises reservations, arrivals, departures, and front-desk staff | Assigned property |
| Front Desk Agent | Handles permitted reservation and guest check-in/out tasks | Assigned property |
| Reservation Agent | Creates and updates reservations within policy | Assigned property |
| Housekeeping Manager | Assigns and supervises housekeeping work | Assigned property |
| Housekeeping Staff | Views and updates assigned housekeeping tasks | Assigned property / assigned tasks |
| Restaurant Manager | Manages restaurant operations, menu, and staff workflows | Assigned property/outlet |
| Restaurant Staff | Handles permitted restaurant orders and service tasks | Assigned property/outlet |
| Cashier | Handles approved payment collection and cashier-session operations | Assigned property |
| Finance Manager | Reviews financial records and performs approved financial operations | Organization or assigned properties |
| Auditor / Read-only | Reads explicitly approved operational and financial reports | Explicitly assigned scope |
| Maintenance Staff | Views and updates assigned maintenance work | Assigned property / assigned tasks |
| Automation Manager | Creates, tests, and publishes approved workflows | Assigned property or organization |
| Platform Operator | Operates platform infrastructure, not ordinary tenant business data | Platform-only; no default tenant data access |

A user may hold more than one role only when the tenant's policy permits it. Combining roles must not bypass separation-of-duties controls. A person cannot gain broader scope merely because they hold the same role name in another property.

## 4. Permission vocabulary

Permissions should be represented as stable machine-readable identifiers, grouped by domain and action. Examples:

- `organization.read`, `organization.update`
- `property.read`, `property.create`, `property.update`
- `membership.read`, `membership.invite`, `membership.role.assign`, `membership.revoke`
- `reservation.read`, `reservation.create`, `reservation.update`, `reservation.cancel`
- `reservation.check_in`, `reservation.check_out`
- `guest.read_limited`, `guest.read_sensitive`, `guest.update`, `guest.export`
- `room.read`, `room.update_status`, `room.manage`
- `housekeeping.task.read`, `housekeeping.task.assign`, `housekeeping.task.update`
- `restaurant.order.read`, `restaurant.order.create`, `restaurant.order.update`, `restaurant.menu.manage`
- `folio.read`, `folio.post_charge`, `folio.adjust`
- `payment.collect`, `payment.refund`, `payment.reconcile`
- `report.operational.read`, `report.financial.read`, `report.export`
- `automation.read`, `automation.create`, `automation.test`, `automation.publish`, `automation.disable`
- `audit.read`, `integration.manage`

The final catalog must be derived from actual use cases and API operations. Avoid broad permissions such as `admin.*` when narrower actions are practical.

## 5. Initial role-to-permission matrix

Legend:

- **F** — proposed full permission for the named domain, still restricted to the role's assigned scope.
- **L** — limited operational permission; sensitive fields/actions remain separately restricted.
- **A** — approve or authorize only where a dedicated approval flow exists.
- **R** — read-only permission.
- **—** — no default permission.
- **C** — conditional; grant only after an explicit business-policy decision.

| Domain / action group | Org Owner | Org Admin | General Manager | Front Desk Mgr | Front Desk Agent | Reservation Agent | Housekeeping Mgr | Housekeeping Staff | Restaurant Mgr | Restaurant Staff | Cashier | Finance Mgr | Auditor | Maintenance Staff | Automation Mgr |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| Organization settings | F | F | R | — | — | — | — | — | — | — | — | R | — | — | — |
| Property configuration | F | F | L | — | — | — | — | — | — | — | — | R | R | — | — |
| Membership administration | F | F | C | — | — | — | — | — | — | — | — | — | — | — | — |
| Room and room-type setup | F | F | F | L | R | R | L | R | — | — | — | R | R | L | — |
| Reservation management | F | F | F | F | L | F | R | — | — | — | R | R | R | — | — |
| Check-in / check-out | F | F | F | F | L | — | R | — | — | — | L | R | R | — | — |
| Guest profile data | F | F | L | L | L | L | — | — | — | — | L | C | C | — | — |
| Housekeeping operations | F | F | F | R | R | — | F | L | — | — | — | — | R | — | C |
| Restaurant operations | F | F | F | R | — | — | — | — | F | L | L | R | R | — | C |
| Folio read access | F | F | F | L | L | L | — | — | L | — | L | F | R | — | — |
| Folio charges / adjustments | F | C | A | C | — | — | — | — | C | — | L | F | — | — | — |
| Payment collection | F | C | A | L | — | — | — | — | L | — | L | F | — | — | — |
| Refunds | C | C | A | — | — | — | — | — | — | — | — | F | — | — | — |
| Financial reconciliation | F | C | R | — | — | — | — | — | — | — | L | F | R | — | — |
| Operational reports | F | F | F | L | L | L | L | — | L | — | L | F | R | — | C |
| Financial reports / export | F | C | C | — | — | — | — | — | — | — | — | F | R | — | — |
| Automation management | F | C | C | C | — | — | C | — | C | — | — | C | R | — | F |
| Audit log access | F | C | C | C | — | — | — | — | — | — | — | C | R | — | R |
| Integration management | F | C | C | — | — | — | — | — | C | — | — | C | — | — | C |

This matrix is intentionally conservative and illustrative. In particular, **F** does not mean unrestricted access to all tenant data, and **C/A** must not be treated as automatic grants. The final matrix must be split into individual permission identifiers and tested per API operation.

## 6. Important limits and separation of duties

The role matrix alone is not enough for high-risk actions. Add explicit policy checks:

### Financial operations

- Cashier collection permission does not imply refund, reconciliation, or ledger-adjustment permission.
- Refunds should require a dedicated permission and may require a second authorized approver based on amount and policy.
- Posted folio entries must be corrected through reversal/adjustment operations, not edited or deleted.
- Cashier close/reconciliation should record the operator and any reviewer.
- A user should not approve their own high-risk adjustment where separation of duties is required.

### Guest information

- Separate basic operational details from sensitive identity/contact data.
- Mask or omit sensitive fields unless the caller has the specific permission.
- Guest export must be a separate permission, audited, rate-limited, and restricted to approved purposes.
- Housekeeping and restaurant roles should not receive broad guest-profile access by default.

### Membership and role administration

- Users may only grant roles within their own authority and scope.
- A role administrator must not grant a role that exceeds their own grant authority.
- Prevent accidental removal of the last organization owner.
- Changes to membership, role, or property scope must be audited and take effect promptly.
- Platform operators must not silently impersonate tenant users.

### Automation

- Workflow editing, testing, publishing, disabling, and execution inspection are separate capabilities.
- Publishing must validate the action catalog and permission requirements.
- Execution-time checks must not rely solely on the permissions of the user who originally created the workflow.
- Sensitive actions such as refunds, financial adjustments, and user administration should not be available as unattended workflow actions unless a separate approval design explicitly allows them.
- Test mode must not send real notifications, charge payments, issue refunds, or mutate production business state.

## 7. Scope evaluation model

Every authorization decision should evaluate:

1. Authenticated actor.
2. Active organization membership.
3. Active property membership or organization-level scope grant.
4. Role assignment and permission identifier.
5. Requested action.
6. Resource ownership and organization/property scope.
7. Resource state and any additional business policy.
8. Required approval or separation-of-duties conditions.

A role name or a client-provided `organization_id` is never sufficient by itself.

Candidate conceptual grant:

```text
actor + membership + role + permission + scope
    -> authorize(action, resource)
```

For multi-property users, the server must resolve and validate the active property for each request. Background workers must carry a trusted, explicit tenant/property context and revalidate it before executing a job.

## 8. Database and API implementation requirements

- Keep the permission catalog stable and version-controlled.
- Model role assignments through trusted membership relationships.
- Use composite tenant/property foreign keys for scope integrity.
- Apply RLS to tenant-specific tables and test with multiple organizations and properties.
- Centralize permission evaluation so route handlers, services, and background jobs do not invent inconsistent rules.
- Do not put authorization decisions in user-editable Auth metadata.
- Avoid broad service-role database access as an authorization shortcut.
- Review SECURITY DEFINER functions and views carefully.
- Return 403 for a known resource the actor may not access only where doing so does not leak cross-tenant existence; otherwise use consistent 404 behavior.
- Audit role changes and sensitive operations.
- Revoke access promptly when a membership or role is removed.

## 9. Required authorization tests

At minimum, automate tests for:

- No session, expired session, and invalid session.
- User with no organization membership.
- User with membership in organization A attempting to access organization B.
- User assigned to property A attempting to access property B within the same organization.
- User with read permission attempting a write.
- User with operational permission attempting a financial action.
- Cashier attempting a refund or reconciliation without the required grant.
- User attempting to grant a role above their authority.
- Last organization owner protection.
- Sensitive guest field masking and export restrictions.
- Automation execution after the creator's access has been revoked.
- Cross-tenant background job/event payloads.
- RLS behavior for SELECT, INSERT, UPDATE, and DELETE.
- Audit records for role grants, revocations, approvals, and sensitive operations.

Tenant isolation, least privilege, and financial authorization tests are release blockers.

## 10. Decisions requiring approval

1. ~~Final role list and whether tenants can create custom roles.~~ Resolved for V1: fixed platform-defined role catalog; see section 12.
2. ~~Whether organization roles automatically apply to all properties.~~ Resolved for V1: organization-scope roles apply only for explicitly mapped permissions within their organization; property roles require explicit property membership.
3. ~~Whether property-specific membership is mandatory or optional.~~ Resolved for V1: operational property roles require explicit property membership.
4. Which guest fields are classified as sensitive.
5. ~~Refund thresholds and approval requirements.~~ Resolved for Nepal V1: property defaults are NPR 25,000 per refund and NPR 5,000 per folio adjustment, with cumulative checks and mandatory exceptions; see section 17.
6. Who may adjust folios, void invoices, close cashier sessions, and reconcile payments.
7. Whether one person may hold conflicting roles.
8. Which reports may be exported and by whom.
9. Whether automation publishing requires a second approver.
10. Whether platform support can obtain time-limited, audited tenant access.
11. Whether restaurant outlets need their own access scope.
12. Whether biometric attendance administration is a separate permission domain.

## 11. Approval gate

Do not implement authorization from this draft as-is. First map every planned API operation and UI action to an individual permission, confirm role scope with hotel operational stakeholders, define sensitive-data classifications, and approve financial separation-of-duties rules. Then implement and test the same policy across server routes, domain services, background workers, and database RLS.

**Status:** Initial proposal — not an approved or implemented permission system.


## 12. Confirmed V1 role and membership decisions

The product owner has selected the following V1 direction:

- **Membership model: Hybrid.** Organization Owner and Organization Administrator are organization-scoped. Operational staff require explicit property-scoped membership and cannot access sibling properties through that membership.
- **Role model: Fixed.** V1 role definitions, permission definitions, and role-to-permission mappings are platform-defined and migration-managed. Tenants cannot create or modify custom roles or permission mappings in V1.
- A user may belong to multiple organizations.
- Organization-level access is limited to the permissions explicitly assigned to the organization-scoped role and data belonging to that organization.
- Property-scoped grants remain restricted to the assigned property.
- Platform support/operator access is separate from tenant roles and has no default tenant business-data access.
- Custom tenant roles are deferred beyond V1.

The concrete proposed tables, composite foreign keys, role-assignment structure, lifecycle rules, and RLS requirements are documented in [FOUNDATION-SCHEMA.md](../database/FOUNDATION-SCHEMA.md).

**Status note:** The membership and role-model decisions above are confirmed for V1. The V1 permission catalog and proposed fixed role mapping are now documented in sections 14–16. Remaining sensitive-action implementation and other open product decisions still require review. This document is not evidence that authorization has been implemented or tested.


## 13. Confirmed V1 Organization Administrator authority

**Decision: Limited administration.** Organization Administrator is not equivalent to Organization Owner.

Allowed:
- Read/update permitted organization settings and manage properties within the organization.
- Invite, suspend and revoke ordinary staff memberships.
- Assign/revoke approved property-scoped operational roles within the same organization.
- Read operational reports, permitted automation metadata and audit records.

Not allowed:
- Grant, revoke, suspend, demote or transfer an Organization Owner.
- Grant or revoke Organization Administrator roles.
- Change the membership or roles of an Owner or another Administrator.
- Transfer ownership or billing ownership.
- Modify the fixed role catalog, permission catalog or role-to-permission mappings.
- Access sensitive guest data/exports, refunds, financial reconciliation, folio adjustments, financial exports or integration secrets merely by virtue of being an Administrator.

The server must enforce an explicit allowlist of property roles that an Administrator may assign. Organization Owner and Administrator roles are excluded from that allowlist. Only an Owner can manage those organization-level role assignments, with last-owner and owner-transfer safeguards.

## 14. V1 permission catalog

| Domain | Permission codes |
|---|---|
| Organization | `organization.read`, `organization.update` |
| Property | `property.read`, `property.create`, `property.update`, `property.manage` |
| Membership | `membership.read`, `membership.invite`, `membership.suspend`, `membership.revoke`, `membership.role.assign`, `membership.role.revoke`, `membership.owner.transfer` |
| Reservations | `reservation.read`, `reservation.create`, `reservation.update`, `reservation.cancel`, `reservation.check_in`, `reservation.check_out` |
| Guests | `guest.read_limited`, `guest.read_sensitive`, `guest.create`, `guest.update`, `guest.export` |
| Rooms | `room.read`, `room.update_status`, `room.manage` |
| Housekeeping | `housekeeping.task.read`, `housekeeping.task.assign`, `housekeeping.task.update` |
| Restaurant | `restaurant.order.read`, `restaurant.order.create`, `restaurant.order.update`, `restaurant.menu.manage` |
| Folio | `folio.read`, `folio.post_charge`, `folio.adjust`, `folio.adjust.approve` |
| Payments / Finance policy | `payment.collect`, `payment.refund`, `payment.refund.approve`, `payment.reconcile`, `finance.approval_policy.manage` |
| Cashier sessions | `cashier.session.open`, `cashier.session.close` |
| Reports | `report.operational.read`, `report.financial.read`, `report.export` |
| Automation | `automation.read`, `automation.create`, `automation.test`, `automation.publish`, `automation.disable` |
| Audit | `audit.read` |
| Integrations | `integration.manage` |

## 15. Fixed V1 role-to-permission mapping

| Fixed V1 role | Scope | Permission grants |
|---|---|---|
| `organization_owner` | Organization | All catalog permissions, within organization scope and subject to approval separation-of-duties, last-owner safeguards and business rules. |
| `organization_admin` | Organization | `organization.read`, `organization.update`, `property.read`, `property.create`, `property.update`, `membership.read`, `membership.invite`, `membership.suspend`, `membership.revoke`, `membership.role.assign`, `membership.role.revoke`, `report.operational.read`, `automation.read`, `audit.read`. Role-management operations are restricted by section 13. |
| `general_manager` | Property | `property.read`, `property.update`, `room.read`, `room.update_status`, `room.manage`, all reservation permissions, `guest.read_limited`, `guest.create`, `guest.update`, all housekeeping task permissions, `restaurant.order.read`, `restaurant.order.create`, `restaurant.order.update`, `folio.read`, `folio.post_charge`, `report.operational.read`, `report.financial.read`, `automation.read`, `audit.read`. |
| `front_desk_manager` | Property | `room.read`, `room.update_status`, all reservation permissions, `guest.read_limited`, `guest.create`, `guest.update`, `folio.read`, `folio.post_charge`, `report.operational.read`, `automation.read`. |
| `front_desk_agent` | Property | `room.read`, `room.update_status`, `reservation.read`, `reservation.create`, `reservation.update`, `reservation.check_in`, `reservation.check_out`, `guest.read_limited`, `guest.create`, `guest.update`, `folio.read`. |
| `reservation_agent` | Property | `room.read`, `reservation.read`, `reservation.create`, `reservation.update`, `reservation.cancel`, `guest.read_limited`, `guest.create`, `guest.update`. |
| `housekeeping_manager` | Property | `room.read`, `room.update_status`, `housekeeping.task.read`, `housekeeping.task.assign`, `housekeeping.task.update`, `report.operational.read`. |
| `housekeeping_staff` | Property | `room.read`, `housekeeping.task.read`, `housekeeping.task.update`, restricted to assigned tasks and allowed state transitions. |
| `restaurant_manager` | Property | `restaurant.order.read`, `restaurant.order.create`, `restaurant.order.update`, `restaurant.menu.manage`, `folio.read`, `report.operational.read`. |
| `restaurant_staff` | Property | `restaurant.order.read`, `restaurant.order.create`, `restaurant.order.update`, restricted to assigned outlet/order state. |
| `cashier` | Property | `reservation.read`, `guest.read_limited`, `folio.read`, `folio.post_charge`, `payment.collect`, `cashier.session.open`, `cashier.session.close`. |
| `finance_manager` | Property | `reservation.read`, `guest.read_limited`, `folio.read`, `folio.post_charge`, `folio.adjust`, `payment.collect`, `payment.refund`, `payment.reconcile`, `report.operational.read`, `report.financial.read`, `report.export`, `audit.read`. Does not receive approval permissions by default. |
| `auditor_read_only` | Property | `reservation.read`, `room.read`, `guest.read_limited`, `housekeeping.task.read`, `restaurant.order.read`, `folio.read`, `report.operational.read`, `report.financial.read`, `audit.read`. |
| `maintenance_staff` | Property | `room.read`, maintenance-limited `room.update_status`. Maintenance task access is deferred until a dedicated maintenance task model exists. |
| `automation_manager` | Property | `automation.read`, `automation.create`, `automation.test`, `automation.publish`, `automation.disable`, `report.operational.read`, `audit.read`. |

**Important:** This is a proposed V1 baseline, not a grant to bypass server-side business rules. `payment.refund` and `folio.adjust` require approval by a separate authorized actor; the initiator cannot approve their own operation. The approval permissions are not granted to finance_manager by default. `guest.read_sensitive`, `guest.export`, `payment.refund.approve`, and `folio.adjust.approve` have no ordinary role grants in this baseline pending a separate policy decision. The Owner's broad authority does not remove the requirement for independent approval of their own high-risk operation.

## 16. Catalog consistency and implementation gates

- Seed role, permission and mapping rows only through reviewed migrations.
- Role assignments must validate role scope against the target assignment table.
- Organization Administrator role grants must be restricted to an allowlist of property roles; Owner/Admin role grants are owner-only.
- Organization Admin cannot manage another Owner or Administrator membership.
- Every API route, server action, background job and automation action must map to a permission and enforce scope.
- Test every allow and deny, including cross-tenant, cross-property, admin-to-owner escalation, last-owner removal, sensitive-data access and approval separation.
- No SQL migration has been applied as part of this documentation update.


## 17. Confirmed V1 financial approval policy — Nepal launch

**Status:** Confirmed product policy for the Nepal V1 production design. Not yet implemented.

### Threshold defaults and configuration

| Operation | Nepal V1 default | Threshold approval |
|---|---:|---|
| Payment refund | NPR 25,000 | Required above the threshold |
| Folio adjustment | NPR 5,000 | Required above the threshold |

- Thresholds are configurable per property, not globally shared across an organization.
- An active Organization Owner is the only role permitted to change a property's thresholds.
- Organization Administrators and property-scoped roles cannot change threshold settings.
- Threshold changes are audited and apply prospectively; they do not change the rules attached to already-pending requests.
- The property's configured operating currency is authoritative. NPR is the Nepal V1 default. Non-NPR transaction evaluation requires an immutable booked FX snapshot and defined rounding behavior before non-NPR payments are enabled.
- An amount exactly equal to the threshold does not trigger threshold-based approval; mandatory exceptions still require approval.

### Initiation and execution authority

- Finance Managers may initiate refunds and folio adjustments within their assigned property.
- Organization Owners may initiate these operations within their organization.
- Organization Administrators, Cashiers, Front Desk roles, and other operational roles do not receive initiation authority by default.
- `payment.refund` and `folio.adjust` are initiation permissions. They do not authorize bypassing server-side threshold, exception, ledger, provider, or approval checks.
- Below-threshold requests without a mandatory exception may be executed by an authorized initiator through the controlled backend financial service.
- Above-threshold requests and exception requests remain pending until a valid Owner approval is recorded.

### Owner-only approval and separation of duties

- Only an active Organization Owner may approve.
- A Finance Manager's request may be approved by any other active Owner in the same organization.
- An Owner's own request requires approval by a different active Owner.
- Self-approval is always denied.
- If only one active Owner exists, an Owner-originated request requiring approval is blocked; there is no Administrator or Finance Manager fallback.
- Approval is bound to the exact transaction/request version, property, amount, currency, original payment or folio entry, destination where applicable, and initiator. Material changes require a new approval.
- Approval must be checked server-side immediately before execution. Execution must be idempotent and safely coordinated with ledger/provider state.

### Mandatory approval regardless of amount

1. Cash refunds.
2. Refunds to a payment method or destination different from the original.
3. Provider-flagged or trusted risk-flagged transactions.
4. Folio adjustments affecting a closed accounting period.
5. Reversals or corrections of previously posted financial transactions.

### Cumulative anti-splitting controls

- Aggregate refunds against the original payment reference, including completed and pending related refunds. Never permit cumulative refunds above the refundable amount.
- Check individual and cumulative absolute folio adjustments against the same folio within a rolling 24-hour window; active pending/approved reservations remain reserved until conclusively resolved.
- Link and aggregate corrections to the same original folio entry across the full lifecycle, regardless of the rolling window.
- Pending requests must reserve or lock the relevant refundable/adjustable amount to prevent concurrent bypass.
- All initiations, decisions, executions, failures, cancellations, and threshold changes are auditable.

### Permission catalog additions

Add `finance.approval_policy.manage` to the permission catalog and grant it only to `organization_owner`. The existing `payment.refund.approve` and `folio.adjust.approve` permissions are also Owner-only. The Finance Manager receives initiation permissions but no approval permissions. Role grants never override the independent-approver rule.

### Required test gates

Test threshold boundaries, cumulative split transactions, exception cases, owner-vs-finance-manager initiators, self-approval denial, sole-owner blocking, unauthorized roles, pending-request immutability after threshold changes, concurrent requests, idempotency, and cross-tenant/property denial.

**Implementation gate:** Design and review the property threshold settings, approval request/decision records, audit events, and backend execution boundary before creating migrations. This policy update does not apply a database migration or activate live refunds.
