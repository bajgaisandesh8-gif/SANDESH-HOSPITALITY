# SANDESH HOSPITALITY — V1 Payment & Folio Ledger Design

**Status:** Detailed design proposal — product and finance review required  
**Version:** 0.1.0  
**Launch market:** Nepal; NPR-only V1 proposed  
**Database:** Supabase PostgreSQL  
**Related documents:** [DATABASE-DESIGN.md](DATABASE-DESIGN.md), [FOUNDATION-SCHEMA.md](FOUNDATION-SCHEMA.md), [FINANCIAL-APPROVAL-SCHEMA.md](FINANCIAL-APPROVAL-SCHEMA.md), [PERMISSION-MATRIX.md](../security/PERMISSION-MATRIX.md)

> This document defines a proposed ledger boundary and entity contract. It is not an approved executable schema, legal accounting opinion, or applied migration. Provider selection, tax/invoice rules, cashier operations, and several folio workflows remain open. Do not create SQL migrations until the open decisions and parent-table contracts are reviewed.

## 1. Goals and non-negotiable invariants

The V1 financial domain must support:
- Guest/reservation/stay folios and property-scoped financial activity.
- Charges from accommodation, restaurant orders, approved services, taxes, fees, and manual adjustments.
- Cash and electronic payment records, partial payments, deposits, and payment-to-folio allocations. Split-tender support remains pending product approval.
- Refund requests and execution with the separately documented Owner approval policy.
- Immutable posted financial history and explicit compensating entries.
- Reliable provider callbacks, idempotency, reconciliation, and auditability.
- Strict organization/property scope and controlled server-side writes.

Invariants:
1. Posted financial records are never silently edited or deleted.
2. Corrections use linked compensating records; the original remains visible.
3. Money is exact decimal data, never floating point.
4. A payment is not considered settled merely because the browser reports success.
5. A payment and a folio allocation are different facts and have separate records.
6. Allocation totals cannot exceed the eligible settled/unallocated payment amount or the amount available on the folio under the final business rules.
7. Every critical mutation is authorized and executed through a controlled transaction boundary.
8. Repeated requests and duplicate provider callbacks must not duplicate money movement or ledger posting.
9. A provider timeout/ambiguous result is not treated as a definitive failure.
10. Financial approval policy is enforced by the backend immediately before execution.

## 2. Financial concepts and boundaries

### 2.1 Folio

A folio is a property-scoped account statement for a guest, reservation, stay, company/house account, or another approved billing party. It records what is owed and credits applied to that account. It is not itself a payment instrument or a bank/cash account.

**V1 product decision — resolved:** use one primary folio per reservation. The corresponding stay uses that same folio; check-in must not create a second stay folio. A guest with multiple reservations has a separate primary folio for each reservation. The guest profile itself does not own a shared folio. Separate room folios, guest-level master folios, company accounts, and house accounts are out of V1. A folio belongs to exactly one property and organization; cross-property sharing is not supported.

### 2.2 Payment

A payment represents a tender or collection attempt and its confirmed financial outcome: cash received, bank transfer verified, or an electronic provider transaction. A payment can be partially allocated, fully allocated, or temporarily unapplied. A payment may have one or more allocations, subject to final deposit and split-tender rules.

### 2.3 Allocation

An allocation applies some amount of a settled payment to a specific folio. It is a separate relationship because one payment may be split across folios and one folio may be paid by multiple payments. Allocation history is immutable; corrections use allocation reversals/compensating rows.

### 2.4 Refund

A refund is a money-out operation against an original payment. It is not merely a negative folio charge. It must reference the original payment and, where relevant, the original allocation(s). Refund approval and execution use [FINANCIAL-APPROVAL-SCHEMA.md](FINANCIAL-APPROVAL-SCHEMA.md).

### 2.5 Invoice

An invoice is a legally relevant document/snapshot issued under the property's fiscal configuration. It must not be treated as a live view that changes when the underlying folio changes. Invoice numbering, tax presentation, cancellation/credit-note rules, and mandatory fields remain jurisdictional decisions.

## 3. Proposed entity map

```text
organizations
  └── properties
        ├── folios
        │     ├── folio_entries (posted, immutable)
        │     ├── invoices (issued document snapshots; fiscal design pending)
        │     └── payment_allocations
        ├── payments
        │     ├── payment_events (provider/cash lifecycle evidence)
        │     ├── payment_allocations
        │     └── refunds
        ├── financial_operation_requests
        ├── financial_approval_decisions
        ├── cashier_sessions (workflow design pending)
        └── audit_logs
```

