# Front Desk Workflows

**Status:** Initial proposal — requires product and operational review  
**Scope:** V1 web-first, cloud-first hotel operations  
**Related documents:** `PRODUCT-REQUIREMENTS.md`, `SYSTEM-ARCHITECTURE.md`, `DATABASE-DESIGN.md`, `PERMISSION-MATRIX.md`, `RESERVATION-LIFECYCLE.md`, `API-DESIGN.md`, `AUTOMATION-ENGINE.md`

> This document describes intended behavior. It is not implemented software, a final legal policy, or a substitute for property-specific operating procedures.

**Controlling V1 reference:** [V1 Product and Architecture Baseline](../planning/V1-PRODUCT-AND-ARCHITECTURE-BASELINE.md). Use its release boundary and defaults where this earlier proposal is broader or still marks a decision open. External validation gates remain mandatory; this file is a design proposal, not proof of implementation.

## 1. Goals and operating principles

The front desk is the operational control point for arrivals, in-house guests, departures, room assignment, guest requests, and shift handover.

- Every operational action must be attributable to an authenticated user and property.
- Reservation and inventory rules must be enforced by backend transactions, not by UI assumptions.
- A walk-in must follow the same availability, rate, reservation, and inventory controls as any other booking.
- Staff must see only the guest and property data required for their assigned duties.
- Exceptional actions must require explicit permission, reason capture, and audit history.
- Automation failures must not prevent core front-desk work.
- V1 requires an internet connection. It does not provide offline check-in or offline booking.

## 2. Front-desk dashboard

The dashboard should be property-scoped and show:

- Expected arrivals, including requested early arrival where recorded.
- Expected departures and late-checkout requests.
- In-house stays and room occupancy.
- Rooms marked clean, inspected, dirty, occupied, out of service, or otherwise unavailable.
- Reservations awaiting confirmation, payment, room assignment, or guest details.
- Outstanding balances and payment exceptions, subject to staff permissions.
- Walk-in and booking activity for the current business date.
- Operational alerts: room not ready, overbooking risk, unresolved room move, payment failure, and maintenance hold.

Dashboard counts must use consistent property-local business dates and clearly distinguish reservation status from room status.

## 3. Shift start and handover

At shift start, staff should review:

1. Current arrivals and departures.
2. In-house guests and expected extensions.
3. Rooms unavailable or not ready.
4. Unresolved guest requests and maintenance issues.
5. Pending payments, refunds, disputed charges, and cashier exceptions where authorized.
6. Tasks carried forward from the previous shift.

A handover record should include author, property, shift/business date, timestamp, open items, assigned follow-up, and completion status. Handover notes must not become an unrestricted place to store sensitive guest information.

The exact cashier-session and shift-closing model remains an open operational decision.

## 4. Arrival and check-in

### 4.1 Find the booking

- Search by reservation reference or limited guest identifiers.
- Show only the minimum information needed to identify the correct booking.
- Confirm property, dates, party size, booking status, room/rate details, and special requests.
- Prevent accidental selection of a similarly named guest by showing a safe secondary identifier.

### 4.2 Validate readiness

Before check-in, validate on the server:

- Reservation belongs to the active organization and property context.
- Reservation is eligible for check-in and has not already been checked in or cancelled.
- Stay dates and room assignment are valid.
- Assigned room is not out of service, blocked, or assigned to an overlapping stay.
- Required guest details and identity checks satisfy the property's configured policy and applicable local requirements.
- Deposit/payment requirements are met or an authorized exception is recorded.

Do not hard-code identity-document requirements until target jurisdictions and legal requirements are confirmed.

### 4.3 Complete check-in

- Confirm the room and guest party.
- Present any required policy acknowledgement or registration information.
- Execute check-in as an atomic, idempotent server operation.
- Record actor, timestamp, property, room, reservation, and relevant before/after state.
- Update the stay and room occupancy consistently.
- Issue a physical key/card only where a supported hotel key system exists; otherwise record the property's manual key procedure.

