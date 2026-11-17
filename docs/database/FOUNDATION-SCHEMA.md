# SANDESH HOSPITALITY — V1 Foundation Schema

**Status:** Decision-aligned schema design draft — review required before migration  
**Version:** 0.3.0  
**Decisions applied:** Hybrid membership access; platform-defined fixed roles for V1  
**Database:** Supabase PostgreSQL

> This document translates the approved V1 access decisions into a concrete foundation schema proposal. It is not an applied migration and does not claim that RLS or authorization has been implemented. Review the role catalog and permission assignments before executable SQL is created.

## 1. Confirmed V1 decisions

1. A user may belong to multiple organizations.
2. An organization may own multiple properties.
3. Organization Owner and Organization Administrator roles are assigned at organization scope.
4. Organization-scope grants are evaluated across that organization's properties, subject to the permissions attached to the role.
5. Other operational roles are assigned at explicit property scope.
6. A property-scoped membership grants no access to sibling properties.
7. V1 roles and permissions are platform-defined and seeded through reviewed migrations. Tenants cannot create or edit role definitions or permission mappings.
8. A user may hold multiple roles only where policy permits; effective permissions are the union of grants, subject to explicit separation-of-duties and approval checks.
9. Platform operations/support access is not a tenant role and grants no default tenant business-data access.
10. Revoked, suspended, or removed memberships must stop authorizing new requests. Sensitive operations must not rely on stale client claims.

## 2. Scope and authorization model

Every tenant data access decision must establish all of the following:

- Authenticated identity from the verified Supabase session (auth.uid()).
- Active organization membership.
- Active organization-level role assignment OR active property membership and property-level role assignment.
- Required permission from the fixed role-permission catalog.
- Resource ownership by the same organization and, for property-owned resources, the same property.
- Any additional business-state, approval, and separation-of-duties conditions.

**Organization scope:** organization_owner, organization_admin.

**Property scope:** operational roles listed in section 5. An organization-level role may authorize across properties only for permissions explicitly mapped to that role. It does not bypass business rules or approval requirements.

**No implicit inheritance:** a property membership never grants access to another property. A property-scoped role never becomes organization-wide because a user has the same role at another property.

## 3. Proposed tables

### 3.1 Identity and tenant structure

#### user_profiles
- user_id uuid primary key references auth.users(id) on delete cascade
- display_name text null
- avatar_storage_path text null
- created_at timestamptz not null default now()
- updated_at timestamptz not null default now()

Authentication credentials remain exclusively in Supabase Auth. Do not duplicate passwords, tokens, or editable authorization claims here.

#### organizations
- id uuid primary key
- name text not null
- slug text not null unique
- status text not null check in active, suspended, closed
- created_by uuid null references auth.users(id)
- created_at timestamptz not null default now()
- updated_at timestamptz not null default now()

#### properties
- id uuid primary key
- organization_id uuid not null references organizations(id)
- name text not null
- slug text not null
- timezone text not null — IANA time-zone identifier, validated by application/database policy
- status text not null check in active, inactive, closed
- created_at timestamptz not null default now()
- updated_at timestamptz not null default now()
- Constraints: unique (organization_id, id), unique (organization_id, slug)

### 3.2 Memberships

#### organization_memberships
- id uuid primary key
- organization_id uuid not null references organizations(id)
- user_id uuid not null references auth.users(id)
- status text not null check in invited, active, suspended, revoked
- invited_by uuid null references auth.users(id)
- joined_at timestamptz null
- created_at timestamptz not null default now()
- updated_at timestamptz not null default now()
- Constraints: unique (organization_id, user_id), unique (organization_id, id)

A user has at most one organization-membership row per organization. Re-invitation/reactivation updates the lifecycle of that membership rather than creating parallel active memberships.

