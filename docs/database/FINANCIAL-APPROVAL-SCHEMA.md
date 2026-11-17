# SANDESH HOSPITALITY — V1 Financial Approval Schema

**Status:** Detailed schema proposal — review required before executable migration  
**Version:** 0.1.0  
**Launch market:** Nepal; NPR defaults  
**Database:** Supabase PostgreSQL  
**Related documents:** [DATABASE-DESIGN.md](DATABASE-DESIGN.md), [FOUNDATION-SCHEMA.md](FOUNDATION-SCHEMA.md), [PERMISSION-MATRIX.md](../security/PERMISSION-MATRIX.md)

> This document translates the confirmed V1 financial approval policy into a relational schema and transaction design. It is a design proposal, not an applied migration. The parent financial entities (`payments`, `refunds`, `folios`, and `folio_entries`) still need their own reviewed ledger and accounting design. Do not implement this document as standalone SQL until those parent contracts are finalized.

## 1. Confirmed policy this schema must enforce

- Each property has its own financial thresholds.
- Nepal V1 defaults: refund threshold NPR 25,000; folio adjustment threshold NPR 5,000.
- Threshold approval is required only when the relevant amount is greater than the threshold. Exact equality does not trigger threshold-based approval.
- Only an active Organization Owner may change property thresholds or approve a request.
- Finance Managers may initiate refunds and folio adjustments only within their assigned property. Organization Owners may initiate within their organization.
- The initiator cannot approve their own request. An Owner's request requiring approval needs a different active Owner in the same organization. A sole Owner cannot self-approve and has no Admin/Finance Manager fallback.
- Mandatory approval regardless of amount: cash refunds; refunds to a different destination/payment method; provider/trusted-risk flagged transactions; folio adjustments affecting a closed accounting period; reversals/corrections of previously posted financial transactions.
- Refund aggregation is against the original payment, includes pending and completed related refunds, and may never exceed the refundable amount.
- Folio adjustment aggregation uses a **rolling 24-hour window** per folio, plus a separate lifetime linkage rule for corrections to the same original folio entry.
- Pending and approved-but-not-executed requests reserve the relevant amount. Rejected, cancelled, and expired requests release the reservation. An ambiguous provider/execution outcome must not release a reservation until reconciled.
- Policy changes are prospective. A request retains the policy version and threshold snapshot captured when it was submitted.
- All material actions and outcomes must be auditable.

These are product rules, not claims about an industry standard.

## 2. Proposed entity map

```text
organizations
  └── properties
        ├── property_financial_policies
        ├── financial_operation_requests
        │     ├── financial_approval_decisions
        │     ├── refunds (parent financial model; FK contract pending)
        │     └── folio_entries (parent financial model; FK contract pending)
        └── audit_logs

payments ── financial_operation_requests (refund requests)
folios ──── financial_operation_requests (folio adjustment requests)
folio_entries ── financial_operation_requests (linked correction requests)
```

Every tenant-owned financial row carries `organization_id` and `property_id`. Composite foreign keys must prove that the property belongs to the same organization and that referenced payments, folios, and entries belong to the same organization/property.

## 3. Table proposals

The names and types below are proposed contracts. Exact money precision, parent-table key names, and UUID generation must be reconciled with the final financial ledger design before migration.

### 3.1 `property_financial_policies`

One current policy row per property.

| Column | Proposed type | Requirement |
|---|---|---|
| `id` | uuid | Primary key |
| `organization_id` | uuid | Not null |
| `property_id` | uuid | Not null |
| `operating_currency` | char(3) | ISO currency code; NPR default for Nepal launch |
| `refund_threshold` | numeric | Not null; non-negative |
| `folio_adjustment_threshold` | numeric | Not null; non-negative |
| `policy_version` | integer | Not null; starts at 1, increments on every change |
| `updated_by` | uuid | Active Organization Owner who changed the policy |
| `change_reason` | text | Required for policy changes |
| `created_at`, `updated_at` | timestamptz | Not null |

