# Refresh-token rotation

One statement checks and writes, so two simultaneous refreshes cannot both succeed.

```python
async def claim_refresh_token(session, token_hash, now):
    """Atomically mark a usable token as revoked and return (id, user_id), or None.

    Of two simultaneous claims on the same token, the second waits for the first's row lock,
    then re-checks `revoked_at IS NULL` and matches nothing."""
    result = await session.execute(
        update(RefreshToken)
        .where(
            RefreshToken.token_hash == token_hash,
            RefreshToken.revoked_at.is_(None),
            RefreshToken.expires_at > now,
        )
        .values(revoked_at=now)
        .returning(RefreshToken.id, RefreshToken.user_id)
    )
    return result.first()
```

When a *used* token comes back, someone else holds a copy. The login is ended:

```python
if claimed is None:
    if revoked_at is not None:
        revoked = await revoke_family(session, family_id, now)
        # Commit BEFORE raising: an exception rolls the session back, which would undo the
        # revocation and leave the thief's replacement token usable.
        await session.commit()
    raise _invalid_refresh_token()   # same response for unknown, expired and reused
```

**Why it matters:** the subtle bug here is the rollback. Raising before committing would silently undo the defence.