Every tenant-owned row must carry organization scope. Property-owned rows must carry both organization_id and property_id, with composite foreign keys proving that all referenced records belong to the same organization/property.

## 4. Proposed table contracts

Exact column types, key names, indexes, and FK definitions must be reconciled into a reviewed ERD before migration.

### 4.1 `folios` — account header

Candidate fields:
- `id`, `organization_id`, `property_id`.
- `folio_number`: human-readable number unique within the agreed property/business scope.
- `folio_type`: candidate values `guest`, `reservation`, `stay`, `company`, `house_account`; final V1 enum requires workflow decision.
- `reservation_id`: required and unique within the property, establishing one primary folio per reservation. `stay_id` is a nullable operational link to the active stay created from that reservation; it must not create a second folio. `guest_id` may be retained as a non-authoritative display/reporting reference only if the guest data model requires it; it does not establish folio ownership. No company/house-account link in V1.
- `currency`: NPR for Nepal V1 if NPR-only launch is approved.
- `status`: candidate `open`, `closed`, `void`; closed does not mean immutable history is deleted.
- `opened_at`, `closed_at`, `opened_by`, `closed_by`, `version`, `created_at`, `updated_at`.

Rules:
- Folio scope and currency are fixed at creation; changing either after posting is prohibited.
- A closed folio cannot accept ordinary new charges or allocations. Reopening, late postings, and closed-period corrections require explicit policy and permission design.
- Current balance is derived from posted ledger entries, not maintained as an independently editable number. A cached balance may be introduced only as a transactionally maintained projection with a reconciliation path.
- Enforce one folio per reservation with a unique constraint on the scoped reservation reference. A stay inherits the reservation's folio. Do not create additional guest, room, master, company, or house-account folios in V1. A guest may have multiple independent reservation folios.

### 4.2 `folio_entries` — immutable posted ledger

Candidate fields:
- `id`, `organization_id`, `property_id`, `folio_id`.
- `entry_number`: ordered folio-local sequence or another concurrency-safe reference.
- `entry_type`: controlled catalog such as `accommodation_charge`, `restaurant_charge`, `service_charge`, `tax`, `fee`, `discount`, `payment_allocation`, `adjustment`, `reversal`, `allocation_reversal`.
- `side`: `debit` or `credit`; amount is stored as a positive exact value. Debit increases the amount owed; credit reduces it.
- `amount`: positive exact numeric value; currency is inherited from and must match the folio.
- `description`, `business_date`, `posted_at`, `posted_by`.
- `source_type`, `source_id`: originating reservation/stay/order/service/adjustment/payment-allocation reference, validated by controlled service and relevant FK contracts.
- `reversal_of_entry_id`: nullable self-reference for a compensating reversal.
- `tax_snapshot` or normalized tax-detail child rows: final tax design pending.
- `metadata`: allowlisted non-sensitive snapshot only; no secrets or unrestricted user payload.
- `created_at`.

Ledger semantics:
- Debit = charge/amount owed increases.
- Credit = payment allocation, discount, or approved credit/adjustment reduces amount owed.
- Balance = SUM(debits) - SUM(credits), per folio and currency.
- Never store signed amounts with ambiguous sign conventions.
- Never update/delete a posted entry through ordinary application roles.
- A reversal is a new opposite-side entry linked to the original. Partial reversals must be supported only if the original-entry remaining reversible amount can be proven under a lock.
- A correction to a posted entry is a new linked correction/reversal plus a new correct entry; do not rewrite the original.
- Every entry is posted once through an atomic, idempotent server-controlled operation.
- Draft cart/order values are not ledger entries until a defined posting event occurs.

### 4.3 `payments` — collection/tender record

Candidate fields:
- `id`, `organization_id`, `property_id`.
- `payment_reference`: property-scoped human reference, unique.
- `amount`, `currency`, `payment_method`: candidate controlled values `cash`, `bank_transfer`, `card`, `wallet`, `other`; enabled methods depend on verified provider/operations decisions.
- `status`: candidate lifecycle `initiated`, `pending_confirmation`, `settled`, `failed`, `cancelled`, `partially_refunded`, `refunded`, `reconciliation_required`.
- `provider_code`, `provider_reference`, `merchant_reference`: nullable; store references only, never PAN/CVV, bank login credentials, API secrets, or raw authentication payloads.
- `initiated_at`, `settled_at`, `failed_at`, `created_by`, `created_at`, `updated_at`, `version`.
- `idempotency_key`: unique within the appropriate property/operation scope.

