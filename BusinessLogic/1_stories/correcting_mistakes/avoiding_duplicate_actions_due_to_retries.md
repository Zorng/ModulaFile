# Avoiding Duplicate Actions During Unstable Connectivity (Cashier / Manager)

## Context

In a busy cafe, the cashier moves fast.

Sometimes the internet is slow, the app freezes, or the screen does not respond.
The cashier taps again just to make sure the action went through.

This is normal behavior under pressure.

---

## The Situation

A cashier finalizes a sale and the screen hangs.
They tap "Finalize" again.

Later, the manager sees:
- two identical sales
- cash movement that looks doubled
- inventory deducted twice

Now the team is forced to clean up an error they did not intend to create.

---

## What the Cashier Expects

They expect:
- the system to treat repeated taps as the same intent
- a clear "pending" or "synced" state so they know what happened
- no duplicate receipts for the same customer

They do **not** expect:
- to remember whether they already tapped
- to manually correct duplicates later

---

## What the Manager / Owner Expects

They expect:
- the system to prevent duplicates from retries
- auditability when a retry occurs
- no surprise cash/inventory distortions caused by network issues

They do **not** expect:
- to reconcile duplicate records as normal daily work

---

## Constraints & Reality

- Network timeouts happen even when "online".
- Offline replay can submit the same operation multiple times.
- Human retries are inevitable.

The system must absorb retries safely without changing business truth twice.

---

## How Modula Helps

Modula should treat repeated submissions of the same action as **one intent**:
- a retry is not a new sale
- a retry is not a second cash movement
- a retry is not a duplicate attendance action

This preserves integrity while keeping work flowing.

---

## What This Story Is Not About

- fraud detection
- reporting analytics
- refunds and reversals after a true duplicate

This story is about **preventing unintended duplicates** caused by retries.

