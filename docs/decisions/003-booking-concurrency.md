# ADR 003: Booking under load: row locks, not optimistic retries

**Status:** accepted

## Context
A class with 12 seats and 20 people tapping "Book" at 18:00 must never overbook, and the waitlist must be fair.

## Options
| Option | Why not |
|---|---|
| Read the count, then insert | Two requests both read 11 and both insert |
| Counter column with a check constraint | Waitlist promotion still needs ordering; more moving parts |
| Serializable isolation with retries | Pushes retry logic into every client path |
| **Lock the session row, then count and write** | Chosen |

## Decision
`rsvp_to_session` opens with `SELECT … FOR UPDATE` on the class session. Concurrent bookings for that class queue behind each other; different classes never block each other. With the lock held, the seat count cannot change before the write.

- Full → the answer is still `200`, with status `waitlist` and the original waiting time kept.
- Freed seat → the earliest waiting people are promoted until the class is full again.
- The RSVP write is a single `INSERT … ON CONFLICT DO UPDATE`, so the same member's double tap is idempotent.

## Consequences
- Throughput per class is serial, which is fine at studio scale (tens of people, not thousands) and gives simple, provable behaviour.
- A dedicated test fires parallel requests against a real Postgres and asserts seat and waitlist outcomes.
