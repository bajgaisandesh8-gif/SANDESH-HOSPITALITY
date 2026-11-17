# Documentation Index

**Project:** SANDESH HOSPITALITY  
**Status:** Planning baseline — documents require review and approval  
**Rule:** A documented proposal is not an implemented or approved production behavior.

## Product

- [Product Requirements](product/PRODUCT-REQUIREMENTS.md) — product scope, V1/V2 direction, modules, and goals.

## Architecture and data

- [System Architecture](architecture/SYSTEM-ARCHITECTURE.md) — application boundaries, Supabase trust boundaries, tenancy, events, deployment, and operations.
- [Database Design](database/DATABASE-DESIGN.md) — conceptual entities, integrity, RLS, indexing, and migration principles.
- [Entity Relationship Diagram](database/ERD.md) — conceptual domain relationships; not a final schema.
- [API Design](api/API-DESIGN.md) — candidate endpoints, request/response principles, authorization, and error handling.
- [Architecture Decision Records](architecture/ARCHITECTURE-DECISIONS.md) — rationale and status of foundational architecture choices.

## Security and quality

- [Security Architecture](security/SECURITY-ARCHITECTURE.md) — trust boundaries, access control, secrets, guest data, audit, and incident response.
- [Permission Matrix](security/PERMISSION-MATRIX.md) — proposed roles and module permissions; not yet approved.
- [Testing and Release Strategy](quality/TESTING-AND-RELEASE-STRATEGY.md) — test layers, critical invariants, CI, release gates, and recovery expectations.
- [Agent Instructions](../AGENTS.md) — repository-wide rules for AI agents and contributors.

## Workflows and automation

- [Reservation Lifecycle](workflows/RESERVATION-LIFECYCLE.md) — reservation through post-stay lifecycle.
- [Front Desk Workflows](workflows/FRONT-DESK-WORKFLOWS.md) — arrivals, walk-ins, room assignment, in-house service, checkout, handover, and outage handling.
- [Automation Engine](automation/AUTOMATION-ENGINE.md) — templates, visual builder, event outbox, execution, idempotency, retries, and safety boundaries.

## Planning and decisions

- [V1 Product and Architecture Baseline](planning/V1-PRODUCT-AND-ARCHITECTURE-BASELINE.md) — consolidated V1 scope, conservative defaults, implementation order, and external validation gates. This is the primary planning reference; external legal/provider gates remain open.

- [Open Decisions](planning/OPEN-DECISIONS.md) — unresolved product, market, legal, operations, and architecture decisions.
- [Implementation Roadmap](planning/IMPLEMENTATION-ROADMAP.md) — phased delivery sequence and entry/exit criteria.
- [Contributing](../CONTRIBUTING.md) — branch, pull-request, and engineering workflow.

## Readiness before feature coding

The consolidated V1 baseline supersedes older generic suggestions where they conflict. The first coding slice is limited to identity and organization/property context. Reservation/finance implementation remains gated by the specific approvals and evidence listed in the baseline.


The following are still required:
1. Review and approve V1 scope and the first release boundary.
2. Resolve decisions that block reservations, inventory, guest data, payments, and financial records.
3. Validate the proposed permission model and tenant/property access rules.
4. Refine the conceptual ERD into an implementable schema and migration plan.
5. Define reservation/inventory transaction invariants and acceptance tests.
6. Define the first vertical slice and its explicit non-goals.
7. Configure isolated local/test/staging environments before using real integrations or guest data.

## Suggested implementation sequence

1. Repository and environment hygiene; CI baseline.
2. Authentication, organization/property membership, and authorization foundation.
3. Property setup, room types, rooms, and operational statuses.
4. Availability and reservation creation with concurrency-safe inventory.
5. Reservation lifecycle and front-desk arrival/check-in.
6. Stay folio and payment integration only after jurisdiction, tax, and gateway decisions.
7. Checkout and housekeeping task handoff.
8. Reporting and automation incrementally, with tests and audit coverage.

This sequence is a proposal, not a commitment to implement every listed module in V1. Do not start a feature whose critical business rules remain unresolved.
