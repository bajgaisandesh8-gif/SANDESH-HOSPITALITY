# SANDESH HOSPITALITY
## Automation Engine

**Document status:** Initial conceptual baseline — for technical review  
**Version:** 0.1.0  
**Scope:** Tenant-facing workflow automation for the V1 web-first platform

> This document describes the proposed automation product and runtime. It is not an implemented engine or final execution contract. Event names, action schemas, limits, and approval rules must be finalized before implementation.

**Controlling V1 reference:** [V1 Product and Architecture Baseline](../planning/V1-PRODUCT-AND-ARCHITECTURE-BASELINE.md). Use its release boundary and defaults where this earlier proposal is broader or still marks a decision open. External validation gates remain mandatory; this file is a design proposal, not proof of implementation.

---

## 1. Purpose and product goals

Automation is a core product capability of SANDESH HOSPITALITY. It should help hotel teams reduce repetitive work while keeping business rules, permissions, and critical transactions under the control of the platform.

The V1 automation capability must provide two experiences:

1. **Ready-made automation templates** that a hotel can enable and configure.
2. **A constrained visual workflow builder** for authorized users to create workflows from approved triggers, conditions, branches, delays, and actions.

The system must be understandable to hotel operators, safe by default, tenant-isolated, auditable, and resilient when background processing is delayed or unavailable.

Automation is supplemental. Core booking, front desk, payment, and restaurant operations must continue to function if the automation runtime is unavailable.

## 2. V1 scope and non-goals

### In scope
- Event-based and scheduled triggers.
- Approved conditions and branching.
- AND/OR condition groups.
- Delays and scheduled continuation.
- A closed catalog of approved actions.
- Ready-made workflow templates.
- Visual workflow authoring.
- Draft and published workflow versions.
- Validation and test mode.
- Enable/disable controls.
- Execution history and step-level status.
- Retry handling, idempotency, and dead-letter investigation.
- Tenant and property scoping.
- Permission checks at authoring and execution time.

### Not in scope for V1
- Arbitrary JavaScript or Python execution.
- Shell commands or arbitrary code execution.
- Direct SQL access from tenant workflows.
- Unrestricted outbound HTTP requests.
- Tenant-controlled infrastructure or worker deployment.
- Unrestricted workflow marketplace or public sharing.
- Automation as a replacement for core application business logic.
- Offline workflow execution or offline sync.

## 3. Architecture overview

Proposed flow:

    Core business operation
            |
            v
    PostgreSQL transaction
      |                 |
      | business data   | event_outbox row
      v                 v
    Commit             Outbox publisher
                           |
                           v
                    Event validation
                           |
                           v
                 Workflow trigger matcher
                           |
                           v
                 Workflow execution queue
                           |
                           v
                 Isolated workflow runner
                           |
                           v
              Approved action dispatcher
                           |
                           v
              Scoped application services
                           |
                           v
                 Execution audit/history

The event outbox row must be written in the same database transaction as the business change that produced the event. This prevents the application from committing a business operation while silently losing its corresponding automation event.

The runner should call approved application services or narrowly scoped internal APIs. It must not receive unrestricted database credentials.

## 4. Core design principles

- Business rules remain in core backend services.
- Workflows orchestrate approved business capabilities; they do not redefine them.
- Workflow definitions are versioned JSON/DSL, not executable source code.
- Every trigger and action is tenant/property scoped.
- Authorization is checked when a workflow is configured and again when it executes.
- Published workflow versions are immutable.
- Event delivery is at least once; consumers must be idempotent.
- Automation failures must not roll back already-committed core business operations.
- Sensitive actions require explicit permission and may require approval.
- Test mode must not create real-world side effects.
- Every execution must be traceable to its trigger, workflow version, tenant, property, and outcome.

## 5. Trigger catalog

The initial trigger catalog is illustrative and must be mapped to real domain events.

### 5.1 Business-event triggers

Candidate events:
- reservation.created
- reservation.confirmed
- reservation.modified
- reservation.cancelled
- reservation.checked_in
- reservation.checked_out
- reservation.no_show
- payment.received
- payment.failed
- folio.balance_due
- restaurant.order.created
- restaurant.order.accepted
- restaurant.order.ready
- housekeeping.task.created
- housekeeping.task.completed
- maintenance.request.created
- guest.feedback.received

