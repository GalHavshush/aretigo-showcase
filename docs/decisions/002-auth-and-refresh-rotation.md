# ADR 002: Our own auth: JWT access tokens and rotating refresh tokens

**Status:** accepted

## Context
Users are coaches and members on phones, often on poor connections, who must stay signed in for weeks but can lose a device.

## Decision
- **Access token**: HS256 JWT, 15 minutes, holds only the user id. Verified without a database call; the user row is then loaded so a deleted account is refused immediately.
- **Refresh token**: random opaque string, 30 days. Only its SHA-256 hash is stored, so a database leak does not leak sessions.
- **Rotation**: each refresh consumes the token and issues a new one in the same *family* (one family per login/device).
- **Reuse detection**: presenting an already-used token means two parties hold it. The whole family is revoked and the user signs in again.
- **Roles stay out of the token**: they are read from the membership per request, so changes apply at once.

## Why rotation is race-safe
`claim_refresh_token` is one `UPDATE … WHERE revoked_at IS NULL … RETURNING`. Of two simultaneous claims on one token, the second waits on the row lock, re-evaluates the condition and matches nothing. A per-user lock orders refresh, logout and account deletion so none can slip a token past another.

## Trade-offs
- A thief who refreshes first and is never contradicted stays signed in until expiry. Reuse is only noticed when the old token comes back. Accepted; documented in the endpoint.
- Sliding expiry means a device used monthly never signs out. An absolute cap is a one-column change if the product wants one.
- Building auth ourselves means owning its security. The test suite includes reuse, concurrent refresh, logout races and deletion cases.
