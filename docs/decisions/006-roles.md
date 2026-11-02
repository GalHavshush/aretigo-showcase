# ADR 006: Roles that stack, stored as data

**Status:** accepted

## Decision
- Three roles, `owner ⊃ coach ⊃ member`, stored as one string on the membership and guarded by a `CHECK` constraint.
- `require_role("coach")` passes for coaches and owners. The rank table is the only place that knows the order.
- An unknown role ranks below everyone, so a bad value is refused rather than allowed.
- A typo in a required role fails when the application loads, not on the first request.

## Consequences
Adding a role means one table entry and one constraint change. Per-permission checks (finer than roles) are deliberately not built until a studio needs them.
