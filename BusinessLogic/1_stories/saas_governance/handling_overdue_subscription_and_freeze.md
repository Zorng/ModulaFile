# Handling Overdue Subscription and Freeze

## Context

Business owners sometimes miss a payment:
- they forget
- a card/bank transfer fails
- they are busy and do not notice the renewal date

But a POS cannot be shut off casually during operations.

The system must balance:
- commercial enforcement
- operational reality

---

## The Situation

The tenant reaches a renewal date and the invoice is not paid.

The owner/admin should:
- be warned clearly
- have a short chance to fix the payment
- understand what happens if they do nothing

---

## What the Owner/Admin Expects

They expect:
- a clear “past due” warning
- a short grace window to pay without operational disruption
- after the grace window, clear restrictions (not partial chaos)
- to be able to pay and restore access quickly

They do **not** expect:
- silent failures
- confusing partial blocks where some actions work and others don’t

---

## Constraints & Reality

- Modula should not lock the counter instantly at renewal time.
- There must still be a real enforcement boundary if payment remains unpaid.
- Past records must remain accessible.

---

## How Modula Helps

Modula supports this situation with clear states:

- `PAST_DUE` (24 hours):
  - operations continue
  - the system warns loudly and repeatedly
  - the owner/admin is guided to pay immediately

- `FROZEN` (after 24 hours unpaid):
  - operational writes are blocked (no selling, no new cash sessions, no new work sessions)
  - read-only access remains available (view history, reports, invoices)
  - paying restores access back to active

---

## What This Story Is Not About

- Debt collection policy
- Customer support escalation
- Legal/compliance policies

This story is about predictable enforcement:
> “Warn first, then freeze cleanly, without destroying history.”

