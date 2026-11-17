<!-- BEGIN:nextjs-agent-rules -->

## This is NOT the Next.js you know

This version has breaking changes — APIs, conventions, and file structure may differ from prior versions. Read the relevant guide in `node_modules/next/dist/docs/` (resolved from this file's directory; in monorepos the `next` package may not be visible from the repo root) before writing code. Heed deprecation notices.

This block is written and re-added by `next dev` — verify at `node_modules/next/dist/server/lib/generate-agent-files.js`. Preserve it; committing it with work keeps the tree clean.

<!-- END:nextjs-agent-rules -->

# Repository Instructions for AI Coding Agents

These rules apply to every agent and contributor working in this repository.

## 1. Project status and source of truth

- Read `README.md`, `docs/README.md`, and `docs/planning/V1-PRODUCT-AND-ARCHITECTURE-BASELINE.md` first.
- Read the relevant product, architecture, database, security, API, workflow, and testing documents before changing an affected area.
- GitHub is the source of truth for reviewed project changes.
- Treat documents marked Initial proposal, Proposed, or Open as unapproved requirements, not as permission to invent production behavior.
- Do not claim a feature, integration, test, or deployment exists unless repository code or an execution result proves it.
- The baseline's external validation gates remain binding. Do not bypass them by inventing tax, legal, payment-provider, data-residency, or hotel operating rules.

## 2. Scope discipline

- Implement one bounded feature at a time.
- Follow the V1 direction: Nepal-first planning, multi-tenant SaaS, web-first and cloud-first.
- The first coding slice is identity plus organization/property context and tenant-isolation tests.
- Do not begin reservation, folio, payment, invoice, or production-data work before the applicable baseline gates and schema reviews are complete.
- Do not add offline synchronization, Tauri desktop, public booking engine, OTA/channel-manager, restaurant POS, KOT, QR ordering, concierge, HIMS, biometric integration, or SaaS subscription billing to the first usable release without an explicit scope decision.
- Do not add tenant-controlled arbitrary code execution, unrestricted HTTP, arbitrary SQL, or unrestricted shared n8n access.
- Do not add real payment, biometric, door-lock, OTA, or messaging integrations until provider, legal requirements, credentials, and acceptance criteria are approved.

## 3. Security and tenancy

- Treat organization and property boundaries as security boundaries.
- Enforce authorization in trusted server-side code and PostgreSQL RLS; hiding UI controls is not authorization.
- Never expose Supabase service-role keys, secret keys, provider secrets, signing keys, or privileged credentials to browser code.
- Do not use user-editable profile metadata as an authorization source.
- Every tenant-scoped query and mutation must carry and verify authorized organization/property context.
- Review SECURITY DEFINER functions, views, storage policies, RPC grants, and webhook handlers as privileged code.
- Avoid logging credentials, payment data, identity-document contents, or unnecessary guest personal data.
- Use synthetic data in development and tests.
- Platform support has no default tenant-data access; any future exceptional access must be explicitly approved, time-limited, scoped, and audited.

## 4. Database and business integrity

- Use reviewed, ordered database migrations. Never edit a migration already applied to a shared environment.
- Add database constraints for invariants that must hold regardless of application code.
- Protect room inventory, reservation transitions, payments, and ledger writes with server-side transactions and idempotency.
- Never silently overwrite or delete posted financial entries.
- Use explicit timezone/business-date semantics; never assume server-local time equals property-local time.
- Use NPR-only financial operations in the initial release; reject unsupported currencies rather than silently converting.
- Preserve the one-primary-folio-per-reservation and split-tender rules in the approved design baseline.
- Document assumptions and unresolved external decisions instead of guessing.

## 5. Implementation quality

- Follow existing framework and repository conventions; inspect current code before proposing replacements.
- Prefer small, typed, testable modules over giant components or one-file implementations.
- Validate all untrusted input at the server boundary.
- Return safe, consistent errors without leaking internal details.
- Keep business rules out of presentation-only components.
- Avoid unnecessary dependencies. Explain the need and maintenance impact before adding major infrastructure.
- Preserve accessibility and responsive behavior.

## 6. Required checks

Before marking a change complete:
- Run the relevant lint, typecheck, unit, integration, database, and build commands that actually exist.
- Add/update tests for changed behavior, especially authorization, tenant isolation, concurrency, idempotency, and failure handling.
- Inspect the final diff for accidental secrets, generated artifacts, unrelated changes, and unsafe defaults.
- State exactly which checks ran and which could not run.
- Never report a test as passing if it was not executed.

## 7. Change workflow

- Work on a focused feature branch; do not push unreviewed feature code directly to `main`.
- Keep commits focused and descriptive.
- Open a pull request with scope, design notes, tests, security impact, migration impact, and known limitations.
- Do not merge or deploy without explicit release approval.
- Do not rewrite history, force-push, delete branches, alter repository permissions, or change secrets/settings without explicit approval.
- Do not apply production Supabase migrations during local implementation or planning.

## 8. Documentation

- Update the documentation index when adding a major document.
- Keep requirements, architecture, database, API, and tests consistent.
- Mark conceptual material as proposed until implemented and reviewed.
- Record important architecture decisions and consequences.
- Never state that the project is production-ready merely because the scaffold builds.
