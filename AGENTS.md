# Repository Instructions for AI Coding Agents

These rules apply to every agent and contributor working in this repository.

## 1. Project status and source of truth

- Read `README.md` and `docs/README.md` first.
- Read the relevant product, architecture, database, security, API, workflow, and testing documents before changing an affected area.
- GitHub is the source of truth for reviewed project changes.
- Treat documents marked "Initial proposal", "Proposed", or "Open" as unapproved requirements, not as permission to invent production behavior.
- Do not claim a feature, integration, test, or deployment exists unless the repository or an execution result proves it.

## 2. Scope discipline

- Implement one bounded feature at a time.
- Follow the approved V1 direction: web-first, cloud-first, multi-tenant SaaS.
- Do not add offline synchronization, a Tauri desktop client, or tenant-controlled arbitrary code execution to V1.
- Do not add real payment, biometric, door-lock, OTA, or messaging integrations until the provider, legal requirements, credentials, and acceptance criteria are explicitly approved.
- Do not silently expand scope or introduce a second architecture for the same responsibility.

## 3. Security and tenancy

- Treat organization and property boundaries as security boundaries.
- Enforce authorization on trusted server-side code and in PostgreSQL RLS; hiding UI controls is not authorization.
- Never expose Supabase service-role keys, provider secrets, signing keys, or privileged credentials to browser code.
- Do not use user-editable profile metadata as an authorization source.
- Every tenant-scoped query and mutation must carry and verify the authorized organization/property context.
- Review SECURITY DEFINER functions, views, storage policies, RPC grants, and webhook handlers as privileged code.
- Avoid logging credentials, payment data, identity-document contents, or unnecessary guest personal data.
- Use synthetic data in development and tests.

## 4. Database and business integrity

- Use reviewed, ordered database migrations. Never edit a migration that has already been applied to a shared environment.
- Add constraints for invariants that must hold regardless of application code.
- Protect room inventory, reservation transitions, payments, and ledger writes with server-side transactions and idempotency.
- Never silently overwrite or delete posted financial entries.
- Use explicit timezone/business-date semantics; do not assume server-local time equals property-local time.
- Document assumptions and unresolved decisions instead of guessing.

## 5. Implementation quality

- Follow the existing framework and repository conventions; inspect the current code before proposing replacements.
- Prefer small, typed, testable modules over giant components or one-file implementations.
- Validate all untrusted input at the server boundary.
- Return safe, consistent errors without leaking internal details.
- Keep business rules out of presentation-only components.
- Avoid unnecessary dependencies. Explain the need and maintenance impact before adding major infrastructure.
- Preserve accessibility and responsive behavior in UI work.

## 6. Required checks

Before marking a change complete:
- Run the relevant lint, typecheck, unit, integration, and build commands that actually exist in `package.json`.
- Add or update tests for changed behavior, especially authorization, tenant isolation, concurrency, and failure handling.
- Inspect the final diff for accidental secrets, generated artifacts, unrelated changes, and unsafe defaults.
- State exactly which checks ran and which could not run.
- Never report a test as passing if it was not executed.

## 7. Change workflow

- Work on a feature branch for implementation; do not push unreviewed feature code directly to `main`.
- Keep commits focused and descriptive.
- Open a pull request with scope, design notes, tests, security impact, migration impact, and known limitations.
- Do not merge or deploy without explicit release approval.
- Do not rewrite history, force-push, delete branches, alter repository permissions, or change secrets/settings without explicit approval.

## 8. Documentation

- Update the documentation index when adding a major document.
- Keep requirements, architecture, database, API, and tests consistent.
- Mark conceptual material as proposed until reviewed.
- Record important architecture decisions and their consequences.