A repeated request must not create duplicate check-ins or duplicate charges.

## 5. Walk-in booking

A walk-in is a reservation created at the property for a guest who arrives without an existing booking.

- Collect only required booking and guest details.
- Check availability and rate using the same backend rules as all other channels.
- Create the reservation and inventory allocation atomically.
- Apply the property's deposit and payment policy.
- Assign a room only after room eligibility is confirmed.
- Complete check-in through the standard check-in flow.

There must be no receptionist-only bypass for room inventory, rates, or payment records. Any permitted override requires a named permission, reason, audit event, and relevant approval policy.

## 6. Room assignment and room changes

- Show room availability for the requested stay dates, not merely current room cleanliness.
- Filter by property, room type, capacity, accessibility attributes where configured, and operational status.
- Revalidate availability at commit time; the UI's earlier result is not a guarantee.
- Prevent overlapping physical-room assignments through database constraints and transactional logic.
- A room move must record the old room, new room, effective time, reason, actor, and any folio or housekeeping implications.
- A room must not be assigned while out of service or blocked.
- Returning a room to service requires the configured authorized role and an auditable status change.

Room assignment timing (at booking, pre-arrival, or check-in) may vary by property and must be configurable without weakening inventory integrity.

## 7. In-house guest service

Authorized staff may view the stay details needed to handle requests, such as room number, stay dates, occupancy, and relevant service notes.

- Guest requests should become trackable tasks with status, priority, owner, timestamps, and property scope.
- Requests involving housekeeping, maintenance, restaurant orders, or billing should be routed to the corresponding module.
- Sensitive guest data must not be copied into general-purpose notes or automation payloads.
- Staff actions and material changes must be audited.

## 8. Departure and checkout

### 8.1 Review the folio

- Retrieve the current folio from the authoritative server-side ledger.
- Show itemized charges, payments, adjustments, taxes/fees where configured, and outstanding balance.
- Resolve pending restaurant or service charges according to the property's configured posting policy.
- Require appropriate permissions and reason capture for adjustments, voids, and refunds.
- Never silently overwrite or delete posted financial entries.

### 8.2 Settle and check out

- Record payment through an approved payment method or record an authorized payment exception.
- Verify the final balance and any required approval.
- Execute checkout as an atomic, idempotent operation.
- Mark the stay checked out and create the required housekeeping task.
- Preserve the folio and audit history.

Checkout must not silently mark a room clean. Housekeeping status changes follow the housekeeping workflow.

## 9. Exceptions and edge cases

| Scenario | Required behavior |
|---|---|
| Early arrival | Check room readiness and configured early check-in policy; do not bypass room/inventory controls. |
| Late checkout | Apply configured policy, price/approval rules, and update room availability implications. |
| Room not ready | Keep the stay pending; offer only valid alternatives and record guest communication. |
| No-show | Follow the reservation lifecycle and configured cutoff/charge policy; require authorization for overrides. |
| Guest requests extension | Recheck inventory for every additional night and commit atomically. |
| Room move | Validate destination room, record reason and timestamps, preserve history, and create housekeeping tasks as needed. |
| Payment provider unavailable | Follow documented manual payment procedure only if approved; never claim the provider confirmed a payment. |
| Guest disputes a charge | Record a dispute/case and route it for review; do not erase the ledger entry. |
| Duplicate guest/booking match | Require staff confirmation using safe secondary details; do not merge records automatically. |
| Overbooking or inventory conflict | Block unsafe assignment and escalate to an authorized manager; do not silently over-allocate. |
| Internet/service outage | Use the property's approved printable/manual contingency process; enter and reconcile records after service returns. |

## 10. Cashier and financial controls

The front-desk UI may expose cashier functions only to roles explicitly granted those permissions.