Each event needs:
- Stable event type.
- Schema version.
- Event ID.
- Occurrence timestamp.
- Organization and optional property scope.
- Aggregate/resource type and identifier.
- Minimal, non-sensitive payload required for matching.
- Correlation and causation identifiers where appropriate.

Do not include unnecessary guest personal information in event payloads. The runner should retrieve only the minimum additional data needed through authorized services.

### 5.2 Scheduled triggers

Candidate schedules:
- Daily operational summary.
- Upcoming-arrival reminder scan.
- Upcoming-departure reminder scan.
- Unpaid folio reminder scan.
- Housekeeping task review.
- Periodic maintenance follow-up.

Scheduled triggers must use the property's configured time zone and explicit local business-time rules. Define behavior for daylight-saving transitions and missed schedule windows.

## 6. Condition and branching model

The visual builder should support a limited, typed condition system.

Candidate operators:
- equals / not equals
- is set / is not set
- greater than / less than
- contains / does not contain, only for approved text or collection fields
- date before / date after
- within a defined time window
- AND / OR groups

Rules:
- Only expose approved fields from a typed event/context schema.
- Validate field types and operator compatibility.
- Bound nesting depth and condition count.
- Do not permit raw expressions, arbitrary code, or dynamic SQL.
- Handle missing/null values explicitly.
- Make timezone behavior explicit for date/time comparisons.
- Revalidate workflow definitions on publish and execution.

Branches must be deterministic for the same event and workflow version, subject to explicitly documented time-dependent conditions.

## 7. Delay and scheduling behavior

The builder may support:
- Wait for a fixed duration.
- Wait until a specified local date/time.
- Continue at a configured offset before/after an event date, where supported.

Requirements:
- Persist pending continuation state; do not keep a worker process sleeping.
- Use durable scheduling records or a managed queue with durable delayed jobs.
- Re-check workflow enabled state, version rules, tenant scope, and permissions when resuming.
- Define what happens when a reservation is cancelled or changed during a delay.
- Prevent duplicate continuation after worker retries.
- Enforce maximum delay horizon and pending-job limits.
- Define behavior for timezone changes and daylight-saving transitions.

## 8. Approved action catalog

Actions are closed, typed capabilities implemented by the platform.

Candidate V1 actions:
- Send approved booking confirmation.
- Send pre-arrival reminder.
- Send check-in notification.
- Create housekeeping task after checkout.
- Notify restaurant staff about a new order.
- Send payment reminder.
- Assign a maintenance request using approved assignment rules.
- Generate or distribute a daily report to authorized recipients.
- Send guest feedback request.
- Create an internal staff notification.

Each action must define:
- Stable action type and schema version.
- Required input fields and validation.
- Required permission.
- Tenant/property scope rules.
- Whether it causes an external side effect.
- Idempotency strategy.
- Retryability and timeout behavior.
- Audit fields.
- Whether approval is required.
- Test-mode behavior.

Actions that send messages must use approved templates and validated recipients. Financial actions such as charging, refunding, or adjusting a folio should not be available as unrestricted workflow actions in the initial builder. If introduced later, they require a separate risk review and explicit approval model.

## 9. Workflow lifecycle and versioning

Candidate workflow states:
- Draft
- Published
- Disabled
- Archived

Rules:
- Draft changes do not affect currently published executions.
- Publishing creates an immutable version snapshot.
- An execution references the exact published version it started with.
- Editing a published workflow creates a new draft/version.
- Disabling a workflow prevents new executions and defines how in-flight executions are handled.
- Archiving prevents future activation but preserves history.
- Publishing requires validation and appropriate permission.
- Workflow ownership and scope cannot be changed through untrusted client payloads.

Open decision: whether a workflow execution already in progress continues with its original version after a newer version is published. Proposed default: yes, unless explicitly cancelled or disabled under documented policy.

## 10. Suggested workflow definition structure

