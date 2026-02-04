# Managing Which Businesses Are Active

## Context

Not all business relationships are permanent.

A business may:
- stop using the service
- pause temporarily
- fail to complete payment
- need to be disabled for operational reasons

In these cases, the system must clearly reflect whether a business is allowed to operate.

---

## The Situation

A decision is made that a business should no longer be active.

This decision may come from:
- the subscriber
- an operator or administrator
- a change in the commercial relationship

The result must be unambiguous:
- the business is either allowed to operate, or it is not

There should be no hidden or partial states.

---

## What the Subscriber or Operator Expects

They expect:
- a clear way to mark a business as active or inactive
- that the decision takes effect consistently
- that operational users are not left guessing
- past records to remain intact

They do **not** expect:
- data to be deleted
- history to be rewritten
- partial failures where some actions still work

From their perspective, this is about **control and clarity**, not punishment.

---

## Constraints & Reality

- Operational staff may still have valid accounts
- Devices may still exist
- Historical data must remain accessible

Disabling a business should stop **new operations**, not erase the past.

---

## How Modula Helps

Modula supports this situation by:
- treating business activity as an explicit state
- ensuring that inactive businesses cannot continue operations
- preserving historical data for review or audit
- keeping the rule simple and understandable

This provides a clear boundary between commercial decisions and operational history.

---

## What This Story Is Not About

- Billing enforcement logic
- Grace periods or retries
- Feature-level restrictions
- Customer communication

This story is about **clearly defining whether a business is allowed to operate**, so the system behaves consistently.