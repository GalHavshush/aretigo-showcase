# Tenant context

The only way a handler learns which studio it is acting in. The header is a claim; the membership lookup is the check.

```python
@dataclass(frozen=True)
class TenantContext:
    """Who is acting, in which studio, and with which role there. Only built after the
    membership was checked, so `tenant_id` and `role` can be trusted."""

    user: User
    tenant_id: uuid.UUID
    role: str  # "owner", "coach" or "member"


async def tenant_ctx(user: CurrentUser, session: DbSession, x_tenant_id: Header = None) -> TenantContext:
    # 401 first (current_user), then 400 for a missing/malformed header, then 403 with no
    # membership. A studio that does not exist gets the same 403, so ids cannot be probed.
    ...
    if role is None:
        raise HTTPException(status_code=403, detail="not_a_member")
    return TenantContext(user=user, tenant_id=tenant_id, role=role)


# Tenant-scoped endpoints write `ctx: TenantCtx`; every query in them filters by ctx.tenant_id.
TenantCtx = Annotated[TenantContext, Depends(tenant_ctx)]
```

A handler then reads naturally, and has no way to name another studio:

```python
@router.get("/tenant")
async def get_tenant(ctx: TenantCtx, session: DbSession) -> TenantDetail:
    row = await session.scalar(select(Tenant).where(Tenant.id == ctx.tenant_id))
    return TenantDetail.model_validate(row)
```

**Why it matters:** the role is read per request rather than stored in the token, so removing a membership applies on the very next call.
