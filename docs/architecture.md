# Architecture

## System

```mermaid
flowchart TB
    subgraph Client["Mobile app (Expo, React Native, TypeScript)"]
        UI["Screens · Expo Router"]
        TQ["Generated API types"]
        I18N["i18n · light/dark theme"]
    end
    subgraph Server["API (FastAPI, async)"]
        MW["Dependencies:<br/>current_user → tenant_ctx → require_role / require_module"]
        R["Routers:<br/>auth · schedule · series · bookings · attendance · reports · notes · timeline"]
    end
    DB[("PostgreSQL 17")]
    UI --> TQ -- "Bearer JWT + X-Tenant-Id" --> MW --> R --> DB
```

Three rules shape everything else:

1. **The API is the only application surface.** The database has no public endpoint and the client holds no database credentials. Authorization lives in one place, in Python, where it is unit-tested.
2. **Identity is global, access is per studio.** A person has one account and any number of memberships. Every request names a studio; the server checks the membership.
3. **Constraints beat conventions.** Whatever the database can enforce (roles, positive capacities, uniqueness, same-studio links) it enforces, so a bug in a handler cannot corrupt data.

## Request flow

```mermaid
sequenceDiagram
    participant C as App
    participant A as FastAPI
    participant D as PostgreSQL
    C->>A: PUT /sessions/{id}/rsvp  (Bearer, X-Tenant-Id)
    A->>A: current_user: verify JWT signature and expiry
    A->>D: load user (reject deleted accounts)
    A->>D: tenant_ctx: membership lookup for (user, studio)
    A->>D: lock the session row (FOR UPDATE)
    A->>D: count seats, upsert RSVP, promote waitlist
    A->>D: COMMIT
    A-->>C: 200 {status: "going" | "waitlist"}
```

Failure order is deliberate and stable: `401` (who are you) → `400` (which studio) → `403` (not your studio, or role too low) → `404` (resource, scoped to your studio) → `409` (state conflict). Clients can rely on it.

## Data model

```mermaid
erDiagram
    USERS ||--o{ MEMBERSHIPS : has
    TENANTS ||--o{ MEMBERSHIPS : has
    TENANTS ||--o{ CLASS_TYPES : owns
    CLASS_TYPES ||--o{ CLASS_SERIES : "repeats as"
    CLASS_TYPES ||--o{ CLASS_SESSIONS : "scheduled as"
    CLASS_SERIES ||--o{ CLASS_SESSIONS : generates
    CLASS_SESSIONS ||--o{ RSVPS : receives
    PARTICIPANTS ||--o{ RSVPS : makes
    CLASS_SESSIONS ||--o| CLASS_REPORTS : "summarised by"
    CLASS_REPORTS ||--o{ PARTICIPANT_NOTES : contains
    USERS ||--o{ REFRESH_TOKENS : holds
```

- Every studio-owned table carries `tenant_id`.
- Child tables reference their parents through **composite keys `(tenant_id, id)`**, so the database refuses a row that links two different studios. See [ADR 001](decisions/001-tenant-isolation.md).
- Times are stored as UTC instants; a studio's IANA timezone is returned with it and the client formats.
- The API returns **codes, not sentences** (`class_type_name_taken`); wording and language belong to the client.

## Security model

| Concern | Approach |
|---|---|
| Passwords | argon2 hashes only |
| Access | HS256 JWT, 15 minutes, carries only the user id |
| Sessions | Opaque refresh tokens, 30 days, stored as SHA-256 hashes, rotated on every use, revoked as a family when reuse is detected ([ADR 002](decisions/002-auth-and-refresh-rotation.md)) |
| Roles | Not in the token: read from the membership on each request, so removing someone takes effect on their next call |
| Probing | A studio that does not exist and a studio you are not in both return `403 not_a_member` |
| Deletion | Deleting an account revokes every device at once and rejects its still-valid access tokens immediately |

## Concurrency

Two places where two requests can collide, both covered by tests that run real concurrent transactions against Postgres:

- **Seat booking**: serialised with a row lock on the session ([ADR 003](decisions/003-booking-concurrency.md)).
- **Refresh rotation**: one `UPDATE … RETURNING` claims a token, so only one of two simultaneous refreshes can win.

## Delivery

```mermaid
flowchart LR
    B["short-lived branch"] --> PR["pull request"] --> CI{"CI"}
    CI -->|"ruff · pytest on Postgres 17"| API["API"]
    CI -->|"tsc · jest"| APP["App"]
    API --> M["squash-merge to master"]
    APP --> M
```

- `master` is always releasable; there is no long-lived staging branch.
- Migrations are tested in CI: upgrade to head, downgrade to base, upgrade again, plus data-backfill cases.
- The OpenAPI document is committed; a test fails when the API changes without regenerating it ([ADR 005](decisions/005-openapi-contract.md)).
