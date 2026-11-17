# SANDESH HOSPITALITY

**Hotel Management & Automation Platform**

SANDESH HOSPITALITY is a planned multi-tenant hotel-management SaaS for managing property operations through a web-first, cloud-first platform. The product direction includes reservations, property management, front-desk operations, guest services, restaurant workflows, finance, reporting, and controlled automation.

> **Project status:** Planning and foundation stage. The repository currently contains a Next.js scaffold and product/architecture documentation. The hotel-management features described here are not yet implemented.

## Product direction

- Multi-tenant organizations with one or more properties.
- Strict organization and property data isolation.
- Web-first V1; internet connectivity is required for normal operation.
- Reservations, PMS foundations, and front desk are the first implementation priorities.
- Automation templates plus a constrained visual workflow builder.
- Offline application support and a Tauri desktop client are deferred beyond V1.
- During a V1 internet outage, properties use documented printable/manual procedures and reconcile records after service returns. V1 is not an offline-capable system.

## Proposed technology

- Next.js and TypeScript
- Supabase: PostgreSQL, Auth, and Storage
- Vercel hosting
- GitHub source control
- A platform-owned, tenant-scoped workflow engine for customer automations

These are architectural proposals and must be validated as implementation decisions. No production Supabase or Vercel environment is implied by this repository.

## Repository structure

```text
AGENTS.md       Repository-wide AI agent instructions
CONTRIBUTING.md Contribution and review workflow
docs/
  api/          API design
  architecture/ System architecture and decision records
  automation/   Automation engine design
  database/     Database design and ERD
  planning/     Open decisions, roadmap, and assumptions
  product/      Product requirements
  quality/      Testing and release strategy
  security/     Security architecture and permission matrix
  workflows/    Reservation and front-desk workflows
src/            Next.js application scaffold
```

See [the documentation index](docs/README.md) for the document register and planning status.

## Local development

Requirements: Node.js and npm.

```bash
npm install
npm run dev
```

Open http://localhost:3000.

Useful checks:

```bash
npm run lint
npm run build
```

## Development principles

1. Requirements and acceptance criteria before implementation.
2. Implement one bounded feature at a time.
3. Enforce authorization on the server and in database policies; UI hiding is not security.
4. Use migrations for database changes; never rewrite an already-applied migration.
5. Protect reservation inventory and financial operations with database constraints and transactions.
6. Keep secrets out of browser code, repository files, logs, and workflow definitions.
7. Test tenant isolation, concurrency, idempotency, and failure cases.
8. Review changes before merging and deploy to staging before production.

## Important

All documents are initial proposals unless explicitly marked otherwise. Open market, legal, tax, payment, and operational decisions must be resolved before implementing affected production features. Do not use real guest data in development.