Illustrative conceptual JSON shape only; not a finalized API contract:

    {
      "schema_version": 1,
      "trigger": {
        "type": "reservation.confirmed",
        "version": 1
      },
      "nodes": [
        {
          "id": "condition-1",
          "type": "condition",
          "config": {
            "field": "reservation.channel",
            "operator": "equals",
            "value": "direct"
          }
        },
        {
          "id": "action-1",
          "type": "action",
          "action": "notification.send_booking_confirmation"
        }
      ],
      "edges": [
        { "from": "condition-1", "to": "action-1", "branch": "true" }
      ]
    }

The real schema must define node/edge validation, supported types, field catalogs, maximum graph size, cycle prevention, and compatibility rules. The example omits sensitive values and is not executable as written.

## 11. Event outbox and delivery guarantees

V1 uses a transactional event outbox.

Requirements:
- Insert the outbox event in the same transaction as the business change.
- Give each event a stable unique ID.
- Include event schema version and tenant/property scope.
- Publish asynchronously after commit.
- Expect at-least-once delivery.
- Make consumers idempotent.
- Track attempts, next retry time, last error category, and processing status.
- Use bounded exponential backoff with jitter.
- Move exhausted events to a dead-letter state for investigation.
- Support safe replay with original event identity and explicit replay audit.
- Define event retention and payload minimization.
- Partition or index operational tables based on measured volume.

The outbox is not the V2 offline sync queue. Do not mix the two concepts or their guarantees.

## 12. Execution engine and runtime

The platform-owned engine is responsible for:
1. Receiving a validated event or schedule occurrence.
2. Finding enabled workflows matching the trigger and scope.
3. Pinning each execution to a published workflow version.
4. Validating the workflow graph and execution limits.
5. Evaluating conditions and branches.
6. Scheduling delays durably.
7. Dispatching approved actions.
8. Recording step outcomes and audit metadata.
9. Retrying eligible failures safely.
10. Finalizing execution state.

Runtime constraints:
- Maximum nodes/steps per workflow.
- Maximum execution duration.
- Maximum concurrent executions per tenant/property.
- Maximum pending delayed jobs.
- Per-tenant action and notification quotas.
- Bounded payload size.
- Bounded retry count and retry horizon.
- Approved outbound destinations only.
- No arbitrary code or shell execution.
- No unrestricted network access from the runner.

The initial runner may be implemented as a platform-owned backend worker. Exact hosting and queue technology remain an architecture decision.

## 13. Idempotency, retries, and failure handling

- Every event has a stable event ID.
- Every action execution has a stable idempotency key derived from execution, node, and relevant attempt semantics.
- External provider calls must use provider idempotency features when available.
- Distinguish retryable failures from permanent validation/authorization failures.
- Do not retry permission denials or invalid workflow definitions indefinitely.
- Use bounded exponential backoff with jitter.
- Record sanitized error categories and correlation IDs.
- Dead-letter exhausted events/executions for authorized review.
- Replay must be explicit, permission-checked, and audited.
- Replaying an event must not duplicate completed external side effects.

At-least-once processing is the expected delivery model. Exactly-once external side effects cannot be assumed.

## 14. Approval and sensitive actions

The initial action catalog should avoid high-risk financial mutations.

If approval-gated actions are added:
- Define the action types requiring approval.
- Require an authorized approver distinct from the workflow author where appropriate.
- Bind approval to the exact action parameters and workflow version.
- Expire stale approvals.
- Revalidate permissions and business state at approval and execution time.
- Record approver, actor, timestamp, decision, and reason.
- Do not let a workflow edit silently change an already-approved action.

## 15. Test mode and validation

Test mode must be safe and non-destructive.

It should:
- Validate the workflow graph and schemas.
- Run against sample or sanitized event payloads.
- Show trigger match, condition results, selected branches, and planned actions.
- Stub external notifications and provider calls.
- Never charge/refund money or mutate production reservations, folios, inventory, or housekeeping records.
- Clearly label simulated results.
- Avoid using live guest data unless specifically authorized and minimized.

Publishing checks should reject invalid graph structure, unsupported schema versions, unauthorized actions, out-of-scope references, unsafe configuration, and workflows exceeding limits.

## 16. Execution history and operator controls