#### property_memberships
- id uuid primary key
- organization_id uuid not null
- property_id uuid not null
- organization_membership_id uuid not null
- status text not null check in active, suspended, revoked
- created_at timestamptz not null default now()
- updated_at timestamptz not null default now()
- Constraints: unique (organization_id, id), unique (property_id, organization_membership_id)
- Composite FK (organization_id, property_id) -> properties(organization_id, id)
- Composite FK (organization_id, organization_membership_id) -> organization_memberships(organization_id, id)

This prevents a membership from being attached to a property in a different organization.

### 3.3 Fixed role and permission catalog

#### roles
- code text primary key — stable machine-readable identifier
- name text not null
- scope text not null check in organization, property
- description text not null
- is_active boolean not null default true

#### permissions
- code text primary key
- description text not null
- domain text not null

#### role_permissions
- role_code text not null references roles(code)
- permission_code text not null references permissions(code)
- Primary key: (role_code, permission_code)

These catalog tables are migration-managed. Tenant-facing database roles must not be able to insert, update, or delete catalog rows. Do not accept role names or permission arrays from browser requests as authority.

### 3.4 Role assignments

Use two explicit assignment tables rather than a polymorphic scope column. This keeps foreign keys and scope validation enforceable.

#### organization_membership_roles
- id uuid primary key
- organization_id uuid not null
- organization_membership_id uuid not null
- role_code text not null references roles(code)
- granted_by uuid null references auth.users(id)
- granted_at timestamptz not null default now()
- revoked_at timestamptz null
- Composite FK (organization_id, organization_membership_id) -> organization_memberships(organization_id, id)
- Role must have scope = organization (enforce with a controlled grant function/trigger and test; do not trust client input).

#### property_membership_roles
- id uuid primary key
- organization_id uuid not null
- property_membership_id uuid not null
- role_code text not null references roles(code)
- granted_by uuid null references auth.users(id)
- granted_at timestamptz not null default now()
- revoked_at timestamptz null
- Composite FK (organization_id, property_membership_id) -> property_memberships(organization_id, id)
- Role must have scope = property (enforce with a controlled grant function/trigger and test).

Use partial unique indexes to prevent duplicate active grants for the same membership and role. Role revocation is represented by setting revoked_at; preserve grant history for audit. Membership suspension/revocation overrides any unrevoked role assignment.

## 4. Relationship overview

    auth.users
      ├── user_profiles
      └── organization_memberships
            ├── organization_membership_roles
            │     └── roles ── role_permissions ── permissions
            └── property_memberships
                  └── property_membership_roles
                        └── roles

    organizations
      ├── organization_memberships
      └── properties
            └── property_memberships (through organization_memberships)

## 5. V1 fixed role catalog proposal

| Role code | Scope | Purpose |
|---|---|---|
| organization_owner | Organization | Tenant owner and account-level authority |
| organization_admin | Organization | Organization configuration and membership administration within granted permissions |
| general_manager | Property | Property operations and approved property-level oversight |
| front_desk_manager | Property | Front desk supervision |
| front_desk_agent | Property | Permitted front desk tasks |
| reservation_agent | Property | Reservation handling |
| housekeeping_manager | Property | Housekeeping assignment and supervision |
| housekeeping_staff | Property | Assigned housekeeping tasks |
| restaurant_manager | Property | Restaurant operations |
| restaurant_staff | Property | Permitted restaurant service tasks |
| cashier | Property | Approved collection and cashier-session tasks |
| finance_manager | Property | Approved property financial operations |
| auditor_read_only | Property | Explicitly scoped read-only audit/report access |
| maintenance_staff | Property | Assigned maintenance tasks |
| automation_manager | Property | Approved property workflow management |

This is a fixed V1 catalog proposal based on the existing permission-matrix draft. Exact permission grants for each role must be reviewed and seeded explicitly. Platform Operator is deliberately excluded from tenant roles.

## 6. Required indexes and constraints

