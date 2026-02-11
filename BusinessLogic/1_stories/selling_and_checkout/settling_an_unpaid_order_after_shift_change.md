# Settling an Unpaid Order After a Shift Change

## Context

In real operations, staff turnover happens during service:
- a cashier ends their shift,
- another cashier takes over,
- customers still need to settle open orders.

This story exists to ensure pay-later does not create an ownership dead-end where only the original cashier can finish an order.

---

## The Situation

Cashier A places an order for a customer (pay-later workflow).

The kitchen prepares and serves the items.

Before the customer pays, Cashier A clocks out.

Later, the customer asks for the bill.

Cashier B must settle the order without:
- changing the items history,
- losing accountability,
- or creating duplicates.

---

## What the Staff Expects

Cashier B expects:
- to find unpaid orders reliably
- to open the order and see what was placed (and when)
- to collect payment and close the order
- the receipt to be correct and stable

Managers/owners expect:
- auditability: who placed the order, who settled it, when
- cash accountability to remain correct within the active cash session

---

## Constraints & Reality

- The business cannot pause just because a shift changed.
- Customers may dispute the total if it is not explainable.
- Network retries can cause double-submit risks.

The system must support safe handover without introducing complex "ownership transfers" for MVP.

---

## How Modula Helps

Modula supports this by:
- making the unpaid order a branch-scoped operational record (not tied to one device)
- allowing authorized staff to settle an unpaid order (permission-based, branch-scoped)
- recording distinct audit evidence for:
  - who placed the order
  - who settled the payment

This keeps the workflow operationally realistic without needing table transfers or split payments in March.

---

## What This Story Is Not About

- Multi-register drawer models
- Assigning orders to individual staff as an exclusive lock
- Advanced restaurant features (tables, courses, split/partial payments)

