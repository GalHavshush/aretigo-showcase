# ADR 001: Tenant isolation: one dependency, every query scoped

**Status:** accepted

## Context
Many studios share one deployment. A single missed filter would show one studio's members to another, which is the worst bug this product can have.

## Options
| Option | Why not |
|---|---|
| Database per studio | Operationally heavy at the scale of small studios; migrations multiply |
| Schema per studio | Same, and harder to pool connections |
| Database row-level security with client-side access | Puts authorization in SQL policies and gives the client database credentials |
| **Shared tables, `tenant_id` on every row, enforced in the API** | Chosen |

## Decision
- Every request names its studio in `X-Tenant-Id`. The `tenant_ctx` dependency treats that header as a *claim* and verifies it against the user's membership. Only then does it build a `TenantContext(user, tenant_id, role)`.
- Handlers take `ctx: TenantCtx` and filter by `ctx.tenant_id`. The tenant is never read from a path or body.
- Child tables use **composite foreign keys `(tenant_id, id)`**. A session cannot point at another studio's class type, even if a handler is wrong.
- A missing studio and a studio you are not in return the same `403`, so ids cannot be probed.

## Consequences
- Isolation depends on discipline in each query plus the composite keys as a safety net, and each handler has tests asserting a second studio's data stays invisible.
- Costs one membership lookup per request, a single indexed read.
- Moving a large studio to its own database later is possible: all of its rows are found by one column.
