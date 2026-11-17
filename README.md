# SANDESH HOSPITALITY

**Hotel Management & Automation Platform**

SANDESH HOSPITALITY is a planned multi-tenant hotel-management SaaS for organizations operating one or more hospitality properties.

> **Current status:** Planning baseline and Next.js scaffold. The product features described here are not yet implemented. The Supabase project exists, but no approved application schema or migrations have been applied.

## Product direction

- Nepal-first launch planning, NPR-only financial transactions in the initial release.
- Multi-tenant organizations with one or more properties and strict organization/property isolation.
- Web-first, cloud-first V1; internet is required for normal operations.
- First release priority: identity and tenant foundation, property/room setup, reservations, and front desk.
- Financial workflows are gated by local accounting, hotel operating-policy, and payment-provider validation.
- Automation uses a platform-owned constrained workflow engine; templates precede a general visual builder.
- Full offline operation, Tauri desktop, public booking engine, OTA/channel-manager sync, restaurant POS, KOT, QR ordering, concierge, HIMS, biometric integrations, and SaaS subscription billing are outside the first usable release.

## Planning source of truth

Start with [the documentation index](docs/README.md), then read the consolidated [V1 Product and Architecture Baseline](docs/planning/V1-PRODUCT-AND-ARCHITECTURE-BASELINE.md).

The baseline records V1 scope, conservative domain defaults, implementation sequence, coding-entry checklist, and external validation gates. A proposal is not an implemented or legally approved behavior.

## Proposed technology

- Next.js App Router and TypeScript
- Supabase PostgreSQL and Auth
- Supabase Storage only for approved private-file workflows
- Vercel hosting
- GitHub source control and review
- Platform-owned tenant-scoped workflow engine

These are the selected planning defaults, subject to deployment, security, licensing, data-residency, and cost validation before production.

## Repository structure

```text
AGENTS.md
CONTRIBUTING.md
docs/
  api/
  architecture/
  automation/
  database/
  planning/
  product/
  quality/
  security/
  workflows/
src/                 Next.js application scaffold
```

## Local development

Requirements: Node.js and npm.

```bash
npm install
npm run dev
```

Open http://localhost:3000.

Useful checks (confirm the actual scripts in package.json):

```bash
npm run lint
npm run build
```

## Engineering principles

1. Requirements, domain rules, and acceptance criteria before implementation.
2. Implement one bounded vertical slice at a time.
3. Enforce authorization on the server and in PostgreSQL RLS; UI hiding is not security.
4. Use reviewed migrations; never rewrite a migration already applied to a shared environment.
5. Protect reservation inventory and financial operations with constraints, transactions, and idempotency.
6. Keep secrets out of browser code, source control, logs, and workflow definitions.
7. Test tenant isolation, concurrency, retries, and failure paths.
8. Use synthetic data outside approved production workflows.
9. Review changes through focused pull requests.
10. Do not merge or deploy without explicit approval.

## Important

The repository is not a production-ready hotel system. No reservations, PMS, front-desk, finance, payment, automation, or third-party integration feature should be claimed as available until implemented and tested. Do not store real guest data in the current development project.