# ADR 004: Optional modules per studio

**Status:** accepted

## Context
A yoga studio wants RSVPs. A boutique cycling studio wants capacity and a waitlist. Forcing every studio through booking makes the simple case worse.

## Decision
- A `tenants.modules` list names what each studio has switched on (today: `booking`).
- A `require_module(name)` dependency guards module-only routes with a clear `403 module_disabled`.
- Routes shared by both kinds of studio check the list themselves, and the app reads it from `GET /tenant` to decide what to show.
- Behaviour is per **class type**: a type is either `rsvp` or `booking`, so one studio can mix both.

## Consequences
- New capabilities ship dark and are enabled per studio, which suits hand-onboarded pilots.
- A JSON list is the simplest store; if modules ever need settings or billing, it becomes a table without changing the dependency's interface.
