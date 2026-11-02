# Stacking roles

```python
ROLE_RANK = {"member": 0, "coach": 1, "owner": 2}


def require_role(min_role: str):
    """Pass when the user's role in the studio is `min_role` or higher. An unknown role ranks
    below everyone, so it is refused, never let through."""
    needed = ROLE_RANK[min_role]   # a typo here fails when the app loads, not per request

    async def dependency(ctx: TenantCtx) -> TenantContext:
        if ROLE_RANK.get(ctx.role, -1) < needed:
            raise HTTPException(status_code=403, detail="insufficient_role")
        return ctx

    return dependency
```

Used as `Depends(require_role("coach"))`. The database backs it with `CHECK (role IN ('owner','coach','member'))`.