**Constraints and behavior**

- Unique `(organization_id, property_id)`.
- Composite FK `(organization_id, property_id)` → `properties(organization_id, id)`.
- Only a server-controlled owner-authorized operation may insert/update this row.
- Every change increments `policy_version` and writes an audit event containing the old/new thresholds, currency, actor, timestamp, and reason.
- Requests copy the relevant policy version, currency, and threshold into immutable request snapshots. Do not recalculate an existing request using a newly edited policy.
- Do not allow client table writes or rely on a UI-hidden setting as authorization.

### 3.2 `financial_operation_requests`

A durable record for every initiated refund or folio adjustment, whether or not approval is required. This is deliberately broader than an approvals-only table: it provides one logical operation identity for idempotency, threshold aggregation, approval binding, and execution reconciliation.

| Column | Proposed type | Requirement |
|---|---|---|
| `id` | uuid | Primary key |
| `organization_id`, `property_id` | uuid | Not null; tenant scope |
| `operation_type` | text | `refund` or `folio_adjustment` |
| `initiator_user_id` | uuid | Not null; authenticated actor |
| `idempotency_key` | text | Not null; client retry key, unique in property scope |
| `request_version` | integer | Not null; starts at 1 |
| `supersedes_request_id` | uuid | Nullable; links a replacement request |
| `amount` | numeric | Not null; strictly positive |
| `currency` | char(3) | Not null |
| `payment_id` | uuid | Required for refund; otherwise null |
| `folio_id` | uuid | Required for folio adjustment; otherwise null |
| `original_folio_entry_id` | uuid | Optional; required for a correction/reversal linked to a posted entry |
| `destination_reference` | text | Optional non-sensitive provider token/reference; never PAN, CVV, credentials, or secret |
| `reason` | text | Not null; meaningful user-entered business reason |
| `policy_version_snapshot` | integer | Not null |
| `threshold_snapshot` | numeric | Not null |
| `cumulative_amount_snapshot` | numeric | Not null; server-calculated |
| `rolling_window_started_at` | timestamptz | Not null for folio adjustment calculation |
| `approval_required` | boolean | Not null; server-calculated |
| `approval_reason_codes` | text[] | Not null; server-calculated allowlisted reason codes |
| `approval_state` | text | Check: `not_required`, `pending`, `approved`, `rejected`, `cancelled`, `expired` |
| `execution_state` | text | Check: `not_started`, `queued`, `executing`, `succeeded`, `failed`, `reconciliation_required`, `cancelled`, `blocked` |
| `request_binding_hash` | text | Not null; server-computed binding of material request fields |
| `requested_at` | timestamptz | Not null |
| `expires_at` | timestamptz | Nullable; policy-defined approval expiry |
| `executed_at` | timestamptz | Nullable; set only after confirmed execution |
| `provider_reference` | text | Nullable; provider operation reference |
| `ledger_reference` | uuid | Nullable; resulting refund/folio ledger record |
| `created_at`, `updated_at` | timestamptz | Not null |

**Integrity rules**

- Composite FK `(organization_id, property_id)` → `properties(organization_id, id)`.
- Composite FKs to payment, folio, and folio-entry parents must include organization/property scope. Parent financial schemas are not yet finalized, so exact FK columns are a migration gate.
- Check operation/resource pairing: a refund must reference its original payment and must not reference a target folio as its primary resource; a folio adjustment must reference its folio and must not reference a payment as its primary resource.
- Unique `(organization_id, property_id, idempotency_key)`.
- A replacement request is a new row. Material fields on a submitted request are immutable. A material change cancels/invalidates the old request as appropriate and creates a new request linked by `supersedes_request_id`; never silently edit an approved request.
- The server computes the binding hash from operation type, property, resource references, amount, currency, destination reference, initiator, request version, and policy snapshot. Approval is valid only for that exact binding.
- Store only safe provider references. Do not store payment credentials or secrets.
- All direct INSERT/UPDATE/DELETE privileges for ordinary client roles are revoked. Use narrowly scoped server-controlled transaction operations.

