# Contributing

SANDESH HOSPITALITY is being developed incrementally. The repository is currently in its planning/foundation stage.

## Before starting

1. Read [README.md](README.md) and [the documentation index](docs/README.md).
2. Read the relevant requirements and design documents.
3. Check [open decisions](docs/planning/OPEN-DECISIONS.md).
4. Confirm that the feature's business rules and acceptance criteria are sufficiently defined.
5. Confirm the current repository state and branch before editing.

## Branch and pull request

- Create a focused branch from the current `main`.
- Keep each pull request limited to one coherent feature or maintenance task.
- Do not commit directly to `main` for normal feature implementation.
- Do not merge or deploy without review and explicit release approval.

A pull request should explain:
- Problem and intended outcome.
- Scope and explicit non-goals.
- Key implementation choices.
- Database migrations and rollback/forward-fix considerations.
- Authorization and tenant-isolation impact.
- Tests executed and their results.
- Known limitations and follow-up decisions.

## Engineering requirements

- Use migrations for schema changes.
- Add automated tests for business rules and security boundaries.
- Use synthetic guest and payment data in development.
- Do not commit secrets, environment files containing credentials, production exports, or real guest data.
- Do not treat UI-level restrictions as security controls.
- Preserve auditability and financial history.

## Local commands

The actual commands are determined by the committed `package.json`. Once the application scaffold is present in the repository, document and use its lint, typecheck, test, and build scripts. Do not invent a passing result for a command that has not been run.