Candidate controls:
- Open and close cashier sessions.
- Record opening float and closing count.
- Reconcile expected versus counted amounts by payment method.
- Record discrepancies with reason and manager review.
- Separate payment collection, refunds, adjustments, and reconciliation permissions where practical.
- Preserve immutable financial entries and audit history.

Whether cashier sessions are per-user, per-terminal, or per-shift is unresolved and must be decided before implementing cashier workflows.

## 11. Events and automation

Candidate domain events include:

- `stay.checked_in`
- `stay.checked_out`
- `room.assignment_changed`
- `guest.request_created`
- `housekeeping.task_created`
- `cashier.session_opened`
- `cashier.session_closed`

Events must be emitted through the transactional outbox and include tenant/property scope, event ID, schema version, occurrence time, and correlation/idempotency identifiers as defined in the automation design.

Examples:
- Check-in may notify configured internal teams.
- Checkout may create a housekeeping task.
- A room move may create cleaning/inspection tasks for the vacated room.
- Shift-end reports may summarize operational exceptions for authorized recipients.

Automation must never be the sole mechanism for enforcing room availability, payment integrity, authorization, or check-in/checkout correctness.

## 12. Permissions and audit

Use the permission model in `PERMISSION-MATRIX.md` as the starting proposal. At minimum, enforce server-side checks for:

- Viewing guest/reservation details.
- Creating walk-ins.
- Assigning and moving rooms.
- Checking in and checking out.
- Posting payments and adjustments.
- Issuing refunds or voids.
- Overriding policy exceptions.
- Opening/closing cashier sessions.
- Viewing reports and exporting data.

Every sensitive mutation must capture actor, organization, property, timestamp, action, target record, and relevant before/after values or a safe change summary. Do not rely on hidden UI controls as authorization.

## 13. V1 outage procedure

V1 is cloud-first and requires internet connectivity. If the internet or hosted service is unavailable:

- Staff use property-approved printable arrival, in-house, departure, and emergency contact sheets prepared while service is available.
- Staff use numbered manual records for new arrivals, departures, payments, and room changes.
- Staff must not represent manual entries as synchronized or confirmed in the system.
- After service is restored, an authorized staff member enters and reconciles each manual transaction, checking for duplicate bookings, room conflicts, and payment discrepancies.
- The reconciliation record must preserve the original manual reference, entry actor, actual event time, system entry time, and reviewer where required.

This is a documented manual contingency, not offline application functionality.

## 14. Acceptance criteria

The front-desk workflows are not ready for release until tests demonstrate:

- A staff member cannot access another tenant's or unauthorized property's records.
- A walk-in cannot bypass availability, rate, or inventory checks.
- Concurrent check-ins/assignments cannot allocate the same physical room to overlapping stays.
- Repeated check-in, checkout, and payment requests are idempotent.
- Room moves preserve history and validate both rooms.
- Checkout does not mark a room clean without the housekeeping process.
- Financial adjustments and refunds enforce permissions and audit history.
- Payment-provider failures cannot be mistaken for successful payments.
- Outbox/automation failure does not block core check-in or checkout.
- Dashboard counts and business dates are property-timezone aware.
- The manual outage reconciliation process can detect duplicates and inventory conflicts.

## 15. Open decisions before implementation

1. Which countries, legal jurisdictions, and hotel operating rules are in V1?
2. What guest identity/registration information is mandatory in each target market?
3. Are walk-ins allowed at every property, and what deposit/payment rules apply?
4. When is room assignment normally made?
5. What are the early check-in, late checkout, no-show, and overbooking policies?
6. Which payment gateways and manual fallback methods are supported?
7. Are cashier sessions per user, terminal, or shift, and who reconciles discrepancies?
8. Which key-card/door-lock vendors, if any, need integration?
9. What is the exact shift handover and escalation process?
10. Which printed outage forms and reconciliation approvals are required?

These decisions must be resolved or explicitly deferred before the corresponding feature is implemented.