**Why separate approval and execution state?** An approval can be granted while execution is queued, or execution can fail after approval. A single overloaded status would lose that distinction. State transitions must be enforced by the controlled backend/database operation, not by client-supplied status values.

### 3.3 `financial_approval_decisions`

An immutable final decision for a request that requires Owner approval.

| Column | Proposed type | Requirement |
|---|---|---|
| `id` | uuid | Primary key |
| `organization_id`, `property_id` | uuid | Not null |
| `request_id` | uuid | Not null |
| `approver_user_id` | uuid | Not null |
| `decision` | text | `approved` or `rejected` |
| `reason` | text | Required; rejection reason mandatory |
| `request_version` | integer | Not null; must match request |
| `request_binding_hash` | text | Not null; must match request |
| `decided_at` | timestamptz | Not null |

**Constraints and behavior**

- Composite FK `(organization_id, property_id, request_id)` → the corresponding unique scope key on `financial_operation_requests`.
- Unique `request_id`: one final decision per immutable request. A rejected or expired request is not reopened; submit a new request.
- Approver must be an active Organization Owner in the same organization at decision time.
- Approver must differ from initiator.
- Approval/rejection is performed through a controlled operation that revalidates membership, role, request state, binding hash, and expiry immediately before writing the decision.
- Decision rows are append-only. No ordinary role can update or delete them.
- A database FK/check cannot by itself prove the approver's current role or that approver differs from the initiator; enforce these rules in the controlled transaction and cover them with database integration tests.

### 3.4 `audit_logs`

A shared append-only audit table for sensitive administrative and business actions, including financial policy and operation events.

| Column | Proposed type | Requirement |
|---|---|---|
| `id` | uuid | Primary key |
| `organization_id` | uuid | Not null for tenant events |
| `property_id` | uuid | Nullable for organization-level events |
| `actor_user_id` | uuid | Nullable for system/provider events; preserve actor snapshot/reference |
| `actor_type` | text | `user`, `system`, or `provider` |
| `action` | text | Stable event code |
| `resource_type` | text | Stable resource category |
| `resource_id` | uuid | Resource identifier |
| `outcome` | text | `succeeded`, `denied`, or `failed` |
| `correlation_id` | uuid | Links request, execution, provider callback, and logs |
| `metadata` | jsonb | Allowlisted non-secret details only |
| `occurred_at` | timestamptz | Not null |

**Constraints and behavior**

- Composite property FK when `property_id` is present; organization FK always applies.
- Index by `(organization_id, occurred_at desc)`, `(organization_id, property_id, occurred_at desc)`, and resource/correlation lookup paths justified by actual queries.
- No UPDATE or DELETE privileges for application roles. Retention/archival is a separately governed platform process.
- Never put passwords, tokens, payment credentials, full sensitive guest data, or provider secrets in metadata.
- Record at least: policy changed; operation requested; approval granted/rejected; operation queued/executing; execution succeeded/failed; reconciliation required/resolved; operation cancelled/expired; authorization denied.
- Audit writes must be part of the same database transaction as the state change they describe where possible. Provider outcomes are recorded after verified callbacks/reconciliation.

## 4. Rolling 24-hour folio adjustment calculation

For a request against a folio at time `T`, calculate:

```text
rolling_window_start = T - 24 hours

cumulative =
  SUM(ABS(amount)) for successful folio-adjustment requests
    whose confirmed executed_at is within [rolling_window_start, T]
  +
  SUM(ABS(amount)) for active reserved requests on the same folio
    whose approval_state is pending or approved
    and whose execution is not yet conclusively completed
```

The current request is included exactly once in the calculation. The operation-request row is the logical unit of aggregation; do not also sum its resulting folio ledger entry, or the same adjustment will be double-counted.

**Include**
- Successful adjustments with `executed_at >= T - interval '24 hours'`.
- Pending approval requests, even if they are older than 24 hours, while they remain active and reserve an amount.
- Approved but not yet executed/settled requests.
- Requests in `reconciliation_required` until the outcome is resolved, because the system cannot safely assume that no adjustment occurred.

