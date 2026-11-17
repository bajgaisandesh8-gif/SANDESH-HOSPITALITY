## Summary

Describe the problem and the change.

## Scope

- What is included?
- What is explicitly not included?

## Requirements and design

- Related requirement/document:
- Any assumptions or open decisions:
- Architecture or API impact:

## Security and data

- [ ] Organization and property authorization reviewed.
- [ ] RLS/storage policies reviewed where applicable.
- [ ] No secrets or sensitive guest/payment data added.
- [ ] Audit and logging impact reviewed.
- [ ] Database migration is additive/reviewed; no applied migration was rewritten.

## Data integrity

- [ ] Reservation/inventory concurrency considered where applicable.
- [ ] Financial history and ledger rules preserved where applicable.
- [ ] Idempotency and retry behavior considered where applicable.
- [ ] Timezone and property business-date behavior considered where applicable.

## Tests

- [ ] Lint
- [ ] Typecheck
- [ ] Unit tests
- [ ] Database/RLS tests
- [ ] API/integration tests
- [ ] End-to-end tests
- [ ] Build

List the exact commands run and results. Do not check tests that were not executed.

## Deployment and operations

- [ ] Environment variables documented without exposing values.
- [ ] Migration/deployment order documented.
- [ ] Monitoring and failure handling considered.
- [ ] Rollback or forward-fix plan documented where needed.

## Screenshots / evidence

Attach screenshots or relevant test output for user-facing changes.

## Known limitations and follow-up work

List remaining risks, deferred decisions, or follow-up tasks.
