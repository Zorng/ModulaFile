# Starting a Shift When Operator Seats Are Full

## Context

A branch may operate with multiple people.
But the subscription may include only a limited number of operator seats.

This matters most at the start of work, when staff arrive and need to begin serving customers quickly.

---

## The Situation

A staff member (or an admin) attempts to start work at a branch.

The system determines that:
- the branch’s Workforce module is active, but
- all operator seats are already in use (too many people are currently active)

This can happen when:
- someone forgot to end work
- a shift overlap is larger than expected
- multiple people tried to start at once

---

## What the Staff Member Expects

They expect:
- a clear explanation of why they cannot start work
- to know what to do next

They do **not** expect:
- vague errors like “operation failed”
- to debug billing while customers are waiting

---

## What the Owner/Admin Expects

They expect:
- the system to enforce the seat limit consistently
- a clear upgrade path if the business truly needs more concurrency
- a way to resolve it operationally (end the correct session) if it was a mistake

---

## Constraints & Reality

- The seat limit is about **simultaneous operation**, not about how many staff names exist.
- The system must avoid allowing “extra” staff to operate silently, because it breaks commercial fairness and operational accountability.

---

## How Modula Helps

Modula supports this situation by:
- blocking the start of work with a clear message: “Operator seat limit reached”
- showing who is currently active (so the right session can be ended)
- offering a simple “upgrade seats” action for the owner/admin

---

## What This Story Is Not About

- Calculating seat price or invoices
- Attendance evaluation or shift planning

This story is about a real counter moment:
> “I’m here to start work — either let me in, or tell me exactly why not.”