At minimum, evaluate and add:
- Unique organization_memberships(organization_id, user_id).
- Unique properties(organization_id, slug).
- Unique property_memberships(property_id, organization_membership_id).
- Composite FKs for all organization/property ownership relationships.
- Index organization_memberships(user_id, status).
- Index property_memberships(organization_id, property_id, status).
- Index active role assignments by membership and role.
- Index all tenant-owned child tables by (organization_id, property_id) and their actual query keys.
- Check constraints for membership states, role scopes, and valid lifecycle timestamps.

Avoid speculative indexes beyond the access paths and constraints above.

## 7. RLS and authorization implementation shape

Enable RLS on every tenant-specific table exposed through the Supabase Data API. Revoke unnecessary table privileges separately; RLS is not a substitute for grants.

Policies must distinguish:
- Organization-level visibility and permitted organization operations.
- Explicit property membership and property-level permission.
- Organization Owner/Admin permissions that intentionally apply across that organization's properties.
- Resource ownership and property scope for child rows.

Do not create a generic policy that lets every active organization member read every row.

Membership and role tables are security-sensitive. Users must not be able to grant themselves memberships, elevate their roles, change organization/property scope, or alter fixed catalog rows. Invitation, acceptance, role grant, role revoke, and owner transfer must use controlled server-side operations with audit records.

RLS helper functions may be needed to avoid policy recursion. If SECURITY DEFINER helpers are used, each must have a documented need, fixed safe search_path, minimal grants, explicit auth.uid() checks, no dynamic SQL, and tests for cross-tenant and cross-property access. Prefer SECURITY INVOKER for ordinary functions.

## 8. Lifecycle and audit requirements

- Suspending or revoking an organization membership blocks both organization and property access immediately for new authorization checks.
- Suspending or revoking a property membership blocks that property's access without changing other property memberships.
- Revoking a role removes its permissions while preserving the historical assignment row.
- Prevent removal or demotion of the last active organization owner without a valid owner-transfer transaction.
- Record actor, target membership, old/new scope, role, timestamp, outcome, and request/correlation ID in audit history.
- Do not hard-delete audit records or financial history.
- Use optimistic versioning for mutable operational records where concurrent edits matter.

## 9. Required tests before any production use

1. User with no membership cannot read tenant data.
2. Organization A user cannot read or mutate Organization B data.
3. Property A-only user cannot read or mutate Property B in the same organization.
4. Organization Owner/Admin can access only properties belonging to their organization and only for mapped permissions.
5. Revoked/suspended organization membership blocks all properties.
6. Revoked/suspended property membership blocks only that property.
7. A property role cannot be inserted into organization role assignments and vice versa.
8. A tenant cannot modify roles, permissions, or role-permission mappings.
9. A user cannot grant themselves or another user a role beyond their authority.
10. A non-owner cannot remove/demote the last active organization owner.
11. Cross-organization composite foreign-key references are rejected.
12. SELECT, INSERT, UPDATE, and DELETE RLS behavior is tested with separate synthetic users and organizations.

## 10. Decisions still open before executable migration

- Exact role-to-permission mapping and permission catalog.
- Whether organization administrators may invite/remove owners or only non-owner members.
- Owner transfer and recovery process.
- Whether property-level finance/audit roles need organization-wide reporting variants.
- Guest profile tenant/property scope.
- Country, currency, tax, and invoice configuration.
- Property business-day and night-audit model.
- Invitation delivery and acceptance workflow.
- Audit-log retention and platform support access process.

**Next gate:** Review and approve this schema draft and permission mapping, then create a local migration and pgTAP/RLS tests. Do not apply an unreviewed migration to the shared Supabase project.


## 11. Confirmed V1 permission catalog and role mapping

**Organization Administrator authority: limited (confirmed).** The organization_admin role is organization-scoped, but it is not an owner-equivalent role.

