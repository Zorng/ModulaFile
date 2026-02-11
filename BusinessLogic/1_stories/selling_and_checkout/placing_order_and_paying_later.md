# Placing an Order and Paying Later (Table Service)

## Context

Not every business collects payment immediately at the counter.

In restaurants and table-service cafes, a common workflow is:
- the customer orders first,
- items are prepared and served,
- payment happens later when the customer asks for the bill.

This story explains the operational reality of **pay-later** and what the system must support so staff can keep moving without losing accountability.

---

## The Situation

A customer sits down and orders multiple items over time.

The staff needs to:
- place the order so the kitchen can start preparing,
- add more items later (another round),
- and settle the full bill at the end.

Payment is not part of "placing the order" in this workflow.

---

## What the Staff Expects

The staff expects:
- to create an order that the kitchen can see **before payment**
- to keep that order visible until it is paid and closed
- to add items without rewriting history (each add is a new batch of work)
- to settle the bill later without re-entering the whole order
- the system to keep totals stable and explainable (no surprise changes)

They do not expect:
- the kitchen to wait for payment to start
- the POS to block operations due to "payment not done yet"
- the final bill to change because the menu or tax settings changed mid-service

---

## Constraints & Reality

- Staff shifts change; the person who placed the order might not be the person who collects payment.
- Connectivity can be unstable.
- Mistakes happen (wrong table, duplicate actions, accidental re-prints).

Even with these realities, the system must remain predictable.

---

## How Modula Helps

Modula supports this workflow by treating "place order" as creating a **real operational record** (an unpaid ticket) that:
- can be fulfilled and printed to the kitchen immediately
- can be extended by adding items (new batches)
- can be settled later to create the immutable financial sale record

The system keeps:
- fulfillment tracking separate from financial finalization
- stable identifiers so retries do not create duplicates

---

## What This Story Is Not About

- Split bills
- Partial payments
- Merging tickets or transferring tables
- Per-line fulfillment status tracking

Those are real, but they are not required for the March MVP.

