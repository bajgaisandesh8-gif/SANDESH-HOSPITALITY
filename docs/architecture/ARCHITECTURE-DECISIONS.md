# Architecture Decision Records

**Status:** Initial decision register. Entries marked Proposed are not final approval.

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

**State:** Proposed architecture  
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

## ADR-003 — Supabase as proposed managed data/auth platform

**State:** Proposed; validate before provisioning production  
**Context:** The project requires PostgreSQL integrity, authentication, storage, and tenant-aware access controls.

**Decision:**
- Use Supabase PostgreSQL and Auth as the current proposed foundation.
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
