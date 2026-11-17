# Architecture Decision Records

**Status:** Consolidated architecture decision register. See the V1 Product and Architecture Baseline for the controlling release boundary and remaining external gates.

This file records why a direction is being considered and what must be validated. New decisions should be added rather than silently changing foundational assumptions.

## ADR-001 — Web-first, cloud-first V1

**State:** Accepted product direction  
**Context:** The first release needs a manageable deployment and operational model. Building a desktop client and offline synchronization would add a second runtime, local data lifecycle, conflict resolution, and security surface before core hotel workflows are proven.

**Decision:**
- V1 is a web application.
- Normal operation requires connectivity to hosted services.
- Tauri desktop packaging and full offline synchronization are deferred.
- Properties use documented printable/manual contingency procedures during an outage.

**Consequences:**
- Do not describe V1 as offline-capable.
- Outage procedures and post-outage reconciliation are product requirements.
- Future offline support must be designed as a separate, explicitly approved architecture phase.

## ADR-002 — Tenant workflows use a constrained platform-owned engine

**State:** Accepted V1 direction; detailed runtime limits and event/action contracts remain to be validated  
**Context:** Customer-created automation is a product requirement, but arbitrary tenant code or unrestricted shared workflow infrastructure creates isolation, credential, abuse, and cost risks.

**Decision:**
- Prefer a platform-owned workflow definition and runner for tenant workflows.
- Use a closed trigger, condition, and action catalog.
- Keep tenant scope and action authorization in trusted backend code.
- Consider n8n only for platform-owned integrations with fixed, narrowly scoped interfaces, after commercial license review.

**Consequences:**
- No arbitrary JavaScript, unrestricted HTTP, or direct database query nodes for tenants.
- Workflow versions, execution history, idempotency, retry limits, loop protection, and tenant quotas are required.
- The engine design must be validated before implementing a visual editor.

## ADR-003 — Supabase as managed development data/auth platform

**State:** Selected for development; production region, contract, and data-residency approval remain open  
**Context:** The project requires PostgreSQL integrity, authentication, storage, and tenant-aware access controls.

**Decision:**
- Use the provisioned Supabase PostgreSQL and Auth project as the development foundation; do not treat it as production.
- Apply PostgreSQL constraints and RLS to tenant-scoped data.
- Keep privileged service credentials on trusted server-side infrastructure only.
- Use migrations as the schema source of truth.

**Consequences:**
- RLS tests are release-blocking.
- SECURITY DEFINER functions, views, storage policies, and RPC grants need explicit review.
- The project must decide deployment region, backup/recovery targets, and environment separation before production.

## ADR-004 — V1 implementation sequence starts with identity and tenant isolation

**State:** Proposed sequence  
**Context:** Reservations, folios, housekeeping, and automation all depend on correct organization/property scope and staff permissions.

**Decision:**
- Establish authentication, organization/property membership, authorization, and tenant-isolation tests before implementing hotel business modules.

**Consequences:**
- No reservation or guest feature should be built on an unverified authorization foundation.
- Permission and RLS behavior must be tested at database and API boundaries.
- Final role definitions remain subject to product/operations review.

## ADR-005 — GitHub is the reviewed source of truth

**State:** Accepted development workflow  
**Context:** Multiple AI tools and local development can otherwise produce divergent or unreviewed implementations.

**Decision:**
- GitHub stores reviewed source and documentation.
- Feature work uses focused branches and pull requests.
- Local changes are not considered integrated until committed and verified remotely.
- AI agents must follow `AGENTS.md` and report checks truthfully.

**Consequences:**
- Keep local and remote branches synchronized.
- Do not deploy unreviewed local work.
- Do not merge or release without explicit approval.


## ADR-006 — Modular monolith for the first release

**State:** Accepted V1 architecture default  
**Context:** The first release needs strong transaction boundaries and one coherent authorization model. Splitting reservations, finance, guest records, and automation into independent services too early would add distributed failure modes and duplicate policy enforcement.

**Decision:**
- Build the V1 application as a modular monolith using Next.js server-side application/domain modules and PostgreSQL.
- Keep clear module boundaries and typed contracts so later extraction remains possible.
- Add a separately deployed worker only when durable background processing (such as automation/outbox delivery) requires it.
- Do not introduce a second backend framework or microservices without a measured requirement and an explicit architecture review.

**Consequences:**
- Critical reservation and financial invariants remain within PostgreSQL transactions.
- Shared authorization and tenant-scope logic must not be duplicated across runtimes.
- Background work must be idempotent and must not block core request paths.

## ADR-007 — NPR-only initial financial operations and reservation-owned folios

**State:** Accepted V1 product direction  
**Decision:**
- Initial financial operations use NPR only; non-NPR operations are rejected until currency/FX rules are approved.
- Each reservation owns one primary folio; the check-in stay reuses it.
- Split tender is supported through multiple independently verified payment records allocated to that same folio.
- A single tender cannot be allocated across multiple folios in V1.

**Consequences:**
- Advance deposits are associated with the reservation folio, not an unassigned guest wallet.
- Multi-currency, master folios, and cross-folio payment splitting are deferred.
- Finance implementation remains blocked by the Nepal accounting, hotel operating-policy, and payment-provider gates in the V1 baseline.