Rules:
- A payment's original amount and currency are immutable after initiation; changes require a new payment or explicit provider-supported adjustment record.
- A payment is allocatable only after its settlement/receipt is verified according to the method-specific rule.
- For cash, the authorized cashier's recorded receipt is the evidence of collection, subject to cashier-session controls. The exact cash workflow is pending.
- For bank transfer, a screenshot or client assertion alone is not sufficient settlement evidence; define an authorized verification process.
- For electronic payments, verify provider signatures and query/reconcile provider state where required.
- Payment status is not a substitute for immutable payment events or audit evidence.
- A payment may be unapplied or partially allocated if deposit/advance-payment workflows are approved.

### 4.4 `payment_events` — append-only lifecycle evidence

Candidate fields:
- `id`, `organization_id`, `property_id`, `payment_id`.
- `event_type`, `source` (`user`, `provider`, `system`), `provider_event_id` where applicable.
- `occurred_at`, `received_at`, `correlation_id`, `safe_payload` (strict allowlist/redaction).
- Unique provider event identity within provider scope to deduplicate webhook retries.

Rules:
- Preserve verified lifecycle evidence; do not let a webhook directly overwrite arbitrary payment fields.
- Validate signature, provider account/tenant mapping, amount, currency, merchant reference, and expected state transition.
- Treat out-of-order and duplicate callbacks safely.
- Keep raw secrets and payment credentials out of event payloads and logs.
- A provider callback is evidence to be validated, not an instruction to trust.

### 4.5 `payment_allocations` — payment-to-folio application

Candidate fields:
- `id`, `organization_id`, `property_id`, `payment_id`, `folio_id`.
- `amount`, `currency`, `status` candidate `posted` only for finalized allocation rows.
- `allocated_at`, `allocated_by`, `idempotency_key`.
- Optional `reverses_allocation_id` for compensating allocation reversal.

Rules:
- Payment and folio must share organization, property, and currency for V1.
- Sum of active allocations must never exceed the verified settled amount less refunds and prior allocation reversals, under a payment-row lock.
- Allocation cannot make a folio credit exceed the applicable policy/available amount unless an approved deposit/credit-balance workflow explicitly allows it.
- Allocation posting and corresponding folio credit entry occur atomically.
- Allocation reversal creates a compensating debit entry on the folio and a linked reversal record; it does not erase the original allocation.
- A retry with the same idempotency key returns the existing allocation result.
- Under the V1 one-primary-folio model, an allocation targets one reservation folio. A single payment is not split across multiple folios in V1. Whether a single folio can be settled using multiple tenders (split tender) remains an explicit open decision; do not infer it from the one-folio choice.

### 4.6 `refunds` — money-out operation

Candidate fields:
- `id`, `organization_id`, `property_id`, `payment_id`.
- `financial_operation_request_id`: FK to the approval/execution request.
- `amount`, `currency`, `reason`, `destination_reference`.
- `status`: candidate `requested`, `pending`, `processing`, `succeeded`, `failed`, `reconciliation_required`, `cancelled`.
- `provider_refund_reference`, `initiated_at`, `completed_at`, `created_by`, `created_at`.

Rules:
- Refund must reference the original payment and cannot exceed its refundable amount after completed and active reserved refunds, chargebacks, and provider constraints.
- Reserve refundable amount before external execution; ambiguous outcomes retain the reservation until reconciled.
- A successful refund is a cash/payment-side outflow and does not automatically create a folio credit. If the guest's folio also needs correction, create a separately authorized linked folio adjustment/reversal so the two accounting effects are explicit.
- Refund execution must follow the Owner approval policy and idempotency rules in FINANCIAL-APPROVAL-SCHEMA.md.
- A refund must not silently reverse or delete the original payment or allocation.

### 4.7 `invoices` — issued document snapshot (fiscal design pending)