**Exclude**
- Rejected, cancelled, and expired requests.
- Definitively failed operations where reconciliation confirms no ledger posting occurred.
- Successful adjustments older than the rolling 24-hour window, except for the separate original-entry correction aggregation below.

**Decision**
- If cumulative amount is greater than the request's threshold snapshot, Owner approval is required.
- If cumulative amount is exactly equal to or below the threshold snapshot, no threshold-based approval is required.
- Mandatory exception rules override the threshold result at every amount.

**Concurrency requirement**
- Serialize all adjustment requests for the same folio by locking the authoritative folio row (or a dedicated per-folio lock row) inside a database transaction.
- Under that lock, re-read policy/request/ledger state, calculate the cumulative amount, reserve the current amount, and persist the operation request before releasing the lock.
- The same controlled transaction must be used by every adjustment path, including API routes, server actions, and automation.
- At execution, lock the folio again and revalidate the request binding, reservation, approval, period state, and ledger balance. If the current total now requires approval and the request was initially marked `not_required`, atomically change its approval state to `pending`, keep execution blocked, and require an independent Owner decision before queuing execution.
- Use a consistent lock order across all financial functions to reduce deadlocks. Final lock order must be documented alongside the parent ledger design.

### 4.1 Corrections linked to the same original folio entry

The rolling window does not replace the separate lifetime anti-splitting control.

- All correction/reversal requests linked to the same original posted `folio_entry` must be linked by `original_folio_entry_id` and aggregated across the full lifecycle of that original entry, even when requests fall outside the rolling 24-hour window.
- A reversal/correction of a previously posted financial transaction is already a mandatory-approval exception regardless of amount.
- Enforce the original-entry relationship and prevent cumulative correction/reversal amounts from exceeding the correctable amount under the final ledger semantics.
- Do not permit a correction to mutate or delete the original posted entry.

## 5. Refund aggregation

Refund requests are aggregated against the original payment, not merely by guest, folio, or date.

- Include completed refunds and all active reserved refund requests against the same original payment.
- Rejected, cancelled, and expired requests release their reservation.
- An ambiguous provider outcome remains reserved until reconciled.
- The sum must never exceed the original payment's refundable amount after prior refunds, chargebacks, and applicable provider constraints.
- Threshold evaluation uses the request's property policy snapshot and the aggregate related refund amount.
- Cash refunds, different-destination refunds, and trusted risk-flagged refunds always require Owner approval.
- Lock the original payment/refund-balance resource during calculation and reservation so concurrent refund requests cannot over-refund or bypass the threshold.

## 6. Request and execution lifecycle

Approval and execution are separate state dimensions.

### Approval state

| State | Meaning | Allowed next states |
|---|---|---|
| `not_required` | No approval required under the initial threshold/exception calculation | `pending` only if execution-time revalidation finds that a newly reserved/cumulative amount or changed business state now requires approval; otherwise remains not required |
| `pending` | Waiting for an eligible independent Owner | `approved`, `rejected`, `cancelled`, `expired` |
| `approved` | Valid Owner decision recorded for exact request binding | Terminal; material change requires new request |
| `rejected` | Owner rejected the request | Terminal |
| `cancelled` | Initiator/system cancelled before execution | Terminal |
| `expired` | Approval window expired before decision | Terminal |

### Execution state

| State | Meaning | Allowed next states |
|---|---|---|
| `not_started` | No execution has been queued | `queued`, `blocked`, `cancelled` |
| `queued` | Eligible operation is ready for controlled execution | `executing`, `blocked`, `cancelled` |
| `executing` | Worker/provider operation has started | `succeeded`, `failed`, `reconciliation_required` |
| `succeeded` | Provider/ledger outcome is confirmed | Terminal |
| `failed` | Definitive failure; no financial effect occurred | Terminal or new request |
| `reconciliation_required` | Provider outcome is ambiguous; reservation remains held | `succeeded` or `failed` after reconciliation |
| `blocked` | Execution stopped by a validation/approval/business rule | May return to `queued` only after controlled revalidation |
| `cancelled` | Execution was cancelled before external/ledger effect | Terminal |