### 11.1 Permission catalog

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
| Payments | `payment.collect`, `payment.refund`, `payment.refund.approve`, `payment.reconcile` |
| Cashier sessions | `cashier.session.open`, `cashier.session.close` |
| Reports | `report.operational.read`, `report.financial.read`, `report.export` |
| Automation | `automation.read`, `automation.create`, `automation.test`, `automation.publish`, `automation.disable` |
| Audit | `audit.read` |
| Integrations | `integration.manage` |

Permission codes are stable API/domain identifiers. Avoid wildcard grants. New permissions require a reviewed migration and explicit role-mapping changes.

### 11.2 Fixed role-to-permission mapping

| Fixed V1 role | Scope | Permission grants |
|---|---|---|
| `organization_owner` | Organization | All organization/property permissions listed in this catalog, including owner transfer, membership administration, financial approvals, integration management and audit. Still subject to immutable safeguards, approval separation-of-duties, tenant scope and business rules. |
| `organization_admin` | Organization | `organization.read`, `organization.update`, `property.read`, `property.create`, `property.update`, `property.manage`, `membership.read`, `membership.invite`, `membership.suspend`, `membership.revoke`, `membership.role.assign`, `membership.role.revoke`, `report.operational.read`, `automation.read`, `audit.read`. Membership/role actions are restricted as described below. |
| `general_manager` | Property | `property.read`, `property.update`, `room.read`, `room.update_status`, `room.manage`, `reservation.read`, `reservation.create`, `reservation.update`, `reservation.cancel`, `reservation.check_in`, `reservation.check_out`, `guest.read_limited`, `guest.create`, `guest.update`, `housekeeping.task.read`, `housekeeping.task.assign`, `housekeeping.task.update`, `restaurant.order.read`, `restaurant.order.create`, `restaurant.order.update`, `restaurant.menu.manage`, `folio.read`, `folio.post_charge`, `report.operational.read`, `report.financial.read`, `automation.read`, `audit.read`. No default refund, reconciliation, sensitive guest export, or approval permission. |
| `front_desk_manager` | Property | `room.read`, `room.update_status`, `reservation.read`, `reservation.create`, `reservation.update`, `reservation.cancel`, `reservation.check_in`, `reservation.check_out`, `guest.read_limited`, `guest.create`, `guest.update`, `folio.read`, `folio.post_charge`, `report.operational.read`, `automation.read`. |
| `front_desk_agent` | Property | `room.read`, `room.update_status`, `reservation.read`, `reservation.create`, `reservation.update`, `reservation.check_in`, `reservation.check_out`, `guest.read_limited`, `guest.create`, `guest.update`, `folio.read`. Cancellation requires manager policy/permission; no payment, refund, sensitive guest, or export permission. |
| `reservation_agent` | Property | `room.read`, `reservation.read`, `reservation.create`, `reservation.update`, `reservation.cancel`, `guest.read_limited`, `guest.create`, `guest.update`. No check-in/out or folio/payment permissions by default. |
| `housekeeping_manager` | Property | `room.read`, `room.update_status`, `housekeeping.task.read`, `housekeeping.task.assign`, `housekeeping.task.update`, `report.operational.read`. |
| `housekeeping_staff` | Property | `room.read`, `housekeeping.task.read`, `housekeeping.task.update` (server enforces assignment-to-self/team and permitted task state transitions). |
| `restaurant_manager` | Property | `restaurant.order.read`, `restaurant.order.create`, `restaurant.order.update`, `restaurant.menu.manage`, `folio.read`, `report.operational.read`. No direct payment refund or financial reconciliation. |
| `restaurant_staff` | Property | `restaurant.order.read`, `restaurant.order.create`, `restaurant.order.update` (restricted to assigned outlet/order state). |
| `cashier` | Property | `reservation.read`, `guest.read_limited`, `folio.read`, `folio.post_charge`, `payment.collect`, `cashier.session.open`, `cashier.session.close`. No refund, adjustment, approval, or reconciliation grant. |
| `finance_manager` | Property | `reservation.read`, `guest.read_limited`, `folio.read`, `folio.post_charge`, `folio.adjust`, `payment.collect`, `payment.refund`, `payment.reconcile`, `cashier.session.read`, `report.operational.read`, `report.financial.read`, `report.export`, `audit.read`. Refund and adjustment operations require separately authorized approval; this role does not receive `payment.refund.approve` or `folio.adjust.approve` by default. |
| `auditor_read_only` | Property | `reservation.read`, `room.read`, `guest.read_limited`, `housekeeping.task.read`, `restaurant.order.read`, `folio.read`, `report.operational.read`, `report.financial.read`, `audit.read`. No write, export, payment or guest-sensitive permission by default. |
| `maintenance_staff` | Property | `room.read`, `room.update_status` (maintenance-related states only; server-enforced), `housekeeping.task.read` only if maintenance tasks are represented in the task system. No reservation, guest, financial or restaurant permissions. |
| `automation_manager` | Property | `automation.read`, `automation.create`, `automation.test`, `automation.publish`, `automation.disable`, `report.operational.read`, `audit.read`. Published workflows may invoke only approved action catalog entries and must re-check actor/service authority, scope and resource state at execution time. |