The UI should show:
- Workflow name and state.
- Published version.
- Trigger event or schedule.
- Started and completed timestamps.
- Overall status.
- Step-by-step status.
- Retry count.
- Sanitized failure reason.
- Correlation/event identifier.
- Replay availability where permitted.

Operator controls:
- Enable/disable workflow.
- View draft and published versions.
- Validate/test draft.
- Publish with permission.
- Inspect execution details.
- Retry or replay eligible failed executions with explicit confirmation and audit.
- Cancel eligible pending/delayed executions.

Execution history must be tenant/property scoped and must redact secrets and unnecessary personal information.

## 17. Tenant isolation and authorization

- Every workflow, version, execution, step, and related event must have organization scope and property scope where applicable.
- A user may view or edit only workflows within their authorized scope.
- Publishing requires a dedicated permission.
- Execution must re-check the workflow's current scope and required action permissions.
- A workflow cannot gain privileges from the identity of the worker.
- The runner must use narrowly scoped service access and trusted internal APIs.
- Never expose service-role credentials to browser clients or tenant-authored workflows.
- Platform-level integrations must be separated from tenant workflow execution.

## 18. n8n and external workflow tools

n8n is not the tenant workflow runtime in this design.

Possible limited use:
- Platform-owned integration operations.
- Fixed, signed, narrowly scoped calls to approved platform services.
- Operational workflows not directly exposed as arbitrary tenant automation.

Before any embedded, hosted, or redistributed n8n use, review its current license and commercial terms. A shared n8n instance must not be treated as tenant-isolated merely because each workflow has a tenant ID.

## 19. Observability and operations

Monitor:
- Outbox backlog age and size.
- Event publish latency and failure rate.
- Workflow queue depth.
- Execution duration and failure rate.
- Retry and dead-letter volume.
- Delayed job backlog.
- Per-tenant usage and quota consumption.
- Provider rate limits and delivery outcomes.

Use correlation IDs across business transaction, outbox event, workflow execution, and action dispatch. Alert on sustained backlog, repeated failures, runaway workflows, and unusual tenant usage.

## 20. Initial ready-made template catalog

| Template | Trigger | Example action |
|---|---|---|
| Booking confirmation | Reservation confirmed | Send booking confirmation |
| Pre-arrival reminder | Scheduled before arrival | Send arrival information |
| Check-in notification | Guest checked in | Notify configured staff/guest |
| Checkout housekeeping | Guest checked out | Create room-cleaning task |
| Restaurant order alert | Order created | Notify restaurant staff |
| Payment reminder | Balance due / scheduled check | Send approved reminder |
| Maintenance assignment | Request created | Assign using configured rules |
| Daily operations report | Property-local schedule | Generate authorized report |
| Feedback request | Checkout completed | Send feedback request |

Templates must be configurable, permission-checked, localized where supported, and disabled until explicitly enabled by an authorized user. Template defaults must not silently activate external messaging.

## 21. Open decisions

1. Exact V1 trigger and event catalog.
2. Exact action catalog and required permissions.
3. Workflow graph schema and supported node types.
4. Visual builder library and accessibility requirements.
5. Queue and worker hosting architecture.
6. Execution and retry limits.
7. Schedule timezone and daylight-saving behavior.
8. Handling of in-flight executions after workflow disable/publish.
9. Approval requirements for sensitive actions.
10. Tenant quotas and usage/billing model.
11. Event and execution retention.
12. Localization and message-template ownership.
13. n8n integration boundary and license review.

## 22. Implementation sequence

1. Approve the event catalog and business-event ownership.
2. Define the event envelope and schema-version policy.
3. Define the outbox table, publisher, and idempotent consumer contract.
4. Define the workflow DSL and graph validator.
5. Approve the initial trigger/action catalog and permission mapping.
6. Implement a minimal worker and execution state machine.
7. Add test mode and execution history.
8. Add templates before exposing the visual builder broadly.
9. Add visual authoring, publishing, and operator controls.
10. Test retries, duplicate delivery, tenant isolation, failure recovery, and quotas.

**Status:** Initial conceptual baseline — not an implemented engine and not approved as a production execution contract.