Candidate fields:
- `id`, `organization_id`, `property_id`, `folio_id`.
- `invoice_number`, `issued_at`, `issued_by`, `currency`, `subtotal`, `tax_total`, `grand_total`.
- Immutable line/tax snapshot or normalized invoice-line child records.
- `status` candidate `issued`, `voided`, `credited`; exact legal transitions depend on Nepal fiscal requirements.
- Link to the exact folio entries included at issuance.

Rules:
- Issued invoice contents and number are immutable.
- Post-issuance corrections require the legally appropriate credit note, cancellation, or replacement workflow; never edit the issued invoice in place.
- Do not assume that a folio statement is a legally compliant tax invoice.
- Numbering, PAN/VAT fields, tax rates, rounding, credit-note requirements, and retention must be confirmed with qualified local accounting/tax guidance before implementation.

## 5. Balance, settlement, and allocation calculations

For a folio:
`folio_balance = SUM(posted_debit_entries) - SUM(posted_credit_entries)`

For a payment:
`unallocated_settled_amount = verified_settled_amount - successful_refunds - active_refund_reservations - active_allocations + confirmed_allocation_reversals`

The exact formula must be represented by one authoritative backend/database operation and tested under concurrent transactions. Do not trust a browser-calculated balance.

Do not use a mutable `balance` field as the accounting source of truth. If a projection is added for performance, it must be updated in the same transaction as ledger posting and be rebuildable from immutable source rows.

## 6. Transaction and concurrency boundaries

Critical operations require short, atomic database transactions. Never keep a database transaction open while waiting for a payment provider.

### Post a charge
1. Authenticate actor and verify property membership and `folio.post_charge` (or a narrowly scoped domain permission).
2. Validate source event, amount, currency, tax snapshot, folio status, and idempotency key.
3. Lock the folio row and relevant source record in documented global lock order.
4. Check whether the source has already been posted.
5. Insert immutable debit entry and audit/outbox event.
6. Commit.

### Collect and allocate payment
1. Create payment intent/record with idempotency key and authorized method.
2. Verify collection/settlement using method-specific evidence.
3. In a short transaction, lock payment then folio in a consistent order.
4. Revalidate settled amount, refunds, existing allocations, folio status, currency, and actor permissions.
5. Insert allocation and matching folio credit entry atomically; write audit/outbox event.
6. Commit.

### Refund
1. Create a financial operation request and reserve refundable amount under payment lock.
2. Apply threshold and mandatory-exception approval policy.
3. If approval is pending, do not call provider.
4. After valid approval (or when no approval is required), queue idempotent execution.
5. Call provider outside the database transaction.
6. Record verified outcome in a new transaction.
7. Ambiguous provider outcome becomes `reconciliation_required`; do not release the reserved amount.
8. Update payment/refund projections only from validated outcomes.

### Reconciliation
- Reconcile internal payment/refund/allocation state against provider settlement records or authorized cash/bank evidence.
- Differences become explicit reconciliation cases; never silently edit ledger history.
- Reconciliation permissions, reviewer separation, cash over/short handling, and period close require separate cashier/finance workflow design.

## 7. Currency and precision proposal

For a Nepal-only V1, the simplest safe launch proposal is **NPR-only transactions** with exact `numeric(18,2)` amounts and explicit `currency = 'NPR'` on financial records. This is a product proposal, not a statement that every future market uses two decimal places.

Do not enable multi-currency by merely allowing another ISO code. Before enabling it, define:
- Currency-specific minor-unit precision and rounding.
- Whether folio currency is fixed for its lifetime.
- FX source, rate timestamp, quote/booking snapshot, and immutable conversion values.
- Which currency controls thresholds, invoices, provider settlement, and accounting.
- FX gain/loss and refund conversion behavior.

Until those decisions are approved, reject non-NPR payment/folio operations rather than silently converting.

## 8. Authorization and RLS boundary

Use the fixed permission model as the first authorization layer:
- `folio.read` for viewing permitted folios.
- `folio.post_charge` for approved charge posting.
- `folio.adjust` for initiating adjustments, subject to Owner approval policy.
- `payment.collect` for collection initiation/recording.
- `payment.refund` for initiating refunds, subject to Owner approval policy.
- `payment.reconcile` for approved reconciliation workflows.
- `payment.refund.approve` and `folio.adjust.approve` remain Owner-only and separation-of-duties constrained.

