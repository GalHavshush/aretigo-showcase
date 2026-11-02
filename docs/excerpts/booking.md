# Booking a seat

Lock the class, count seats, write once. Different classes never wait on each other.

```python
# Locks the session row: if someone else is booking this class, wait here for them to finish.
found = (
    await session.execute(
        select(ClassSession.status, ClassSession.capacity, ClassType.attendance_mode)
        .join(ClassType, ClassType.id == ClassSession.class_type_id)
        .where(ClassSession.id == session_id, ClassSession.tenant_id == ctx.tenant_id)
        .with_for_update(of=ClassSession, key_share=True)
    )
).first()

# With the row locked, the count below cannot go stale before the write.
full = (
    wants_a_seat
    and previous_status != "going"          # someone who holds a seat keeps it
    and await _count_going(session, session_id) >= found.capacity
)
if full:                                     # wait in line; keep your place if you were already waiting
    status = "waitlist"
    waitlisted_at = (previous.waitlisted_at if previous else None) or datetime.now(UTC)

# One atomic statement, so the same member's double tap is idempotent.
await session.execute(
    insert(Rsvp).values(...)
    .on_conflict_do_update(index_elements=[Rsvp.session_id, Rsvp.participant_id],
                           set_={"status": status, "waitlisted_at": waitlisted_at})
)
```

**Why it matters:** `FOR UPDATE … key_share` takes the weakest lock that still serialises bookers, so unrelated writes to the row are not blocked.
