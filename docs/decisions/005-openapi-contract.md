# ADR 005: API contract as a checked-in snapshot

**Status:** accepted

## Context
The app and API change together. A renamed field should break the build, not a user's screen.

## Decision
- The API exports its OpenAPI document into the app project as a committed `openapi.json`.
- The app generates TypeScript types from it (`openapi-typescript`), so calls are checked at compile time.
- A test regenerates the document and fails if it differs from the committed file. Changing an endpoint therefore requires committing the new contract, and the diff shows reviewers exactly what changed.

## Consequences
- No hand-written client and no drift between server and client.
- Contract changes are visible in code review as data, not prose.