Transitions must be enforced server-side. A client must never directly set either state. A retry of the same logical request reuses the idempotency key and returns the existing request/result; a materially different operation gets a new request.

## 7. Transaction boundaries and external providers

Do not keep a PostgreSQL transaction open while waiting for an external payment provider.

1. In a short database transaction, authenticate and authorize the actor; lock the payment or folio; validate policy and current state; calculate cumulative amounts; create/reserve the operation request; write audit/outbox records; commit.
2. If approval is pending, stop. No provider or ledger effect occurs.
3. When approval is recorded (or approval is not required), enqueue the operation through an idempotent backend worker.
4. In a short transaction, revalidate authorization/business state and mark execution as started with a stable idempotency reference.
5. Call the provider outside the database transaction when needed.
6. In a new transaction, verify the provider result, post the immutable ledger/refund record, mark success, release/update reservations, and write audit events.
7. If provider outcome is uncertain, mark `reconciliation_required`; do not blindly retry with a new key or release the reservation.
8. Reconciliation resolves the provider reference and records a definitive success/failure.

For a non-provider folio adjustment, the posting of the immutable folio entry and the request's successful execution state should be committed atomically in one database transaction.

## 8. RLS, privileges, and authorization boundaries

- Enable RLS on all tenant-specific tables exposed through Supabase Data API.
- Revoke direct INSERT/UPDATE/DELETE from ordinary authenticated client roles for policies, operation requests, decisions, financial ledger rows, and audit logs.
- Permit only appropriately scoped reads. Read access must use active organization/property membership and the relevant permission; sensitive approval details are not automatically visible to every staff member.
- Create requests and execute operations through narrowly scoped server-controlled functions/endpoints that derive the actor from a verified session. Never trust client-supplied actor, organization, property, role, threshold, approval state, or cumulative total.
- A privileged service-role backend bypasses RLS; it must independently enforce authentication, tenant scope, permissions, active membership, Owner approval rules, and resource ownership. Never expose the service-role key to the browser.
- Prefer SECURITY INVOKER. If SECURITY DEFINER is unavoidable, document the reason, set a fixed safe `search_path`, revoke PUBLIC execution, grant only to the intended caller, and perform explicit authorization checks.
- Recheck active membership/Owner role at decision time and immediately before execution. Do not rely on stale JWT custom claims.
- Organization Admin, Cashier, Front Desk, and other operational roles cannot initiate or approve these operations by default.
- Automation must call the same controlled financial boundary and cannot bypass approvals or invoke arbitrary SQL.

## 9. Indexes and uniqueness candidates

Add only after validating the final query plan and parent-table keys:

- `property_financial_policies(organization_id, property_id)` unique.
- `financial_operation_requests(organization_id, property_id, idempotency_key)` unique.
- `financial_operation_requests(organization_id, property_id, folio_id, executed_at)` for rolling-window aggregation, with a partial predicate for folio adjustments and successful execution where supported.
- `financial_operation_requests(organization_id, property_id, payment_id, executed_at)` for refund aggregation.
- `financial_operation_requests(organization_id, property_id, original_folio_entry_id)` for lifetime correction aggregation.
- `financial_operation_requests(approval_state, execution_state, requested_at)` for worker/reconciliation queues, scoped as appropriate.
- `financial_approval_decisions(request_id)` unique.
- `audit_logs(organization_id, occurred_at desc)`.
- `audit_logs(organization_id, property_id, occurred_at desc)`.
- `audit_logs(correlation_id)` where correlation lookup is required.

Partial-index predicates must match the actual final status model. Avoid adding speculative indexes before representative EXPLAIN/ANALYZE tests.

## 10. Required database and integration tests