RLS must enforce tenant/property visibility, but critical writes must go through narrowly scoped server-controlled database operations. Ordinary browser roles must not directly insert/update/delete posted ledger, allocation, refund, payment-event, or approval rows. Review SECURITY DEFINER functions, grants, search_path, and explicit actor authorization. Do not expose service-role credentials.

## 9. Integration with Financial Approval Schema

The financial approval schema depends on these parent contracts:
- `payments`: stable payment ID, organization/property scope, currency, verified settled amount, and refundable amount derivation.
- `folios`: stable folio ID, scope, currency, open/closed/period state.
- `folio_entries`: immutable posted entry ID, amount, side, original-entry linkage, and correction/reversal semantics.
- `refunds`: stable refund execution record linked to original payment and financial operation request.
- A single transaction boundary for reservation of refundable/adjustable amounts and execution-time revalidation.

Before migration, reconcile the exact refund aggregation formula, correction amount limits, and lock ordering in both documents. The rolling 24-hour folio-adjustment rule and lifetime linked-correction rule remain governed by FINANCIAL-APPROVAL-SCHEMA.md.

## 10. Required test gates

At minimum:
1. Debit/credit balance arithmetic and currency consistency.
2. Posted entry cannot be updated or deleted by application roles.
3. Reversal preserves original entry and cannot exceed reversible amount.
4. Duplicate charge posting with same idempotency key creates one entry.
5. Same source event cannot be posted twice under concurrent requests.
6. Payment cannot be allocated before verified settlement.
7. Concurrent allocations cannot exceed settled/unallocated amount.
8. Duplicate provider callbacks do not duplicate settlement or ledger posting.
9. Invalid provider signature/account/amount/currency is rejected.
10. Out-of-order provider events do not regress a settled payment.
11. Refunds plus active reservations cannot exceed refundable amount.
12. Ambiguous refund outcome retains reservation.
13. Refund does not implicitly create a folio credit.
14. Allocation and corresponding folio credit are atomic.
15. Allocation reversal creates compensating records, not destructive edits.
16. Cross-property and cross-organization references are rejected.
17. Unauthorized roles cannot collect, adjust, refund, reconcile, or view restricted records.
18. Invoice snapshot remains unchanged after later folio activity.
19. Closed folio/period behavior follows approved policy.
20. RLS and server authorization tests cover all permitted and denied roles.

## 11. Decisions required before this design can be approved

1. **Resolved — folio ownership/cardinality:** one primary folio per reservation; its stay reuses the same folio. No separate guest/room/master/company/house folios in V1.
2. **Open — payment allocation:** must V1 support split tender (multiple payment methods for one folio)? A single payment will not be split across multiple folios in V1.
3. **Advance deposits:** Can hotels collect deposits before check-in and leave them unapplied until a folio/stay exists?
4. **Cashier workflow:** Are cashier opening/closing sessions, cash drawer counts, shift handover, over/short, and supervisor reconciliation required in V1?
5. **Payment methods:** Which methods are launch-critical: cash, bank transfer, card terminal, wallet/QR provider, or online payment gateway? Provider names are not selected.
6. **Tax/invoices:** Confirm Nepal invoice/PAN/VAT and fiscal workflow with a qualified local accountant before finalizing schema.
7. **Closed periods:** Define night audit/business date, period closing, late postings, and who can reopen or correct a closed period.
8. **Restaurant posting:** At what exact event does a restaurant order become a folio charge—order accepted, served, or explicitly charged to room?
9. **Refund destination:** Which cash/bank/provider refund flows are supported and how is original destination verified?

Do not infer answers from the current permission matrix. It does not yet define all of these workflows.

## 12. Implementation gate

Before executable schema:
1. Confirm the remaining split-tender and advance-deposit scope; the one-primary-folio-per-reservation decision is already recorded.
2. Confirm cashier, restaurant posting, and closed-period workflows.
3. Obtain qualified local review of Nepal tax/invoice rules.
4. Produce an ERD with exact cardinalities, unique keys, composite FKs, and lock order.
5. Reconcile this design with FINANCIAL-APPROVAL-SCHEMA.md.
6. Define migration sequence, RLS, RPC contracts, and integration tests.
7. Review locally before applying to staging; production requires a separate explicit approval.

**Current status:** Design proposal only. No SQL migration has been created or applied. No Supabase tables or database settings were changed.