### 11.3 Organization Administrator hard limits

An organization_admin may:
- Read and update permitted organization settings.
- Create and update properties within the organization, subject to product-level safeguards.
- Invite, suspend, and revoke ordinary staff memberships.
- Assign and revoke approved property-scoped operational roles within the same organization.
- Read operational reports, automation metadata and permitted audit records.

An organization_admin may not:
- Create, assign, revoke, suspend or transfer an organization_owner role.
- Assign or revoke organization_admin roles; those actions are owner-only.
- Suspend, revoke, demote or alter the membership/roles of an organization_owner or another organization_admin.
- Transfer organization ownership or change billing ownership.
- Grant permissions outside the fixed catalog or alter role/permission definitions.
- Receive automatic access to sensitive guest fields, guest exports, refunds, financial reconciliation, folio adjustments, financial exports, or integration secrets through the admin role.

The server-side grant operation must validate both the actor's grant authority and the target role's scope. Client-supplied role codes are never sufficient evidence of authorization.

### 11.4 Sensitive-action and separation-of-duties rules

- `payment.refund` and `folio.adjust` are initiation/execution permissions, not approval permissions.
- `payment.refund.approve` and `folio.adjust.approve` are separate approval permissions. V1 default role mappings do not grant these to finance_manager; owner-level authority may approve only through a dedicated approval flow.
- The initiator must not approve their own refund or adjustment. The approval record must identify both actors and the exact amount/resource.
- A posted folio entry is never edited or deleted; corrections use reversal/adjustment records.
- `guest.read_sensitive` and `guest.export` are not included in ordinary operational role defaults. They require a separately approved data-classification policy and explicit grant design before implementation.
- `report.export` is granted only to finance_manager in this initial mapping; exports must be audited and scoped.
- `automation.publish` does not authorize arbitrary code, unrestricted HTTP, direct database access, or unattended sensitive financial/user-administration actions.
- All permissions remain bounded by the role's organization/property scope, active membership status, resource ownership, and business rules.

### 11.5 Mapping implementation requirements

- Seed roles, permissions and role_permissions through version-controlled migrations only.
- Add a database constraint/trigger or controlled grant function that rejects organization-scoped roles in property assignments and property-scoped roles in organization assignments.
- The organization_admin grant function must use an allowlist of assignable property roles and reject organization_owner and organization_admin targets.
- Only organization_owner may grant/revoke organization_admin and organization_owner roles, subject to last-owner protection and owner-transfer flow.
- Never let a user modify role_permissions or the catalog through ordinary tenant-facing APIs.
- Test every permission grant and denial with distinct users, organizations and properties. Test privilege escalation attempts, including admin-to-owner and cross-organization role assignment.

**Status:** Organization Administrator limited authority is confirmed. The permission catalog and role mapping above are the V1 proposed baseline for product-owner review; they are not yet implemented in Supabase.