### Policy and threshold
1. NPR 25,000 refund and NPR 5,000 adjustment exactly at threshold do not require threshold-based approval.
2. One minor currency unit above threshold requires approval.
3. A property cannot read or change another property's policy.
4. Only an active Owner can change policy; Admin and all property roles are denied.
5. Policy version increments on change and a pending request retains its original snapshot.

### Rolling 24-hour aggregation
6. Several individually small adjustments whose absolute sum crosses NPR 5,000 require approval.
7. A successful adjustment executed just inside 24 hours is included.
8. A successful adjustment executed just outside 24 hours is excluded from the rolling sum.
9. Pending and approved-but-unexecuted requests are included, including those older than 24 hours while still active.
10. Rejected, cancelled, and expired requests are excluded and release reservations.
11. An ambiguous provider/execution result remains reserved until reconciliation.
12. Negative adjustment amounts are aggregated by absolute value.
13. The current request is counted exactly once; linked ledger entries do not double-count it.
14. Concurrent requests against the same folio serialize; neither can bypass the threshold.
15. A request that becomes approval-required on execution revalidation cannot execute without a valid Owner decision.
16. Corrections linked to the same original folio entry aggregate across the full lifecycle, outside the rolling window too.

### Refund aggregation and execution
17. Multiple pending/completed refunds against one payment are aggregated.
18. Cumulative refunds cannot exceed the refundable amount.
19. Cash, different-destination, and risk-flagged refunds require approval below threshold.
20. Concurrent refund requests cannot over-refund or bypass approval.
21. Duplicate idempotency keys and provider callbacks do not create duplicate financial effects.
22. Ambiguous provider outcomes are reconciled before reservation release.

### Authorization and audit
23. Finance Manager can initiate only within assigned property scope.
24. Only an active Owner in the same organization can approve.
25. Self-approval, cross-organization Owner approval, inactive Owner approval, and Admin approval are denied.
26. A sole Owner's high-risk self-initiated request remains blocked.
27. Material request changes invalidate the old approval binding.
28. Unauthorized direct table writes are denied.
29. Cross-tenant and cross-property references are rejected by composite FKs.
30. Every state change and denied sensitive action produces the required audit evidence without leaking secrets.

Use pgTAP or equivalent PostgreSQL tests plus application/provider integration tests. Test with distinct synthetic organizations, properties, Owners, Finance Managers, Admins, and operational users.

## 11. Unresolved dependencies and explicit non-goals

This design does not settle the following:

- Final `payments`, `refunds`, `folios`, and `folio_entries` table structures and their exact composite key names.
- Final accounting semantics for debit/credit sign conventions, refundable balance, folio balance, and correction limits.
- Currency minor-unit/precision and rounding rules. NPR is the Nepal V1 default; non-NPR transactions must remain disabled until immutable FX snapshot and rounding rules are specified.
- Approval expiry duration and cancellation policy.
- Whether provider-specific refund destinations can be safely represented by a non-sensitive reference.
- Audit retention/archival and legal retention requirements.
- Exact worker/outbox implementation and provider reconciliation schedule.
- Whether an Owner can be temporarily suspended while a decision is pending; decision-time and execution-time checks must deny inactive authority.

Not included: production SQL, applied migrations, live payment provider calls, live refund execution, or enabling non-NPR transactions.

## 12. Implementation gate

Before writing SQL:

1. Approve the final parent financial ledger model and exact composite keys.
2. Review this schema with the permission matrix and confirmed Owner-only policy.
3. Decide currency precision, rounding, and non-NPR enablement rules.
4. Decide approval expiry and reconciliation behavior.
5. Produce an ERD and transaction/lock-order diagram.
6. Write pgTAP tests for constraints, RLS, role checks, rolling-window aggregation, and concurrency.
7. Implement in a disposable/local Supabase database first.
8. Review migration diff, privileges, RLS, SECURITY DEFINER functions, and rollback/recovery plan before any shared environment.

**Current status:** Design proposal only. No Supabase tables or migrations have been created or applied.
